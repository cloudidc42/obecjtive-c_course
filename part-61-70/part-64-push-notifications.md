# Part 64 - Push Notifications

## บทนำ

Push Notifications ทำให้ app สามารถส่งข้อความหรือ alerts ถึงผู้ใช้ได้แม้ app จะปิดอยู่หรืออยู่ใน background ซึ่งเป็นหนึ่งในฟีเจอร์ที่สำคัญที่สุดของ mobile apps

ในบทนี้เราจะเรียนรู้ทุกอย่างเกี่ยวกับ Push Notifications ตั้งแต่ APNs architecture ไปจนถึง rich notifications และ silent push

---

## 64.1 APNs (Apple Push Notification service)

### Architecture ของ APNs

```
[Your Server] → [APNs] → [iOS Device] → [Your App]

1. App ลงทะเบียนกับ APNs และได้รับ device token
2. App ส่ง device token ไปยัง server ของคุณ
3. Server ส่ง notification ไปยัง APNs พร้อม device token
4. APNs ส่ง notification ไปยัง device
5. iOS แสดง notification
```

### ขั้นตอนการ Setup

```
1. สร้าง App ID ใน Apple Developer Portal พร้อมเปิด Push Notifications
2. สร้าง APNs key หรือ certificate
3. Upload key/certificate ไปยัง server
4. ใน app: ขอ permission และลงทะเบียน device token
5. ส่ง token ไปยัง server
```

### APNs Authentication

มี 2 วิธีใน authentication ระหว่าง server กับ APNs:

```
1. APNs Auth Key (แนะนำ):
   - ใช้ .p8 key file
   - Key เดียวใช้ได้กับทุก apps ใน account
   - ไม่มีวันหมดอายุ
   - ง่ายต่อการ manage
   
2. APNs Certificate (legacy):
   - ใช้ .p12 certificate
   - แต่ละ app มี certificate แยก
   - หมดอายุทุก 1 ปี
   - ต้อง renew เป็นระยะ
```

---

## 64.2 การลงทะเบียน Push Notifications

### ใน AppDelegate

```objc
#import <UserNotifications/UserNotifications.h>

@interface AppDelegate : UIResponder <UIApplicationDelegate, UNUserNotificationCenterDelegate>
@end

@implementation AppDelegate

- (BOOL)application:(UIApplication *)application 
    didFinishLaunchingWithOptions:(NSDictionary *)launchOptions {
    
    // Setup notification center delegate
    [UNUserNotificationCenter currentNotificationCenter].delegate = self;
    
    // ขอ permission
    [self requestNotificationPermission];
    
    // ตรวจสอบว่า app ถูกเปิดจาก notification หรือไม่
    UNNotificationResponse *response = launchOptions[UIApplicationLaunchOptionsRemoteNotificationKey];
    if (response) {
        NSLog(@"App launched from notification");
        // จัดการ launch notification
    }
    
    return YES;
}

// ลงทะเบียนกับ APNs สำเร็จ
- (void)application:(UIApplication *)application 
    didRegisterForRemoteNotificationsWithDeviceToken:(NSData *)deviceToken {
    
    // แปลง device token เป็น string
    NSString *tokenString = [self deviceTokenStringFromData:deviceToken];
    NSLog(@"Device Token: %@", tokenString);
    
    // บันทึก token ใน UserDefaults (สำหรับใช้ใน app)
    [[NSUserDefaults standardUserDefaults] setObject:tokenString forKey:@"deviceToken"];
    
    // ส่ง token ไปยัง server ของคุณ
    [self sendDeviceTokenToServer:tokenString];
}

// ลงทะเบียน failed
- (void)application:(UIApplication *)application 
    didFailToRegisterForRemoteNotificationsWithError:(NSError *)error {
    NSLog(@"Failed to register for notifications: %@", error);
    
    // อาจเกิดจาก simulator หรือ network issues
    // ในการ production ให้ log error นี้
}

// Helper: แปลง NSData เป็น token string
- (NSString *)deviceTokenStringFromData:(NSData *)data {
    // วิธีที่ 1: แปลง byte ต่อ byte
    NSMutableString *token = [NSMutableString string];
    const unsigned char *bytes = (const unsigned char *)[data bytes];
    for (NSUInteger i = 0; i < [data length]; i++) {
        [token appendFormat:@"%02x", bytes[i]];
    }
    return [token copy];
    
    // วิธีที่ 2 (iOS 13+):
    // return [data description]; // ไม่แนะนำ เพราะ format อาจเปลี่ยน
}

// ส่ง token ไปยัง server
- (void)sendDeviceTokenToServer:(NSString *)token {
    NSString *serverURL = @"https://your-server.com/api/register-device";
    NSURL *url = [NSURL URLWithString:serverURL];
    NSMutableURLRequest *request = [NSMutableURLRequest requestWithURL:url];
    request.HTTPMethod = @"POST";
    [request setValue:@"application/json" forHTTPHeaderField:@"Content-Type"];
    
    NSDictionary *body = @{
        @"device_token": token,
        @"platform": @"ios",
        @"app_version": [[NSBundle mainBundle] objectForInfoDictionaryKey:@"CFBundleShortVersionString"]
    };
    
    request.HTTPBody = [NSJSONSerialization dataWithJSONObject:body options:0 error:nil];
    
    NSURLSession *session = [NSURLSession sharedSession];
    [[session dataTaskWithRequest:request 
               completionHandler:^(NSData *data, NSURLResponse *response, NSError *error) {
        if (error) {
            NSLog(@"Failed to register device: %@", error);
        } else {
            NSLog(@"Device registered successfully");
        }
    }] resume];
}

@end
```

