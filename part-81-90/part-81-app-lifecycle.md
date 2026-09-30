# Part 81: App Lifecycle Advanced (วงจรชีวิตแอปพลิเคชันขั้นสูง)

## บทนำ

การเข้าใจวงจรชีวิตของแอปพลิเคชัน (App Lifecycle) เป็นพื้นฐานสำคัญของการพัฒนาแอป iOS ที่มีคุณภาพ ในบทนี้เราจะศึกษาเกี่ยวกับสถานะต่างๆ ของแอปพลิเคชัน การจัดการการเปลี่ยนแปลงสถานะ การปรับปรุงประสิทธิภาพการเปิดแอป และเทคนิคขั้นสูงต่างๆ ที่ developer มืออาชีพใช้ในการพัฒนาแอปจริง

---

## 1. UIApplication States (สถานะของ UIApplication)

iOS จัดการสถานะของแอปพลิเคชันผ่านระบบที่มีการออกแบบอย่างรอบคอบ โดยแอปสามารถอยู่ในสถานะต่างๆ ดังนี้:

### 1.1 Not Running (ยังไม่ทำงาน)
แอปยังไม่ถูกเปิดโดยผู้ใช้ หรือถูก terminate โดยระบบ หรือผู้ใช้ปิดแอปจาก App Switcher

### 1.2 Inactive (ไม่ใช้งาน)
แอปกำลังทำงานอยู่ใน foreground แต่ไม่ได้รับ events เช่น ระหว่างที่มีการโทรเข้า หรือเมื่อผู้ใช้กด Home button

### 1.3 Active (ใช้งานอยู่)
แอปกำลังทำงานอยู่ใน foreground และรับ events ต่างๆ ได้ปกติ

### 1.4 Background (ทำงานพื้นหลัง)
แอปอยู่ใน background และยังสามารถ execute code ได้ เช่น กำลัง download file หรือ playing audio

### 1.5 Suspended (หยุดชั่วคราว)
แอปอยู่ใน background แต่ไม่ได้ execute code ระบบอาจ terminate แอปนี้ได้ตลอดเวลาโดยไม่มีการแจ้งเตือน

```objc
// ตรวจสอบสถานะปัจจุบันของแอป
UIApplicationState currentState = [[UIApplication sharedApplication] applicationState];

switch (currentState) {
    case UIApplicationStateActive:
        NSLog(@"App is Active - กำลังใช้งานอยู่");
        break;
    case UIApplicationStateInactive:
        NSLog(@"App is Inactive - ไม่ใช้งาน");
        break;
    case UIApplicationStateBackground:
        NSLog(@"App is in Background - ทำงานพื้นหลัง");
        break;
    default:
        NSLog(@"Unknown state");
        break;
}
```

### ไดอะแกรม State Transitions

```
Not Running → (Launch) → Inactive → (Activate) → Active
                                         ↑              ↓
                                    (Reactivate)   (Interrupt/Home)
                                         |              ↓
                                    Background ← Inactive
                                         ↓
                                    Suspended
                                         ↓
                                   Not Running
```

---

## 2. AppDelegate Methods (เมธอดของ AppDelegate)

AppDelegate เป็น central coordinator สำหรับการจัดการ app-level events ใน iOS

### 2.1 Application Launch

```objc
// AppDelegate.h
#import <UIKit/UIKit.h>

@interface AppDelegate : UIResponder <UIApplicationDelegate>

@property (strong, nonatomic) UIWindow *window;

// Custom properties
@property (strong, nonatomic) NSString *launchOptions;
@property (assign, nonatomic) NSTimeInterval appLaunchTime;

@end
```

```objc
// AppDelegate.m
#import "AppDelegate.h"
#import "MainViewController.h"
#import "UserDefaultsManager.h"
#import "NetworkManager.h"

@implementation AppDelegate

// เมธอดแรกที่ถูกเรียกเมื่อแอปเปิดขึ้น
- (BOOL)application:(UIApplication *)application 
    didFinishLaunchingWithOptions:(NSDictionary *)launchOptions {
    
    // บันทึกเวลาเปิดแอปเพื่อวัด launch time
    self.appLaunchTime = CFAbsoluteTimeGetCurrent();
    
    NSLog(@"=== App Launch Started ===");
    NSLog(@"Launch options: %@", launchOptions);
    
    // 1. ตั้งค่าพื้นฐาน (ทำก่อนอื่น)
    [self setupCoreConfiguration];
    
    // 2. ตั้งค่า UI
    [self setupRootViewController];
    
    // 3. ตั้งค่า Services ต่างๆ (async ถ้าเป็นไปได้)
    [self setupServicesAsync];
    
    // 4. ตรวจสอบ launch options
    [self handleLaunchOptions:launchOptions];
    
    NSTimeInterval launchDuration = CFAbsoluteTimeGetCurrent() - self.appLaunchTime;
    NSLog(@"App launched in %.3f seconds", launchDuration);
    
    return YES;
}

- (void)setupCoreConfiguration {
    // ตั้งค่า UserDefaults
    [UserDefaultsManager setupDefaults];
    
    // ตั้งค่า appearance
    [self configureAppearance];
    
    // ตั้งค่า Analytics
    // [AnalyticsManager configure];
    
    NSLog(@"Core configuration completed");
}

- (void)configureAppearance {
    // ตั้งค่า navigation bar appearance
    UINavigationBarAppearance *navAppearance = [[UINavigationBarAppearance alloc] init];
    [navAppearance configureWithOpaqueBackground];
    navAppearance.backgroundColor = [UIColor systemBlueColor];
    navAppearance.titleTextAttributes = @{
        NSForegroundColorAttributeName: [UIColor whiteColor],
        NSFontAttributeName: [UIFont boldSystemFontOfSize:18.0]
    };
    
    [UINavigationBar appearance].standardAppearance = navAppearance;
    [UINavigationBar appearance].scrollEdgeAppearance = navAppearance;
    [UINavigationBar appearance].tintColor = [UIColor whiteColor];
    
    // ตั้งค่า tab bar appearance
    UITabBarAppearance *tabAppearance = [[UITabBarAppearance alloc] init];
    [tabAppearance configureWithOpaqueBackground];
    [UITabBar appearance].standardAppearance = tabAppearance;
    
    NSLog(@"Appearance configured");
}

- (void)setupRootViewController {
    self.window = [[UIWindow alloc] initWithFrame:[[UIScreen mainScreen] bounds]];
    
    // ตรวจสอบว่าผู้ใช้ login แล้วหรือยัง
    BOOL isLoggedIn = [[NSUserDefaults standardUserDefaults] boolForKey:@"isLoggedIn"];
    
    UIViewController *rootVC;
    if (isLoggedIn) {
        rootVC = [[MainViewController alloc] init];
    } else {
        rootVC = [[LoginViewController alloc] init];
    }
    
    UINavigationController *navController = [[UINavigationController alloc] 
                                              initWithRootViewController:rootVC];
    self.window.rootViewController = navController;
    [self.window makeKeyAndVisible];
    
    NSLog(@"Root view controller set up");
}

- (void)setupServicesAsync {
    dispatch_async(dispatch_get_global_queue(DISPATCH_QUEUE_PRIORITY_DEFAULT, 0), ^{
        // ตั้งค่า services ที่ไม่จำเป็นต้องทำใน main thread
        [[NetworkManager shared] configure];
        // [[DatabaseManager shared] initialize];
        
        dispatch_async(dispatch_get_main_queue(), ^{
            NSLog(@"Async services setup completed");
        });
    });
}

- (void)handleLaunchOptions:(NSDictionary *)launchOptions {
    if (!launchOptions) return;
    
    // ตรวจสอบว่าถูกเปิดจาก push notification
    NSDictionary *remoteNotification = launchOptions[UIApplicationLaunchOptionsRemoteNotificationKey];
    if (remoteNotification) {
        NSLog(@"Launched from remote notification: %@", remoteNotification);
        // จัดการ notification
    }
    
    // ตรวจสอบว่าถูกเปิดจาก URL
    NSURL *url = launchOptions[UIApplicationLaunchOptionsURLKey];
    if (url) {
        NSLog(@"Launched with URL: %@", url);
        // จัดการ URL
    }
    
    // ตรวจสอบ shortcut item
    UIApplicationShortcutItem *shortcutItem = 
        launchOptions[UIApplicationLaunchOptionsShortcutItemKey];
    if (shortcutItem) {
        NSLog(@"Launched with shortcut: %@", shortcutItem.type);
        [self handleShortcutItem:shortcutItem];
    }
}
```

