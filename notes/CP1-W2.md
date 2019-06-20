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

### Finally all errors are gone!
OpenFace was compiled as static with PIC but still I faced the same errors. Either I wrote binding.gyp incorrectly to include static OpenFace lib or node-gyp just doesn't support linking against static libs. Whatever be the reason, I decided to recompile OpenFace as shared lib. I had to make some changes to its CMakeLists.txt files. The modified OpenFace project is on [my fork](https://github.com/sziraqui/OpenFace/tree/dynamic-compile). The binding.gyp file will now have
```
'libraries+': [
            '-L/usr/local/lib',
            '-lUtilities',
        ],
'include_dirs+': [
            '/usr/local/include',
            '/usr/local/include/OpenFace'
        ],
```
within `conditions` array for linux.

## Initial test of ImageCapture bindings
`nodoface/examples/image_capture_example.js` runs all binded methods. A directory containing images was passed to ImageCapture.Open() with -fdir switch. Before executing NextImage(), getProgress returns 0 and Width and Height attributes are garbage values. After executing NextImage(), progress attribute is increased, image height and width are 479 and 639. The actual image dimensions are 480 and 640. Values were decremented with setters and they are working. Next task is to convert between cv::Mat and Napi::Array.
```
Attempting to read from directory: /home/sziraqui/Projects/OpenFace/samples/image_sequence
Actual properties 
progress 0 
Image Ht 32653 
Image Wd 1973120160 
Fx -1 
Fy -1 
Cx -1 
Cy -1 
Next Image true 
Gray frame true
New properties 
progress 0.03333333333333333 
Image Ht 479 
Image Wd 639 
Fx 501 
Fy 501 
Cx 321 
Cy 241 
Next Image true 
Gray frame true

```

## Handling cv::Mat types
C++ side uses `cv::Mat` to represent images. There is no standard way in node to efficiently represent an image. Canvas could have been used if there was no overhead of DOM. The simplest way I think is to use `Uint8Array`. The image will be flattened in C++ side into a 1-D array (pointer to uint8_t) and then converted to either a `Napi::TypedArrayOf<primitive_type>` or    `Napi::ArrayBuffer`. Both can be passed to nodejs side for using elsewhere.
### Image from Nodejs to C++
Image as JS Uint8Array has underlying ArrayBuffer. We can get reference to the ArrayBuffer on C++ side and get a pointer to its underlying primitive array;
```
Napi::Uint8Array img = info[0].As<Napi::Uint8Array>();
Napi::ArrayBuffer buf = img.ArrayBuffer();
uint8_t * raw = reinterpret_cast<uint8_t*>(buf.Data());
```
Refer [6] for a concrete example
### Image from C++ to Nodejs
`cv::Mat` is stored as a 1-D array. Dimension of `cv::Mat` is atleast 2 (rows, cols) and usually goes till 4 (rows, cols, channels, alpha). I do not need to handle `cv::Mat` of higher dimensions. Each row is guaranteed to have contigious memory allocation. After a row, some padding may be present. This is called `step_size` in CV jargon. Each row may have a different step_size.
If `mat.isContinuous()` is true, step_size is zero for every row and the array we need is simply `mat.data` [8]
```
std::vector<uchar> array(mat.rows*mat.cols);
if (mat.isContinuous())
    array = mat.data;
```
Otherwise, we need to copy each pixel as by `mat.at(i,j)` [2D] or `mat.at(i,j,k)` [3D]. For now, let's not conside higher dimensions. Easier solution is to make every mat continuous and then get `mat.data`. Any Mat constructed with Mat constructor is continuous. Mat constructed from anything other than OpenCV by referencing memory may not be continuous. Eg. bmp images.
Hence, copying pixels cannot be avoided if we want to avoid calculating addresses of pixels on JS side.
```
if ( ! mat.isContinuous() ) { 
    mat = mat.clone(); // O(rows*cols*channels)
}
uint8_t * data = mat.data; // O(1)
``` 
Then this 1-D array can be converted to `Napi::ArrayBuffer` or `Napi::Uint8Array` and passed to nodejs side.  
First construct ArrayBuffer from pointer to c++ array using this method
```
static ArrayBuffer Napi::ArrayBuffer::New(
    napi_env env,
    void * 	externalData,
    size_t 	byteLength 
)	
```
For Uint8Array, byteLength is 1 (8 bit = 1 byte)
Then create `Napi::Uint8Array` or any other `Napi::TypedArrayOf<type>` with his method
```
static TypedArrayOf Napi::TypedArrayOf<T>::New(
    napi_env env,
    size_t elementLength,
    Napi::ArrayBuffer arrayBuffer,
    size_t bufferOffset,
    napi_typedarray_type type = TypedArray::TypedArrayTypeForPrimitiveType<T>() 
)	
```
byteOffset can be set to zero.
For initializing Uint8ClampedArray, type must be explicitly specified
```
Uint8Array::New(env, length, buffer, 0, napi_uint8_clamped_array) 
```
length = buffer.Data/buffer.ByteLength()

