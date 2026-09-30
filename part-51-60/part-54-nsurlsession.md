# ตอนที่ 54: NSURLSession Deep Dive

## บทนำ

NSURLSession เป็น API ที่ทรงพลังและยืดหยุ่นสูงสำหรับการทำ Networking ใน iOS และ macOS ในบทนี้เราจะเจาะลึกทุกแง่มุมของ NSURLSession ตั้งแต่ Task Types พื้นฐานไปจนถึง Background Downloads, Authentication Challenges และการสร้าง Networking Layer ที่สมบูรณ์

---

## 54.1 NSURLSession และ NSURLSessionConfiguration

### โครงสร้างของ NSURLSession

```objc
// การสร้าง NSURLSession มี 3 วิธี

// 1. Shared Session - ง่ายสุด ไม่ต้องการ Delegate
NSURLSession *sharedSession = [NSURLSession sharedSession];

// 2. Session with Configuration - ปรับแต่งได้
NSURLSessionConfiguration *config = [NSURLSessionConfiguration defaultSessionConfiguration];
config.timeoutIntervalForRequest = 30.0;
NSURLSession *customSession = [NSURLSession sessionWithConfiguration:config];

// 3. Session with Configuration and Delegate - ควบคุมได้เต็มที่
NSURLSessionConfiguration *fullConfig = [NSURLSessionConfiguration defaultSessionConfiguration];
NSURLSession *delegateSession = [NSURLSession sessionWithConfiguration:fullConfig
                                                              delegate:self
                                                         delegateQueue:[NSOperationQueue mainQueue]];
```

### NSURLSessionConfiguration Properties ทั้งหมด

```objc
NSURLSessionConfiguration *config = [NSURLSessionConfiguration defaultSessionConfiguration];

// === Timeout ===
config.timeoutIntervalForRequest = 30.0;     // ระหว่างรอ Response
config.timeoutIntervalForResource = 604800.0; // ทั้งหมด (7 วัน)

// === Connectivity ===
config.allowsCellularAccess = YES;
// iOS 13+
config.allowsExpensiveNetworkAccess = YES;      // Hot Spot, บาง VPN
config.allowsConstrainedNetworkAccess = YES;   // Low Data Mode
config.waitsForConnectivity = YES;              // รอจนมี Internet

// === HTTP ===
config.HTTPMaximumConnectionsPerHost = 6;
config.HTTPShouldUsePipelining = NO;
config.HTTPShouldSetCookies = YES;
config.HTTPCookieAcceptPolicy = NSHTTPCookieAcceptPolicyOnlyFromMainDocumentDomain;

// === Cache ===
config.requestCachePolicy = NSURLRequestUseProtocolCachePolicy;
config.URLCache = [NSURLCache sharedURLCache];

// === Credential ===
config.URLCredentialStorage = [NSURLCredentialStorage sharedCredentialStorage];

// === Cookie ===
config.HTTPCookieStorage = [NSHTTPCookieStorage sharedHTTPCookieStorage];

// === Custom Headers ===
config.HTTPAdditionalHeaders = @{
    @"User-Agent": @"MyApp/1.0 iOS/17.0",
    @"Accept-Language": @"th-TH, en-US",
    @"X-Client-Version": @"1.0.0"
};

// === TLS ===
config.TLSMinimumSupportedProtocolVersion = tls_protocol_version_TLSv12;
config.TLSMaximumSupportedProtocolVersion = tls_protocol_version_TLSv13;

// === Multiplexing ===
config.multipathServiceType = NSURLSessionMultipathServiceTypeNone;
```

---

## 54.2 Data Tasks

Data Task เป็น Task ที่ใช้บ่อยที่สุด ใช้สำหรับ Request/Response ทั่วไป

### Data Task แบบ Completion Handler

```objc
@interface DataTaskExample : NSObject

@property (nonatomic, strong) NSURLSession *session;

@end

@implementation DataTaskExample

- (instancetype)init {
    self = [super init];
    if (self) {
        NSURLSessionConfiguration *config = [NSURLSessionConfiguration defaultSessionConfiguration];
        config.timeoutIntervalForRequest = 30.0;
        self.session = [NSURLSession sessionWithConfiguration:config];
    }
    return self;
}

// GET Request
- (void)fetchUserWithID:(NSInteger)userID 
             completion:(void(^)(NSDictionary *user, NSError *error))completion {
    
    NSString *urlString = [NSString stringWithFormat:@"https://api.example.com/users/%ld", (long)userID];
    NSURL *url = [NSURL URLWithString:urlString];
    
    NSURLSessionDataTask *task = [self.session dataTaskWithURL:url
                                             completionHandler:^(NSData *data,
                                                                 NSURLResponse *response,
                                                                 NSError *error) {
        // จัดการ Error
        if (error) {
            dispatch_async(dispatch_get_main_queue(), ^{
                completion(nil, error);
            });
            return;
        }
        
        // ตรวจสอบ Status Code
        NSHTTPURLResponse *httpResponse = (NSHTTPURLResponse *)response;
        if (httpResponse.statusCode < 200 || httpResponse.statusCode >= 300) {
            NSError *statusError = [NSError errorWithDomain:@"HTTPError"
                                                       code:httpResponse.statusCode
                                                   userInfo:@{
                NSLocalizedDescriptionKey: [NSString stringWithFormat:
                    @"HTTP Error: %ld", (long)httpResponse.statusCode]
            }];
            dispatch_async(dispatch_get_main_queue(), ^{
                completion(nil, statusError);
            });
            return;
        }
        
        // Parse JSON
        NSError *jsonError;
        NSDictionary *user = [NSJSONSerialization JSONObjectWithData:data
                                                             options:0
                                                               error:&jsonError];
        dispatch_async(dispatch_get_main_queue(), ^{
            if (jsonError) {
                completion(nil, jsonError);
            } else {
                completion(user, nil);
            }
        });
    }];
    
    [task resume]; // ต้อง resume เสมอ!
}

// POST Request ด้วย NSURLRequest
- (void)createUser:(NSDictionary *)userData
        completion:(void(^)(NSDictionary *newUser, NSError *error))completion {
    
    NSURL *url = [NSURL URLWithString:@"https://api.example.com/users"];
    NSMutableURLRequest *request = [NSMutableURLRequest requestWithURL:url];
    [request setHTTPMethod:@"POST"];
    [request setValue:@"application/json" forHTTPHeaderField:@"Content-Type"];
    
    NSError *jsonError;
    NSData *bodyData = [NSJSONSerialization dataWithJSONObject:userData
                                                       options:0
                                                         error:&jsonError];
    if (jsonError) {
        completion(nil, jsonError);
        return;
    }
    [request setHTTPBody:bodyData];
    
    NSURLSessionDataTask *task = [self.session dataTaskWithRequest:request
                                                completionHandler:^(NSData *data,
                                                                    NSURLResponse *response,
                                                                    NSError *error) {
        if (error) {
            dispatch_async(dispatch_get_main_queue(), ^{
                completion(nil, error);
            });
            return;
        }
        
        NSError *parseError;
        NSDictionary *newUser = [NSJSONSerialization JSONObjectWithData:data
                                                               options:0
                                                                 error:&parseError];
        dispatch_async(dispatch_get_main_queue(), ^{
            completion(newUser, parseError);
        });
    }];
    
    [task resume];
}

@end
```

