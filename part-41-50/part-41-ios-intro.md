# ตอนที่ 41: การพัฒนาแอปพลิเคชัน iOS เบื้องต้น (iOS Development Introduction)

## บทนำ

การพัฒนาแอปพลิเคชัน iOS เป็นหนึ่งในทักษะที่มีคุณค่าสูงในวงการพัฒนาซอฟต์แวร์ปัจจุบัน Apple มี ecosystem ที่แข็งแกร่งประกอบด้วยอุปกรณ์ iPhone, iPad, iPod touch และ Apple TV ซึ่งทั้งหมดใช้งานระบบปฏิบัติการ iOS หรือ iPadOS ที่พัฒนาต่อมาจาก iOS

ในตอนนี้เราจะเรียนรู้พื้นฐานของการพัฒนาแอป iOS ตั้งแต่โครงสร้างของโปรเจกต์ไปจนถึงการสร้างและรันแอปพลิเคชันแรกของเรา

---

## 41.1 โครงสร้างแอป iOS (iOS App Structure)

### ภาพรวมของแอป iOS

แอปพลิเคชัน iOS มีโครงสร้างที่ชัดเจนและสม่ำเสมอ ประกอบด้วยไฟล์และโฟลเดอร์หลักดังนี้:

```
MyApp/
├── MyApp/
│   ├── AppDelegate.h
│   ├── AppDelegate.m
│   ├── SceneDelegate.h       (iOS 13+)
│   ├── SceneDelegate.m       (iOS 13+)
│   ├── ViewController.h
│   ├── ViewController.m
│   ├── Main.storyboard
│   ├── Assets.xcassets/
│   ├── LaunchScreen.storyboard
│   └── Info.plist
├── MyApp.xcodeproj/
├── MyAppTests/
└── MyAppUITests/
```

### บทบาทของแต่ละไฟล์

| ไฟล์ | หน้าที่ |
|------|---------|
| `AppDelegate.m` | จัดการ lifecycle ของแอปพลิเคชัน |
| `SceneDelegate.m` | จัดการ UI lifecycle (iOS 13+) |
| `ViewController.m` | ควบคุม View หลักของแอป |
| `Main.storyboard` | Interface Builder สำหรับออกแบบ UI |
| `Assets.xcassets` | เก็บรูปภาพและทรัพยากร |
| `Info.plist` | การตั้งค่าและข้อมูลของแอป |

---

## 41.2 การตั้งค่าโปรเจกต์ Xcode สำหรับ iOS (Xcode Project Setup for iOS)

### การสร้างโปรเจกต์ใหม่

1. เปิด Xcode และเลือก **Create a new Xcode project**
2. เลือก **iOS** → **App**
3. กรอกข้อมูลโปรเจกต์:
   - **Product Name**: ชื่อแอปของคุณ
   - **Organization Identifier**: com.yourcompany (เช่น com.example)
   - **Bundle Identifier**: จะสร้างอัตโนมัติจาก Organization Identifier + Product Name
   - **Interface**: Storyboard
   - **Language**: Objective-C
4. เลือกตำแหน่งที่บันทึกโปรเจกต์

### โครงสร้างโปรเจกต์ใน Xcode

```
Navigator Area (ซ้าย)     Editor Area (กลาง)     Inspector Area (ขวา)
     |                          |                        |
  Project                   ไฟล์ที่เปิดอยู่           Attributes/
  Navigator                                           Properties
```

### การตั้งค่า Build Settings พื้นฐาน

```objc
// ตรวจสอบ iOS Deployment Target ใน Build Settings
// IPHONEOS_DEPLOYMENT_TARGET = 15.0

// ตั้งค่า Bundle Identifier
// PRODUCT_BUNDLE_IDENTIFIER = com.example.myapp

// เปิดใช้ Automatic Signing
// CODE_SIGN_STYLE = Automatic
```

---

## 41.3 Info.plist

### Info.plist คืออะไร?

`Info.plist` (Information Property List) คือไฟล์ XML ที่เก็บการตั้งค่าและข้อมูลสำคัญของแอปพลิเคชัน iOS อ่านโดย iOS runtime เพื่อกำหนดพฤติกรรมของแอป

### โครงสร้างพื้นฐานของ Info.plist

```xml
<?xml version="1.0" encoding="UTF-8"?>
<!DOCTYPE plist PUBLIC "-//Apple//DTD PLIST 1.0//EN" 
    "http://www.apple.com/DTDs/PropertyList-1.0.dtd">
<plist version="1.0">
<dict>
    <!-- ชื่อแอปที่แสดงบนหน้าจอ -->
    <key>CFBundleDisplayName</key>
    <string>My App</string>
    
    <!-- Bundle Identifier (unique ID ของแอป) -->
    <key>CFBundleIdentifier</key>
    <string>$(PRODUCT_BUNDLE_IDENTIFIER)</string>
    
    <!-- เวอร์ชันของแอป -->
    <key>CFBundleShortVersionString</key>
    <string>1.0</string>
    
    <!-- Build number -->
    <key>CFBundleVersion</key>
    <string>1</string>
    
    <!-- ชื่อไฟล์ Storyboard หลัก -->
    <key>UIMainStoryboardFile</key>
    <string>Main</string>
    
    <!-- รองรับทิศทางการหมุนหน้าจอ -->
    <key>UISupportedInterfaceOrientations</key>
    <array>
        <string>UIInterfaceOrientationPortrait</string>
        <string>UIInterfaceOrientationLandscapeLeft</string>
        <string>UIInterfaceOrientationLandscapeRight</string>
    </array>
    
    <!-- สีของ Status Bar -->
    <key>UIStatusBarStyle</key>
    <string>UIStatusBarStyleDefault</string>
    
    <!-- ไม่ใช้ Status Bar -->
    <key>UIStatusBarHidden</key>
    <false/>
    
    <!-- Requires full screen (iPad) -->
    <key>UIRequiresFullScreen</key>
    <true/>
    
    <!-- สิทธิ์การเข้าถึงกล้อง (ต้องระบุเหตุผล) -->
    <key>NSCameraUsageDescription</key>
    <string>แอปต้องการใช้กล้องเพื่อถ่ายภาพ</string>
    
    <!-- สิทธิ์การเข้าถึง Photo Library -->
    <key>NSPhotoLibraryUsageDescription</key>
    <string>แอปต้องการเข้าถึงคลังภาพของคุณ</string>
    
    <!-- สิทธิ์การเข้าถึง Location -->
    <key>NSLocationWhenInUseUsageDescription</key>
    <string>แอปต้องการตำแหน่งของคุณ</string>
    
</dict>
</plist>
```

