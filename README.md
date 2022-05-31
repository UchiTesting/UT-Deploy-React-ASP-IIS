

Deploying React and ASP .NET Core to IIS
========================================

**Table of Content**

  - [Setting up IIS](#setting-up-iis)
    - [Installing IIS](#installing-iis)
    - [Installing The Hosting bundle for ASP .NET Core](#installing-the-hosting-bundle-for-asp-net-core)
    - [Setting up the site](#setting-up-the-site)
      - [Creating a new site](#creating-a-new-site)
      - [Using the default Site](#using-the-default-site)
        - [Enable Logging](#enable-logging)
        - [Allow writing to the server](#allow-writing-to-the-server)
    - [Extra configuration for IdentityServer](#extra-configuration-for-identityserver)
  - [Setting up the Database](#setting-up-the-database)
    - [Create the database](#create-the-database)
    - [Create a user](#create-a-user)
  - [Publishing React Code](#publishing-react-code)
  - [Publishing .NET Core Web API](#publishing-net-core-web-api)
  - [Trouble-Shooting](#trouble-shooting)
    - [Error 500.19](#error-50019)
    - [Error 500.30](#error-50030)
  - [Managing Self-Signed Certificates on Windows](#managing-self-signed-certificates-on-windows)
    - [Create a Self-Signed Certificate](#create-a-self-signed-certificate)
    - [Move certificates to WebHosting Cert Store](#move-certificates-to-webhosting-cert-store)
    - [Move a specific certificate to WebHosting Cert Store](#move-a-specific-certificate-to-webhosting-cert-store)

##  Setting up IIS

### Installing IIS

In Windows Features Select IIS and click OK to proceed.

![Installing IIS from Windows Features dialog](<img/IIS - Install IIS.png>)


###  Installing The Hosting bundle for ASP .NET Core

We can download the ASP .NET Core hosting bundle from the [.NET Download page](https://dotnet.microsoft.com/en-us/download/dotnet). Each version of the framework has a corresponding bundle. 

![Focus from the .NET Download page for the ASP .NET runtimes](<img/ASP .NET - Runtimes.png>)

###  Setting up the site

There are 2 possibilities : Use the default site, or create one. 

####  Creating a new site

> This part may be documented in a further version.  
> It simply shall describe the process of creating a new site.  
> The configuration reflects on the default site which description can be found bellow.

####  Using the default Site

##### Enable Logging

By default logging of errors is disabled.
For a specific site, go to the *Configuration Editor* and toggle `stdoutLogEnabled` to `True`. Also `stdoutLogFile` allow to define where the logs will be located. the default value of `.\logs\stdout` is fine.

![Enabling Logging for a site](<img/IIS - Enable Logging.png>)

For more control on logging we can edit `Web.config` as follows :

```xml
<?xml version="1.0" encoding="utf-8"?>
<configuration>
  <location path="." inheritInChildApplications="false">
    <system.webServer>
      <handlers>
        <add name="aspNetCore" path="*" verb="*" modules="AspNetCoreModuleV2" resourceType="Unspecified" />
      </handlers>
			<aspNetCore processPath=".\BookLibrary.exe" stdoutLogEnabled="true" stdoutLogFile=".\logs\stdout" hostingModel="inprocess">
				<handlerSettings>
					<handlerSetting name="debugFile" value=".\logs\EnhancedAspnet-debug.log" />
					<handlerSetting name="debugLevel" value="FILE,TRACE" />
				</handlerSettings>
		</aspNetCore>
    </system.webServer>
  </location>
</configuration>
<!--ProjectGuid: 36b0bd70-df0a-4de4-a563-08c348d4b5ca-->
```

> Logging is not limited in size and can consume a lot of capacity which could lead to server failure. Disable logging when not needed.

##### Allow writing to the server

IIS has a user called `IIS_IUSRS`.  
On the screenshots bellow:
- Default permissions first.
- New permissions last.

![Default Permissions to IIS user](<img/IIS - Default permissions for IIS.png>)
![New Permissions to IIS user](<img/IIS - New permissions for IIS.png>)

> New permissions are given sequentially until you have enough for your site to work. You don't want to give to much of them to prevent any security risk. The screenshot is for illustration purpose.

> The Web site, related pool or server may need a restart to apply changes.

###  Extra configuration for IdentityServer

The last part is to provide a certificate to use by Identity Server.

in either `appsettings.json` or `appsettings.Production.json` add the following

```json
"IdentityServer": {
  "Key": {
    "Type": "Store",
    "StoreName": "My",
    "StoreLocation": "LocalMachine",
    "Name": "CN=localhost"
  }	 
}
```


Here is a full `appsettings.Production.json`
```json
{
  "ConnectionStrings": {
    "DefaultConnection": "{The Connection string}"
  },
  "Logging": {
    "LogLevel": {
      "Default": "Information",
      "Microsoft": "Warning",
      "Microsoft.Hosting.Lifetime": "Information"
    }
  },
  "IdentityServer": {
    "Clients": {
    "BookLibrary": {
      "Profile": "IdentityServerSPA"
    }
    },
    "Key": {
      "Type": "Store",
      "StoreName": "My",
      "StoreLocation": "LocalMachine",
      "Name": "CN=localhost"
    }	 
  },
  "AllowedHosts": "*"
}
```

> Here we still used the Development certificate provided by Visual Studio. You should create a dedicated self-signed certificate or get one from a cert authority.
>
> This is also true should you want to test the application from other device. see [Managing Self-Signed Certificates on Windows](#managing-self-signed-certificates-on-windows) section for more info.

Extra info:  
[Topic @GitHub](https://github.com/dotnet/aspnetcore/issues/28417) | 
[Docs @MDSN](https://docs.microsoft.com/en-us/aspnet/core/security/authentication/identity-api-authorization?view=aspnetcore-5.0#example-deploy-to-a-non-azure-web-hosting-provider)

[Back to Top](#deploying-react-and-asp-net-core-to-iis)

## Setting up the Database

### Create the database

In the context of ASP .NET Core we used Entity Framework Core.

During the developement, we created migrations. Independently of how many migrations have happened, upon the first use of a database hosted in SQL Server, we need to apply a script. This script can be generated via the `Packet Manager Console` in Visual Studio.

![Opening the NuGet Package Manager in Visual Studio](<img/VS - Open NuGet Package Manager Console.png>)

> Migration commandlets are provided by the NuGet `Microsoft.EntityFrameworkCore.Tools` package. Make sure it is installed.

> Make sure the correct project (having the migrations and context) is selected.

Generate the script using the command:

`Script-Migration`

> With `dotnet` command:
>
> `dotnet ef migrations script`

Possible switches are:

- `From {MigrationName}`
- `To {MigrationName}`

Sample:

```
PM> Script-Migration -From 20210706111054_NewCategoryType -To 20210724092019_AddCategoryToBook
Build started...
Build succeeded.
Connection string is not set through environment variable.
Trying to retrieve connection string from configuration file.
Loaded connection string from configuration.
```

It will open a new tab with the T-SQL script in it.

```
BEGIN TRANSACTION;
GO

CREATE TABLE [Books] (
    [Id] int NOT NULL IDENTITY,
    [Title] nvarchar(255) NOT NULL,
    [Author] nvarchar(255) NOT NULL,
    [Editor] nvarchar(max) NULL,
    [Edition] nvarchar(max) NULL,
    [NumberOfPages] int NULL,
    [FirstRelevantPageNumber] int NULL,
    [LastRelevantPageNumber] int NULL,
    CONSTRAINT [PK_Books] PRIMARY KEY ([Id])
);
GO

INSERT INTO [__EFMigrationsHistory] ([MigrationId], [ProductVersion])
VALUES (N'20210706114502_NewBookType', N'5.0.10');
GO

COMMIT;
GO

BEGIN TRANSACTION;
GO

ALTER TABLE [Books] ADD [CategoryId] int NULL;
GO

CREATE INDEX [IX_Books_CategoryId] ON [Books] ([CategoryId]);
GO

ALTER TABLE [Books] ADD CONSTRAINT [FK_Books_Categories_CategoryId] FOREIGN KEY ([CategoryId]) REFERENCES [Categories] ([Id]) ON DELETE NO ACTION;
GO

INSERT INTO [__EFMigrationsHistory] ([MigrationId], [ProductVersion])
VALUES (N'20210724092019_AddCategoryToBook', N'5.0.10');
GO

COMMIT;
GO
```

In SSMS open a new query window on the target database.

Copy the content from the script in VS and paste it in the query window in SSMS. Execute the query.

###  Create a user

We can set our own user to access the database with credentials.

For that matter under `Security ► Login` 
- Right-click on the folder
- Pick `New Login...`

![Context menu option to create a new login](<img/SSMS - Create New Login.png>)

A new dialog is displayed

![Empty dialog box for new login creation](<img/SSMS - New Login Dialog.png>)

Provide a username and password. Update other values as needed. Especially the default database if you foresee the login to be for a specific database as in our scenario.

![New values for the login](<img/SSMS - New Login Dialog Filled.png>)

On `Server Roles` make sure `public` is ticked.

On `User Mappings`

- In the top list, tick the database your login is related to
- In the bottom list, tick the relevant permissions:
  - db_datareader
  - db_datawriter
  - db_owner
  - db_public

![A user mapping to a related database](<img/SSMS - User Mappings.png>)

By default only Windows Authentication is enabled after installing SQL Server and SSMS.

We need to make sure the related setting is switched from `Windows Authentication mode` to `SQL Server and Windows Authentication mode`.

![Server Properties to enable logging to SQL Server with credentials](<img/SSMS - Server Properties.png>)

[Back to Top](#deploying-react-and-asp-net-core-to-iis)

##  Publishing React Code

In the general scenario, React code must be built using the command `npm run build`.

This will create the production build in the `/build` folder. This consist of ready to use HTML, CSS and JavaScript files that can simply be uploaded on the server.

Because we used the React project template from Visual Studio, it will take care of this step for us upon building the project. 

[Back to Top](#deploying-react-and-asp-net-core-to-iis)

##  Publishing .NET Core Web API

> Should you deploy to the default site provided by IIS, the location is `C:\inetpub\wwwroot` which needs an elevation. When elevation is needed, start Visual Studio as Administrator for that operation.

Right click on the project and use the `Publish...`  option.

Provide the information needed


For DB itself

`Data Source=PRODUCER\SQLEXPRESS;Initial Catalog=BL-Prod;Persist Security Info=False;User ID=YourUserId;Password=YourPassword;MultipleActiveResultSets=True;`

For EF Migragtions

`Data Source=PRODUCER\SQLEXPRESS;Initial Catalog=BL-Prod;Persist Security Info=False;User ID=YourUserId;Password=YourPassword;MultipleActiveResultSets=True;`

> We can setup user secrets in the project in development environment. It is located in `%AppData%\Roaming\Microsoft\UserSecrets\`. This prevents pushing sensitive data such as credentials or API Keys to a source control manager such as GIT.
> 
> To initialise user secrets to a project either:
> 
> - From Visual Studio right click the project in Solution Explorer. Pick `Manage User Secrets`
> 
> ![Creating User Secrets for a project in Visual Studio](<img/VS - Manage User Secrets.png>)
> 
> - Use the `dotnet user-secrets init` command.
> 
> This will create a new *GUID* which will be used to create a folder in the user profile. In there a `secrets.json` file contains the secret configuration keys. In the project, corresponding keys must be given empty values.
> 
> To add a Key in the User Secrets
> 
> `dotnet user-secrets set "My:Hierarchical:Data" "Hello World 123 !!!"`
> 
> The separator is the colon(`:`). More details on that [@MSDN](https://docs.microsoft.com/en-us/aspnet/core/fundamentals/configuration/?view=aspnetcore-6.0#hierarchical-configuration-data).

## Using the site

After all the steps above, the site should be up and running. It displays, allow to authenticate with Identity Server and upload images to the server.

Underneath is a screenshot showing a successful update of a book cover. To perform that operation:

- you need to be logged in to edit books
- the server shall accept to write uploaded images to disk too.

In other words it covers all the points mentioned earlier (display is implied and obvious).

Should you meet issues deploying, check the [Trouble-Shooting](#trouble-shooting) section.

![The site works as expected](<img/Web - Upload Cover.png>)

[Back to Top](#deploying-react-and-asp-net-core-to-iis)

## Trouble-shooting

### Error 500.19

On the first launch of the site we may get this error with an IIS look and feel.

![Error 500.19 sent by IIS](<img/IIS - 500.19 0x8007000d.png>)

The [Doc @MSDN](https://docs.microsoft.com/en-us/troubleshoot/developer/webapps/iis/health-diagnostic-performance/http-error-500-19-webpage#hresult-code-0x8007000d) states

> Cause
>
> This problem occurs because the ApplicationHost.config or Web.config file contains a malformed > or unidentified XML element. IIS can't identify the XML elements of the modules that are not
> installed. For example, IIS URL Rewrite module.

It suggest the configuration is malformed and needs to be checked, but the file has been automatically generated by VS Publish wizard and this is unlikely.

Instead we need to make sure the modules needed are installed. 

One way is simply to open the relevant file and check what modules it refers to.

```xml
<?xml version="1.0" encoding="utf-8"?>
<configuration>
  <location path="." inheritInChildApplications="false">
    <system.webServer>
      <handlers>
        <add name="aspNetCore" path="*" verb="*" modules="AspNetCoreModuleV2" resourceType="Unspecified" />
      </handlers>
      <aspNetCore processPath=".\BookLibrary.exe" stdoutLogEnabled="false" stdoutLogFile=".\logs\stdout" hostingModel="inprocess" />
    </system.webServer>
  </location>
</configuration>
<!--ProjectGuid: 36b0bd70-df0a-4de4-a563-08c348d4b5ca-->
```

> Note the `<add>` tag that mentions `AspNetCoreModuleV2` module. If the proper ASP .NET Hosting is not installed this error will come out. This module is part of the ASP .NET Core Runtime.
>
> See [Installing the Hosting Bundle for ASP .NET Core](#installing-the-hosting-bundle-for-asp-net-core) section.

### Error 500.30

This error message can have several causes so logs need to be investigated.

#### Writing files is denied

![Error 500.30 sent by ASP](<img/Web HTTP 500.30 Error.png>)

Inspecting the logs we may see interesting lines:

```
...
Could not start stdout file redirection to '.\logs\stdout' with application base 'C:\inetpub\wwwroot\'. create_directories: Accès refusé.: "C:\inetpub\wwwroot\.\logs".
...

...
System.UnauthorizedAccessException: Access to the path 'C:\inetpub\wwwroot\Uploads' is denied.
...
```

From there we see there is a permission issue.

So we need to change the rights step by step until it works.

> See [Allow writing to the server](#allow-writing-to-the-server) section.

### Missing Certificate

In such case we could see this kind of lines in the log.

```
...
An unhandled exception has occurred while executing the request.
  System.NullReferenceException: Object reference not set to an instance of an object.
    at Microsoft.Extensions.DependencyInjection.IdentityServerBuilderConfigurationExtensions.<>c.<AddSigningCredentials>b__10_2(IServiceProvider sp)
...
```

See [Extra Configuration for IdentityServer](#extra-configuration-for-identityserver) section.

[Back to Top](#deploying-react-and-asp-net-core-to-iis)

## Managing Self-Signed Certificates on Windows

Here are a few useful Powershell commands to manage certificates. It goes from creating a certificate to match specific server settings.
Because we cannot directly create the cert in the WebHosting Cert Store, the commands to move all or specific certs are added.

> Those commands need elevations. Run them in a PowerShell terminal as Administrator.

>  Certificate on Local Machine Manager (`certlm`) can be useful for some operations such as renewing certificates (sometimes the option is grayed out in IIS Manager) or exporting them to another format if needs be.
> 
> ![Web Hosting Cert Store in Certificate on Local Machine Manager](<img/CERTLM - Web Hosting Store.png>)

### Create a Self-Signed Certificate
`New-SelfSignedCertificate -DnsName template-react.local -CertStoreLocation cert:\LocalMachine\My -FriendlyName "React Template Test PS"`

### Move certificates to WebHosting Cert Store
`Get-ChildItem cert:\LocalMachine\My -SSLServerAuthentication | Move-Item -Destination cert:\LocalMachine\WebHosting`

### Move a specific certificate to WebHosting Cert Store
`Get-ChildItem cert:\LocalMachine\My\${THUMBPRINT} -SSLServerAuthentication | Move-Item -Destination cert:\LocalMachine\WebHosting`

[Back to Top](#deploying-react-and-asp-net-core-to-iis)