### 2.2 App State Transition Methods

```objc
// เมื่อแอปกำลังจะออกจาก Active state
- (void)applicationWillResignActive:(UIApplication *)application {
    NSLog(@"=== applicationWillResignActive ===");
    
    // หยุด timers ที่ไม่จำเป็น
    // บันทึก state ชั่วคราว
    // หยุด animations
    // ปิด audio/video ถ้าจำเป็น
    
    [[NSNotificationCenter defaultCenter] 
     postNotificationName:@"AppWillResignActive" 
     object:nil];
}

// เมื่อแอปเข้า background
- (void)applicationDidEnterBackground:(UIApplication *)application {
    NSLog(@"=== applicationDidEnterBackground ===");
    
    // บันทึกข้อมูลสำคัญ
    [self saveApplicationState];
    
    // ขอเวลาทำงานใน background เพิ่มเติม
    UIBackgroundTaskIdentifier bgTask = [application beginBackgroundTaskWithName:@"SaveData" 
                                         expirationHandler:^{
        NSLog(@"Background task expired");
        [application endBackgroundTask:bgTask];
    }];
    
    dispatch_async(dispatch_get_global_queue(DISPATCH_QUEUE_PRIORITY_DEFAULT, 0), ^{
        // ทำงานที่ต้องทำก่อนแอปจะ suspend
        [self performBackgroundWork];
        
        dispatch_async(dispatch_get_main_queue(), ^{
            [application endBackgroundTask:bgTask];
        });
    });
}

// เมื่อแอปกลับมาจาก background
- (void)applicationWillEnterForeground:(UIApplication *)application {
    NSLog(@"=== applicationWillEnterForeground ===");
    
    // เตรียม UI สำหรับการแสดงผล
    // ตรวจสอบว่ามีข้อมูลใหม่หรือไม่
    [self refreshDataIfNeeded];
}

// เมื่อแอปกลับมา Active
- (void)applicationDidBecomeActive:(UIApplication *)application {
    NSLog(@"=== applicationDidBecomeActive ===");
    
    // เริ่ม timers ใหม่
    // รีเฟรช UI
    // ตรวจสอบการเชื่อมต่อเครือข่าย
    
    [[NSNotificationCenter defaultCenter] 
     postNotificationName:@"AppDidBecomeActive" 
     object:nil];
    
    // ตรวจสอบว่าผ่านมานานแค่ไหนแล้ว
    NSDate *lastActiveDate = [[NSUserDefaults standardUserDefaults] 
                               objectForKey:@"lastActiveDate"];
    if (lastActiveDate) {
        NSTimeInterval elapsed = [[NSDate date] timeIntervalSinceDate:lastActiveDate];
        NSLog(@"Time since last active: %.0f seconds", elapsed);
        
        if (elapsed > 3600) { // มากกว่า 1 ชั่วโมง
            [self handleLongInactivity];
        }
    }
    
    [[NSUserDefaults standardUserDefaults] setObject:[NSDate date] 
                                              forKey:@"lastActiveDate"];
}

// เมื่อแอปกำลังจะ terminate
- (void)applicationWillTerminate:(UIApplication *)application {
    NSLog(@"=== applicationWillTerminate ===");
    
    // บันทึกข้อมูลสำคัญครั้งสุดท้าย
    [self saveApplicationState];
    
    // Clean up resources
    // ยกเลิก pending operations
}

- (void)saveApplicationState {
    NSLog(@"Saving application state...");
    // บันทึก user data, settings, etc.
    [[NSUserDefaults standardUserDefaults] synchronize];
}

- (void)performBackgroundWork {
    NSLog(@"Performing background work...");
    // ทำงานที่ต้องทำใน background
}

- (void)refreshDataIfNeeded {
    NSLog(@"Refreshing data if needed...");
}

- (void)handleLongInactivity {
    NSLog(@"Handling long inactivity - refreshing data");
    // อาจจะ refresh token หรือ reload data
}
```

---

## 3. SceneDelegate (iOS 13+)

ตั้งแต่ iOS 13 Apple แนะนำ Scene-based lifecycle ซึ่งรองรับ multiple windows บน iPad

### 3.1 Info.plist Configuration

```xml
<!-- ใน Info.plist ต้องเพิ่ม -->
<key>UIApplicationSceneManifest</key>
<dict>
    <key>UIApplicationSupportsMultipleScenes</key>
    <true/>
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

### 3.2 SceneDelegate Implementation

```objc
// SceneDelegate.h
#import <UIKit/UIKit.h>

@interface SceneDelegate : UIResponder <UIWindowSceneDelegate>

@property (strong, nonatomic) UIWindow *window;

@end
```

```objc
// SceneDelegate.m
#import "SceneDelegate.h"

@interface SceneDelegate ()

@property (strong, nonatomic) NSDate *sceneEnterBackgroundDate;

@end

@implementation SceneDelegate

