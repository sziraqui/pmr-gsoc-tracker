# Coding Period 2 - Week 1

This week is for learning Tensorflow, especially it's Nodejs version and the Layers API. 
There are tons of tutorials on TensorflowJS but most of them only explain it for running in browser. 
Browser based examples read images from `HTMLImageElement`. But I need to read image natively in Node. 
So I need to look for examples that deal with image processing in tfjs-node.

## Understanding TFJS in Node context
tfjs-node is a Nodejs binding to `libtensorflow` - the Tensorflow C library. 
It has a typescript interface too. This is pretty much how I made bindings to OpenFace and some OpenCV functions. 
I am not sure if it uses NAPI or something else. 
To learn it's API, I am reading their Medium articles and code samples from tfjs-examples repository. 
I have little experience with `Tensorflow` in `Python` but the lack of `numpy` and `Keras` API makes it necessary to learn TFJS as if it was a new framework. 
The Layers API is somewhat similar to Keras at first glance.

### TFJS package differences:
1. `@tensorflow/tfjs-core`: Hardware accelerated operations with Eager execution and low level APIs. This contains everything from the original deeplearnJS. It uses WebGL when GPU is available to improve performance in browser.
2. `@tensorflow/tfjs`: All operations are performed in vanilla JavaScript on CPU without libtensorflow. It provides Layers API along with low level APIs from tfjs-core. 
3. `@tensorflow/tfjs-node`: Uses libtensorflow bindings to Nodejs. All operations are performed on CPU even if GPU is available. Depends on tfjs for its API. But this version cannot use WebGL.
4. `@tensorflow/tfjs-node-gpu`: Uses libtensorflow compiled with CUDA and cuDNN backend for GPU based acceleration.
5. `tensorflowjs` from `pip`: Python package for converting models to tfjs* loadable format.

### Installation
TFJS is available for Node via npm. So installation is easy.
```
$ npm install @tensorflow/tfjs-node
```
But it depends on tfjs-core which is a peer dependency and must be installed separately.
```
$ npm install @tensorflow/tfjs-core
```
I expected these commands to take time to compile `libtensorflow` but they finished quite quickly. 
I guess pre-built binaries are being downloaded and used. It doesn't have AVX2/SSE4 optimisations. 
Hence, I will compile libtensorflow from source [2]

## Converting Nodoface Image to tensor
There is no numpy in Node. 
Closest package is `numjs` but Tensorflow does not support it to my knowledge. 
The mnist-node [3] example reads single channel image pixels into a `Float32Array` and converts it into tensor using `tf.tensor4d`. 
[Code link](https://github.com/tensorflow/tfjs-examples/blob/master/mnist-node/data.js#L166). 
Nodoface needs to be updated to output Image data as `TypedArray`.

# References
[1] [Medium articles on TensorflowJS](https://medium.com/tensorflow/tagged/javascript)  
[2] [Compile Tensorflow C library](https://github.com/tensorflow/tensorflow/blob/master/tensorflow/tools/lib_package/README.md)     
[3] [tfjs-examples/mnist-node](https://github.com/tensorflow/tfjs-examples/tree/master/mnist-node)      