### Data Task ด้วย Delegate

```objc
// ใช้ Delegate แทน Completion Handler เมื่อต้องการควบคุมมากขึ้น
@interface DataTaskDelegate : NSObject <NSURLSessionDataDelegate>

@property (nonatomic, strong) NSMutableData *receivedData;
@property (nonatomic, copy) void(^completionHandler)(NSData *data, NSError *error);

@end

@implementation DataTaskDelegate

- (void)URLSession:(NSURLSession *)session
          dataTask:(NSURLSessionDataTask *)dataTask
didReceiveResponse:(NSURLResponse *)response
 completionHandler:(void (^)(NSURLSessionResponseDisposition disposition))completionHandler {
    
    // เรียก completionHandler เพื่อเริ่มรับข้อมูล
    self.receivedData = [NSMutableData data];
    completionHandler(NSURLSessionResponseAllow);
    
    NSHTTPURLResponse *httpResponse = (NSHTTPURLResponse *)response;
    NSLog(@"Started receiving response, status: %ld", (long)httpResponse.statusCode);
}

- (void)URLSession:(NSURLSession *)session
          dataTask:(NSURLSessionDataTask *)dataTask
    didReceiveData:(NSData *)data {
    
    // ได้รับข้อมูล (อาจถูกเรียกหลายครั้ง)
    [self.receivedData appendData:data];
    NSLog(@"Received %lu bytes, total: %lu bytes", 
          (unsigned long)data.length, 
          (unsigned long)self.receivedData.length);
}

- (void)URLSession:(NSURLSession *)session
              task:(NSURLSessionTask *)task
didCompleteWithError:(NSError *)error {
    
    if (error) {
        NSLog(@"Task completed with error: %@", error.localizedDescription);
        if (self.completionHandler) {
            self.completionHandler(nil, error);
        }
    } else {
        NSLog(@"Task completed successfully, total data: %lu bytes",
              (unsigned long)self.receivedData.length);
        if (self.completionHandler) {
            self.completionHandler(self.receivedData, nil);
        }
    }
}

@end
```

---

## 54.3 Upload Tasks

Upload Task ใช้สำหรับส่งข้อมูลขนาดใหญ่ขึ้น Server

### Upload ด้วย Data

```objc
- (void)uploadImage:(UIImage *)image 
             toURL:(NSString *)urlString
        completion:(void(^)(BOOL success, NSError *error))completion {
    
    NSURL *url = [NSURL URLWithString:urlString];
    NSMutableURLRequest *request = [NSMutableURLRequest requestWithURL:url];
    [request setHTTPMethod:@"POST"];
    [request setValue:@"image/jpeg" forHTTPHeaderField:@"Content-Type"];
    
    // แปลง UIImage เป็น Data
    NSData *imageData = UIImageJPEGRepresentation(image, 0.8);
    
    // Upload Task ด้วย Data
    NSURLSessionUploadTask *uploadTask = 
        [self.session uploadTaskWithRequest:request
                                   fromData:imageData
                          completionHandler:^(NSData *data, NSURLResponse *response, NSError *error) {
        
        if (error) {
            dispatch_async(dispatch_get_main_queue(), ^{
                completion(NO, error);
            });
            return;
        }
        
        NSHTTPURLResponse *httpResponse = (NSHTTPURLResponse *)response;
        BOOL success = (httpResponse.statusCode >= 200 && httpResponse.statusCode < 300);
        
        dispatch_async(dispatch_get_main_queue(), ^{
            completion(success, nil);
        });
    }];
    
    [uploadTask resume];
}
```

### Multipart Upload (หลายไฟล์)

```objc
- (void)uploadMultipartData:(NSDictionary *)formData 
                      files:(NSArray<NSDictionary *> *)files
                      toURL:(NSString *)urlString
                 completion:(void(^)(NSDictionary *response, NSError *error))completion {
    
    // สร้าง Boundary
    NSString *boundary = [NSString stringWithFormat:@"Boundary-%@",
                          [[NSUUID UUID] UUIDString]];
    
    NSURL *url = [NSURL URLWithString:urlString];
    NSMutableURLRequest *request = [NSMutableURLRequest requestWithURL:url];
    [request setHTTPMethod:@"POST"];
    [request setValue:[NSString stringWithFormat:@"multipart/form-data; boundary=%@", boundary]
               forHTTPHeaderField:@"Content-Type"];
    
    // สร้าง Body
    NSMutableData *body = [NSMutableData data];
    
    // Form Fields
    [formData enumerateKeysAndObjectsUsingBlock:^(NSString *key, NSString *value, BOOL *stop) {
        [body appendData:[[NSString stringWithFormat:@"--%@\r\n", boundary] 
                          dataUsingEncoding:NSUTF8StringEncoding]];
        [body appendData:[[NSString stringWithFormat:@"Content-Disposition: form-data; name=\"%@\"\r\n\r\n", key]
                          dataUsingEncoding:NSUTF8StringEncoding]];
        [body appendData:[[NSString stringWithFormat:@"%@\r\n", value]
                          dataUsingEncoding:NSUTF8StringEncoding]];
    }];
    
    // Files
    for (NSDictionary *fileInfo in files) {
        NSString *fieldName = fileInfo[@"fieldName"];
        NSString *fileName = fileInfo[@"fileName"];
        NSString *mimeType = fileInfo[@"mimeType"];
        NSData *fileData = fileInfo[@"data"];
        
        [body appendData:[[NSString stringWithFormat:@"--%@\r\n", boundary]
                          dataUsingEncoding:NSUTF8StringEncoding]];
        [body appendData:[[NSString stringWithFormat:
                           @"Content-Disposition: form-data; name=\"%@\"; filename=\"%@\"\r\n",
                           fieldName, fileName]
                          dataUsingEncoding:NSUTF8StringEncoding]];
        [body appendData:[[NSString stringWithFormat:@"Content-Type: %@\r\n\r\n", mimeType]
                          dataUsingEncoding:NSUTF8StringEncoding]];
        [body appendData:fileData];
        [body appendData:[@"\r\n" dataUsingEncoding:NSUTF8StringEncoding]];
    }
    
    // Closing Boundary
    [body appendData:[[NSString stringWithFormat:@"--%@--\r\n", boundary]
                      dataUsingEncoding:NSUTF8StringEncoding]];
    
    NSURLSessionUploadTask *uploadTask = 
        [self.session uploadTaskWithRequest:request
                                   fromData:body
                          completionHandler:^(NSData *data, NSURLResponse *response, NSError *error) {
        if (error) {
            dispatch_async(dispatch_get_main_queue(), ^{ completion(nil, error); });
            return;
        }
        
        NSError *jsonError;
        NSDictionary *json = [NSJSONSerialization JSONObjectWithData:data 
                                                             options:0 
                                                               error:&jsonError];
        dispatch_async(dispatch_get_main_queue(), ^{
            completion(json, jsonError);
        });
    }];
    
    [uploadTask resume];
}

// การใช้งาน
UIImage *profilePhoto = [UIImage imageNamed:@"profile"];
NSData *photoData = UIImageJPEGRepresentation(profilePhoto, 0.8);

NSDictionary *fields = @{@"name": @"John Doe", @"email": @"john@example.com"};
NSArray *files = @[@{
    @"fieldName": @"photo",
    @"fileName": @"profile.jpg",
    @"mimeType": @"image/jpeg",
    @"data": photoData
}];

[manager uploadMultipartData:fields 
                       files:files 
                       toURL:@"https://api.example.com/users"
                  completion:^(NSDictionary *response, NSError *error) {
    if (response) {
        NSLog(@"Upload successful! New user ID: %@", response[@"id"]);
    }
}];
```

