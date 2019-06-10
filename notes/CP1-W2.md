# Coding Period 1 - Week 2
## My guidelines to binding classes
- For every class in OpenFace, create a class in namespace 'Nodoface' and let the class retain its original name.
- Let the class extend Napi::ObjectWrap to allow sending and receiving its instances from/to node and c++.
- Hold reference to actual class in binding class.
- Bind all public methods. Private methods anyways are not visible to end users.
- Create Getter and Setter for public class attributes.
 
## Binding OpenFace I/O related classes
1. ImageCapture
2. SequenceCapture

These classes have methods that accept/return opencv's Mat, Rect and Point types.

### The dependency hell
I have to write bindings for ImageCapture, but it depends on cv Mat, so write bindings for cv Mat, but that depends on Rect, so write bindings for Rect too but hey Rect depends on Point... 
This dependency hell may go deeper.
### The Solution
Methods either have dependent type in method call or its return type. 
1. Dependency in return type:    
   Eg. from ImageCapture class
   ```
   cv::Mat_<uchar> GetGrayFrame();;
   ```
   Corresponding Napi binded method signature
   ```
   Napi::Value GetGrayFrame(const Napi::CallbackInfo& info);
   ```
   Napi::Value needs to be replaced with something like a cv Mat. But to avoid dependency hell, I can simply return a Uint8Array and *assume* that the nodejs side of it knows the returned array represents an image.    
   Equivalent method in TypeScript/nodejs
   ```
   getGrayFrame(): Uint8Array
   ```
   Things get complicated when you have multi-dimensional array, eg. an RGB image. I am yet to find a solution to returning a multi-channel cv::Mat. I will try to avoid writing bindings for cv::Mat as far as possible.

2. Dependency in method call:   
   Eg. from ImageCapture class
   ```
   bool OpenImageFiles(const std::vector<std::string>& image_files, float fx = -1, float fy = -1, float cx = -1, float cy = -1);
   ```
   Corresponding Napi binded method signature
   ```
    Napi::Boolean OpenImageFiles(const Napi::CallbackInfo& info);
   ```
   Equivalent TypeScript/nodejs function would be
   ```
   openImageFile(imageFiles: Array, fx?: number, fy?: number, cx?: number, cy?: number): boolean
   ```
   So instead writing bindings to std::vector, we can just pass a regular JavaScript type to our binded function and let the function *convert* input array into vector. 

### How to handle class attributes?
Create getter and setter methods wrapped with Napi.     
Eg. From ImageCapture class, Attribute-      
`int image_height`      
Getter-
```
Napi::Number Nodoface::ImageCapture::GetImageHeight(const Napi::CallbackInfo& info) {
    return Napi::Number::New(info.Env(), this->imageCapture->image_height);
}
```
Setter-
```
Napi::Boolean Nodoface::ImageCapture::SetImageHeight(const Napi::CallbackInfo& info) {
    Napi::Env env = info.Env();
    if(info.Length() != 1 || !info[0].IsNumber()) {
        JSErrors::SetterError(env, JSErrors::NUMBER);
    }
    Napi::Number num = info[0].As<Napi::Number>();
    this->imageCapture->image_height = num.DoubleValue();
    return Napi::Boolean::New(env, true);
}
```
Only members visible to end user need to be binded. Leave private members alone.

## Build errors
I am facing the following error whenever the project include anything from opencv
```
internal/modules/cjs/loader.js:730
  return process.dlopen(module, path.toNamespacedPath(filename));
                 ^
```
After a lot of trial/error and reading online to find some clue, I figured out the problem is because of static libraries. 
OpenFace CMakeLists.txt does not compile shared libs (.so) but static libs (.a) which require additional syntax in binding.gyp not known to me from its documentation. Found the correct way to include static libraries from this [issue](https://github.com/nodejs/node-gyp/issues/328)[1].   
Updated binding.gyp 'libraries' list
```
'libraries': [
            '-L/usr/local/lib',
            '-Wl,-rpath, /usr/local/lib/libUtilities.a',
            '<!@(pkg-config opencv --libs)'
        ],
```
This resulted compiler error while building with node-gyp
```
/usr/bin/ld: /usr/local/lib/libUtilities.a(ImageCapture.cpp.o): relocation R_X86_64_PC32 against symbol `_ZSt4cout@@GLIBCXX_3.4' can not be used when making a shared object; recompile with -fPIC
/usr/bin/ld: final link failed: Bad value
collect2: error: ld returned 1 exit status
nodoface.target.mk:147: recipe for target 'Release/obj.target/nodoface.node' failed
```
This error wasted another day.     
node-gyp by default creates a dynamic library so any static library that it depends on must be compiled with `-fPIC` flag.
Since OpenFace's cmake file does not use -fPIC so I added the following line to its CMakeLists.txt [2]
```
set(CMAKE_POSITION_INDEPENDENT_CODE ON)
```
Build succesful! But when importing the addon in node, I get the old opendl error again. This error occurs when a source file or lib is missing [3]. I guess the missing lib was opencv because it was also statically compiled. pkg-config command in binding.gyp returns clfags which work only for shared libraries. OpenCV was compiled as per OpenFace's instructions and this resulted in statically compiled OpenCV. Now recompiling opencv with its default shared library configuration gave me some relief. But here comes another issue. Libboost which is a dependency of both OpenFace and OpenCV uses a non-standard installation prefix on Ubuntu. OpenCV cmake script couldn't find two important libs .vz libboost and openblas. See [this](https://github.com/opencv/opencv/issues/9953) issue for details. Finally, I decided to compile libboost and openblas from source. I hope this will help solve the opendl error I am facing for the past 4 days (first encountered on June 6)


## References
[1] [Linking to static libraries node-gyp issue 328](https://github.com/nodejs/node-gyp/issues/328)     
[2] [Setting -fPIC to cmake build](https://stackoverflow.com/questions/38296756/what-is-the-idiomatic-way-in-cmake-to-add-the-fpic-compiler-option)    
[3] [dlopen error with statically compiled opencv](https://github.com/justadudewhohacks/opencv4nodejs/issues/113)    
[4] [OpenCV cannot find installed openblas headers](https://github.com/opencv/opencv/issues/9953)