### การอ่านค่าจาก Info.plist ใน Code

```objc
// ดึงค่าจาก Info.plist
NSDictionary *infoDictionary = [[NSBundle mainBundle] infoDictionary];

// ดึงชื่อแอป
NSString *appName = infoDictionary[@"CFBundleDisplayName"];
NSLog(@"App Name: %@", appName);

// ดึงเวอร์ชัน
NSString *version = infoDictionary[@"CFBundleShortVersionString"];
NSString *build = infoDictionary[@"CFBundleVersion"];
NSLog(@"Version: %@ (Build: %@)", version, build);

// ดึง Bundle ID
NSString *bundleID = [[NSBundle mainBundle] bundleIdentifier];
NSLog(@"Bundle ID: %@", bundleID);
```

### Keys สำคัญใน Info.plist

```objc
// Minimum iOS Version
// MinimumOSVersion = "15.0"

// Required device capabilities
// UIRequiredDeviceCapabilities = {
//     "armv7" = true,
//     "telephony" = true  // ต้องเป็นโทรศัพท์
// }

// App Transport Security (ATS)
// NSAppTransportSecurity = {
//     NSAllowsArbitraryLoads = true  // อนุญาต HTTP (ไม่แนะนำ)
// }

// Background modes
// UIBackgroundModes = [
//     "audio",           // เล่นเสียงตอน background
//     "location",        // ติดตามตำแหน่ง background
//     "fetch",           // Background fetch
//     "remote-notification"  // Push notifications
// ]
```

---

## 41.4 AppDelegate

### AppDelegate คืออะไร?

`AppDelegate` คือ class หลักที่ทำหน้าที่เป็น delegate ของ `UIApplication` จัดการ lifecycle ของทั้งแอปพลิเคชัน เป็นจุดแรกที่ iOS เรียกเมื่อแอปเริ่มทำงาน

### AppDelegate.h

```objc
// AppDelegate.h

#import <UIKit/UIKit.h>

@interface AppDelegate : UIResponder <UIApplicationDelegate>

// สำหรับ iOS 12 และต่ำกว่า
@property (strong, nonatomic) UIWindow *window;

@end
```

### AppDelegate.m - การ Implement Methods หลัก

```objc
// AppDelegate.m

#import "AppDelegate.h"

@implementation AppDelegate

// เรียกเมื่อแอปเปิดขึ้นครั้งแรก
- (BOOL)application:(UIApplication *)application 
    didFinishLaunchingWithOptions:(NSDictionary *)launchOptions {
    
    NSLog(@"แอปเริ่มทำงาน");
    
    // ตั้งค่าเริ่มต้นสำหรับ UserDefaults
    [[NSUserDefaults standardUserDefaults] 
        registerDefaults:@{
            @"firstLaunch": @YES,
            @"theme": @"light"
        }];
    
    // ตั้งค่า UIWindow สำหรับ iOS 12 และต่ำกว่า
    // (iOS 13+ จะใช้ SceneDelegate แทน)
    if (@available(iOS 13.0, *)) {
        // Scene-based lifecycle จะจัดการเอง
    } else {
        self.window = [[UIWindow alloc] initWithFrame:
            [[UIScreen mainScreen] bounds]];
        
        UIViewController *rootVC = [[UIViewController alloc] init];
        self.window.rootViewController = rootVC;
        [self.window makeKeyAndVisible];
    }
    
    return YES;
}

// เรียกเมื่อแอปถูกย่อหน้าจอ (inactive → background)
- (void)applicationDidEnterBackground:(UIApplication *)application {
    NSLog(@"แอปเข้าสู่ Background");
    
    // บันทึกข้อมูลสำคัญ
    [[NSUserDefaults standardUserDefaults] synchronize];
    
    // หยุดงานที่ไม่จำเป็น
    // เช่น หยุด timer, หยุด animation
}

// เรียกเมื่อแอปกลับมา foreground
- (void)applicationWillEnterForeground:(UIApplication *)application {
    NSLog(@"แอปกำลังกลับมา Foreground");
    
    // เตรียมความพร้อม เช่น รีเฟรชข้อมูล
}

// เรียกเมื่อแอปทำงานปกติ (active)
- (void)applicationDidBecomeActive:(UIApplication *)application {
    NSLog(@"แอป Active แล้ว");
    
    // เริ่มงานที่หยุดไป
    // เช่น เริ่ม timer ใหม่
}

// เรียกเมื่อแอปกำลังจะ inactive
- (void)applicationWillResignActive:(UIApplication *)application {
    NSLog(@"แอปกำลัง Resign Active");
    
    // หยุดงานชั่วคราว เช่น หยุดเกม
}

// เรียกเมื่อแอปกำลังจะถูกปิด
- (void)applicationWillTerminate:(UIApplication *)application {
    NSLog(@"แอปกำลังจะปิด");
    
    // บันทึกข้อมูลสุดท้าย
    // ล้าง resources
}

// รับ Push Notification token
- (void)application:(UIApplication *)application 
    didRegisterForRemoteNotificationsWithDeviceToken:(NSData *)deviceToken {
    
    NSString *token = [self tokenStringFromData:deviceToken];
    NSLog(@"Device Token: %@", token);
    
    // ส่ง token ไปยัง server
}

// กรณีสมัคร Push Notification ล้มเหลว
- (void)application:(UIApplication *)application 
    didFailToRegisterForRemoteNotificationsWithError:(NSError *)error {
    
    NSLog(@"Failed to register: %@", error.localizedDescription);
}

// เปิดแอปจาก URL Scheme
- (BOOL)application:(UIApplication *)application 
    openURL:(NSURL *)url 
    options:(NSDictionary<UIApplicationOpenURLOptionsKey,id> *)options {
    
    NSLog(@"เปิดจาก URL: %@", url.absoluteString);
    
    // จัดการ URL scheme
    if ([url.scheme isEqualToString:@"myapp"]) {
        // จัดการ deep link
        return YES;
    }
    
    return NO;
}

#pragma mark - Helper Methods

- (NSString *)tokenStringFromData:(NSData *)data {
    const unsigned char *buffer = data.bytes;
    NSMutableString *hex = [NSMutableString stringWithCapacity:data.length * 2];
    for (int i = 0; i < data.length; i++) {
        [hex appendFormat:@"%02x", buffer[i]];
    }
    return hex;
}

@end
```

---

## 41.5 SceneDelegate (iOS 13+)

