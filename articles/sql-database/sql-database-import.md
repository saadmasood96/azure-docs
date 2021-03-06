---
title: Import a BACPAC file to create an Azure SQL database | Microsoft Docs
description: Create a newAzure SQL database by importing a BACPAC file.
services: sql-database
ms.service: sql-database
ms.subservice: migration
ms.custom: 
ms.devlang: 
ms.topic: conceptual
author: CarlRabeler
ms.author: carlrab
ms.reviewer: 
manager: craigg
ms.date: 02/18/2019
---
# Quickstart: Import a BACPAC file to a database in Azure SQL Database

You can import a SQL Server database into a database in Azure SQL Database using a [BACPAC](https://docs.microsoft.com/sql/relational-databases/data-tier-applications/data-tier-applications#bacpac) file. You can import the data from a `BACPAC` file stored in Azure Blob storage (standard storage only) or from local storage in an on-premises location. To maximize import speed by providing more and faster resources, scale your database to a higher service tier and compute size during the import process. You can then scale down after the import is successful.

> [!NOTE]
> The imported database's compatibility level is based on the source database's compatibility level.
> [!IMPORTANT]
> After importing your database, you can choose to operate the database at its current compatibility level (level 100 for the AdventureWorks2008R2 database) or at a higher level. For more information on the implications and options for operating a database at a specific compatibility level, see [ALTER DATABASE Compatibility Level](https://docs.microsoft.com/sql/t-sql/statements/alter-database-transact-sql-compatibility-level). See also [ALTER DATABASE SCOPED CONFIGURATION](https://docs.microsoft.com/sql/t-sql/statements/alter-database-scoped-configuration-transact-sql) for information about additional database-level settings related to compatibility levels.

## Import from a BACPAC file in the Azure portal

The [Azure portal](https://portal.azure.com) *only* supports creating a single database in Azure SQL Database and *only* from a BACPAC file stored in Azure Blob storage.

> [!NOTE]
> [A managed instance](sql-database-managed-instance.md) does not currently support migrating a database into an instance database from a BACPAC file using the Azure portal. To import into a managed instance, use SQL Server Management Studio or SQLPackage.

1. To import from a BACPAC file into a new single database using the Azure portal, open the appropriate database server page and then, on the toolbar, select **Import database**.  

   ![Database import1](./media/sql-database-import/import1.png)

2. Select the storage account and the container for the BACPAC file and then select the BACPAC file from which to import.
3. Specify the new database size (usually the same as origin) and provide the destination SQL Server credentials. For a list of possible values for a new Azure SQL database, see [Create Database](https://docs.microsoft.com/sql/t-sql/statements/create-database-transact-sql?view=azuresqldb-current).

   ![Database import2](./media/sql-database-import/import2.png)

4. Click **OK**.

5. To monitor an import's progress, open the database's server page, and, under **Settings**, select **Import/Export history**. When successful, the import has a **Completed** status.

   ![Database import status](./media/sql-database-import/import-status.png)

6. To verify the database is live on the database server, select **SQL databases** and verify the new database is **Online**.

## Import from a BACPAC file using SqlPackage

To import a SQL Server database using the [SqlPackage](https://docs.microsoft.com/sql/tools/sqlpackage) command-line utility, see [import parameters and properties](https://docs.microsoft.com/sql/tools/sqlpackage#import-parameters-and-properties). SqlPackage has the latest [SQL Server Management Studio](https://docs.microsoft.com/sql/ssms/download-sql-server-management-studio-ssms) and [SQL Server Data Tools for Visual Studio](https://msdn.microsoft.com/library/mt204009.aspx). You can also download the latest [SqlPackage](https://www.microsoft.com/download/details.aspx?id=53876) from the Microsoft download center.

For scale and performance, we recommend using SqlPackage in most production environments rather than using the Azure portal. For a SQL Server Customer Advisory Team blog about migrating using `BACPAC` files, see [migrating from SQL Server to Azure SQL Database using BACPAC Files](https://blogs.msdn.microsoft.com/sqlcat/2016/10/20/migrating-from-sql-server-to-azure-sql-database-using-bacpac-files/).

The following SqlPackage command imports the **AdventureWorks2008R2** database from local storage to an Azure SQL Database server called **mynewserver20170403**. It creates a new database called **myMigratedDatabase** with a **Premium** service tier and a **P6** Service Objective. Change these values as appropriate for your environment.

```cmd
SqlPackage.exe /a:import /tcs:"Data Source=mynewserver20170403.database.windows.net;Initial Catalog=myMigratedDatabase;User Id=<your_server_admin_account_user_id>;Password=<your_server_admin_account_password>" /sf:AdventureWorks2008R2.bacpac /p:DatabaseEdition=Premium /p:DatabaseServiceObjective=P6
```

> [!IMPORTANT]
> To connect to a SQL Database server managing a single database from behind a corporate firewall, the firewall must have port 1433 open. To connect to a managed instance, you must have a [point-to-site connection](sql-database-managed-instance-configure-p2s.md) or an express route connection.
>

This example shows how to import a database using SqlPackage with Active Directory Universal Authentication.

```cmd
SqlPackage.exe /a:Import /sf:testExport.bacpac /tdn:NewDacFX /tsn:apptestserver.database.windows.net /ua:True /tid:"apptest.onmicrosoft.com"
```

## Import into a single database from a BACPAC file using PowerShell

> [!NOTE]
> [A managed instance](sql-database-managed-instance.md) does not currently support migrating a database into an instance database from a BACPAC file using Azure PowerShell]. To import into a managed instance, use SQL Server Management Studio or SQLPackage.


Use the [New-AzureRmSqlDatabaseImport](/powershell/module/azurerm.sql/new-azurermsqldatabaseimport) cmdlet to submit an import database request to the Azure SQL Database service. Depending on database size, the import may take some time to complete.

 ```powershell
 $importRequest = New-AzureRmSqlDatabaseImport
    -ResourceGroupName "<your_resource_group>" `
    -ServerName "<your_server>" `
    -DatabaseName "<your_database>" `
    -DatabaseMaxSizeBytes "<database_size_in_bytes>" `
    -StorageKeyType "StorageAccessKey" `
    -StorageKey $(Get-AzureRmStorageAccountKey -ResourceGroupName "<your_resource_group>" -StorageAccountName "<your_storage_account").Value[0] `
    -StorageUri "https://myStorageAccount.blob.core.windows.net/importsample/sample.bacpac" `
    -Edition "Standard" `
    -ServiceObjectiveName "P6" `
    -AdministratorLogin "<your_server_admin_account_user_id>" `
    -AdministratorLoginPassword $(ConvertTo-SecureString -String "<your_server_admin_account_password>" -AsPlainText -Force)

 ```

 You can use the [Get-AzureRmSqlDatabaseImportExportStatus](/powershell/module/azurerm.sql/get-azurermsqldatabaseimportexportstatus) cmdlet to check the import's progress. Running the cmdlet immediately after the request usually returns **Status: InProgress**. The import is complete when you see **Status: Succeeded**.

```powershell
$importStatus = Get-AzureRmSqlDatabaseImportExportStatus -OperationStatusLink $importRequest.OperationStatusLink
[Console]::Write("Importing")
while ($importStatus.Status -eq "InProgress")
{
    $importStatus = Get-AzureRmSqlDatabaseImportExportStatus -OperationStatusLink $importRequest.OperationStatusLink
    [Console]::Write(".")
    Start-Sleep -s 10
}
[Console]::WriteLine("")
$importStatus
```

> [!TIP]
For another script example, see [Import a database from a BACPAC file](scripts/sql-database-import-from-bacpac-powershell.md).

## Limitations

Importing to a database in elastic pool isn't supported. You can import data into a single database and then move the database to an elastic pool.

## Import using wizards

You can also use these wizards.

- [Import Data-tier Application Wizard in SQL Server Management Studio](https://docs.microsoft.com/sql/relational-databases/data-tier-applications/import-a-bacpac-file-to-create-a-new-user-database#using-the-import-data-tier-application-wizard).
- [SQL Server Import and Export Wizard](https://docs.microsoft.com/sql/integration-services/import-export-data/start-the-sql-server-import-and-export-wizard).

## Next steps

- To learn how to connect to and query an imported SQL Database, see [Quickstart: Azure SQL Database: Use SQL Server Management Studio to connect and query data](sql-database-connect-query-ssms.md).
- For a SQL Server Customer Advisory Team blog about migrating using BACPAC files, see [Migrating from SQL Server to Azure SQL Database using BACPAC Files](https://techcommunity.microsoft.com/t5/DataCAT/Migrating-from-SQL-Server-to-Azure-SQL-Database-using-Bacpac/ba-p/305407).
- For a discussion of the entire SQL Server database migration process, including performance recommendations, see [SQL Server database migration to Azure SQL Database](sql-database-single-database-migrate.md).
- To learn how to manage and share storage keys and shared access signatures securely, see [Azure Storage Security Guide](https://docs.microsoft.com/azure/storage/common/storage-security-guide).
