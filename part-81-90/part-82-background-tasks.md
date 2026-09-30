# Part 82: Background Tasks (งานที่ทำในพื้นหลัง)

## บทนำ

Background Tasks เป็นความสามารถที่สำคัญของ iOS ที่ช่วยให้แอปสามารถทำงานได้แม้ผู้ใช้จะกดออกจากแอปไปแล้ว iOS มีกลไกหลายอย่างสำหรับการทำงานใน background แต่ละแบบมีข้อจำกัดและวิธีใช้ที่แตกต่างกัน เพื่อรักษาแบตเตอรี่และ performance ของอุปกรณ์

---

## 1. Background Execution Modes (โหมดการทำงานพื้นหลัง)

iOS รองรับ background modes หลายประเภท ซึ่งต้องประกาศใน Info.plist

### 1.1 ประเภทของ Background Modes

| Mode | Key | ใช้สำหรับ |
|------|-----|---------|
| Audio | audio | เล่นเพลง/เสียงใน background |
| Location | location | ติดตาม GPS ต่อเนื่อง |
| VoIP | voip | แอปโทรศัพท์ |
| Background Fetch | fetch | ดึงข้อมูลเป็นระยะ |
| Remote Notifications | remote-notification | Push notifications |
| Background Processing | processing | งานหนักเป็นครั้งคราว |
| Bluetooth | bluetooth-central, bluetooth-peripheral | Bluetooth |
| External Accessory | external-accessory | Accessories |
| NFC | nfc | NFC tags |

### 1.2 การประกาศ Background Modes ใน Info.plist

```xml
<key>UIBackgroundModes</key>
<array>
    <string>audio</string>
    <string>fetch</string>
    <string>processing</string>
    <string>remote-notification</string>
</array>
```

---

## 2. Background Fetch (การดึงข้อมูลพื้นหลัง)

Background Fetch ช่วยให้แอปดึงข้อมูลใหม่เป็นระยะๆ แม้ว่าจะไม่ได้ใช้งานอยู่

### 2.1 เปิดใช้งาน Background Fetch

```objc
// AppDelegate.m
- (BOOL)application:(UIApplication *)application 
    didFinishLaunchingWithOptions:(NSDictionary *)launchOptions {
    
    // กำหนด interval สำหรับ background fetch
    // UIApplicationBackgroundFetchIntervalMinimum = ระบบกำหนดเอง
    [application setMinimumBackgroundFetchInterval:
     UIApplicationBackgroundFetchIntervalMinimum];
    
    return YES;
}

// เมธอดที่ถูกเรียกเมื่อถึงเวลา fetch
- (void)application:(UIApplication *)application 
    performFetchWithCompletionHandler:(void (^)(UIBackgroundFetchResult))completionHandler {
    
    NSLog(@"=== Background Fetch Started ===");
    NSDate *fetchStartTime = [NSDate date];
    
    // ดึงข้อมูลใหม่
    [[DataService shared] fetchLatestNewsWithCompletion:^(NSArray *newItems, NSError *error) {
        NSTimeInterval elapsed = [[NSDate date] timeIntervalSinceDate:fetchStartTime];
        NSLog(@"Fetch completed in %.2f seconds", elapsed);
        
        if (error) {
            NSLog(@"Fetch failed: %@", error.localizedDescription);
            completionHandler(UIBackgroundFetchResultFailed);
            return;
        }
        
        if (newItems.count > 0) {
            NSLog(@"Fetched %lu new items", (unsigned long)newItems.count);
            
            // บันทึกข้อมูลใหม่
            [[DataStore shared] saveItems:newItems];
            
            // อัพเดท badge
            dispatch_async(dispatch_get_main_queue(), ^{
                [UIApplication sharedApplication].applicationIconBadgeNumber = newItems.count;
            });
            
            completionHandler(UIBackgroundFetchResultNewData);
        } else {
            NSLog(@"No new data");
            completionHandler(UIBackgroundFetchResultNoData);
        }
    }];
    
    // ต้องเรียก completionHandler ภายใน 30 วินาที!
}
```

### 2.2 DataService Implementation

```objc
// DataService.h
@interface DataService : NSObject

+ (instancetype)shared;
- (void)fetchLatestNewsWithCompletion:(void (^)(NSArray *items, NSError *error))completion;

@end

// DataService.m
@implementation DataService

+ (instancetype)shared {
    static DataService *instance = nil;
    static dispatch_once_t onceToken;
    dispatch_once(&onceToken, ^{
        instance = [[self alloc] init];
    });
    return instance;
}

- (void)fetchLatestNewsWithCompletion:(void (^)(NSArray *items, NSError *error))completion {
    NSURL *url = [NSURL URLWithString:@"https://api.example.com/news/latest"];
    
    NSURLSessionDataTask *task = [[NSURLSession sharedSession] 
        dataTaskWithURL:url 
        completionHandler:^(NSData *data, NSURLResponse *response, NSError *error) {
        
        if (error) {
            completion(nil, error);
            return;
        }
        
        NSError *parseError;
        NSArray *items = [NSJSONSerialization JSONObjectWithData:data 
                                                        options:0 
                                                          error:&parseError];
        
        if (parseError) {
            completion(nil, parseError);
            return;
        }
        
        completion(items, nil);
    }];
    
    [task resume];
}

@end
```

---

## 3. BGTaskScheduler (iOS 13+)

`BGTaskScheduler` เป็น API ใหม่ที่ Apple แนะนำใน iOS 13 ให้ใช้แทน background fetch เดิม

### 3.1 BGAppRefreshTask

```objc
// AppDelegate.m
#import <BackgroundTasks/BackgroundTasks.h>

static NSString *const kAppRefreshTaskId = @"com.example.app.refresh";

- (BOOL)application:(UIApplication *)application 
    didFinishLaunchingWithOptions:(NSDictionary *)launchOptions {
    
    // ลงทะเบียน background tasks
    [self registerBackgroundTasks];
    
    return YES;
}

- (void)registerBackgroundTasks {
    // ลงทะเบียน app refresh task
    [[BGTaskScheduler shared] registerForTaskWithIdentifier:kAppRefreshTaskId
                                                 usingQueue:nil
                                              launchHandler:^(BGTask *task) {
        [self handleAppRefresh:(BGAppRefreshTask *)task];
    }];
    
    NSLog(@"Background tasks registered");
}

- (void)scheduleAppRefresh {
    BGAppRefreshTaskRequest *request = 
        [[BGAppRefreshTaskRequest alloc] initWithIdentifier:kAppRefreshTaskId];
    
    // กำหนด earliest begin date
    request.earliestBeginDate = [NSDate dateWithTimeIntervalSinceNow:15 * 60]; // 15 นาที
    
    NSError *error;
    BOOL success = [[BGTaskScheduler shared] submitTaskRequest:request error:&error];
    
    if (!success) {
        NSLog(@"Could not schedule app refresh: %@", error.localizedDescription);
    } else {
        NSLog(@"App refresh scheduled for: %@", request.earliestBeginDate);
    }
}

- (void)handleAppRefresh:(BGAppRefreshTask *)task {
    NSLog(@"=== Handling App Refresh Task ===");
    
    // ตั้งเวลา task ต่อไป
    [self scheduleAppRefresh];
    
    // ตรวจสอบ expiration
    task.expirationHandler = ^{
        NSLog(@"App refresh task expired!");
        // หยุดงานทันที
    };
    
    // ดึงข้อมูลใหม่
    [[DataService shared] fetchLatestNewsWithCompletion:^(NSArray *items, NSError *error) {
        if (error || items.count == 0) {
            [task setTaskCompletedWithSuccess:NO];
        } else {
            [[DataStore shared] saveItems:items];
            [task setTaskCompletedWithSuccess:YES];
        }
    }];
}

// เรียก schedule เมื่อแอปเข้า background
- (void)applicationDidEnterBackground:(UIApplication *)application {
    [self scheduleAppRefresh];
}
```