### Implementation result
I made a helper class NdArray to manage passing images. This class for responsible for all the above conversions using just built-in JavaScript types. When tested on cv::Mat, it did not work :(. When I try to pass the Napi::Object holding Mat's data as Uint8Array to nodejs side, I encountered suspicious error 'Unknown Failure' is all I got from the error message. This wasted a lot of my time. Almost 4 days. I decided to drop the idea of NdArray and moved on to wrapping cv::Mat.

### Bindings for cv::Mat
As I mentioned earlier, wrapping cv::Mat is really a task. It's a huge class and has a lot of dependent classes. But I just need to pass on the Mat data. I really don't need all the static methods and instance methods that Mat provides. Just a light-weight wrapper that will allow me passing an image to/from Nodejs and C++.

### New approach comes with new problems
Binding any c++ datatype seems straight-forward.... at first. 

To bind cv::Mat 
1. Create a new class say `Nodoface::Image` extending `Napi::ObjectWrap<Nodoface::Image>`. 
2. Create a static `Napi::FunctionReference` variable that will be responsible for instantiating and deleting the wrapper class and synchronising it with Nodejs.
3. Maintain a private instance of cv::Mat that the wrapper respresents. 
4. A constructor for instantiating it from Nodejs
     
I really do not need to bind any methods of cv::Mat but for sake of testing, I wrote methods to retrieve Mat's dimensions and its CV_XCN value known as 'type'.    
**Creating the constructor**    
I need two constructors. One for that accepts a cv::Mat from within cpp and one that accepts parameters like Uint8Array from Nodejs.
Now to use this wrapper, I will get cv::Mat returned from ImageCapture.GetNextFrame(), create an instance of `Nodoface::Image` and return it from our ImageCapture.GetNextFrame() wrapper function. Implementation completed but got 'Unknown failure' whenever I return my wrapper instance to nodejs.   
**Why?**   
Because ObjectWrap<T> IS NOT a subclass of Napi::Value! I should get Napi::Object (a subclass of Napi::Value) from Nodoface::Image. But there is no method within ObjectWrap<T> that converts it to Object! What's the use of ObjectWrap then? Turns out that it is only used to define the wrapper object and not for creating its instances directly.     
What about the static FunctionReference 'constructor'?      
I have a static constructor variable but I never every used it. Digging into the node-addon-examples, I found it has a method 'New' which returns Napi::Object!    

Putting it all together:
1. NewInstance(cv::Mat) converts cv::Mat to Uint8Array (data), Int32Array (dimensions) and Number (type).
2. These three arguments are passed to constructor.New() which accepts initalizer_list of `Napi::Value`s.
3. constructor.New() call is equivalent to calling the wrapper class's constructor from Nodejs. It creates an instance on Node side and hence a corresponding instance is created on c++ side.
4. Inside the class constructor, I have to use Uint8Array data and Mat dimensions to reconstruct the cv::Mat. (redundant right?).
5. Finally, constructor.New() returns Napi::Object of the wrapper class. This can be passed to Nodejs from any C++ function.

Everything implemented over a night. No errors this time!
But my happiness was about vanish. Recived image is not the original image retured from ImageCapture. It is just a Mat with garbage values. Moreover, I got segfault many a times. I am already behind schedule, I dropped previous NdArray code (which was 4 days of work!) and now the new method is still not working. I talked to my mentor Carlos, got the energy to continue working on this and raised this issue to the developers of node-addon-api [[11]](https://github.com/nodejs/node-addon-api/issues/500) on his advice. 

The developers behind NAPI are genius minds and extremely busy folks. They really don't have the time to answer an issue with a detailed solution. We must pay attention to every word they say while answering our issue. One of the devs pointed out the problem as being related to memory allocation of data pointer. He said I should `ensure the Mat data is not on stack`. This was the key! But it took me to relate it to my code. 
### Always look back to fundamentals!
Being a Java and Python programmer for over 2 years, I assumed any class instance I created was on Heap. But in C++, you have to explicitly use `new` operator to create objects on heap instead of stack. Any stack allocated data exists in memory as long as it is in scope. Whereas, objects on Heap remains in memory until we `delete[]` them explicitly. So after wasting a lot of time to find a solution in node-addon-api (tried things like EscapableHandleScope, Napi::Reference), I finally found the solution in C++ fundamentals. I just needed to add stars to my code :P    
Changing `cv::Mat mat(params..)` to `cv::Mat* mat = new cv::Mat(params..)` fixed the data loss problem.

## Wrap up week 2
Something I proposed to finish in one week took me over 2 weeks. Evaluation phase is approaching and I must finish the remaining work before it. But I am very confident with node-addon-api now. The issues I faced this week forced me to goto to source code level of the node-addon-api to undertand how things actually work in it. I would attribute the time loss to lack of documentation. But thats the case with any evolving technology. Nevertheless, now I can blindly write bindings to any C++ class regardless of its complexity. 

## Week 3 todos:
1. Bindings for OpenFace's Face detection models.
2. TypeScript API for all the work done so far.
I have 4 days left to complete 2 weeks work. Hope I can complete it.


## References
[1] [Linking to static libraries node-gyp issue 328](https://github.com/nodejs/node-gyp/issues/328)     
[2] [Setting -fPIC to cmake build](https://stackoverflow.com/questions/38296756/what-is-the-idiomatic-way-in-cmake-to-add-the-fpic-compiler-option)    
[3] [dlopen error with statically compiled opencv](https://github.com/justadudewhohacks/opencv4nodejs/issues/113)    
[4] [OpenCV cannot find installed openblas headers](https://github.com/opencv/opencv/issues/9953)    
[5] [Compile OpenFace as shared library](https://github.com/TadasBaltrusaitis/OpenFace/issues/23)    
[6] [JS ArrayBuffer to C++ array](https://github.com/nodejs/node-addon-examples/blob/master/array_buffer_to_native/node-addon-api/array_buffer_to_native.cc)    
[7] [cv::Mat documentation](https://docs.opencv.org/3.4.6/d3/d63/classcv_1_1Mat.html#details)       
[8] [cv::Mat to array/vector](https://stackoverflow.com/a/26685567/6699069)     
[9] [Napi::ArrayBuffer from c++ array](https://nodejs.github.io/node-addon-api/class_napi_1_1_array_buffer.html#ad773d4b1e1c664a94fa9e47954a9012f)        
[10] [Napi::TypedArrayOf<T> from Napi::ArrayBuffer](https://nodejs.github.io/node-addon-api/class_napi_1_1_typed_array_of.html#af27064a817bd47ae1133c455d6e1e267)       
[11] [Send C++ class to nodejs without data loss](https://github.com/nodejs/node-addon-api/issues/500)