## Setup

### Libraries

```python
# Import required libraries

from unittest import result
from azure.cognitiveservices.vision.computervision import ComputerVisionClient
from azure.cognitiveservices.vision.computervision.models import OperationStatusCodes
from azure.cognitiveservices.vision.computervision.models import VisualFeatureTypes
from msrest.authentication import CognitiveServicesCredentials
from datetime import datetime
from pathlib import Path

from array import array
import os
from PIL import Image
import sys
import time
import pandas as pd
```

### Variables Set up

```python
tenant = <yourTenantID>

#storage account variables
storage_account_name = <yourStorageAcctName>
container = <yourContainer>

#application identity variables
appID = <yourAppID>
secret = <yourAppSecret>

#Cognitive Services Authentication variables
subscription_key = <yourCognitiveServicesSubscriptionKey>
endpoint = <yourCognitiveServicesEndpoint>

#variable to control dev/test workflow - when TRUE we only use subsets of datafiles to keep debugging light
dbutils.widgets.dropdown("Development Flag", 'True',['True','False'])
dbutils.widgets.text("Dev Sample Size","10")

#directory name for storing output files:
dt = datetime.now().strftime('%Y%m%d')
dir = 'mnt/data/output/'+dt+'/'

#file that stores the ImageURLs - do not include .csv
imagescsv = "yourImageFile"

```
### Storage Account Connection

Setup our storage account for access.  Here we create the connection and mount it to our filesystem for access.

```python
configs = {"fs.azure.account.auth.type": "OAuth",
          "fs.azure.account.oauth.provider.type": "org.apache.hadoop.fs.azurebfs.oauth2.ClientCredsTokenProvider",
          "fs.azure.account.oauth2.client.id": appID,
          "fs.azure.account.oauth2.client.secret": secret,
          "fs.azure.account.oauth2.client.endpoint": "https://login.microsoftonline.com/"+tenant+"/oauth2/token"}

mp = "/mnt/data"

#mount if not mounted already
if not any(mount.mountPoint == mp for mount in dbutils.fs.mounts()):
  dbutils.fs.mount(
      source = "abfss://"+container+"@"+storage_account_name+".dfs.core.windows.net/",
      mount_point = mp,
      extra_configs = configs)

#confirm mount
#dbutils.fs.mounts()
```
### Create ComputerVision Object

We'll use the computer vision object to call the Cognitive service later in the script

```python
#create a computervision client
computervision_client = ComputerVisionClient(endpoint, CognitiveServicesCredentials(subscription_key))

```

## Read Data

Use pandas to read the URL list from the CSV file into a dataframe (called imagesDF)

```python

#read data
imagesDF = pd.read_csv("/dbfs/mnt/data/"+imagescsv+".csv")

```

## Functions

We define a number of functions to deal with our return data from the cognitive service.  Depending on the visual_features parameter we send to the service, we will recieve various responses, each of which can be parsed seperately using these functions.

At time of writing (October 17, 2022) the following visual_features are supported:

| Viual Feature| Definition|link|
|--------------|------------|---|
|Categories| categorizes image content according to a taxonomy defined in documentation | [link](https://docs.microsoft.com/en-us/azure/cognitive-services/computer-vision/concept-categorizing-images) 
