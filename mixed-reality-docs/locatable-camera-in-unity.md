---
title: Locatable camera in Unity
description: 
author: 
ms.author: wguyman
ms.date: 2/28/2018
ms.topic: article
keywords: 
---



# Locatable camera in Unity

## Enabling the capability for Photo Video Camera

The "WebCam" capability must be declared for an app to use the [camera](locatable-camera.md).
1. In the Unity Editor, go to the player settings by navigating to "Edit > Project Settings > Player" page
2. Click on the "Windows Store" tab
3. In the "Publishing Settings > Capabilities" section, check the **WebCam** and **Microphone** capabilities

Only a single operation can occur with the camera at a time. To determine which mode (photo, video, or none) the camera is currently in, you can check UnityEngine.VR.WSA.WebCam.Mode.

## Photo Capture

**Namespace:** *UnityEngine.VR.WSA.WebCam*

**Type:** *PhotoCapture*

The PhotoCapture type allows you to take still photographs with the Photo Video Camera. The general pattern for using PhotoCapture to take a photo is as follows:
1. Create a PhotoCapture object
2. Create a CameraParameters object with the settings we want
3. Start Photo Mode via StartPhotoModeAsync
4. Take the desired photo
* (optional) Interact with that picture
5. Stop Photo Mode and clean up resources

### Common Set Up for PhotoCapture

For all three uses, we start with the same first 3 steps above

We start by creating a PhotoCapture object

```
PhotoCapture photoCaptureObject = null;
   void Start()
   {
       PhotoCapture.CreateAsync(false, OnPhotoCaptureCreated);
   }
```

Next we store our object, set our parameters, and start Photo Mode

```
void OnPhotoCaptureCreated(PhotoCapture captureObject)
   {
       photoCaptureObject = captureObject;

       Resolution cameraResolution = PhotoCapture.SupportedResolutions.OrderByDescending((res) => res.width * res.height).First();

       CameraParameters c = new CameraParameters();
       c.hologramOpacity = 0.0f;
       c.cameraResolutionWidth = cameraResolution.width;
       c.cameraResolutionHeight = cameraResolution.height;
       c.pixelFormat = CapturePixelFormat.BGRA32;

       captureObject.StartPhotoModeAsync(c, false, OnPhotoModeStarted);
   }
```

In the end, we will also use the same clean up code presented here

```
void OnStoppedPhotoMode(PhotoCapture.PhotoCaptureResult result)
   {
       photoCaptureObject.Dispose();
       photoCaptureObject = null;
   }
```

After these steps, you can choose which type of photo to capture.

### Capture a Photo to a File

The simplest operation is to capture a photo directly to a file. The photo can be saved as a JPG or a PNG.

If we successfully started photo mode, we now will take a photo and store it on disk

```
private void OnPhotoModeStarted(PhotoCapture.PhotoCaptureResult result)
   {
       if (result.success)
       {
           string filename = string.Format(@"CapturedImage{0}_n.jpg", Time.time);
           string filePath = System.IO.Path.Combine(Application.persistentDataPath, filename);

           photoCaptureObject.TakePhotoAsync(filePath, PhotoCaptureFileOutputFormat.JPG, OnCapturedPhotoToDisk);
       }
       else
       {
           Debug.LogError("Unable to start photo mode!");
       }
   }
```

After capturing the photo to disk, we will exit photo mode and then clean up our objects

```
void OnCapturedPhotoToDisk(PhotoCapture.PhotoCaptureResult result)
   {
       if (result.success)
       {
           Debug.Log("Saved Photo to disk!");
           photoCaptureObject.StopPhotoModeAsync(OnStoppedPhotoMode);
       }
       else
       {
           Debug.Log("Failed to save Photo to disk");
       }
   }
```

### Capture a Photo to a Texture2D

When capturing data to a Texture2D, the process is extremely similar to capturing to disk.

We will follow the set up process above.

In OnPhotoModeStarted, we will capture a frame to memory.

```
private void OnPhotoModeStarted(PhotoCapture.PhotoCaptureResult result)
   {
       if (result.success)
       {
           photoCaptureObject.TakePhotoAsync(OnCapturedPhotoToMemory);
       }
       else
       {
           Debug.LogError("Unable to start photo mode!");
       }
   }
```

We will then apply our result to a texture and use the common clean up code above.

```
void OnCapturedPhotoToMemory(PhotoCapture.PhotoCaptureResult result, PhotoCaptureFrame photoCaptureFrame)
   {
       if (result.success)
       {
           // Create our Texture2D for use and set the correct resolution
           Resolution cameraResolution = PhotoCapture.SupportedResolutions.OrderByDescending((res) => res.width * res.height).First();
           Texture2D targetTexture = new Texture2D(cameraResolution.width, cameraResolution.height);
           // Copy the raw image data into our target texture
           photoCaptureFrame.UploadImageDataToTexture(targetTexture);
           // Do as we wish with the texture such as apply it to a material, etc.
       }
       // Clean up
       photoCaptureObject.StopPhotoModeAsync(OnStoppedPhotoMode);
   }
```

### Capture a Photo and Interact with the Raw bytes

