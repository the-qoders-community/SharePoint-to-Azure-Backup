# Simple SharePoint to Azure Backup
A low-code solution to let end-users backup SharePoint content to Azure storage

Simple Azure serverless SharePoint backup uses a logic app and storage account to backup SharePoint files. The solution only backups files that are stored in the Shared Documents library of your site (which is ideal if you use Teams). The solution uses the 'Delta' method of the Graph files api. Only newly added or edited files are updated (for more information see: https://docs.microsoft.com/en-us/graph/api/driveitem-delta?view=graph-rest-1.0&tabs=http).

# Installation

Start by creating an Azure Active Directory application. Follow the steps in this blog  https://www.lee-ford.co.uk/using-flow-with-graph-api/ and make sure your application (so not delegated!) has 'Sites.ReadWrite.All' Graph Permissions.

<img src="https://github.com/Robert1976/SharePointBackup/blob/master/images/graph.PNG" width="600" >

Make sure you save your TenantID, your applications ClientID and your applications ClientSecret to a temporary notepad. You need them further on.

Now create a new storage account in Azure. Create a new blob storage container (this container will hold the SharePoint files) and a new table (this will hold references to the sites that will be backed up) and a new queue. I named my blob 'sharepointfiles', my table 'sharepointsites' and my queue 'sharepointazurebackup'. Make sure you remember the name of your storage account and copy one of your access keys (key1 or key2) to a temporary note. Your blob should be a private blob!

The next step is to import the first logic app: 'QodersAddSiteToBackUp'. This is a logic app with an HTTP endpoint. The endpoint is called by the SharePoint site design/site script. The template is saved within this repository with title 'QodersAddSiteToBackUp.json' in folder 'Resources'. It is a ARM template and you can import it in the same way as importing a Power Automate template. Navigate to https://docs.microsoft.com/en-us/azure/logic-apps/export-from-microsoft-flow-logic-app-template#deploy-template-by-using-the-azure-portal and follow step 1-9. At step 4 you should upload 'QodersAddSiteToBackUp.json'.

In step 7 use the storage account name, and key1 or key2 that you have created and saved previously in the yellow marked input fields.

<img src="https://github.com/the-qoders-community/SharePoint-to-Azure-Backup/blob/master/Images/QodersSharePointBackup.PNG" width="600" >

Click 'Purchase' and after import open the logic app in edit mode. First make sure you copy the HTTP Post endpoint to a notepad.

<img src="https://github.com/the-qoders-community/SharePoint-to-Azure-Backup/blob/master/Images/QodersSharePointBackup_http.PNG" width="400" >

Followed by changing the placeholder TENANTNAME in action "Tenantname" by your own O365 tenantname. In the 'HTTP' action replace the values TENANT, CLIENT and SECRET by the TenantID, the ClientID and the ClientSecret that you obtained in previous steps. Save your logic app.

You can now configure the Sitescript and Sitedesign. This is very well descibed here: https://docs.microsoft.com/en-us/sharepoint/dev/declarative-customization/site-design-trigger-flow-tutorial#create-the-site-design.

You can use the example JSON. Just make sure you replace the [paste the workflow trigger URL here]-part with your own HTTP-endpoint. Also, replace the 'Add-SPOSiteDesign' command with: 

Add-SPOSiteDesign -Title "Azure Backup" -Description "Add site to daily Azure Backup" -SiteScripts -script id- -WebTemplate 0

The '-WebTemplate 0' part makes sure that your Site Design can only be applied to existing sites.

Now import the second logic app. It is named 'QodersScheduledBus.json'. Just follow the same steps as mentioned above and again, in step 7, use the storage account name, and key1 or key2 that you have created and saved previously in the yellow marked input fields. This logic app uses the Azure storage queue.

<img src="https://github.com/the-qoders-community/SharePoint-to-Azure-Backup/blob/master/Images/QodersScheduledBus.PNG" width="600" >

Finnaly import the third logic app. This logic app will pick up messages from the queue and backup the files from SharePoint to Azure. Again follow step 1-9. At step 4 you should upload 'QodersSharePointBackup.json'.

In step 7 use the storage account and key1 or key2 that you have created and saved previously.

<img src="https://github.com/the-qoders-community/SharePoint-to-Azure-Backup/blob/master/Images/QodersSharePointBackup_settings.PNG" width="600" >

Edit your newly created Logic App. Make sure that variables 'TenantID', 'ClientID' and 'ClientSecret' are set with the values from your AAD application. Also make sure that the 'GetRows' action references your storage table and the 'Create blob' action (rather deep in the Logic App!) Folder path is set to your blob storage.

# Add versioning

Adding versioning to the solution is as simple as adding a extra 'date' folder to the path in the blob storage (which makes sure that the blobs that were stored the previous day are not overwritten)

<img src="https://github.com/the-qoders-community/SharePoint-to-Azure-Backup/blob/master/Images/versioning.png" width="600" >

# Performance

We tested the performance of the solution with a back-up of a SharePoint site with 2 GB of data spread over 4500 Office documents (each about 500 kb in size). It takes about 17 minutes to back-up the site. Extrapolated this would give you a performance of about 6.5 GB per hour, per workflow instance (or site collection if you like).

By default the number of concurrent Logic App workflow instances is unlimited. In other words: all your sites will be backed up at the same time. For more information see: https://docs.microsoft.com/en-us/azure/logic-apps/logic-apps-workflow-actions-triggers#change-trigger-concurrency

# Some thoughts on doing a 'Restore'

At the end of 2019 Microsoft has purchased the migration tool 'mover.io'. Mover enables you to backup your sites to Azure Blob storage. But at moment of writing it does not support using the API (while the tool does have an API) and it does not support metadata (who created te file, when and who changed the file, when).

This solution saves basic metadata of all files to a json file at the root of the site collection related folder in the blob. The filename of the metadata file is 'metadata.json'. This file also contains the filepath.

A way of restoring a back-up as an administrator is for example to use 'mover.io' to restore the files from Azure storage back to SharePoint online, followed by some post-processing that adds the metadata to the files using the 'metadata.json' file. You could temporarely disable versioning of the document library to prevent that an new version is created while setting the metadata. A future post will dive a little deeper into this.  