// Scene ถูกสร้างและเชื่อมต่อ
- (void)scene:(UIScene *)scene 
    willConnectToSession:(UISceneSession *)session 
    options:(UISceneConnectionOptions *)connectionOptions {
    
    NSLog(@"=== Scene will connect to session ===");
    NSLog(@"Scene identifier: %@", session.persistentIdentifier);
    
    UIWindowScene *windowScene = (UIWindowScene *)scene;
    if (!windowScene) return;
    
    self.window = [[UIWindow alloc] initWithWindowScene:windowScene];
    
    // ตรวจสอบ user activities
    NSUserActivity *userActivity = connectionOptions.userActivities.anyObject;
    if (userActivity) {
        NSLog(@"Connecting with user activity: %@", userActivity.activityType);
        [self configureWindowForUserActivity:userActivity];
    } else {
        [self configureDefaultWindow];
    }
    
    [self.window makeKeyAndVisible];
    
    // ตรวจสอบ URL contexts
    NSSet<UIOpenURLContext *> *urlContexts = connectionOptions.URLContexts;
    if (urlContexts.count > 0) {
        [self handleURLContexts:urlContexts];
    }
    
    // ตรวจสอบ shortcut item
    UIApplicationShortcutItem *shortcutItem = connectionOptions.shortcutItem;
    if (shortcutItem) {
        NSLog(@"Connected with shortcut: %@", shortcutItem.type);
    }
}

- (void)configureDefaultWindow {
    UIViewController *mainVC = [[MainViewController alloc] init];
    UINavigationController *navVC = [[UINavigationController alloc] 
                                      initWithRootViewController:mainVC];
    self.window.rootViewController = navVC;
}

- (void)configureWindowForUserActivity:(NSUserActivity *)activity {
    // ตั้งค่า window ตาม user activity
    if ([activity.activityType isEqualToString:@"com.example.viewItem"]) {
        NSString *itemId = activity.userInfo[@"itemId"];
        DetailViewController *detailVC = [[DetailViewController alloc] initWithItemId:itemId];
        self.window.rootViewController = detailVC;
    } else {
        [self configureDefaultWindow];
    }
}

- (void)handleURLContexts:(NSSet<UIOpenURLContext *> *)URLContexts {
    for (UIOpenURLContext *context in URLContexts) {
        NSLog(@"Opening URL: %@", context.URL);
        [self processURL:context.URL];
    }
}

- (void)processURL:(NSURL *)url {
    // จัดการ URL scheme
    NSLog(@"Processing URL: %@", url);
}

// Scene disconnect
- (void)sceneDidDisconnect:(UIScene *)scene {
    NSLog(@"=== Scene did disconnect ===");
    // ปล่อย resources ที่ไม่จำเป็น
    // ข้อมูลจะถูก recreate เมื่อ reconnect
}

// Scene กลับมา Active
- (void)sceneDidBecomeActive:(UIScene *)scene {
    NSLog(@"=== Scene did become active ===");
    
    // Resume paused activities
    // ตรวจสอบว่ามีข้อมูลใหม่หรือไม่
    
    if (self.sceneEnterBackgroundDate) {
        NSTimeInterval elapsed = [[NSDate date] 
                                   timeIntervalSinceDate:self.sceneEnterBackgroundDate];
        NSLog(@"Scene was in background for %.0f seconds", elapsed);
        
        if (elapsed > 1800) { // 30 นาที
            [self refreshContent];
        }
    }
}

// Scene กำลังจะ inactive
- (void)sceneWillResignActive:(UIScene *)scene {
    NSLog(@"=== Scene will resign active ===");
    // บันทึก state
    // หยุด operations ที่ไม่จำเป็น
}

// Scene กำลังจะเข้า background
- (void)sceneWillEnterForeground:(UIScene *)scene {
    NSLog(@"=== Scene will enter foreground ===");
    // เตรียม UI
}

// Scene เข้า background แล้ว
- (void)sceneDidEnterBackground:(UIScene *)scene {
    NSLog(@"=== Scene did enter background ===");
    self.sceneEnterBackgroundDate = [NSDate date];
    
    // บันทึกข้อมูล
    [self saveSceneState];
}

- (void)saveSceneState {
    NSLog(@"Saving scene state...");
}

- (void)refreshContent {
    NSLog(@"Refreshing content after long background...");
}

// จัดการ URL ที่ถูกส่งมาขณะแอปทำงานอยู่
- (void)scene:(UIScene *)scene 
    openURLContexts:(NSSet<UIOpenURLContext *> *)URLContexts {
    NSLog(@"=== Scene open URL contexts ===");
    [self handleURLContexts:URLContexts];
}

// จัดการ User Activity (Handoff, Universal Links)
- (void)scene:(UIScene *)scene 
    continueUserActivity:(NSUserActivity *)userActivity {
    NSLog(@"=== Continue user activity: %@ ===", userActivity.activityType);
    [self handleUserActivity:userActivity];
}

- (void)handleUserActivity:(NSUserActivity *)activity {
    if ([activity.activityType isEqualToString:NSUserActivityTypeBrowsingWeb]) {
        NSURL *webURL = activity.webpageURL;
        NSLog(@"Universal link: %@", webURL);
        // จัดการ Universal Link
    }
}

@end
```

### 3.3 Multiple Scenes Support

```objc
// AppDelegate สำหรับ Scene-based app
@implementation AppDelegate

- (BOOL)application:(UIApplication *)application 
    didFinishLaunchingWithOptions:(NSDictionary *)launchOptions {
    
    // ใน iOS 13+ ส่วนใหญ่ทำงานใน SceneDelegate แทน
    // ทำเฉพาะ app-level setup ที่ไม่เกี่ยวกับ UI
    [self setupAppLevel];
    return YES;
}

// กำหนด configuration สำหรับ scene ใหม่
- (UISceneConfiguration *)application:(UIApplication *)application 
    configurationForConnectingSceneSession:(UISceneSession *)connectingSceneSession 
    options:(UISceneConnectionOptions *)options {
    
    // ตรวจสอบ shortcut item เพื่อกำหนด scene configuration
    UIApplicationShortcutItem *shortcutItem = options.shortcutItem;
    
    NSString *configName;
    if (shortcutItem && [shortcutItem.type isEqualToString:@"com.example.newItem"]) {
        configName = @"Create Configuration";
    } else {
        configName = @"Default Configuration";
    }
    
    return [UISceneConfiguration configurationWithName:configName 
                                          sessionRole:connectingSceneSession.role];
}

// เมื่อ scene session ถูก discard
- (void)application:(UIApplication *)application 
    didDiscardSceneSessions:(NSSet<UISceneSession *> *)sceneSessions {
    
    for (UISceneSession *session in sceneSessions) {
        NSLog(@"Discarded scene session: %@", session.persistentIdentifier);
        // ลบข้อมูลที่เกี่ยวข้องกับ session นี้
    }
}

- (void)setupAppLevel {
    // ตั้งค่า services ที่ใช้ร่วมกันทุก scene
    // Analytics, Crash reporting, etc.
}

