---
title: Veeam Backup Integrity Checking - Part 1
categories:
  - blog
tags:
  - Jekyll
  - update
toc: true

---
## The Problem
In many companies, it is important (if not required) to perform regular testing of backups to ensure recoverability and integrity of data.  Often times this is a fairly manual process in which you recover partial or full datasets and run some checks to ensure the data is whole.  One tool that is useful is to have automated regular checks of your backups that can provide a minimum level of integrity checking to ensure that the data, for the most part, is complete and readable.  

This problem becomes exponentially more time consuming when working with tens, hundreds, or thousands of file server backups, database backups, etc.  It would be nice to get a report each month that lets you know right away if there is a problem with data integrity that requires further investigation.

## The Solution
For my purposes, I wanted to have a quick and automated way to check my fileserver and database backups were whole.  By making use of the built in PowerShell cmdlets provided by Veeam on machines with the [Veeam Backup & Replication Console installed](https://helpcenter.veeam.com/docs/backup/hyperv/install_console.html?ver=110), we can easily create a scheduled task that will do some basic checks.

### The Setup
I've considered using external configuration files for scripts that require manual configuration, but I have yet to find a solution that satisfies everything I'd like from a configuration file.  For this script, I opted to include the configuration parameters in the form of an array of [PSCustomObjects](https://docs.microsoft.com/en-us/powershell/scripting/learn/deep-dives/everything-about-pscustomobject?view=powershell-7.2).  There are a number of parameters I will need to know in order to check each item.

#### Integrity Testing Definitions

```powershell
#File definitions
$RestoreFileDefs = @(
  [pscustomobject]@{ 
        Name = 'FILESERVER01'
        BackupName = 'FILESERVER01'
        RestorePoint = 'FILESERVER01_RP'
        DestinationPath = 'C:\Automation\BackupRecovery\'
        RestorePath = '\DFS\SHARED\IT\Test Integrity\'
        ProductionPath = '\\domain.com\share\IT\Test Integrity\'
        FileIndex = ''; 
    }
  [pscustomobject]@{ 
        Name = 'FILESERVER02'
        BackupName = 'FILESERVER02'
        RestorePoint = 'FILESERVER02_RP'
        DestinationPath = 'C:\Automation\BackupRecovery\'
        RestorePath = '\Program Files (x86)\Test Integrity\' 
        ProductionPath = '\\domain.com\share\Test Integrity\'
        FileIndex = ''
    }
)
```
Here, I have defined two backups of fileservers.  I need to provide a number of parameters that will be used by the Veeam PowerShell cmdlets when restoring and verifying the data.

|**BackupName**|This is the name of the backup in Veeam.|
|**RestorePoint**|This is the name of the restore point in the Veeam backup.|
|**DestinationPath**|This is the destination where I will recover the data from backup that I want to test.|
|**RestorePath**|This is the path of the files inside of the Veaam restore point that I want to test.|
|**ProductionPath**|This is the path to the same files in production, to be used when testing integrity of the restored files.|

---

```powershell
#Database definitions
$RestoreDBDefs = @(
  [pscustomobject]@{ 
        Name = 'App01 Database'
        BackupName = 'SQL01'
        Database = 'App01'
        StagingServer = 'BACKUP01'
        DestinationPath = '\\backup01\automation\BackupRecovery\'
    }
  [pscustomobject]@{ 
        Name = 'App02 Database'
        BackupName = 'SQL02'
        Database = 'App02'
        StagingServer = 'BACKUP01'
        DestinationPath = '\\backup01\automation\BackupRecovery\'
    }
)
```
Here I have done a similar configuration for databases.  For this, we will supply another set of parameters that will be used with the Veeam cmdlets when recovering our data.

|**BackupName**|This is the name of the backup in Veeam.|
|**Database**|This is the database to be recovered.|
|**StagingServer**|This is the server we will use to stage the database during recovery.|
|**Destinationpath**|This is the path we will store our recovered database.|

---

#### Connecting to Veeam

Next I will need to connect to Veeam and return the connection object for use wit subsequent requests.  I will run this script under the context of a user with Veeam access so that I do not need to pass credentials when connecting to Veeam.

I start by creating a function `Connect-BackupServer` which basically wraps the `Connect-VBRServer` command and returns the connection.  Using this function, I can also test for a successful connection to the VBR Server and handle an error if the connection fails.

You will notice I also use a function called `LogWrite` which is a [helper function]() I created for use in many of my scripts to handle logging to file.  You could also use the [built-in transcript feature](https://docs.microsoft.com/en-us/powershell/scripting/windows-powershell/wmf/whats-new/script-logging?view=powershell-7.2) if you wanted a quick way to achieve the same effect.

```powershell
function Connect-BackupServer
{
  param
  (
    [string]$BackupServer = $BackupServer
  )
  LogWrite "[INFO] Connecting to $($BackupServer)"
  try
  {
    #Connect to Veeam Backup and Replication server
    $conn = Connect-VBRServer -Server $BackupServer
  }
  catch
  {
    LogWrite "[ERROR] Connecting to $($BackupServer)..." $_.InvocationInfo.ScriptLineNumber $_
    exit
  }
  
  return $conn
}
```

#### Mounting the Restore Point

For my purposes, I have two main types of Veeam backups that I want to validate.  The first is File System -- basically Windows file shares.  The second is Databases -- in my case these are all MSSQL database backups.  In order to run the validation I will need to mount the restore points to access the backups for testing.  To do this I will use either [Get-VBRBackup](https://helpcenter.veeam.com/docs/backup/powershell/get-vbrbackup.html?ver=110) and [Start-VBRWindowsFileRestore](https://helpcenter.veeam.com/docs/backup/powershell/start-vbrwindowsfilerestore.html?ver=110) for file shares, or [Get-VBRApplicationRestorePoint](https://helpcenter.veeam.com/docs/backup/powershell/get-vbrapplicationrestorepoint.html?ver=110) and [Start-VESQLRestoreSession](https://helpcenter.veeam.com/docs/backup/explorers_powershell/start-vesqlrestoresession.html?ver=110) for databases.

Using a simple function `Mount-RestorePoint` I can call a switch to determine which time of restore point I would like to mount, and pass in the appropriate paramaters from my configuration objects. 

```powershell
function Mount-RestorePoint
{
  param (
    [string]$BackupName = $BackupName,
    [string]$RestorePoint = $RestorePoint,
    [string]$BackupType = $BackupType
  )
  LogWrite "[INFO] Mounting Restore Point for $($BackupName)"
  
  switch ($BackupType) {
    File {
      try
      {
        #Get latest restore point for the backup and start a Windows File Restore session.
        #We will keep the restore session alive until we have copied the required files to the Temp directory for hash comparison.
        $RestoreSession = Get-VBRBackup -Name $BackupName | Get-VBRRestorePoint -Name $RestorePoint | Sort-Object -Property CreationTime -Descending | Select-Object -First 1 | Start-VBRWindowsFileRestore -Reason "Backup Recovery Testing"
        return $RestoreSession
      }
      catch
      {
        LogWrite "[ERROR] Mounting restore point for $($BackupName)..." $_.InvocationInfo.ScriptLineNumber $_
        
        #If an error occurs while mounting the restore session, discard the server connection.
        Disconnect-VBRServer
        throw
      }
    }
    DB {
      try
      {
        #Get latest restore point for the backup and start a SQL Restore session.
        #We will keep the restore session alive until we have expoted the database to the Temp directory for verification.
        $RestoreSession = Get-VBRApplicationRestorePoint -SQL -Name $BackupName | Sort -Property CreationTime -Descending | select -First 1 | Start-VESQLRestoreSession
        return $RestoreSession
      }
      catch
      {
        LogWrite "[ERROR] Mounting restore point for $($BackupName)..." $_.InvocationInfo.ScriptLineNumber $_
        
        #If an error occurs while mounting the restore session, discard the server connection.
        Disconnect-VBRServer
        throw
      }
    }
    default {
      #<code>
    }
  }
}
```

|**BackupName**|This is the name of the backup in Veeam.|
|**RestorePoint**|This is the restore point to be recovered.|
|**BackupType**|This is the type of recovery, either `File` or `DB`.|

If the restore type is `File` we will return the session using this command:
```powershell
$RestoreSession = Get-VBRBackup -Name $BackupName | Get-VBRRestorePoint -Name $RestorePoint | Sort-Object -Property CreationTime -Descending | Select-Object -First 1 | Start-VBRWindowsFileRestore -Reason "Backup Recovery Testing"
```
This command will get the backup (Get-VBRBackup), get the restore point (Get-VBRRestorePoint), select the most recent restore point, and then start a restore session (Start-VBRWindowsFIleRestore).

If the restore type is 'DB' we will return the session using this command:
```powershell
$RestoreSession = Get-VBRApplicationRestorePoint -SQL -Name $BackupName | Sort -Property CreationTime -Descending | select -First 1 | Start-VESQLRestoreSession
```
This command will get the restorepoint from the backup (Get-VBRApplicationRestorePoint), select the most recent restore point, and then start a restore session (Start-VESQLRestoreSession).

In each case, the appropriate session is stored in a variable and returned to the script.

In Part 2 I will discuss how I validate each of these restore points and send a report with the results to the appropriate recipients.