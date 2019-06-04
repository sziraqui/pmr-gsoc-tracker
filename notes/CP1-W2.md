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