@end
```

---

## 4. Background Execution Basics (พื้นฐานการทำงานพื้นหลัง)

```objc
// ขอเวลาทำงานใน background
- (void)applicationDidEnterBackground:(UIApplication *)application {
    // เริ่ม background task
    __block UIBackgroundTaskIdentifier backgroundTask = UIBackgroundTaskInvalid;
    
    backgroundTask = [application beginBackgroundTaskWithName:@"ImportantTask" 
                                            expirationHandler:^{
        // เวลาหมด ต้องหยุดทันที
        NSLog(@"Background task expired, stopping...");
        
        // ทำความสะอาดก่อนหยุด
        [self cleanupBackgroundTask];
        
        [application endBackgroundTask:backgroundTask];
        backgroundTask = UIBackgroundTaskInvalid;
    }];
    
    // ทำงานใน background
    dispatch_async(dispatch_get_global_queue(DISPATCH_QUEUE_PRIORITY_DEFAULT, 0), ^{
        NSLog(@"Background time remaining: %.0f seconds", 
              [UIApplication sharedApplication].backgroundTimeRemaining);
        
        [self performImportantBackgroundWork];
        
        // สิ้นสุด background task
        [application endBackgroundTask:backgroundTask];
        backgroundTask = UIBackgroundTaskInvalid;
    });
}

- (void)performImportantBackgroundWork {
    // จำลองการทำงาน
    NSLog(@"Performing important background work...");
    [NSThread sleepForTimeInterval:2.0]; // ไม่ควรทำใน production
    NSLog(@"Background work completed");
}

- (void)cleanupBackgroundTask {
    NSLog(@"Cleaning up background task...");
}
```

---

## 5. Application State Restoration (การกู้คืนสถานะ)

State Restoration ช่วยให้ผู้ใช้กลับมาในสภาวะที่ตนอยู่ก่อนหน้าที่แอปจะถูก terminate

### 5.1 เปิดใช้งาน State Restoration

```objc
// AppDelegate.m
- (BOOL)application:(UIApplication *)application 
    shouldSaveSecureApplicationState:(NSCoder *)coder {
    return YES; // เปิดใช้งาน state saving
}

- (BOOL)application:(UIApplication *)application 
    shouldRestoreSecureApplicationState:(NSCoder *)coder {
    
    // ตรวจสอบว่าควร restore หรือไม่
    NSString *savedVersion = [coder decodeObjectOfClass:[NSString class] 
                                                 forKey:@"AppVersion"];
    NSString *currentVersion = [[NSBundle mainBundle] 
                                 objectForInfoDictionaryKey:@"CFBundleShortVersionString"];
    
    // ถ้า version ต่างกันมาก อาจไม่ต้อง restore
    if (![savedVersion isEqualToString:currentVersion]) {
        NSLog(@"Version mismatch, skipping restoration");
        return NO;
    }
    
    return YES;
}

// บันทึก app-level state
- (void)application:(UIApplication *)application 
    willEncodeRestorableStateWithCoder:(NSCoder *)coder {
    
    NSLog(@"Saving restorable state");
    [coder encodeObject:@"1.0" forKey:@"AppVersion"];
    [coder encodeObject:[NSDate date] forKey:@"SaveDate"];
}

// กู้คืน app-level state
- (void)application:(UIApplication *)application 
    didDecodeRestorableStateWithCoder:(NSCoder *)coder {
    
    NSLog(@"Restoring state");
    NSDate *saveDate = [coder decodeObjectOfClass:[NSDate class] forKey:@"SaveDate"];
    NSLog(@"State was saved on: %@", saveDate);
}
```

### 5.2 UIViewController State Restoration

```objc
// ViewController.h
@interface ProductDetailViewController : UIViewController <UIStateRestoring>

@property (strong, nonatomic) NSString *productId;
@property (strong, nonatomic) NSString *restorationIdentifier;

@end
```

```objc
// ViewController.m
@implementation ProductDetailViewController

+ (UIViewController *)viewControllerWithRestorationIdentifierPath:(NSArray *)identifierComponents 
                                                            coder:(NSCoder *)coder {
    // สร้าง view controller จาก restoration data
    ProductDetailViewController *vc = [[ProductDetailViewController alloc] init];
    vc.restorationIdentifier = identifierComponents.lastObject;
    
    // กู้คืน product ID
    vc.productId = [coder decodeObjectOfClass:[NSString class] forKey:@"productId"];
    
    return vc;
}

- (void)viewDidLoad {
    [super viewDidLoad];
    
    // ตั้งค่า restoration identifier
    self.restorationIdentifier = @"ProductDetailViewController";
    self.restorationClass = [self class];
}

// บันทึก view controller state
- (void)encodeRestorableStateWithCoder:(NSCoder *)coder {
    [super encodeRestorableStateWithCoder:coder];
    
    [coder encodeObject:self.productId forKey:@"productId"];
    [coder encodeInteger:self.tableView.contentOffset.y 
                  forKey:@"scrollPosition"];
    
    NSLog(@"Encoded state for product: %@", self.productId);
}

// กู้คืน view controller state
- (void)decodeRestorableStateWithCoder:(NSCoder *)coder {
    [super decodeRestorableStateWithCoder:coder];
    
    self.productId = [coder decodeObjectOfClass:[NSString class] forKey:@"productId"];
    CGFloat scrollY = [coder decodeIntegerForKey:@"scrollPosition"];
    
    // กู้คืน scroll position
    [self.tableView setContentOffset:CGPointMake(0, scrollY) animated:NO];
    
    NSLog(@"Decoded state for product: %@", self.productId);
}

@end
```

---

## 6. Launch Time Optimization (การปรับปรุงเวลาเปิดแอป)

### 6.1 วัด Launch Time

```objc
// ใน main.m หรือ AppDelegate
static CFTimeInterval sAppStartTime;

int main(int argc, char * argv[]) {
    sAppStartTime = CACurrentMediaTime();
    @autoreleasepool {
        return UIApplicationMain(argc, argv, nil, NSStringFromClass([AppDelegate class]));
    }
}

// ใน viewDidAppear ของ root view controller
- (void)viewDidAppear:(BOOL)animated {
    [super viewDidAppear:animated];
    
    extern CFTimeInterval sAppStartTime;
    CFTimeInterval launchTime = CACurrentMediaTime() - sAppStartTime;
    NSLog(@"Time to first frame: %.3f seconds", launchTime);
    
    // Apple guideline: ควรน้อยกว่า 400ms
    if (launchTime > 0.4) {
        NSLog(@"WARNING: Launch time too slow!");
    }
}
```

### 6.2 Lazy Loading

```objc
@interface AppDelegate ()

// ใช้ lazy properties
@property (strong, nonatomic) DatabaseManager *databaseManager;
@property (strong, nonatomic) NetworkManager *networkManager;
@property (strong, nonatomic) CacheManager *cacheManager;

@end

@implementation AppDelegate

// Lazy initialization - ไม่สร้างจนกว่าจะใช้งาน
- (DatabaseManager *)databaseManager {
    if (!_databaseManager) {
        _databaseManager = [[DatabaseManager alloc] init];
        NSLog(@"Database manager created lazily");
    }
    return _databaseManager;
}

