# Azure Batch AI Workshop

This workshop lab is based on <a href="https://docs.microsoft.com/en-us/azure/batch-ai/">this example</a>. We will be working through a simple test to deploy an Azure Batch AI cluster and run a Microsoft CNTK training job on an MNIST handwriting dataset. We will alter the example to run on a CPU container rather than a GPU container. 

Before we get there we need to do some basic setup: 

## 1) Enable Your Subscription

Your instructor will provide you with a code and guidance here. You will redeem your Azure Pass code via <a href="https://www.microsoftazurepass.com/">the AzurePass website</a>, which will enable your subscription and credit. You will need to provide an e-mail address, or can simply create a new outlook.com address. Detailed instructions are available in <a href=ActivateAzurePass.pdf>this document</a>.

## 2) Deploy an Ubuntu Data Science VM

Follow the instructions under <a href="https://docs.microsoft.com/en-us/azure/machine-learning/data-science-virtual-machine/dsvm-ubuntu-intro">"Create your Data Science Virtual Machine for Linux"</a>

Be sure not to select the Deep Learning VM, but select the Data Science VM, as your Azure Pass may not yet have access to GPU nodes, it will not allow you to provision the Data Science VM. 

## 3) Login to Your Subscription via CLI + Web Browser

SSH to your Ubuntu Data Science VM, and login via the Azure CLI: 

```
azureuser@dsvm:~$ az login
To sign in, use a web browser to open the page https://aka.ms/devicelogin and enter the code AYQ64RG35 to authenticate.
```

The CLI waits at this point. Follow the instruction and login via a browser:

:![devcelogin](images/devicelogin.PNG)

At this point the prompt returns in your linux window and you can continue again there: 
```
azureuser@dsvm:~$ az login
To sign in, use a web browser to open the page https://aka.ms/devicelogin and enter the code AYQ64RG35 to auth
enticate.
[
  {
    "cloudName": "AzureCloud",
    "id": "a0f92f07-9b51-4686-9233-c1ef4d19a8bf",
    "isDefault": true,
    "name": "Azure Pass",
    "state": "Enabled",
    "tenantId": "e0d1a93e-e295-42fc-954b-05b11079f167",
    "user": {
      "name": "kiernanek@outlook.com",
      "type": "user"
    }
  }
]
```
You can also check the subscription you are logged into is the correct one with:
```
azureuser@dsvm:~$ az account list -o table
Name        CloudName    SubscriptionId                        State    IsDefault
----------  -----------  ------------------------------------  -------  -----------
Azure Pass  AzureCloud   a0f92f07-9b51-4686-9233-c1ef4d19a8bf  Enabled  True
```

# Now we are logged in

## 1) Download the BatchAIQuickStart.zip

From your Ubuntu Data Science VM, grab the file with wget and unzip it: 

```
azureuser@dsvm:~$ wget https://github.com/azurebigcompute/CloudWorkshops/raw/master/BatchAIQuickStart.zip
--2017-12-06 22:17:37--  https://github.com/azurebigcompute/CloudWorkshops/raw/master/BatchAIQuickStart.zip
Resolving github.com... 192.30.253.113, 192.30.253.112
Connecting to github.com|192.30.253.113|:443... connected.
HTTP request sent, awaiting response... 302 Found
Location: https://raw.githubusercontent.com/azurebigcompute/CloudWorkshops/master/BatchAIQuickStart.zip [following]
--2017-12-06 22:17:37--  https://raw.githubusercontent.com/azurebigcompute/CloudWorkshops/master/BatchAIQuickStart.zip
Resolving raw.githubusercontent.com... 151.101.32.133
Connecting to raw.githubusercontent.com|151.101.32.133|:443... connected.
HTTP request sent, awaiting response... 200 OK
Length: 16139626 (15M) [application/zip]
Saving to: ‘BatchAIQuickStart.zip’

BatchAIQuickStart.zip       100%[=========================================>]  15.39M  46.0MB/s    in 0.3s

2017-12-06 22:17:38 (46.0 MB/s) - ‘BatchAIQuickStart.zip’ saved [16139626/16139626]

azureuser@dsvm:~$ unzip BatchAIQuickStart.zip
Archive:  BatchAIQuickStart.zip
  inflating: ConvNet_MNIST.py
  inflating: Test-28x28_cntk_text.txt
  inflating: Train-28x28_cntk_text.txt
azureuser@dsvm:~$ ls
BatchAIQuickStart.zip  Desktop    Test-28x28_cntk_text.txt
ConvNet_MNIST.py       notebooks  Train-28x28_cntk_text.txt
azureuser@dsvm:~$
```

