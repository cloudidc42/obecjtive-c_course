# ตอนที่ 53: พื้นฐาน Networking ใน Objective-C

## บทนำ

การพัฒนาแอปพลิเคชันสมัยใหม่แทบทุกตัวจำเป็นต้องสื่อสารกับอินเทอร์เน็ต ไม่ว่าจะเป็นการดึงข้อมูลจาก API, อัปโหลดรูปภาพ, หรือส่งข้อความแบบเรียลไทม์ ใน Objective-C เราใช้ Framework และ API หลายอย่างในการจัดการกับ Networking โดยมีตัวหลักคือ **NSURLSession**

ในบทนี้เราจะเรียนรู้:
- พื้นฐาน HTTP Protocol
- โครงสร้างของ NSURLSession
- การตั้งค่า SSL/TLS และ App Transport Security
- การตรวจสอบสถานะเครือข่าย (Reachability)
- การจัดการ Cache

---

## 53.1 พื้นฐาน HTTP Protocol

### HTTP คืออะไร?

HTTP (HyperText Transfer Protocol) เป็น Protocol ที่ใช้สื่อสารระหว่าง Client และ Server บนอินเทอร์เน็ต การสื่อสารจะอยู่ในรูปแบบ Request/Response โดย Client ส่ง Request ไปยัง Server และ Server ตอบกลับด้วย Response

### HTTP Methods หลัก

| Method | การใช้งาน | ตัวอย่าง |
|--------|-----------|---------|
| GET | ดึงข้อมูล | ดึงรายการสินค้า |
| POST | สร้างข้อมูลใหม่ | สร้างบัญชีผู้ใช้ |
| PUT | อัปเดตข้อมูลทั้งหมด | แก้ไขข้อมูลผู้ใช้ |
| PATCH | อัปเดตข้อมูลบางส่วน | เปลี่ยนเฉพาะชื่อ |
| DELETE | ลบข้อมูล | ลบบัญชีผู้ใช้ |

### ตัวอย่าง HTTP Request ด้วย NSURLRequest

```objc
// GET Request - ง่ายที่สุด
NSURL *url = [NSURL URLWithString:@"https://api.example.com/users"];
NSURLRequest *request = [NSURLRequest requestWithURL:url];

// POST Request - ส่งข้อมูล
NSURL *postURL = [NSURL URLWithString:@"https://api.example.com/users"];
NSMutableURLRequest *mutableRequest = [NSMutableURLRequest requestWithURL:postURL];
[mutableRequest setHTTPMethod:@"POST"];

// กำหนด Headers
[mutableRequest setValue:@"application/json" forHTTPHeaderField:@"Content-Type"];
[mutableRequest setValue:@"application/json" forHTTPHeaderField:@"Accept"];
[mutableRequest setValue:@"Bearer mytoken123" forHTTPHeaderField:@"Authorization"];

// กำหนด Body
NSDictionary *bodyData = @{
    @"name": @"John Doe",
    @"email": @"john@example.com",
    @"age": @25
};
NSError *jsonError;
NSData *jsonData = [NSJSONSerialization dataWithJSONObject:bodyData
                                                   options:NSJSONWritingPrettyPrinted
                                                     error:&jsonError];
if (!jsonError) {
    [mutableRequest setHTTPBody:jsonData];
}

// PUT Request - อัปเดตทั้งหมด
NSURL *putURL = [NSURL URLWithString:@"https://api.example.com/users/123"];
NSMutableURLRequest *putRequest = [NSMutableURLRequest requestWithURL:putURL];
[putRequest setHTTPMethod:@"PUT"];
[putRequest setValue:@"application/json" forHTTPHeaderField:@"Content-Type"];

// DELETE Request - ลบข้อมูล
NSURL *deleteURL = [NSURL URLWithString:@"https://api.example.com/users/123"];
NSMutableURLRequest *deleteRequest = [NSMutableURLRequest requestWithURL:deleteURL];
[deleteRequest setHTTPMethod:@"DELETE"];
```

---

## 53.2 Request Headers และ Body

### HTTP Headers คืออะไร?

Headers เป็นข้อมูล Metadata ที่ส่งไปพร้อมกับ Request หรือ Response เพื่อให้ข้อมูลเพิ่มเติมเกี่ยวกับการสื่อสาร

### Headers ที่ใช้บ่อย