### SceneDelegate คืออะไร?

ตั้งแต่ iOS 13 Apple แนะนำ Scene-based lifecycle ซึ่งช่วยให้แอปรองรับ multiple windows บน iPad ได้ `SceneDelegate` จัดการ UI lifecycle แยกจาก `AppDelegate`

### SceneDelegate.h

```objc
// SceneDelegate.h

#import <UIKit/UIKit.h>

@interface SceneDelegate : UIResponder <UIWindowSceneDelegate>

@property (strong, nonatomic) UIWindow *window;

@end
```

### SceneDelegate.m

```objc
// SceneDelegate.m

#import "SceneDelegate.h"
#import "ViewController.h"

@implementation SceneDelegate

// เรียกเมื่อ Scene ใหม่ถูกสร้าง (เชื่อมต่อครั้งแรก)
- (void)scene:(UIScene *)scene 
    willConnectToSession:(UISceneSession *)session 
    options:(UISceneConnectionOptions *)connectionOptions {
    
    // ตรวจสอบว่าเป็น UIWindowScene
    if (![scene isKindOfClass:[UIWindowScene class]]) return;
    
    UIWindowScene *windowScene = (UIWindowScene *)scene;
    
    // สร้าง Window
    self.window = [[UIWindow alloc] initWithWindowScene:windowScene];
    
    // ตั้งค่า Root View Controller
    ViewController *rootVC = [[ViewController alloc] init];
    UINavigationController *navController = 
        [[UINavigationController alloc] initWithRootViewController:rootVC];
    
    self.window.rootViewController = navController;
    [self.window makeKeyAndVisible];
    
    // ตรวจสอบว่าเปิดจาก URL
    UIOpenURLContext *urlContext = connectionOptions.URLContexts.anyObject;
    if (urlContext) {
        NSURL *url = urlContext.URL;
        NSLog(@"เปิดจาก URL: %@", url);
    }
}

// เรียกเมื่อ Scene ถูกยกเลิกการเชื่อมต่อ (จะถูก deallocate)
- (void)sceneDidDisconnect:(UIScene *)scene {
    NSLog(@"Scene ถูก Disconnect");
    // ปล่อย resources ที่ไม่จำเป็น
}

// เรียกเมื่อ Scene กลายเป็น active
- (void)sceneDidBecomeActive:(UIScene *)scene {
    NSLog(@"Scene กลายเป็น Active");
    // เริ่มงานที่หยุดไว้
}

// เรียกเมื่อ Scene กำลังจะ resign active
- (void)sceneWillResignActive:(UIScene *)scene {
    NSLog(@"Scene กำลัง Resign Active");
    // หยุดงานชั่วคราว
}

// เรียกเมื่อ Scene กำลังจะเข้า foreground
- (void)sceneWillEnterForeground:(UIScene *)scene {
    NSLog(@"Scene กำลังจะเข้า Foreground");
}

// เรียกเมื่อ Scene เข้า background
- (void)sceneDidEnterBackground:(UIScene *)scene {
    NSLog(@"Scene เข้า Background");
    
    // บันทึกข้อมูล
    [[NSUserDefaults standardUserDefaults] synchronize];
}

// รับ URL เมื่อแอปเปิดอยู่แล้ว
- (void)scene:(UIScene *)scene 
    openURLContexts:(NSSet<UIOpenURLContext *> *)URLContexts {
    
    for (UIOpenURLContext *context in URLContexts) {
        NSURL *url = context.URL;
        NSLog(@"ได้รับ URL: %@", url);
        // จัดการ deep link
    }
}

@end
```

### การตั้งค่า Scene ใน Info.plist

```xml
<!-- Info.plist - Application Scene Manifest -->
<key>UIApplicationSceneManifest</key>
<dict>
    <key>UIApplicationSupportsMultipleScenes</key>
    <false/>
    <key>UISceneConfigurations</key>
    <dict>
        <key>UIWindowSceneSessionRoleApplication</key>
        <array>
            <dict>
                <key>UISceneConfigurationName</key>
                <string>Default Configuration</string>
                <key>UISceneDelegateClassName</key>
                <string>$(PRODUCT_MODULE_NAME).SceneDelegate</string>
                <key>UISceneStoryboardFile</key>
                <string>Main</string>
            </dict>
        </array>
    </dict>
</dict>
```

---

## 41.6 Application Lifecycle (วงจรชีวิตของแอปพลิเคชัน)

### States ของแอปพลิเคชัน

```
Not Running
     ↓ (Launch)
  Inactive ←→ Active
     ↓ (Home button / Swipe up)
 Background
     ↓ (System terminates)
 Suspended
     ↓
Not Running
```

### รายละเอียด States

```
State           | คำอธิบาย
----------------|--------------------------------------------------
Not Running     | แอปไม่ได้ทำงาน หรือถูก terminate โดย OS
Inactive        | แอปทำงาน foreground แต่ไม่รับ events
                | (เช่น ระหว่างรับสายโทรศัพท์)
Active          | แอปทำงาน foreground และรับ events ปกติ
Background      | แอปทำงาน background มีเวลาจำกัด
Suspended       | แอปอยู่ใน memory แต่ไม่ได้ execute code
```

### การจัดการ Lifecycle

```objc
// ตัวอย่างการจัดการ State Changes

@implementation AppDelegate

- (BOOL)application:(UIApplication *)application 
    didFinishLaunchingWithOptions:(NSDictionary *)launchOptions {
    
    // ตรวจสอบว่าเปิดจาก notification
    NSDictionary *notification = 
        launchOptions[UIApplicationLaunchOptionsRemoteNotificationKey];
    if (notification) {
        NSLog(@"เปิดจาก Remote Notification");
    }
    
    // ตรวจสอบว่าเปิดจาก URL
    NSURL *url = launchOptions[UIApplicationLaunchOptionsURLKey];
    if (url) {
        NSLog(@"เปิดจาก URL: %@", url);
    }
    
    return YES;
}

// Background Task - ขอเวลาเพิ่มเติม
- (void)applicationDidEnterBackground:(UIApplication *)application {
    
    UIBackgroundTaskIdentifier taskID = 
        [application beginBackgroundTaskWithName:@"FinishTask" 
                                 expirationHandler:^{
        NSLog(@"Background task หมดเวลา");
        [application endBackgroundTask:UIBackgroundTaskInvalid];
    }];
    
    // ทำงานสำคัญ
    dispatch_async(dispatch_get_global_queue(DISPATCH_QUEUE_PRIORITY_DEFAULT, 0), ^{
        // บันทึกข้อมูล, sync กับ server
        NSLog(@"ทำงาน background...");
        
        // เสร็จแล้ว
        [application endBackgroundTask:taskID];
    });
}

@end
```

