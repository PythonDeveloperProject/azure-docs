---
title: Quickstart for managing Azure file shares with Azure PowerShell
description: Use this quickstart to learn to manage Azure file shares using Azure PowerShell.
services: storage
author: wmgries
ms.service: storage
ms.topic: quickstart
ms.date: 10/18/2018
ms.author: wgries
ms.component: files
#Customer intent: As a < type of user >, I want < what? > so that < why? >.
---

# Quickstart: Create and manage an Azure file share with Azure PowerShell 
This guide walks you through the basics of working with [Azure file shares](storage-files-introduction.md) with PowerShell. Azure file shares are just like other file shares, but stored in the cloud and backed by the Azure platform. Azure File shares support the industry standard SMB protocol and enable file sharing across multiple machines, applications, and instances. 

If you don't have an Azure subscription, create a [free account](https://azure.microsoft.com/free/?WT.mc_id=A261C142F) before you begin.

[!INCLUDE [cloud-shell-powershell.md](../../../includes/cloud-shell-powershell.md)]

If you would like to install and use the PowerShell locally, this guide requires the Azure PowerShell module version 5.1.1 or later. To find out which version of the Azure PowerShell module you are running, execute `Get-Module -ListAvailable AzureRM`. If you need to upgrade, see [Install Azure PowerShell module](/powershell/azure/install-azurerm-ps). If you are running PowerShell locally, you also need to run `Login-AzureRmAccount` to login to your Azure account.

## Create a resource group
A resource group is a logical container into which Azure resources are deployed and managed. If you don't already have an Azure resource group, you can create a new one with the [New-AzureRmResourceGroup](/powershell/module/azurerm.resources/new-azurermresourcegroup) cmdlet. 

The following example creates a resource group named *myResourceGroup* in the East US region:

```azurepowershell-interactive
New-AzureRmResourceGroup `
    -Name myResourceGroup `
    -Location EastUS
```

## Create a storage account
A storage account is a shared pool of storage you can use to deploy Azure file shares, or other storage resources such as blobs or queues. A storage account can contain an unlimited number of shares, and a share can store an unlimited number of files, up to the capacity limits of the storage account.

This example creates a storage account using the [New-AzureRmStorageAccount](/powershell/module/azurerm.storage/new-azurermstorageaccount) cmdlet. The storage account is named *mystorageaccount<random number>* and a reference to that storage account is stored in the variable **$storageAcct**. Storage account names must be unique, so use `Get-Random` to append a number to the name to make it unique. 

```azurepowershell-interactive 
$storageAcct = New-AzureRmStorageAccount `
                  -ResourceGroupName "myResourceGroup" `
                  -Name "mystorageacct$(Get-Random)" `
                  -Location eastus `
                  -SkuName Standard_LRS 
```

## Create an Azure file share
Now you can create your first Azure file share. You can create a file share using the [New-AzureStorageShare](/powershell/module/azurerm.storage/new-azurestorageshare) cmdlet. This example creates a share named `myshare`.

```azurepowershell-interactive
New-AzureStorageShare `
   -Name myshare `
   -Context $storageAcct.Context
```

