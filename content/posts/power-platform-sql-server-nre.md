+++
date = '2025-08-22T00:00:00'
draft = true
title = 'Issues adding SQL Server Data Source to Power Platform Canvas App in a VNET-Integrated Environment'
+++

I’ve been working on a Power Platform Canvas App that uses SQL Server connections to update and retrieve data. The app relies on stored procedures for both reads and writes. Everything worked fine in one Power Platform environment, and the plan was to migrate the solution to another environment that is VNET-integrated.  

That’s when I hit a strange issue: I couldn’t add a SQL Server data source. I was able to fill in the **Server** and **Database** fields, select my stored procedures, and click **OK**—but instead of being added, the drawer simply closed, and no data source appeared.  

Checking the browser’s network tab revealed a failing `POST` request with an HTTP 500 error. The response contained this exception:  

```c#
System.NullReferenceException: Object reference not set to an instance of an object.
    at Microsoft.Azure.Connectors.Sql.NativeExecutor.SqlNativeExecutor.ExecuteStoredProcedureMetadataAsync(ISqlConnectionCredentials credentials, String procedureName, IReadOnlyDictionary`2 actualParameters, CancellationToken cancellationToken)   
    at Microsoft.Azure.Connectors.Sql.PlexExtension.Operations.NativeExecutorOperations.ExecuteStoredProcedureMetadataAsync(RequestWrapper`2 wrapper, CancellationToken cancellationToken) in C:\__w\1\s\src\Connectors\FirstParty\sql\PlexExtension\Operations\NativeExecutorOperations.cs:line 145   
    at Microsoft.Azure.Connectors.Common.SubnetDelegation.PlexExtension.PlexExtension.ExecuteAsync(ExtensionRequest extensionRequest, IExtensionServiceProvider serviceProvider, CancellationToken cancellationToken)
    clientRequestId: 4daadd37-9ca4-4441-b682-acdaef3e5a36
```

Well, that's really weird, since NRE is not something you would expect to see in the response from a service. A clear business or configuration error would’ve been more useful. Still, the stack trace gave me two important hints:  

1. `ExecuteStoredProcedureMetadataAsync` suggested the issue was tied to stored procedure metadata.  
2. `SubnetDelegation` implied it had something to do with the fact that this environment was **VNET-integrated**.  

In my original environment (non-VNET), all the same stored procedures worked fine.  

After some trial and error, we discovered a pattern:  
- Stored procedures that **return data** could be added as data sources.  
- Stored procedures that **only perform actions** (insert/update, no return set) triggered the `NullReferenceException`.  

 I remembered that some Microsoft products require stored procedures to return a result set, and the method name `ExecuteStoredProcedureMetadataAsync` reinforced that idea—it probably analyzes the schema of the returned data.  

To test the theory, I modified one of the action-only procedures to return a dummy row:  

```sql
SELECT 'Ok' AS Result;
```  

It worked! 🎉 I was finally able to add the SQL Server data source.  

I tried to confirm this requirement in Microsoft’s documentation but couldn’t find anything relevant. The docs I checked:  

- [Power Apps SQL Server connection overview](https://learn.microsoft.com/en-us/power-apps/maker/canvas-apps/connections/sql-connection-overview)  
- [SQL connector reference](https://learn.microsoft.com/en-us/connectors/sql/)  

So far, it looks like this behavior is specific to **VNET-integrated environments**. When the SQL connection goes over the public internet, the extra requirement doesn’t seem to apply.  

Hopefully this helps someone else who runs into the same confusing issue.  