```objc
NSMutableURLRequest *request = [NSMutableURLRequest requestWithURL:url];

// Content-Type - บอกประเภทข้อมูลที่ส่ง
[request setValue:@"application/json" forHTTPHeaderField:@"Content-Type"];

// Accept - บอกประเภทข้อมูลที่ยอมรับ
[request setValue:@"application/json" forHTTPHeaderField:@"Accept"];

// Authorization - ข้อมูลยืนยันตัวตน
[request setValue:@"Bearer eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9..." 
           forHTTPHeaderField:@"Authorization"];

// User-Agent - บอกว่าเป็น App อะไร
[request setValue:@"MyApp/1.0 (iOS 17.0)" forHTTPHeaderField:@"User-Agent"];

// Cache-Control - ควบคุมการ Cache
[request setValue:@"no-cache" forHTTPHeaderField:@"Cache-Control"];

// Custom Headers
[request setValue:@"my-api-key-here" forHTTPHeaderField:@"X-API-Key"];
[request setValue:@"th-TH" forHTTPHeaderField:@"Accept-Language"];
```

### Request Body

```objc
// JSON Body
NSDictionary *parameters = @{
    @"username": @"john_doe",
    @"password": @"secret123",
    @"remember_me": @YES
};

NSError *error;
NSData *jsonBody = [NSJSONSerialization dataWithJSONObject:parameters 
                                                   options:0 
                                                     error:&error];
if (!error) {
    [request setHTTPBody:jsonBody];
    NSLog(@"Body size: %lu bytes", (unsigned long)jsonBody.length);
}

// Form URL Encoded Body
NSString *formBody = @"username=john_doe&password=secret123&remember_me=true";
NSData *formData = [formBody dataUsingEncoding:NSUTF8StringEncoding];
[request setHTTPBody:formData];
[request setValue:@"application/x-www-form-urlencoded" 
           forHTTPHeaderField:@"Content-Type"];

// Binary Data Body (เช่น รูปภาพ)
UIImage *image = [UIImage imageNamed:@"photo.jpg"];
NSData *imageData = UIImageJPEGRepresentation(image, 0.8);
[request setHTTPBody:imageData];
[request setValue:@"image/jpeg" forHTTPHeaderField:@"Content-Type"];
[request setValue:[NSString stringWithFormat:@"%lu", (unsigned long)imageData.length]
           forHTTPHeaderField:@"Content-Length"];
```

---

## 53.3 HTTP Response Codes

### รหัสสถานะ HTTP (Status Codes)

รหัสสถานะแบ่งออกเป็น 5 กลุ่มหลัก:

**1xx - Informational (ข้อมูลเชิงแจ้ง)**
- 100 Continue
- 101 Switching Protocols

**2xx - Success (สำเร็จ)**
- 200 OK - ทำงานสำเร็จ
- 201 Created - สร้างข้อมูลสำเร็จ
- 204 No Content - สำเร็จแต่ไม่มีข้อมูลส่งกลับ

**3xx - Redirection (เปลี่ยนเส้นทาง)**
- 301 Moved Permanently - ย้ายไปถาวร
- 302 Found - เปลี่ยนชั่วคราว
- 304 Not Modified - ข้อมูลไม่เปลี่ยนแปลง

**4xx - Client Error (ข้อผิดพลาดฝั่ง Client)**
- 400 Bad Request - Request ไม่ถูกต้อง
- 401 Unauthorized - ไม่ได้ยืนยันตัวตน
- 403 Forbidden - ไม่มีสิทธิ์
- 404 Not Found - ไม่พบข้อมูล
- 409 Conflict - ข้อมูลขัดแย้ง
- 422 Unprocessable Entity - ข้อมูลไม่ถูกต้อง
- 429 Too Many Requests - Request มากเกินไป

**5xx - Server Error (ข้อผิดพลาดฝั่ง Server)**
- 500 Internal Server Error - ข้อผิดพลาดใน Server
- 502 Bad Gateway - Gateway ล้มเหลว
- 503 Service Unavailable - Service ไม่พร้อม
- 504 Gateway Timeout - หมดเวลา

### การตรวจสอบ Status Code

```objc
// ตรวจสอบ Response
NSHTTPURLResponse *httpResponse = (NSHTTPURLResponse *)response;
NSInteger statusCode = httpResponse.statusCode;

// จัดการ Status Code
switch (statusCode) {
    case 200:
    case 201:
        NSLog(@"Success! Status: %ld", (long)statusCode);
        // ประมวลผลข้อมูล
        break;
        
    case 204:
        NSLog(@"No Content - Operation successful");
        break;
        
    case 400:
        NSLog(@"Bad Request - ตรวจสอบข้อมูลที่ส่งไป");
        break;
        
    case 401:
        NSLog(@"Unauthorized - ต้องเข้าสู่ระบบก่อน");
        // แสดงหน้า Login
        break;
        
    case 403:
        NSLog(@"Forbidden - ไม่มีสิทธิ์ใช้งาน");
        break;
        
    case 404:
        NSLog(@"Not Found - ไม่พบข้อมูล");
        break;
        
    case 429:
        NSLog(@"Too Many Requests - รอสักครู่แล้วลองใหม่");
        // Retry หลังจาก delay
        break;
        
    case 500 ... 599:
        NSLog(@"Server Error: %ld", (long)statusCode);
        // แสดงข้อความ Error ให้ผู้ใช้
        break;
        
    default:
        NSLog(@"Unexpected status code: %ld", (long)statusCode);
        break;
}

// ดู Response Headers
NSDictionary *headers = httpResponse.allHeaderFields;
NSString *contentType = headers[@"Content-Type"];
NSString *rateLimit = headers[@"X-RateLimit-Remaining"];
NSLog(@"Content-Type: %@", contentType);
NSLog(@"Rate Limit Remaining: %@", rateLimit);
```

