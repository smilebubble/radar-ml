*Work in progress*

# radar-ml

The goal of this project is to allow you to accurately detect (classify and localize) people, pets and objects using low-power millimeter-wave radar. You will see how self-supervised machine learning techniques, leveraging conventional camera-based object detection, can be used to generate radar-based detection models.

Radar-based object detection has certain advantages over camera-based methods, including increased user privacy, lower power consumption, zero-light operation and more flexible placement. However, radar does not have the resolution of most camera-based systems and therefore may be challenged to distinguish objects that have very similar characteristics. More importantly, there are very few radar datasets which have been used to train widely available object classification or detection models. This is in stark contrast to the ubiquity of visual perception datasets and models.

[Vayyar's](https://vayyar.com/) radar in the [Walabot](https://api.walabot.com/) reference design is used in this project. An object detection model running on Google's Coral [Edge TPU](https://coral.ai/) is used to ground truth radar observations in real-time which are in turn used to train an SVM-based model (DNNs are being explored).

# Overview
A photo of the hardware used for this project is shown below. The Walabot radar is mounted up front and horizontally with a camera located at its top center. The white box contains the Google Coral Edge TPU with the camera connected to it over USB and the black box contains a Raspberry Pi 4 with the radar connected to it over USB. The TPU runs a real-time object detection server over [grpc](https://grpc.io/) that a client on the Raspberry Pi communicates with. The object detection server code can be found [here](https://github.com/goruck/detection_server). The Raspberry Pi handles the processing required to determine if a detected radar target is the same as an object detected by the TPU to establish ground truth. The Pi and the TPU communicate over a dedicated Ethernet link to minimize latency which is critical to accurately determining if a radar target is the same as a detected object. The Pi also runs predictions on novel radar targets using a trained SVM model and can be used to train the SVM model from radar targets with ground truth.

![Alt text](./images/training_setup.jpg?raw=true "test setup.")

# Data Collection and Establishing Ground Truth
This section describes how target samples from the radar are collected and how ground truth (i.e, target identification) is established. There may be multiple targets detected by the radar in a single scan so localization is required as part of this process. A key challenge here is accurately translating the 3-D radar image into the 2-D camera image in real-time since people and pets can move rapidly through the scene. Since the radar image is in 3-D, three orthogonal views of the target can be used to generate the dataset. Choosing the view(s) to include in the dataset and the resolution of each is a tradeoff exercise between model complexity, size and accuracy.

## Radar and Camera Coordinate Systems
The figure below shows the world (X, Y, Z), radar (XR, YR, YZ), camera (XC, YC, ZC), image (x, y) and pixel (u, v) coordinate systems. The camera needs to be placed on the top middle of the radar unit which you can mount either horizontally or vertically (the USB connector is used as a reference, see the Walabot documentation). This ensures that the camera's optical z-axis is aligned with the radar's z-axis and fixed offsets between the camera axis' and the radar axis' can be determined (these are known as the camera extrinsic parameters). Additionally, you should make sure that the camera's angle of view is closely matched to the radar's [Arena](https://api.walabot.com/_features.html#_arena).

![Alt text](./images/coord_system.jpg?raw=true "coordinate systems.")

Its important for you to understand these relationships to convert a 3-D point in the world coordinate system to the radar system which is used as the reference. A point in the world is shown in blue as P(X, Y, Z) and the corresponding point in the radar system is shown in green as PR(XR, YR, Z). Note that Z is the same in both systems since the radar's z-axis is aligned with the camera's optical z-axis and the image plane is placed as close as possible to radar z = 0.

## Radar Target and Camera Object Correlation
You must first calibrate the camera to determine its intrinsic parameters (x and y focal and principal points) and to correct distortion. The intrinsic parameters are used to convert from image to world coordinates. You can use [OpenCV](https://opencv-python-tutroals.readthedocs.io/en/latest/py_tutorials/py_calib3d/py_calibration/py_calibration.html) to do camera calibration quite easily which was done here.

You use the Python module [ground_truth_samples.py](./ground_truth_samples.py) to ground truth radar samples with camera-based observations gathered from the [object detection server](https://github.com/goruck/detection_server). These observations are returned from the server in the form of the centroid coordinates of the detected object's bounding box and its label. The centroid coordinates are converted into the radar frame of reference and then a distance metric is computed between each radar target and the centroid. If the distance is less than a threshold then a match is declared and the radar target's signal and label is stored as an observation.

## Optimizing and Visualizing Radar Dataset

Three orthogonal views of the radar target's return signal can be obtained. You can think of these as 2-D projections of the 3-D target signal on the radar's X-Y plane, the X-Z plane and the Y-Z plane. Any one of these or any combination of them can be used as an observation. Using all three of them will result in the best accuracy but the dataset (~10k ```float32```'s per sample) and resulting SVM model (~10MB) can be large especially if the radar scan resolution is small. You can think of choosing the optimal combination as a training hyperparameter.

You can configure ```ground_truth_samples.py``` to visualize the ground truth process to ensure its working correctly. An example visualization can be found [here](./images/ground-truth-samples.mp4). 

# Training the Model
You use the Python module [train.py](./train.py) to train an SVM on the radar samples with ground truth. The samples are scaled to the [-1, 1] range, the classes balanced and then the model is fitted using a stratified 5-Folds cross-validator. The fitted model is evaluated for accuracy, pickled and saved to disk. Training results from a recent run are shown in the figure below.

![Alt text](./train-results/svm_cm.png?raw=true "svm confusion matrix.")

# Making Predictions
The Python module [predict.py](./predict.py) is used to illustrate how predictions are made on novel radar samples using the trained SVM model. You should make sure the radar Arena, Threshold, Filter Type and Profile are similar to what was used for training. Additionally, the observations need to be constructed from the same orthogonal planes.

You can use this module and the trained model to run predictions on your own Walabot radar unit.

# License
Everything here is licensed under the [MIT license](./LICENSE).

# Contact
For questions or comments about this project please contact the author goruck (Lindo St. Angel) at {lindostangel} AT {gmail} DOT {com}.

# Appendix
References.
1. [Camera Calibration and 3D Reconstruction](https://docs.opencv.org/2.4/modules/calib3d/doc/camera_calibration_and_3d_reconstruction.html)
2. [Camera calibration](http://homepages.inf.ed.ac.uk/rbf/CVonline/LOCAL_COPIES/OWENS/LECT9/node2.html)
3. [Geometric Camera Parameters](https://www.cse.unr.edu/~bebis/CS791E/Notes/CameraParameters.pdf)
4. [To calculate world coordinates from screen coordinates with OpenCV](https://stackoverflow.com/questions/12007775/to-calculate-world-coordinates-from-screen-coordinates-with-opencv)
5. [REAL-TIME HUMAN ACTIVITY RECOGNITION BASED ON RADAR](https://www.google.com/url?sa=t&rct=j&q=&esrc=s&source=web&cd=&cad=rja&uact=8&ved=2ahUKEwi8i9e-h9jpAhXLup4KHRsdCNMQFjABegQIBhAB&url=http%3A%2F%2Fcardinalscholar.bsu.edu%2Fbitstream%2Fhandle%2F123456789%2F201689%2FGuoH_2019-2_BODY.pdf%3Fsequence%3D1&usg=AOvVaw2Ps7ptIYpZGBYPnMQyPArp)
6. [Through-Wall Pose Imaging in Real-Time with a Many-to-Many
Encoder/Decoder Paradigm](https://www.google.com/url?sa=t&rct=j&q=&esrc=s&source=web&cd=&cad=rja&uact=8&ved=2ahUKEwi_19iHiNjpAhXIi54KHUy1A_4QFjABegQIBxAB&url=https%3A%2F%2Farxiv.org%2Fpdf%2F1904.00739&usg=AOvVaw05g69S1Wbh3JwGiKmQPeH8)
7. [Unobtrusive Activity Recognition and Position Estimation for Work Surfaces Using RF-Radar Sensing](https://dl.acm.org/doi/fullHtml/10.1145/3241383)