# Create-a-Blob-storage-resources-with-the-.NET-client-library.
This project, we create an Azure Storage account and build a .NET console application using the Azure Storage Blob client library to create containers, upload files to blob storage, list blobs, and download files. 
Also, authenticate with Azure, perform blob storage operations programmatically, and verify content results in the Azure portal.


Tasks performed in this project:
1. Prepare the Azure resources
2. Create a console app to create and download data
3. Run the app and verify results


>>> Create an Azure Storage account
In this section of the project, you create the needed resources in Azure with the Azure CLI.

In your browser, navigate to the Azure portal https://portal.azure.com, signing in with your Azure credentials if prompted.

>>> Use the [>_] button to the right of the search bar at the top of the page to create a new Cloud Shell in the Azure portal, selecting a Bash environment. The cloud shell provides a command-line interface in a pane at the bottom of the Azure portal. If you are prompted to select a storage account to persist your files, select No storage account required, your subscription, and then select Apply

Note: If you have previously created a cloud shell that uses a PowerShell environment, switch it to Bash.

In the cloud shell toolbar, in the Settings menu, select Go to Classic version (this is required to use the code editor).

>>> Create a resource group for the resources needed for this project. 
Replace myResourceGroup with a name you want to use for the resource group. 
You can replace eastus2 with a region near you if needed. If you already have a resource group you want to use, proceed to the next step.

In this project, your Resource Group has already been created (myResourceGrouplod54781912) and you may safely skip this step if you choose to or you can go ahead and create 1.

#BASH 1
>>> az group create --location eastus2 --name myResourceGroup

Many of the commands require unique names and use the same parameters. 
Creating some variables will reduce the changes needed to the commands that create resources. Run the following commands to create the needed variables. Replace myResourceGroup with the name you're using for this exercise.

#BASH 2
>> resourceGroup=myResourceGrouplod54781912
>> location=eastus
>> accountName=storageacct$RANDOM

Run the following commands to create the Azure Storage account, each account name must be unique. The first command creates a variable with a unique name for your storage account. Record the name of your account from the output of the echo command.

#BASH 3
>> az storage account create --name $accountName \
   >> --resource-group $resourceGroup \
    >> --location $location \
    >> --sku Standard_LRS 

>> echo $accountName

Assign a role to your Microsoft Entra user name
To allow your app to create resources and items, assign your Microsoft Entra user to the Storage Blob Data Owner role.
Perform the following steps in the cloud shell.



Run the following command to retrieve the userPrincipalName from your account. This represents who the role will be assigned to.

#BASH 4
>> userPrincipal=$(az rest --method GET --url https://graph.microsoft.com/v1.0/me \
   >>  --headers 'Content-Type=application/json' \
    >> --query userPrincipalName --output tsv)

Run the following command to retrieve the resource ID of the storage account.
The resource ID sets the scope for the role assignment to a specific namespace.

#BASH 5
>> resourceID=$(az storage account show --name $accountName \
   >> --resource-group $resourceGroup \
    >> --query id --output tsv)

Run the following command to create and assign the Storage Blob Data Owner role. This role gives you the permissions to manage containers and items.

#BASH 6
>> az role assignment create --assignee $userPrincipal \
   >>  --role "Storage Blob Data Owner" \
    >> --scope $resourceID

Create a .NET console app to create containers and items
Now that the needed resources are deployed to Azure the next step is to set up the console application.


The following steps are performed in the cloud shell.

Run the following commands to create a directory to contain the project and change into the project directory.

#BASH 7
>> mkdir azstor
>> cd azstor

Create the .NET console application.

#BASH 8
>> dotnet new console

Run the following commands to add the required packages in the application.

#BASH 9
>> dotnet add package Azure.Storage.Blobs
>> dotnet add package Azure.Identity
Run the following command to create a data folder in your project.

#BASH 10
>> mkdir data

Now it's time to add the code for the project.

Add the starter code for the project
Run the following command in the cloud shell to begin editing the application.

csharp code area

code Program.cs
Replace any existing contents with the following code. Be sure to review the comments in the code.

using Azure.Storage.Blobs;
using Azure.Storage.Blobs.Models;
using Azure.Identity;

Console.WriteLine("Azure Blob Storage exercise\n");

// Create a DefaultAzureCredentialOptions object to configure the DefaultAzureCredential
DefaultAzureCredentialOptions options = new()
{
    ExcludeEnvironmentCredential = true,
    ExcludeManagedIdentityCredential = true
};

// Run the examples asynchronously, wait for the results before proceeding
await ProcessAsync();

Console.WriteLine("\nPress enter to exit the sample application.");
Console.ReadLine();

