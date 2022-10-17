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
