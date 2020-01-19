# Behavioral-Cloning
In this Project I am going to teach about behavioral cloning. First I am going to drive a car through tracks and then I make a neural network which will watch me and copy me to drive the vehicle autonomous.The task for the neural network is to pick up my driving skill and copy it and do the same.


### How to collect data

Data is collected with the [Udacity self-driving car simulator](https://github.com/udacity/self-driving-car-sim) (see Image below), which has two modes(training and autonomous) and two tracks.In the training mode, the simulator captures images from three cameras mounted on the car: 

1. A center camera, 
2. A right camera
3. A left camera. 

That’s because of the issue of recovering from being off-center. In the simulator, you can weave all over the road and turn recording on and off to record recovery driving. In a real car, however, that’s not really possible. At least not legally. So in a real car, we’ll have multiple cameras on the vehicle, and we’ll map recovery paths from each camera. For example, if you train the model to associate a given image from the center camera with a left turn, then you could also train the model to associate the corresponding image from the left camera with a somewhat softer left turn and you could train the model to associate the corresponding image from the right camera with an even harder left turn. In that way, you can simulate your vehicle being in different positions, somewhat further off the center line(see below image). To read more about this approach, see this [paper](http://images.nvidia.com/content/tegra/automotive/images/2016/solutions/pdf/end-to-end-dl-using-px.pdf) by the friends at NVIDIA that makes use of this technique.



<p align="center"> <img src="./img/2.PNG" style="right;" alt=" udacity self-driving car simulator" width="600" height="400"> </p> 

### Explanation of How Multiple Cameras Work

The image below gives a sense for how multiple cameras are used to train a self-driving car. This image shows a bird's-eye perspective of the car. The driver is moving forward but wants to turn towards a destination on the left.From the perspective of the left camera, the steering angle would be less than the steering angle from the center camera. From the right camera's perspective, the steering angle would be larger than the angle from the center camera. 



<p align="center"> <img src="./img/1.PNG" style="right;" alt=" Explanation of How Multiple Cameras Work" width="600" height="400"> </p> 


When recording, the simulator will simultaneously save an image for the left, center and right cameras. Each row of the csv log file (driving_log.csv) contains the file path for each camera as well as information about the steering measurement, throttle, brake and speed of the vehicle.

During training, you want to feed the left and right camera images to your model as if they were coming from the center camera. This way, you can teach your model how to steer if the car drifts off to the left or the right and during prediction ("autonomous mode"), you only need to predict with the center camera image.


### Data Collection Strategy

If you drive and record normal laps around the track, even if you record a lot of them, it might not be enough to train your model to drive properly. Here’s the problem: if your training data is all focused on driving down the middle of the road, our model won’t ever learn what to do if it gets off to the side of the road and probably when you run your model to predict steering measurements, things won’t go perfectly and the car will wander off to the side of the road at some point.

As the human driver, you can weave back and forth between the middle of the road and the shoulder, but you need to turn off data recording when you weave out to the side, and turn it back on when you steer back to the middle.

As konwn, the vehicle's driving behavior is only as good as the behavior of the driver who provided the data for this reason i implement the below presented strategy to collect accurate data:

1. The car should be driven percisely in the center of the track 1 as much as possible for 2 lap( smoothly around curves).
    
2. The car should be driven counter-clockwise in the center of  the track 1 as much as possible for 1 lap which can help to the generalization of the model.
    
3. The car should be driven clockwise and counter-clockwise on the second track which can help to avoid overfitting or underfitting when training the model.
    
4. A better approach is to record data when the car is driving (clockwise and counter-clockwise) from the side of the road back toward the center line(recovery driving from the sides).
    
    
### Data Visulization

The udacity simulator collects three different images (Center Image,Left Image,Right Image)and corresponding steering angle and logs into a folder with csv file having each image location with the steering angle.

<p align="center"> <img src="./img/4.PNG" style="right;" alt=" Data Visulization
" width="600" height="400"> </p> 


    The image size: (160, 320, 3)
    The number of the images: (24108, 160, 320, 3) 
    The number of the steering angles (24108,) 


### Dataset Balancing

After loading data I ended up with 24108 images. But looking at steering angle distribution it looks like most of the image have zero or close to zero steering angle and because the training track includes long sections with very slight or no curvature, the data captured from it tends to be heavily skewed toward low and zero turning angles. This creates a problem for the neural network, which then becomes biased toward driving in a straight line and can become easily confused by sharp turns. The distribution of the input data can be observed below, the black line represents what would be a uniform distribution of the data points.




<p align="center"> <img src="./img/5.PNG" style="right;" alt=" Dataset Balancing
" width="300" height="200"> </p> 


As you can see data is not a normal distribution or gaussian To overcome this problem I used the following method to correct the data distribution.

To reduce the occurrence of low and zero angle data points, I first chose a number of bins (I decided upon 23) and produced a histogram of the turning angles using numpy.histogram. I also computed the average number of samples per bin (avg_samples_per_bin - what would be a uniform distribution) and plotted them together. Next, I determined a "keep probability" (keep_prob) for the samples belonging to each bin. That keep probability is 1.0 for bins that contain less than avg_samples_per_bin, and for other bins the keep probability is calculated to be the number of samples for that bin divided by avg_samples_per_bin (for example, if a bin contains twice the average number of data points its keep probability will be 0.5). Finally, I removed random data points from the data set with a frequency of (1 - keep_prob).

The resulting data distribution can be seen in the chart below. The distribution is not uniform overall, but it is much closer to uniform for lower and zero turning angles.

<p align="center"> <img src="./img/6.PNG" style="right;" alt=" Dataset Balancing
" width="300" height="200"> </p> 

    The number of images After datset balancing : (7101, 160, 320, 3)
    The number of steering angles After datset balancing : (7101,)

### Data Augmentation

As you can see above, the number of our dataset has decreased from 24108 to 7101, indicating the need for data augmentation to address the challenge of small datasets. To overcome this challenge I implement tow strategies presented below:

* I used histogram equalization to equalize each intensity values
* Flipping Images and Steering angles by using keras preprocessing image function, which is a effective technique for helping with the left turn bias involves flipping images and taking the opposite sign of the steering angle.


### Model

I used the below explained architecture that include;

* First,I cropped all the unneeded part of the image( at the top of all the images are sky-mountain-forest and the hood of the car at the bottom).

* Second I used a lambda layer which is a convenient way to parallelize image normalization. The lambda layer will also ensure that the model will normalize input images when making predictions.

* Then I used 2 Block(as layer 3 and 4) where each block consists of the 2 Residual modules of ResNet(titled “Deep Residual Learning for Image Recognition”) with batch normalization and max pooling layer then foollowed by 1 Flatten layer and 4 dense layer with activation ELU and using dropout to not overfit the network,
* Optimizer Adam with learning rate 1e-4 instead of default 1e-3.

Keras provides the Cropping2D layer for image cropping within the model. This is relatively fast, because the model is parallelized on the GPU, so many images are cropped simultaneously.By contrast, image cropping outside the model on the CPU is relatively slow.

Also, by adding the cropping layer, the model will automatically crop the input images when making predictions in drive.py. The Cropping2D layer might be useful for choosing an area of interest that excludes the sky and/or the hood of the car.The cropping layer crops:

    60 rows pixels from the top of the image
    20 rows pixels from the bottom of the image
    0 columns of pixels from the left of the image
    0 columns of pixels from the right of the image

Since this model outputs a single continuous numeric value, one appropriate error metric would be mean squared error. If the mean squared error is high on both a training and validation set, the model is underfitting. If the mean squared error is low on a training set but high on a validation set, the model is overfitting. Collecting more data can help improve a model when the model is overfitting.

<p align="center"> <img src="./img/3_1.png" style="right;" alt="Architecture" width="300" height="500"> </p> 

### Train Model

I train model with the generator and Parameters I used in my model:

    No of epochs: 5
    Optimizer Used: Adam
    Learning Rate: 1e-4
    Validation Data split: 0.2
    Generator batch size: 32
    Correction factor: 0.25 and -0.25
    Loss Function Used: MSE(Mean Squared Error as it is efficient for regression problem).
    Activation Function: RELU and ELU
    Dropout: 50%
    
When calling model.fit_generator(), Keras outputs a history object that contains the training and validation loss for each epoch. Here is an example of how you can use the history object to visualize the loss which shows like below.

### Autonomous Mode

After training the neural network I will use [drive.py](https://github.com/A2Amir/Behavioral-Cloning/blob/master/drive.py) to drive the vehicle in the autonomous mode of the Udacity simulator to record a video of my vehicle driving autonomously around the track. As seen below, the video includes at least one full lap around the track.  "No tire may leave the drivable portion of the track surface. The car may not pop up onto ledges or roll over any surfaces that would otherwise be considered unsafe (if humans were in the vehicle)."

Below are presented the instructions on how to use [the trained weights](https://github.com/A2Amir/Behavioral-Cloning/blob/master/model.h5) to drive the vehicle in the autonomous mode to record a video.
   
                   python drive.py model.h5 run1
                   
The fourth argument, **run1**, is the directory in which to save the images seen by the autonomous agent and after saving all images based on a timestamp of when the image was seen, this information is used by video.py to create a chronological video of the agent driving using [video.py](https://github.com/A2Amir/Behavioral-Cloning/blob/master/drive.py).


                    python video.py run1
                    
       
<p align="center"> <img src="./img/1.gif" style="right;"  width="600" height="400"> </p> 



