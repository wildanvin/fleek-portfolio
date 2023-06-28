---
title: 'Using Bacalhau with OpenCV'
description: 'Bacalhau Compute Over Data Tutorial'
date: '2022-10-10'
banner:
  src: '../../images/bacalhau/nasa.jpg'
  alt: 'Bacalhau Tutorial'
  caption: 'Photo by <u><a href="https://unsplash.com/@nasa?utm_source=unsplash&utm_medium=referral&utm_content=creditCopyText">NASA</a></u>'
categories:
  - 'CoD'
  - 'Tutorial'
keywords:
  - 'Bacalhau'
  - 'Docker'
---

You can read this post on [notion](https://www.notion.so/Using-Bacalhau-with-OpenCV-2b3e473de89a41548a5fb77c4f2fd073) too.

Bacalhau is a platform that allows you to run virtually any program you want (from digital image processing to machine learning and AI) over data stored in IPFS/Filecoin.

For this tutorial I will show you how I used OpenCV with Bacalhau to compute over an image and get to know how “green” it is. We want to do this because we are assuming that the image belongs to a reforestation project and we want to determine the “health” of it. We will keep it simple and use an algorithm called color segmentation.

We will be following the code in this [repository](https://github.com/wildanvin/bacalhau-tutorial) in GitHub. The basic steps that we will walk through are:

1. Run the script locally.
2. Make a Docker image.
3. Upload files to IPFS.
4. Use Bacalhau.

## 1. Run the script locally

The first step is to have our script working locally in our computer. For our example we are going to need numpy in addition to OpenCV. You can find instructions on how to install them in the [repository](https://github.com/wildanvin/bacalhau-tutorial).

In order to use color segmentation we need to transform the image into HSV format. HSV stands for hue, saturation and value, it is a way to represent an image that facilitates separate it by colors.

```python
import cv2
import numpy as np

image = cv2.imread("../testImages/dron2.jpg")
image = cv2.cvtColor(image, cv2.COLOR_BGR2RGB)
hsv_image = cv2.cvtColor(image, cv2.COLOR_RGB2HSV)
```

Next, we specify a range of green values in HSV format and create a “mask” using OpenCV .

```python
#Select the range of colors in HSV
light_green = (50, 50, 50)
dark_green = (120, 255, 255)

# Segment image using inRange() function
mask = cv2.inRange(hsv_image, light_green, dark_green)
```

Finally, we do a bit-wise AND operation to obtain just the pixels where the image is green.

```python
# Bitwise-AND mask and original image
result = cv2.bitwise_and(image, image, mask=mask)
```

The process can be summarized in this image:

![Project Mind Map](../../images/bacalhau/bacalhau-map.jpg '[Project mental map](https://file.notion.so/f/s/35fc388a-91e9-47a4-a9e1-44c2620a7b0a/bacalhau-tutorial-0.jpg?id=9f245668-0f66-40f5-adcb-7cdeeee74964&table=block&spaceId=2b50c96a-2efe-4814-9663-16137417799d&expirationTimestamp=1688012031162&signature=ZxjZon9BXWu47AwKafngTTfKq9KH54jg-B7lDhipwpk&downloadName=bacalhau-tutorial-0.jpg).')

With this information we can calculate the percentage of green in the image. I used the following code:

```python
#Calculate a score based on how much "green" is in the image
greenPixels = np.count_nonzero(mask == 255)

# height and width in image
height = image.shape[0]
width = image.shape[1]

score = round((greenPixels*10/(height*width)),1)
print('Green score is:', score)
```

Although the `result` image is not used to calculate the score, it helps to reassure visually that the algorithm is working properly.

I added the next lines to the script to save the images as `jpgs` and the score in a `txt` file:

```python
#Save the result image:
cv2.imwrite('result.jpg',result)
cv2.imwrite('mask.jpg',mask)

print('Saving the result image as result.jpg')

#Save the green score in a file
f = open("score.txt", "w")
f.write(str(score))
f.close()
```

### Color segmentation algorithm

I borrowed heavily from this color segmentation [tutorial](https://realpython.com/python-opencv-color-spaces/). Feel free to check it out if you want to better understand the algorithm and visualize it step by step. This is the [GitHub repo](https://github.com/realpython/materials/blob/master/opencv-color-spaces/finding-nemo.py).

## 2. Make a Docker image

Now that our script is working locally we need to make a Docker image with it and upload it to [docker hub](https://hub.docker.com/) (you will need to create an account, if you don’t have one already). Needless to say that you need Docker in your machine for this step. I’m using Linux and found this nice [tutorial](https://www.digitalocean.com/community/tutorials/how-to-install-and-use-docker-on-ubuntu-20-04) from Digital Ocean, please check the official [documentation](https://docs.docker.com/get-docker/) if you are using mac OS or Windows.

If you are new to Docker, it is a basically a technology that packs your code inside a very minimalist OS with all the dependencies your script needs so it can run virtually anywhere with out having to install those dependencies again. We call this minimalist OS an image.

To create the Docker image we only need two files: `color-seg.py` and `Dockerfile`, you can find both in the `2-docker-image` directory in the [repo](https://github.com/wildanvin/bacalhau-tutorial). Lets take a look at those files:

```python
import cv2
import numpy as np
import sys

#Read the image
image = cv2.imread(sys.argv[1])
image = cv2.cvtColor(image, cv2.COLOR_BGR2RGB)
hsv_image = cv2.cvtColor(image, cv2.COLOR_RGB2HSV)

#Select the range of colors in HSV
light_green = (50, 50, 50)
dark_green = (120, 255, 255)

# Segment image using inRange() function
mask = cv2.inRange(hsv_image, light_green, dark_green)

# Bitwise-AND mask and original image
result = cv2.bitwise_and(image, image, mask=mask)

#Calculate a score based on how much "green" is in the image
greenPixels = np.count_nonzero(mask == 255)

# height and width in image
height = image.shape[0]
width = image.shape[1]

score = round((greenPixels*10/(height*width)),1)
print('Green score is:', score)

#Save the result image:
print('Saving the result image as result.jpg')
cv2.imwrite('./outputs/result.jpg',result)
print('Saved!')

#Save the green score in a file
f = open("./outputs/score.txt", "w")
f.write(str(score))
f.close()
print('Saving score in outputs/score.txt')
```

This `color-seg.py` is slightly different from the one in the step 1. The difference is that we are using `import sys` at the beginning because we will pass the path to the picture as an argument from the console. Also, at the end, we are saving our files in the `outputs` directory in the Docker image.

Now lets take a look at the `Dockerfile`

```docker
FROM python:3.8-slim-buster

WORKDIR /

RUN pip install numpy
RUN pip install opencv-python-headless

COPY . .
```

Its function is to tell Docker how to create the image. I will explain briefly what each command do:

- FROM: tells Docker what the base image is, in this case we will start with an image that already has python 3 in it
- WORKDIR: creates the named directory and `cd` into it. In our case we will be working in the root directory of the Docker image
- RUN: tells Docker to execute whatever command follows as if we were doing it ourselves at a command prompt inside the image.
- COPY: copies content from the host directory (our PC) to a destination directory in the image’s file, it is in the form `COPY <src> <dest>`with this command we copy the `color-seg.py` into the Docker image.

Before we create the Docker image lets log in into docker hub from the command line:

```bash
sudo docker login --username <your_username>
```

To create the image we run:

```bash
export IMAGE=<your_dockerhub_username/your_image_name>
sudo docker build -t ${IMAGE} .
sudo docker image push ${IMAGE}
```

If everything went smooth you should see the name of your Docker image in the web interface of docker hub.

## 3. Upload files to IPFS

We have now our Docker image ready to be used with Bacalhau but before we do that we have to upload our pictures to IPFS. There are a lot of services that you can use, some really good ones are:

- [web3storage](https://web3.storage/)
- [nft.storage](https://nft.storage/docs/quickstart/)
- [pinata](https://app.pinata.cloud/)
- [IPFS desktop](https://docs.ipfs.tech/install/ipfs-desktop/) (it is really handy to quickly check data on IPFS)

One **really important** detail to mention is that you need to upload a folder with the pictures, otherwise when we execute Bacalhau it wont work.

For this example I used Pinata to upload the `testImages` folder that you can find in the [repo](https://github.com/wildanvin/bacalhau-tutorial). After you upload your files you will get a CID. We will need this CID when we are using Bacalhau.

![Pinata interface showing the CID of a folder of files](../../images/bacalhau/pinata.png '[Pinata interface showing the CID of a folder of files](https://file.notion.so/f/s/4d30e4c4-def3-4c12-8866-cd8db9ead3d1/Untitled.png?id=e215c3be-dda9-4894-845b-b1567b2f6539&table=block&spaceId=2b50c96a-2efe-4814-9663-16137417799d&expirationTimestamp=1688012336989&signature=j5sXMqd_JrH-zi8uBgUsMbL_Hjlw4pP1c2uwrhpCIks&downloadName=Untitled.png).')

## 4. Use Bacalhau

Finally we can use Bacalhau to run our script on the files that we uploaded to IPFS, we have to install Bacalhau first:

```bash
curl -sL https://get.bacalhau.org/install.sh | bash
```

Run Bacalhau with the Docker image that we created:

```bash
bacalhau docker run \
-v <the_CID_of_the_folder_uploaded_to_IPFS>:/inputs \
<name_of_your_image_on_docker_hub> \
-- sh -c 'python3 color-seg.py ./inputs/dron2.jpg'
```

As an example, if you run the following command:

```bash
bacalhau docker run \
-v Qmbjc3dqLoJ7FgF5RHqUxQEmHujqvmURbprXnfjiEwxeAV:/inputs \
wildanvin/bacalhau-tutorial \
-- sh -c 'python3 color-seg.py ./inputs/dron2.jpg'
```

You should get a response similar to this:

![1st response from Bacalhau](../../images/bacalhau/response1.png '[Response from Bacalhau](https://file.notion.so/f/s/13d70d4e-0e88-417a-839a-a7b5fec7b8b6/Untitled.png?id=f2a62ed2-54e1-4838-8b77-6f25e5d523e6&table=block&spaceId=2b50c96a-2efe-4814-9663-16137417799d&expirationTimestamp=1688012629637&signature=zK02pjGHU_nLOKnVpnvgGMkasQtP_FFL8xPjmPvHYa0&downloadName=Untitled.png).')

Bacalhau mounts a volume inside the image at an `inputs` directory with the contents of the CID, then gets the docker image from docker hub (wildanvin/bacalhau-tutorial in this case) and runs the shell script.

As Bacalhau says, you can run `bacalhau describe f1fc49e5-5665-4c24-94fa-1c54f6687b3c` to get more info about this job, the output is large but look for `PublishedResults` and you will find the CID of your output images already on IPFS:

![More info from Bacalhau](../../images/bacalhau/response2.png '[Get more info from Bacalhau](https://file.notion.so/f/s/a06768f6-50b3-4115-a9ba-579513b98c07/Untitled.png?id=3276c917-b654-4250-8f99-f0c38a9ddfef&table=block&spaceId=2b50c96a-2efe-4814-9663-16137417799d&expirationTimestamp=1688012746537&signature=Kpgx8694Eco9yCpLJEyVE842rYaZIw8tW4xJyHjJAac&downloadName=Untitled.png).')

You can enter this CID in IPFS Desktop and see your outputs. If you didn’t install IPFS Desktop you can watch your CID at this link:

- https://ipfs.io/ipfs/<replace_with_your_CID>

For our example it would be:

- [https://ipfs.io/ipfs/QmanyDhcoTLJhrSFrEe3oSvqYmWMYNWZUMdqM9fQY7VrGF](https://ipfs.io/ipfs/QmanyDhcoTLJhrSFrEe3oSvqYmWMYNWZUMdqM9fQY7VrGF)

It may take a bit of time to load.

## Conclusion

This was a basic example on how to use Python and OpenCV to compute over data with Bacalhau, of course you can add the algorithm of your preference and do a lot more (compute over video, do machine learning, AI, the sky is the limit).

Once the data is on IPFS different parties can run different computations over the data, this opens the door to new collaborative research and decentralized science DeSci, I am personally very curious and excited to see what people will build.

We actually built a more robust implementation of this idea of reforestation projects for the Chainlink Hackaton, it was a really nice experience and we learned a lot. You can check it out [here](https://devpost.com/software/hypertally).

If you have any feedback feel free to reach out on [twitter](https://twitter.com/wildanvin). If you want to learn more about Bacalhau check out the [docs](https://docs.bacalhau.org/) and if you have a deep technical question you can ask the Bacalhau team on [slack](https://app.slack.com/client/TEHTVS1L6/C02RLM3JHUY/thread/C02RLM3JHUY-1668425873.929059), they respond very quick.

That is all from me, hope you learned something **✌️**.