---

## 41.7 UIApplication

### UIApplication คืออะไร?

`UIApplication` คือ singleton class ที่เป็นตัวแทนของแอปพลิเคชัน มีเพียง instance เดียวในแอป เรียกใช้ผ่าน `[UIApplication sharedApplication]`

### การใช้งาน UIApplication

```objc
// เข้าถึง shared instance
UIApplication *app = [UIApplication sharedApplication];

// ตรวจสอบ application state
UIApplicationState state = app.applicationState;
switch (state) {
    case UIApplicationStateActive:
        NSLog(@"App กำลัง Active");
        break;
    case UIApplicationStateInactive:
        NSLog(@"App กำลัง Inactive");
        break;
    case UIApplicationStateBackground:
        NSLog(@"App กำลัง Background");
        break;
}

// เปิด URL ใน browser
NSURL *url = [NSURL URLWithString:@"https://www.apple.com"];
if ([app canOpenURL:url]) {
    [app openURL:url 
         options:@{} 
completionHandler:^(BOOL success) {
        NSLog(@"เปิด URL: %@", success ? @"สำเร็จ" : @"ล้มเหลว");
    }];
}

// เปิดแอปตั้งค่า
NSURL *settingsURL = [NSURL URLWithString:UIApplicationOpenSettingsURLString];
[app openURL:settingsURL options:@{} completionHandler:nil];

// Badge count บน icon แอป
app.applicationIconBadgeNumber = 5;

// ขอสมัคร Push Notifications
[app registerForRemoteNotifications];

// ตรวจสอบสิทธิ์ Push Notifications
UIUserNotificationSettings *settings = 
    [app currentUserNotificationSettings];
NSLog(@"Notification types: %lu", settings.types);

// Network Activity Indicator (deprecated iOS 13)
// app.networkActivityIndicatorVisible = YES;

// Idle Timer (ป้องกันหน้าจอดับ)
app.idleTimerDisabled = YES;

// ดึง status bar frame
CGRect statusBarFrame = [app statusBarFrame];
NSLog(@"Status bar height: %.0f", statusBarFrame.size.height);

// Key window
UIWindow *keyWindow = nil;
for (UIWindow *window in app.windows) {
    if (window.isKeyWindow) {
        keyWindow = window;
        break;
    }
}

// ดึง AppDelegate
AppDelegate *delegate = (AppDelegate *)app.delegate;
```

### UIApplication Notifications

```objc
// สมัครรับ Notifications จาก UIApplication

// แอปกลายเป็น Active
[[NSNotificationCenter defaultCenter] 
    addObserver:self
       selector:@selector(appBecameActive:)
           name:UIApplicationDidBecomeActiveNotification
         object:nil];

// แอปเข้า Background
[[NSNotificationCenter defaultCenter]
    addObserver:self
       selector:@selector(appEnteredBackground:)
           name:UIApplicationDidEnterBackgroundNotification
         object:nil];

// หน้าจอถูกล็อค
[[NSNotificationCenter defaultCenter]
    addObserver:self
       selector:@selector(screenLocked:)
           name:UIApplicationProtectedDataWillBecomeUnavailable
         object:nil];

// Memory Warning
[[NSNotificationCenter defaultCenter]
    addObserver:self
       selector:@selector(memoryWarning:)
           name:UIApplicationDidReceiveMemoryWarningNotification
         object:nil];

- (void)appBecameActive:(NSNotification *)notification {
    NSLog(@"แอป Active แล้ว - จาก Notification");
}

- (void)appEnteredBackground:(NSNotification *)notification {
    NSLog(@"แอปเข้า Background - จาก Notification");
}

- (void)memoryWarning:(NSNotification *)notification {
    NSLog(@"คำเตือนหน่วยความจำ!");
    // ล้าง cache
}
```

---

## 41.8 Target และ Deployment Settings

### Deployment Target

Deployment Target กำหนดว่าแอปของเราจะรองรับ iOS เวอร์ชันต่ำสุดที่เท่าไหร่

```objc
// ตรวจสอบ iOS Version ใน code
if (@available(iOS 16.0, *)) {
    // ใช้ feature ของ iOS 16
    NSLog(@"ใช้ iOS 16 feature");
} else {
    // Fallback สำหรับ iOS เวอร์ชันเก่า
    NSLog(@"ใช้ feature เดิม");
}

// ตรวจสอบแบบเก่า (deprecated)
// if ([[UIDevice currentDevice].systemVersion floatValue] >= 16.0) { }

// การตรวจสอบ Runtime
NSOperatingSystemVersion osVersion = 
    [[NSProcessInfo processInfo] operatingSystemVersion];
NSLog(@"iOS Version: %ld.%ld.%ld", 
      osVersion.majorVersion, 
      osVersion.minorVersion, 
      osVersion.patchVersion);

// ตรวจสอบ iOS Version สำหรับ Feature
BOOL isIOS16OrLater = [[NSProcessInfo processInfo] 
    isOperatingSystemAtLeastVersion:(NSOperatingSystemVersion){16, 0, 0}];

if (isIOS16OrLater) {
    NSLog(@"iOS 16 หรือใหม่กว่า");
}
```

### การตั้งค่า Build Configurations

```
Debug Configuration:
- Optimization: None [-O0]
- Debug Information: DWARF with dSYM
- Assertions: Enabled

Release Configuration:
- Optimization: Fastest [-O2] หรือ Smallest [-Os]  
- Debug Information: DWARF with dSYM (stripped from binary)
- Assertions: Disabled
```

```objc
// ใช้ Preprocessor Macros
#ifdef DEBUG
    #define DLog(fmt, ...) NSLog((@"DEBUG: " fmt), ##__VA_ARGS__)
#else
    #define DLog(...)
#endif

// ใช้ใน code
DLog(@"ข้อมูล debug: %@", someVariable); // จะแสดงเฉพาะ Debug build
```

### Capabilities และ Entitlements

```
Project → Signing & Capabilities → + Capability

Capabilities สำคัญ:
- Push Notifications     : รับ push notifications
- Background Modes       : ทำงาน background
- In-App Purchase        : ซื้อในแอป
- Sign In with Apple     : Login ด้วย Apple ID
- HealthKit              : เข้าถึงข้อมูลสุขภาพ
- CloudKit               : ใช้ iCloud storage
- Maps                   : ใช้ Apple Maps
- Wallet                 : Passbook/Wallet
- NFC Tag Reading        : อ่าน NFC
- App Groups             : แชร์ data ระหว่างแอป
```