---

## 54.4 Download Tasks

Download Task ใช้สำหรับดาวน์โหลดไฟล์และบันทึกลง Disk

```objc
@interface DownloadManager : NSObject <NSURLSessionDownloadDelegate>

@property (nonatomic, strong) NSURLSession *session;
@property (nonatomic, strong) NSMutableDictionary *downloadProgress;
@property (nonatomic, strong) NSMutableDictionary *downloadCompletions;

+ (instancetype)shared;

- (NSURLSessionDownloadTask *)downloadFileFromURL:(NSString *)urlString
                                         progress:(void(^)(float progress))progressHandler
                                       completion:(void(^)(NSURL *localURL, NSError *error))completion;

@end

@implementation DownloadManager

+ (instancetype)shared {
    static DownloadManager *instance = nil;
    static dispatch_once_t onceToken;
    dispatch_once(&onceToken, ^{
        instance = [[DownloadManager alloc] init];
    });
    return instance;
}

- (instancetype)init {
    self = [super init];
    if (self) {
        NSURLSessionConfiguration *config = [NSURLSessionConfiguration defaultSessionConfiguration];
        self.session = [NSURLSession sessionWithConfiguration:config
                                                     delegate:self
                                                delegateQueue:nil];
        self.downloadProgress = [NSMutableDictionary dictionary];
        self.downloadCompletions = [NSMutableDictionary dictionary];
    }
    return self;
}

- (NSURLSessionDownloadTask *)downloadFileFromURL:(NSString *)urlString
                                         progress:(void(^)(float progress))progressHandler
                                       completion:(void(^)(NSURL *localURL, NSError *error))completion {
    
    NSURL *url = [NSURL URLWithString:urlString];
    NSURLSessionDownloadTask *task = [self.session downloadTaskWithURL:url];
    
    // เก็บ handlers
    NSString *taskID = [NSString stringWithFormat:@"%lu", (unsigned long)task.taskIdentifier];
    if (progressHandler) {
        self.downloadProgress[taskID] = progressHandler;
    }
    if (completion) {
        self.downloadCompletions[taskID] = completion;
    }
    
    [task resume];
    return task;
}

// Download Progress
- (void)URLSession:(NSURLSession *)session
      downloadTask:(NSURLSessionDownloadTask *)downloadTask
      didWriteData:(int64_t)bytesWritten
 totalBytesWritten:(int64_t)totalBytesWritten
totalBytesExpectedToWrite:(int64_t)totalBytesExpectedToWrite {
    
    float progress = 0.0;
    if (totalBytesExpectedToWrite > 0) {
        progress = (float)totalBytesWritten / (float)totalBytesExpectedToWrite;
    }
    
    NSString *taskID = [NSString stringWithFormat:@"%lu", 
                        (unsigned long)downloadTask.taskIdentifier];
    void(^progressHandler)(float) = self.downloadProgress[taskID];
    
    if (progressHandler) {
        dispatch_async(dispatch_get_main_queue(), ^{
            progressHandler(progress);
        });
    }
    
    NSLog(@"Downloaded: %.1f%% (%lld / %lld bytes)", 
          progress * 100, totalBytesWritten, totalBytesExpectedToWrite);
}

// Download Finished
- (void)URLSession:(NSURLSession *)session
      downloadTask:(NSURLSessionDownloadTask *)downloadTask
didFinishDownloadingToURL:(NSURL *)location {
    
    // ย้ายไฟล์ไปยัง Documents
    NSFileManager *fileManager = [NSFileManager defaultManager];
    NSURL *documentsURL = [fileManager URLsForDirectory:NSDocumentDirectory 
                                              inDomains:NSUserDomainMask].firstObject;
    
    NSString *fileName = downloadTask.currentRequest.URL.lastPathComponent;
    NSURL *destinationURL = [documentsURL URLByAppendingPathComponent:fileName];
    
    NSError *moveError;
    if ([fileManager fileExistsAtPath:destinationURL.path]) {
        [fileManager removeItemAtURL:destinationURL error:nil];
    }
    [fileManager moveItemAtURL:location toURL:destinationURL error:&moveError];
    
    NSString *taskID = [NSString stringWithFormat:@"%lu", 
                        (unsigned long)downloadTask.taskIdentifier];
    void(^completion)(NSURL *, NSError *) = self.downloadCompletions[taskID];
    
    if (completion) {
        dispatch_async(dispatch_get_main_queue(), ^{
            completion(moveError ? nil : destinationURL, moveError);
        });
        [self.downloadCompletions removeObjectForKey:taskID];
    }
    [self.downloadProgress removeObjectForKey:taskID];
}

// Task Completed (Error handling)
- (void)URLSession:(NSURLSession *)session
              task:(NSURLSessionTask *)task
didCompleteWithError:(NSError *)error {
    
    if (error) {
        NSString *taskID = [NSString stringWithFormat:@"%lu", 
                            (unsigned long)task.taskIdentifier];
        void(^completion)(NSURL *, NSError *) = self.downloadCompletions[taskID];
        
        if (completion) {
            dispatch_async(dispatch_get_main_queue(), ^{
                completion(nil, error);
            });
            [self.downloadCompletions removeObjectForKey:taskID];
        }
        [self.downloadProgress removeObjectForKey:taskID];
    }
}

@end

// การใช้งาน
[[DownloadManager shared] downloadFileFromURL:@"https://example.com/large-file.pdf"
                                     progress:^(float progress) {
    NSLog(@"Progress: %.0f%%", progress * 100);
    // อัปเดต Progress Bar
}
                                   completion:^(NSURL *localURL, NSError *error) {
    if (localURL) {
        NSLog(@"Downloaded to: %@", localURL.path);
    } else {
        NSLog(@"Download failed: %@", error.localizedDescription);
    }
}];
```