### 3.2 BGProcessingTask

BGProcessingTask สำหรับงานหนักๆ ที่ต้องการเวลานาน

```objc
static NSString *const kProcessingTaskId = @"com.example.app.processing";

- (void)registerBackgroundTasks {
    // ลงทะเบียน app refresh
    [[BGTaskScheduler shared] registerForTaskWithIdentifier:kAppRefreshTaskId
                                                 usingQueue:nil
                                              launchHandler:^(BGTask *task) {
        [self handleAppRefresh:(BGAppRefreshTask *)task];
    }];
    
    // ลงทะเบียน processing task
    [[BGTaskScheduler shared] registerForTaskWithIdentifier:kProcessingTaskId
                                                 usingQueue:nil
                                              launchHandler:^(BGTask *task) {
        [self handleProcessingTask:(BGProcessingTask *)task];
    }];
}

- (void)scheduleProcessingTask {
    BGProcessingTaskRequest *request = 
        [[BGProcessingTaskRequest alloc] initWithIdentifier:kProcessingTaskId];
    
    // ต้องการ power (ชาร์จไฟ)
    request.requiresNetworkConnectivity = YES;
    request.requiresExternalPower = YES;
    request.earliestBeginDate = [NSDate dateWithTimeIntervalSinceNow:60 * 60]; // 1 ชั่วโมง
    
    NSError *error;
    BOOL success = [[BGTaskScheduler shared] submitTaskRequest:request error:&error];
    
    if (!success) {
        NSLog(@"Could not schedule processing task: %@", error.localizedDescription);
    } else {
        NSLog(@"Processing task scheduled");
    }
}

- (void)handleProcessingTask:(BGProcessingTask *)task {
    NSLog(@"=== Handling Processing Task ===");
    NSLog(@"Available time: approximately 1-10 minutes");
    
    task.expirationHandler = ^{
        NSLog(@"Processing task expired!");
        // บันทึก progress และหยุด
        [[DatabaseManager shared] saveProgress];
    };
    
    // ทำงานหนักที่ต้องการเวลา
    dispatch_async(dispatch_get_global_queue(QOS_CLASS_BACKGROUND, 0), ^{
        BOOL success = [self performHeavyProcessing];
        
        dispatch_async(dispatch_get_main_queue(), ^{
            [task setTaskCompletedWithSuccess:success];
            NSLog(@"Processing task completed: %@", success ? @"success" : @"failed");
        });
    });
}

- (BOOL)performHeavyProcessing {
    NSLog(@"Starting heavy processing...");
    
    // ตัวอย่าง: sync database, process images, etc.
    NSArray *unprocessedItems = [[DatabaseManager shared] getUnprocessedItems];
    
    for (NSDictionary *item in unprocessedItems) {
        // ตรวจสอบว่ายังมีเวลาอยู่
        // ในความเป็นจริงควรตรวจสอบ expiration flag
        
        [self processItem:item];
        NSLog(@"Processed item: %@", item[@"id"]);
    }
    
    return YES;
}

- (void)processItem:(NSDictionary *)item {
    // ประมวลผล item
    [NSThread sleepForTimeInterval:0.1]; // จำลองการประมวลผล
}
```

### 3.3 Info.plist สำหรับ BGTaskScheduler

```xml
<key>BGTaskSchedulerPermittedIdentifiers</key>
<array>
    <string>com.example.app.refresh</string>
    <string>com.example.app.processing</string>
</array>
```

---

## 4. Background URLSession (URLSession พื้นหลัง)

Background URLSession ช่วยให้ download/upload ต่อได้แม้แอปจะถูก terminate

### 4.1 สร้าง Background URLSession

```objc
// DownloadManager.h
@interface DownloadManager : NSObject <NSURLSessionDownloadDelegate>

+ (instancetype)shared;

- (void)downloadFileFromURL:(NSURL *)url 
               toLocalPath:(NSString *)localPath
                completion:(void (^)(NSURL *localURL, NSError *error))completion;

- (void)handleEventsForBackgroundURLSession:(NSString *)identifier 
                          completionHandler:(void (^)(void))completionHandler;

@end
```

```objc
// DownloadManager.m
@interface DownloadManager ()

@property (strong, nonatomic) NSURLSession *backgroundSession;
@property (strong, nonatomic) NSMutableDictionary *completionHandlers;
@property (strong, nonatomic) NSMutableDictionary *downloadCompletions;
@property (copy, nonatomic) void (^backgroundSessionCompletionHandler)(void);

@end

@implementation DownloadManager

static NSString *const kBackgroundSessionId = @"com.example.app.backgroundSession";

+ (instancetype)shared {
    static DownloadManager *instance = nil;
    static dispatch_once_t onceToken;
    dispatch_once(&onceToken, ^{
        instance = [[self alloc] init];
    });
    return instance;
}

- (instancetype)init {
    self = [super init];
    if (self) {
        _downloadCompletions = [NSMutableDictionary dictionary];
        [self setupBackgroundSession];
    }
    return self;
}

- (void)setupBackgroundSession {
    NSURLSessionConfiguration *config = 
        [NSURLSessionConfiguration backgroundSessionConfigurationWithIdentifier:kBackgroundSessionId];
    
    // ตั้งค่า
    config.timeoutIntervalForRequest = 30.0;
    config.timeoutIntervalForResource = 3600.0; // 1 ชั่วโมง
    config.allowsCellularAccess = YES;
    config.networkServiceType = NSURLNetworkServiceTypeBackground;
    config.discretionary = YES; // ให้ระบบเลือกเวลาที่เหมาะสม
    
    self.backgroundSession = [NSURLSession sessionWithConfiguration:config
                                                          delegate:self
                                                     delegateQueue:nil];
    
    NSLog(@"Background URL session created: %@", kBackgroundSessionId);
}

- (void)downloadFileFromURL:(NSURL *)url 
               toLocalPath:(NSString *)localPath
                completion:(void (^)(NSURL *localURL, NSError *error))completion {
    
    NSURLSessionDownloadTask *task = [self.backgroundSession downloadTaskWithURL:url];
    
    // บันทึก completion handler
    self.downloadCompletions[@(task.taskIdentifier)] = completion;
    
    [task resume];
    NSLog(@"Download started: %@ (task ID: %lu)", url, (unsigned long)task.taskIdentifier);
}

- (void)handleEventsForBackgroundURLSession:(NSString *)identifier 
                          completionHandler:(void (^)(void))completionHandler {
    
    if ([identifier isEqualToString:kBackgroundSessionId]) {
        self.backgroundSessionCompletionHandler = completionHandler;
        NSLog(@"Stored background session completion handler");
    }
}

#pragma mark - NSURLSessionDownloadDelegate

- (void)URLSession:(NSURLSession *)session 
      downloadTask:(NSURLSessionDownloadTask *)downloadTask 
didFinishDownloadingToURL:(NSURL *)location {
    
    NSLog(@"Download finished to: %@", location);
    
    // ย้ายไฟล์ไปที่ permanent location
    NSString *documentsPath = [NSSearchPathForDirectoriesInDomains(
        NSDocumentDirectory, NSUserDomainMask, YES) firstObject];
    NSString *fileName = downloadTask.response.suggestedFilename ?: @"downloaded_file";
    NSURL *destURL = [NSURL fileURLWithPath:[documentsPath stringByAppendingPathComponent:fileName]];
    
    NSError *error;
    [[NSFileManager defaultManager] moveItemAtURL:location 
                                            toURL:destURL 
                                            error:&error];
    
    if (error) {
        NSLog(@"Error moving file: %@", error.localizedDescription);
    } else {
        NSLog(@"File saved to: %@", destURL.path);
    }
    
    // เรียก completion handler
    void (^completion)(NSURL *, NSError *) = 
        self.downloadCompletions[@(downloadTask.taskIdentifier)];
    
    if (completion) {
        dispatch_async(dispatch_get_main_queue(), ^{
            completion(error ? nil : destURL, error);
        });
        [self.downloadCompletions removeObjectForKey:@(downloadTask.taskIdentifier)];
    }
}

- (void)URLSession:(NSURLSession *)session 
      downloadTask:(NSURLSessionDownloadTask *)downloadTask 
      didWriteData:(int64_t)bytesWritten 
 totalBytesWritten:(int64_t)totalBytesWritten 
totalBytesExpectedToWrite:(int64_t)totalBytesExpectedToWrite {
    
    if (totalBytesExpectedToWrite > 0) {
        float progress = (float)totalBytesWritten / totalBytesExpectedToWrite;
        NSLog(@"Download progress: %.1f%%", progress * 100);
        
        // ส่ง notification เพื่ออัพเดท UI
        dispatch_async(dispatch_get_main_queue(), ^{
            [[NSNotificationCenter defaultCenter] 
             postNotificationName:@"DownloadProgressUpdated"
             object:nil
             userInfo:@{
                 @"taskId": @(downloadTask.taskIdentifier),
                 @"progress": @(progress)
             }];
        });
    }
}

- (void)URLSession:(NSURLSession *)session 
              task:(NSURLSessionTask *)task 
didCompleteWithError:(NSError *)error {
    
    if (error) {
        NSLog(@"Task failed: %@", error.localizedDescription);
        
        void (^completion)(NSURL *, NSError *) = 
            self.downloadCompletions[@(task.taskIdentifier)];
        
        if (completion) {
            dispatch_async(dispatch_get_main_queue(), ^{
                completion(nil, error);
            });
            [self.downloadCompletions removeObjectForKey:@(task.taskIdentifier)];
        }
    }
}

// เรียกเมื่อ background events ทั้งหมดถูก deliver แล้ว
- (void)URLSessionDidFinishEventsForBackgroundURLSession:(NSURLSession *)session {
    NSLog(@"All background URL session events delivered");
    
    dispatch_async(dispatch_get_main_queue(), ^{
        if (self.backgroundSessionCompletionHandler) {
            self.backgroundSessionCompletionHandler();
            self.backgroundSessionCompletionHandler = nil;
        }
    });
}

@end
```

