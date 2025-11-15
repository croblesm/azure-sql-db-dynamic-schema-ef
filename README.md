---
page_type: sample
languages:
- tsql
- sql
- csharp
products:
- azure-sql-database
- dotnet
- ef-core
- sql-server
description: "Dynamic Schema Management With Azure SQL and Entity Framework "
---

# Dynamic Schema Management With Azure SQL and Entity Framework 

![License](https://img.shields.io/badge/license-MIT-green.svg)

A sample project that shows how to deal with dynamic schema in Azure SQL, using the native JSON support and Entity Framework Core. This repo is a variation of the "hybrid" sample discussed and shown in the 

https://github.com/azure-samples/azure-sql-db-dynamic-schema

repository, but using Entity Framework Core instead of Dapper. Since EF Core 7, in fact, it is now possible to let the framework handle the serialization and deserialization of an object into a JSON column, making the code much cleaner and easier to maintain:

https://learn.microsoft.com/ef/core/what-is-new/ef-core-7.0/whatsnew#json-columns

## Local Database Setup

Make sure you have SQL Server 2025 installed. The easiest way to do this is to use Docker or Podman and the VSCode MSSQL extension with a [local SQL Server container](https://learn.microsoft.com/sql/tools/visual-studio-code-extensions/mssql/mssql-local-container?view=sql-server-ver17).

If you don't want to use a local database, you can also use an Azure SQL Database. You can use the *Free Offer* to have a [completely free Azure SQL database to use](https://learn.microsoft.com/azure/azure-sql/database/free-offer?view=azuresql).

### Option 1: Using SA User (Simplest - Recommended for Local Development)

This is the simplest approach as the SA user has all permissions needed to create the database and run migrations.

1. **Copy the environment file:**
   ```bash
   cp .env.sample .env
   ```

2. **Update `.env` with your SA credentials:**
   ```
   MSSQL="SERVER=localhost;DATABASE=dynamic-schema-ef;UID=sa;PWD=P@ssw0rd!;TrustServerCertificate=True"
   ```
   Replace `P@ssw0rd!` with your actual SA password.

3. **Install Entity Framework Core tools (if not already installed):**
   ```bash
   dotnet tool install --global dotnet-ef
   ```

4. **Create the database and run migrations:**

   Since `dotnet ef` doesn't automatically load the .env file, you need to provide the connection string directly:

   ```bash
   dotnet ef database update --connection "SERVER=localhost;DATABASE=dynamic-schema-ef;UID=sa;PWD=P@ssw0rd!;TrustServerCertificate=True"
   ```

   Replace `P@ssw0rd!` with your actual SA password.

   This will:
   - Create the `dynamic-schema-ef` database if it doesn't exist
   - Create the `global_sequence` sequence
   - Create the `todo_hybrid` table with JSON column support

5. **Run the application:**
   ```bash
   dotnet watch
   ```

### Option 2: Using Dedicated Application User (Production-like Setup)

This approach creates a dedicated user with limited permissions, similar to a production environment.

1. **Create the database and user using SA:**

   Connect to SQL Server with your SA user and run:
   ```bash
   sqlcmd -S localhost -U sa -P P@ssw0rd! -i Database/00-create.sql
   ```

   Or execute the `Database/00-create.sql` script manually. This will:
   - Create the `dynamic-schema-ef` database
   - Create the `web` schema
   - Create the `dynamic-schema-test-user` login and user
   - Grant `db_datareader` and `db_datawriter` permissions

2. **Grant additional permissions for migrations:**

   The application user needs additional permissions to run EF migrations. Connect with SA and run:
   ```sql
   USE [dynamic-schema-ef];
   ALTER ROLE db_ddladmin ADD MEMBER [dynamic-schema-test-user];
   ```

3. **Copy and configure environment file:**
   ```bash
   cp .env.sample .env
   ```

   Update `.env` to use the application user:
   ```
   MSSQL="SERVER=localhost;DATABASE=dynamic-schema-ef;UID=dynamic-schema-test-user;PWD=Super_Str0ng*P@ZZword!;TrustServerCertificate=True"
   ```

4. **Run migrations:**

   Since `dotnet ef` doesn't automatically load the .env file, provide the connection string directly:

   ```bash
   dotnet ef database update --connection "SERVER=localhost;DATABASE=dynamic-schema-ef;UID=dynamic-schema-test-user;PWD=Super_Str0ng*P@ZZword!;TrustServerCertificate=True"
   ```

5. **Run the application:**
   ```bash
   dotnet watch
   ```

### Regenerating Migrations (Optional)

If you need to regenerate migrations (migrations already exist in this project):

```bash
# Remove existing migrations
rm -rf Migrations/

# Create new migration
dotnet ef migrations add InitialCreate

# Apply migration
dotnet ef database update
```

### Notes

- **Database Creation**: When using Option 1 (SA user), EF Core will automatically create the `dynamic-schema-ef` database if it doesn't exist. No manual database creation is needed.
- **Schema Usage**: The EF migrations create objects in the default `dbo` schema. The `web` schema created by `Database/00-create.sql` is not currently used by the application but is included for reference.
- **Permissions**: For Option 2, the `dynamic-schema-test-user` needs `db_ddladmin` role to run migrations. In production, you would typically run migrations with an admin account during deployment and use a limited user for the running application.

# Run the sample app

Run in watch mode

```
dotnet watch
```

then use the `Sample/sample.http` file to test out the API.

Uncomment the `ToDoExtension` properties in the `Entities/ToDo.cs` file and the properties in the `Controllers/ToDoHybridController.cs` file to add more fields to your entity.

The new properties will be automatically serialized and deserialized by EF Core without the need to change the database schema, and you can use them in your application without any additional code.