- (NetworkManager *)networkManager {
    if (!_networkManager) {
        _networkManager = [[NetworkManager alloc] init];
        NSLog(@"Network manager created lazily");
    }
    return _networkManager;
}

- (BOOL)application:(UIApplication *)application 
    didFinishLaunchingWithOptions:(NSDictionary *)launchOptions {
    
    // ทำเฉพาะสิ่งจำเป็นใน launch
    [self setupMinimalRequiredServices];
    [self setupRootViewController];
    
    // เลื่อนการทำงานที่ไม่จำเป็นออกไป
    dispatch_after(dispatch_time(DISPATCH_TIME_NOW, (int64_t)(2.0 * NSEC_PER_SEC)), 
                   dispatch_get_main_queue(), ^{
        [self setupNonCriticalServices];
    });
    
    return YES;
}

- (void)setupMinimalRequiredServices {
    // เฉพาะสิ่งที่ต้องการสำหรับ first screen
    [UserDefaultsManager setup];
}

- (void)setupNonCriticalServices {
    // Services ที่ไม่จำเป็นสำหรับ launch
    // Analytics, crash reporting, background sync, etc.
    NSLog(@"Setting up non-critical services...");
}

@end
```

### 6.3 ลด dylib Loading

```objc
// หลีกเลี่ยงการใช้ dynamic frameworks ที่ไม่จำเป็นใน launch path
// ใช้ weak linking สำหรับ optional frameworks

// ตรวจสอบว่า framework มีอยู่หรือไม่
if (NSClassFromString(@"ARSCNView") != nil) {
    // ARKit available
    NSLog(@"ARKit is available");
} else {
    NSLog(@"ARKit is not available");
}
```

---

## 7. Cold vs Warm Launch (การเปิดแอปเย็นและอุ่น)

### Cold Launch
- แอปไม่ได้อยู่ใน memory เลย
- ต้องโหลดทุกอย่างใหม่
- ใช้เวลานานกว่า
- เกิดขึ้นเมื่อ: รีสตาร์ทเครื่อง, แอปถูก terminate, ครั้งแรกที่เปิด

### Warm Launch
- แอปยังคงอยู่ใน memory แต่ process ยังทำงานอยู่
- ไม่ต้องโหลด libraries ใหม่
- เร็วกว่า cold launch

### Resume (Hot)
- แอปยังคงอยู่ใน suspended state
- แทบไม่ต้องทำอะไร เพียงแค่ resume

```objc
// วัดและบันทึก launch type
@implementation AppDelegate

static NSDate *sLaunchDate;

+ (void)initialize {
    if (self == [AppDelegate class]) {
        sLaunchDate = [NSDate date];
    }
}

- (BOOL)application:(UIApplication *)application 
    didFinishLaunchingWithOptions:(NSDictionary *)launchOptions {
    
    // ตรวจสอบว่านี่คือ cold launch หรือไม่
    NSDate *lastTerminateDate = [[NSUserDefaults standardUserDefaults] 
                                  objectForKey:@"lastTerminateDate"];
    
    BOOL isColdLaunch = (lastTerminateDate == nil) || 
                         ([[NSDate date] timeIntervalSinceDate:lastTerminateDate] > 60);
    
    if (isColdLaunch) {
        NSLog(@"Cold launch detected");
        [self optimizeForColdLaunch];
    } else {
        NSLog(@"Warm launch detected");
    }
    
    return YES;
}

- (void)applicationWillTerminate:(UIApplication *)application {
    [[NSUserDefaults standardUserDefaults] setObject:[NSDate date] 
                                              forKey:@"lastTerminateDate"];
}

- (void)optimizeForColdLaunch {
    // สำหรับ cold launch ให้ focus เฉพาะ critical path
    // Defer everything non-essential
}

@end
```

---

## 8. App Launch Sequence (ลำดับการเปิดแอป)

ลำดับขั้นตอนเมื่อแอปถูกเปิดขึ้น:

```
1. Kernel loads app
2. Dyld loads shared libraries and frameworks
3. Objective-C runtime initializes classes (+load methods)
4. main() is called
5. UIApplicationMain() is called
6. UIApplication is created
7. AppDelegate is instantiated
8. application:didFinishLaunchingWithOptions: is called
9. Window and root view controller are created
10. viewDidLoad is called
11. viewWillAppear is called  
12. First layout pass
13. viewDidAppear is called
14. App is fully interactive
```

```objc
// ติดตาม launch sequence
@implementation MyViewController

- (instancetype)init {
    self = [super init];
    if (self) {
        NSLog(@"[Launch] ViewController init");
    }
    return self;
}

- (void)loadView {
    [super loadView];
    NSLog(@"[Launch] loadView");
}

- (void)viewDidLoad {
    [super viewDidLoad];
    NSLog(@"[Launch] viewDidLoad");
}

- (void)viewWillAppear:(BOOL)animated {
    [super viewWillAppear:animated];
    NSLog(@"[Launch] viewWillAppear");
}

- (void)viewDidAppear:(BOOL)animated {
    [super viewDidAppear:animated];
    NSLog(@"[Launch] viewDidAppear - App is ready!");
    
    // นี่คือจุดที่ผู้ใช้สามารถโต้ตอบกับแอปได้
    [self startNonCriticalWork];
}

- (void)startNonCriticalWork {
    // โหลดข้อมูลเพิ่มเติม, ตั้งค่า analytics, etc.
}

@end
```

---

## 9. UIApplicationDelegate Notifications

```objc
// ลงทะเบียน notifications เพื่อตอบสนองต่อ app lifecycle events
@implementation SomeViewController

- (void)viewDidLoad {
    [super viewDidLoad];
    [self registerForAppLifecycleNotifications];
}

- (void)registerForAppLifecycleNotifications {
    NSNotificationCenter *center = [NSNotificationCenter defaultCenter];
    
    [center addObserver:self
               selector:@selector(appWillResignActive:)
                   name:UIApplicationWillResignActiveNotification
                 object:nil];
    
    [center addObserver:self
               selector:@selector(appDidBecomeActive:)
                   name:UIApplicationDidBecomeActiveNotification
                 object:nil];
    
    [center addObserver:self
               selector:@selector(appDidEnterBackground:)
                   name:UIApplicationDidEnterBackgroundNotification
                 object:nil];
    
    [center addObserver:self
               selector:@selector(appWillEnterForeground:)
                   name:UIApplicationWillEnterForegroundNotification
                 object:nil];
    
    [center addObserver:self
               selector:@selector(appWillTerminate:)
                   name:UIApplicationWillTerminateNotification
                 object:nil];
    
    // Memory warning
    [center addObserver:self
               selector:@selector(handleMemoryWarning:)
                   name:UIApplicationDidReceiveMemoryWarningNotification
                 object:nil];
}

- (void)appWillResignActive:(NSNotification *)notification {
    NSLog(@"[Notification] App will resign active");
    // หยุดวิดีโอ, เกม, animations
    [self.videoPlayer pause];
    [self.gameTimer invalidate];
}