### 4.2 AppDelegate Integration

```objc
// AppDelegate.m
- (void)application:(UIApplication *)application 
    handleEventsForBackgroundURLSession:(NSString *)identifier 
                      completionHandler:(void (^)(void))completionHandler {
    
    NSLog(@"Handle events for background URL session: %@", identifier);
    
    // ส่งต่อให้ DownloadManager จัดการ
    [[DownloadManager shared] handleEventsForBackgroundURLSession:identifier 
                                               completionHandler:completionHandler];
}
```

---

## 5. Push Notifications for Background (Push สำหรับ Background)

### 5.1 Silent Push Notifications

```objc
// AppDelegate.m - ลงทะเบียน push notifications
- (BOOL)application:(UIApplication *)application 
    didFinishLaunchingWithOptions:(NSDictionary *)launchOptions {
    
    [self registerForPushNotifications];
    return YES;
}

- (void)registerForPushNotifications {
    UNUserNotificationCenter *center = [UNUserNotificationCenter currentNotificationCenter];
    center.delegate = self;
    
    [center requestAuthorizationWithOptions:(UNAuthorizationOptionAlert | 
                                             UNAuthorizationOptionBadge | 
                                             UNAuthorizationOptionSound)
                          completionHandler:^(BOOL granted, NSError *error) {
        if (granted) {
            NSLog(@"Notification permission granted");
            dispatch_async(dispatch_get_main_queue(), ^{
                [[UIApplication sharedApplication] registerForRemoteNotifications];
            });
        } else {
            NSLog(@"Notification permission denied: %@", error.localizedDescription);
        }
    }];
}

- (void)application:(UIApplication *)application 
    didRegisterForRemoteNotificationsWithDeviceToken:(NSData *)deviceToken {
    
    // แปลง token เป็น string
    NSString *tokenString = [self stringFromDeviceToken:deviceToken];
    NSLog(@"Device token: %@", tokenString);
    
    // ส่ง token ไปยัง server
    [[APIClient shared] registerDeviceToken:tokenString];
}

- (void)application:(UIApplication *)application 
    didFailToRegisterForRemoteNotificationsWithError:(NSError *)error {
    NSLog(@"Failed to register: %@", error.localizedDescription);
}

- (NSString *)stringFromDeviceToken:(NSData *)deviceToken {
    const unsigned char *dataBuffer = (const unsigned char *)deviceToken.bytes;
    NSMutableString *hexString = [NSMutableString stringWithCapacity:deviceToken.length * 2];
    for (NSInteger i = 0; i < deviceToken.length; ++i) {
        [hexString appendFormat:@"%02x", dataBuffer[i]];
    }
    return [hexString copy];
}

// รับ silent push notification
- (void)application:(UIApplication *)application 
    didReceiveRemoteNotification:(NSDictionary *)userInfo 
          fetchCompletionHandler:(void (^)(UIBackgroundFetchResult))completionHandler {
    
    NSLog(@"=== Remote notification received ===");
    NSLog(@"User info: %@", userInfo);
    
    // ตรวจสอบว่าเป็น silent notification
    NSDictionary *aps = userInfo[@"aps"];
    NSNumber *contentAvailable = aps[@"content-available"];
    
    if ([contentAvailable integerValue] == 1) {
        NSLog(@"Silent notification - fetching data");
        
        // ดึงข้อมูลใหม่
        [[DataService shared] fetchLatestNewsWithCompletion:^(NSArray *items, NSError *error) {
            if (error) {
                completionHandler(UIBackgroundFetchResultFailed);
            } else if (items.count > 0) {
                [[DataStore shared] saveItems:items];
                completionHandler(UIBackgroundFetchResultNewData);
            } else {
                completionHandler(UIBackgroundFetchResultNoData);
            }
        }];
    } else {
        // Visible notification
        completionHandler(UIBackgroundFetchResultNewData);
    }
}
```

### 5.2 Payload สำหรับ Silent Push

```json
{
    "aps": {
        "content-available": 1
    },
    "type": "dataUpdate",
    "category": "news",
    "timestamp": "2024-01-15T10:30:00Z"
}
```

---

## 6. Audio Background Mode (เล่นเสียงใน Background)

```objc
// AudioPlayerManager.h
#import <AVFoundation/AVFoundation.h>

@interface AudioPlayerManager : NSObject

+ (instancetype)shared;

- (void)setupAudioSession;
- (void)playAudioFromURL:(NSURL *)url;
- (void)pause;
- (void)resume;
- (void)stop;
- (BOOL)isPlaying;

@end
```

