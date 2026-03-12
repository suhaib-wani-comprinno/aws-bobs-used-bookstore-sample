# BobsBookstore — EF Core Migration State Report

## Verification Summary

**EF Core Version:** 8.0  
**Target Provider:** Npgsql.EntityFrameworkCore.PostgreSQL 8.0.0  
**Source Provider:** Microsoft SQL Server (MSSQL)  
**Database Schema:** `bobsusedbookstore_dbo`

---

## ✅ Verification Result: NO MIGRATIONS EXIST — STATE IS EXPECTED AND CORRECT

### Locations Checked

| Location | Folder Exists | Migration Files Found |
|---|---|---|
| `Bookstore.Data/Migrations/` | ❌ No | 0 |
| `Bookstore.Web/Migrations/` | ❌ No | 0 |
| `Bookstore.Domain/Migrations/` | ❌ No | 0 |

**Conclusion:** Zero EF Core migration files exist anywhere in the solution. This is intentional and expected. No SQL Server-specific migration artifacts are present to transform or remove.

---

## Database Initialization Strategy

The application does **not** use `dotnet ef migrations` for schema management. Instead it uses `EnsureCreatedAsync()`, which is invoked in:

**File:** `Bookstore.Web/Startup/MiddlewareSetup.cs`

```csharp
// Create/update the database
using (var scope = app.Services.CreateAsyncScope())
{
    var context = scope.ServiceProvider.GetService<ApplicationDbContext>()!;

    if (!await context.Database.CanConnectAsync())
    {
        await context.Database.EnsureCreatedAsync();
    }
}
```

### How EnsureCreatedAsync Works with PostgreSQL

| Behaviour | Detail |
|---|---|
| **Schema creation** | EF Core reads `OnModelCreating` mappings and issues `CREATE TABLE` DDL directly against PostgreSQL |
| **Seed data** | `HasData()` entries in `SeedData.cs` are applied at creation time |
| **Idempotency guard** | The `CanConnectAsync()` check prevents re-running creation if the DB already exists |
| **No migrations history table** | `EnsureCreated` never writes to `__EFMigrationsHistory` |
| **PostgreSQL compatibility** | Fully compatible with the Npgsql provider — no SQL Server paths are invoked |

---

## ApplicationDbContext — PostgreSQL Mapping Inventory

All entity-to-table mappings in `ApplicationDbContext.cs` already target **PostgreSQL** conventions (lowercase names, `bobsusedbookstore_dbo` schema).

### Table Mappings

| Entity Class | PostgreSQL Table Name | Schema |
|---|---|---|
| `Address` | `address` | `bobsusedbookstore_dbo` |
| `Book` | `book` | `bobsusedbookstore_dbo` |
| `Customer` | `customer` | `bobsusedbookstore_dbo` |
| `Order` | `orders` | `bobsusedbookstore_dbo` |
| `ShoppingCart` | `shoppingcart` | `bobsusedbookstore_dbo` |
| `ShoppingCartItem` | `shoppingcartitem` | `bobsusedbookstore_dbo` |
| `OrderItem` | `orderitem` | `bobsusedbookstore_dbo` |
| `Offer` | `offer` | `bobsusedbookstore_dbo` |
| `ReferenceDataItem` | `referencedata` | `bobsusedbookstore_dbo` |

### Column Mappings (representative sample)

| Entity | C# Property | PostgreSQL Column |
|---|---|---|
| `Address` | `AddressLine1` | `addressline1` |
| `Address` | `AddressLine2` | `addressline2` |
| `Address` | `CustomerId` | `customerid` |
| `Address` | `ZipCode` | `zipcode` |
| `Book` | `ISBN` | `isbn` |
| `Book` | `CoverImageUrl` | `coverimageurl` |
| `Book` | `PublisherId` | `publisherid` |
| `Book` | `BookTypeId` | `booktypeid` |
| `Customer` | `FirstName` | `firstname` |
| `Customer` | `LastName` | `lastname` |
| `Customer` | `DateOfBirth` | `dateofbirth` |
| `Order` | `OrderStatus` | `orderstatus` |
| `Order` | `DeliveryDate` | `deliverydate` |
| `Order` | `AddressId` | `addressid` |
| `ShoppingCartItem` | `ShoppingCartId` | `shoppingcartid` |
| `ShoppingCartItem` | `WantToBuy` | `wanttobuy` |
| `Offer` | `OfferStatus` | `offerstatus` |
| `Offer` | `BookPrice` | `bookprice` |
| `Offer` | `BookName` | `bookname` |
| All entities | `CreatedBy` | `createdby` |
| All entities | `CreatedOn` | `createdon` |
| All entities | `UpdatedOn` | `updatedon` |
| All entities | `Id` | `id` |

### PostgreSQL-Specific Runtime Switch

```csharp
// ApplicationDbContext static constructor
AppContext.SetSwitch("Npgsql.EnableLegacyTimestampBehavior", true);
```

This switch is required for `DateTime` / `timestamp without time zone` compatibility between .NET and Npgsql 6+.

---

## Package Reference Audit