---

## 53.4 NSURLSession Architecture

### NSURLSession คืออะไร?

NSURLSession เป็น API หลักสำหรับการทำ Networking ใน iOS/macOS ตั้งแต่ iOS 7 เป็นต้นมา มีความสามารถสำคัญคือ:

1. **รองรับ Background Sessions** - ดาวน์โหลด/อัปโหลดได้แม้ App อยู่ใน Background
2. **รองรับ HTTP/2** - เร็วและมีประสิทธิภาพมากขึ้น
3. **Authentication Challenges** - จัดการการยืนยันตัวตนได้หลายรูปแบบ
4. **Cookie Handling** - จัดการ Cookie อัตโนมัติ
5. **Cache Control** - ควบคุมการ Cache ได้ละเอียด

### โครงสร้างของ NSURLSession

```
NSURLSession
├── NSURLSessionConfiguration (การตั้งค่า)
├── NSURLSessionTask (งานแต่ละชิ้น)
│   ├── NSURLSessionDataTask (รับข้อมูล)
│   ├── NSURLSessionUploadTask (อัปโหลด)
│   ├── NSURLSessionDownloadTask (ดาวน์โหลด)
│   └── NSURLSessionStreamTask (Stream)
└── NSURLSessionDelegate (การจัดการ Events)
```

### การสร้างและใช้งาน NSURLSession

```objc
// Simple Session
NSURLSession *session = [NSURLSession sharedSession];

NSURL *url = [NSURL URLWithString:@"https://jsonplaceholder.typicode.com/posts/1"];
NSURLSessionDataTask *task = [session dataTaskWithURL:url 
                                    completionHandler:^(NSData *data, 
                                                        NSURLResponse *response, 
                                                        NSError *error) {
    if (error) {
        NSLog(@"Error: %@", error.localizedDescription);
        return;
    }
    
    NSHTTPURLResponse *httpResponse = (NSHTTPURLResponse *)response;
    NSLog(@"Status Code: %ld", (long)httpResponse.statusCode);
    
    if (data) {
        NSError *jsonError;
        NSDictionary *json = [NSJSONSerialization JSONObjectWithData:data 
                                                            options:0 
                                                              error:&jsonError];
        if (!jsonError) {
            NSLog(@"Response: %@", json);
        }
    }
}];
[task resume];

// Custom Session with Configuration
NSURLSessionConfiguration *config = [NSURLSessionConfiguration defaultSessionConfiguration];
config.timeoutIntervalForRequest = 30.0;
config.timeoutIntervalForResource = 60.0;
config.HTTPMaximumConnectionsPerHost = 5;

NSURLSession *customSession = [NSURLSession sessionWithConfiguration:config];
```

---

## 53.5 Background Sessions

### เมื่อไหร่ควรใช้ Background Sessions?

Background Session เหมาะสำหรับ:
- การดาวน์โหลดไฟล์ขนาดใหญ่
- การอัปโหลดรูปภาพ/วิดีโอ
- งานที่ต้องทำต่อแม้ผู้ใช้ออกจาก App

```objc
// สร้าง Background Session
NSString *identifier = @"com.myapp.background-session";
NSURLSessionConfiguration *backgroundConfig = 
    [NSURLSessionConfiguration backgroundSessionConfigurationWithIdentifier:identifier];

backgroundConfig.discretionary = YES;  // ให้ระบบเลือกเวลาที่เหมาะสม
backgroundConfig.sessionSendsLaunchEvents = YES;  // ปลุก App เมื่อเสร็จ

NSURLSession *backgroundSession = [NSURLSession sessionWithConfiguration:backgroundConfig
                                                               delegate:self
                                                          delegateQueue:nil];

// Background Download Task
NSURL *downloadURL = [NSURL URLWithString:@"https://example.com/large-file.zip"];
NSURLSessionDownloadTask *downloadTask = [backgroundSession downloadTaskWithURL:downloadURL];
[downloadTask resume];

// ใน AppDelegate - จัดการเมื่อ Background Session เสร็จ
- (void)application:(UIApplication *)application 
handleEventsForBackgroundURLSession:(NSString *)identifier
  completionHandler:(void (^)(void))completionHandler {
    // เก็บ completionHandler ไว้ใช้ภายหลัง
    self.backgroundCompletionHandler = completionHandler;
    NSLog(@"Background session %@ finished", identifier);
}

// ใน NSURLSessionDelegate
- (void)URLSessionDidFinishEventsForBackgroundURLSession:(NSURLSession *)session {
    dispatch_async(dispatch_get_main_queue(), ^{
        AppDelegate *appDelegate = (AppDelegate *)[[UIApplication sharedApplication] delegate];
        if (appDelegate.backgroundCompletionHandler) {
            appDelegate.backgroundCompletionHandler();
            appDelegate.backgroundCompletionHandler = nil;
        }
    });
}
```

