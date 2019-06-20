# Poor Man's Rekognition - Project Tracker   
CCExtractor Development project under Google Summer of Code 2019  

## Overview
A free and Open Source alternative to Amazon Rekognition.
[View proposal](https://github.com/sziraqui/pmr-gsoc-tracker/blob/master/Proposal-PMR-CCExtractor.pdf)   

The project is divided into 3 parts-    

[**1. Nodoface**](https://github.com/sziraqui/nodoface) (Phase 1 : Jun 3 - Jun 24)  
Nodejs bindings for [Tadas OpenFace](https://github.com/TadasBaltrusaitis/OpenFace) to bring all face detection and facial attribute prediction features into PMR   

**2. PMR-Core** (Phase 2 : Jun 28 - July 22)    
Will include    
- TensorflowJS implementation of [ArcFace](https://github.com/deepinsight/insightface).
- Integration of Nodoface and ArcFace TFJS model into a high level API in TypeScript.   
- Real-time video annotation using object tracking and PMR-Core for local execution.    

**3. PMR-web**  (Phase 3: July 26 - August 19)    
A REST API for PMR-Core following Amazon Rekognition API design.  
 
---
## Complete timeline
### Coding Period 1 
**Week 1 (May 27 - Jun 3)** 6hr/d
- [x] Project Tracker
- [x] Environment setup ~~with vcpkg~~ (Compile OpenCV, Dlib, OpenFace)
- [x] NAPI practice
- [x] Select OpenFace classes and methods to be binded

**Week 2 (Jun 3 - Jun 10)** 6hr/d   
Write bindings for:
- [x] Image I/O utilities
- [x] Video I/O utilities

**Week 3 (Jun 10 - Jun 17)** 8hr/d  
Bindings for face detection models
- [ ] ~~YOLO and TinyYOLO models~~ Dropped.
- [ ] MTCNN model (In progress)

**Week 4 (Jun 17 - Jun 24)** 6hr/d  
Bindings for facial attributes
- [ ] Gender classification
- [ ] Age group prediction
- [ ] Eye-gaze estimation (In progress)

**Evalution (Jun 24 - Jun 28)** 6hr/d

### Coding Period 2
**Week 1 (Jun 28 - July 5)** 8hr/d
- [ ] ArcFace implementation in TensorflowJS
- [ ] Train and test on small dataset

**Week 2 (Jul 5 - Jul 12)** 8hr/d
- [ ] Attempt to convert pretrained MXNET model to Tensorflow model
- [ ] Training and testing scripts for large datasets

**Week 3 (Jul 12 - Jul 19)** 6hr/d
- [ ] Nodoface and ArcFace integration
- [ ] Implementation of SORT/DeepSORT (object tracking)

**Week 4 (Jul 19 - Jul 22)** 8hr/d
- [ ] Video annotation with object tracking

**Evaluation (Jul 22 - Jul 26)** 6hr/d

### Coding Period 3
**Week 1 (Jul 26 - Aug 2)** 6hr/d
- [ ] Amazon rekognition Datatypes and Actions
- [ ] LFW face embeddings database

**Week 2 (Aug 2 - Aug 9)** 6hr/d
- [ ] REST API for face detection and verification on images
- [ ] Support for fixed size video

**Week 3 (Aug 9 - Aug 16)** 8hr/d
- [ ] Learn video streaming with sockets
- [ ] Web service for streaming annotation

**Week 4 (Aug 16 - Aug 19)** 6hr/d
- [ ] Manual tests and benchmarking
- [ ] Prepare a release

**Final Evaluation (Aug 19 - Aug 26)** 6hr/d
- [ ] Compile reports on CPU and GPU performance
- [ ] Create and publish Docker image

---
