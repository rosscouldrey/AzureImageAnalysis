# AzureImageAnalysis
This repo is a Databricks based python script to connect to Azure Cognitive services, loop through a list of Image URLs, and extract image data.  The image data is saved to files for further analysis.

## Setup

This script requires that you have access to

1. A Cognitive Services account [Setup Instructions](https://learn.microsoft.com/en-us/azure/cognitive-services/cognitive-services-apis-create-account?tabs=multiservice%2Canomaly-detector%2Clanguage-service%2Ccomputer-vision%2Cwindows)
2. An Azure Storage Account  [Setup Instructions](https://learn.microsoft.com/en-us/azure/storage/common/storage-account-create?tabs=azure-portal)
3. An Azure Databricks environment [Setup Instructions](https://learn.microsoft.com/en-us/azure/databricks/scenarios/quickstart-create-databricks-workspace-portal?tabs=azure-portal)

Additionally, you will need to create a Service Principal for access to your storage account.

## Variables

Please obtain the following information from your Azure Subscription before proceeding as these are required to set up the variables at the start of the script.

### Tenant Details
- TenantID

### Storage Account Details
- Storage Account Name
- Storage account key (SAS key)
- Container Name

### Service Principal
- Application ID
- Secret

## Code
All code for this exercise is contained in [ImageAnalysisPoC.md](https://github.com/rosscouldrey/AzureImageAnalysis/blob/main/ImageAnalysisPoC.md)
