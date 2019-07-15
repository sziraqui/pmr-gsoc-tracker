# Coding Period 2 - Week 2
This week's goal is to implement ArcFace or atleast port it from existing implementations
I found two Tensorflow implementations of ArcFace  long after submitting GSoC proposal. 
The initial plan was to implement ArcFace in Tensorflow and and save it as a Tensorflowjs model.
Now that we have pre-trained ArcFace models already in Tensorflow, instead of reinventing the wheel, I would try to save them into tensorflowjs model
For the sake of learning, I would try to implement ArcFace loss and make scripts for training and fine tuning the existing models

## Using InsightFace_TF model [1]
This is the first known TF implementation of ArcFace. At first, I found it would be easy to convert this into tfjs but things are not so simple.
It uses TensorLayer library and two custom Layers. The implementation is undoubtedly quite complex. 
After a lot of reading about new Resnet architecture - ResNet-SE, I was able to partially code it in tfjs layers API.
The author made custom layers viz `BatchNormLayer` and `ElementwiseLayer` for ResNet-SE. BatchNormLayer actually doinig everything that the built-in `tf.layers.batchNormalisation()` does. However, there's a catch. Authors's BathcNormLayer has slightly different behaviour during training and inference.
I could not find any equivalent of ElementWiseLayer. 

## Converting pretrained model from InsightFace_TF [3]
The author provides a lot of pre-trained models. Unfortunately, they are uploaded on a Chinese website. 
I have to register with a Chinese phone number to download it. But that's not possible for me. 
Another implementation provides a FrozenModel which I was able to download. Surprisingly, this frozen model has node names identical to InsightFace_TF.
I am assuming both are using the same model architecture.

To convert, I installed tensorflowjs (0.8.5) and ran the arguments for frozen model
```
    tensorflowjs_converter \
    --input_format=tf_frozen_model \
    --output_node_names='resnet_v1_50/E_DenseLayer/W' \
    --output_format=tensorflowjs insightface.pb output_dir
```
Note: tensorflowjs is not available on conda repos. I used pip from conda to install it. 
Latest 1.x versions have deprected frozen model so there is no other way to convert a frozen model.

This results in following files:
```
- tensorflowjs_model.pb
- weights_manifest.json
- group1-shared*of13
```
The tfjs documentation mentions the script will create a model.json file and some binary files.
To load the model in tfjs, I need a model.json file. I digged into `0.8.5` version of their github repo and found instructions to load .pb and .json model
The release notes of 1.x version mentions that model.json combines .pb and .json files representing model definition and model weigths respectively.
But I did not find any way to combine these two files. The only workaround was to use an older version of tfjs-node. 
Thats `tfjs-node` version `0.3.2` with `tfjs` version `0.15.3`. 
So finally to load arcface frozen model I ran
```
let model = tf.loadFrozenModel('file://path_to_dot_pb_file', 'file://path_to_manifest_json_file')
```
But this does not work when running inference. I get error pointing the input must contain two tensors while I was passing only one.
Interesting because from InsightFace_TF code I can see the model takes only one tensor as input.

## Fallback to Facenet [4]
I have to proceed to next goals. So I decided to use pre-trained Facenet.
Just like other Tensorflow models, Facenet also uses the old Session code. It provides a frozen model too.
I could not convert it though. The code is quite dated for a succesful conversion. I also explored and tried many other options including a C++ implementation [5].
But none were compatible with our use case.

# References
[1] [InsightFace_TF - ArcFace implementation in TF](https://github.com/auroua/InsightFace_TF)       
[2] [InsightFace frozen model from a 3rd-party](https://github.com/AIInAi/tf-insightface)       
[3] [Tensorflowjs version that supports frozen model](https://github.com/tensorflow/tfjs-converter/tree/0.8.x)      
[4] [Facenet by davisking](https://github.com/davidsandberg/facenet)    
[5] [Facenet C++ implementation](https://github.com/mndar/facenet_classifier)