---

## 64.3 UNUserNotificationCenter

UNUserNotificationCenter เป็น center หลักสำหรับจัดการ notifications ทุกประเภท

```objc
#import <UserNotifications/UserNotifications.h>

// เข้าถึง notification center
UNUserNotificationCenter *center = [UNUserNotificationCenter currentNotificationCenter];

// ตรวจสอบ settings ปัจจุบัน
[center getNotificationSettingsWithCompletionHandler:^(UNNotificationSettings *settings) {
    NSLog(@"Authorization status: %ld", (long)settings.authorizationStatus);
    NSLog(@"Alert setting: %ld", (long)settings.alertSetting);
    NSLog(@"Badge setting: %ld", (long)settings.badgeSetting);
    NSLog(@"Sound setting: %ld", (long)settings.soundSetting);
    NSLog(@"Lock screen: %ld", (long)settings.lockScreenSetting);
    NSLog(@"Notification center: %ld", (long)settings.notificationCenterSetting);
    NSLog(@"Critical alerts: %ld", (long)settings.criticalAlertSetting);
    
    switch (settings.authorizationStatus) {
        case UNAuthorizationStatusNotDetermined:
            NSLog(@"Permission not yet requested");
            break;
        case UNAuthorizationStatusDenied:
            NSLog(@"Permission denied by user");
            break;
        case UNAuthorizationStatusAuthorized:
            NSLog(@"Permission granted");
            break;
        case UNAuthorizationStatusProvisional:
            NSLog(@"Provisional permission (quiet delivery)");
            break;
        case UNAuthorizationStatusEphemeral:
            NSLog(@"Ephemeral permission (App Clips)");
            break;
    }
}];
```

---

## 64.4 การขอ Notification Permissions

```objc
- (void)requestNotificationPermission {
    UNUserNotificationCenter *center = [UNUserNotificationCenter currentNotificationCenter];
    
    // กำหนด options ที่ต้องการ
    UNAuthorizationOptions options = 
        UNAuthorizationOptionAlert |    // แสดง alert
        UNAuthorizationOptionBadge |    // แสดง badge บน icon
        UNAuthorizationOptionSound |    // เล่นเสียง
        UNAuthorizationOptionProvisional; // quiet notifications (ไม่ขอ permission จริงๆ)
    
    [center requestAuthorizationWithOptions:options 
                          completionHandler:^(BOOL granted, NSError *error) {
        if (error) {
            NSLog(@"Permission request error: %@", error);
            return;
        }
        
        if (granted) {
            NSLog(@"Notification permission granted");
            dispatch_async(dispatch_get_main_queue(), ^{
                // ลงทะเบียนสำหรับ remote notifications
                [[UIApplication sharedApplication] registerForRemoteNotifications];
            });
        } else {
            NSLog(@"Notification permission denied");
            // แนะนำให้ user ไปเปิดใน Settings
            [self showPermissionDeniedAlert];
        }
    }];
}

// แนะนำให้ user เปิด permission ใน Settings
- (void)showPermissionDeniedAlert {
    dispatch_async(dispatch_get_main_queue(), ^{
        UIAlertController *alert = [UIAlertController
            alertControllerWithTitle:@"เปิด Notifications"
                             message:@"เพื่อรับการแจ้งเตือน กรุณาไปที่ Settings → Notifications → [App Name] และเปิดใช้งาน"
                      preferredStyle:UIAlertControllerStyleAlert];
        
        [alert addAction:[UIAlertAction actionWithTitle:@"ไป Settings"
                                                  style:UIAlertActionStyleDefault
                                                handler:^(UIAlertAction *action) {
            NSURL *settingsURL = [NSURL URLWithString:UIApplicationOpenSettingsURLString];
            [[UIApplication sharedApplication] openURL:settingsURL 
                                               options:@{} 
                                     completionHandler:nil];
        }]];
        
        [alert addAction:[UIAlertAction actionWithTitle:@"ยกเลิก"
                                                  style:UIAlertActionStyleCancel
                                                handler:nil]];
        
        UIViewController *topVC = [UIApplication sharedApplication].keyWindow.rootViewController;
        [topVC presentViewController:alert animated:YES completion:nil];
    });
}
```

---

## 64.5 Local Notifications

Local Notifications สร้างและส่งโดย app เอง ไม่ต้องผ่าน server

### สร้าง Time-based Notification

```objc
- (void)scheduleReminderAfterMinutes:(NSInteger)minutes 
                               title:(NSString *)title 
                                body:(NSString *)body {
    
    // สร้าง content
    UNMutableNotificationContent *content = [[UNMutableNotificationContent alloc] init];
    content.title = title;
    content.body = body;
    content.sound = [UNNotificationSound defaultSound];
    content.badge = @1;
    
    // User info สำหรับส่งข้อมูลเพิ่มเติม
    content.userInfo = @{
        @"type": @"reminder",
        @"action": @"open_note"
    };
    
    // Trigger: หลังจากกี่วินาที
    NSTimeInterval timeInterval = minutes * 60.0;
    UNTimeIntervalNotificationTrigger *trigger = 
        [UNTimeIntervalNotificationTrigger triggerWithTimeInterval:timeInterval 
                                                          repeats:NO];
    
    // สร้าง request
    NSString *identifier = [NSUUID UUID].UUIDString;
    UNNotificationRequest *request = 
        [UNNotificationRequest requestWithIdentifier:identifier 
                                             content:content 
                                             trigger:trigger];
    
    // เพิ่มลงใน notification center
    [[UNUserNotificationCenter currentNotificationCenter] 
        addNotificationRequest:request 
         withCompletionHandler:^(NSError *error) {
        if (error) {
            NSLog(@"Failed to schedule notification: %@", error);
        } else {
            NSLog(@"Notification scheduled with ID: %@", identifier);
        }
    }];
}
```