```objc
// AudioPlayerManager.m
#import "AudioPlayerManager.h"
#import <MediaPlayer/MediaPlayer.h>

@interface AudioPlayerManager () <AVAudioPlayerDelegate>

@property (strong, nonatomic) AVPlayer *player;
@property (strong, nonatomic) AVPlayerItem *currentItem;
@property (assign, nonatomic) BOOL isSetup;

@end

@implementation AudioPlayerManager

+ (instancetype)shared {
    static AudioPlayerManager *instance = nil;
    static dispatch_once_t onceToken;
    dispatch_once(&onceToken, ^{
        instance = [[self alloc] init];
    });
    return instance;
}

- (void)setupAudioSession {
    if (self.isSetup) return;
    
    NSError *error;
    AVAudioSession *session = [AVAudioSession sharedInstance];
    
    // ตั้งค่า category เป็น playback เพื่อเล่นเสียงใน background
    [session setCategory:AVAudioSessionCategoryPlayback 
             withOptions:AVAudioSessionCategoryOptionAllowBluetooth | 
                         AVAudioSessionCategoryOptionAllowAirPlay
                   error:&error];
    
    if (error) {
        NSLog(@"Error setting category: %@", error.localizedDescription);
        return;
    }
    
    // เปิดใช้งาน session
    [session setActive:YES error:&error];
    if (error) {
        NSLog(@"Error activating session: %@", error.localizedDescription);
        return;
    }
    
    self.isSetup = YES;
    NSLog(@"Audio session set up for background playback");
    
    // ตั้งค่า remote controls
    [self setupRemoteTransportControls];
}

- (void)setupRemoteTransportControls {
    MPRemoteCommandCenter *commandCenter = [MPRemoteCommandCenter sharedCommandCenter];
    
    // Play
    [commandCenter.playCommand addTargetWithHandler:^MPRemoteCommandHandlerStatus(MPRemoteCommandEvent *event) {
        [self resume];
        return MPRemoteCommandHandlerStatusSuccess;
    }];
    
    // Pause
    [commandCenter.pauseCommand addTargetWithHandler:^MPRemoteCommandHandlerStatus(MPRemoteCommandEvent *event) {
        [self pause];
        return MPRemoteCommandHandlerStatusSuccess;
    }];
    
    // Next Track
    [commandCenter.nextTrackCommand addTargetWithHandler:^MPRemoteCommandHandlerStatus(MPRemoteCommandEvent *event) {
        [self playNext];
        return MPRemoteCommandHandlerStatusSuccess;
    }];
    
    // Previous Track
    [commandCenter.previousTrackCommand addTargetWithHandler:^MPRemoteCommandHandlerStatus(MPRemoteCommandEvent *event) {
        [self playPrevious];
        return MPRemoteCommandHandlerStatusSuccess;
    }];
    
    NSLog(@"Remote transport controls configured");
}

- (void)playAudioFromURL:(NSURL *)url {
    [self setupAudioSession];
    
    if (self.player) {
        [self.player pause];
    }
    
    self.currentItem = [AVPlayerItem playerItemWithURL:url];
    self.player = [AVPlayer playerWithPlayerItem:self.currentItem];
    [self.player play];
    
    // อัพเดท Now Playing info
    [self updateNowPlayingInfo:@{
        MPMediaItemPropertyTitle: @"Song Title",
        MPMediaItemPropertyArtist: @"Artist Name",
        MPNowPlayingInfoPropertyPlaybackRate: @1.0,
        MPMediaItemPropertyPlaybackDuration: @(CMTimeGetSeconds(self.currentItem.asset.duration))
    }];
    
    NSLog(@"Playing: %@", url);
}

- (void)updateNowPlayingInfo:(NSDictionary *)info {
    [[MPNowPlayingInfoCenter defaultCenter] setNowPlayingInfo:info];
}

- (void)pause {
    [self.player pause];
    NSLog(@"Audio paused");
}

- (void)resume {
    [self.player play];
    NSLog(@"Audio resumed");
}

- (void)stop {
    [self.player pause];
    self.player = nil;
    self.currentItem = nil;
    
    [[MPNowPlayingInfoCenter defaultCenter] setNowPlayingInfo:nil];
    NSLog(@"Audio stopped");
}

- (BOOL)isPlaying {
    return self.player.rate > 0;
}

- (void)playNext {
    NSLog(@"Playing next track");
    // implement playlist logic
}

- (void)playPrevious {
    NSLog(@"Playing previous track");
    // implement playlist logic
}

@end
```

---

## 7. Location Updates in Background (อัพเดทตำแหน่งใน Background)

```objc
// LocationManager.h
#import <CoreLocation/CoreLocation.h>

@interface LocationManager : NSObject <CLLocationManagerDelegate>

+ (instancetype)shared;

- (void)startBackgroundLocationUpdates;
- (void)stopLocationUpdates;
- (CLLocation *)currentLocation;

@property (copy, nonatomic) void (^locationUpdateHandler)(CLLocation *location);

@end
```

```objc
// LocationManager.m
@interface LocationManager ()

@property (strong, nonatomic) CLLocationManager *locationManager;
@property (strong, nonatomic) CLLocation *lastKnownLocation;

@end

@implementation LocationManager

+ (instancetype)shared {
    static LocationManager *instance = nil;
    static dispatch_once_t onceToken;
    dispatch_once(&onceToken, ^{
        instance = [[self alloc] init];
    });
    return instance;
}

- (instancetype)init {
    self = [super init];
    if (self) {
        [self setupLocationManager];
    }
    return self;
}

- (void)setupLocationManager {
    self.locationManager = [[CLLocationManager alloc] init];
    self.locationManager.delegate = self;
    self.locationManager.desiredAccuracy = kCLLocationAccuracyBestForNavigation;
    self.locationManager.distanceFilter = 10; // อัพเดทเมื่อขยับ 10 เมตร
    self.locationManager.pausesLocationUpdatesAutomatically = NO;
    
    // เปิดใช้ background updates
    if ([CLLocationManager locationServicesEnabled]) {
        self.locationManager.allowsBackgroundLocationUpdates = YES;
        self.locationManager.showsBackgroundLocationIndicator = YES; // แสดง indicator บน status bar
    }
}

- (void)startBackgroundLocationUpdates {
    CLAuthorizationStatus status = [CLLocationManager authorizationStatus];
    
    if (status == kCLAuthorizationStatusNotDetermined) {
        [self.locationManager requestAlwaysAuthorization];
    } else if (status == kCLAuthorizationStatusAuthorizedAlways ||
               status == kCLAuthorizationStatusAuthorizedWhenInUse) {
        [self.locationManager startUpdatingLocation];
        NSLog(@"Started background location updates");
    } else {
        NSLog(@"Location access denied");
    }
}

- (void)stopLocationUpdates {
    [self.locationManager stopUpdatingLocation];
    NSLog(@"Stopped location updates");
}

- (CLLocation *)currentLocation {
    return self.lastKnownLocation;
}

#pragma mark - CLLocationManagerDelegate

- (void)locationManager:(CLLocationManager *)manager 
     didUpdateLocations:(NSArray<CLLocation *> *)locations {
    
    CLLocation *location = locations.lastObject;
    self.lastKnownLocation = location;
    
    NSLog(@"Location updated: %.6f, %.6f (accuracy: %.0fm)", 
          location.coordinate.latitude, 
          location.coordinate.longitude,
          location.horizontalAccuracy);
    
    if (self.locationUpdateHandler) {
        self.locationUpdateHandler(location);
    }
    
    // บันทึกตำแหน่งลง database (เช่น สำหรับ tracking app)
    [self saveLocationToDatabase:location];
}

- (void)locationManager:(CLLocationManager *)manager 
      didFailWithError:(NSError *)error {
    NSLog(@"Location error: %@", error.localizedDescription);
    
    if (error.code == kCLErrorDenied) {
        [self stopLocationUpdates];
    }
}

- (void)locationManagerDidChangeAuthorization:(CLLocationManager *)manager {
    CLAuthorizationStatus status = manager.authorizationStatus;
    NSLog(@"Location authorization changed: %ld", (long)status);
    
    if (status == kCLAuthorizationStatusAuthorizedAlways) {
        [manager startUpdatingLocation];
    }
}

- (void)saveLocationToDatabase:(CLLocation *)location {
    NSDictionary *locationData = @{
        @"latitude": @(location.coordinate.latitude),
        @"longitude": @(location.coordinate.longitude),
        @"accuracy": @(location.horizontalAccuracy),
        @"timestamp": location.timestamp
    };
    
    // บันทึกลง local database
    // [[LocationDatabase shared] insertLocation:locationData];
    NSLog(@"Location saved to database");
}

@end
```

