# Next Steps

## Issues resolved
- Transformed Bookstore.Domain.csproj to net8.0
- Transformed Bookstore.Data.csproj to net8.0
- Transformed Bookstore.Web.csproj to net8.0
- Transformed Bookstore.Cdk.csproj to net8.0
- Transformed Bookstore.Domain.Tests.csproj to net8.0

## Overview

The solution has been transformed with no build errors across all projects:

- `Bookstore.Data`
- `Bookstore.Domain.Tests`
- `Bookstore.Cdk`
- `Bookstore.Web`
- `Bookstore.Domain`

Since no build errors were detected, the transformation appears to have been successful. The following steps outline how to validate, test, and deploy the migrated solution.

---

## 1. Restore and Build the Solution

Run the following commands from the root of the solution to confirm a clean restore and build:

```bash
dotnet restore
dotnet build --configuration Release
```

Ensure there are no warnings that could indicate compatibility issues, deprecated APIs, or missing references that were silently ignored during transformation.

---

## 2. Run the Domain Unit Tests

Execute the test project to verify that existing business logic behaves as expected after migration:

```bash
dotnet test app/Bookstore.Domain.Tests/Bookstore.Domain.Tests.csproj --configuration Release --verbosity normal
```

Review the test output carefully. Any failing tests should be investigated before proceeding, as they may indicate behavioral regressions introduced during the transformation.

---

## 3. Verify Runtime Behavior of the Web Project

Run the web application locally to confirm it starts and operates correctly:

```bash
dotnet run --project app/Bookstore.Web/Bookstore.Web.csproj --configuration Release
```

Manually test the following areas at a minimum:

- Application startup with no unhandled exceptions
- Database connectivity through `Bookstore.Data`
- Core domain workflows exposed through `Bookstore.Web`
- Any authentication or authorization flows if present

---

## 4. Validate the Data Layer

Confirm that `Bookstore.Data` is functioning correctly against the target database:

- If the project uses Entity Framework Core, check that all migrations are present and up to date:

```bash
dotnet ef migrations list --project app/Bookstore.Data/Bookstore.Data.csproj
```

- Apply any pending migrations to a test database:

```bash
dotnet ef database update --project app/Bookstore.Data/Bookstore.Data.csproj
```

- Verify that all queries and data access operations return expected results.

---

## 5. Review the CDK Project

The `Bookstore.Cdk` project likely defines infrastructure. Review its configuration to ensure it reflects the correct target environment for the migrated .NET version:

- Confirm that any runtime identifiers or Lambda/EC2 runtime targets reference the updated .NET version (e.g., `net8.0` instead of a legacy target).
- Review any hardcoded environment-specific values such as connection strings, region settings, or resource names.

---

## 6. Check Target Framework Consistency

Verify that all projects in the solution are targeting the same or compatible .NET versions. Open each `.csproj` file and confirm the `<TargetFramework>` element is consistent:

```xml
<TargetFramework>net8.0</TargetFramework>
```

Inconsistent target frameworks across projects can cause runtime issues even when the build succeeds.

---

## 7. Review NuGet Package Compatibility

Check that all NuGet packages referenced across the solution are compatible with the target framework:

```bash
dotnet list package --outdated
```

Update any packages that have newer versions compatible with the current framework. Pay particular attention to packages in `Bookstore.Data` and `Bookstore.Domain`, as these are the most foundational.

---

## 8. Publish the Web Application

Once validation is complete, publish the web application to confirm a clean release artifact is produced:

```bash
dotnet publish app/Bookstore.Web/Bookstore.Web.csproj --configuration Release --output ./publish
```

Review the output directory to ensure all required files, configuration, and assets are present before deploying to the target environment.