---

## 41.9 Simulator vs Device

### iOS Simulator

iOS Simulator คือโปรแกรมจำลอง iOS ที่มากับ Xcode ใช้สำหรับทดสอบแอปโดยไม่ต้องมีอุปกรณ์จริง

```objc
// ตรวจสอบว่ากำลังรันบน Simulator
#if TARGET_OS_SIMULATOR
    NSLog(@"กำลังรันบน Simulator");
    // ทำสิ่งพิเศษสำหรับ Simulator
#else
    NSLog(@"กำลังรันบนอุปกรณ์จริง");
#endif

// ข้อจำกัดของ Simulator:
// - ไม่รองรับ Camera (ใช้ภาพจาก photo library แทน)
// - ประสิทธิภาพต่างจากอุปกรณ์จริง
// - ไม่รองรับ Push Notifications
// - ไม่รองรับ Bluetooth
// - ไม่รองรับ Accelerometer/Gyroscope
// - ไม่รองรับ NFC
```

### การทดสอบบนอุปกรณ์จริง

```
ขั้นตอนเชื่อมต่อ iPhone กับ Xcode:
1. เชื่อมต่อ iPhone ผ่าน USB หรือ WiFi
2. เปิด Xcode → Window → Devices and Simulators
3. ตรวจสอบอุปกรณ์ปรากฏใน list
4. เลือก Device ใน Xcode toolbar
5. Pair ด้วย Trust dialog บน iPhone
6. Build & Run
```

```objc
// ข้อมูลอุปกรณ์
UIDevice *device = [UIDevice currentDevice];
NSLog(@"Model: %@", device.model);
NSLog(@"System Name: %@", device.systemName);
NSLog(@"System Version: %@", device.systemVersion);
NSLog(@"Name: %@", device.name);
NSLog(@"Identifier For Vendor: %@", 
      device.identifierForVendor.UUIDString);

// ขนาดหน้าจอ
CGRect screenBounds = [[UIScreen mainScreen] bounds];
CGFloat scale = [[UIScreen mainScreen] scale];
CGFloat nativeScale = [[UIScreen mainScreen] nativeScale];

NSLog(@"Screen size: %.0f x %.0f", 
      screenBounds.size.width, 
      screenBounds.size.height);
NSLog(@"Scale: %.1f", scale);
NSLog(@"Physical pixels: %.0f x %.0f", 
      screenBounds.size.width * scale,
      screenBounds.size.height * scale);
```

### ขนาดหน้าจอ iPhone และ iPad

```
iPhone Sizes (Points):
- iPhone SE (3rd gen)  : 375 × 667  @2x
- iPhone 14           : 390 × 844  @3x
- iPhone 14 Plus      : 428 × 926  @3x
- iPhone 14 Pro       : 393 × 852  @3x
- iPhone 14 Pro Max   : 430 × 932  @3x

iPad Sizes (Points):
- iPad mini (6th gen) : 744 × 1133 @2x
- iPad (10th gen)     : 820 × 1180 @2x
- iPad Pro 11"        : 834 × 1194 @2x
- iPad Pro 12.9"      : 1024 × 1366 @2x
```

---

## 41.10 iOS Human Interface Guidelines (HIG)

### หลักการออกแบบ iOS

Apple กำหนด Human Interface Guidelines (HIG) เป็นมาตรฐานการออกแบบแอป iOS ที่ดี

#### 6 หลักการหลัก

```
1. Clarity (ความชัดเจน)
   - ข้อความอ่านง่าย ตัวอักษรชัดเจน
   - ไอคอนและกราฟิกเข้าใจง่าย
   - เน้นเนื้อหาสำคัญ

2. Deference (การยอมตาม)
   - UI ไม่แข่งกับเนื้อหา
   - ใช้พื้นหลังโปร่งใส/blur ให้ดูสวยงาม
   - Content คือดาวเด่น

3. Depth (ความลึก)
   - ใช้ layering และ motion บอก hierarchy
   - Animation ให้ความรู้สึกของ space
   - Translucency ช่วยบอก context

4. Simplicity (ความเรียบง่าย)
   - ลดความซับซ้อน
   - แต่ละหน้าจอมีจุดประสงค์เดียว

5. Consistency (ความสม่ำเสมอ)
   - ใช้ UI ที่ผู้ใช้คุ้นเคย
   - ทำงานคล้าย system apps

6. Feedback (การตอบสนอง)
   - แสดงผลทันทีเมื่อผู้ใช้ interact
   - ใช้ haptic feedback
   - แสดง loading state
```

### Touch Target Size

```objc
// แนะนำขนาด Touch Target อย่างน้อย 44x44 points
CGFloat minimumTouchSize = 44.0;

UIButton *button = [[UIButton alloc] initWithFrame:
    CGRectMake(0, 0, minimumTouchSize, minimumTouchSize)];

// ถ้า visual size เล็กกว่า ควรขยาย hit area
// ด้วยการ override pointInside:withEvent:
@interface SmallButton : UIButton
@end

@implementation SmallButton
- (BOOL)pointInside:(CGPoint)point withEvent:(UIEvent *)event {
    CGRect extendedBounds = CGRectInset(self.bounds, -10, -10);
    return CGRectContainsPoint(extendedBounds, point);
}
@end
```

### Typography (การใช้ตัวอักษร)

```objc
// ใช้ Dynamic Type สำหรับ accessibility
UILabel *titleLabel = [[UILabel alloc] init];
titleLabel.font = [UIFont preferredFontForTextStyle:UIFontTextStyleTitle1];
titleLabel.adjustsFontForContentSizeCategory = YES;

UILabel *bodyLabel = [[UILabel alloc] init];
bodyLabel.font = [UIFont preferredFontForTextStyle:UIFontTextStyleBody];
bodyLabel.adjustsFontForContentSizeCategory = YES;

// Text Styles:
// UIFontTextStyleLargeTitle  - ขนาดใหญ่มาก (iOS 11+)
// UIFontTextStyleTitle1      - หัวข้อใหญ่
// UIFontTextStyleTitle2      - หัวข้อกลาง
// UIFontTextStyleTitle3      - หัวข้อเล็ก
// UIFontTextStyleHeadline    - หัวข้อ Bold
// UIFontTextStyleSubheadline - หัวข้อย่อย
// UIFontTextStyleBody        - เนื้อหา
// UIFontTextStyleCallout     - Callout text
// UIFontTextStyleFootnote    - หมายเหตุ
// UIFontTextStyleCaption1    - Caption ใหญ่
// UIFontTextStyleCaption2    - Caption เล็ก
```

