---
layout:     post
title:      Migrating Parse to Azure
date:       2016-06-10
author:     Michael Probst
summary:    
categories: Azure
thumbnail:  cogs
tags:
 - parse
 - azure
 - mongodb
---

## What is Parse? ##

[Parse][1] is a free and easy BAAS (Backend-as-a-Service) Provider founded by Facebook. 
You can easily manage your data, user and even files and access it via different APIs for platforms
like Xamarin, .NET, REST and Javascript. 
It's perfect to setup a full functional backup in no time.
<br/>
Long story short: It's a great tool .. unfortunately the hosted Parse Service will shutdown with the beginning of next year (January 2017).

**Now the good part is...**

Everything Parse has build is now available on Github. That is mainly the [Parse Server][2] itself, and the [Parse Dashboard][3] which provides a very nice view on your data.

**... and the bad?**

Now you need to migrate your data and host your own parse server ;-)<br/>
(Of course you can rebuild your complete backend and use other providers - but lets assume that you want to stay at the Parse ecosystem)
<br/>
Parse offers a lot of choices to migrate: https://github.com/ParsePlatform/Parse-Server/wiki#migrating-to-parse-server-from-hosted-parse


## Migration Mapping ##

The Parse-Server is based on node-js, data is stored on a MongoDB and all files will be provided by an AWS S3 bucket.

Based on this our migration will have the following setup:

1. Parse-Server: There is a fully managed Parse Server available on Azure.
1. Data: We will use an external MongoDB hoster
1. Files: The Azure Parse-Server is configured to use the Azure Blob Storage for Files

<i>Note: The manged Azure-Parse Server uses a new version of the Azure Document-DB for the parse data.
Unfortunately the new '[Document-DB with protocol support for MongoDB][4]' is still in preview - this is why I currently prefer an external mongoDB hoster</i>


## Step 1: Migrate the Parse-App data ##

At first - of course - we need an app to migrate. For this guide I created an standard parse app and added some classes and users:

![Parse-App](/images/parse_migration/ParseCore.png)

If you navigate through the Parse-Dashboard you will find an option to migrate your applications data to another database:

![Parse-App](/images/parse_migration/ParseMigrate.png)

