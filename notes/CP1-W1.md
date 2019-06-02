# Coding Period 1 - Week 1

## Development machine
CPU: i5520U     
RAM: 8 GB DDR3-L      
OS: Ubuntu 18.04 
GCC: 7.3
Nodejs: 10.15.3   
IDE: CLion, VS Code

## Environment setup for Nodoface

Nodoface depends on OpenFace which depends on OpenCV and dlib. I have few existing python projects that use OpenCV and Dlib installed in /usr/local. Then there are CUDA enabled versions existing for some other projects. I am probably not alone with such configuration. To avoid messing up of library versions and to ease the dependency installation of Nodoface, a package manager for C++ is desirable. 

### Two options
#### 1. Microsoft's [vcpkg](https://github.com/microsoft/vcpkg)
Installation:
1. Clone and cd to vcpkg root
2. `./bootstrap-vcpkg.sh`
3. `./vcpkg integrate install`
4. `./vcpkg integrate bash`

I expected step 3 will make vcpkg command available from terminal but it didn't.
So manually **added vcpkg to system path**

##### Installing OpenCV and Dlib:

1. `vcpkg install opencv` # installs v3.4.3
2. `vcpkg install dlib` # installs v19.17

Headers and libs will goto `$vcpkgRoot/installed/x64-linux`
Add the `include` path to c_cpp_configuration.json in vscode for IntelliSense to work.

##### Compiling OpenFace:
1. `git clone --depth 1 https://github.com/TadasBaltrusaitis/OpenFace.git` # depth=1 because repo is too large
2. Create and goto build directory
3. `cmake -D CMAKE_BUILD_TYPE=RELEASE CMAKE_CXX_FLAGS="-std=c++11" -D CMAKE_EXE_LINKER_FLAGS="-std=c++11" -DCMAKE_TOOLCHAIN_FILE=$vcpkgRoot/scripts/buildsystems/vcpkg.cmake ..`

`DCMAKE_TOOLCHAIN_FILE` is additional flag because I am using vcpkg

