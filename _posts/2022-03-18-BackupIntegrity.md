---
layout: post
title: Automating Basic Veeam Backup Integrity Checking - Part 1
toc: true
excerpt_separator: <!--more-->
---
## The Problem
In many companies, it is important (if not required) to perform regular testing of backups to ensure recoverability and integrity of data.  Often times this is a fairly manual process in which you recover partial or full datasets and run some checks to ensure the data is whole.  One tool that is useful is to have automated regular checks of your backups that can provide a minimum level of integrity checking to ensure that the data, for the most part, is complete and readable.  

This problem becomes exponentially more time consuming when working with tens, hundreds, or thousands of file server backups, database backups, etc.  It would be nice to get a report each month that lets you know right away if there is a problem with data integrity that requires further investigation.

## The Solution
For my purposes, I wanted to have a quick and automated way to check my fileserver and database backups were whole.  By making use of the built in PowerShell cmdlets provided by Veeam on machines with the [Veeam Backup & Replication Console installed](), we can easily create a scheduled task that will do some basic checks.

### The Setup
I've considered using external configuration files for scripts that require manual configuration, but I have yet to find a solution that satisfies everything I'd like from a configuration file.  For this script, I opted to include the configuration parameters in the form of an array of [PSCustomObjects]().  There are a number of parameters I will need to know in order to check each item.

#### Integrity Testing Definitions

```powershell
#File definitions
$RestoreFileDefs = @(
	[pscustomobject]@{ 
        Name = 'FILESERVER01'; 
        BackupName = 'FILESERVER01'; 
        RestorePoint = 'FILESERVER01_RP'; 
        DestinationPath = 'C:\Automation\BackupRecovery\'; 
        RestorePath = '\DFS\SHARED\IT\Test Integrity\'; 
        ProductionPath = '\\domain.com\share\IT\Test Integrity\'; 
        FileIndex = ''; 
    }
	[pscustomobject]@{ 
        Name = 'FILESERVER02'; 
        BackupName = 'FILESERVER02'; 
        RestorePoint = 'FILESERVER02_RP'; 
        DestinationPath = 'C:\Automation\BackupRecovery\'; 
        RestorePath = '\Program Files (x86)\Test Integrity\'; 
        ProductionPath = '\\domain.com\share\Test Integrity\'; 
        FileIndex = ''; 
    }
)
```
Here, I have defined two backups of fileservers.  I need to provide a number of parameters that will be used by the Veeam PowerShell cmdlets when restoring and verifying the data.

> BackupName
This is the name of the backup in Veeam.

> RestorePoint
This is the name of the restore point in the Veeam backup.

> DestinationPath
This is the destination where I will recover the data from backup that I want to test.

> RestorePath
This is the path of the files inside of the Veaam restore point that I want to test.

> ProductionPath
This is the path to the same files in production, to be used when testing integrity of the restored files.

---

```powershell
#Database definitions
$RestoreDBDefs = @(
	[pscustomobject]@{ 
        Name = 'App01 Database'; 
        BackupName = 'SQL01'; 
        Database = 'App01'; 
        StagingServer = 'BACKUP01'; 
        DestinationPath = '\\backup01\automation\BackupRecovery\'; }
	[pscustomobject]@{ 
        Name = 'App02 Database'; 
        BackupName = 'SQL02'; 
        Database = 'App02'; 
        StagingServer = 'BACKUP01'; 
        DestinationPath = '\\backup01\automation\BackupRecovery\'; }
)
```
Here I have done a similar configuration for databases.  For this, we will supply another set of parameters that will be used with the Veeam cmdlets when recovering our data.

> BackupName
This is the name of the backup in Veeam.

> Database
This is the database to be recovered.

> StagingServer
This is the server we will use to stage the database during recovery.

> Destinationpath
This is the path we will store our recovered database.

---