### Colors (สี)

```objc
// ใช้ Semantic Colors ที่รองรับ Dark Mode อัตโนมัติ
UIColor *labelColor = [UIColor labelColor];                    // ข้อความหลัก
UIColor *secondaryLabel = [UIColor secondaryLabelColor];       // ข้อความรอง
UIColor *tertiaryLabel = [UIColor tertiaryLabelColor];         // ข้อความสาม
UIColor *background = [UIColor systemBackgroundColor];         // พื้นหลัง
UIColor *secondaryBackground = [UIColor secondarySystemBackgroundColor];
UIColor *groupedBackground = [UIColor systemGroupedBackgroundColor];

// System Colors
UIColor *blue = [UIColor systemBlueColor];
UIColor *red = [UIColor systemRedColor];
UIColor *green = [UIColor systemGreenColor];
UIColor *yellow = [UIColor systemYellowColor];
UIColor *orange = [UIColor systemOrangeColor];
UIColor *pink = [UIColor systemPinkColor];
UIColor *purple = [UIColor systemPurpleColor];
UIColor *teal = [UIColor systemTealColor];
UIColor *indigo = [UIColor systemIndigoColor];
UIColor *gray = [UIColor systemGrayColor];

// Custom color ที่รองรับ Dark Mode
UIColor *adaptiveColor = [UIColor colorWithDynamicProvider:^UIColor *(UITraitCollection *traitCollection) {
    if (traitCollection.userInterfaceStyle == UIUserInterfaceStyleDark) {
        return [UIColor colorWithRed:0.9 green:0.9 blue:0.9 alpha:1.0]; // สีอ่อนสำหรับ Dark Mode
    } else {
        return [UIColor colorWithRed:0.1 green:0.1 blue:0.1 alpha:1.0]; // สีเข้มสำหรับ Light Mode
    }
}];
```

---

## 41.11 MVC Pattern ใน iOS

### Model-View-Controller คืออะไร?

MVC เป็น design pattern หลักใน iOS development ที่แบ่งโค้ดออกเป็น 3 ส่วน:

```
     Model ←→ Controller ←→ View
       ↑                      ↑
  ข้อมูลและ              UI Elements
  Business Logic
```

### ความสัมพันธ์ของ MVC

```
Model → Controller: Notification, KVO, Delegation
Controller → Model: Direct access, Method calls
Controller → View: IBOutlet, Direct access
View → Controller: Target-Action, Delegation, IBAction
```

### ตัวอย่าง MVC ใน Objective-C

#### Model

```objc
// User.h - Model

#import <Foundation/Foundation.h>

@interface User : NSObject

@property (nonatomic, strong) NSString *name;
@property (nonatomic, strong) NSString *email;
@property (nonatomic, assign) NSInteger age;

- (instancetype)initWithName:(NSString *)name 
                       email:(NSString *)email 
                         age:(NSInteger)age;
- (NSString *)displayName;

@end
```

```objc
// User.m - Model Implementation

#import "User.h"

@implementation User

- (instancetype)initWithName:(NSString *)name 
                       email:(NSString *)email 
                         age:(NSInteger)age {
    self = [super init];
    if (self) {
        _name = name;
        _email = email;
        _age = age;
    }
    return self;
}

- (NSString *)displayName {
    return [NSString stringWithFormat:@"%@ (%ld ปี)", self.name, self.age];
}

@end
```

#### View

```objc
// UserProfileView.h - View

#import <UIKit/UIKit.h>
#import "User.h"

@interface UserProfileView : UIView

@property (nonatomic, strong) UILabel *nameLabel;
@property (nonatomic, strong) UILabel *emailLabel;
@property (nonatomic, strong) UIImageView *avatarImageView;

- (void)configureWithUser:(User *)user;

@end
```

```objc
// UserProfileView.m - View Implementation

#import "UserProfileView.h"

@implementation UserProfileView

- (instancetype)initWithFrame:(CGRect)frame {
    self = [super initWithFrame:frame];
    if (self) {
        [self setupUI];
    }
    return self;
}

- (void)setupUI {
    // Avatar
    self.avatarImageView = [[UIImageView alloc] init];
    self.avatarImageView.contentMode = UIViewContentModeScaleAspectFill;
    self.avatarImageView.clipsToBounds = YES;
    self.avatarImageView.layer.cornerRadius = 40;
    [self addSubview:self.avatarImageView];
    
    // Name Label
    self.nameLabel = [[UILabel alloc] init];
    self.nameLabel.font = [UIFont boldSystemFontOfSize:18];
    [self addSubview:self.nameLabel];
    
    // Email Label
    self.emailLabel = [[UILabel alloc] init];
    self.emailLabel.font = [UIFont systemFontOfSize:14];
    self.emailLabel.textColor = [UIColor secondaryLabelColor];
    [self addSubview:self.emailLabel];
}

- (void)configureWithUser:(User *)user {
    // View แสดงข้อมูลจาก Model ผ่าน Controller
    self.nameLabel.text = user.displayName;
    self.emailLabel.text = user.email;
}

@end
```

#### Controller

```objc
// UserViewController.h - Controller

#import <UIKit/UIKit.h>

@interface UserViewController : UIViewController

- (instancetype)initWithUserID:(NSString *)userID;

@end
```

```objc
// UserViewController.m - Controller Implementation

#import "UserViewController.h"
#import "User.h"
#import "UserProfileView.h"

@interface UserViewController ()

@property (nonatomic, strong) User *user;          // Model
@property (nonatomic, strong) UserProfileView *profileView;  // View

@end

@implementation UserViewController

- (instancetype)initWithUserID:(NSString *)userID {
    self = [super init];
    if (self) {
        // Controller สร้าง/โหลด Model
        _user = [self loadUserWithID:userID];
    }
    return self;
}

- (void)viewDidLoad {
    [super viewDidLoad];
    
    // Controller สร้าง View
    self.profileView = [[UserProfileView alloc] 
        initWithFrame:self.view.bounds];
    [self.view addSubview:self.profileView];
    
    // Controller ส่งข้อมูลจาก Model ไปยัง View
    [self.profileView configureWithUser:self.user];
}

- (User *)loadUserWithID:(NSString *)userID {
    // โหลดข้อมูลจาก database/network
    return [[User alloc] initWithName:@"สมชาย ใจดี"
                                email:@"somchai@example.com"
                                  age:25];
}

@end
```