---

## 53.6 URLSession Configuration Types

### 3 ประเภทหลักของ NSURLSessionConfiguration

```objc
// 1. Default Configuration
// - ใช้ Disk Cache และ Cookie Storage
// - เหมาะสำหรับการใช้งานทั่วไป
NSURLSessionConfiguration *defaultConfig = 
    [NSURLSessionConfiguration defaultSessionConfiguration];
defaultConfig.timeoutIntervalForRequest = 30.0;
defaultConfig.allowsCellularAccess = YES;
defaultConfig.HTTPShouldSetCookies = YES;
defaultConfig.HTTPCookieAcceptPolicy = NSHTTPCookieAcceptPolicyAlways;

// 2. Ephemeral Configuration
// - ไม่บันทึก Cache, Cookies, หรือ Credentials ลง Disk
// - เหมาะสำหรับ Private/Incognito Mode
// - เมื่อปิด Session ข้อมูลทั้งหมดหายไป
NSURLSessionConfiguration *ephemeralConfig = 
    [NSURLSessionConfiguration ephemeralSessionConfiguration];
ephemeralConfig.timeoutIntervalForRequest = 15.0;

// 3. Background Configuration  
// - ทำงานต่อได้แม้ App อยู่ใน Background
// - ต้องใช้ Identifier เฉพาะ
NSURLSessionConfiguration *backgroundConfig = 
    [NSURLSessionConfiguration backgroundSessionConfigurationWithIdentifier:
        @"com.myapp.background"];
backgroundConfig.discretionary = YES;
backgroundConfig.sessionSendsLaunchEvents = YES;

// การเปรียบเทียบ
NSLog(@"Default uses disk cache: YES");
NSLog(@"Ephemeral uses disk cache: NO");
NSLog(@"Background supports background tasks: YES");
```

### การตั้งค่า Configuration เพิ่มเติม

```objc
NSURLSessionConfiguration *config = [NSURLSessionConfiguration defaultSessionConfiguration];

// Timeout Settings
config.timeoutIntervalForRequest = 30.0;   // timeout สำหรับแต่ละ Request
config.timeoutIntervalForResource = 300.0; // timeout สำหรับ Resource ทั้งหมด

// Connectivity
config.allowsCellularAccess = YES;
config.allowsExpensiveNetworkAccess = YES;      // iOS 13+
config.allowsConstrainedNetworkAccess = YES;   // iOS 13+
config.waitsForConnectivity = YES;  // รอจนกว่าจะมีการเชื่อมต่อ

// HTTP Settings
config.HTTPMaximumConnectionsPerHost = 6;
config.HTTPShouldUsePipelining = NO;  // HTTP/1.1 Pipelining
config.HTTPShouldSetCookies = YES;

// Cache Policy
config.requestCachePolicy = NSURLRequestUseProtocolCachePolicy;

// TLS Settings
config.TLSMinimumSupportedProtocolVersion = tls_protocol_version_TLSv12;

// Custom Headers ที่ใส่ทุก Request
config.HTTPAdditionalHeaders = @{
    @"User-Agent": @"MyApp/1.0",
    @"Accept": @"application/json",
    @"X-App-Version": @"1.0.0"
};
```

---

## 53.7 NSURLConnection (Deprecated - ประวัติศาสตร์)

### NSURLConnection คืออะไร?

NSURLConnection เป็น API เก่าที่ใช้ก่อน iOS 7 ตอนนี้ถูก Deprecated แล้ว แต่ควรรู้จักไว้เพื่อ:
1. Maintain โค้ดเก่า
2. เข้าใจวิวัฒนาการของ iOS Networking