Alternatively, create portfile.cmake for OpenFace as explained [here](https://github.com/sziraqui/vcpkg/blob/master/docs/examples/packaging-github-repos.md).
Either submit PR to vcpkg repo or add the port to $vcpkgRoot/ports locally. Then simply call vcpkg install.

#### 2. [Conan](https://conan.io)
// Todo (Not using it, may try later)

### Results after setup
Removed existing opencv and dlib from /usr/local
Manual compilation from source using instructions from install.sh completed successfully. 

Executing sample binary     
`./bin/FaceLandmarkVid -f "../samples/2015-10-15-15-14.avi"`        
gives error:    
```
...
Starting tracking
OpenCV Error: Assertion failed (s >= 0) in setSize, file /home/sziraqui/Projects/openface-standalone/opencv-3.4.0/modules/core/src/matrix.cpp, line 310
OpenCV Error: Insufficient memory (Failed to allocate 2568164540132953040 bytes) in OutOfMemoryError, file /home/sziraqui/Projects/openface-standalone/opencv-3.4.0/modules/core/src/alloc.cpp, line 55
OpenCV Error: Assertion failed (s >= 0) in setSize, file /home/sziraqui/Projects/openface-standalone/opencv-3.4.0/modules/core/src/matrix.cpp, line 310
OpenCV Error: Assertion failed (u != 0) in create, file /home/sziraqui/Projects/openface-standalone/opencv-3.4.0/modules/core/src/matrix.cpp, line 436
terminate called after throwing an instance of 'cv::Exception'
  what():  /home/sziraqui/Projects/openface-standalone/opencv-3.4.0/modules/core/src/matrix.cpp:310: error: (-215) s >= 0 in function setSize

Aborted (core dumped)
```
Opencv's imread() is working fine. But OpenFace's image reader is throwiing error. There are just 30 image files to handle so OutOfMemoryError is suspicious.

Executing FaceLandmarkVid binary   
`./bin/FaceLandmarkVid -f "../samples/2015-10-15-15-14.avi"`    
gives error
```
...
OpenCV Error: Assertion failed (!fixedType() || ((Mat*)obj)->type() == mtype) in create, file /home/sziraqui/Projects/openface-standalone/opencv-3.4.0/modules/core/src/matrix.cpp, line 2386
terminate called after throwing an instance of 'cv::Exception'
  what():  /home/sziraqui/Projects/openface-standalone/opencv-3.4.0/modules/core/src/matrix.cpp:2386: error: (-215) !fixedType() || ((Mat*)obj)->type() == mtype in function create

Aborted (core dumped)
```
Installing opencv and dlib from vcpkg and then compiling OpenFace has no effect. 

#### Solution
Model files (*.dat) were probably corrupt.  
Redownloading and copying models to build directory fixed the problem. 
```
cd <openface_root_dir>
rm lib/local/LandmarkDetector/model/patch_experts/*.dat
./download_models.sh
cp -r lib/local/LandmarkDetector/model/patch_experts/ build/bin/model
```
Once everythig is working fine, I did
`sudo make install` from OpenFace build directory to install the library in `/usr/local`

## NAPI - pre-requisites for Nodoface

### [node-gyp](https://github.com/nodejs/node-gyp)
A build system made for Node Addons. It's an alternative to cmake

### bindings.gyp
The CMakeLists.txt for node-gyp. A python-like dictionary with support for variables and commands.
[Example projects](https://github.com/nodejs/node-gyp/wiki/%22binding.gyp%22-files-out-in-the-wild) using bindings.gyp
#### bindings.gyp of [nodecv](https://github.com/macacajs/nodecv/blob/master/binding.gyp) project
```
{
  "variables": {
    "module_name": "nodecv",
    "module_path": "./build",
    "opencv_version": "2.4.13.2"
  },
  "targets": [{
    "target_name": "<(module_name)",
      "sources": [
        "src/init.cc",
        "src/core/Mat.cc",
        "src/highgui/highgui.cc",
        "src/objdetect/CascadeClassifier.cc",
        "src/features2d/features2d.cc",
        "src/imgproc/imgproc.cc"
      ],
      "libraries": [
        "<!@(pkg-config \"opencv >= <(opencv_version)\" --libs)"
      ],
      "include_dirs": [
        "<!@(pkg-config \"opencv >= <(opencv_version)\" --cflags)",
        "<!(node -e \"require('nan')\")"
      ],
      "cflags!" : [
        "-fno-exceptions"
      ],
      "cflags_cc!": [
        "-fno-rtti",
        "-fno-exceptions"
      ],
      "conditions": [
        [
          "OS==\"linux\" or OS==\"freebsd\" or OS==\"openbsd\" or OS==\"solaris\" or OS==\"aix\"",
          {
            "cflags": [
              "<!@(pkg-config \"opencv >= <(opencv_version)\" --cflags)",
              "-Wall"
            ]
          }
        ],
      ],
    }, 
  ]
}
```

### GYP files
GYP files are python-like dictionaries that encompass instructions to a build system to compile a c/c++ project. It is a tool written in python 2.
The confusing parts of .gyp file are those using !,<,>,@, and any combination of these symbols on keys or values.    
My understanding so far:    
**<** : Expands a variable in early phase of GYP. Eg. `<(module_name)` will be replaced with value of `module_name` key in `variables` dict.    
**>** : Similar to **<** but expansion happens in Late phase of GYP. In general, < should be prefered.    
**!** : As suffix to a key, `key!` will exclude the values associated with it. Eg. `cflags! : ['-fnoexceptions']` implies -fnoexceptions flag is not to be used. As prefix to a value `!(value)` will execute `value` as a shell command.   
**@** : Expands a whitespace separated string into a list. Eg. `<!@(pkg-config --cflags opencv)` will execute the shell command and each line of output will become an item of the enclosing list.    
More symbols exists viz. `=`, `+`, `?`, `$` but they are not used often in a node-gyp project.    

Explaination of nodecv's binding.gyp file based on [Input Format specification](https://gyp.gsrc.io/docs/InputFormatReference.md) and [node-gyp documentation](https://github.com/nodejs/node-gyp):     
**variables**: Keys used to reference variable values in other parts of the .gyp file.   
**targets**: Defines the build (for generating Makefile). Each item in the list will result in creation of .node file in a node-gyp project.     
**target_name**: Name of compiled .node file without extension.    
**sources** : List of source files (.cpp/.cc) to be compiled.     
**cflags** : Compiler flags (-Wsomething).    
**include_dirs** : List of include paths to be passed to compiler with -I flag.     
**libraries** : List of libs path to be passed to compiler with -l flag.     
**conditions** : List of sub-dicts to be merged depending on conditions in first item of each list. Second item is the actual dict to be merged with parent.     


### NAPI tutorials and examples
- [x] [Beginners guide to writing Nodejs C++ Addons](https://medium.com/@atulanand94/beginners-guide-to-writing-nodejs-addons-using-c-and-n-api-node-addon-api-9b3b718a9a7f)
- [ ] [NAPI Asynchronous worker](http://www.adaltas.com/en/2018/12/12/native-modules-node-js-n-api/)

## Nodoface project setup
### Initial setup on CLion
No IDE yet supports node-gyp.
CLion generates an index of classes, methods, etc using headers. Header location is retrieved from CMakeLists.txt. In absence of cmake lists, CLion will not recognise a project as a C++ project and autocomplete will not work. As a workaround, a dummy CMakeLists.txt will be added to Nodoface while I am working on CLion in the first week.    
Example of dummy CMakeLists.txt:
```
# dummy cmakelists to make intellisense work in CLion
cmake_minimum_required(VERSION 3.12)
project(dummy)
set(CMAKE_CXX_FLAGS "${CMAKE_CXX_FLAGS} -std=c++11")
set(SOURCE_FILES src/main.cpp src/funcs.cpp)
include_directories(node_modules/node-addon-api node_modules/node-addon-api/src)

add_executable(dummy ${SOURCE_FILES})
```
The project will be like any npm project with a package.json file. A binding.gyp file will rest alongside package.json. A src directory will have C++ code and structure will be like any typical C++ project. An api directory will hold the .js files that will handle importing .node files and exporting them into a familiar nodejs module. It will also hold typings to support TypeScript.
```
Nodoface/
- package.json
- binding.gyp
- CMakeLists.txt (dummy)
- src/
  - bindings code (C++ project)
- api/
  - .js files
  - typings
- examples/
  - .ts files (using the API manually testing it)
```
Later on, I will write a plugin to support node-gyp.
## Things to do in week 2
Next, I will start working on Nodoface by writing bindings for following classes:
(All paths are relative to OpenFace's root directory)
1. ImageCapture:    
  Implemented in lib/local/Utilities/src/ImageCapture.cpp
  It supports reading image frames from image directory, video file or webcam. However, it is used only for image directory.
2. SequenceCapture:
   Defined in lib/local/Utilities/src/SequenceCapture.cpp
   Similar to ImageCapture but used only for webcam and video files.
