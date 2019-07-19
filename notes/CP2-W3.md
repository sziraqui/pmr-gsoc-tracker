# Coding Period 2 - Week 3
I started this week keeping aside the tfjs problems. 
Spent some time on Nodoface and PMR-core integration.

## Nodoface integration with PMR-Core
### Solve SequenceCapture issues
SequenceCapture is a multi-threaded classes. I should have used AsyncWorker to make its bindings. 
It almost always segfaults after all frames from input are read.
To resolve this, I decided to bind opencv's VideoCapture class. This class does not crash unlike SequenceCapture class

### Integrate Nodoface
I exported all nodoface classes and methods and made them importable from within `pmr-core`.
A wrapper class for face detector was also made to provide a single API for face detection

## Add face recogniser
Since all methods I tried to convert to tjs failed, I deicided to use face-api.js's facent;
FaceAPIjs was used in my PoC. It have a resnet 50 model similar to facent for Face recognition.
Faceapi.js uses post 1.x versions of tfjs so I removed all code that 0.8.x version of the code. 
To keep it loosely coupled, I am direclty using FaceRecognitionNet class instead of using global API.
This helped keep the project working without browser polyfills. 
With this, it was possible to run inference by passing a tensor3D instead of HTMLImageElement. 
All models acceot `TNetInput` which is basically a ranked Tensor. I found this after reading `tfjs-image-recognition-base`'s code by the same author.
faceapi.js is a quite solid piece of work.
It now has age-gender and face expression models. But there is a tradeoff in accuracy because it is intended for browsers.

## Some performance tests
On my i7-8650u Kabylake processor, FaceDetection with MTCNN takes from `55-75` ms for a 680x480x3 image.
Face recognition by facenet model takes  `125-135` ms on same dimension image.
Thats significant improvment if we compare with my PoC which took roughly 1.5 seconds to run MTCNN and facenet