```objc
// ตัวอย่าง NSURLConnection แบบเก่า (DEPRECATED - อย่าใช้ในโค้ดใหม่!)
/*
NSURL *url = [NSURL URLWithString:@"https://example.com/api/data"];
NSURLRequest *request = [NSURLRequest requestWithURL:url];

// Async (เก่า)
[NSURLConnection sendAsynchronousRequest:request 
                                   queue:[NSOperationQueue mainQueue]
                       completionHandler:^(NSURLResponse *response, 
                                          NSData *data, 
                                          NSError *error) {
    if (error) {
        NSLog(@"Error: %@", error);
        return;
    }
    NSLog(@"Data received: %lu bytes", data.length);
}];
*/

// แบบใหม่ที่ควรใช้แทน - NSURLSession
NSURLSession *session = [NSURLSession sharedSession];
NSURL *url = [NSURL URLWithString:@"https://example.com/api/data"];
NSURLSessionDataTask *task = [session dataTaskWithURL:url
                                    completionHandler:^(NSData *data, 
                                                        NSURLResponse *response, 
                                                        NSError *error) {
    if (error) {
        NSLog(@"Error: %@", error);
        return;
    }
    NSLog(@"Data received: %lu bytes", (unsigned long)data.length);
}];
[task resume];
```

### Timeline การเปลี่ยนแปลง

```
2003 - Mac OS X 10.3: NSURLConnection เปิดตัว
2007 - iPhone OS 1.0: รองรับ NSURLConnection
2013 - iOS 7: NSURLSession เปิดตัวแทน
2015 - iOS 9: NSURLConnection ถูก Deprecated
2023 - ปัจจุบัน: ใช้ NSURLSession เท่านั้น
```

---

## 53.8 SSL/TLS และ App Transport Security

### SSL/TLS คืออะไร?

SSL (Secure Sockets Layer) และ TLS (Transport Layer Security) เป็น Protocol ที่เข้ารหัสข้อมูลในการส่ง เพื่อความปลอดภัย iOS บังคับให้ใช้ HTTPS (HTTP + TLS) สำหรับทุก Network Connection

### App Transport Security (ATS)

ATS เป็นฟีเจอร์ความปลอดภัยที่ Apple เพิ่มมาใน iOS 9 บังคับให้:
- ใช้ HTTPS สำหรับทุก HTTP Connection
- ใช้ TLS 1.2 ขึ้นไป
- ใช้ Perfect Forward Secrecy

```xml
<!-- Info.plist - การตั้งค่า ATS -->
<key>NSAppTransportSecurity</key>
<dict>
    <!-- ปิด ATS ทั้งหมด (ไม่แนะนำ!) -->
    <key>NSAllowsArbitraryLoads</key>
    <false/>
    
    <!-- อนุญาต HTTP เฉพาะบาง Domain -->
    <key>NSExceptionDomains</key>
    <dict>
        <key>api.example.com</key>
        <dict>
            <!-- อนุญาต HTTP ที่ไม่ปลอดภัย (ชั่วคราว สำหรับ Test) -->
            <key>NSExceptionAllowsInsecureHTTPLoads</key>
            <true/>
            <!-- ลด TLS Version ที่ต้องการ -->
            <key>NSExceptionMinimumTLSVersion</key>
            <string>TLSv1.0</string>
        </dict>
    </dict>
    
    <!-- อนุญาต Local Networking (iOS 14+) -->
    <key>NSAllowsLocalNetworking</key>
    <true/>
</dict>
```

### การจัดการ SSL Certificate Pinning

Certificate Pinning เป็นเทคนิคความปลอดภัยที่ตรวจสอบ Certificate ของ Server ให้ตรงกับที่กำหนดไว้

```objc
// SSL Pinning ด้วย NSURLSessionDelegate
@interface NetworkManager : NSObject <NSURLSessionDelegate>
@property (nonatomic, strong) NSURLSession *session;
@end

@implementation NetworkManager

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

// Delegate Method สำหรับ SSL Pinning
- (void)URLSession:(NSURLSession *)session 
didReceiveChallenge:(NSURLAuthenticationChallenge *)challenge
 completionHandler:(void (^)(NSURLSessionAuthChallengeDisposition, 
                             NSURLCredential *))completionHandler {
    
    if ([challenge.protectionSpace.authenticationMethod 
         isEqualToString:NSURLAuthenticationMethodServerTrust]) {
        
        SecTrustRef serverTrust = challenge.protectionSpace.serverTrust;
        SecCertificateRef certificate = SecTrustGetCertificateAtIndex(serverTrust, 0);
        
        // เปรียบเทียบกับ Certificate ที่ embed ไว้
        NSData *serverCertData = (__bridge NSData *)SecCertificateCopyData(certificate);
        
        // โหลด Certificate จาก Bundle
        NSString *certPath = [[NSBundle mainBundle] pathForResource:@"server" ofType:@"cer"];
        NSData *pinnedCertData = [NSData dataWithContentsOfFile:certPath];
        
        if ([serverCertData isEqualToData:pinnedCertData]) {
            // Certificate ตรงกัน - อนุญาต
            NSURLCredential *credential = [NSURLCredential credentialForTrust:serverTrust];
            completionHandler(NSURLSessionAuthChallengeUseCredential, credential);
        } else {
            // Certificate ไม่ตรง - ปฏิเสธ
            completionHandler(NSURLSessionAuthChallengeCancelAuthenticationChallenge, nil);
        }
    } else {
        completionHandler(NSURLSessionAuthChallengePerformDefaultHandling, nil);
    }
}

@end
```