---

## 54.5 Stream Tasks

Stream Task ใช้สำหรับการสื่อสารแบบ Two-way เช่น WebSocket หรือ Server-Sent Events

```objc
// Stream Task (TCP Connection)
NSURL *url = [NSURL URLWithString:@"https://stream.example.com"];
NSURLSessionStreamTask *streamTask = [self.session streamTaskWithHostName:@"stream.example.com"
                                                                     port:443];
[streamTask resume];

// เขียนข้อมูล
NSData *requestData = [@"Hello Server!" dataUsingEncoding:NSUTF8StringEncoding];
[streamTask writeData:requestData timeout:10.0 completionHandler:^(NSError *error) {
    if (error) {
        NSLog(@"Write error: %@", error);
    } else {
        NSLog(@"Data sent successfully");
    }
}];

// อ่านข้อมูล
[streamTask readDataOfMinLength:1 
                      maxLength:65536 
                        timeout:30.0
              completionHandler:^(NSData *data, BOOL atEOF, NSError *error) {
    if (error) {
        NSLog(@"Read error: %@", error);
    } else if (data) {
        NSString *response = [[NSString alloc] initWithData:data encoding:NSUTF8StringEncoding];
        NSLog(@"Received: %@", response);
        
        if (!atEOF) {
            // อ่านต่อ...
        }
    }
}];

// ปิด Stream
[streamTask closeWrite];
```

---

## 54.6 Background Download/Upload

### Background Download

```objc
@interface BackgroundDownloadManager : NSObject <NSURLSessionDownloadDelegate>

@property (nonatomic, strong) NSURLSession *backgroundSession;
@property (nonatomic, copy) void (^backgroundSessionCompletionHandler)(void);

+ (instancetype)shared;
- (void)downloadInBackground:(NSString *)urlString;
- (NSURLSession *)recreateBackgroundSession;

@end

@implementation BackgroundDownloadManager

static NSString *const BackgroundSessionIdentifier = @"com.myapp.background-download";

+ (instancetype)shared {
    static BackgroundDownloadManager *instance = nil;
    static dispatch_once_t onceToken;
    dispatch_once(&onceToken, ^{
        instance = [[BackgroundDownloadManager alloc] init];
    });
    return instance;
}

- (instancetype)init {
    self = [super init];
    if (self) {
        [self recreateBackgroundSession];
    }
    return self;
}

- (NSURLSession *)recreateBackgroundSession {
    NSURLSessionConfiguration *config = 
        [NSURLSessionConfiguration backgroundSessionConfigurationWithIdentifier:
            BackgroundSessionIdentifier];
    
    // ตั้งค่า
    config.discretionary = NO;            // ดาวน์โหลดทันที (ไม่รอ iOS เลือกเวลา)
    config.sessionSendsLaunchEvents = YES; // ปลุก App เมื่อเสร็จ
    
    self.backgroundSession = [NSURLSession sessionWithConfiguration:config
                                                           delegate:self
                                                      delegateQueue:nil];
    return self.backgroundSession;
}

- (void)downloadInBackground:(NSString *)urlString {
    NSURL *url = [NSURL URLWithString:urlString];
    NSURLSessionDownloadTask *task = [self.backgroundSession downloadTaskWithURL:url];
    [task resume];
    
    NSLog(@"Background download started: %@", urlString);
}

// เรียกเมื่อดาวน์โหลดเสร็จ
- (void)URLSession:(NSURLSession *)session
      downloadTask:(NSURLSessionDownloadTask *)downloadTask
didFinishDownloadingToURL:(NSURL *)location {
    
    // ย้ายไฟล์
    NSFileManager *fileManager = [NSFileManager defaultManager];
    NSString *fileName = downloadTask.originalRequest.URL.lastPathComponent;
    NSURL *documentsURL = [[fileManager URLsForDirectory:NSDocumentDirectory 
                                               inDomains:NSUserDomainMask] firstObject];
    NSURL *destination = [documentsURL URLByAppendingPathComponent:fileName];
    
    NSError *error;
    if ([fileManager fileExistsAtPath:destination.path]) {
        [fileManager removeItemAtURL:destination error:nil];
    }
    [fileManager moveItemAtURL:location toURL:destination error:&error];
    
    if (!error) {
        NSLog(@"Background download complete: %@", destination.path);
        // แจ้งเตือน User ถ้า App อยู่ Background
        [self sendLocalNotification:fileName];
    }
}

- (void)URLSessionDidFinishEventsForBackgroundURLSession:(NSURLSession *)session {
    dispatch_async(dispatch_get_main_queue(), ^{
        // เรียก completion handler ที่ได้รับจาก AppDelegate
        if (self.backgroundSessionCompletionHandler) {
            self.backgroundSessionCompletionHandler();
            self.backgroundSessionCompletionHandler = nil;
        }
    });
}

- (void)sendLocalNotification:(NSString *)fileName {
    UNMutableNotificationContent *content = [[UNMutableNotificationContent alloc] init];
    content.title = @"Download Complete";
    content.body = [NSString stringWithFormat:@"%@ has been downloaded", fileName];
    content.sound = [UNNotificationSound defaultSound];
    
    UNNotificationRequest *request = [UNNotificationRequest requestWithIdentifier:[[NSUUID UUID] UUIDString]
                                                                          content:content
                                                                          trigger:nil];
    
    [[UNUserNotificationCenter currentNotificationCenter] addNotificationRequest:request
                                                           withCompletionHandler:nil];
}

@end

// ใน AppDelegate
- (void)application:(UIApplication *)application 
handleEventsForBackgroundURLSession:(NSString *)identifier
  completionHandler:(void (^)(void))completionHandler {
    
    if ([identifier isEqualToString:@"com.myapp.background-download"]) {
        [BackgroundDownloadManager shared].backgroundSessionCompletionHandler = completionHandler;
    }
}
```

### Resumable Downloads

