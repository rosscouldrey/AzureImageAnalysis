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

### Cognitive Services Parsing Functions
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

#### Get Tags
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

#### Get Brands
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

#### Get Description

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

#### Get Colour Scheme

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

#### Get Objects

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

#### Get Categories

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

#### Get Adult Content

```python

def get_adultContent(id,imganalysis):
    
    adultcontentList = []
    
    adultcontentList.append([id,"is_adult_content",imganalysis.adult.is_adult_content,imganalysis.adult.adult_score])
    adultcontentList.append([id,"is_racy_content",imganalysis.adult.is_racy_content,imganalysis.adult.racy_score])
    adultcontentList.append([id,"is_gory_content",imganalysis.adult.is_gory_content,imganalysis.adult.gore_score])
    
    return pd.DataFrame(adultcontentList, columns = ['imageID','Attribute','Value','Score'])

```

#### Get Text

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

### Other Functions

#### Write to file

```python
#Export DFs to files for storage
def writetofile(df, path, name):
    
    filename = str(path) + name + '.csv'
    print(filename)
    
    try:
        dbutils.fs.cp(filename, 'file:/tmp/' + name + '.csv')
        with open('/tmp/tags.csv', 'a') as f:
           df.to_csv(f, header=False)
        dbutils.fs.mv("file:/tmp/tags.csv","dbfs:/mnt/dev/tmp/ml_p/tags.csv")
       
    except error:
        print("error writing to file " + filename)
```

#### Directory Exists Function

```python
#create a custom function to determine if directory already exists
def dir_exists(dir):
  try:
    dbutils.fs.ls(dir)
  except:
    return False  
  return True
 ```
 
 #### File Exists Function
 
 ```python
 
 def file_exists(filepath):
    try:
        dbutils.fs.ls(filepath)
    except:
        return False
    return True
 
 ```
 
 ## Main Program
 
 ### Define features for use
 
 ```python
 
 #define the features to request from the cognitive services endpoint
features = [VisualFeatureTypes.color,  #get the color schema
            VisualFeatureTypes.categories, #determine categories
            VisualFeatureTypes.description, #generate an AI description of the visual
            VisualFeatureTypes.objects, #identify and locate objects in the picture
            VisualFeatureTypes.brands, #extract any brands found
            VisualFeatureTypes.tags, #tag image
            VisualFeatureTypes.adult] #determine adult content and gore
 
 ```
 
 ### Determine Development Status
 The purpose of this section is to check the development flag widget and sample our data if we're in development.  This avoids processing all of our images while in Dev/Test mode.  If you're using a small file of image URLs this is not required.
 
 ```python
 
 isDevelopment = dbutils.widgets.get("Development Flag")

#controls how many records to use in testing/dev
samplesize =  int(dbutils.widgets.get("Dev Sample Size"))


if isDevelopment == "True":
    imgDF = imagesDF.sample(samplesize)
else:
    imgDF = imagesDF
 
 ```
 
 ### Run Analysis
 
 ```python
 
#Export DFs to files for storage

# define lists for image analysis return dataframes
tagsDFList = []
brandsDFList = []
descriptionDFList = []
colourProfileDFList = []
objectDFList = []
catDFList = []
adultDFList = []
textDFList = []

errorList = []

#itterate through our images Dataframe getting each URL and passing to the Cognitive Services Endpoint.
#in my example style_colour was the name of the unique ID field in my URL dataset.  You should update the ID variable to hold your own unique ID.
for _, row in imgDF.iterrows():

    url = "http:" + row.url 
    id = row.style_colour  #UPDATE THIS

    try:
        result = computervision_client.analyze_image(url,features)
    except:
        print("error with URL: " + url)
        errorList.append(url)
        continue

    tagsDFList.append(get_tags(id,result))
    brandsDFList.append(get_brands(id,result))
    descriptionDFList.append(get_AIdescription(id,result))
    colourProfileDFList.append(get_ColourScheme(id,result))
    objectDFList.append(get_objects(id,result))
    catDFList.append(get_categories(id,result))
    adultDFList.append(get_adultContent(id,result))
    textDFList.append(get_textfromimage(id,url))
 
 ```
 
 ### Write Results to File
 
 ```python

#create directory if required
if dir_exists(dir):
    pass
else:
    dbutils.fs.mkdirs(dir)

path = '/dbfs/' + dir  

if(not len(tagsDFList) ==0):
    tagsDF = pd.concat(tagsDFList)
    tagsDF.to_csv(str(path) + 'tags.csv', header=True, index = False)
if(not len(brandsDFList) ==0):
    brandsDF = pd.concat(brandsDFList)
    brandsDF.to_csv(str(path) + 'brands.csv', header=True, index = False)
if(not len(descriptionDFList) ==0):
    descriptionDF = pd.concat(descriptionDFList)
    descriptionDF.to_csv(str(path) + 'description.csv', header=True, index = False)
if(not len(colourProfileDFList) ==0):
    colourProfileDF = pd.concat(colourProfileDFList)
    colourProfileDF.to_csv(str(path) + 'colours.csv', header=True, index = False)
if(not len(objectDFList) ==0):
    objectDF = pd.concat(objectDFList)
    objectDF.to_csv(str(path) + 'objects.csv', header=True, index = False)
if(not len(catDFList) ==0):
    categoryDF = pd.concat(catDFList)
    categoryDF.to_csv(str(path) + 'categories.csv', header=True, index = False)
if(not len(adultDFList) ==0):
    adultDF = pd.concat(adultDFList)
    adultDF.to_csv(str(path) + 'adult.csv', header=True, index = False)
if(not len(textDFList) ==0):
    textDF = pd.concat(textDFList)
    textDF.to_csv(str(path) + 'texts.csv', header=True, index = False)
 ```