Share names need to be all lower-case letters, numbers, and single hyphens but cannot start with a hyphen. For complete details about naming file shares and files, see [Naming and Referencing Shares, Directories, Files, and Metadata](https://docs.microsoft.com/rest/api/storageservices/Naming-and-Referencing-Shares--Directories--Files--and-Metadata).

## Use your Azure file share
Azure Files provides two methods of working with files and folders within your Azure file share: the industry standard [Server Message Block (SMB) protocol](https://msdn.microsoft.com/library/windows/desktop/aa365233.aspx) and the [File REST protocol](https://docs.microsoft.com/rest/api/storageservices/file-service-rest-api). 

To mount a file share with SMB, see the following document based on your OS:
- [Windows](storage-how-to-use-files-windows.md)
- [Linux](storage-how-to-use-files-linux.md)
- [macOS](storage-how-to-use-files-mac.md)

### Using an Azure file share with the File REST protocol 
It is possible work directly with the File REST protocol directly (i.e. handcrafting REST HTTP calls yourself), but the most common way to use the File REST protocol is to use the AzureRM PowerShell module, the [Azure CLI](storage-how-to-use-files-cli.md), or an Azure Storage SDK, all of which provide a nice wrapper around the File REST protocol in the scripting/programming language of your choice.  

In most cases, you will use your Azure file share over the SMB protocol, as this allows you to use the existing applications and tools you expect to be able to use, but there are several reasons why it is advantageous to use the File REST API rather than SMB, such as:

- You are browsing your file share from the PowerShell Cloud Shell (which cannot mount file shares over SMB).
- You need to execute a script or application from a client which cannot mount an SMB shares, such as on-premises clients which do not have port 445 unblocked.
- You are taking advantage of serverless resources, such as [Azure Functions](../../azure-functions/functions-overview.md). 

The following examples show how to use the AzureRM PowerShell module to manipulate your Azure file share with the File REST protocol. 

#### Create directory
To create a new directory named *myDirectory* at the root of your Azure file share, use the [New-AzureStorageDirectory](/powershell/module/azurerm.storage/new-azurestoragedirectory) cmdlet.

```azurepowershell-interactive
New-AzureStorageDirectory `
   -Context $storageAcct.Context `
   -ShareName "myshare" `
   -Path "myDirectory"
```

#### Upload a file
To demonstrate how to upload a file using the [Set-AzureStorageFileContent](/powershell/module/azure.storage/set-azurestoragefilecontent) cmdlet, we first need to create a file inside your PowerShell Cloud Shell's scratch drive to upload. 

This example puts the current date and time into a new file on your scratch drive, then uploads the file to the file share.

```azurepowershell-interactive
# this expression will put the current date and time into a new file on your scratch drive
Get-Date | Out-File -FilePath "C:\Users\ContainerAdministrator\CloudDrive\SampleUpload.txt" -Force

# this expression will upload that newly created file to your Azure file share
Set-AzureStorageFileContent `
   -Context $storageAcct.Context `
   -ShareName "myshare" `
   -Source "C:\Users\ContainerAdministrator\CloudDrive\SampleUpload.txt" `
   -Path "myDirectory\SampleUpload.txt"
```   

If you're running PowerShell locally, you should substitute `C:\Users\ContainerAdministrator\CloudDrive\` with a path that exists on your machine.

After uploading the file, you can use [Get-AzureStorageFile](/powershell/module/Azure.Storage/Get-AzureStorageFile) cmdlet to check to make sure that the file was uploaded to your Azure file share. 

```azurepowershell-interactive
Get-AzureStorageFile -Context $storageAcct.Context -ShareName "myshare" -Path "myDirectory" 
```

#### Download a file
You can use the [Get-AzureStorageFileContent](/powershell/module/azure.storage/get-azurestoragefilecontent) cmdlet to download a copy of the file you just uploaded to the scratch drive of your Cloud Shell.

```azurepowershell-interactive
# Delete an existing file by the same name as SampleDownload.txt, if it exists because you've run this example before.
Remove-Item 
    `-Path "C:\Users\ContainerAdministrator\CloudDrive\SampleDownload.txt" `
     -Force `
     -ErrorAction SilentlyContinue

Get-AzureStorageFileContent `
    -Context $storageAcct.Context `
    -ShareName "myshare" `
    -Path "myDirectory\SampleUpload.txt" ` 
    -Destination "C:\Users\ContainerAdministrator\CloudDrive\SampleDownload.txt"
```

After downloading the file, you can use the `Get-ChildItem` to see that the file has been downloaded to your PowerShell Cloud Shell's scratch drive.

```azurepowershell-interactive
Get-ChildItem -Path "C:\Users\ContainerAdministrator\CloudDrive"
``` 

#### Copy files
One common task is to copy files from one file share to another file share, or to/from an Azure Blob storage container. To demonstrate this functionality, you can create a new share and copy the file you just uploaded over to this new share using the [Start-AzureStorageFileCopy](/powershell/module/azure.storage/start-azurestoragefilecopy) cmdlet. 

```azurepowershell-interactive
New-AzureStorageShare `
    -Name "myshare2" `
    -Context $storageAcct.Context
  
New-AzureStorageDirectory `
   -Context $storageAcct.Context `
   -ShareName "myshare2" `
   -Path "myDirectory2"

Start-AzureStorageFileCopy `
    -Context $storageAcct.Context `
    -SrcShareName "myshare" `
    -SrcFilePath "myDirectory\SampleUpload.txt" `
    -DestShareName "myshare2" `
    -DestFilePath "myDirectory2\SampleCopy.txt" `
    -DestContext $storageAcct.Context
```

Now, if you list the files in the new share, you should see your copied file.

```azurepowershell-interactive
Get-AzureStorageFile -Context $storageAcct.Context -ShareName "myshare2" -Path "myDirectory2" 
```

While the `Start-AzureStorageFileCopy` cmdlet is convenient for ad-hoc file moves between Azure file shares and Azure Blob storage containers, we recommend AzCopy for larger moves (in terms of number or size of files being moved). Learn more about [AzCopy for Windows](../common/storage-use-azcopy.md) and [AzCopy for Linux](../common/storage-use-azcopy-linux.md). AzCopy must be installed locally - it is not available in Cloud Shell. 

## Clean up resources
When you are done, you can use the [Remove-AzureRmResourceGroup](/powershell/module/azurerm.resources/remove-azurermresourcegroup) cmdlet to remove the resource group and all related resources. 

```azurepowershell-interactive
Remove-AzureRmResourceGroup -Name myResourceGroup
```

You can alternatively remove resources one by one:

- To remove the Azure file shares we created for this quickstart.
```azurepowershell-interactive
Get-AzureStorageShare -Context $storageAcct.Context | Where-Object { $_.IsSnapshot -eq $false } | ForEach-Object { 
    Remove-AzureStorageShare -Context $storageAcct.Context -Name $_.Name
}
```

- To remove the storage account itself (this will implicitly remove the Azure file shares we created as well as any other storage resources you may have created such as an Azure Blob storage container).
```azurepowershell-interactive
Remove-AzureRmStorageAccount -ResourceGroupName $storageAcct.ResourceGroupName -Name $storageAcct.StorageAccountName
```

## Next steps

> [!div class="nextstepaction"]
> [What is Azure Files?](storage-files-introduction.md)