```objc
@interface ResumableDownloader : NSObject <NSURLSessionDownloadDelegate>

@property (nonatomic, strong) NSURLSession *session;
@property (nonatomic, strong) NSURLSessionDownloadTask *currentTask;
@property (nonatomic, strong) NSData *resumeData;
@property (nonatomic, copy) void(^progressHandler)(float progress);
@property (nonatomic, copy) void(^completionHandler)(NSURL *url, NSError *error);

- (void)startDownload:(NSString *)urlString;
- (void)pauseDownload;
- (void)resumeDownload;
- (void)cancelDownload;

@end

@implementation ResumableDownloader

- (instancetype)init {
    self = [super init];
    if (self) {
        NSURLSessionConfiguration *config = [NSURLSessionConfiguration defaultSessionConfiguration];
        self.session = [NSURLSession sessionWithConfiguration:config
                                                     delegate:self
                                                delegateQueue:nil];
    }
    return self;
}

- (void)startDownload:(NSString *)urlString {
    NSURL *url = [NSURL URLWithString:urlString];
    self.currentTask = [self.session downloadTaskWithURL:url];
    [self.currentTask resume];
}

- (void)pauseDownload {
    if (self.currentTask) {
        [self.currentTask cancelByProducingResumeData:^(NSData *resumeData) {
            self.resumeData = resumeData;
            self.currentTask = nil;
            NSLog(@"Download paused, resume data: %lu bytes", 
                  (unsigned long)resumeData.length);
            
            // บันทึก resumeData ไว้ใน Disk
            [self saveResumeData:resumeData];
        }];
    }
}

- (void)resumeDownload {
    NSData *savedResumeData = self.resumeData ?: [self loadResumeData];
    
    if (savedResumeData) {
        self.currentTask = [self.session downloadTaskWithResumeData:savedResumeData];
        [self.currentTask resume];
        self.resumeData = nil;
        NSLog(@"Download resumed");
    } else {
        NSLog(@"No resume data available");
    }
}

- (void)cancelDownload {
    [self.currentTask cancel];
    self.currentTask = nil;
    self.resumeData = nil;
    [self deleteResumeData];
    NSLog(@"Download cancelled");
}

- (void)saveResumeData:(NSData *)data {
    NSString *path = [NSTemporaryDirectory() stringByAppendingPathComponent:@"resumeData.tmp"];
    [data writeToFile:path atomically:YES];
}

- (NSData *)loadResumeData {
    NSString *path = [NSTemporaryDirectory() stringByAppendingPathComponent:@"resumeData.tmp"];
    return [NSData dataWithContentsOfFile:path];
}

- (void)deleteResumeData {
    NSString *path = [NSTemporaryDirectory() stringByAppendingPathComponent:@"resumeData.tmp"];
    [[NSFileManager defaultManager] removeItemAtPath:path error:nil];
}

// Delegate Methods
- (void)URLSession:(NSURLSession *)session
      downloadTask:(NSURLSessionDownloadTask *)downloadTask
      didWriteData:(int64_t)bytesWritten
 totalBytesWritten:(int64_t)totalBytesWritten
totalBytesExpectedToWrite:(int64_t)totalBytesExpectedToWrite {
    
    if (totalBytesExpectedToWrite > 0 && self.progressHandler) {
        float progress = (float)totalBytesWritten / totalBytesExpectedToWrite;
        dispatch_async(dispatch_get_main_queue(), ^{
            self.progressHandler(progress);
        });
    }
}

- (void)URLSession:(NSURLSession *)session
      downloadTask:(NSURLSessionDownloadTask *)downloadTask
didFinishDownloadingToURL:(NSURL *)location {
    
    // ย้ายไฟล์
    NSFileManager *fm = [NSFileManager defaultManager];
    NSURL *docs = [[fm URLsForDirectory:NSDocumentDirectory inDomains:NSUserDomainMask] firstObject];
    NSURL *dest = [docs URLByAppendingPathComponent:@"downloaded_file"];
    
    NSError *error;
    [fm moveItemAtURL:location toURL:dest error:&error];
    
    dispatch_async(dispatch_get_main_queue(), ^{
        if (self.completionHandler) {
            self.completionHandler(error ? nil : dest, error);
        }
    });
}

@end
```

---

## 54.7 NSURLSessionDelegate

### Delegate Protocol ทั้งหมด

```
NSURLSessionDelegate (Base)
├── URLSession:didBecomeInvalidWithError:
├── URLSession:didReceiveChallenge:completionHandler:
└── URLSessionDidFinishEventsForBackgroundURLSession:

NSURLSessionTaskDelegate (extends NSURLSessionDelegate)
├── URLSession:task:willPerformHTTPRedirection:newRequest:completionHandler:
├── URLSession:task:didReceiveChallenge:completionHandler:
├── URLSession:task:needNewBodyStream:
├── URLSession:task:didSendBodyData:totalBytesSent:totalBytesExpectedToSend:
├── URLSession:task:didFinishCollectingMetrics:
└── URLSession:task:didCompleteWithError:

NSURLSessionDataDelegate (extends NSURLSessionTaskDelegate)
├── URLSession:dataTask:didReceiveResponse:completionHandler:
├── URLSession:dataTask:didBecomeDownloadTask:
├── URLSession:dataTask:didBecomeStreamTask:
├── URLSession:dataTask:didReceiveData:
└── URLSession:dataTask:willCacheResponse:completionHandler:

NSURLSessionDownloadDelegate (extends NSURLSessionTaskDelegate)
├── URLSession:downloadTask:didFinishDownloadingToURL:
├── URLSession:downloadTask:didWriteData:totalBytesWritten:totalBytesExpectedToWrite:
└── URLSession:downloadTask:didResumeAtOffset:expectedTotalBytes:

NSURLSessionStreamDelegate (extends NSURLSessionTaskDelegate)
├── URLSession:readClosedForStreamTask:
├── URLSession:writeClosedForStreamTask:
├── URLSession:betterRouteDiscoveredForStreamTask:
└── URLSession:streamTask:didBecomeInputStream:outputStream:
```

---

## 54.8 Authentication Challenges

