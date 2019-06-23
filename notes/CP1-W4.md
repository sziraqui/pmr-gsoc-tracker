# Coding Period 1 - Week 4
This week I focused on Eye gaze estimation. This is one of the optional features I proposed to include other features being age group prediction and gender classification. The latter two after not present in OpenFace.

## Eye gaze - Basic concepts
Eye gaze estimation is the task of determining the direction a person is looking at relative to a given point of reference. Usually, 'world' center or 'camera' coordinates are used as reference points ('world' as in 3d video game development). In 2d environments, camera coordinates can be assumed to be the center of image at some depth. If camera properties are available (eg. through video metadata or known otherwise), focal length of camera can be taken as depth. A face landmark model is used to locate the eye and eyeball. The line joining the center of eye ball and center of camera (assummed coordinates of camera) determines direction of gaze. My understanding maybe wrong though.

## Classes to be binded
OpenFace provides GazeAnalysis as a namespace with static methods for eye gaze estimation. It depends on LandmarkDetectorModel which is a CNLF model to predict face landmarks. Eye landmarks from the landmark model are used along with camera properties to get gaze angle.

Source code location:   
 `OpenFace/lib/local/GazeAnalyser/src/GazeEstimation.cpp`   
 Dependencies:
1. FaceModelParameters at     
`OpenFace/lib/local/LandmarkDetector/src/LandmarkDetectorParameters.cpp`      
2. CNLF Face Model at      
`OpenFace/lib/local/LandmarkDetector/src/LandmarkDetectorModel.cpp`
3. `cv::Mat_<int>`
4. `cv::Mat_<float>`

Binding CNLF model brings in few other features too. HOG-based and Haar cascade face detector models are implemented in CNLF. Thus two addtional face detectors will be available while I bind GazeAnalyser.

## Implementation
### Binding typed Mat
Binding a templated class is tricky in node-addon-api. I treat each of int and float type Mat as separate classes and made two classes viz. `IntImage` and `FloatImage` respectively.
### Binding FaceModelParameters
This is actually a `struct` not a class in OpenFace. Bindined as `LandmarkModelConfig` class in Nodoface, only initialising the parameters is important. Most of the attributes are not used in OpenFace itself in its public api. So only few important attributes are binded. On nodejs side, a JSON file can be used to configure model parameters and can be converted to string array to initialise underlying `FaceModelParameters` class.
### CNLF
This is the actual landmark model class that does a lot of important work. Not only GazeAnalyser but also the FaceAnalyser depends on this class. It can be initialised by providing path to .txt model file present in `OpenFace/lib/local/LandmarkDetector/model/`. Default model is `main_ceclm_general.txt`. 
### GazeAnalysis
This was relatively easy to bind after impementing all its dependencies. The binding class is called `GazeAnalyser` in Nodoface. It does not require any internal instance since OpenFace's implementation comprises of few static functions that compute gaze angle provided, CNLF's DetectLandmarks() method was called prior to it. The methos return cv::Point3f and cv::Vec2f. Instead of binding these simple types, I instead resorted to creating plain `Napi::Object`s with required fields in `NapiExtra` namespace.

## Summary
With this the core functionality from OpenFace for PMR is implemented. Before Evaluation phase, I would like to provide TypeScript support in Nodoface. Some refactoring is also needed. And lastly, README should be update to document this binded library.