### Calendar-based Notification

```objc
// Notification ทุกวันตอนเวลาที่กำหนด
- (void)scheduleDailyReminder:(NSString *)title 
                         body:(NSString *)body 
                         hour:(NSInteger)hour 
                       minute:(NSInteger)minute {
    
    UNMutableNotificationContent *content = [[UNMutableNotificationContent alloc] init];
    content.title = title;
    content.body = body;
    content.sound = [UNNotificationSound defaultSound];
    
    // กำหนดเวลา
    NSDateComponents *dateComponents = [[NSDateComponents alloc] init];
    dateComponents.hour = hour;
    dateComponents.minute = minute;
    
    // Trigger แบบ calendar
    UNCalendarNotificationTrigger *trigger = 
        [UNCalendarNotificationTrigger triggerWithDateMatchingComponents:dateComponents 
                                                                 repeats:YES];
    
    UNNotificationRequest *request = 
        [UNNotificationRequest requestWithIdentifier:@"daily_reminder" 
                                             content:content 
                                             trigger:trigger];
    
    // ลบ notification เดิมก่อน
    [[UNUserNotificationCenter currentNotificationCenter] 
        removePendingNotificationRequestsWithIdentifiers:@[@"daily_reminder"]];
    
    [[UNUserNotificationCenter currentNotificationCenter] 
        addNotificationRequest:request 
         withCompletionHandler:^(NSError *error) {
        if (error) {
            NSLog(@"Failed to schedule daily reminder: %@", error);
        }
    }];
}

// Notification เฉพาะวันใน week
- (void)scheduleWeekdayMorningBriefing {
    for (NSInteger weekday = 2; weekday <= 6; weekday++) { // 2=Monday, 6=Friday
        UNMutableNotificationContent *content = [[UNMutableNotificationContent alloc] init];
        content.title = @"สวัสดีตอนเช้า!";
        content.body = @"ดูสรุปงานประจำวันของคุณ";
        content.sound = [UNNotificationSound defaultSound];
        
        NSDateComponents *components = [[NSDateComponents alloc] init];
        components.hour = 8;
        components.minute = 0;
        components.weekday = weekday;
        
        UNCalendarNotificationTrigger *trigger = 
            [UNCalendarNotificationTrigger triggerWithDateMatchingComponents:components 
                                                                     repeats:YES];
        
        NSString *identifier = [NSString stringWithFormat:@"morning_brief_%ld", (long)weekday];
        UNNotificationRequest *request = 
            [UNNotificationRequest requestWithIdentifier:identifier 
                                                 content:content 
                                                 trigger:trigger];
        
        [[UNUserNotificationCenter currentNotificationCenter] 
            addNotificationRequest:request withCompletionHandler:nil];
    }
}
```

### Location-based Notification

```objc
#import <CoreLocation/CoreLocation.h>

- (void)scheduleNotificationAtLocation:(CLLocationCoordinate2D)coordinate 
                                radius:(CLLocationDistance)radius
                                 title:(NSString *)title
                                  body:(NSString *)body {
    
    UNMutableNotificationContent *content = [[UNMutableNotificationContent alloc] init];
    content.title = title;
    content.body = body;
    content.sound = [UNNotificationSound defaultSound];
    
    // สร้าง circular region
    CLCircularRegion *region = [[CLCircularRegion alloc] 
        initWithCenter:coordinate 
                radius:radius 
            identifier:@"target_location"];
    region.notifyOnEntry = YES;   // แจ้งเมื่อเข้าพื้นที่
    region.notifyOnExit = NO;
    
    // Location trigger
    UNLocationNotificationTrigger *trigger = 
        [UNLocationNotificationTrigger triggerWithRegion:region repeats:NO];
    
    UNNotificationRequest *request = 
        [UNNotificationRequest requestWithIdentifier:@"location_reminder" 
                                             content:content 
                                             trigger:trigger];
    
    [[UNUserNotificationCenter currentNotificationCenter] 
        addNotificationRequest:request 
         withCompletionHandler:^(NSError *error) {
        if (error) {
            NSLog(@"Failed to schedule location notification: %@", error);
        }
    }];
}
```

### จัดการ Pending Notifications

```objc
// ดูรายการ notifications ที่รอส่ง
- (void)listPendingNotifications {
    [[UNUserNotificationCenter currentNotificationCenter] 
        getPendingNotificationRequestsWithCompletionHandler:^(NSArray *requests) {
        
        NSLog(@"Pending notifications: %lu", (unsigned long)requests.count);
        for (UNNotificationRequest *request in requests) {
            NSLog(@"- ID: %@, Title: %@", request.identifier, request.content.title);
        }
    }];
}

// ยกเลิก notification เฉพาะตัว
- (void)cancelNotificationWithIdentifier:(NSString *)identifier {
    [[UNUserNotificationCenter currentNotificationCenter] 
        removePendingNotificationRequestsWithIdentifiers:@[identifier]];
}

// ยกเลิกทุก pending notifications
- (void)cancelAllNotifications {
    [[UNUserNotificationCenter currentNotificationCenter] 
        removeAllPendingNotificationRequests];
}

// ลบ delivered notifications (ที่แสดงแล้วแต่ยังอยู่ใน notification center)
- (void)clearDeliveredNotifications {
    [[UNUserNotificationCenter currentNotificationCenter] 
        removeAllDeliveredNotifications];
}
```

