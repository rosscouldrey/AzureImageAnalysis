# AzureImageAnalysis
This repo is a python script to connect to Azure Cognitive services, loop through a list of URLs and extract image data to files for further analysis.

## Setup

This script requires that you have access to

1. A Cognitive Services account. [Setup Instructions](https://learn.microsoft.com/en-us/azure/cognitive-services/cognitive-services-apis-create-account?tabs=multiservice%2Canomaly-detector%2Clanguage-service%2Ccomputer-vision%2Cwindows)
2. An Azure Storage Account.  [Setup Instructions](https://learn.microsoft.com/en-us/azure/storage/common/storage-account-create?tabs=azure-portal)

## Variables

Please obtain the following information from your Azure Subscription before proceeding as these are required to set up the variables at the start of the script.

1. Tenant Details
- TenantID

2. Storage Account Details
- Storage Account Name
- Storage account key (SAS key)
- Container Name

2. 