---

## 41.12 First iOS App Walkthrough

### สร้างแอปแรก - Hello World

มาสร้างแอป iOS แรกของเราที่แสดงข้อความและปุ่มกด

#### ViewController.h

```objc
// ViewController.h

#import <UIKit/UIKit.h>

@interface ViewController : UIViewController

// ไม่จำเป็นต้องมี properties สาธารณะสำหรับแอปนี้

@end
```

#### ViewController.m - Programmatic UI

```objc
// ViewController.m

#import "ViewController.h"

@interface ViewController ()

// Private properties
@property (nonatomic, strong) UILabel *greetingLabel;
@property (nonatomic, strong) UIButton *greetButton;
@property (nonatomic, strong) UITextField *nameTextField;
@property (nonatomic, assign) NSInteger tapCount;

@end

@implementation ViewController

- (void)viewDidLoad {
    [super viewDidLoad];
    
    // ตั้งสีพื้นหลัง
    self.view.backgroundColor = [UIColor systemBackgroundColor];
    self.title = @"Hello World";
    
    [self setupUI];
}

- (void)setupUI {
    CGFloat screenWidth = self.view.bounds.size.width;
    CGFloat centerX = screenWidth / 2;
    
    // Title Label
    UILabel *titleLabel = [[UILabel alloc] initWithFrame:
        CGRectMake(20, 100, screenWidth - 40, 50)];
    titleLabel.text = @"ยินดีต้อนรับสู่ iOS";
    titleLabel.font = [UIFont boldSystemFontOfSize:24];
    titleLabel.textAlignment = NSTextAlignmentCenter;
    titleLabel.textColor = [UIColor labelColor];
    [self.view addSubview:titleLabel];
    
    // Name TextField
    self.nameTextField = [[UITextField alloc] initWithFrame:
        CGRectMake(40, 180, screenWidth - 80, 44)];
    self.nameTextField.placeholder = @"กรอกชื่อของคุณ";
    self.nameTextField.borderStyle = UITextBorderStyleRoundedRect;
    self.nameTextField.textAlignment = NSTextAlignmentCenter;
    self.nameTextField.returnKeyType = UIReturnKeyDone;
    self.nameTextField.delegate = self;
    [self.view addSubview:self.nameTextField];
    
    // Greet Button
    self.greetButton = [UIButton buttonWithType:UIButtonTypeSystem];
    self.greetButton.frame = CGRectMake(
        centerX - 75, 250, 150, 44);
    [self.greetButton setTitle:@"สวัสดี!" forState:UIControlStateNormal];
    self.greetButton.titleLabel.font = [UIFont systemFontOfSize:18];
    self.greetButton.backgroundColor = [UIColor systemBlueColor];
    [self.greetButton setTitleColor:[UIColor whiteColor] 
                            forState:UIControlStateNormal];
    self.greetButton.layer.cornerRadius = 10;
    
    // เพิ่ม action
    [self.greetButton addTarget:self 
                         action:@selector(greetButtonTapped:)
               forControlEvents:UIControlEventTouchUpInside];
    [self.view addSubview:self.greetButton];
    
    // Greeting Label (ผลลัพธ์)
    self.greetingLabel = [[UILabel alloc] initWithFrame:
        CGRectMake(20, 330, screenWidth - 40, 80)];
    self.greetingLabel.text = @"กด 'สวัสดี!' เพื่อทักทาย";
    self.greetingLabel.textAlignment = NSTextAlignmentCenter;
    self.greetingLabel.numberOfLines = 0;
    self.greetingLabel.font = [UIFont systemFontOfSize:18];
    self.greetingLabel.textColor = [UIColor secondaryLabelColor];
    [self.view addSubview:self.greetingLabel];
    
    // Tap Count Label
    UILabel *countLabel = [[UILabel alloc] initWithFrame:
        CGRectMake(20, 440, screenWidth - 40, 30)];
    countLabel.tag = 100;
    countLabel.textAlignment = NSTextAlignmentCenter;
    countLabel.font = [UIFont systemFontOfSize:14];
    countLabel.textColor = [UIColor tertiaryLabelColor];
    [self.view addSubview:countLabel];
    [self updateCountLabel];
}

#pragma mark - Button Actions

- (void)greetButtonTapped:(UIButton *)sender {
    self.tapCount++;
    
    NSString *name = self.nameTextField.text;
    if (name.length == 0) {
        name = @"ผู้ใช้";
    }
    
    NSArray *greetings = @[
        @"สวัสดีครับ",
        @"สวัสดีค่ะ",
        @"Hello",
        @"Hola",
        @"Bonjour"
    ];
    
    NSUInteger index = self.tapCount % greetings.count;
    NSString *greeting = greetings[index];
    
    self.greetingLabel.text = [NSString stringWithFormat:
        @"%@, %@! 👋", greeting, name];
    self.greetingLabel.textColor = [UIColor labelColor];
    
    // Animation
    [UIView animateWithDuration:0.3 animations:^{
        self.greetingLabel.transform = CGAffineTransformMakeScale(1.1, 1.1);
    } completion:^(BOOL finished) {
        [UIView animateWithDuration:0.2 animations:^{
            self.greetingLabel.transform = CGAffineTransformIdentity;
        }];
    }];
    
    [self updateCountLabel];
}

- (void)updateCountLabel {
    UILabel *countLabel = (UILabel *)[self.view viewWithTag:100];
    countLabel.text = [NSString stringWithFormat:
        @"กดแล้ว %ld ครั้ง", self.tapCount];
}

#pragma mark - UITextFieldDelegate

- (BOOL)textFieldShouldReturn:(UITextField *)textField {
    [textField resignFirstResponder];
    return YES;
}

- (void)touchesBegan:(NSSet<UITouch *> *)touches withEvent:(UIEvent *)event {
    [self.view endEditing:YES];
}

@end
```

---

## 41.13 Build และ Run

### การ Build แอป

```
ใน Xcode:
1. เลือก Target (iPhone/iPad Simulator หรือ Device)
2. กด ▶ (Run) หรือ Cmd+R
3. Xcode จะ Compile, Link, และ Install แอป
4. แอปจะเปิดขึ้นอัตโนมัติ
```

### Build Phases

```
1. Compile Sources
   - Compile ไฟล์ .m ทุกไฟล์
   - ตรวจสอบ syntax errors

2. Link Binary With Libraries
   - เชื่อมต่อ frameworks และ libraries

3. Copy Bundle Resources
   - คัดลอก assets, storyboards, plists

4. Run Script (ถ้ามี)
   - รัน custom build scripts
```