### 7.1 Significant Location Changes

```objc
// สำหรับแอปที่ไม่ต้องการ GPS ตลอดเวลา แต่อยากรู้เมื่อ location เปลี่ยนมาก
- (void)startSignificantLocationChanges {
    if (![CLLocationManager significantLocationChangeMonitoringAvailable]) {
        NSLog(@"Significant location changes not available");
        return;
    }
    
    [self.locationManager startMonitoringSignificantLocationChanges];
    NSLog(@"Started monitoring significant location changes");
}

// Geofencing
- (void)addGeofenceAtCoordinate:(CLLocationCoordinate2D)coordinate 
                         radius:(CLLocationDistance)radius 
                     identifier:(NSString *)identifier {
    
    CLCircularRegion *region = [[CLCircularRegion alloc] 
                                 initWithCenter:coordinate
                                        radius:radius
                                    identifier:identifier];
    region.notifyOnEntry = YES;
    region.notifyOnExit = YES;
    
    [self.locationManager startMonitoringForRegion:region];
    NSLog(@"Geofence added: %@ (radius: %.0fm)", identifier, radius);
}

- (void)locationManager:(CLLocationManager *)manager 
         didEnterRegion:(CLRegion *)region {
    NSLog(@"Entered region: %@", region.identifier);
    
    // สร้าง local notification
    [self sendLocalNotificationWithTitle:@"Arrived" 
                                    body:[NSString stringWithFormat:@"You arrived at %@", region.identifier]];
}

- (void)locationManager:(CLLocationManager *)manager 
          didExitRegion:(CLRegion *)region {
    NSLog(@"Exited region: %@", region.identifier);
}

- (void)sendLocalNotificationWithTitle:(NSString *)title body:(NSString *)body {
    UNMutableNotificationContent *content = [[UNMutableNotificationContent alloc] init];
    content.title = title;
    content.body = body;
    content.sound = [UNNotificationSound defaultSound];
    
    UNTimeIntervalNotificationTrigger *trigger = 
        [UNTimeIntervalNotificationTrigger triggerWithTimeInterval:1 repeats:NO];
    
    NSString *requestId = [[NSUUID UUID] UUIDString];
    UNNotificationRequest *request = [UNNotificationRequest requestWithIdentifier:requestId
                                                                          content:content
                                                                          trigger:trigger];
    
    [[UNUserNotificationCenter currentNotificationCenter] addNotificationRequest:request 
                                                           withCompletionHandler:nil];
}
```

---

## 8. VoIP Apps (แอปโทรศัพท์ VoIP)

```objc
// VoIPManager.h
#import <PushKit/PushKit.h>

@interface VoIPManager : NSObject <PKPushRegistryDelegate>

+ (instancetype)shared;
- (void)registerForVoIPPush;

@end
```

```objc
// VoIPManager.m
#import "VoIPManager.h"
#import <CallKit/CallKit.h>

@interface VoIPManager () <CXProviderDelegate>

@property (strong, nonatomic) PKPushRegistry *pushRegistry;
@property (strong, nonatomic) CXProvider *callProvider;
@property (strong, nonatomic) CXCallController *callController;

@end

@implementation VoIPManager

+ (instancetype)shared {
    static VoIPManager *instance = nil;
    static dispatch_once_t onceToken;
    dispatch_once(&onceToken, ^{
        instance = [[self alloc] init];
    });
    return instance;
}

- (instancetype)init {
    self = [super init];
    if (self) {
        [self setupCallKit];
    }
    return self;
}

- (void)setupCallKit {
    CXProviderConfiguration *config = 
        [[CXProviderConfiguration alloc] initWithLocalizedName:@"MyVoIP App"];
    config.supportsVideo = YES;
    config.maximumCallsPerCallGroup = 1;
    config.supportedHandleTypes = [NSSet setWithObject:@(CXHandleTypePhoneNumber)];
    
    self.callProvider = [[CXProvider alloc] initWithConfiguration:config];
    [self.callProvider setDelegate:self queue:nil];
    
    self.callController = [[CXCallController alloc] init];
    NSLog(@"CallKit configured");
}

- (void)registerForVoIPPush {
    self.pushRegistry = [[PKPushRegistry alloc] initWithQueue:dispatch_get_main_queue()];
    self.pushRegistry.delegate = self;
    self.pushRegistry.desiredPushTypes = [NSSet setWithObject:PKPushTypeVoIP];
    NSLog(@"Registering for VoIP push");
}

#pragma mark - PKPushRegistryDelegate

- (void)pushRegistry:(PKPushRegistry *)registry 
didUpdatePushCredentials:(PKPushCredentials *)credentials 
             forType:(PKPushType)type {
    
    NSData *token = credentials.token;
    NSString *tokenString = [self stringFromData:token];
    NSLog(@"VoIP push token: %@", tokenString);
    
    // ส่ง token ไปยัง server
    [[APIClient shared] registerVoIPToken:tokenString];
}

- (void)pushRegistry:(PKPushRegistry *)registry 
    didReceiveIncomingPushWithPayload:(PKPushPayload *)payload 
                              forType:(PKPushType)type 
                withCompletionHandler:(void (^)(void))completion {
    
    NSLog(@"VoIP push received: %@", payload.dictionaryPayload);
    
    // ต้องรายงาน incoming call ทันที
    NSDictionary *callInfo = payload.dictionaryPayload;
    NSString *callerId = callInfo[@"callerId"];
    NSString *callerName = callInfo[@"callerName"];
    NSUUID *callUUID = [[NSUUID alloc] init];
    
    // รายงาน incoming call ผ่าน CallKit
    CXCallUpdate *update = [[CXCallUpdate alloc] init];
    update.remoteHandle = [[CXHandle alloc] initWithType:CXHandleTypePhoneNumber 
                                                   value:callerId];
    update.localizedCallerName = callerName;
    update.hasVideo = [callInfo[@"hasVideo"] boolValue];
    
    [self.callProvider reportNewIncomingCallWithUUID:callUUID 
                                              update:update 
                                          completion:^(NSError *error) {
        if (error) {
            NSLog(@"Error reporting incoming call: %@", error.localizedDescription);
        } else {
            NSLog(@"Incoming call reported: %@", callerId);
        }
        completion();
    }];
}

#pragma mark - CXProviderDelegate

- (void)providerDidReset:(CXProvider *)provider {
    NSLog(@"Provider reset - terminate all calls");
}

- (void)provider:(CXProvider *)provider performAnswerCallAction:(CXAnswerCallAction *)action {
    NSLog(@"Answering call: %@", action.callUUID);
    
    // ตั้งค่า audio session
    [[AVAudioSession sharedInstance] setCategory:AVAudioSessionCategoryPlayAndRecord 
                                           error:nil];
    [[AVAudioSession sharedInstance] setActive:YES error:nil];
    
    // เชื่อมต่อ call จริง
    // [self connectCallWithUUID:action.callUUID];
    
    [action fulfill];
}

- (void)provider:(CXProvider *)provider performEndCallAction:(CXEndCallAction *)action {
    NSLog(@"Ending call: %@", action.callUUID);
    
    // ตัดการเชื่อมต่อ
    // [self disconnectCallWithUUID:action.callUUID];
    
    [action fulfill];
}

- (void)provider:(CXProvider *)provider didActivateAudioSession:(AVAudioSession *)audioSession {
    NSLog(@"Audio session activated for call");
    // เริ่มส่ง audio
}

- (void)provider:(CXProvider *)provider didDeactivateAudioSession:(AVAudioSession *)audioSession {
    NSLog(@"Audio session deactivated");
}

- (NSString *)stringFromData:(NSData *)data {
    NSMutableString *string = [NSMutableString stringWithCapacity:data.length * 2];
    const unsigned char *bytes = data.bytes;
    for (NSUInteger i = 0; i < data.length; i++) {
        [string appendFormat:@"%02x", bytes[i]];
    }
    return string;
}

@end
```

