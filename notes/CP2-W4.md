# Coding Period 2 - Week 4
After I got a recognition model working, I proceeded to implementing a face tracker using MOT method

## High level tracking algo
For first frame,     
Step1: Detect all faces and find their embeddings (Detection+Recog ~ 200ms) -> ObjectBlob           
Step2: Run a face matcher to assign names to by computing their euclidean distance -> ObjectBlob with embedding          
For next frames,        
Step3: Detect all faces in current frame -> faceBlobs       
Pass 1          
Step4: For each faceBlob in faceBlobs, find highest IOU ObjectBlob from past frame -> Assignment vector     
Step5: Update ObjectBlobs with props of assigned faceBlobs.     
Pass 2      
Step5: For each unassigned faceBlob, find closest match by embedding distance -> Assignment vector      
Step6: Step5: Update ObjectBlobs with props of assigned faceBlobs.      
Pass 3      
Step6: For each remaining faceBlob, create a new ObjecBlob -> ObjectBlobs       
Step7: Delete ObjectBlobs older than blobDelInterval time of frames     
Step8: Run face matcher to assign names to ObjectBlobs -> ObjectBlobs with labels       

## Issues
Implmentation is done but `VideoCapture` binding is SEGFAULTing in the same way like `SequenceCapture` does. This happens only when I try to `showImage`.
Because of this, VideoCapture passed all test cases but segfaults when local objects are passed to 
Image pixels are not retained while passing image to showImage method. This is unexpected because image is allocated on Heap memory so there should not be any loss of data going out of scope.
