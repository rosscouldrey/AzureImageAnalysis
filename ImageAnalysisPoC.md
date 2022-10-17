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

| Visual Feature| Definition|
|--------------|------------|
|[Categories](https://docs.microsoft.com/en-us/azure/cognitive-services/computer-vision/concept-categorizing-images)| Categorizes image content according to a taxonomy defined in documentation |
|[Tags](https://docs.microsoft.com/en-us/azure/cognitive-services/computer-vision/concept-tagging-images)| Tags the image with a detailed list of words related to the image content|
|[Description](https://docs.microsoft.com/en-us/azure/cognitive-services/computer-vision/concept-describing-images)| Describes the image content with a complete English sentence|
|[Faces](https://docs.microsoft.com/en-us/azure/cognitive-services/computer-vision/concept-detecting-faces)| Detects if faces are present. If present, generate coordinates, gender and age|
|[ImageType](https://docs.microsoft.com/en-us/azure/cognitive-services/computer-vision/concept-detecting-image-types)| Detects if image is clipart or a line drawing|
|[Color](https://docs.microsoft.com/en-us/azure/cognitive-services/computer-vision/concept-detecting-color-schemes)| Determines the accent color, dominant color, and whether an image is black&white|
|[Adult](https://docs.microsoft.com/en-us/azure/cognitive-services/computer-vision/concept-detecting-adult-content)| Detects if the image is pornographic in nature (depicts nudity or a sex act), or is gory (depicts extreme violence or blood). Sexually suggestive content (aka racy content) is also detected|
|[Objects](https://docs.microsoft.com/en-us/azure/cognitive-services/computer-vision/concept-object-detection)| Detects various objects within an image, including the approximate location. The Objects argument is only available in English|
|[Brands](https://docs.microsoft.com/en-us/azure/cognitive-services/computer-vision/concept-brand-detection)| Detects various brands within an image, including the approximate location. The Brands argument is only available in English|

Each function takes an ID and an imageanalysis object as parameters and returns a dataframe containing the parsed results.

### Get Tags
```python
def get_tags(id, imganalysis):
    
    imgtagList = []
    
    #if no tags found create a 'None' entry
    if(len(imganalysis.tags)==0):
            return pd.DataFrame()
            #imgtagList.append([img_url,'None',1])
            #tempdf = pd.DataFrame({'imageurl':[img_url],'Tag':['None'],'Confidence':[1]}) 
    else:
        #when tags are found - loop through tags and parse each
            for t in imganalysis.tags:
                imgtagList.append([id,t.name,t.confidence])
                #tempdf = pd.DataFrame({'imageurl':[img_url],'Tag':[t.name],'Confidence':[t.confidence]})
                      
                      
    #return final list as dataframe
    return pd.DataFrame(imgtagList, columns = ['imageID','Tag','Confidence'])  
```

### Get Brands
```python

#define function to parse brands from CognitiveServices Object

def get_brands(id, imganalysis):
    
    brandlist = []
    
    if (len(imganalysis.brands)==0):
        return pd.DataFrame()
    else:
        for brand in imganalysis.brands:
            brandlist.append([id,brand.name,brand.confidence])
    
    return pd.DataFrame(brandlist, columns = ['imageID','Brand','Confidence'])

```

### Get Description

```python

#define a function to 
def get_AIdescription (id, imganalysis):
    
    descriptionlist = []
    
    if(len(imganalysis.description.captions) == 0):
           return pd.DataFrame()
    else:
       for desc in imganalysis.description.captions:
           descriptionlist.append([id,desc.text,desc.confidence])
       
    return pd.DataFrame(descriptionlist, columns = ['imageID','Description','Confidence'])

```

### Get Colour Scheme

``` python

def get_ColourScheme (id, imganalysis):
    
    colourProfile = []
    
    colourProfile.append([id,"Dominant FG colour",imganalysis.color.dominant_color_foreground])
    colourProfile.append([id,"Dominant BG colour",imganalysis.color.dominant_color_background])
    colourProfile.append([id,"isBlackWhite",imganalysis.color.is_bw_img])
    colourProfile.append([id,"AccentColour",imganalysis.color.accent_color])
    if len(imganalysis.color.dominant_colors) == 0:
      pass
    else:
        for c in imganalysis.color.dominant_colors:
            colourProfile.append([id,"dominantcolour",c])
    
    return pd.DataFrame(colourProfile, columns = ['imageID','ColourAttribute','Value'])

```

### Get Objects

```python

def get_objects(id,imganalysis):
    
    objectlist = []
    
    if(len(imganalysis.objects)==0):
        return pd.DataFrame()
    else:
        for obj in imganalysis.objects:
            objectlist.append([id,obj.object_property,obj.confidence])
    
    return pd.DataFrame(objectlist, columns = ['imageID','Object','Confidence'])

```

### Get Categories

```python

def get_categories(id,imganalysis):
    
    catList = []
    
    if (len(imganalysis.categories)==0):
        return pd.DataFrame()
    else:
        for cat in imganalysis.categories:
            catList.append([id,cat.name,cat.score])
    
    return pd.DataFrame(catList, columns = ['imageID','Category','Score'])

```

### Get Adult Content

```python

def get_adultContent(id,imganalysis):
    
    adultcontentList = []
    
    adultcontentList.append([id,"is_adult_content",imganalysis.adult.is_adult_content,imganalysis.adult.adult_score])
    adultcontentList.append([id,"is_racy_content",imganalysis.adult.is_racy_content,imganalysis.adult.racy_score])
    adultcontentList.append([id,"is_gory_content",imganalysis.adult.is_gory_content,imganalysis.adult.gore_score])
    
    return pd.DataFrame(adultcontentList, columns = ['imageID','Attribute','Value','Score'])

```

### Get Text

```python

def get_textfromimage(id,url):
    textinimage = []
    numberOfCharsInOperationId = 36

    # SDK call
    try:
        rawHttpResponse = computervision_client.read(url, language="en", raw=True)

        # Get ID from returned headers
        operationLocation = rawHttpResponse.headers["Operation-Location"]
        idLocation = len(operationLocation) - numberOfCharsInOperationId
        operationId = operationLocation[idLocation:]

        # SDK call
        OCRresult = computervision_client.get_read_result(operationId)

        #loop until status code is no longer "running"
        while(OCRresult.status == OperationStatusCodes.running):
            OCRresult = computervision_client.get_read_result(operationId)

        # Get data
        if OCRresult.status == OperationStatusCodes.succeeded:
            for line in OCRresult.analyze_result.read_results[0].lines:
                textinimage.append([id,line.text])

        return pd.DataFrame(textinimage, columns = ['imageID','Text'])
    
    except:
        print("OCR error: " + url)
        return pd.DataFrame()

```
