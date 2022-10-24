---
title: SQL Version Control in AzureDevOps
categories:
  - blog
tags:
  - Jekyll
  - update
toc: true

---
<meta property="og:url" content="{{site.url}}/blog/AzureDevOpsStoredProcedures/"/>
<meta property="og:title" content="SQL Version Control Using Azure DevOps"/>
<meta property="og:site_name" content="The Schwartz Pub"/>
<meta property="og:image" content="{{site.url}}/assets/images/SqlRepository.png"/>

When you're working with code, it is important to keep a history of changes that you can refer to whenever an unexpected breaking change is introduced that needs to be rolled back.  This also applies to SQL where changes to Stored Procedures, Views, and other objects in the database can result in unintended side effects down the road.  If we can place our SQL objects into git, then we can keep track of change history, approve changes to our production environment, and run tests to ensure the changes will not immediately break our workflows.

In my case, it made sense to create a repository in Azure DevOps that would hold my SQL objects.  I didn't want to have to remind myself each time I made a change to a Stored Procedure that Wealso needed to make the change in the repo to store it, so I decided I would test my changes and then commit them to my repository where pipelines would pick up the changes and deploy them into production.  Using Azure DevOps pipelines, self hosted agents, and group managed service accounts (gMSA) this was a painless process.

Another option would be using SSDT in Visual Studio to gain version control of your SQL objects. The method I use in this article worked better for my purposes in a set of large, vendor managed databases where I add my own customizations to a particular schema(s). Many people will prefer to use the SSDT approach, but this article just provides another option.

