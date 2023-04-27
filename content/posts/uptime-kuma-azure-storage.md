---
title: "Uptime Kuma on Azure with Database Storage"
date: 2023-04-27T11:00:00+02:00
tags:
  - Azure
categories:
  - Devops
---

# Uptime Kuma on Azure with Database Storage
[Uptime Kuma](https://github.com/louislam/uptime-kuma) is a self-hosted monitoring tool that provides you with a status 
page for your underlying services. Uptime Kuma uses a sqlite database to store the data, so the file resides on the file
system together with the solution. 

The solution can be deployed to Azure using the [Azure App Service](https://azure.microsoft.com/en-us/services/app-service/)
and backed with a file storage account for the database. 

To achieve this we mount the file share from the storage account into the App Service, this way, any files that are written
to the mount are actually written to the file share. 

## The dreaded SQLITE_BUSY error
If you receive an error page when you first access the application, check the storage account, file share. Does it contain a
kuma.db and a error.log file? Does the error.log file contain ``SQLITE_BUSY: database is locked``?

When the application first starts, it initialises a sqllite DB. It takes long enough for azure to think the application 
is not responding, so it kills the application and restarts it, in the middle of this database creation. As a result, the
DB is now in a locked state.

To fix this:
1. Stop the application
2. Delete the error.log and kuma.db files. If it says the file is in use, then the application hasnt shut down yet.
3. clone or download the initial database files from the repo https://github.com/zamedic/uptime-kuma-initial-db
3. Copy the 3 files from this projects database folder into the file share
4. start the application again 