```objc
@interface AuthenticationHandler : NSObject <NSURLSessionDelegate, NSURLSessionTaskDelegate>
@end

@implementation AuthenticationHandler

// Session-level Authentication
- (void)URLSession:(NSURLSession *)session
didReceiveChallenge:(NSURLAuthenticationChallenge *)challenge
 completionHandler:(void (^)(NSURLSessionAuthChallengeDisposition, NSURLCredential *))completionHandler {
    
    NSString *authMethod = challenge.protectionSpace.authenticationMethod;
    NSLog(@"Auth challenge: %@", authMethod);
    
    if ([authMethod isEqualToString:NSURLAuthenticationMethodServerTrust]) {
        // SSL/TLS Certificate Validation
        SecTrustRef serverTrust = challenge.protectionSpace.serverTrust;
        NSURLCredential *credential = [NSURLCredential credentialForTrust:serverTrust];
        completionHandler(NSURLSessionAuthChallengeUseCredential, credential);
        
    } else if ([authMethod isEqualToString:NSURLAuthenticationMethodClientCertificate]) {
        // Client Certificate
        [self handleClientCertificateChallenge:challenge completionHandler:completionHandler];
        
    } else {
        completionHandler(NSURLSessionAuthChallengePerformDefaultHandling, nil);
    }
}

// Task-level Authentication (สำหรับ HTTP Basic/Digest)
- (void)URLSession:(NSURLSession *)session
              task:(NSURLSessionTask *)task
didReceiveChallenge:(NSURLAuthenticationChallenge *)challenge
 completionHandler:(void (^)(NSURLSessionAuthChallengeDisposition, NSURLCredential *))completionHandler {
    
    NSString *authMethod = challenge.protectionSpace.authenticationMethod;
    
    if ([authMethod isEqualToString:NSURLAuthenticationMethodHTTPBasic] ||
        [authMethod isEqualToString:NSURLAuthenticationMethodHTTPDigest]) {
        
        if (challenge.previousFailureCount == 0) {
            // ครั้งแรก - ลอง Credentials
            NSURLCredential *credential = [NSURLCredential credentialWithUser:@"username"
                                                                     password:@"password"
                                                                  persistence:NSURLCredentialPersistenceForSession];
            completionHandler(NSURLSessionAuthChallengeUseCredential, credential);
            
        } else {
            // ล้มเหลวแล้ว - ยกเลิก
            NSLog(@"Authentication failed after %ld attempts", 
                  (long)challenge.previousFailureCount);
            completionHandler(NSURLSessionAuthChallengeCancelAuthenticationChallenge, nil);
        }
        
    } else if ([authMethod isEqualToString:NSURLAuthenticationMethodNTLM]) {
        // Windows NTLM Authentication
        NSURLCredential *credential = [NSURLCredential credentialWithUser:@"DOMAIN\\username"
                                                                 password:@"password"
                                                              persistence:NSURLCredentialPersistenceForSession];
        completionHandler(NSURLSessionAuthChallengeUseCredential, credential);
        
    } else {
        completionHandler(NSURLSessionAuthChallengePerformDefaultHandling, nil);
    }
}

- (void)handleClientCertificateChallenge:(NSURLAuthenticationChallenge *)challenge
                       completionHandler:(void (^)(NSURLSessionAuthChallengeDisposition, 
                                                   NSURLCredential *))completionHandler {
    // โหลด Client Certificate จาก Keychain หรือ Bundle
    NSString *certPath = [[NSBundle mainBundle] pathForResource:@"client" ofType:@"p12"];
    NSData *certData = [NSData dataWithContentsOfFile:certPath];
    
    if (!certData) {
        completionHandler(NSURLSessionAuthChallengeCancelAuthenticationChallenge, nil);
        return;
    }
    
    CFDataRef certDataRef = (__bridge CFDataRef)certData;
    CFStringRef password = (__bridge CFStringRef)@"certificate_password";
    
    const void *keys[] = { kSecImportExportPassphrase };
    const void *values[] = { password };
    CFDictionaryRef options = CFDictionaryCreate(NULL, keys, values, 1, NULL, NULL);
    
    CFArrayRef items;
    OSStatus status = SecPKCS12Import(certDataRef, options, &items);
    CFRelease(options);
    
    if (status == errSecSuccess && CFArrayGetCount(items) > 0) {
        CFDictionaryRef identity = CFArrayGetValueAtIndex(items, 0);
        SecIdentityRef identityRef = (SecIdentityRef)CFDictionaryGetValue(identity, 
                                                                           kSecImportItemIdentity);
        
        NSURLCredential *credential = [NSURLCredential credentialWithIdentity:identityRef
                                                                   certificates:nil
                                                                    persistence:NSURLCredentialPersistenceForSession];
        completionHandler(NSURLSessionAuthChallengeUseCredential, credential);
    } else {
        completionHandler(NSURLSessionAuthChallengeCancelAuthenticationChallenge, nil);
    }
    
    if (items) CFRelease(items);
}

@end
```

---

## 54.9 Progress Tracking

```objc
@interface ProgressTracker : NSObject <NSURLSessionTaskDelegate>

@property (nonatomic, weak) UIProgressView *progressView;
@property (nonatomic, weak) UILabel *progressLabel;

@end

@implementation ProgressTracker

// Upload Progress
- (void)URLSession:(NSURLSession *)session
              task:(NSURLSessionTask *)task
   didSendBodyData:(int64_t)bytesSent
    totalBytesSent:(int64_t)totalBytesSent
totalBytesExpectedToSend:(int64_t)totalBytesExpectedToSend {
    
    float progress = 0.0;
    if (totalBytesExpectedToSend > 0) {
        progress = (float)totalBytesSent / totalBytesExpectedToSend;
    }
    
    dispatch_async(dispatch_get_main_queue(), ^{
        self.progressView.progress = progress;
        self.progressLabel.text = [NSString stringWithFormat:@"%.0f%% (%.1f MB / %.1f MB)",
                                   progress * 100,
                                   totalBytesSent / 1024.0 / 1024.0,
                                   totalBytesExpectedToSend / 1024.0 / 1024.0];
    });
}

@end

// ใช้ KVO สำหรับ Progress Tracking
NSURLSessionDownloadTask *task = [session downloadTaskWithURL:url];

// iOS 11+: ใช้ Progress Object
NSProgress *progress = task.progress;
[progress addObserver:self forKeyPath:@"fractionCompleted" options:NSKeyValueObservingOptionNew context:nil];

[task resume];

// KVO Observer
- (void)observeValueForKeyPath:(NSString *)keyPath
                      ofObject:(id)object
                        change:(NSDictionary *)change
                       context:(void *)context {
    if ([keyPath isEqualToString:@"fractionCompleted"]) {
        NSProgress *progress = (NSProgress *)object;
        dispatch_async(dispatch_get_main_queue(), ^{
            float fraction = (float)progress.fractionCompleted;
            NSLog(@"Progress: %.1f%%", fraction * 100);
            // อัปเดต UI
        });
    }
}
```

---

## 54.10 Canceling Tasks