---

## 64.6 Push Notification Payload

Push notification ที่ส่งจาก server มาใน JSON format:

### Payload พื้นฐาน

```json
{
    "aps": {
        "alert": {
            "title": "New Message",
            "subtitle": "From John",
            "body": "Hey, how are you?"
        },
        "badge": 5,
        "sound": "default",
        "category": "MESSAGE",
        "thread-id": "chat-room-123",
        "mutable-content": 1,
        "content-available": 1
    },
    "custom_key": "custom_value",
    "message_id": "abc123",
    "sender_id": "user456"
}
```

### ประเภทของ Sound

```json
{
    "aps": {
        "sound": "default",
        "alert": "New notification"
    }
}

// Custom sound (ต้องมีไฟล์ในตัว app)
{
    "aps": {
        "sound": "notification.aiff",
        "alert": "Custom sound"
    }
}

// Critical Alert (ต้องขอ permission พิเศษ)
{
    "aps": {
        "sound": {
            "critical": 1,
            "name": "emergency.aiff",
            "volume": 1.0
        },
        "alert": "Emergency!"
    }
}
```

### ตัวอย่าง Payload ประเภทต่างๆ

```json
// Silent Push (ไม่แสดง UI, wake app ใน background)
{
    "aps": {
        "content-available": 1
    },
    "action": "refresh_data"
}

// Rich Notification (มีรูป/วิดีโอ/audio)
{
    "aps": {
        "alert": {
            "title": "New Photo",
            "body": "John shared a photo"
        },
        "mutable-content": 1,
        "category": "PHOTO"
    },
    "media_url": "https://example.com/photo.jpg",
    "media_type": "image"
}

// Notification with Actions
{
    "aps": {
        "alert": {
            "title": "New Task",
            "body": "Complete this task?"
        },
        "category": "TASK",
        "sound": "default"
    },
    "task_id": "task-789"
}
```

---

## 64.7 การจัดการ Notifications ใน Foreground และ Background

### UNUserNotificationCenterDelegate

```objc
@interface AppDelegate : UIResponder <UIApplicationDelegate, UNUserNotificationCenterDelegate>
@end

@implementation AppDelegate

// เรียกเมื่อ notification มาถึงขณะ app อยู่ใน Foreground
- (void)userNotificationCenter:(UNUserNotificationCenter *)center
       willPresentNotification:(UNNotification *)notification
         withCompletionHandler:(void (^)(UNNotificationPresentationOptions))completionHandler {
    
    UNNotificationContent *content = notification.request.content;
    NSLog(@"Foreground notification: %@", content.title);
    
    // ตัดสินใจว่าจะแสดง notification หรือไม่
    NSDictionary *userInfo = content.userInfo;
    NSString *notificationType = userInfo[@"type"];
    
    if ([notificationType isEqualToString:@"chat"]) {
        // ถ้าเป็น chat notification และอยู่ใน chat room นั้น → ไม่แสดง
        if ([self isCurrentlyInChatRoom:userInfo[@"room_id"]]) {
            completionHandler(UNNotificationPresentationOptionNone);
        } else {
            completionHandler(UNNotificationPresentationOptionBanner | 
                            UNNotificationPresentationOptionSound |
                            UNNotificationPresentationOptionBadge);
        }
    } else {
        // แสดง notification ตามปกติ
        if (@available(iOS 14.0, *)) {
            completionHandler(UNNotificationPresentationOptionBanner | 
                            UNNotificationPresentationOptionSound);
        } else {
            completionHandler(UNNotificationPresentationOptionAlert | 
                            UNNotificationPresentationOptionSound);
        }
    }
}

// เรียกเมื่อ user tap บน notification
- (void)userNotificationCenter:(UNUserNotificationCenter *)center
    didReceiveNotificationResponse:(UNNotificationResponse *)response
             withCompletionHandler:(void (^)(void))completionHandler {
    
    NSString *actionIdentifier = response.actionIdentifier;
    UNNotification *notification = response.notification;
    NSDictionary *userInfo = notification.request.content.userInfo;
    
    NSLog(@"User tapped notification: %@", notification.request.content.title);
    NSLog(@"Action: %@", actionIdentifier);
    
    // จัดการ action ต่างๆ
    if ([actionIdentifier isEqualToString:UNNotificationDefaultActionIdentifier]) {
        // User tap ที่ notification ปกติ (ไม่ใช่ action button)
        [self handleDefaultNotificationAction:userInfo];
    } else if ([actionIdentifier isEqualToString:@"ACCEPT_ACTION"]) {
        [self handleAcceptAction:userInfo];
    } else if ([actionIdentifier isEqualToString:@"DECLINE_ACTION"]) {
        [self handleDeclineAction:userInfo];
    } else if ([actionIdentifier isEqualToString:@"REPLY_ACTION"]) {
        // Text input action
        if ([response isKindOfClass:[UNTextInputNotificationResponse class]]) {
            UNTextInputNotificationResponse *textResponse = 
                (UNTextInputNotificationResponse *)response;
            NSString *replyText = textResponse.userText;
            NSLog(@"Reply text: %@", replyText);
            [self sendReply:replyText toMessageID:userInfo[@"message_id"]];
        }
    } else if ([actionIdentifier isEqualToString:UNNotificationDismissActionIdentifier]) {
        // User เลื่อนลบ notification
        NSLog(@"Notification dismissed");
    }
    
    completionHandler();
}

- (void)handleDefaultNotificationAction:(NSDictionary *)userInfo {
    NSString *type = userInfo[@"type"];
    
    dispatch_async(dispatch_get_main_queue(), ^{
        if ([type isEqualToString:@"chat"]) {
            // Navigate ไปยัง chat screen
            [[NSNotificationCenter defaultCenter] 
                postNotificationName:@"OpenChatRoom" 
                              object:nil 
                            userInfo:userInfo];
        } else if ([type isEqualToString:@"order"]) {
            // Navigate ไปยัง order detail
            [[NSNotificationCenter defaultCenter] 
                postNotificationName:@"OpenOrderDetail" 
                              object:nil 
                            userInfo:userInfo];
        }
    });
}

@end
```