---

## 9. BGAppRefreshTask - รายละเอียด

```objc
// BackgroundTaskManager.h
#import <BackgroundTasks/BackgroundTasks.h>

@interface BackgroundTaskManager : NSObject

+ (instancetype)shared;

- (void)registerAllTasks;
- (void)scheduleAllTasks;

@end
```

```objc
// BackgroundTaskManager.m
@implementation BackgroundTaskManager

static NSString *const kNewsRefreshId = @"com.example.app.news.refresh";
static NSString *const kDatabaseCleanupId = @"com.example.app.database.cleanup";
static NSString *const kImageProcessingId = @"com.example.app.image.processing";

+ (instancetype)shared {
    static BackgroundTaskManager *instance = nil;
    static dispatch_once_t onceToken;
    dispatch_once(&onceToken, ^{
        instance = [[self alloc] init];
    });
    return instance;
}

- (void)registerAllTasks {
    // News Refresh (App Refresh Task - เร็ว)
    [[BGTaskScheduler shared] registerForTaskWithIdentifier:kNewsRefreshId
                                                 usingQueue:nil
                                              launchHandler:^(BGTask *task) {
        [self handleNewsRefresh:(BGAppRefreshTask *)task];
    }];
    
    // Database Cleanup (Processing Task - หนัก)
    [[BGTaskScheduler shared] registerForTaskWithIdentifier:kDatabaseCleanupId
                                                 usingQueue:nil
                                              launchHandler:^(BGTask *task) {
        [self handleDatabaseCleanup:(BGProcessingTask *)task];
    }];
    
    // Image Processing (Processing Task - หนักและต้องการไฟ)
    [[BGTaskScheduler shared] registerForTaskWithIdentifier:kImageProcessingId
                                                 usingQueue:nil
                                              launchHandler:^(BGTask *task) {
        [self handleImageProcessing:(BGProcessingTask *)task];
    }];
    
    NSLog(@"All background tasks registered");
}

- (void)scheduleAllTasks {
    [self scheduleNewsRefresh];
    [self scheduleDatabaseCleanup];
    // Image processing เฉพาะเมื่อมีงานรอ
    if ([self hasPendingImageWork]) {
        [self scheduleImageProcessing];
    }
}

- (void)scheduleNewsRefresh {
    BGAppRefreshTaskRequest *request = 
        [[BGAppRefreshTaskRequest alloc] initWithIdentifier:kNewsRefreshId];
    request.earliestBeginDate = [NSDate dateWithTimeIntervalSinceNow:30 * 60]; // 30 นาที
    
    NSError *error;
    if (![[BGTaskScheduler shared] submitTaskRequest:request error:&error]) {
        NSLog(@"Failed to schedule news refresh: %@", error.localizedDescription);
    }
}

- (void)scheduleDatabaseCleanup {
    BGProcessingTaskRequest *request = 
        [[BGProcessingTaskRequest alloc] initWithIdentifier:kDatabaseCleanupId];
    request.requiresNetworkConnectivity = NO;
    request.requiresExternalPower = NO;
    request.earliestBeginDate = [NSDate dateWithTimeIntervalSinceNow:24 * 60 * 60]; // ทุกวัน
    
    NSError *error;
    if (![[BGTaskScheduler shared] submitTaskRequest:request error:&error]) {
        NSLog(@"Failed to schedule DB cleanup: %@", error.localizedDescription);
    }
}

- (void)scheduleImageProcessing {
    BGProcessingTaskRequest *request = 
        [[BGProcessingTaskRequest alloc] initWithIdentifier:kImageProcessingId];
    request.requiresNetworkConnectivity = YES;
    request.requiresExternalPower = YES; // ต้องชาร์จไฟ
    
    NSError *error;
    if (![[BGTaskScheduler shared] submitTaskRequest:request error:&error]) {
        NSLog(@"Failed to schedule image processing: %@", error.localizedDescription);
    }
}

- (void)handleNewsRefresh:(BGAppRefreshTask *)task {
    [self scheduleNewsRefresh]; // ตั้งเวลาครั้งต่อไป
    
    __block BOOL expired = NO;
    task.expirationHandler = ^{
        expired = YES;
        NSLog(@"News refresh expired");
    };
    
    [[NewsAPI shared] fetchLatestWithCompletion:^(NSArray *articles, NSError *error) {
        if (expired) {
            [task setTaskCompletedWithSuccess:NO];
            return;
        }
        
        if (articles.count > 0) {
            [[NewsDatabase shared] saveArticles:articles];
            NSLog(@"Saved %lu new articles", (unsigned long)articles.count);
        }
        
        [task setTaskCompletedWithSuccess:(error == nil)];
    }];
}

- (void)handleDatabaseCleanup:(BGProcessingTask *)task {
    __block BOOL expired = NO;
    task.expirationHandler = ^{
        expired = YES;
    };
    
    dispatch_async(dispatch_get_global_queue(QOS_CLASS_BACKGROUND, 0), ^{
        if (!expired) {
            [[DatabaseManager shared] deleteOldRecords];
            [[DatabaseManager shared] vacuum];
            NSLog(@"Database cleanup completed");
        }
        
        [task setTaskCompletedWithSuccess:!expired];
    });
}

- (void)handleImageProcessing:(BGProcessingTask *)task {
    __block BOOL expired = NO;
    task.expirationHandler = ^{
        expired = YES;
    };
    
    NSArray *pendingImages = [[ImageQueue shared] getPendingImages];
    __block NSInteger processedCount = 0;
    
    dispatch_async(dispatch_get_global_queue(QOS_CLASS_BACKGROUND, 0), ^{
        for (NSDictionary *imageInfo in pendingImages) {
            if (expired) break;
            
            [[ImageProcessor shared] processImage:imageInfo];
            processedCount++;
        }
        
        NSLog(@"Processed %ld images", (long)processedCount);
        [task setTaskCompletedWithSuccess:YES];
    });
}

- (BOOL)hasPendingImageWork {
    return [[ImageQueue shared] getPendingImages].count > 0;
}

@end
```