- (void)appDidBecomeActive:(NSNotification *)notification {
    NSLog(@"[Notification] App did become active");
    // Resume
    [self.videoPlayer play];
    [self startGameTimer];
}

- (void)appDidEnterBackground:(NSNotification *)notification {
    NSLog(@"[Notification] App did enter background");
    [self saveCurrentState];
}

- (void)appWillEnterForeground:(NSNotification *)notification {
    NSLog(@"[Notification] App will enter foreground");
    [self prepareForForeground];
}

- (void)appWillTerminate:(NSNotification *)notification {
    NSLog(@"[Notification] App will terminate");
    [self performFinalSave];
}

- (void)handleMemoryWarning:(NSNotification *)notification {
    NSLog(@"[Notification] Memory warning received!");
    // ปล่อย cached data
    [self.imageCache removeAllObjects];
    [self.dataCache removeAllObjects];
}

- (void)dealloc {
    [[NSNotificationCenter defaultCenter] removeObserver:self];
}

@end
```

---

## 10. State Preservation and Restoration (การบันทึกและกู้คืนสถานะ)

### 10.1 Custom UIStateRestoring

```objc
// CustomRestorationClass.h
@interface CustomRestorationClass : NSObject <UIViewControllerRestoration>

@end

// CustomRestorationClass.m
@implementation CustomRestorationClass

+ (UIViewController *)viewControllerWithRestorationIdentifierPath:(NSArray *)identifierComponents 
                                                            coder:(NSCoder *)coder {
    NSString *storyboardName = [coder decodeObjectOfClass:[NSString class] 
                                                   forKey:@"storyboardName"];
    
    UIStoryboard *storyboard = [UIStoryboard storyboardWithName:storyboardName 
                                                         bundle:nil];
    
    NSString *identifier = identifierComponents.lastObject;
    UIViewController *vc = [storyboard instantiateViewControllerWithIdentifier:identifier];
    
    return vc;
}

@end
```

```objc
// ใน View Controller
@implementation ItemListViewController

- (void)viewDidLoad {
    [super viewDidLoad];
    
    // กำหนด restoration class
    self.restorationClass = [CustomRestorationClass class];
    self.restorationIdentifier = @"ItemListViewController";
}

- (void)encodeRestorableStateWithCoder:(NSCoder *)coder {
    [super encodeRestorableStateWithCoder:coder];
    
    // บันทึก storyboard name
    [coder encodeObject:@"Main" forKey:@"storyboardName"];
    
    // บันทึก current state
    [coder encodeInteger:self.selectedIndex forKey:@"selectedIndex"];
    [coder encodeObject:self.searchQuery forKey:@"searchQuery"];
    [coder encodeBool:self.isFilterActive forKey:@"isFilterActive"];
    
    NSLog(@"State encoded: index=%ld, query=%@", 
          (long)self.selectedIndex, self.searchQuery);
}

- (void)decodeRestorableStateWithCoder:(NSCoder *)coder {
    [super decodeRestorableStateWithCoder:coder];
    
    // กู้คืน state
    self.selectedIndex = [coder decodeIntegerForKey:@"selectedIndex"];
    self.searchQuery = [coder decodeObjectOfClass:[NSString class] 
                                           forKey:@"searchQuery"];
    self.isFilterActive = [coder decodeBoolForKey:@"isFilterActive"];
    
    NSLog(@"State decoded: index=%ld, query=%@", 
          (long)self.selectedIndex, self.searchQuery);
    
    // ปรับ UI ตาม restored state
    [self applyRestoredState];
}

- (void)applicationFinishedRestoringState {
    // เรียกหลังจาก restoration เสร็จสิ้น
    NSLog(@"Restoration finished, updating UI");
    [self.tableView reloadData];
    
    if (self.selectedIndex >= 0) {
        NSIndexPath *indexPath = [NSIndexPath indexPathForRow:self.selectedIndex 
                                                    inSection:0];
        [self.tableView selectRowAtIndexPath:indexPath 
                                    animated:NO 
                              scrollPosition:UITableViewScrollPositionMiddle];
    }
}

@end
```

---

## 11. Quick Actions (Home Screen Shortcuts)

Quick Actions ช่วยให้ผู้ใช้เข้าถึงฟังก์ชันสำคัญได้รวดเร็วจาก Home Screen

### 11.1 Static Quick Actions (Info.plist)

```xml
<key>UIApplicationShortcutItems</key>
<array>
    <dict>
        <key>UIApplicationShortcutItemType</key>
        <string>com.example.app.newMessage</string>
        <key>UIApplicationShortcutItemTitle</key>
        <string>New Message</string>
        <key>UIApplicationShortcutItemSubtitle</key>
        <string>Compose a new message</string>
        <key>UIApplicationShortcutItemIconType</key>
        <string>UIApplicationShortcutIconTypeCompose</string>
    </dict>
    <dict>
        <key>UIApplicationShortcutItemType</key>
        <string>com.example.app.search</string>
        <key>UIApplicationShortcutItemTitle</key>
        <string>Search</string>
        <key>UIApplicationShortcutItemIconType</key>
        <string>UIApplicationShortcutIconTypeSearch</string>
    </dict>
</array>
```

### 11.2 Dynamic Quick Actions

```objc
// สร้าง dynamic quick actions
- (void)updateQuickActions {
    NSMutableArray *shortcutItems = [NSMutableArray array];
    
    // เพิ่ม "Favorite Contacts" shortcuts
    NSArray *recentContacts = [self getRecentContacts];
    for (NSDictionary *contact in recentContacts) {
        UIApplicationShortcutIcon *icon = 
            [UIApplicationShortcutIcon iconWithSystemImageName:@"person.fill"];
        
        UIApplicationShortcutItem *item = 
            [[UIApplicationShortcutItem alloc] 
             initWithType:@"com.example.openContact"
             localizedTitle:contact[@"name"]
             localizedSubtitle:contact[@"phone"]
             icon:icon
             userInfo:@{@"contactId": contact[@"id"]}];
        
        [shortcutItems addObject:item];
        
        if (shortcutItems.count >= 3) break; // สูงสุด 4 items
    }
    
    // เพิ่ม static item
    UIApplicationShortcutIcon *newIcon = 
        [UIApplicationShortcutIcon iconWithSystemImageName:@"plus.circle.fill"];
    UIApplicationShortcutItem *newItem = 
        [[UIApplicationShortcutItem alloc] 
         initWithType:@"com.example.newContact"
         localizedTitle:@"New Contact"
         localizedSubtitle:nil
         icon:newIcon
         userInfo:nil];
    
    [shortcutItems addObject:newItem];
    
    [UIApplication sharedApplication].shortcutItems = shortcutItems;
    NSLog(@"Updated %lu quick actions", (unsigned long)shortcutItems.count);
}