To interact with the raw bytes of an in memory frame, we will follow the same set up steps as above and OnPhotoModeStarted as in capturing a photo to a Texture2D. The difference is in OnCapturedPhotoToMemory where we can get the raw bytes and interact with them.

In this example, we will create a List<Color> which could be further processed or applied to a texture via SetPixels()

```
void OnCapturedPhotoToMemory(PhotoCapture.PhotoCaptureResult result, PhotoCaptureFrame photoCaptureFrame)
   {
       if (result.success)
       {
           List<byte> imageBufferList = new List<byte>();
           // Copy the raw IMFMediaBuffer data into our empty byte list.
           photoCaptureFrame.CopyRawImageDataIntoBuffer(imageBufferList);

           // In this example, we captured the image using the BGRA32 format.
           // So our stride will be 4 since we have a byte for each rgba channel.
           // The raw image data will also be flipped so we access our pixel data
           // in the reverse order.
           int stride = 4;
           float denominator = 1.0f / 255.0f;
           List<Color> colorArray = new List<Color>();
           for (int i = imageBufferList.Count - 1; i >= 0; i -= stride)
           {
               float a = (int)(imageBufferList[i - 0]) * denominator;
               float r = (int)(imageBufferList[i - 1]) * denominator;
               float g = (int)(imageBufferList[i - 2]) * denominator;
               float b = (int)(imageBufferList[i - 3]) * denominator;

               colorArray.Add(new Color(r, g, b, a));
           }
           // Now we could do something with the array such as texture.SetPixels() or run image processing on the list
       }
       photoCaptureObject.StopPhotoModeAsync(OnStoppedPhotoMode);
   }
```

## Video Capture

**Namespace:** *UnityEngine.VR.WSA.WebCam*

**Type:** *VideoCapture*

VideoCapture functions very similarly to PhotoCapture. The only two differences are that you must specify a Frames Per Second (FPS) value and you can only save directly to disk as an .mp4 file. The steps to use VideoCapture are as follows:
1. Create a VideoCapture object
2. Create a CameraParameters object with the settings we want
3. Start Video Mode via StartVideoModeAsync
4. Start recording video
5. Stop recording video
6. Stop Video Mode and clean up resources

We start by creating our VideoCapture object VideoCapture m_VideoCapture = null;

```
void Start ()
   {
       VideoCapture.CreateAsync(false, OnVideoCaptureCreated);
   }
```

We then will set up the parameters we will want for the recording and start.

```
void OnVideoCaptureCreated (VideoCapture videoCapture)
   {
       if (videoCapture != null)
       {
           m_VideoCapture = videoCapture;

           Resolution cameraResolution = VideoCapture.SupportedResolutions.OrderByDescending((res) => res.width * res.height).First();
           float cameraFramerate = VideoCapture.GetSupportedFrameRatesForResolution(cameraResolution).OrderByDescending((fps) => fps).First();

           CameraParameters cameraParameters = new CameraParameters();
           cameraParameters.hologramOpacity = 0.0f;
           cameraParameters.frameRate = cameraFramerate;
           cameraParameters.cameraResolutionWidth = cameraResolution.width;
           cameraParameters.cameraResolutionHeight = cameraResolution.height;
           cameraParameters.pixelFormat = CapturePixelFormat.BGRA32;

           m_VideoCapture.StartVideoModeAsync(cameraParameters,
                                               VideoCapture.AudioState.None,
                                               OnStartedVideoCaptureMode);
       }
       else
       {
           Debug.LogError("Failed to create VideoCapture Instance!");
       }
   }
```

Once started, we will begin the recording

```
void OnStartedVideoCaptureMode(VideoCapture.VideoCaptureResult result)
   {
       if (result.success)
       {
           string filename = string.Format("MyVideo_{0}.mp4", Time.time);
           string filepath = System.IO.Path.Combine(Application.persistentDataPath, filename);

           m_VideoCapture.StartRecordingAsync(filepath, OnStartedRecordingVideo);
       }
   }
```

After recording has started, you could update your UI or behaviors to enable stopping. Here we just log

```
void OnStartedRecordingVideo(VideoCapture.VideoCaptureResult result)
   {
       Debug.Log("Started Recording Video!");
       // We will stop the video from recording via other input such as a timer or a tap, etc.
   }
```

At a later point, we will want to stop the recording. This could happen from a timer or user input, for instance.

```
// The user has indicated to stop recording
   void StopRecordingVideo()
   {
       m_VideoCapture.StopRecordingAsync(OnStoppedRecordingVideo);
   }
```

Once the recording has stopped, we will stop video mode and clean up our resources.

```
void OnStoppedRecordingVideo(VideoCapture.VideoCaptureResult result)
   {
       Debug.Log("Stopped Recording Video!");
       m_VideoCapture.StopVideoModeAsync(OnStoppedVideoCaptureMode);
   }

   void OnStoppedVideoCaptureMode(VideoCapture.VideoCaptureResult result)
   {
       m_VideoCapture.Dispose();
       m_VideoCapture = null;
   }
```

## Troubleshooting
* No resolutions are available
* Ensure the **WebCam** capability is specified in your project.

## See Also
* [Locatable camera](locatable-camera.md)