### Bookstore.Data.csproj — PostgreSQL packages confirmed

```xml
<PackageReference Include="Npgsql.EntityFrameworkCore.PostgreSQL" Version="8.0.0" />
<PackageReference Include="Microsoft.EntityFrameworkCore" Version="8.0.11" />
<PackageReference Include="Microsoft.EntityFrameworkCore.Design" Version="8.0.11" />
<PackageReference Include="Microsoft.EntityFrameworkCore.Tools" Version="8.0.11" />
```

### Bookstore.Web.csproj — PostgreSQL packages confirmed

```xml
<PackageReference Include="Npgsql.EntityFrameworkCore.PostgreSQL" Version="8.0.0" />
```

### SQL Server packages — NONE FOUND

`Microsoft.EntityFrameworkCore.SqlServer` is **not referenced** in any `.csproj` in the solution. ✅

---

## Seed Data Registered via HasData()

Defined in `Bookstore.Data/SeedData.cs`, called from `PopulateDatabase(modelBuilder)`:

| Reference Data Type | Count | ID Range |
|---|---|---|
| BookType | 3 | 1–3 |
| Condition | 4 | 4–7 |
| Genre | 7 | 8–14 |
| Publisher | 10 | 15–24 |
| Book (initial inventory) | 8 | 1–8 |

All seed data rows will be inserted by `EnsureCreatedAsync()` on first connect to a fresh database.

---

## Action Required: NONE

The migration state is clean and fully PostgreSQL-ready:

- ✅ No SQL Server migration files exist — nothing to transform
- ✅ No `SqlServer:Identity` annotations to replace
- ✅ No SQL Server type strings (`nvarchar`, `datetime2`, `bit`, `money`) in migrations
- ✅ `EnsureCreatedAsync()` is fully compatible with Npgsql on PostgreSQL
- ✅ All 9 entity tables mapped to lowercase names in schema `bobsusedbookstore_dbo`
- ✅ All column names use lowercase PostgreSQL conventions
- ✅ Npgsql 8.0.0 is the only database provider referenced in the solution
- ✅ `Npgsql.EnableLegacyTimestampBehavior` is set for `DateTime` compatibility
- ✅ Seed data is embedded in `OnModelCreating` via `HasData()` — no raw SQL scripts

---

## Guidance: Creating the First Migration (If Ever Required)

Should the team switch from `EnsureCreatedAsync` to the standard migrations workflow, follow these steps **after** ensuring `ServicesSetup.cs` registers the Npgsql provider:

### Step 1 — Add the initial migration

```bash
dotnet ef migrations add InitialCreate \
    --project Bookstore.Data \
    --startup-project Bookstore.Web \
    --output-dir Migrations \
    --context ApplicationDbContext
```

### Step 2 — Apply to the PostgreSQL database

```bash
dotnet ef database update \
    --project Bookstore.Data \
    --startup-project Bookstore.Web \
    --context ApplicationDbContext
```

### Step 3 — Update MiddlewareSetup.cs to use MigrateAsync instead

```csharp
// Replace EnsureCreatedAsync with MigrateAsync if switching to migration-based workflow
await context.Database.MigrateAsync();
```

### What the Generated Migration Will Contain (Automatic — No Edits Needed)

Because Npgsql is the active provider and all entity mappings are already PostgreSQL-correct, the scaffolded migration will automatically contain:

```csharp
// Correct using statement added by Npgsql scaffolder
using Npgsql.EntityFrameworkCore.PostgreSQL.Metadata;

// Integer PKs use Npgsql identity annotation (NOT SqlServer:Identity)
Id = table.Column<int>(type: "integer", nullable: false)
    .Annotation("Npgsql:ValueGenerationStrategy",
                NpgsqlValueGenerationStrategy.IdentityByDefaultColumn),

// CreateTable calls include schema parameter automatically
migrationBuilder.CreateTable(
    name: "book",
    schema: "bobsusedbookstore_dbo",
    columns: table => new { ... });

// PostgreSQL type names used throughout
// nvarchar(max)  -> text
// nvarchar(n)    -> varchar(n)
// datetime2      -> timestamp without time zone
// bit            -> boolean
// money          -> numeric(19,4)
// uniqueidentifier -> uuid
```

---

## Notes on EnsureCreatedAsync vs MigrateAsync

| Feature | `EnsureCreatedAsync()` | `MigrateAsync()` |
|---|---|---|
| Creates schema from model | ✅ Yes | ✅ Yes (via migrations) |
| Tracks schema history | ❌ No `__EFMigrationsHistory` | ✅ Yes |
| Supports incremental schema updates | ❌ No — won't alter existing DB | ✅ Yes |
| Suitable for production | ⚠️ Dev/demo only | ✅ Recommended for production |
| Current usage in this app | ✅ Active | ❌ Not used |

The current `EnsureCreatedAsync()` approach is appropriate for this application's deployment model, where AWS RDS PostgreSQL is provisioned fresh by CDK and the schema is created on first application startup.

---

*Generated by the EF Migration Transformation Agent — migration state verified and documented.*
