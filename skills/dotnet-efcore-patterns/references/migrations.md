# Migration Management

## CLI Commands

### Create Migration

```bash
# Create a new migration
dotnet ef migrations add MigrationName --project MyApp.Infrastructure

# With specific context
dotnet ef migrations add MigrationName --context AppDbContext --project MyApp.Infrastructure

# With output directory
dotnet ef migrations add MigrationName --project MyApp.Infrastructure --output-dir Data/Migrations
```

### Apply Migrations

```bash
# Update to latest migration
dotnet ef database update --project MyApp.Infrastructure

# Update to specific migration
dotnet ef database update TargetMigration --project MyApp.Infrastructure

# Generate SQL script without applying
dotnet ef migrations script --project MyApp.Infrastructure --output migration.sql

# Generate idempotent script (can run multiple times safely)
dotnet ef migrations script --idempotent --project MyApp.Infrastructure --output migration.sql
```

### Remove Migration

```bash
# Remove the last migration (only if not applied to database)
dotnet ef migrations remove --project MyApp.Infrastructure

# Force remove (even if applied - dangerous!)
dotnet ef migrations remove --force --project MyApp.Infrastructure
```

### List Migrations

```bash
# Show all migrations
dotnet ef migrations list --project MyApp.Infrastructure

# Show pending migrations
dotnet ef database update --dry-run --project MyApp.Infrastructure
```

## Migration Workflow

### 1. Make Model Changes

```csharp
public class Order
{
    public int Id { get; set; }
    public string CustomerId { get; set; }
    public decimal Total { get; set; }
    // NEW: Add this property
    public OrderStatus Status { get; set; }
}
```

### 2. Create Migration

```bash
dotnet ef migrations add AddOrderStatus --project MyApp.Infrastructure
```

This generates:
- `20240128123456_AddOrderStatus.cs` - Migration code
- `20240128123456_AddOrderStatus.Designer.cs` - Model snapshot

### 3. Review Migration

```csharp
public partial class AddOrderStatus : Migration
{
    protected override void Up(MigrationBuilder migrationBuilder)
    {
        migrationBuilder.AddColumn<int>(
            name: "Status",
            table: "Orders",
            type: "int",
            nullable: false,
            defaultValue: 0);
    }

    protected override void Down(MigrationBuilder migrationBuilder)
    {
        migrationBuilder.DropColumn(
            name: "Status",
            table: "Orders");
    }
}
```

### 4. Apply Migration

```bash
# Development
dotnet ef database update --project MyApp.Infrastructure

# Production - use migration service (see migration-service.md)
```

## Best Practices

### ✅ DO

- Review generated migrations before applying
- Use descriptive migration names
- Test migrations on staging before production
- Generate idempotent scripts for production
- Version control your migrations
- Run migrations in dedicated service on startup

### ❌ DON'T

- Edit migration files manually (regenerate instead)
- Delete migrations that have been applied
- Apply migrations directly in production without testing
- Use `dotnet ef database update` in production
- Modify model snapshot files

## Handling Migration Conflicts

### Conflict: Multiple Developers Create Migrations

```bash
# Developer A creates migration
git checkout main
dotnet ef migrations add AddOrderStatus

# Developer B creates migration (conflict!)
git checkout feature-branch
dotnet ef migrations add AddCustomerAddress

# Merge conflicts in model snapshot
```

**Solution:**

```bash
# Developer B: Pull latest main
git pull origin main

# Remove your migration
dotnet ef migrations remove

# Recreate after pulling
dotnet ef migrations add AddCustomerAddress
```

## Data Migrations

For complex data transformations:

```csharp
protected override void Up(MigrationBuilder migrationBuilder)
{
    // 1. Add new column
    migrationBuilder.AddColumn<string>(
        name: "FullName",
        table: "Customers",
        nullable: true);

    // 2. Migrate data
    migrationBuilder.Sql(@"
        UPDATE Customers
        SET FullName = FirstName + ' ' + LastName
        WHERE FirstName IS NOT NULL AND LastName IS NOT NULL
    ");

    // 3. Make column required
    migrationBuilder.AlterColumn<string>(
        name: "FullName",
        table: "Customers",
        nullable: false);

    // 4. Drop old columns
    migrationBuilder.DropColumn("FirstName", "Customers");
    migrationBuilder.DropColumn("LastName", "Customers");
}
```

## Rolling Back Migrations

```bash
# Roll back one migration
dotnet ef database update PreviousMigration --project MyApp.Infrastructure

# Roll back all migrations
dotnet ef database update 0 --project MyApp.Infrastructure
```
