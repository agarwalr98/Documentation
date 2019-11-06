# Developer Guide

### Prerequisites
* Azure subscription - [create one for free][ 1 ]
* Python 3.x

[1]: https://azure.microsoft.com/free/

### Installation 
Install the client library
```sh
$ pip install --upgrade azure-cognitiveservices-vision-face
```

Install the following python libraries.

```sh
$ pip install urllib3
$ pip install Pillow
$ pip install msrest
$ pip install numpy
$ pip install idna
$ pip install certifi
$ pip install chardet
$ pip install opencv-contrib-python
$ pip install requests
$ pip install requests-oauthlib
$ pip install azure-common
```

### Setting Up

#### Create a Face Azure resource
Create a resource for the Face using [Microsoft Azure](https://docs.microsoft.com/en-gb/azure/cognitive-services/cognitive-services-apis-create-account?tabs=multiservice%2Clinux). or 
  - Get a [trial key](https://azure.microsoft.com/en-gb/try/cognitive-services/#decision) valid for seven days for free. After signup, it will be visible in your account.
  - View your resource on the [Microsoft Azure](https://docs.microsoft.com/en-gb/azure/cognitive-services/cognitive-services-apis-create-account?tabs=multiservice%2Clinux).

#### Create a new python application
Creata a new python script, for example - FaceIdentification.py. Then open it in your preferred text editor and import the following libraries.
```sh
import asyncio, io, glob, os, sys, time, uuid, requests, subprocess
from urllib.parse import urlparse
import math, datetime, logging
import cv2
import requests
from multiprocessing import Queue, Process
from io import BytesIO
from PIL import Image, ImageDraw
from azure.cognitiveservices.vision.face import FaceClient
from msrest.authentication import CognitiveServicesCredentials
from azure.cognitiveservices.vision.face.models import TrainingStatusType, Person, SnapshotObjectType, OperationStatusType
```

Then create variables for Azure's Endpoint and Key. 
```sh
KEY = 'FACE_SUBSCRIPTION_KEY'
ENDPOINT = 'FACE_ENDPOINT'
```

***Note*** - Key and Endpoint shall be visible in your Azure's account.
<!-- ~~~~ -->

### Authenticate the client
Use Key and Endpoint to initiate a client. Create a [CognitiveServicesCredentials](https://docs.microsoft.com/python/api/msrest/msrest.authentication.cognitiveservicescredentials?view=azure-python) object with the help of key, and use it with your endpoint to create a [FaceClient](https://docs.microsoft.com/en-gb/python/api/azure-cognitiveservices-vision-face/azure.cognitiveservices.vision.face.faceclient?view=azure-python) object.
```sh
face_client = FaceClient(ENDPOINT, CognitiveServicesCredentials(KEY))
```

### Create and train a person Group

The following code creates a **personGroup** common to all the students with different **Person** objects. It associates each Person with a set of example images, and then it trains to be able to recognize each person.
***Note*** - Student's Data is taken from a directory with name 'Dataset'. It contains folders with the name as rollnumber of each student. Each folder contains three images to a corresponding student.

#### Create a PersonGroup
To step through this scenario, you need to save the training images to the root directory with a proper name.
This group of images contains three sets of face images corresponding to various people. The code will define the **Person** objects equal to the number of folders in a directory and associate them with the image files in corresponding folder.

Define a lable for the **PersonGroup** object that we will create.
```sh
# SOURCE_PERSON_GROUP_ID should be all lowercase and alphanumeric. For example, 'mygroupname' (dashes are OK).
PERSON_GROUP_ID = 'Your-Person-Group'
# Used for the Snapshot and Delete Person Group examples.
TARGET_PERSON_GROUP_ID = str(uuid.uuid4()) # assign a random ID (or name it anything)
```

**Note** - You can also use list_person_groups to list all preexisting **PersonGroups**

Add the following code to the bottom of your script. It will create a **PersonGroup** with the above defined labels.
```sh
face_client.person_group.create(person_group_id=PERSON_GROUP_ID, name=PERSON_GROUP_ID)
```

#### Assign faces to persons
The following code will create **Person** objects and assign the faces to each **Person** object.
```sh
for student_name in os.listdir('./Dataset'):
    # Define a student friend 
    student = face_client.person_group_person.create(PERSON_GROUP_ID, student_name)
    # Add studentName and corresponding personId i diction
    diction.add(student.person_id ,student_name)
    path = './Dataset' + '/' + student_name
    student_images = student_name + "_images"
    # Take all images of a student in one tensor
    student_images = [file for file in os.listdir(path)]
    print(student_images)
    for image in student_images:
        image_path = path + '/' + image
        w = open(image_path, 'r+b')
        try:
            # Add face to corresponding student
            face_client.person_group_person.add_face_from_stream(PERSON_GROUP_ID, student.person_id, w)
        except:
            print("NO face is detected ")
```

#### Train PersonGroup
Once you've assigned faces, you must train the **PersonGroup** so that it can identify the visual features associated with each of its Person objects. The following code calls the asynchronous **train** method and polls the result, printing the status to the console.

```sh
''' 
Train PersonGroup
'''
print('Training the person group...')
# Train the person group
face_client.person_group.train(PERSON_GROUP_ID)
while (True):
    training_status = face_client.person_group.get_training_status(PERSON_GROUP_ID)
    print("Training status: {}.".format(training_status.status))
    print()
    if (training_status.status is TrainingStatusType.succeeded):
        break
    elif (training_status.status is TrainingStatusType.failed):
        sys.exit('Training the person group has failed.')
    time.sleep(5)
```
### Configuration on raspberry-pi
Installed the Raspbian OS on raspberry-pi.
[Click here](https://thepi.io/how-to-install-raspbian-on-the-raspberry-pi/) for installation instructions.<br>
Create the variables which will be used later.

```sh
#Time in seconds to wait between successive requests to the server.
FETCH_INTERVAL = 60
# Time in seconds to wait before checking for the next scheduled class.
SLEEP_INTERVAL = 2400
# # Time before the scheduled time from which the system should start
# the attendance process.
CLASSTIME_LEFT_MARGIN = {'hour':0, 'minute':10, 'second':0}
# Time after the scheduled time upto which the system should continue
# the attendance process
CLASSTIME_RIGHT_MARGIN = {'hour':0, 'minute':10, 'second':0}
# Time in seconds between successive captures of images while the attendance process is running.
IMAGE_CAPTURE_RATE = 3
# API Endpoint to get the schedule using GET request.
API_GET_SCHEDULE = 'http://10.8.10.247:8000/api/lecture/'
# API Endpoint to submit the identified students' information using POST request.
API_POST_ATTENDANCE = 'http://10.8.10.247:8000/api/lecture/s'
# Format in which the time is sent from the server
FETCH_TIME_FORMAT = '%Y-%m-%dT%H:%M:%S+05:30'
# Endpoint of Microsoft Azure Face API
ENDPOINT = 'https://westcentralus.api.cognitive.microsoft.com/'
# Keys of Microsoft Azure Face API
KEY = 'abddb967436d4acea1a3fd149d3ad3d1'
PERSON_GROUP_ID = 'Your-Person_Group'
# Threshold value of confidence for which a student's attendance be marked 
THRESHOLD = 0.6
```

Run updateConfig.py to change the above variables incase some variables gets modified on the server.


### Identify a face
The following code takes an image with multiple faces and looks to find the identity of each person in the image. It compares each detected face to a PersonGroup, a database of different Person objects whose facial features are known.

#### Get the API status
```sh
training_status = face_client.person_group.get_training_status(PERSON_GROUP_ID)
print("Training status: {}.".format(training_status.status))
```

#### Get a test image
The following code looks in the root of your project for an image test-image-example.jpg and detects the faces in the image.

```sh
'''
Identify a face against a defined PersonGroup
'''
# Reference image for testing against
group_photo = 'test-image-example.jpg'
IMAGES_FOLDER = os.path.join(os.path.dirname(os.path.realpath(__file__)))

# Get test image
test_image_array = glob.glob(os.path.join(IMAGES_FOLDER, group_photo))
image = open(test_image_array[0], 'r+b')
```

#### Detect faces in an image
Faces will be detected and features will be stored in an array.
```sh
# Detect faces
face_ids = []
faces = face_client.face.detect_with_stream(image)
print(faces)
for face in faces:
    face_ids.append(face.face_id)
```
#### Identify faces
The identify method takes an array of detected faces and compares them to a PersonGroup. If it can match a detected face to a Person, it saves the result. This code prints detailed match results to the console.

```sh
# Identify faces
results = face_client.face.identify(face_ids, PERSON_GROUP_ID)
print('Identifying faces in {}')
if not results:
    print('No person identified in the person group for faces from the {}.'.format(os.path.basename(image.name)))
for person in results:
	if person.candidates != []:
	    print('Person for face ID {} is identified in {} with a confidence of {}.'.format(person.face_id, os.path.basename(image.name), person.candidates[0].confidence)) # Get topmost confidence score
	    # print(diction[person.candidates[0].person_id])
```