```objc
// เก็บ Task references
@property (nonatomic, strong) NSMutableDictionary<NSString *, NSURLSessionTask *> *activeTasks;

// ยกเลิก Task เฉพาะ
- (void)cancelTaskWithID:(NSString *)taskID {
    NSURLSessionTask *task = self.activeTasks[taskID];
    if (task) {
        [task cancel];
        [self.activeTasks removeObjectForKey:taskID];
        NSLog(@"Task %@ cancelled", taskID);
    }
}

// ยกเลิก Task ทั้งหมด
- (void)cancelAllTasks {
    [self.activeTasks enumerateKeysAndObjectsUsingBlock:^(NSString *taskID, 
                                                          NSURLSessionTask *task, 
                                                          BOOL *stop) {
        [task cancel];
    }];
    [self.activeTasks removeAllObjects];
}

// Invalidate Session
- (void)invalidateSession {
    // รอให้ Tasks ปัจจุบันเสร็จแล้วค่อย Invalidate
    [self.session finishTasksAndInvalidate];
    
    // หรือ Invalidate ทันที
    // [self.session invalidateAndCancel];
}

// Task State
- (void)checkTaskState:(NSURLSessionTask *)task {
    switch (task.state) {
        case NSURLSessionTaskStateRunning:
            NSLog(@"Task is running");
            break;
        case NSURLSessionTaskStateSuspended:
            NSLog(@"Task is suspended");
            [task resume]; // ปลุกขึ้นมา
            break;
        case NSURLSessionTaskStateCanceling:
            NSLog(@"Task is being cancelled");
            break;
        case NSURLSessionTaskStateCompleted:
            NSLog(@"Task is completed");
            break;
    }
}
```

---

## 54.11 Cookie Handling

```objc
// จัดการ Cookies
NSHTTPCookieStorage *cookieStorage = [NSHTTPCookieStorage sharedHTTPCookieStorage];

// สร้าง Cookie
NSHTTPCookie *cookie = [NSHTTPCookie cookieWithProperties:@{
    NSHTTPCookieName: @"session_token",
    NSHTTPCookieValue: @"abc123xyz",
    NSHTTPCookieDomain: @"api.example.com",
    NSHTTPCookiePath: @"/",
    NSHTTPCookieSecure: @YES,
    NSHTTPCookieExpires: [NSDate dateWithTimeIntervalSinceNow:86400] // 1 วัน
}];
[cookieStorage setCookie:cookie];

// อ่าน Cookies
NSArray *cookies = [cookieStorage cookiesForURL:[NSURL URLWithString:@"https://api.example.com"]];
for (NSHTTPCookie *c in cookies) {
    NSLog(@"Cookie: %@ = %@", c.name, c.value);
}

// ลบ Cookie
[cookieStorage deleteCookie:cookie];

// ตั้งค่า Cookie Accept Policy
cookieStorage.cookieAcceptPolicy = NSHTTPCookieAcceptPolicyAlways;
// หรือ
// NSHTTPCookieAcceptPolicyNever - ไม่รับ Cookies เลย
// NSHTTPCookieAcceptPolicyOnlyFromMainDocumentDomain - รับเฉพาะ Main Domain

// แนบ Cookies ใน Request
NSDictionary *cookieHeaders = [NSHTTPCookie requestHeaderFieldsWithCookies:cookies];
[request setAllHTTPHeaderFields:cookieHeaders];
```

---

## 54.12 Redirects Handling

```objc
// จัดการ HTTP Redirects
- (void)URLSession:(NSURLSession *)session
              task:(NSURLSessionTask *)task
willPerformHTTPRedirection:(NSHTTPURLResponse *)response
        newRequest:(NSURLRequest *)request
 completionHandler:(void (^)(NSURLRequest *))completionHandler {
    
    NSLog(@"Redirect from: %@ to: %@",
          task.currentRequest.URL.absoluteString,
          request.URL.absoluteString);
    
    // ตรวจสอบ redirect destination
    if ([request.URL.host isEqualToString:@"trusted-domain.com"]) {
        // อนุญาต Redirect
        completionHandler(request);
    } else if (response.statusCode == 301) {
        // Permanent Redirect - อนุญาต แต่บันทึก URL ใหม่
        [[NSUserDefaults standardUserDefaults] setObject:request.URL.absoluteString
                                                  forKey:@"newBaseURL"];
        completionHandler(request);
    } else {
        // ปฏิเสธ Redirect
        completionHandler(nil);
    }
}
```

---

## 54.13 Complete Networking Layer