## Create Group Managed Service Account
To start we want to create a [Group Managed Service Account (gMSA)](https://learn.microsoft.com/en-us/windows-server/security/group-managed-service-accounts/getting-started-with-group-managed-service-accounts) that will be used to run the Azure Agent.  This account will have just the permissions it needs in order to run the deployment, including creating and/or altering the tables and views in the database schema in production.

You'll want to create a group of computer objects that will utilize this gMSA.

![Server Group Screenshot](/assets/images/ServerGroupScreenshot.png)

To create the account, we will open up a PowerShell window with elevated AD credentials and run:

```powershell
New-ADServiceAccount AzureDevOps -DNSHostName AzureDevOps.sp.local -PrincipalsAllowedToRetrieveManagedPassword Servers -KerberosencryptionType AES256 -ManagedPasswordIntervalInDays 30
```

Now we have a new gMSA we can use it for setting up the rest of our workflow.

![gMSA Screenshot](/assets/images/gMSAScreenshot.png)

## Add gMSA to SQL server 
Add the gMSA to your SQL server and grant it the appropriate permissions to perform the workflows needed.

In our case, we will add the `db_reader` role.

Add a new user:

![Add gMSA to SQL 1](/assets/images/AddgMSAtoSQL_1.png)

Set Permissions:

![Add gMSA to SQL 2](/assets/images/AddgMSAtoSQL_2.png)

## Add gMSA permissions to create and alter Stored Procedures and Schema
Now we need to add the permissions on the database that will allow our gMSA to create and alter stored procedures on the schema.  The first permission `CREATE PROCEDURE` allows the account to create Stored Procedures, but does not tell the database where the account is allowed to create those objects, so we must also set the `ALTER SCHEMA` permission, which tells the database *where* the account is allowed to make those changes. We can do this by running the following query:

```sql
GRANT CREATE PROCEDURE TO [SP\AzureDevOps$];
GRANT ALTER ON SCHEMA::AppUserCustom TO [SP\AzureDevOps$];
```

## Create Azure DevOps Repository
We need a repository to store our files in within Azure DevOps.  We will not be going into the various security and management considerations that need to be applied when managing an Azure DevOps environment in this blog post, but there are a lot of resources available to do so.

Sign into your Azure DevOps environment and create a new project:

![New ADO Project Screenshot](/assets/images/CreateADOProject.png)

Initialize the project for use:

![Initialize ADO Project](/assets/images/InitializeADORepository.png)

## Clone Repository to your local machine
Now we want to clone the repository to our local machine so we can add folders, code, and other assets to the repository.  Start by clicking the clone button.

![Clone Repo Screenshot](/assets/images/CloneButton.png)

Copy the URL for your respository and then in VSCode, use the URL to clone the resository to your machine.  Any location should be fine.

![Copy URL VSCode Screenshot](/assets/images/CloneRepo.png)

VSCode:

![Clone Repo VSCode Screenshot](/assets/images/CloneVSCode.png)

## Create Folders for Stored Procedures and pipelines
Next we are going to create a couple of folders.

We will start with a folder called .pipelines that will contain the YAML files for our deployments.  You can place all of your YAML files in this folder, or if you choose you can create subfolders for different portions of your environment.

Additionally, we will create a folder called 'App01' that we will place our SQL files into.

![Simple Folders Screenshot](/assets/images/SimpleRepoFolders.png)

Now we have a basic scaffolding to use for storing our code and pipeline definitions.

## Create Azure DevOps Agent Pool
The next step isn't strictly necessary, however I prefer to use a self hosted agent for my builds and deployments because it allows for better control of security, connectivity to on-premise resources, and other advantages.  You will want to follow best pratices when deploying a self-hosted agent to ensure security of your environment. [Guidance from Microsoft on creating self-hosted agents can be found here](https://learn.microsoft.com/en-us/azure/devops/pipelines/agents/v2-windows?view=azure-devops).

We'll start by setting up an Agent Pool and creating a self hosted agent on one of our local servers.

First, go to `Project Settings > Agent Pools, and click 'Add pool'`:

![Create Agent Pool Screenshot](/assets/images/CreateAgentPool.png)

Open the new pool and select `'New agent'`. Download the agent and follow the instructions to add the new agent to your pool.

Download the agent and extracted the contents of the zip file into a temp folder.

In PowerShell we will run the config script to set up the agent:

```powershell
.\config.cmd
```

Fill out the various fields, in my case I chose to authenticate using a Personal Access Token.  To acquire a Personal Access Token, in Azure DevOps click on the gear icon in the upper right and select `Personal Access Tokens`.

![Create Personal Access Token](/assets/images/PersonalAccessToken.png)

Then fill out the fields to generate an access token, be sure to save this in a safe location because you will have to generate a new one if you lose it.

![Generate Personal Access Token](/assets/images/PersonalAccessTokenCreate.png)

Now you can fill out the rest of the prompts. You will need to provide the name of the Agent Pool we created in the previous step, as well as the name you want to give this agent.  You can leave the remaining fields s default. In the next step we will update the service to run using the gMSA we created earlier.

![Configure Agent Powershell](/assets/images/ConfigureAgentPosh.png)

## Add Azure DevOps Agent with gMSA 

Open Services on your agent server and change the Logon User to your gMSA.  Leave the password blank, the server will get this information from AD itself.

![Configure Logon As](/assets/images/ServiceLogonAs.png)

## Export Stored Procedures to local repository

Next, lets export our stored procedures to our repository.  This can be done pretty quickly using SSMS -- right click on your database and select Generate Scripts...

![Export Stored Procedures](/assets/images/ChooseObjectsExport.png)

Then save each object as a separate file in your repo.

![Save Stored Procedures](/assets/images/SaveScriptsToLocation.png)

## Modify Stored Procedures to ALTER instead of CREATE
Since all of our exported procedures will have CREATE scripts, we will want to change these to ALTER, otherwise our pipelines later will fail since these procedures will already exist in the database. 

You can use FIND/REPLACE in VSCode or use PowerShell to do the same.

![Create To Alter](/assets/images/CreateToAlter.png)

## Populate repository with Stored Procedures

We are now ready to commit our stored procedure scripts into Azure DevOps.  Once we have initialized these files we will be ready to set up our Pipelines.

![Commit To AzureDevOps](/assets/images/CommitToRepo.png)

## Create Template YAML using DACPAC task

The next step is to create our YAML template that will be used to create our pipeline that deploys our stored procedures into our database.  This is a pretty simple pipeline since it will be using our self hosted agent, running with our gMSA that now has permission to create and alter procedures in our database.

The first part of our template will set up our trigger. Since this is a monorepo, we don't want to deploy all of our files every time we commit to the repository.  Using paths in our trigger, we can isolate each of our pipelines to the stored procedure that is being updated.

```yaml
trigger:
  paths:
    include:
      - App01/AppUserCustom.pPerson.StoredProcedure.sql
```

We'll also add a variable to let the DACPAC task know the name of the file to be deployed during the pipeline run.

```yaml
variables:
- name: buildFile
  value: 'AppUserCustom.pPerson.StoredProcedure.sql'
```

We want to run this pipeline on our self hosted agent, so we will add the pool containing our self hosted agent. If you have multiple agents with tags, this is where you would define the appropriate tags or agents.  Since we only have the single agent in the pool, we just need to define the pool.

```yaml
pool:
 name: DB01
```

Finally, we will add the DACPAC task to our steps.  We will use the SQL Query task type and reference our sql file containing our stored procedure definition.  This will also reference our database server name, the name of our database, and authentication scheme (windows authentication) so that our agent knows how to push the deployment to the server.

`$(System.DefaultWOrkingDirectory)` tells Azure DevOps to reference the working directory on our agent machine, where the repository files will be checked out to.

`$(buildFile)` references our variable we defined earlier, that contains the name of our stored procedure definition file.

```yaml
steps:
- task: SqlDacpacDeploymentOnMachineGroup@0
  inputs:
    TaskType: 'sqlQuery'
    SqlFile: '$(System.DefaultWorkingDirectory)\App01\$(buildFile)'
    ServerName: 'DB01'
    DatabaseName: 'AdventureWorks2019'
    AuthScheme: 'windowsAuthentication'
```

With just these few components, we have all we need to create a pipeline that can deploy our stored procedure to our database whenever we make updates and changes to the procedure and commit them to the repository.  The full contents of your YAML file should look similar to this:

```yaml
trigger:
  paths:
    include:
      - App01/AppUserCustom.pPerson.StoredProcedure.sql

variables:
- name: buildFile
  value: 'AppUserCustom.pPerson.StoredProcedure.sql'

pool:
 name: DB01

steps:
- task: SqlDacpacDeploymentOnMachineGroup@0
  inputs:
    TaskType: 'sqlQuery'
    SqlFile: '$(System.DefaultWorkingDirectory)\App01\$(buildFile)'
    ServerName: 'DB01'
    DatabaseName: 'AdventureWorks2019'
    AuthScheme: 'windowsAuthentication'
```

## Powershell to generate YAML for each Stored Procedure

Even though it was pretty simple to create this YAML file, what if we have several or even hundreds of stored procedures that we want to include?  Using a little bit of PowerShell and a 'template' YAML, we can quickly duplicate our pipeline files.

Lets start by creating a temporary base.yml file that will make it easy to do a find and replace.  We will save this to C:\temp for now.

Change your file to include the `paths`, `pool name`, database `ServerName` and `DatabaseName` that you will use in your environment.

```yaml
trigger:
  paths:
    include:
      - App01/REPLACE_FILENAME

variables:
- name: buildFile
  value: 'REPLACE_FILENAME'

pool:
 name: DB01

steps:
- task: SqlDacpacDeploymentOnMachineGroup@0
  inputs:
    TaskType: 'sqlQuery'
    SqlFile: '$(System.DefaultWorkingDirectory)\App01\$(buildFile)'
    ServerName: 'DB01'
    DatabaseName: 'AdventureWorks'
    AuthScheme: 'windowsAuthentication'
```

Next, lets use some PowerShell to ingest this yml file, get a list of our stored procedure files from our local copy of the repository (C:\Repository\App01\), and replace `REPLACE_FILENAME` with the right data.  Then we will output all of these files into our .pipelines directory in our local repository.

We will keep consistent naming of our deployment files by extracting the stored procedure name from the sql file.

```powershell
$procs = Get-ChildItem "C:\Repository\App01\"
$yaml = Get-Content C:\temp\base.yml

foreach ($proc in $procs) {
    $yamlname = $proc.Name.Split(".")[1]
    $yaml -replace "REPLACE_FILENAME",$proc | Out-File "C:\Repository\.pipelines\App01\$($yamlname)-Deploy.yml"
}
```

Now lets commit these files to our repository so we can generate the pipelines using Az CLI.  If you followed along, you should now have your YAML files in your repo:

![Repo Yaml Files](/assets/images/RepositoryYaml.png)

## Use AZ CLI to generate pipelines from YAMLs

We now have all of the yml files we need in our repository but we need to create the pipelines that use these files.  We could go through these one by one in the Azure DevOps portal to create pipelines out of existing yml files, or we could take the easy route and loop over them in PowerShell and use Az CLI to generate the pipelines.

This small script takes all of our yml files from our local repo and loops over each one, creating a new pipeline from each one.  You will need to replace the various fields in your script with the relevant locations and names from your environment.  Don't forget to log into Azure first using `az login` or `az devops login`:

|**organization**|This is the url of your Azure DevOps environment.|
|**repository**|This is the name of your repository.|
|**project**|This is the name of the project that contains your repository.|
|**branch**|This is the name of the branch that will contain your pipelines.|
|**folder-path**|This is the path in your pipeline hierarchy that will store your pipelines|
|**yaml-path**|This is the path in your repository that contains your pipeline yml files|

```powershell
$pipelines = Get-ChildItem C:\Repository\.pipelines\App01\

foreach ($p in $pipelines){
    az pipelines create --name $p.Name --description "Pipeline for $p.Name" --organization "https://dev.azure.com/schwartzpub/" --repository "SQL Stored Procedures" --project "SQL Stored Procedures" --branch "main" --folder-path ".app01-pipelines" --yaml-path ".pipelines\App01\$p" --skip-run --repository-type tfsgit
}
```

*You will need to install the Az CLI application, which can be found [here](https://learn.microsoft.com/en-us/cli/azure/install-azure-cli-windows?tabs=azure-cli).*

Now that we have our pipelines, we are ready to test!

![Pipeline Yaml Directory](/assets/images/PipelineYaml.png)

## Test pull request of Stored Procedure update

If everything is configured correclty, we can now test out our new pipelines by making some changes to one of our stored procedures and committing those changes to the Azure DevOps repository.  We will add a comment to one of the Stored Procedures we exported earlier, and commit that change to the repo.

![Test Comment](/assets/images/TestComment.png)

We see that the pipeline begins to run shortly after the commit is complete, and can watch the job progress through to completion.

![Pipeline Success](/assets/images/PipelineSuccess.png)

Now when we check the Stored Procedure on the server, we see that the comment is there, and the updated stored procedure has been successfully deployed.

![Updated Stored Procedure](/assets/images/UpdatedStoredProcedure.png)

## Conclusion

From here, you can set up additional customizations, tests, and rules surrounding your workflow.  Adding newly created Stored Procedures, views, and other objects is just a matter of adding the new SQL files and a new pipeline following the same steps above.  If your database moves to a new server, it's a quick process of updating the pipelines to point deployments to the new server and adding the gMSA account with permissions to add the objects to the schema.  I hope you found this article helpful!