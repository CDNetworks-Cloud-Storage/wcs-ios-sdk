## Prerequisites
- Object Storage is activated.
- The AccessKey and SecretKey are created
- iOS V7.0 or above

## Project Introduction
- [Download SDK ](https://wcsd.chinanetcenter.com/sdk/cnc-ios-sdk-wcs.zip)
- [Demo & Examples](https://github.com/CDNetworks-Object-Storage/wcs-ios-sdk/tree/master/tools/TestWCSiOS)

## Install
### Preparations for mobile end

1. Set the SDK version as iOS7 or above.
2. Add **WCSiOS.framework** to the project environment, and make sure that **WCSiOS.framework** has been addded to *Build Phases -> Link Binary With Libraries*
3. SDK has a dependency on below system libs, please add below libs to **Link Binary With Libraries**
```
MobileCoreServices.framework
libz.dylib(libz.tbd for Xcode7+)
```
4. **Category** is inclueded in SDK framework, so you have to choose `-ObjC` option, or the selector may cannot be recognized when using SDK.
e.g.
-[__NSCFDictionary safeStringForKey:]: unrecognized selector sent to instance 0x7f8c51d3c260 
![image](https://user-images.githubusercontent.com/98135632/151472425-cb0d9b5f-ad78-4d43-af31-89786abc24db.png)

### Preparations for server end
For server end, please refer to [wcs-Java-SDK](https://github.com/CDNetworks-Object-Storage/wcs-java-sdk)

## Initialization
- AK/SK is required to authenticate the user's signature when accessing Object Storage. With regard to security of AK and SK, it is recommended to build your own server to provide authentication credentials.
- Configure upload domain & time out
```
self.client = [[WCSClient alloc] initWithBaseURL:[NSURL URLWithString:@"http://yourUploadDomain.com"] andTimeout:30];

```
## How to use
### Normal Upload
- It is recommeded to set returnurl only when using sheet upload.
- Multipart upload are recommended when file to upload is larger than 20MB.
- Object Storage provides a normal domain for you. If you are sensitive to the upload speed, we suggest you to use our CDN for the upload acceleration.

#### Example of normal upload
```
- (void)normalUpload {
  WCSUploadObjectRequest *request = [[WCSUploadObjectRequest alloc] init];
  request.token = @"token of uploading, provided by server end";
  request.fileName = @"File name";
  request.key = @"the file name in Object storage, if remain it empty, it will follow as fileName";
  request.fileData = fileData; // The file to be uploaded 
  request.uploadProgress = ^(int64_t bytesSent, int64_t totalBytesSent, int64_t totalBytesExpectedToSend) {
    NSLog(@"%lld bytes sent, %lld total bytes sent, %lld total byte exptected", bytesSent, totalBytesSent, totalBytesExpectedToSend);
  };
  
  WCSClient *client = [[WCSClient alloc] initWithBaseURL:nil andTimeout:30];
  
  // Notes: If you are using callback upload, you need to use uploadRequestRaw, which will avoid unnecessary abnormity in base64 resolve
  [[client uploadRequest:request] continueWithBlock:^id _Nullable(WCSTask<WCSUploadObjectResult *> * _Nonnull task) {
    if (task.error) {
      NSLog(@"The request failed. error: [%@]", task.error);
} else {
  // Request successfully, the content returned by server are as belows
      NSDictionary *responseResult = task.result.results;
    }
    return nil;
  }];
}

```
#### Cancel the request of upload

Example
```
- (void)normalUploadCancelled {
  WCSUploadObjectRequest *request = [[WCSUploadObjectRequest alloc] init];
  request.token = @"Token for upload, provided by server end";
  request.fileName = @"File name";
  request.key = @"the file name in object storage, if remain it empty, it will follow as fileName";
  request.fileData = fileData; // The file need to be uploaded 
  request.uploadProgress = ^(int64_t bytesSent, int64_t totalBytesSent, int64_t totalBytesExpectedToSend) {
    NSLog(@"%lld bytes sent, %lld total bytes sent, %lld total byte exptected", bytesSent, totalBytesSent, totalBytesExpectedToSend);
  };

  WCSClient *client = [[WCSClient alloc] initWithBaseURL:nil andTimeout:30];
  [[client uploadRequest:request] continueWithBlock:^id _Nullable(WCSTask<WCSUploadObjectResult *> * _Nonnull task) {
    if (task.error) {
      // Request is cancelled 
      if (task.error.code == NSURLErrorCancelled) {
        NSLog(@"request cancelled.");
      } else {
        NSLog(@"%@", task.error);
      }
    } else {
      NSLog(@"%@", task.result.results);
}
return nil;
  }];
  dispatch_after(dispatch_time(DISPATCH_TIME_NOW, (int64_t)(0.5 * NSEC_PER_SEC)),   dispatch_get_main_queue(), ^{
    [uploadRequest cancel];
  });
}

```

#### Upload by customized variables (POST)

Examples
```
- (void)normalUpload {
  WCSUploadObjectRequest *request = [[WCSUploadObjectRequest alloc] init];
  request.token = @"token of uploading, provided by server end";
  // Customized variables, which will be returned to client end after successful upload
  request.customParams = @{@"x:test" : @"customParams"};
  request.fileName = @"File name";
  request.key = @"the file name in object storage, if remain it empty, it will follow as fileName";
  request.fileData = fileData; // The file need to be uploaded 
  request.uploadProgress = ^(int64_t bytesSent, int64_t totalBytesSent, int64_t totalBytesExpectedToSend) {
    NSLog(@"%lld bytes sent, %lld total bytes sent, %lld total byte exptected", bytesSent, totalBytesSent, totalBytesExpectedToSend);
  };
  // it is recommended to use WCSClient
  WCSClient *client = [[WCSClient alloc] initWithBaseURL:nil andTimeout:30];
  
  // Notes: If you are using callback upload, you need to use uploadRequestRaw, which will avoid unnecessary abnormity in base64 resolve
  [[client uploadRequest:request] continueWithBlock:^id _Nullable(WCSTask<WCSUploadObjectResult *> * _Nonnull task) {
    if (task.error) {
      NSLog(@"The request failed. error: [%@]", task.error);
} else {
  // Request successfully, the content returned by server are as belows
      NSDictionary *responseResult = task.result.results;
    }
    return nil;
  }];
}

```


### Multipart Upload 

- Normally, it takes a long time to upload large files on the mobile end. And all files need to be retransmitted f an error occurs. To avoid this problem, multipart upload mechanism is introduced.
- Multipart upload slices a large file into many small-size blocks, which will be uploaded in parallel. Once a block upload fails, the client just needs to re-upload the upload-fail block. 
Note: Block size should not exceed 100MB and be less than 4MB.

Example
```
- (void)chunkedUpload {
  WCSBlockUploadRequest *blockRequest = [[WCSBlockUploadRequest alloc] init];
  blockRequest.fileKey = @"the file name in object storage, if remain it empty, it will follow as fileName";
  blockRequest.fileURL = fileURL; // URL of this file
  blockRequest.token = @"token of uploading, provided by server end";
  blockRequest.chunkSize = 256 * 1024; // Note: the chunk size must be multiple of 64K, and the max size can't exceed the block size
  blockRequest.blockSize = 4 * 1024 * 1024; // Note: the bock size must be multiple of 4M, and the max size can't exceed 100M
  [blockRequest setUploadProgress:^(int64_t bytesSent, int64_t totalBytesSent, int64_t totalBytesExpectedToSend) {
    NSLog(@"%@ %@", @(totalBytesSent), @(totalBytesExpectedToSend));
  }];
  // it is recommended to use WCSClient
  WCSClient *client = [[WCSClient alloc] initWithBaseURL:nil andTimeout:30];
  
  // Notes: If you are using callback upload, you need to use blockUploadRequestRaw, which will avoid unnecessary abnormity in base64 resolve
  [[client blockUploadRequest:blockRequest] continueWithBlock:^id _Nullable(WCSTask<WCSBlockUploadResult *> * _Nonnull task) {
    if (task.error) {
      NSLog(@"error %@", task.error.localizedDescription);
} else {
  // Request successfully, the content returned by server are as belows
      NSLog(@"results %@", task.result.results);
    }
    return nil;
  }];

```

## Commond Issues
1. Unrecognized method, e.g. `-[__NSCFDictionary safeStringForKey:]: unrecognized selector sent to instance 0x7f8c51d3c260`. 
- Please make sure you have already add `-ObjC` in Other Linker Flags.
2. Link _crc32 abnormal. 
Please add libz.tbd to project.
3. Link _UTTypeCopyPreferredTagWithClass is abnormal.
Please add MobileCoreServices.framework to project.
4. To save **nslog** in local， please refer to ` [self redirectNSLogToDocumentFolder] of AppDelegate.m` in the demo, .
