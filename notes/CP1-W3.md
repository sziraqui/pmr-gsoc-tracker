# Coding Period 1 - Week 3
It's week 4 actually, but week 3 work is pending because of the issues faced in week 2. To complete this week in time, I would focus on the essentials. Only one face detector would suffice to proceed with the project. So I am dropping YOLO (will do it later though) and only implement MTCNN through OpenFace's implementation.
## Binding MTCNN face detector
`FaceDetectorMTCNN` class is part of `LandmarkDetector` class which holds several other face detector models in OpenFace. Only 3 methods need to be binded:
```
1. void Read(string location)
2. void DetectFaces(vector<Rect> bboxes, Mat img, vector<float> confidences, int min_face=60, float t1=0.6, float t2=0.7, float t3=0.7)
3. bool Empty()
```
`location` is the absolute path of a .txt file containing relative location of .dat files that store PNet, ONet and RNet models that make up the MTCNN model.

Equivalent Nodejs/TypeScript methods are:
```
1. Read(location: string)
2. DetectFaces(img: nodoface.Image, min_face: number, t1:number, t2:number, t3:number) : { detections: Array<Rect>, confidences: FloatArray<number> } 
3. Empty(): boolean
```
`cv::Rect` was not binded. Any interface like `{ x:number, y:number, width:number, height:number }` can be used as Rect because I return Napi::Object with key-value pairs as per properties of `cv::Rect`.

### Errors face
The first completed bindings gave this error on runtime:
```
terminate called after throwing an instance of std::bad_alloc
```
All answers on StackOverflow suggest the problem is with vector allocation. But the error was actually in my wrapper that uses `Napi::CallbackInfo`'s `[]` operator. The index I passed to the operator was non-existent. Ideally, error thrown should be something like `index out of bounds` but node-addon-api as of yet does not check for out-of-bound indices. Though its possible to manually check if index exists using `Has(uint index)` method. 

### Memory issue
When running the face detector on a video sample, memory usage increased rapidly (from ~400mb to ~7gb until my laptop freezed) with the number of frames processed. This indicated that I am not releasing the memory used by `cv::Mat` which I allocated on heap. User is responsible to manage memory of anything allocated on heap. So I wrote a simple destructor. But just  `delete`ing `*mat` was not sufficient. I had to also `delete[]` `mat->data` to completely release memory. After this, memory usage was constant (around 400 mb) irrespective of video size.

## Demonstration
To demonstarte, I wrote nodejs wrapper to opencv's draw function `cv::rectangle()`.
I wrote a simple script (nodoface/examples/mtcnn_on_video.js) to verify MTCNN bindings. The script takes two arguments: 1) path to a video file and 2) path to model's .txt file. The text file can be found at `OpenFace/lib/local/LandmarkDetector/model/mtcnn_detector/MTCNN_detector.txt`
### Here's a sample run:
[![IMAGE ALT Video Annotation](https://img.youtube.com/vi/7PUnz9dDZPQ/0.jpg)](https://www.youtube.com/watch?v=7PUnz9dDZPQ)