async Task ProcessAsync()
{
    // Create a credential using DefaultAzureCredential with configured options
    // ⚠️ REPLACE THE VALUE BELOW WITH YOUR ACTUAL STORAGE ACCOUNT NAME
    string accountName = "REPLACE_WITH_YOUR_STORAGE_ACCOUNT_NAME"; // e.g. "mystorage123"

    // Use the DefaultAzureCredential with the options configured at the top of the program
    DefaultAzureCredential credential = new DefaultAzureCredential(options);

    // Create the BlobServiceClient using the endpoint and DefaultAzureCredential
    string blobServiceEndpoint = $"https://{accountName}.blob.core.windows.net";
    BlobServiceClient blobServiceClient = new BlobServiceClient(new Uri(blobServiceEndpoint), credential);

    // Create a unique name for the container
    string containerName = "wtblob" + Guid.NewGuid().ToString();

    // Create the container and return a container client object
    Console.WriteLine("Creating container: " + containerName);
    BlobContainerClient containerClient = 
        await blobServiceClient.CreateBlobContainerAsync(containerName);

    // Check if the container was created successfully
    if (containerClient != null)
    {
        Console.WriteLine("Container created successfully, press 'Enter' to continue.");
        Console.ReadLine();
    }
    else
    {
        Console.WriteLine("Failed to create the container, exiting program.");
        return;
    }

    // Create a local file in the ./data/ directory for uploading and downloading
    Console.WriteLine("Creating a local file for upload to Blob storage...");
    string localPath = "./data/";
    string fileName = "wtfile" + Guid.NewGuid().ToString() + ".txt";
    string localFilePath = Path.Combine(localPath, fileName);

    // Write text to the file
    await File.WriteAllTextAsync(localFilePath, "Hello, World!");
    Console.WriteLine("Local file created, press 'Enter' to continue.");
    Console.ReadLine();

    // Get a reference to the blob and upload the file
    BlobClient blobClient = containerClient.GetBlobClient(fileName);

    Console.WriteLine("Uploading to Blob storage as blob:\n\t {0}", blobClient.Uri);

    // Open the file and upload its data
    using (FileStream uploadFileStream = File.OpenRead(localFilePath))
    {
        await blobClient.UploadAsync(uploadFileStream);
        uploadFileStream.Close();
    }

    // Verify if the file was uploaded successfully
    bool blobExists = await blobClient.ExistsAsync();
    if (blobExists)
    {
        Console.WriteLine("File uploaded successfully, press 'Enter' to continue.");
        Console.ReadLine();
    }
    else
    {
        Console.WriteLine("File upload failed, exiting program..");
        return;
    }

    Console.WriteLine("Listing blobs in container...");
    await foreach (BlobItem blobItem in containerClient.GetBlobsAsync())
    {
        Console.WriteLine("\t" + blobItem.Name);
    }

    Console.WriteLine("Press 'Enter' to continue.");
    Console.ReadLine();

    // Adds the string "DOWNLOADED" before the .txt extension, so it doesn't 
    // overwrite the original file
    string downloadFilePath = localFilePath.Replace(".txt", "DOWNLOADED.txt");

    Console.WriteLine("Downloading blob to: {0}", downloadFilePath);

    // Download the blob's contents and save it to a file
    BlobDownloadInfo download = await blobClient.DownloadAsync();

    using (FileStream downloadFileStream = File.OpenWrite(downloadFilePath))
    {
        await download.Content.CopyToAsync(downloadFileStream);
    }

    Console.WriteLine("Blob downloaded successfully to: {0}", downloadFilePath);
}

```
>>> az login

You must sign in to Azure, even though the cloud shell session is already authenticated.

Note: In most scenarios, just using az login will be sufficient. However, if you have subscriptions in multiple tenants, you may need to specify the tenant by using the --tenant parameter. See Sign into Azure interactively using Azure CLI for details.

Run the following command to start the console app. The app will pause many times during execution, waiting for you to press any key to continue. This allows you to view the messages in the Azure portal.

#BASH
>> dotnet run

In the Azure portal, navigate to the Azure Storage account you created.

Expand > Data storage in the left navigation and select Containers.

Select the container the application created, and you can view the blob that was uploaded.

Run the two commands below to change into the data directory and list the files that were uploaded and downloaded.

#BASH 
>> cd data

 >> ls

Clean up resources
At the end of this project, you should delete the cloud resources you created to avoid unnecessary resource usage.

In your browser, navigate to the Azure portal https://portal.azure.com, signing in with your Azure credentials if prompted.
Navigate to the resource group you created and view the contents of the resources used in this exercise.
On the toolbar, select Delete resource group.
Enter the resource group name and confirm that you want to delete it.

NOTE: Deleting a resource group deletes all resources contained within it. If you chose an existing resource group for this exercise, any existing resources outside the scope of this exercise will also be deleted.

Congratulations, this is the end of our mini-project.