### Remote Notification ใน Background

```objc
// รับ remote notification ขณะ app อยู่ใน background
- (void)application:(UIApplication *)application 
    didReceiveRemoteNotification:(NSDictionary *)userInfo 
          fetchCompletionHandler:(void (^)(UIBackgroundFetchResult))completionHandler {
    
    NSLog(@"Received remote notification in background/foreground");
    NSLog(@"UserInfo: %@", userInfo);
    
    NSString *action = userInfo[@"action"];
    
    if ([action isEqualToString:@"refresh_data"]) {
        // ดึงข้อมูลใหม่ (max 30 วินาที)
        [self fetchLatestData:^(BOOL success) {
            completionHandler(success ? UIBackgroundFetchResultNewData : UIBackgroundFetchResultFailed);
        }];
    } else {
        completionHandler(UIBackgroundFetchResultNoData);
    }
}

- (void)fetchLatestData:(void (^)(BOOL success))completion {
    NSURL *url = [NSURL URLWithString:@"https://api.yourserver.com/latest"];
    NSURLSession *session = [NSURLSession sharedSession];
    
    [[session dataTaskWithURL:url completionHandler:^(NSData *data, NSURLResponse *response, NSError *error) {
        if (error || !data) {
            completion(NO);
            return;
        }
        
        // Process data...
        NSError *parseError;
        NSDictionary *json = [NSJSONSerialization JSONObjectWithData:data options:0 error:&parseError];
        
        if (parseError) {
            completion(NO);
            return;
        }
        
        // บันทึกข้อมูลและ notify UI
        dispatch_async(dispatch_get_main_queue(), ^{
            // อัปเดต local cache
            [[NSNotificationCenter defaultCenter] postNotificationName:@"DataRefreshed"
                                                                object:json];
        });
        
        completion(YES);
    }] resume];
}
```

---

## 64.8 Rich Notifications

Rich notifications ทำให้ notification มีรูปภาพ วิดีโอ หรือ audio

### Notification Service Extension

สร้าง Extension เพื่อปรับแต่ง notification ก่อนแสดง:

1. ใน Xcode: File → New → Target → Notification Service Extension

```objc
// NotificationService.m
#import "NotificationService.h"

@interface NotificationService ()
@property (nonatomic, strong) void (^contentHandler)(UNNotificationContent *contentToDeliver);
@property (nonatomic, strong) UNMutableNotificationContent *bestAttemptContent;
@end

@implementation NotificationService

- (void)didReceiveNotificationRequest:(UNNotificationRequest *)request 
                   withContentHandler:(void (^)(UNNotificationContent *contentToDeliver))contentHandler {
    
    self.contentHandler = contentHandler;
    self.bestAttemptContent = [request.content mutableCopy];
    
    // แก้ไข notification content
    self.bestAttemptContent.title = 
        [NSString stringWithFormat:@"[Modified] %@", self.bestAttemptContent.title];
    
    // ดาวน์โหลดและแนบ media
    NSString *mediaURL = request.content.userInfo[@"media_url"];
    NSString *mediaType = request.content.userInfo[@"media_type"];
    
    if (mediaURL) {
        [self downloadMedia:mediaURL 
                       type:mediaType 
                 completion:^(UNNotificationAttachment *attachment) {
            if (attachment) {
                self.bestAttemptContent.attachments = @[attachment];
            }
            self.contentHandler(self.bestAttemptContent);
        }];
    } else {
        self.contentHandler(self.bestAttemptContent);
    }
}

- (void)downloadMedia:(NSString *)urlString 
                 type:(NSString *)mediaType 
           completion:(void (^)(UNNotificationAttachment *))completion {
    
    NSURL *url = [NSURL URLWithString:urlString];
    if (!url) {
        completion(nil);
        return;
    }
    
    NSURLSession *session = [NSURLSession sharedSession];
    [[session downloadTaskWithURL:url completionHandler:^(NSURL *location, NSURLResponse *response, NSError *error) {
        if (error || !location) {
            completion(nil);
            return;
        }
        
        // ตรวจสอบ file extension
        NSString *fileExt = @"jpg"; // default
        if ([mediaType isEqualToString:@"video"]) fileExt = @"mp4";
        else if ([mediaType isEqualToString:@"gif"]) fileExt = @"gif";
        else if ([mediaType isEqualToString:@"audio"]) fileExt = @"mp3";
        
        // ย้ายไฟล์ไปยัง temp location พร้อม extension ที่ถูกต้อง
        NSURL *tmpURL = [NSURL fileURLWithPath:[NSTemporaryDirectory() 
                         stringByAppendingPathComponent:[NSString stringWithFormat:@"attachment.%@", fileExt]]];
        
        NSError *moveError;
        [[NSFileManager defaultManager] moveItemAtURL:location toURL:tmpURL error:&moveError];
        
        if (moveError) {
            completion(nil);
            return;
        }
        
        // สร้าง attachment
        NSError *attachmentError;
        UNNotificationAttachment *attachment = 
            [UNNotificationAttachment attachmentWithIdentifier:@"media"
                                                          URL:tmpURL
                                                      options:nil
                                                        error:&attachmentError];
        
        completion(attachmentError ? nil : attachment);
    }] resume];
}

// เรียกเมื่อ timeout (30 วินาที)
- (void)serviceExtensionTimeWillExpire {
    // ส่ง content ที่มีอยู่แทน
    self.contentHandler(self.bestAttemptContent);
}

@end
```