// จัดการ shortcut item ที่ถูกเลือก
- (void)application:(UIApplication *)application 
    performActionForShortcutItem:(UIApplicationShortcutItem *)shortcutItem 
               completionHandler:(void (^)(BOOL))completionHandler {
    
    BOOL handled = [self handleShortcutItem:shortcutItem];
    completionHandler(handled);
}

- (BOOL)handleShortcutItem:(UIApplicationShortcutItem *)shortcutItem {
    NSLog(@"Handling shortcut: %@", shortcutItem.type);
    
    if ([shortcutItem.type isEqualToString:@"com.example.app.newMessage"]) {
        [self navigateToNewMessage];
        return YES;
    } else if ([shortcutItem.type isEqualToString:@"com.example.app.search"]) {
        [self navigateToSearch];
        return YES;
    } else if ([shortcutItem.type isEqualToString:@"com.example.openContact"]) {
        NSString *contactId = shortcutItem.userInfo[@"contactId"];
        [self openContactWithId:contactId];
        return YES;
    }
    
    return NO;
}
```

---

## 12. Handoff and Universal Links

### 12.1 Handoff

Handoff ช่วยให้ผู้ใช้ต่อเนื่องการทำงานระหว่าง devices ของ Apple ได้

```objc
// เปิดใช้ Handoff ใน View Controller
@implementation DocumentViewController

- (void)viewDidAppear:(BOOL)animated {
    [super viewDidAppear:animated];
    
    // สร้าง user activity สำหรับ Handoff
    NSUserActivity *activity = [[NSUserActivity alloc] 
                                 initWithActivityType:@"com.example.viewDocument"];
    activity.title = self.document.title;
    activity.userInfo = @{
        @"documentId": self.document.documentId,
        @"scrollPosition": @(self.textView.contentOffset.y)
    };
    
    // เปิดใช้ eligibleForHandoff
    activity.eligibleForHandoff = YES;
    activity.eligibleForSearch = YES;
    activity.eligibleForPublicIndexing = NO; // เป็น private content
    
    // กำหนด keywords สำหรับ Spotlight
    activity.keywords = [NSSet setWithArray:@[self.document.title, @"document"]];
    
    self.userActivity = activity;
    [activity becomeCurrent];
    
    NSLog(@"User activity set for Handoff: %@", self.document.title);
}

- (void)viewDidDisappear:(BOOL)animated {
    [super viewDidDisappear:animated];
    [self.userActivity resignCurrent];
}

// อัพเดท activity ขณะใช้งาน
- (void)updateUserActivity {
    [self.userActivity addUserInfoEntriesFromDictionary:@{
        @"scrollPosition": @(self.textView.contentOffset.y),
        @"cursorPosition": @(self.textView.selectedRange.location)
    }];
}

@end

// รับ Handoff ใน AppDelegate
- (BOOL)application:(UIApplication *)application 
    continueUserActivity:(NSUserActivity *)userActivity 
      restorationHandler:(void (^)(NSArray<id<UIUserActivityRestoring>> *))restorationHandler {
    
    NSLog(@"Continuing activity: %@", userActivity.activityType);
    
    if ([userActivity.activityType isEqualToString:@"com.example.viewDocument"]) {
        NSString *documentId = userActivity.userInfo[@"documentId"];
        NSNumber *scrollPosition = userActivity.userInfo[@"scrollPosition"];
        
        [self navigateToDocumentWithId:documentId scrollPosition:scrollPosition.floatValue];
        return YES;
    }
    
    return NO;
}
```

### 12.2 Universal Links

```objc
// รับ Universal Links
- (BOOL)application:(UIApplication *)application 
    continueUserActivity:(NSUserActivity *)userActivity 
      restorationHandler:(void (^)(NSArray<id<UIUserActivityRestoring>> *))restorationHandler {
    
    if ([userActivity.activityType isEqualToString:NSUserActivityTypeBrowsingWeb]) {
        NSURL *url = userActivity.webpageURL;
        NSLog(@"Universal link received: %@", url);
        
        return [self handleUniversalLink:url];
    }
    
    return NO;
}

- (BOOL)handleUniversalLink:(NSURL *)url {
    // Parse URL
    NSURLComponents *components = [NSURLComponents componentsWithURL:url 
                                                resolvingAgainstBaseURL:YES];
    
    NSString *path = components.path;
    
    // ตัวอย่าง: https://example.com/product/123
    if ([path hasPrefix:@"/product/"]) {
        NSString *productId = [path substringFromIndex:@"/product/".length];
        NSLog(@"Navigate to product: %@", productId);
        [self navigateToProductWithId:productId];
        return YES;
    }
    
    // ตัวอย่าง: https://example.com/user/john
    if ([path hasPrefix:@"/user/"]) {
        NSString *username = [path substringFromIndex:@"/user/".length];
        NSLog(@"Navigate to user: %@", username);
        [self navigateToUserProfile:username];
        return YES;
    }
    
    return NO;
}
```

```json
// apple-app-site-association (AASA) file
// ต้องวางที่ https://example.com/.well-known/apple-app-site-association
{
    "applinks": {
        "details": [
            {
                "appIDs": ["TEAMID.com.example.app"],
                "components": [
                    {
                        "/": "/product/*",
                        "comment": "Match any URL whose path starts with /product/"
                    },
                    {
                        "/": "/user/*",
                        "comment": "Match any URL whose path starts with /user/"
                    },
                    {
                        "/": "/",
                        "comment": "Match the home page"
                    }
                ]
            }
        ]
    }
}
```

---

## 13. แบบฝึกหัด (Practice Exercises)

### Exercise 1: App State Monitor

สร้าง class ที่ track และ log การเปลี่ยนแปลงสถานะทั้งหมด

```objc
// AppStateMonitor.h
@interface AppStateMonitor : NSObject

+ (instancetype)shared;

- (void)startMonitoring;
- (void)stopMonitoring;
- (NSArray<NSDictionary *> *)getStateHistory;

@end

// AppStateMonitor.m
@interface AppStateMonitor ()

@property (strong, nonatomic) NSMutableArray *stateHistory;
@property (assign, nonatomic) BOOL isMonitoring;

@end

@implementation AppStateMonitor

+ (instancetype)shared {
    static AppStateMonitor *instance = nil;
    static dispatch_once_t onceToken;
    dispatch_once(&onceToken, ^{
        instance = [[self alloc] init];
    });
    return instance;
}

- (instancetype)init {
    self = [super init];
    if (self) {
        _stateHistory = [NSMutableArray array];
    }
    return self;
}

- (void)startMonitoring {
    if (self.isMonitoring) return;
    self.isMonitoring = YES;
    
    NSNotificationCenter *center = [NSNotificationCenter defaultCenter];
    NSArray *notifications = @[
        UIApplicationDidBecomeActiveNotification,
        UIApplicationWillResignActiveNotification,
        UIApplicationDidEnterBackgroundNotification,
        UIApplicationWillEnterForegroundNotification,
        UIApplicationWillTerminateNotification
    ];
    
    for (NSString *name in notifications) {
        [center addObserver:self
                   selector:@selector(handleStateChange:)
                       name:name
                     object:nil];
    }
    
    NSLog(@"AppStateMonitor: Started monitoring");
}