---

## 53.9 Reachability - Network Framework

### ตรวจสอบสถานะการเชื่อมต่อ

ตั้งแต่ iOS 12 มี Network Framework ใหม่ที่ดีกว่า SystemConfiguration

```objc
#import <Network/Network.h>

// NetworkMonitor.h
@interface NetworkMonitor : NSObject

@property (nonatomic, readonly) BOOL isReachable;
@property (nonatomic, readonly) BOOL isOnWifi;
@property (nonatomic, readonly) BOOL isOnCellular;

+ (instancetype)shared;
- (void)startMonitoring;
- (void)stopMonitoring;

@end

// NetworkMonitor.m
@interface NetworkMonitor ()
@property (nonatomic, strong) id pathMonitor; // nw_path_monitor_t
@property (nonatomic, strong) dispatch_queue_t monitorQueue;
@property (nonatomic, assign) BOOL isReachable;
@property (nonatomic, assign) BOOL isOnWifi;
@property (nonatomic, assign) BOOL isOnCellular;
@end

@implementation NetworkMonitor

+ (instancetype)shared {
    static NetworkMonitor *instance = nil;
    static dispatch_once_t onceToken;
    dispatch_once(&onceToken, ^{
        instance = [[NetworkMonitor alloc] init];
    });
    return instance;
}

- (instancetype)init {
    self = [super init];
    if (self) {
        self.monitorQueue = dispatch_queue_create("NetworkMonitorQueue", 
                                                   DISPATCH_QUEUE_SERIAL);
        _isReachable = NO;
        _isOnWifi = NO;
        _isOnCellular = NO;
    }
    return self;
}

- (void)startMonitoring {
    // ใช้ Network framework ผ่าน C API
    nw_path_monitor_t monitor = nw_path_monitor_create();
    
    nw_path_monitor_set_update_handler(monitor, ^(nw_path_t path) {
        nw_path_status_t status = nw_path_get_status(path);
        
        self->_isReachable = (status == nw_path_status_satisfied);
        self->_isOnWifi = nw_path_uses_interface_type(path, nw_interface_type_wifi);
        self->_isOnCellular = nw_path_uses_interface_type(path, nw_interface_type_cellular);
        
        NSLog(@"Network status changed:");
        NSLog(@"  Reachable: %@", self->_isReachable ? @"YES" : @"NO");
        NSLog(@"  WiFi: %@", self->_isOnWifi ? @"YES" : @"NO");
        NSLog(@"  Cellular: %@", self->_isOnCellular ? @"YES" : @"NO");
        
        // แจ้งเตือน
        dispatch_async(dispatch_get_main_queue(), ^{
            [[NSNotificationCenter defaultCenter] 
                postNotificationName:@"NetworkStatusChanged" 
                object:nil 
                userInfo:@{
                    @"isReachable": @(self->_isReachable),
                    @"isOnWifi": @(self->_isOnWifi),
                    @"isOnCellular": @(self->_isOnCellular)
                }];
        });
    });
    
    nw_path_monitor_set_queue(monitor, self.monitorQueue);
    nw_path_monitor_start(monitor);
}

@end

// การใช้งาน
[[NetworkMonitor shared] startMonitoring];

if ([NetworkMonitor shared].isReachable) {
    NSLog(@"มีการเชื่อมต่ออินเทอร์เน็ต");
    if ([NetworkMonitor shared].isOnWifi) {
        NSLog(@"กำลังใช้ WiFi");
    } else if ([NetworkMonitor shared].isOnCellular) {
        NSLog(@"กำลังใช้ Cellular");
    }
} else {
    NSLog(@"ไม่มีการเชื่อมต่ออินเทอร์เน็ต");
}
```

---

## 53.10 HTTP Caching

### Cache Policy ใน NSURLRequest

```objc
// Cache Policies
NSURLRequestCachePolicy policies[] = {
    NSURLRequestUseProtocolCachePolicy,           // ค่าเริ่มต้น - ตามที่ Server กำหนด
    NSURLRequestReloadIgnoringLocalCacheData,     // ไม่ใช้ Local Cache
    NSURLRequestReloadIgnoringCacheData,          // ไม่ใช้ Cache เลย
    NSURLRequestReturnCacheDataElseLoad,          // ใช้ Cache ถ้ามี ไม่งั้น Load ใหม่
    NSURLRequestReturnCacheDataDontLoad,          // ใช้ Cache เท่านั้น
    NSURLRequestReloadRevalidatingCacheData       // ตรวจสอบ Cache กับ Server ก่อน
};

// ตัวอย่างการใช้งาน
NSURL *url = [NSURL URLWithString:@"https://api.example.com/data"];
NSURLRequest *cachedRequest = [NSURLRequest requestWithURL:url
                                               cachePolicy:NSURLRequestReturnCacheDataElseLoad
                                           timeoutInterval:30.0];

// Force Refresh - ไม่ใช้ Cache
NSURLRequest *freshRequest = [NSURLRequest requestWithURL:url
                                              cachePolicy:NSURLRequestReloadIgnoringLocalCacheData
                                          timeoutInterval:30.0];
```