```objc
// NetworkLayer.h
typedef NS_ENUM(NSInteger, HTTPMethod) {
    HTTPMethodGET,
    HTTPMethodPOST,
    HTTPMethodPUT,
    HTTPMethodPATCH,
    HTTPMethodDELETE
};

typedef void(^NetworkCompletion)(id responseObject, NSHTTPURLResponse *response, NSError *error);
typedef void(^ProgressCompletion)(float progress);

@interface NetworkRequest : NSObject
@property (nonatomic, copy) NSString *urlString;
@property (nonatomic, assign) HTTPMethod method;
@property (nonatomic, strong) NSDictionary *headers;
@property (nonatomic, strong) NSDictionary *parameters;
@property (nonatomic, strong) NSData *body;
@property (nonatomic, assign) NSTimeInterval timeout;
@end

@interface NetworkLayer : NSObject

+ (instancetype)shared;

- (NSURLSessionDataTask *)executeRequest:(NetworkRequest *)request
                              completion:(NetworkCompletion)completion;

- (NSURLSessionDownloadTask *)downloadRequest:(NetworkRequest *)request
                                     progress:(ProgressCompletion)progress
                                   completion:(void(^)(NSURL *localURL, NSError *error))completion;

- (NSURLSessionUploadTask *)uploadData:(NSData *)data
                               request:(NetworkRequest *)request
                               progress:(ProgressCompletion)progress
                             completion:(NetworkCompletion)completion;

- (void)cancelAllRequests;
- (void)setAuthToken:(NSString *)token;

@end

// NetworkLayer.m
@interface NetworkLayer () <NSURLSessionDelegate, NSURLSessionTaskDelegate,
                             NSURLSessionDataDelegate, NSURLSessionDownloadDelegate>

@property (nonatomic, strong) NSURLSession *session;
@property (nonatomic, strong) NSString *authToken;
@property (nonatomic, strong) NSMutableDictionary *activeTasks;

@end

@implementation NetworkLayer

+ (instancetype)shared {
    static NetworkLayer *instance = nil;
    static dispatch_once_t onceToken;
    dispatch_once(&onceToken, ^{
        instance = [[NetworkLayer alloc] init];
    });
    return instance;
}

- (instancetype)init {
    self = [super init];
    if (self) {
        NSURLSessionConfiguration *config = [NSURLSessionConfiguration defaultSessionConfiguration];
        config.timeoutIntervalForRequest = 30.0;
        config.waitsForConnectivity = YES;
        config.HTTPAdditionalHeaders = @{
            @"User-Agent": @"MyApp/1.0",
            @"Accept": @"application/json"
        };
        
        self.session = [NSURLSession sessionWithConfiguration:config
                                                     delegate:self
                                                delegateQueue:nil];
        self.activeTasks = [NSMutableDictionary dictionary];
    }
    return self;
}

- (void)setAuthToken:(NSString *)token {
    self.authToken = token;
}

- (NSURLRequest *)buildRequestFrom:(NetworkRequest *)networkRequest {
    NSURL *url = [NSURL URLWithString:networkRequest.urlString];
    
    // เพิ่ม Query Parameters สำหรับ GET
    if (networkRequest.method == HTTPMethodGET && networkRequest.parameters.count > 0) {
        NSURLComponents *components = [NSURLComponents componentsWithURL:url resolvingAgainstBaseURL:NO];
        NSMutableArray *queryItems = [NSMutableArray array];
        [networkRequest.parameters enumerateKeysAndObjectsUsingBlock:^(NSString *key, id value, BOOL *stop) {
            [queryItems addObject:[NSURLQueryItem queryItemWithName:key value:[value description]]];
        }];
        components.queryItems = queryItems;
        url = components.URL;
    }
    
    NSMutableURLRequest *request = [NSMutableURLRequest requestWithURL:url];
    
    // Method
    NSArray *methods = @[@"GET", @"POST", @"PUT", @"PATCH", @"DELETE"];
    [request setHTTPMethod:methods[networkRequest.method]];
    
    // Timeout
    request.timeoutInterval = networkRequest.timeout > 0 ? networkRequest.timeout : 30.0;
    
    // Auth Token
    if (self.authToken) {
        [request setValue:[NSString stringWithFormat:@"Bearer %@", self.authToken]
                   forHTTPHeaderField:@"Authorization"];
    }
    
    // Custom Headers
    [networkRequest.headers enumerateKeysAndObjectsUsingBlock:^(NSString *key, NSString *value, BOOL *stop) {
        [request setValue:value forHTTPHeaderField:key];
    }];
    
    // Body
    if (networkRequest.body) {
        [request setHTTPBody:networkRequest.body];
    } else if (networkRequest.parameters && networkRequest.method != HTTPMethodGET) {
        NSError *jsonError;
        NSData *bodyData = [NSJSONSerialization dataWithJSONObject:networkRequest.parameters
                                                           options:0
                                                             error:&jsonError];
        if (!jsonError) {
            [request setHTTPBody:bodyData];
            [request setValue:@"application/json" forHTTPHeaderField:@"Content-Type"];
        }
    }
    
    return request;
}

- (NSURLSessionDataTask *)executeRequest:(NetworkRequest *)request
                              completion:(NetworkCompletion)completion {
    
    NSURLRequest *urlRequest = [self buildRequestFrom:request];
    
    NSURLSessionDataTask *task = [self.session dataTaskWithRequest:urlRequest
                                                completionHandler:^(NSData *data,
                                                                    NSURLResponse *response,
                                                                    NSError *error) {
        NSHTTPURLResponse *httpResponse = (NSHTTPURLResponse *)response;
        
        if (error) {
            dispatch_async(dispatch_get_main_queue(), ^{
                completion(nil, httpResponse, error);
            });
            return;
        }
        
        NSError *jsonError;
        id jsonObject = nil;
        if (data.length > 0) {
            jsonObject = [NSJSONSerialization JSONObjectWithData:data options:0 error:&jsonError];
        }
        
        dispatch_async(dispatch_get_main_queue(), ^{
            completion(jsonObject, httpResponse, jsonError);
        });
    }];
    
    [task resume];
    return task;
}

- (void)cancelAllRequests {
    [self.session getTasksWithCompletionHandler:^(NSArray *dataTasks,
                                                   NSArray *uploadTasks,
                                                   NSArray *downloadTasks) {
        for (NSURLSessionTask *task in dataTasks) { [task cancel]; }
        for (NSURLSessionTask *task in uploadTasks) { [task cancel]; }
        for (NSURLSessionTask *task in downloadTasks) { [task cancel]; }
    }];
}

@end

// การใช้งาน NetworkLayer
NetworkRequest *request = [[NetworkRequest alloc] init];
request.urlString = @"https://api.example.com/users/1";
request.method = HTTPMethodGET;

[[NetworkLayer shared] executeRequest:request 
                           completion:^(id response, NSHTTPURLResponse *httpResponse, NSError *error) {
    if (error) {
        NSLog(@"Error: %@", error.localizedDescription);
    } else {
        NSLog(@"User: %@", response[@"name"]);
    }
}];
```

---

## 54.14 แบบฝึกหัด

### แบบฝึกหัดที่ 1: File Downloader

สร้าง `FileDownloader` ที่รองรับ:
- ดาวน์โหลดพร้อมกันหลายไฟล์
- หยุด/ดำเนินการต่อ Download ได้
- แสดง Progress รวมของทุกไฟล์
- Queue สำหรับจัดลำดับ

### แบบฝึกหัดที่ 2: Photo Uploader

สร้าง `PhotoUploader` ที่:
- รับ Array ของ UIImage
- Compress แต่ละภาพ
- Upload พร้อมกัน (max 3 concurrent)
- แสดง Progress แต่ละภาพ
- Retry เมื่อ Upload ล้มเหลว

### แบบฝึกหัดที่ 3: Background Download Manager

สร้าง App ที่:
- ให้ผู้ใช้ระบุ URL ที่ต้องการดาวน์โหลด
- ดาวน์โหลดใน Background
- แสดง Notification เมื่อเสร็จ
- รองรับ App Restart ระหว่างดาวน์โหลด

---

## สรุปบทที่ 54

ในบทนี้เราได้เรียนรู้:

1. **NSURLSessionConfiguration** - ประเภทต่าง ๆ และ Properties ทั้งหมด
2. **Data Tasks** - สำหรับ HTTP Request/Response ทั่วไป
3. **Upload Tasks** - สำหรับส่งไฟล์ขึ้น Server รวมถึง Multipart
4. **Download Tasks** - สำหรับดาวน์โหลดไฟล์พร้อม Resume Support
5. **Stream Tasks** - สำหรับ TCP Streams
6. **Background Sessions** - ทำงานต่อแม้ App อยู่ใน Background
7. **Delegates** - ควบคุม Authentication, Redirects, Progress
8. **Cookie Handling** - จัดการ Cookies
9. **Complete Networking Layer** - การสร้าง Layer ที่พร้อมใช้งานจริง

---

*ต่อไป: ตอนที่ 55 - JSON Parsing*
