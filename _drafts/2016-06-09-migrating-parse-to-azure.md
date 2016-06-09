---
layout:     post
title:      Migrating Parse to Azure
date:       2016-06-09
author:     Michael Probst
summary:    
categories: Azure
thumbnail:  cogs
tags:
 - parse
 - azure
 - mongodb
---

<h2> What is Parse?</h2>

[Parse][1] is a free and easy BAAS (Backend-as-a-Service) Provider founded by Facebook. 
You can easily manage your data, user and even files and access it via different APIs for platforms
like Xamarin, .NET, REST and Javascript. 
It's perfect to setup a full functional backup in no time.

Long story short: It's a great tool .. unfortunately the hosted Parse Service will shutdown with the beginning of next year (January 2017).

Now the good part is:

Everything Parse has build is now available on Github. That is mainly the [Parse Server][2] itself, and the [Parse Dashboard][3] which provides a very nice view on your data.

And the bad?

Now you need to migrate your data and host your own parse server ;-)
(Of course you can rebuild your complete backend and use other providers - but lets assume that you want to stay at the Parse ecosystem)

Parse offers a lot of choices to migrate: https://github.com/ParsePlatform/Parse-Server/wiki#migrating-to-parse-server-from-hosted-parse


<h2>Migration Mapping</h2>

The Parse-Server is based on node-js, data is stored on a MongoDB and all files will be provided by an AWS S3 bucket.

Based on this our migration will have the following setup:
1. Parse-Server: There is a fully managed Parse Server available on Azure.
1. Data: We will use an external MongoDB hoster
1. Files: The Azure Parse-Server is configured to use the Azure Blob Storage for Files

<i>Note: The manged Azure-Parse Server uses a new version of the Azure Document-DB for the parse data.
Unfortunately the new '[Document-DB with protocol support for MongoDB][4]' is still in preview - this is why I currently prefer an external mongoDB hoster</i>


<h2>Step 1: Migrate the Parse-App data</h2>

<h2>Step 2: Setup a Azure managed Parse-Server</h2>

<h2>Step 3: Use the Azure Parse-Dashboard </h2>

<h2>Step 4: tbd</h2>

[1] http://parse.com
[2] https://github.com/ParsePlatform/parse-server
[3] https://github.com/ParsePlatform/parse-dashboard
[4] https://azure.microsoft.com/de-de/documentation/articles/documentdb-protocol-mongodb/