### Notification Content Extension

สร้าง custom notification UI:

1. File → New → Target → Notification Content Extension

```objc
// NotificationViewController.m
#import "NotificationViewController.h"
#import <UserNotifications/UserNotifications.h>
#import <UserNotificationsUI/UserNotificationsUI.h>

@interface NotificationViewController () <UNNotificationContentExtension>

@property (nonatomic, weak) IBOutlet UILabel *titleLabel;
@property (nonatomic, weak) IBOutlet UILabel *bodyLabel;
@property (nonatomic, weak) IBOutlet UIImageView *mediaImageView;
@property (nonatomic, weak) IBOutlet UIProgressView *progressView;

@end

@implementation NotificationViewController

- (void)viewDidLoad {
    [super viewDidLoad];
}

// รับข้อมูล notification
- (void)didReceiveNotification:(UNNotification *)notification {
    UNNotificationContent *content = notification.request.content;
    
    self.titleLabel.text = content.title;
    self.bodyLabel.text = content.body;
    
    // แสดงรูปภาพถ้ามี attachment
    if (content.attachments.count > 0) {
        UNNotificationAttachment *attachment = content.attachments.firstObject;
        
        if ([attachment.URL startAccessingSecurityScopedResource]) {
            NSData *imageData = [NSData dataWithContentsOfURL:attachment.URL];
            self.mediaImageView.image = [UIImage imageWithData:imageData];
            [attachment.URL stopAccessingSecurityScopedResource];
        }
    }
    
    // ดึงข้อมูลจาก userInfo
    NSDictionary *userInfo = content.userInfo;
    NSNumber *progress = userInfo[@"progress"];
    if (progress) {
        self.progressView.progress = progress.floatValue;
    }
}

// จัดการ action จาก notification
- (void)didReceiveNotificationResponse:(UNNotificationResponse *)response 
                     completionHandler:(void (^)(UNNotificationContentExtensionResponseOption))completion {
    
    if ([response.actionIdentifier isEqualToString:@"LIKE_ACTION"]) {
        // อัปเดต UI แสดง liked state
        self.titleLabel.text = @"Liked! ❤️";
        completion(UNNotificationContentExtensionResponseOptionDoNotDismiss);
    } else if ([response.actionIdentifier isEqualToString:@"SHARE_ACTION"]) {
        // ส่งให้ app จัดการ
        completion(UNNotificationContentExtensionResponseOptionDismissAndForwardAction);
    } else {
        completion(UNNotificationContentExtensionResponseOptionDismiss);
    }
}

@end
```

---

## 64.9 Notification Actions

### สร้าง Custom Actions

```objc
// ลงทะเบียน notification categories และ actions
- (void)registerNotificationCategories {
    UNUserNotificationCenter *center = [UNUserNotificationCenter currentNotificationCenter];
    
    // Actions สำหรับ message
    UNNotificationAction *replyAction = [UNTextInputNotificationAction
        actionWithIdentifier:@"REPLY_ACTION"
                       title:@"ตอบกลับ"
                     options:UNNotificationActionOptionNone
        textInputButtonTitle:@"ส่ง"
        textInputPlaceholder:@"พิมพ์ข้อความ..."];
    
    UNNotificationAction *likeAction = [UNNotificationAction
        actionWithIdentifier:@"LIKE_ACTION"
                       title:@"ถูกใจ ❤️"
                     options:UNNotificationActionOptionNone];
    
    UNNotificationAction *deleteAction = [UNNotificationAction
        actionWithIdentifier:@"DELETE_ACTION"
                       title:@"ลบ"
                     options:UNNotificationActionOptionDestructive | 
                             UNNotificationActionOptionAuthenticationRequired];
    
    // Category สำหรับ message
    UNNotificationCategory *messageCategory = [UNNotificationCategory
        categoryWithIdentifier:@"MESSAGE"
                       actions:@[replyAction, likeAction, deleteAction]
             intentIdentifiers:@[]
                       options:UNNotificationCategoryOptionCustomDismissAction];
    
    // Actions สำหรับ task
    UNNotificationAction *completeAction = [UNNotificationAction
        actionWithIdentifier:@"COMPLETE_ACTION"
                       title:@"เสร็จแล้ว ✓"
                     options:UNNotificationActionOptionNone];
    
    UNNotificationAction *snoozeAction = [UNNotificationAction
        actionWithIdentifier:@"SNOOZE_ACTION"
                       title:@"เตือนใหม่ใน 1 ชั่วโมง"
                     options:UNNotificationActionOptionNone];
    
    UNNotificationCategory *taskCategory = [UNNotificationCategory
        categoryWithIdentifier:@"TASK"
                       actions:@[completeAction, snoozeAction]
             intentIdentifiers:@[]
                       options:UNNotificationCategoryOptionNone];
    
    // ลงทะเบียน categories
    [center setNotificationCategories:[NSSet setWithObjects:messageCategory, taskCategory, nil]];
}
```

### จัดการ Action ใน AppDelegate

