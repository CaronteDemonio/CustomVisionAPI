# How to upload multiple images with one batch in Microsoft Customer Vision service

This sample code will let you upload multiple image files, and their related tags, in one batch. It can be useful when you have to use multiple files for your object classification project.
Let's start by creating the project. Remember to replace the keys with your own keys! You might also want to change the project name.

```
from azure.cognitiveservices.vision.customvision.training import training_api
from azure.cognitiveservices.vision.customvision.training.models import ImageFileCreateEntry

# Replace with a valid key
training_key = "<YOUR TRAINING KEY>"
prediction_key = "<YOUR PREDICTION KEY>"

trainer = training_api.TrainingApi(training_key)

# Create a new project
print("Creating project...")
project = trainer.create_project("Object Classification Multiple Files")
```

Now let's create the tags. I am using only two tags, but of course you can create as many as you need.

```
# Make two tags in the new project
up_tag = trainer.create_tag(project.id, "Up")
down_tag = trainer.create_tag(project.id, "Down")
```

Now I am reading the images from a directory - again, remember to change the directory with the one you will use - and then 
I am uploading the images into the Custom Vision service, adding the related tags.

```
import os

# Maximum number of images to be uploaded in each single batch
MAX_IMAGES = 30

# Total number of uploaded images
total_images_uploaded = 0

# Change the directory with the one you will be using
up_dir = "Images\\Train\\Up"
list_of_images = []
list_of_tags = []
image_count = 0
# Upload all the images in the folder with the same 'Up' tag
# There is a limit of 64 images and 20 tags for each image
for image in os.listdir(up_dir):
    with open(os.path.join(up_dir, image), mode="rb") as img_data: 
        list_of_images.append(ImageFileCreateEntry(name=os.path.basename(image), contents = img_data.read()))
        list_of_tags.append(up_tag.id)
        # Updates the image count the upload the batch if the count reached MAX_IMAGES
        image_count += 1
        total_images_uploaded += 1
        if (image_count == MAX_IMAGES):
            trainer.create_images_from_files(project.id, list_of_images, tag_ids = list_of_tags)
            list_of_images = []
            list_of_tags = []
            image_count = 0
trainer.create_images_from_files(project.id, list_of_images, tag_ids = list_of_tags)
print("Uploaded ", total_images_uploaded, " images. Now starting second batch")

# Change the directory with the one you will be using
down_dir = "Images\\Train\\Down"
list_of_images = []
list_of_tags = []
image_count = 0

# Upload all the images in the folder with the same 'Down' tag
# There is a limit of 64 images and 20 tags for each image
for image in os.listdir(down_dir):
    with open(os.path.join(down_dir, image), mode="rb") as img_data: 
        list_of_images.append(ImageFileCreateEntry(name=os.path.basename(image), contents = img_data.read()))
        list_of_tags.append(down_tag.id)
        image_count += 1
        total_images_uploaded += 1
        if (image_count == MAX_IMAGES):
            trainer.create_images_from_files(project.id, list_of_images, tag_ids = list_of_tags)
            list_of_images = []
            list_of_tags = []
            image_count = 0
trainer.create_images_from_files(project.id, list_of_images, tag_ids = list_of_tags)
print("Uploaded ", total_images_uploaded, " images. Job completed!")
```

Now let's train the model. The code is exactly the same provided in the sample code documentation.

```
import time

print("Training...")
iteration = trainer.train_project(project.id)
while (iteration.status != "Completed"):
    iteration = trainer.get_iteration(project.id, iteration.id)
    print("Training status: " + iteration.status)
    time.sleep(1)

# The iteration is now trained. Make it the default project endpoint
trainer.update_iteration(project.id, iteration.id, is_default=True)
print("Done!")
```

And finally let's predict the tags of some images. The only difference here from the sample code provided in the service is that here I am using the "no store" API, which does not store the predicted image in the service. You can change it with the standard API call if you need to store your predicted images within the Custom Vision service.
Unfortunately there is not currently an API to predict multiple images with one call.

```
from azure.cognitiveservices.vision.customvision.prediction import prediction_endpoint
from azure.cognitiveservices.vision.customvision.prediction.prediction_endpoint import models

# Now there is a trained endpoint that can be used to make a prediction

predictor = prediction_endpoint.PredictionEndpoint(prediction_key)

# Open the sample image and get back the prediction results.
with open("Images\\test\\200000.png", mode="rb") as test_data:
     results = predictor.predict_image_with_no_store(project.id, test_data, iteration.id)

# Display the results.
for prediction in results.predictions:
    print("\t" + prediction.tag_name + ": {0:.2f}%".format(prediction.probability * 100))
```
