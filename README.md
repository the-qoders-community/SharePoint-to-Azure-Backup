# Easy SharePoint to Azure Backup
A low-code solution to let end-users backup SharePoint content to Azure storage

[This is the first part of the solution that does the heavy lifting of backing up SharePoint files to Azure blob storage. In the second part this solution will be extended with a SharePoint Site Design/Site Script that will automatically add a newly created Microsoft Team/SharePoint Site to the backup process. Also a service bus will then be added to the architecture]

Simple Azure serverless SharePoint backup uses a logic app and storage account to backup SharePoint files. The solution only backups files that are stored in the Shared Documents library of your site (which is ideal if you use Teams). The solution uses the 'Delta' method of the Graph files api. Only newly added or edited files are updated (for more information see: https://docs.microsoft.com/en-us/graph/api/driveitem-delta?view=graph-rest-1.0&tabs=http).

# Installation

Start by creating an Azure Active Directory application. Follow the steps in this blog  https://www.lee-ford.co.uk/using-flow-with-graph-api/ and make sure your application (so not delegated!) has 'Sites.ReadWrite.All' Graph Permissions.

<img src="https://github.com/Robert1976/SharePointBackup/blob/master/images/graph.PNG" width="600" >

Make sure you save your TenantID, Your applications ClientID and your applications ClientSecret. You need them further on.

Now create a new storage account in Azure. Create a new blob storage container (this container will hold the SharePoint files) and a new table (this will hold references to the sites that will be backed up) and a new Queue. I named my blob 'sharepointfiles', my table 'sharepointsites' and my queue 'sharepointazurebackup'. Make sure you remember the name of your storage account and copy one of your access keys (key1 or key2) to a temporary note. Your blob should be a private blob!

The next step is to import the first logic app: 'QodersAddSiteToBackUp'. This is a logic app with an HTTP endpoint. The endpoint is called by the SharePoint site design/site script. The template is saved within this repository with title 'QodersAddSiteToBackUp.json' It is a ARM template and you can import it in the same way as importing a Power Automate template. Navigate to https://docs.microsoft.com/en-us/azure/logic-apps/export-from-microsoft-flow-logic-app-template#deploy-template-by-using-the-azure-portal and follow step 1-9. At step 4 you should upload 'QodersAddSiteToBackUp.json'.

In step 7 use the storage account name, and key1 or key2 that you have created and saved previously in the yellow marked input fields.

<img src="https://github.com/the-qoders-community/SharePoint-to-Azure-Backup/blob/master/Images/QodersSharePointBackup.PNG" width="600" >

Followed by configuring a Sitescript / Sitedesign -> Add-SPOSiteDesign -Title "Azure Backup" -Description "Add site to daily Azure Backup" -SiteScripts ab81b263-1f89-4c3d-9a2e-0d7479729787 -WebTemplate 0

https://docs.microsoft.com/en-us/sharepoint/dev/declarative-customization/site-design-trigger-flow-tutorial#create-the-site-design

Next import the logic app. It is saved within this repository with title 'QodersSharePointBackup.json'. It is a ARM template and you can import it in the same way as importing a Power Automate template. Navigate to https://docs.microsoft.com/en-us/azure/logic-apps/export-from-microsoft-flow-logic-app-template#deploy-template-by-using-the-azure-portal and follow step 1-9. At step 4 you should upload 'QodersSharePointBackup.json'.

In step 7 use the storage account and key1 or key2 that you have created and saved previously.

<img src="https://github.com/the-qoders-community/SharePoint-to-Azure-Backup/blob/master/images/QodersSharePointBackup.png" width="600" >

Finally edit you newly created Logic App. Make sure that variables 'TenantID', 'ClientID' and 'ClientSecret' are set with the values from your AAD application. Also make sure that the 'GetRows' action references your storage table and the 'Create blob' action (rather deep in the Logic App!) Folder path is set to your blob storage.

# Configuration

To add a site to the backup process you need to add it to the Azure storage table that you created in the previous step. First you need the siteid of you SharePoint site. You can easily retrieve the siteid by navigating to your site and appending '/_api/site/id' to the url.

<img src="https://github.com/Robert1976/SharePointBackup/blob/master/images/getsiteid.PNG" width="900" >

Then you need the webid of your SharePoint site. You can retrieve the webid by navigating to your site and appending '/_api/web/id' to the url.

Now use storage explorer (you can download storage explorer from https://azure.microsoft.com/nl-nl/features/storage-explorer/) to add a new row to your storage table. This row will contain the data that the logic app will need to backup your SharePoint files.

PartitionKey: Anything you like, in the example I choose 'Sites'

RowKey: [tenantname].sharepoint.com,[siteid],[webid] (for example: qoders.sharepoint.com,2ad13d96-af88-4aa1-9fd1-c92ca098498c,a4995daa-ce03-4a20-a317-9a21525c894d)

DeltaLink: https://graph.microsoft.com/v1.0/sites/[tenantname].sharepoint.com,[siteid],[webid]/drive/root/delta (for example: https://graph.microsoft.com/v1.0/sites/qoders.sharepoint.com,2ad13d96-af88-4aa1-9fd1-c92ca098498c,a4995daa-ce03-4a20-a317-9a21525c894d/drive/root/delta)

SiteUrl: The relative url of your SharePoint site (for example /sites/Testsite)

<img src="https://github.com/Robert1976/SharePointBackup/blob/master/images/storageexplorer1.PNG" width="400" >

That's it! Your DeltaLink will be updated after each run of the logic app so that it will only pick up changes and new documents that have happened after the last run.
