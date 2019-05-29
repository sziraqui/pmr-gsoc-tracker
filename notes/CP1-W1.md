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
// Todo

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

//todo: Fix above errors

## NAPI - pre-requisites for Nodoface

### [node-gyp](https://github.com/nodejs/node-gyp)
A build system made for Node Addons. It's an alternative to cmake

### bindings.gyp
The CMakeLists.txt for node-gyp. A python-like dictionary with support for variables and commands.
[Example projects](https://github.com/nodejs/node-gyp/wiki/%22binding.gyp%22-files-out-in-the-wild) using bindings.gyp

### GYP files
// In progress: Reading [Input Format specification](https://gyp.gsrc.io/docs/InputFormatReference.md)

### NAPI tutorials and examples
- [x] [Beginners guide to writing Nodejs C++ Addons](https://medium.com/@atulanand94/beginners-guide-to-writing-nodejs-addons-using-c-and-n-api-node-addon-api-9b3b718a9a7f)
- [ ] [NAPI Asynchronous worker](http://www.adaltas.com/en/2018/12/12/native-modules-node-js-n-api/)

## Nodoface project setup
//todo