```objc
- (void)userNotificationCenter:(UNUserNotificationCenter *)center
    didReceiveNotificationResponse:(UNNotificationResponse *)response
             withCompletionHandler:(void (^)(void))completionHandler {
    
    NSDictionary *userInfo = response.notification.request.content.userInfo;
    NSString *categoryID = response.notification.request.content.categoryIdentifier;
    NSString *actionID = response.actionIdentifier;
    
    if ([categoryID isEqualToString:@"MESSAGE"]) {
        if ([actionID isEqualToString:@"REPLY_ACTION"]) {
            UNTextInputNotificationResponse *textResponse = 
                (UNTextInputNotificationResponse *)response;
            NSString *replyText = textResponse.userText;
            NSString *messageID = userInfo[@"message_id"];
            
            // ส่ง reply โดยไม่ต้องเปิด app
            [self sendQuickReply:replyText toMessageID:messageID];
            
        } else if ([actionID isEqualToString:@"DELETE_ACTION"]) {
            NSString *messageID = userInfo[@"message_id"];
            [self deleteMessage:messageID];
        }
        
    } else if ([categoryID isEqualToString:@"TASK"]) {
        NSString *taskID = userInfo[@"task_id"];
        
        if ([actionID isEqualToString:@"COMPLETE_ACTION"]) {
            [self markTaskComplete:taskID];
        } else if ([actionID isEqualToString:@"SNOOZE_ACTION"]) {
            [self snoozeTask:taskID forHours:1];
        }
    }
    
    completionHandler();
}

- (void)sendQuickReply:(NSString *)text toMessageID:(NSString *)messageID {
    // ส่ง API request
    NSLog(@"Sending quick reply '%@' to message %@", text, messageID);
    // ... API call ...
}

- (void)snoozeTask:(NSString *)taskID forHours:(NSInteger)hours {
    NSLog(@"Snoozing task %@ for %ld hours", taskID, (long)hours);
    
    // ยกเลิก notification เดิม
    [[UNUserNotificationCenter currentNotificationCenter]
        removePendingNotificationRequestsWithIdentifiers:@[taskID]];
    
    // สร้าง notification ใหม่
    UNMutableNotificationContent *content = [[UNMutableNotificationContent alloc] init];
    content.title = @"งานที่ต้องทำ (เตือนซ้ำ)";
    content.body = [NSString stringWithFormat:@"Task %@ ยังไม่เสร็จ!", taskID];
    content.categoryIdentifier = @"TASK";
    content.userInfo = @{@"task_id": taskID};
    content.sound = [UNNotificationSound defaultSound];
    
    NSTimeInterval delay = hours * 3600.0;
    UNTimeIntervalNotificationTrigger *trigger = 
        [UNTimeIntervalNotificationTrigger triggerWithTimeInterval:delay repeats:NO];
    
    NSString *newID = [NSString stringWithFormat:@"%@_snooze", taskID];
    UNNotificationRequest *request = 
        [UNNotificationRequest requestWithIdentifier:newID content:content trigger:trigger];
    
    [[UNUserNotificationCenter currentNotificationCenter]
        addNotificationRequest:request withCompletionHandler:nil];
}
```

---

## 64.10 Silent Notifications

Silent notifications (content-available) wake app ใน background เพื่อดึงข้อมูล

### การ Setup Background Modes

ใน Info.plist เพิ่ม:
```xml
<key>UIBackgroundModes</key>
<array>
    <string>remote-notification</string>
</array>
```

หรือใน Xcode: Signing & Capabilities → Background Modes → Remote notifications

### ส่ง Silent Notification จาก Server

```json
{
    "aps": {
        "content-available": 1
    },
    "action": "sync_messages",
    "timestamp": 1695000000
}
```

### จัดการใน App

```objc
- (void)application:(UIApplication *)application 
    didReceiveRemoteNotification:(NSDictionary *)userInfo 
          fetchCompletionHandler:(void (^)(UIBackgroundFetchResult))completionHandler {
    
    NSString *action = userInfo[@"action"];
    NSLog(@"Silent notification: action = %@", action);
    
    if ([action isEqualToString:@"sync_messages"]) {
        [self syncNewMessages:^(NSInteger newCount) {
            if (newCount > 0) {
                // อัปเดต badge
                dispatch_async(dispatch_get_main_queue(), ^{
                    [UIApplication sharedApplication].applicationIconBadgeNumber = newCount;
                });
                completionHandler(UIBackgroundFetchResultNewData);
            } else {
                completionHandler(UIBackgroundFetchResultNoData);
            }
        }];
    } else if ([action isEqualToString:@"clear_cache"]) {
        [self clearLocalCache];
        completionHandler(UIBackgroundFetchResultNewData);
    } else {
        completionHandler(UIBackgroundFetchResultNoData);
    }
}

- (void)syncNewMessages:(void (^)(NSInteger))completion {
    NSString *lastSyncTime = [[NSUserDefaults standardUserDefaults] 
        stringForKey:@"last_sync_time"];
    
    NSURL *url = [NSURL URLWithString:
        [NSString stringWithFormat:@"https://api.yourserver.com/messages?since=%@", lastSyncTime]];
    
    [[NSURLSession.sharedSession dataTaskWithURL:url 
                              completionHandler:^(NSData *data, NSURLResponse *resp, NSError *error) {
        if (!data || error) {
            completion(0);
            return;
        }
        
        NSError *parseError;
        NSDictionary *json = [NSJSONSerialization JSONObjectWithData:data 
                                                            options:0 
                                                              error:&parseError];
        
        NSArray *messages = json[@"messages"];
        NSInteger count = messages.count;
        
        // บันทึกข้อมูลใหม่
        // ...
        
        // อัปเดต last sync time
        [[NSUserDefaults standardUserDefaults] setObject:json[@"timestamp"] forKey:@"last_sync_time"];
        
        completion(count);
    }] resume];
}
```