## 2) Customize the job.json File

Unfortunately the default Azure Pass does not have GPU (N-Series nodes) enabled by default. You can fix this by opening a helpdesk ticket via the '?' icon in portal.azure.com, and request that N-Series is enabled for your subscription. 

However in the meantime for this workshop, we will simply replace the GPU container with a CPU container. Hence, please download <a href="job.json">This job.json</a>file. 

Note the key difference from the example is that we replace the GPU container:
```
 "containerSettings": {
         "imageSourceRegistry": {
             "image": "microsoft/cntk:2.1-gpu-python3.5-cuda8.0-cudnn6.0"
         }
     }
```
With an equivalent CPU container: 
```
     "containerSettings": {
         "imageSourceRegistry": {
             "image": "microsoft/cntk:2.1-cpu-python3.5"
         }
     }
```

## 3) Setup Azure Batch AI

At this point you can <a href="https://docs.microsoft.com/en-us/azure/batch-ai/quickstart-cli">follow the instructions in the example</a>. Alternatively, the commands are summarized here for your convenience:

```
#-- Enable Resource Providers
az provider register -n Microsoft.BatchAI
az provider register -n Microsoft.Batch

#-- Setup Resource Group & Storage Account
az group create --name mikebatchaigrp --location eastus
az configure --defaults group=mikebatchaigrp
az configure --defaults location=eastus
az storage account create --name mkbatchaistore --sku Standard_LRS -g mikebatchaigrp

#-- Set Default Accounts
export AZURE_STORAGE_ACCOUNT=mkbatchaistore
export AZURE_STORAGE_KEY=$(az storage account keys list --account-name mkbatchaistore -o tsv --query [0].value
)
export AZURE_BATCHAI_STORAGE_ACCOUNT=mkbatchaistore
export AZURE_BATCHAI_STORAGE_KEY=$(az storage account keys list --account-name mkbatchaistore -o tsv --query [
0].value)

#-- Create Shares & Upload the BatchAIQuickStart files
az storage share create --name batchaiquickstart
az storage directory create --share-name batchaiquickstart  --name mnistcntksample
az storage file upload --share-name batchaiquickstart --source Train-28x28_cntk_text.txt --path mnistcntksampl
e
az storage file upload --share-name batchaiquickstart --source Test-28x28_cntk_text.txt --path mnistcntksample
az storage file upload --share-name batchaiquickstart --source ConvNet_MNIST.py --path mnistcntksample

#-- Create Your Cluster
az batchai cluster create --name mycluster --vm-size STANDARD_NC6 --image UbuntuLTS --min 1 --max 1 --afs-name
 batchaiquickstart --afs-mount-path azurefileshare --user-name azureuser --password AL0ng0bscurePassw0rd
az batchai cluster list -o table

#-- Create your job.json

#-- Start Your Machine Learning Job
az batchai job create --name myjob --cluster-name mycluster --config job.json
az batchai job list -o table

#-- Check the Outputs
az batchai job list-files --name myjob --output-directory-id stdouterr
az batchai job stream-file --job-name myjob --output-directory-id stdouterr --name stderr.txt

#-- Delete Your Cluster
az batchai job delete --name myjob
az batchai cluster delete --name mycluster
```

# Next Steps

Congratulations! You have completed the workshop. 

1) Open a helpdesk ticket to enable GPU's in your subscription. Once the ticket has been fulfilled, re-run the test above and GPU, and compare the speed of the Training job. 

2) Access the <a href="https://github.com/Azure/BatchAI">Azure Batch AI Github</a>, and execute some more of the recipes for the other toolkits. 