### การแก้ Error และ Warning

```objc
// Compiler Errors - ต้องแก้ก่อน Build ได้
// Error: ไม่พบ selector 'nonExistentMethod'

// Analyzer Warnings - ควรแก้
// Warning: Value stored is never read

// Runtime Errors (Crashes)
// - NSInvalidArgumentException
// - NSRangeException  
// - EXC_BAD_ACCESS (nil dereference)

// เปิด Address Sanitizer เพื่อตรวจหา memory errors
// Product → Scheme → Edit Scheme → Diagnostics → Address Sanitizer
```

### Debugging

```objc
// Breakpoint
// คลิกที่ gutter ด้านซ้ายของ source code
// หรือกด Cmd+\ 

// LLDB Commands ใน Xcode console
// po someVariable          - print object
// p someVariable           - print primitive
// expr someVariable = 10   - เปลี่ยนค่า
// bt                       - backtrace (stack trace)
// continue                 - ทำงานต่อ
// step                     - step into
// next                     - step over
// finish                   - step out

// NSLog สำหรับ debugging
NSLog(@"[DEBUG] user: %@, count: %ld", user.name, count);

// Assert สำหรับ catch bugs ตอน development
NSAssert(user != nil, @"user ต้องไม่เป็น nil");
NSParameterAssert(name.length > 0);
```

### Instruments สำหรับ Profiling

```
Instruments (Xcode → Product → Profile หรือ Cmd+I):

Time Profiler    - วิเคราะห์ CPU usage
Allocations      - ตรวจหา memory leaks  
Leaks            - ตรวจ memory leaks โดยตรง
Energy Log       - วิเคราะห์ battery usage
Network          - ตรวจสอบ network calls
Core Data        - วิเคราะห์ database operations
```

---

## 41.14 สรุปและแบบฝึกหัด

### สรุปหัวข้อที่เรียน

ในตอนนี้เราได้เรียนรู้:

1. **โครงสร้างแอป iOS** - ไฟล์และโฟลเดอร์ในโปรเจกต์
2. **Xcode Project Setup** - การสร้างและตั้งค่าโปรเจกต์
3. **Info.plist** - การตั้งค่าแอปพลิเคชัน
4. **AppDelegate** - จัดการ app lifecycle
5. **SceneDelegate** - จัดการ UI lifecycle (iOS 13+)
6. **Application Lifecycle** - States และการเปลี่ยนแปลง
7. **UIApplication** - Singleton class หลัก
8. **Target Settings** - Deployment target และ capabilities
9. **Simulator vs Device** - การทดสอบแอป
10. **HIG** - หลักการออกแบบ iOS
11. **MVC Pattern** - Architecture ใน iOS
12. **Hello World App** - แอปตัวอย่างแรก
13. **Build & Run** - กระบวนการ build และ debug

### แบบฝึกหัด

```
แบบฝึกหัดที่ 1:
สร้างโปรเจกต์ iOS ใหม่ชื่อ "MyFirstApp"
- ใส่ข้อมูลใน Info.plist: App version, display name
- แก้ไข AppDelegate ให้ log ทุก lifecycle event
- สร้าง ViewController ที่แสดงชื่อของคุณ

แบบฝึกหัดที่ 2:
สร้างหน้าจอที่มี:
- Label แสดงเวลาปัจจุบัน
- Button ที่กดแล้วอัปเดตเวลา
- Label แสดงจำนวนครั้งที่กด
- ใช้ Timer อัปเดตเวลาทุกวินาที

แบบฝึกหัดที่ 3:
ปรับปรุง Hello World app:
- เพิ่ม UISwitch สำหรับเปลี่ยนภาษาทักทาย (ไทย/อังกฤษ)
- เพิ่ม UIPickerView สำหรับเลือกท่าทักทาย
- บันทึกชื่อใน UserDefaults
- แสดงชื่อเดิมเมื่อเปิดแอปใหม่
```

### ตัวอย่าง Timer ใน ViewController

```objc
// ตัวอย่างการใช้ NSTimer

@interface TimerViewController : UIViewController

@property (nonatomic, strong) NSTimer *timer;
@property (nonatomic, strong) UILabel *timeLabel;
@property (nonatomic, assign) NSInteger seconds;

@end

@implementation TimerViewController

- (void)viewDidLoad {
    [super viewDidLoad];
    
    // สร้าง Label
    self.timeLabel = [[UILabel alloc] initWithFrame:
        CGRectMake(0, 200, self.view.bounds.size.width, 60)];
    self.timeLabel.textAlignment = NSTextAlignmentCenter;
    self.timeLabel.font = [UIFont monospacedDigitSystemFontOfSize:48 
                                                          weight:UIFontWeightLight];
    [self.view addSubview:self.timeLabel];
    
    // เริ่ม Timer
    [self startTimer];
}

- (void)startTimer {
    self.timer = [NSTimer scheduledTimerWithTimeInterval:1.0
                                                 target:self
                                               selector:@selector(timerFired:)
                                               userInfo:nil
                                                repeats:YES];
}

- (void)timerFired:(NSTimer *)timer {
    self.seconds++;
    
    NSInteger hours = self.seconds / 3600;
    NSInteger minutes = (self.seconds % 3600) / 60;
    NSInteger secs = self.seconds % 60;
    
    self.timeLabel.text = [NSString stringWithFormat:
        @"%02ld:%02ld:%02ld", hours, minutes, secs];
}

- (void)viewWillDisappear:(BOOL)animated {
    [super viewWillDisappear:animated];
    
    // สำคัญ: หยุด timer เมื่อ view หายไป
    [self.timer invalidate];
    self.timer = nil;
}

@end
```

---

## บทสรุป

การพัฒนาแอป iOS เริ่มจากการเข้าใจโครงสร้างพื้นฐาน ตั้งแต่ไฟล์ต่างๆ ในโปรเจกต์ไปจนถึง lifecycle ของแอปพลิเคชัน การเข้าใจ AppDelegate และ SceneDelegate เป็นสิ่งสำคัญในการจัดการการทำงานของแอปในสถานการณ์ต่างๆ

ในตอนต่อไปเราจะเรียนรู้เกี่ยวกับ UIKit framework ซึ่งเป็นชุดเครื่องมือหลักสำหรับสร้าง UI บน iOS

---

*ตอนที่ 41 จบแล้ว - ไปต่อที่ตอนที่ 42: UIKit Basics*