The migration needs a valid connection string to an existing database. There are several MongoDb hosters in this world ([ObjectRocket](http://www.objectrocket.com) or in my case [mLab](http://www.mlab.com)).
For demo purposes I added a free (limited to 500MB) Mongo database with the help of mLab:

![Mlab](/images/parse_migration/mlabDBNoUser.png)
(Be sure to add a user for the DB - you will need it for the connection string)

Now you can take the connection string and use it for the migration step with Parse. <br/>
(Btw.. don't try it - I changed the password ;-))

Start Migration:
![Parse-Start-Migration](/images/parse_migration/ParseStartMigration.png)

Migration in progress:
![Parse-Migration-Process](/images/parse_migration/ParseMigrationProgress.png)


_Note: Parse creates a snapshot of its own database and copies the data to the new one. Its important to understand that parse still uses its own database. Only if you hit the following 'Finalize' button parse will change the connection string to your DB (and checks for missing data during the mirgation process_

Parse recommends to check if all the data has been migrated. You should double check this (via an database viewer or online) and finalize the migration.
From now on the hosted Parse-Server will use your own database.

_Note: Once you finalize a migration you **cannot** undo your action. The only thing you can do is changing the connection string - going back to a parse database is not possible_

![Parse-Finalize](/images/parse_migration/ParseFinalize.png)


## Step 2: Setup a Azure managed Parse-Server ##

After we migrated the database a important step has been done. A database containing parse data can be used by mulitple parse-servers. So even if we set up our own parse-server it is still possible that the 'old' hosted parse-server uses the same database.
This is important to know when it comes to compatibility problems (old / new clients).
<br/>
To host your own parse-server via Azure you just need to add the product "Parse Server on managed Azure services".
Choose a name and prefill "MasterKey" and "App Id" with your key from the old parse-dashboard.

The template for Azure will create a few items:

1. A Blob Storage which will be uses automatically by the azure parse server to store images / files
1. An Azure Notification Hub (not covered by this blog entry)
1. An Azure Parse-Server (based on the open source parse-server)
1. An DocumentDB with protocol support for mongoDB (not used by me because of preview status)

![Parse-Created-Parse](/images/parse_migration/AzureCreatedParse.png)


**Btw.. is the Parse-Server Code integrated into the Azure template!?**

No - fortunatley it is not. The Azure Parse-Server service uses a nice feature of Azure: _Deployment-Source_. <br/>
When navigating to "Application Settings" -> "Deployment Sources" you can see that Azure is actually doing nothing but getting all the parse-server sourcecode from the original Github page.
This also means, that you can always update your parse-server with the latest changes / bugfixes.

![Azure-Deployment-Source](/images/parse_migration/AzureDeploymentSource.png)

**.. back to our migration**

Now we need to connect the created Azure Parse-Server with our mongoDB containing all our parse data. 
This is were really cool part of Azure comes into action. You do not need to change any code - its just a ApplicationSetting.

Just navigate to your "migrated-parse" Service / Applictation Settings and change it with your connection string:

![Azure-Connection-String](/images/parse_migration/AzureConnectionString.png)

_Note: Please be sure that your connection string does **not** contains a trailing slash_

Now our new parse-server should be able to talk with the mongo database. Of course even our hosted parse server is still connected with the mongoDB.
The next step should be to actually _see_ our data via the parse-dashboard.

## Step 3: Use the Azure Parse-Dashboard ##

As I mentioned also the [Parse-Dashboard][3] is hosted on Github and can be forked / modified or just used. We do not want to use it by a custom installation - there is also a Azure Extension available which can be installed very easily.
The only thing you have to do is to install this extension.

Unfortunately the Parse-Dashboard on Azure is hardly documented. You need to navigtate to the Parse-App -> Tools -> Extensions -> Add -> 'ParseDashboard'.

![Azure-Parse-Dashboard-Installation](/images/parse_migration/AzureParseDashboardInstallation.png)

Once the Dashboard is installed you can just open it with the link:<br/>
https://YOUR-AZURE-WEBSITE-NAME.scm.azurewebsites.net/parse-dashboard/apps/

And taadaa :)

![Azure-Parse-Dashboard](/images/parse_migration/AzureParseDashboard.png)

## Step 4: Working with Files ##

The Azure Parse Server can also handle files (like images) and store it within its own storage (BLOB storage). 
Just navigate to the Dashboard on your Azure Parse-Server add a new Column named "ImageExample" (of type 'File'). If you upload a file you will notice that this file will be stored within your BLOB storage.
The Parse-Server code for Azure hase been configured to use this as the storage (again, you can modify this within the ApplicationSettings).

![Azure-File](/images/parse_migration/AzureFile.png)

Now - here is the interesting part.. Since Azure Parse-Server does not stores the file within the S3 Bucket of the hosted Parse-Server you can actually *see* the file entry in the old dashboard - but you cannot open it ;-) 
The File-Key is just not present on the S3 Bucket.

**..and why is this the intresting part?**

.. because it works the other way round :) <br/>
Now we upload a file over the hosted Parse-Dashboard. If you now try to open the file on the Azure Parse-Dashboard you will see something like this:

![Azure Missing Filekey](/images/parse_migration/AzureMissingFileKey.png)

Appearantly (see the URL) there is a file key missing - this is easy to fix.
Just copy the file key of your original parse app (App Settings -> Security & Keys -> File key) and add a new ApplicationSetting row within your Azure Parse-Server (within Azure Portal).

![Azure Add Filekey](/images/parse_migration/AzureAddFileKey.png)

Unfortunately you need to read and set the file key by yourself within the Cloud-Code (at least until today).

To achieve this just navigate to your Azure Parse-Server WebApp -> Tools -> Visual Studio Online (Preview) -> Open (or activate if not present).
With **Visual Studio Online** (which is basically the new Visual Studio Code) you can modify and write cloud code directly within the browser.

The file we want to edit is: 

{% highlight ruby %}

    /node_modules/parse-server-azure-config/index.js

{% endhighlight %}

Just read the added setting "FILE_KEY" and add it to the configuration:

{% highlight javascript %}

 var server = {
    appId: process.env.APP_ID || 'appId',
    masterKey: process.env.MASTER_KEY || 'masterKey',
    databaseURI: process.env.DATABASE_URI || 'mongodb://localhost:27017/dev',
    serverURL: (process.env.SERVER_URL || 'http://localhost:1337') + '/parse',
    cloud: siteRoot + '/cloud/main.js',
    logFolder: siteRoot + '/logs',
    fileKey: process.env.FILE_KEY, // <-- Add here the files key
    filesAdapter: () => {
      if (validate('storage', ['name', 'container', 'accessKey']))
        return new AzureStorageAdapter(storage.name, storage.container, storage);
      else {
        return new DefaultFilesAdapter();
      }
    },
    push: { 
      adapter: () => {
        if (validate('push', ['HubName', 'ConnectionString']))
          return AzurePushAdapter(push);
        else {
          return new DefaultPushAdapter();
        } 
      }
    },
    allowClientClassCreation: false,
    enableAnonymousUsers: false
  };

{% endhighlight %}

Thats all - now navigate back to your Azure Parse-Dashboard and you will see that you can open the file which was originally uploaded by the hosted Parse-Dashboard :)

![Azure Working File](/images/parse_migration/AzureWorkingFile.png)

## Summary ##

We have learned to..

1. Migrate Parse App Data to an external DB
1. Set up a fully managed Azure Parse-Server
1. Set up the Parse-Dashboard for our Azure Server
1. What we can do with files


I hope you enjoyed it :)

Michael


[1]: http://parse.com
[2]: https://github.com/ParsePlatform/parse-server
[3]: https://github.com/ParsePlatform/parse-dashboard
[4]: https://azure.microsoft.com/de-de/documentation/articles/documentdb-protocol-mongodb/