---

## 10. Debugging Background Tasks (การ Debug)

### 10.1 Simulating Background Tasks ใน Simulator

```objc
// ใช้ LLDB ใน Xcode Simulator
// 1. เปิด Debug > Simulate Background Fetch
// 2. หรือใช้ command ใน LLDB:
// e -l objc -- (void)[[BGTaskScheduler sharedScheduler] _simulateLaunchForTaskWithIdentifier:@"com.example.app.refresh"]
```

### 10.2 Logging สำหรับ Background Tasks

```objc
// BackgroundLogger.h
@interface BackgroundLogger : NSObject

+ (instancetype)shared;
- (void)logEvent:(NSString *)event;
- (void)logTask:(NSString *)taskId 
          event:(NSString *)event 
           info:(NSDictionary *)info;
- (void)exportLogs;

@end

@implementation BackgroundLogger

+ (instancetype)shared {
    static BackgroundLogger *instance = nil;
    static dispatch_once_t onceToken;
    dispatch_once(&onceToken, ^{
        instance = [[self alloc] init];
    });
    return instance;
}

- (void)logEvent:(NSString *)event {
    [self logTask:@"general" event:event info:nil];
}

- (void)logTask:(NSString *)taskId 
          event:(NSString *)event 
           info:(NSDictionary *)info {
    
    NSMutableDictionary *log = [NSMutableDictionary dictionary];
    log[@"timestamp"] = [NSDate date].description;
    log[@"taskId"] = taskId;
    log[@"event"] = event;
    log[@"appState"] = @([UIApplication sharedApplication].applicationState);
    
    if (info) {
        [log addEntriesFromDictionary:info];
    }
    
    // บันทึกลงไฟล์
    [self appendLogEntry:log];
    
    NSLog(@"[BGLog] %@ - %@", taskId, event);
}

- (void)appendLogEntry:(NSDictionary *)entry {
    NSString *logPath = [self logFilePath];
    
    NSMutableArray *logs = [self loadLogs];
    [logs addObject:entry];
    
    // เก็บแค่ 1000 entries ล่าสุด
    if (logs.count > 1000) {
        [logs removeObjectsInRange:NSMakeRange(0, logs.count - 1000)];
    }
    
    NSError *error;
    NSData *data = [NSJSONSerialization dataWithJSONObject:logs options:NSJSONWritingPrettyPrinted error:&error];
    [data writeToFile:logPath atomically:YES];
}

- (NSMutableArray *)loadLogs {
    NSString *logPath = [self logFilePath];
    NSData *data = [NSData dataWithContentsOfFile:logPath];
    if (!data) return [NSMutableArray array];
    
    NSError *error;
    NSArray *logs = [NSJSONSerialization JSONObjectWithData:data options:0 error:&error];
    return logs ? [logs mutableCopy] : [NSMutableArray array];
}

- (NSString *)logFilePath {
    NSString *docs = [NSSearchPathForDirectoriesInDomains(
        NSDocumentDirectory, NSUserDomainMask, YES) firstObject];
    return [docs stringByAppendingPathComponent:@"background_tasks.json"];
}

- (void)exportLogs {
    NSArray *logs = [self loadLogs];
    NSLog(@"=== Background Task Logs (%lu entries) ===", (unsigned long)logs.count);
    for (NSDictionary *entry in logs) {
        NSLog(@"%@ | %@ | %@", entry[@"timestamp"], entry[@"taskId"], entry[@"event"]);
    }
}

@end
```

---

## 11. Battery and Performance Considerations (แบตเตอรี่และ Performance)

```objc
// BatteryManager.h
@interface BatteryManager : NSObject

+ (instancetype)shared;
- (void)startMonitoring;
- (BOOL)shouldPerformBatteryIntensiveTask;
- (float)currentBatteryLevel;
- (UIDeviceBatteryState)currentBatteryState;

@end
```

```objc
// BatteryManager.m
@implementation BatteryManager

+ (instancetype)shared {
    static BatteryManager *instance = nil;
    static dispatch_once_t onceToken;
    dispatch_once(&onceToken, ^{
        instance = [[self alloc] init];
    });
    return instance;
}

- (void)startMonitoring {
    [[UIDevice currentDevice] setBatteryMonitoringEnabled:YES];
    
    [[NSNotificationCenter defaultCenter] addObserver:self
                                             selector:@selector(batteryLevelChanged:)
                                                 name:UIDeviceBatteryLevelDidChangeNotification
                                               object:nil];
    
    [[NSNotificationCenter defaultCenter] addObserver:self
                                             selector:@selector(batteryStateChanged:)
                                                 name:UIDeviceBatteryStateDidChangeNotification
                                               object:nil];
    
    NSLog(@"Battery monitoring started. Level: %.0f%%, State: %ld", 
          [self currentBatteryLevel] * 100, (long)[self currentBatteryState]);
}

- (BOOL)shouldPerformBatteryIntensiveTask {
    float level = [self currentBatteryLevel];
    UIDeviceBatteryState state = [self currentBatteryState];
    
    // ดำเนินการได้ถ้าชาร์จอยู่ หรือแบตมากกว่า 30%
    if (state == UIDeviceBatteryStateCharging || 
        state == UIDeviceBatteryStateFull) {
        return YES;
    }
    
    return level > 0.3;
}

- (float)currentBatteryLevel {
    return [UIDevice currentDevice].batteryLevel;
}

- (UIDeviceBatteryState)currentBatteryState {
    return [UIDevice currentDevice].batteryState;
}

- (void)batteryLevelChanged:(NSNotification *)notification {
    float level = [self currentBatteryLevel];
    NSLog(@"Battery level: %.0f%%", level * 100);
    
    if (level < 0.1) {
        NSLog(@"Low battery! Reducing background activity");
        [[NSNotificationCenter defaultCenter] 
         postNotificationName:@"LowBatteryDetected" object:nil];
    }
}

- (void)batteryStateChanged:(NSNotification *)notification {
    UIDeviceBatteryState state = [self currentBatteryState];
    NSString *stateStr;
    switch (state) {
        case UIDeviceBatteryStateCharging: stateStr = @"Charging"; break;
        case UIDeviceBatteryStateFull: stateStr = @"Full"; break;
        case UIDeviceBatteryStateUnplugged: stateStr = @"Unplugged"; break;
        default: stateStr = @"Unknown"; break;
    }
    NSLog(@"Battery state: %@", stateStr);
}

@end
```

### 11.1 Performance Monitoring

```objc
// ตรวจสอบ memory usage
+ (uint64_t)memoryUsageInBytes {
    struct task_basic_info info;
    mach_msg_type_number_t size = sizeof(info);
    kern_return_t kerr = task_info(mach_task_self(), 
                                   TASK_BASIC_INFO, 
                                   (task_info_t)&info, 
                                   &size);
    if (kerr == KERN_SUCCESS) {
        return info.resident_size;
    }
    return 0;
}

// ตรวจสอบก่อนทำงานหนัก
- (void)performBackgroundTaskWithBatteryCheck:(void(^)(void))task {
    if (![[BatteryManager shared] shouldPerformBatteryIntensiveTask]) {
        NSLog(@"Skipping task due to low battery");
        return;
    }
    
    uint64_t memBefore = [self memoryUsageInBytes];
    NSDate *start = [NSDate date];
    
    task();
    
    uint64_t memAfter = [self memoryUsageInBytes];
    NSTimeInterval elapsed = [[NSDate date] timeIntervalSinceDate:start];
    
    NSLog(@"Task completed in %.2fs, memory delta: %lld KB", 
          elapsed, 
          (memAfter - memBefore) / 1024);
}
```