- (void)stopMonitoring {
    [[NSNotificationCenter defaultCenter] removeObserver:self];
    self.isMonitoring = NO;
    NSLog(@"AppStateMonitor: Stopped monitoring");
}

- (void)handleStateChange:(NSNotification *)notification {
    NSDictionary *record = @{
        @"notification": notification.name,
        @"timestamp": [NSDate date],
        @"backgroundTimeRemaining": @([UIApplication sharedApplication].backgroundTimeRemaining)
    };
    
    [self.stateHistory addObject:record];
    NSLog(@"[StateMonitor] %@ at %@", notification.name, record[@"timestamp"]);
}

- (NSArray<NSDictionary *> *)getStateHistory {
    return [self.stateHistory copy];
}

- (void)dealloc {
    [self stopMonitoring];
}

@end
```

### Exercise 2: Launch Performance Tracker

```objc
// LaunchPerformanceTracker.h
@interface LaunchPerformanceTracker : NSObject

+ (void)markStart;
+ (void)markFirstFrame;
+ (void)markInteractive;
+ (void)printReport;

@end

// LaunchPerformanceTracker.m
@implementation LaunchPerformanceTracker

static CFTimeInterval sStartTime = 0;
static CFTimeInterval sFirstFrameTime = 0;
static CFTimeInterval sInteractiveTime = 0;

+ (void)markStart {
    sStartTime = CACurrentMediaTime();
    NSLog(@"[LaunchTracker] Start marked");
}

+ (void)markFirstFrame {
    sFirstFrameTime = CACurrentMediaTime();
    NSLog(@"[LaunchTracker] First frame: %.3f seconds from start", 
          sFirstFrameTime - sStartTime);
}

+ (void)markInteractive {
    sInteractiveTime = CACurrentMediaTime();
    NSLog(@"[LaunchTracker] Interactive: %.3f seconds from start", 
          sInteractiveTime - sStartTime);
}

+ (void)printReport {
    if (sStartTime == 0) {
        NSLog(@"[LaunchTracker] No data recorded");
        return;
    }
    
    NSLog(@"=== Launch Performance Report ===");
    
    if (sFirstFrameTime > 0) {
        NSLog(@"Time to first frame: %.3f seconds", sFirstFrameTime - sStartTime);
    }
    
    if (sInteractiveTime > 0) {
        NSLog(@"Time to interactive: %.3f seconds", sInteractiveTime - sStartTime);
    }
    
    // Apple recommendations
    NSLog(@"Recommendation: First frame < 0.4s, Interactive < 0.6s");
    
    if (sFirstFrameTime > 0 && (sFirstFrameTime - sStartTime) > 0.4) {
        NSLog(@"WARNING: First frame time exceeds recommendation");
    }
}

@end
```

### Exercise 3: State Restoration Helper

```objc
// StateRestorationHelper.h
@interface StateRestorationHelper : NSObject

+ (instancetype)shared;

- (void)saveState:(NSDictionary *)state forKey:(NSString *)key;
- (NSDictionary *)loadStateForKey:(NSString *)key;
- (void)clearStateForKey:(NSString *)key;
- (void)clearAllStates;

@end

// StateRestorationHelper.m
@implementation StateRestorationHelper

+ (instancetype)shared {
    static StateRestorationHelper *instance = nil;
    static dispatch_once_t onceToken;
    dispatch_once(&onceToken, ^{
        instance = [[self alloc] init];
    });
    return instance;
}

static NSString *const kStateStorageKey = @"AppStateStorage";

- (void)saveState:(NSDictionary *)state forKey:(NSString *)key {
    NSMutableDictionary *storage = [self loadAllStates];
    storage[key] = state;
    
    NSData *data = [NSKeyedArchiver archivedDataWithRootObject:storage 
                                        requiringSecureCoding:YES 
                                                        error:nil];
    [[NSUserDefaults standardUserDefaults] setObject:data forKey:kStateStorageKey];
    [[NSUserDefaults standardUserDefaults] synchronize];
    
    NSLog(@"State saved for key: %@", key);
}

- (NSDictionary *)loadStateForKey:(NSString *)key {
    NSMutableDictionary *storage = [self loadAllStates];
    return storage[key];
}

- (void)clearStateForKey:(NSString *)key {
    NSMutableDictionary *storage = [self loadAllStates];
    [storage removeObjectForKey:key];
    
    NSData *data = [NSKeyedArchiver archivedDataWithRootObject:storage 
                                        requiringSecureCoding:YES 
                                                        error:nil];
    [[NSUserDefaults standardUserDefaults] setObject:data forKey:kStateStorageKey];
}

- (void)clearAllStates {
    [[NSUserDefaults standardUserDefaults] removeObjectForKey:kStateStorageKey];
    NSLog(@"All states cleared");
}

- (NSMutableDictionary *)loadAllStates {
    NSData *data = [[NSUserDefaults standardUserDefaults] objectForKey:kStateStorageKey];
    if (!data) return [NSMutableDictionary dictionary];
    
    NSDictionary *storage = [NSKeyedUnarchiver unarchivedObjectOfClass:[NSDictionary class] 
                                                              fromData:data 
                                                                 error:nil];
    return storage ? [storage mutableCopy] : [NSMutableDictionary dictionary];
}

@end
```

---

## สรุป (Summary)

ในบทนี้เราได้เรียนรู้:

1. **UIApplication States** - 5 สถานะหลักของแอป iOS และการเปลี่ยนแปลงระหว่างสถานะ
2. **AppDelegate Methods** - เมธอดสำคัญสำหรับการจัดการ lifecycle events
3. **SceneDelegate** - Scene-based lifecycle สำหรับ iOS 13+ และการรองรับ multiple windows
4. **Background Execution** - การใช้ background tasks อย่างถูกต้อง
5. **State Restoration** - การบันทึกและกู้คืนสถานะของแอป
6. **Launch Optimization** - เทคนิคการปรับปรุงเวลาเปิดแอป
7. **Quick Actions** - Home screen shortcuts สำหรับเข้าถึงฟังก์ชันได้รวดเร็ว
8. **Handoff & Universal Links** - การทำงานร่วมกันระหว่าง devices และเว็บ

### Best Practices

- ทำงานน้อยที่สุดใน `didFinishLaunchingWithOptions:`
- ใช้ lazy initialization สำหรับ services ที่ไม่จำเป็นตอน launch
- บันทึก state ใน `applicationDidEnterBackground:`
- ใช้ Background Tasks อย่างระมัดระวังและ efficient
- ทดสอบ state restoration อย่างสม่ำเสมอ
- วัด launch time และ optimize อย่างต่อเนื่อง

---

*ต่อไป: Part 82 - Background Tasks (งานที่ทำในพื้นหลัง)*