### การจัดการ Cache ด้วย NSURLCache

```objc
// ตั้งค่า Cache ขนาดใหญ่ขึ้น
NSURLCache *cache = [[NSURLCache alloc] initWithMemoryCapacity:50 * 1024 * 1024    // 50 MB Memory
                                                  diskCapacity:200 * 1024 * 1024   // 200 MB Disk
                                                  directoryURL:nil];  // iOS 13+
[NSURLCache setSharedURLCache:cache];

// ล้าง Cache
[[NSURLCache sharedURLCache] removeAllCachedResponses];

// ล้าง Cache สำหรับ URL เฉพาะ
NSURL *url = [NSURL URLWithString:@"https://api.example.com/users"];
NSURLRequest *request = [NSURLRequest requestWithURL:url];
[[NSURLCache sharedURLCache] removeCachedResponseForRequest:request];

// ดูขนาด Cache ปัจจุบัน
NSURLCache *currentCache = [NSURLCache sharedURLCache];
NSLog(@"Memory Cache: %lu / %lu bytes", 
      (unsigned long)currentCache.currentMemoryUsage,
      (unsigned long)currentCache.memoryCapacity);
NSLog(@"Disk Cache: %lu / %lu bytes",
      (unsigned long)currentCache.currentDiskUsage,
      (unsigned long)currentCache.diskCapacity);

// บันทึก Response ลง Cache ด้วยตัวเอง
NSCachedURLResponse *cachedResponse = [[NSCachedURLResponse alloc] 
    initWithResponse:response
                data:data
            userInfo:@{@"cachedAt": [NSDate date]}
       storagePolicy:NSURLCacheStorageAllowed];
[[NSURLCache sharedURLCache] storeCachedResponse:cachedResponse 
                                      forRequest:request];
```

### HTTP Cache Headers

```objc
// จัดการ HTTP Cache Headers
- (void)handleCacheHeaders:(NSHTTPURLResponse *)response {
    NSDictionary *headers = response.allHeaderFields;
    
    // Cache-Control
    NSString *cacheControl = headers[@"Cache-Control"];
    NSLog(@"Cache-Control: %@", cacheControl);
    // ตัวอย่าง: "max-age=3600, must-revalidate"
    
    // ETag - สำหรับ Conditional Requests
    NSString *etag = headers[@"ETag"];
    NSLog(@"ETag: %@", etag);
    // เก็บไว้ใช้ใน If-None-Match header ครั้งหน้า
    
    // Last-Modified
    NSString *lastModified = headers[@"Last-Modified"];
    NSLog(@"Last-Modified: %@", lastModified);
    // เก็บไว้ใช้ใน If-Modified-Since header ครั้งหน้า
    
    // Expires
    NSString *expires = headers[@"Expires"];
    NSLog(@"Expires: %@", expires);
}

// Conditional Request ด้วย ETag
NSMutableURLRequest *conditionalRequest = [NSMutableURLRequest requestWithURL:url];
NSString *savedETag = [[NSUserDefaults standardUserDefaults] stringForKey:@"lastETag"];
if (savedETag) {
    [conditionalRequest setValue:savedETag forHTTPHeaderField:@"If-None-Match"];
}

NSURLSessionDataTask *task = [session dataTaskWithRequest:conditionalRequest
                                        completionHandler:^(NSData *data, 
                                                            NSURLResponse *response, 
                                                            NSError *error) {
    NSHTTPURLResponse *httpResponse = (NSHTTPURLResponse *)response;
    
    if (httpResponse.statusCode == 304) {
        // Not Modified - ใช้ข้อมูลเก่า
        NSLog(@"Data unchanged, using cache");
    } else if (httpResponse.statusCode == 200) {
        // ข้อมูลใหม่
        NSString *newETag = httpResponse.allHeaderFields[@"ETag"];
        [[NSUserDefaults standardUserDefaults] setObject:newETag forKey:@"lastETag"];
        // ประมวลผลข้อมูลใหม่
    }
}];
[task resume];
```

---

## 53.11 แบบฝึกหัด

### แบบฝึกหัดที่ 1: HTTP Request พื้นฐาน

สร้าง Class `SimpleHTTPClient` ที่มี Method ต่อไปนี้:

```objc
@interface SimpleHTTPClient : NSObject

// GET Request
- (void)getFromURL:(NSString *)urlString 
        completion:(void(^)(NSDictionary *data, NSError *error))completion;

// POST Request
- (void)postToURL:(NSString *)urlString 
             body:(NSDictionary *)body 
       completion:(void(^)(NSDictionary *data, NSError *error))completion;

// PUT Request
- (void)putToURL:(NSString *)urlString 
            body:(NSDictionary *)body 
      completion:(void(^)(NSDictionary *data, NSError *error))completion;

// DELETE Request
- (void)deleteFromURL:(NSString *)urlString 
           completion:(void(^)(BOOL success, NSError *error))completion;

@end

@implementation SimpleHTTPClient

- (instancetype)init {
    self = [super init];
    if (self) {
        // TODO: ตั้งค่า NSURLSession
    }
    return self;
}

- (void)getFromURL:(NSString *)urlString 
        completion:(void(^)(NSDictionary *data, NSError *error))completion {
    // TODO: ใช้ NSURLSession ทำ GET Request
}

// ... implement อื่น ๆ

@end
```

**ทดสอบกับ API:**
```objc
SimpleHTTPClient *client = [[SimpleHTTPClient alloc] init];

// ทดสอบ GET
[client getFromURL:@"https://jsonplaceholder.typicode.com/posts/1"
        completion:^(NSDictionary *data, NSError *error) {
    if (data) {
        NSLog(@"Post title: %@", data[@"title"]);
    }
}];

// ทดสอบ POST
[client postToURL:@"https://jsonplaceholder.typicode.com/posts"
             body:@{@"title": @"Test Post", @"body": @"Content here", @"userId": @1}
       completion:^(NSDictionary *data, NSError *error) {
    if (data) {
        NSLog(@"Created post ID: %@", data[@"id"]);
    }
}];
```

### แบบฝึกหัดที่ 2: Network Reachability

สร้าง `NetworkStatusView` ที่แสดงสถานะการเชื่อมต่อแบบ Real-time:

```objc
@interface NetworkStatusView : UIView

- (void)startMonitoring;
- (void)stopMonitoring;

@end
```

เมื่อเครือข่ายเปลี่ยน ให้แสดง:
- สีเขียว + "Connected via WiFi"
- สีน้ำเงิน + "Connected via Cellular"
- สีแดง + "No Connection"

### แบบฝึกหัดที่ 3: Cache Manager

สร้าง `CacheManager` ที่:
1. Cache Response สำหรับ URL ที่กำหนด
2. ตรวจสอบว่า Cache หมดอายุหรือไม่
3. ล้าง Cache เมื่อต้องการ

```objc
@interface CacheManager : NSObject

+ (instancetype)shared;
- (NSData *)cachedDataForURL:(NSURL *)url;
- (void)cacheData:(NSData *)data forURL:(NSURL *)url duration:(NSTimeInterval)duration;
- (BOOL)isCacheValidForURL:(NSURL *)url;
- (void)clearCache;

@end
```

### แบบฝึกหัดที่ 4: Request Logger

สร้าง `RequestLogger` ที่บันทึก:
- URL ที่ Request
- HTTP Method
- Status Code
- Duration (เวลาที่ใช้)
- Data Size

```objc
@interface RequestLog : NSObject
@property (nonatomic, copy) NSString *url;
@property (nonatomic, copy) NSString *method;
@property (nonatomic, assign) NSInteger statusCode;
@property (nonatomic, assign) NSTimeInterval duration;
@property (nonatomic, assign) NSUInteger dataSize;
@property (nonatomic, strong) NSDate *timestamp;
@end

@interface RequestLogger : NSObject
+ (instancetype)shared;
- (void)logRequest:(NSURLRequest *)request 
          response:(NSURLResponse *)response 
              data:(NSData *)data 
          duration:(NSTimeInterval)duration;
- (NSArray<RequestLog *> *)allLogs;
- (void)clearLogs;
@end
```

---

## สรุปบทที่ 53

ในบทนี้เราได้เรียนรู้:

1. **HTTP Basics** - Method ต่าง ๆ (GET, POST, PUT, DELETE) และ Status Codes
2. **NSURLSession** - API หลักสำหรับ Networking ใน iOS สมัยใหม่
3. **Session Configurations** - Default, Ephemeral, Background
4. **Background Sessions** - การทำงานใน Background
5. **SSL/TLS** - ความปลอดภัยด้านการสื่อสาร
6. **App Transport Security** - การบังคับ HTTPS
7. **Network Reachability** - การตรวจสอบสถานะเครือข่าย
8. **HTTP Caching** - การลดการใช้ Bandwidth

ในบทถัดไปเราจะเจาะลึก NSURLSession เพิ่มเติม รวมถึง Task Types, Delegates, และ Authentication

---

*ต่อไป: ตอนที่ 54 - NSURLSession Deep Dive*