---

## 12. แบบฝึกหัด (Practice Exercises)

### Exercise 1: News Refresh System

สร้าง complete background news refresh system:

```objc
// NewsRefreshSystem.h
@interface NewsRefreshSystem : NSObject

+ (instancetype)shared;
- (void)setup;
- (void)refresh;

@end

// NewsRefreshSystem.m
@implementation NewsRefreshSystem

+ (instancetype)shared {
    static NewsRefreshSystem *instance = nil;
    static dispatch_once_t onceToken;
    dispatch_once(&onceToken, ^{
        instance = [[self alloc] init];
    });
    return instance;
}

- (void)setup {
    // ลงทะเบียน BGTask
    NSString *taskId = @"com.example.news.refresh";
    [[BGTaskScheduler shared] registerForTaskWithIdentifier:taskId
                                                 usingQueue:nil
                                              launchHandler:^(BGTask *task) {
        [self handleRefreshTask:(BGAppRefreshTask *)task];
    }];
    
    // ลงทะเบียน notification observer
    [[NSNotificationCenter defaultCenter] addObserver:self
                                             selector:@selector(scheduleRefresh)
                                                 name:UIApplicationDidEnterBackgroundNotification
                                               object:nil];
    
    NSLog(@"NewsRefreshSystem set up");
}

- (void)scheduleRefresh {
    BGAppRefreshTaskRequest *request = 
        [[BGAppRefreshTaskRequest alloc] initWithIdentifier:@"com.example.news.refresh"];
    request.earliestBeginDate = [NSDate dateWithTimeIntervalSinceNow:15 * 60];
    
    NSError *error;
    [[BGTaskScheduler shared] submitTaskRequest:request error:&error];
    
    if (error) {
        NSLog(@"Failed to schedule: %@", error);
    }
}

- (void)handleRefreshTask:(BGAppRefreshTask *)task {
    [self scheduleRefresh];
    
    [self refresh];
    
    // Timeout handler
    task.expirationHandler = ^{
        NSLog(@"Refresh expired");
        [task setTaskCompletedWithSuccess:NO];
    };
}

- (void)refresh {
    [[DataService shared] fetchLatestNewsWithCompletion:^(NSArray *items, NSError *error) {
        if (error == nil && items.count > 0) {
            [[DataStore shared] saveItems:items];
            [self sendNewsAvailableNotification:items.count];
        }
    }];
}

- (void)sendNewsAvailableNotification:(NSInteger)count {
    UNMutableNotificationContent *content = [[UNMutableNotificationContent alloc] init];
    content.title = @"ข่าวใหม่มาแล้ว!";
    content.body = [NSString stringWithFormat:@"มี %ld ข่าวใหม่รอคุณอยู่", (long)count];
    content.badge = @(count);
    
    UNTimeIntervalNotificationTrigger *trigger = 
        [UNTimeIntervalNotificationTrigger triggerWithTimeInterval:1 repeats:NO];
    
    UNNotificationRequest *request = 
        [UNNotificationRequest requestWithIdentifier:[[NSUUID UUID] UUIDString]
                                             content:content
                                             trigger:trigger];
    
    [[UNUserNotificationCenter currentNotificationCenter] addNotificationRequest:request 
                                                           withCompletionHandler:nil];
}

@end
```

### Exercise 2: File Download Queue

```objc
// DownloadQueue.h
@interface DownloadQueue : NSObject

+ (instancetype)shared;
- (void)addDownload:(NSURL *)url priority:(NSInteger)priority;
- (void)startProcessing;
- (NSArray *)getPendingDownloads;

@end

// DownloadQueue.m - ใช้ Background URLSession
@implementation DownloadQueue

+ (instancetype)shared {
    static DownloadQueue *instance = nil;
    static dispatch_once_t onceToken;
    dispatch_once(&onceToken, ^{
        instance = [[self alloc] init];
    });
    return instance;
}

- (void)addDownload:(NSURL *)url priority:(NSInteger)priority {
    // บันทึกลง queue database
    [[NSUserDefaults standardUserDefaults] 
     setObject:url.absoluteString
        forKey:[NSString stringWithFormat:@"download_%@", [[NSUUID UUID] UUIDString]]];
    
    NSLog(@"Added to download queue: %@ (priority: %ld)", url, (long)priority);
}

- (void)startProcessing {
    NSArray *pending = [self getPendingDownloads];
    NSLog(@"Processing %lu pending downloads", (unsigned long)pending.count);
    
    for (NSString *urlString in pending) {
        NSURL *url = [NSURL URLWithString:urlString];
        [[DownloadManager shared] downloadFileFromURL:url
                                         toLocalPath:nil
                                          completion:^(NSURL *localURL, NSError *error) {
            if (error) {
                NSLog(@"Download failed: %@", error);
            } else {
                NSLog(@"Downloaded to: %@", localURL);
                [self markDownloadComplete:urlString];
            }
        }];
    }
}

- (NSArray *)getPendingDownloads {
    NSMutableArray *pending = [NSMutableArray array];
    NSDictionary *allDefaults = [[NSUserDefaults standardUserDefaults] dictionaryRepresentation];
    
    for (NSString *key in allDefaults) {
        if ([key hasPrefix:@"download_"]) {
            [pending addObject:allDefaults[key]];
        }
    }
    return pending;
}

- (void)markDownloadComplete:(NSString *)urlString {
    // ลบออกจาก queue
    NSDictionary *allDefaults = [[NSUserDefaults standardUserDefaults] dictionaryRepresentation];
    for (NSString *key in allDefaults) {
        if ([key hasPrefix:@"download_"] && [allDefaults[key] isEqualToString:urlString]) {
            [[NSUserDefaults standardUserDefaults] removeObjectForKey:key];
            break;
        }
    }
}

@end
```

---

## สรุป (Summary)

ในบทนี้เราได้เรียนรู้:

1. **Background Execution Modes** - ประเภทต่างๆ ของ background modes ที่ iOS รองรับ
2. **Background Fetch** - วิธีดึงข้อมูลใน background แบบเดิม
3. **BGTaskScheduler** - API ใหม่สำหรับ iOS 13+ ที่มีประสิทธิภาพมากกว่า
4. **Background URLSession** - Download/Upload ที่ทำงานต่อได้แม้แอปถูก terminate
5. **Silent Push Notifications** - ใช้ push เพื่อ trigger background fetch
6. **Audio Background** - เล่นเสียงต่อใน background ด้วย AVAudioSession
7. **Location Background** - ติดตาม GPS, Geofencing ใน background
8. **VoIP** - ใช้ PushKit และ CallKit สำหรับ VoIP apps
9. **Debugging** - เทคนิคการ debug background tasks
10. **Battery Considerations** - ดูแลแบตเตอรี่และ performance

### Best Practices

- ใช้ `BGTaskScheduler` แทน background fetch เดิมสำหรับ iOS 13+
- เรียก `completionHandler` เสมอภายในเวลาที่กำหนด
- ตรวจสอบ `expirationHandler` และหยุดทำงานเมื่อหมดเวลา
- ใช้ `discretionary` = YES เมื่อไม่ต้องการเร่งด่วน
- ประกาศเฉพาะ background modes ที่ใช้จริงใน Info.plist
- ทดสอบ background behaviors อย่างละเอียดก่อน submit

---

*ต่อไป: Part 83 - WatchKit Development (การพัฒนาแอปสำหรับ Apple Watch)*