---

## 64.11 Background Fetch

Background Fetch เป็นอีกวิธีที่ iOS อนุญาตให้ app รันใน background

### การ Setup

ใน Info.plist:
```xml
<key>UIBackgroundModes</key>
<array>
    <string>fetch</string>
</array>
```

### Implementation

```objc
@implementation AppDelegate

- (BOOL)application:(UIApplication *)application 
    didFinishLaunchingWithOptions:(NSDictionary *)launchOptions {
    
    // กำหนด minimum fetch interval
    [application setMinimumBackgroundFetchInterval:UIApplicationBackgroundFetchIntervalMinimum];
    // หรือกำหนดเองเป็นวินาที:
    // [application setMinimumBackgroundFetchInterval:3600]; // 1 ชั่วโมง
    
    return YES;
}

// iOS เรียก method นี้เป็นระยะๆ ใน background
- (void)application:(UIApplication *)application 
    performFetchWithCompletionHandler:(void (^)(UIBackgroundFetchResult))completionHandler {
    
    NSLog(@"Background fetch started");
    NSDate *startTime = [NSDate date];
    
    [self fetchNewContent:^(BOOL hasNewContent) {
        NSTimeInterval elapsed = [[NSDate date] timeIntervalSinceDate:startTime];
        NSLog(@"Background fetch completed in %.2f seconds", elapsed);
        
        // ต้องเรียก completionHandler ภายใน 30 วินาที!
        completionHandler(hasNewContent ? UIBackgroundFetchResultNewData : UIBackgroundFetchResultNoData);
    }];
}

- (void)fetchNewContent:(void (^)(BOOL))completion {
    NSURL *url = [NSURL URLWithString:@"https://api.yourserver.com/feed"];
    
    NSURLSessionConfiguration *config = [NSURLSessionConfiguration ephemeralSessionConfiguration];
    config.timeoutIntervalForRequest = 25; // น้อยกว่า 30 วินาที
    NSURLSession *session = [NSURLSession sessionWithConfiguration:config];
    
    [[session dataTaskWithURL:url completionHandler:^(NSData *data, NSURLResponse *resp, NSError *err) {
        if (!data || err) {
            completion(NO);
            return;
        }
        
        // ตรวจสอบว่ามีข้อมูลใหม่หรือไม่
        NSDictionary *json = [NSJSONSerialization JSONObjectWithData:data options:0 error:nil];
        BOOL hasNew = [json[@"has_new"] boolValue];
        
        if (hasNew) {
            // บันทึกข้อมูลใหม่
            // อัปเดต local database
        }
        
        completion(hasNew);
    }] resume];
}

@end
```

---

## 64.12 แบบฝึกหัด

### แบบฝึกหัดที่ 1: Notification Manager

สร้าง `NotificationManager` class ที่จัดการทุกอย่างเกี่ยวกับ notifications:

```objc
@interface NotificationManager : NSObject

+ (instancetype)sharedManager;

// Permission
- (void)requestPermission:(void (^)(BOOL granted))completion;
- (void)checkPermissionStatus:(void (^)(UNAuthorizationStatus status))completion;

// Local Notifications
- (NSString *)scheduleNotification:(NSString *)title 
                              body:(NSString *)body 
                              date:(NSDate *)date 
                          userInfo:(NSDictionary *)userInfo;
- (void)cancelNotification:(NSString *)identifier;
- (void)cancelAllNotifications;

// Categories & Actions
- (void)registerCategories;

// Badge
- (void)updateBadgeCount:(NSInteger)count;
- (void)clearBadge;

@end
```

### แบบฝึกหัดที่ 2: Chat App Notifications

สร้างระบบ notification สำหรับ chat app:
- แสดง rich notification พร้อมรูป avatar
- Quick reply action โดยไม่ต้องเปิด app
- Group notifications ด้วย thread-id
- ล้าง notifications เมื่อเข้าดู conversation

### แบบฝึกหัดที่ 3: Reminder App

สร้าง reminder app ที่:
- สร้าง reminders แบบ time-based และ location-based
- รองรับ repeat (daily, weekly, monthly)
- Snooze feature จาก notification
- Sync reminders ผ่าน iCloud

### แบบฝึกหัดที่ 4: News App Background Sync

สร้างระบบ background sync สำหรับ news app:
- ใช้ silent push + background fetch
- Cache articles สำหรับ offline reading
- อัปเดต badge count
- แสดง notification เมื่อมี breaking news

---

## สรุป

ในบทนี้เราได้เรียนรู้:

1. **APNs Architecture**: วิธีทำงานของ push notification system
2. **Registration**: ขอ permission และ device token
3. **UNUserNotificationCenter**: center หลักสำหรับ notifications
4. **Local Notifications**: สร้าง notifications จาก app เอง
5. **Push Payload**: JSON format สำหรับ push notifications
6. **Foreground/Background Handling**: จัดการ notifications ในสถานะต่างๆ
7. **Rich Notifications**: เพิ่ม media ด้วย Service Extension
8. **Notification Actions**: buttons และ text input actions
9. **Silent Notifications**: wake app ใน background
10. **Background Fetch**: ดึงข้อมูลสม่ำเสมอใน background

Push Notifications เป็นเครื่องมือทรงพลังที่ช่วยเพิ่ม engagement ของ users แต่ควรใช้อย่างระมัดระวัง ไม่ส่ง notifications มากเกินไปจนรบกวน user
