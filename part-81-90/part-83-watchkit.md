# Part 83: WatchKit Development (การพัฒนาแอปสำหรับ Apple Watch)

## บทนำ

Apple Watch เป็นอุปกรณ์ wearable ที่มีประสิทธิภาพสูง ซึ่งช่วยให้ผู้ใช้สามารถเข้าถึงข้อมูลและฟังก์ชันสำคัญได้อย่างรวดเร็วโดยไม่ต้องหยิบโทรศัพท์ขึ้นมา ในบทนี้เราจะเรียนรู้วิธีสร้างแอป watchOS ด้วย WatchKit, การสื่อสารระหว่าง Watch กับ iPhone, การสร้าง Complications, และการพัฒนา Workout apps

---

## 1. watchOS App Structure (โครงสร้างแอป watchOS)

### 1.1 โครงสร้างพื้นฐาน

watchOS app ประกอบด้วย 2 targets หลัก:
- **iOS App** - แอป iPhone ปกติ
- **watchOS App** - แอป Watch (มี WatchKit Extension)

```
MyApp (iOS Target)
├── AppDelegate.m
├── ViewControllers/
└── Resources/

MyApp WatchKit App (watchOS App Target)
└── Interface.storyboard

MyApp WatchKit Extension (watchOS Extension Target)
├── ExtensionDelegate.m
├── InterfaceControllers/
└── ComplicationController.m
```

### 1.2 ExtensionDelegate

```objc
// ExtensionDelegate.h
#import <WatchKit/WatchKit.h>

@interface ExtensionDelegate : NSObject <WKExtensionDelegate>

@end
```

```objc
// ExtensionDelegate.m
#import "ExtensionDelegate.h"
#import <WatchConnectivity/WatchConnectivity.h>

@implementation ExtensionDelegate

- (void)applicationDidFinishLaunching {
    NSLog(@"watchOS app launched");
    
    // ตั้งค่า WatchConnectivity
    if ([WCSession isSupported]) {
        WCSession *session = [WCSession defaultSession];
        session.delegate = self;
        [session activateSession];
        NSLog(@"WCSession activated");
    }
}

- (void)applicationDidBecomeActive {
    NSLog(@"Watch app became active");
}

- (void)applicationWillResignActive {
    NSLog(@"Watch app will resign active");
}

- (void)applicationWillEnterForeground {
    NSLog(@"Watch app entering foreground");
}

- (void)applicationDidEnterBackground {
    NSLog(@"Watch app entered background");
}

// Handle background refresh
- (void)handleBackgroundTasks:(NSSet<WKRefreshBackgroundTask *> *)backgroundTasks {
    for (WKRefreshBackgroundTask *task in backgroundTasks) {
        if ([task isKindOfClass:[WKApplicationRefreshBackgroundTask class]]) {
            // App refresh
            WKApplicationRefreshBackgroundTask *refreshTask = 
                (WKApplicationRefreshBackgroundTask *)task;
            [self handleAppRefreshTask:refreshTask];
            
        } else if ([task isKindOfClass:[WKSnapshotRefreshBackgroundTask class]]) {
            // UI snapshot
            WKSnapshotRefreshBackgroundTask *snapshotTask = 
                (WKSnapshotRefreshBackgroundTask *)task;
            [snapshotTask setTaskCompletedWithDefaultResult];
            
        } else if ([task isKindOfClass:[WKWatchConnectivityRefreshBackgroundTask class]]) {
            // Watch connectivity
            WKWatchConnectivityRefreshBackgroundTask *connectTask = 
                (WKWatchConnectivityRefreshBackgroundTask *)task;
            [connectTask setTaskCompletedWithSnapshot:NO];
            
        } else if ([task isKindOfClass:[WKURLSessionRefreshBackgroundTask class]]) {
            // URL session
            WKURLSessionRefreshBackgroundTask *urlTask = 
                (WKURLSessionRefreshBackgroundTask *)task;
            [self handleURLSessionTask:urlTask];
        } else {
            [task setTaskCompletedWithSnapshot:NO];
        }
    }
}

- (void)handleAppRefreshTask:(WKApplicationRefreshBackgroundTask *)task {
    // ดึงข้อมูลใหม่
    [[DataService shared] fetchDataWithCompletion:^(NSDictionary *data, NSError *error) {
        if (data) {
            // อัพเดท complications
            [[CLKComplicationServer sharedInstance] reloadTimelineForComplication:
             [[CLKComplicationServer sharedInstance].activeComplications firstObject]];
        }
        [task setTaskCompletedWithSnapshot:YES];
    }];
}

- (void)handleURLSessionTask:(WKURLSessionRefreshBackgroundTask *)task {
    NSURLSessionConfiguration *config = 
        [NSURLSessionConfiguration backgroundSessionConfigurationWithIdentifier:task.sessionIdentifier];
    NSURLSession *session = [NSURLSession sessionWithConfiguration:config 
                                                          delegate:nil 
                                                     delegateQueue:nil];
    [session finishTasksAndInvalidate];
    [task setTaskCompletedWithSnapshot:NO];
}

// ตั้งเวลา background refresh
- (void)scheduleBackgroundRefresh {
    NSDate *fireDate = [NSDate dateWithTimeIntervalSinceNow:30 * 60]; // 30 นาที
    
    [[WKExtension sharedExtension] scheduleBackgroundRefreshWithPreferredFireDate:fireDate
                                                                        userInfo:nil
                                                           scheduledCompletion:^(NSError *error) {
        if (error) {
            NSLog(@"Failed to schedule background refresh: %@", error);
        } else {
            NSLog(@"Background refresh scheduled for: %@", fireDate);
        }
    }];
}

@end
```

---

## 2. WKInterfaceController

WKInterfaceController เป็น base class สำหรับทุก interface ใน watchOS

```objc
// MainInterfaceController.h
#import <WatchKit/WatchKit.h>

@interface MainInterfaceController : WKInterfaceController

@end
```

```objc
// MainInterfaceController.m
#import "MainInterfaceController.h"
#import <WatchConnectivity/WatchConnectivity.h>

@interface MainInterfaceController () <WCSessionDelegate>

// IBOutlets เชื่อมกับ storyboard
@property (weak, nonatomic) IBOutlet WKInterfaceLabel *titleLabel;
@property (weak, nonatomic) IBOutlet WKInterfaceLabel *statusLabel;
@property (weak, nonatomic) IBOutlet WKInterfaceButton *actionButton;
@property (weak, nonatomic) IBOutlet WKInterfaceTable *dataTable;
@property (weak, nonatomic) IBOutlet WKInterfaceImage *iconImage;
@property (weak, nonatomic) IBOutlet WKInterfaceGroup *containerGroup;
@property (weak, nonatomic) IBOutlet WKInterfaceTimer *countdownTimer;

@property (strong, nonatomic) NSArray *dataItems;
@property (strong, nonatomic) NSDate *lastRefresh;

@end

@implementation MainInterfaceController

// เทียบเท่า viewDidLoad
- (void)awakeWithContext:(id)context {
    [super awakeWithContext:context];
    
    NSLog(@"Interface awoke with context: %@", context);
    
    // ตั้งค่า UI เริ่มต้น
    [self.titleLabel setText:@"My Watch App"];
    [self.statusLabel setText:@"กำลังโหลด..."];
    
    // รับ context ถ้ามี
    if ([context isKindOfClass:[NSDictionary class]]) {
        NSDictionary *dict = (NSDictionary *)context;
        [self.titleLabel setText:dict[@"title"]];
    }
    
    // ตั้งค่า WatchConnectivity
    [self setupWatchConnectivity];
}

// เทียบเท่า viewWillAppear
- (void)willActivate {
    [super willActivate];
    NSLog(@"Interface will activate");
    
    // รีเฟรชข้อมูล
    [self refreshData];
    
    // อัพเดท timer
    [self.countdownTimer setDate:[NSDate dateWithTimeIntervalSinceNow:60]];
    [self.countdownTimer start];
}

// เทียบเท่า viewDidDisappear
- (void)didDeactivate {
    [super didDeactivate];
    NSLog(@"Interface did deactivate");
    
    [self.countdownTimer stop];
}

- (void)setupWatchConnectivity {
    if ([WCSession isSupported]) {
        WCSession.defaultSession.delegate = self;
        [WCSession.defaultSession activateSession];
    }
}

- (void)refreshData {
    [self.statusLabel setText:@"กำลังอัพเดท..."];
    
    // ดึงข้อมูลจาก iPhone
    if (WCSession.defaultSession.isReachable) {
        [WCSession.defaultSession sendMessage:@{@"action": @"getData"}
                                 replyHandler:^(NSDictionary<NSString *,id> *replyMessage) {
            dispatch_async(dispatch_get_main_queue(), ^{
                [self updateWithData:replyMessage[@"data"]];
            });
        } errorHandler:^(NSError *error) {
            dispatch_async(dispatch_get_main_queue(), ^{
                [self.statusLabel setText:@"ไม่สามารถเชื่อมต่อ iPhone"];
            });
        }];
    } else {
        // ใช้ข้อมูล cache
        [self loadCachedData];
    }
}

- (void)updateWithData:(NSArray *)data {
    self.dataItems = data;
    self.lastRefresh = [NSDate date];
    
    [self.statusLabel setText:[NSString stringWithFormat:@"อัพเดทล่าสุด: %@",
                               [self formattedTime:self.lastRefresh]]];
    
    [self.dataTable setNumberOfRows:data.count withRowType:@"DataRow"];
    
    for (NSInteger i = 0; i < data.count; i++) {
        DataRowController *row = [self.dataTable rowControllerAtIndex:i];
        [row configureWithData:data[i]];
    }
}

- (void)loadCachedData {
    NSArray *cached = [[NSUserDefaults standardUserDefaults] objectForKey:@"cachedData"];
    if (cached) {
        [self updateWithData:cached];
        [self.statusLabel setText:@"ข้อมูล cache (ออฟไลน์)"];
    } else {
        [self.statusLabel setText:@"ไม่มีข้อมูล"];
    }
}

- (NSString *)formattedTime:(NSDate *)date {
    NSDateFormatter *formatter = [[NSDateFormatter alloc] init];
    formatter.timeStyle = NSDateFormatterShortStyle;
    return [formatter stringFromDate:date];
}

// IBAction
- (IBAction)actionButtonTapped {
    NSLog(@"Action button tapped");
    
    // แสดง menu หรือ push controller
    [self pushControllerWithName:@"DetailController" context:@{@"source": @"button"}];
}

// Table selection
- (void)table:(WKInterfaceTable *)table didSelectRowAtIndex:(NSInteger)rowIndex {
    NSLog(@"Row selected: %ld", (long)rowIndex);
    
    NSDictionary *item = self.dataItems[rowIndex];
    [self pushControllerWithName:@"ItemDetailController" context:item];
}

#pragma mark - WCSessionDelegate

- (void)session:(WCSession *)session 
    activationDidCompleteWithState:(WCSessionActivationState)activationState 
                             error:(NSError *)error {
    NSLog(@"WCSession activation: %ld, error: %@", (long)activationState, error);
}

- (void)session:(WCSession *)session 
    didReceiveMessage:(NSDictionary<NSString *,id> *)message {
    
    dispatch_async(dispatch_get_main_queue(), ^{
        NSLog(@"Received message: %@", message);
        
        if (message[@"data"]) {
            [self updateWithData:message[@"data"]];
        }
    });
}

- (void)session:(WCSession *)session 
    didReceiveApplicationContext:(NSDictionary<NSString *,id> *)applicationContext {
    
    dispatch_async(dispatch_get_main_queue(), ^{
        NSLog(@"Received application context: %@", applicationContext);
        // อัพเดท UI ด้วยข้อมูลใหม่
        if (applicationContext[@"userData"]) {
            [self updateWithData:applicationContext[@"userData"]];
        }
    });
}

@end
```

---

## 3. Watch UI Controls (ตัวควบคุม UI บน Watch)

### 3.1 WKInterfaceLabel

```objc
// การใช้งาน label
[self.titleLabel setText:@"สวัสดีชาว Watch!"];

// Text attributes
NSAttributedString *attrText = [[NSAttributedString alloc] 
    initWithString:@"Bold Text"
        attributes:@{
            NSFontAttributeName: [UIFont boldSystemFontOfSize:16],
            NSForegroundColorAttributeName: [UIColor yellowColor]
        }];
[self.titleLabel setAttributedText:attrText];

// Hidden/Visible
[self.titleLabel setHidden:NO];
[self.titleLabel setAlpha:0.8];
```

### 3.2 WKInterfaceButton

```objc
// การตั้งค่าปุ่ม
[self.actionButton setTitle:@"กดเลย"];
[self.actionButton setBackgroundColor:[UIColor systemBlueColor]];
[self.actionButton setEnabled:YES];
[self.actionButton setHidden:NO];

// ปุ่มกับรูปภาพ
[self.actionButton setBackgroundImage:[UIImage imageNamed:@"button_bg"]];
```

### 3.3 WKInterfaceTable

```objc
// RowController.h
@interface DataRowController : NSObject

@property (weak, nonatomic) IBOutlet WKInterfaceLabel *nameLabel;
@property (weak, nonatomic) IBOutlet WKInterfaceLabel *valueLabel;
@property (weak, nonatomic) IBOutlet WKInterfaceImage *iconImage;

- (void)configureWithData:(NSDictionary *)data;

@end

// RowController.m
@implementation DataRowController

- (void)configureWithData:(NSDictionary *)data {
    [self.nameLabel setText:data[@"name"]];
    [self.valueLabel setText:data[@"value"]];
    
    NSString *imageName = data[@"icon"] ?: @"defaultIcon";
    [self.iconImage setImageNamed:imageName];
    
    // Color based on value
    NSInteger value = [data[@"numericValue"] integerValue];
    UIColor *color = value > 50 ? [UIColor greenColor] : [UIColor redColor];
    [self.valueLabel setTextColor:color];
}

@end
```

### 3.4 WKInterfaceImage สำหรับ Animations

```objc
// แสดงรูปภาพ
[self.iconImage setImageNamed:@"heart"];
[self.iconImage setTintColor:[UIColor redColor]];

// Animation
[self.iconImage setImageNamed:@"heartbeat"];
[self.iconImage startAnimatingWithImagesInRange:NSMakeRange(0, 10)
                                       duration:1.0
                                    repeatCount:0]; // 0 = infinite

// หยุด animation
[self.iconImage stopAnimating];
```

### 3.5 WKInterfacePicker

```objc
// Picker (Digital Crown)
@property (weak, nonatomic) IBOutlet WKInterfacePicker *valuePicker;

- (void)setupPicker {
    NSMutableArray *items = [NSMutableArray array];
    
    for (NSInteger i = 0; i <= 100; i += 5) {
        WKPickerItem *item = [[WKPickerItem alloc] init];
        item.title = [NSString stringWithFormat:@"%ld%%", (long)i];
        item.caption = [NSString stringWithFormat:@"ระดับ %ld", (long)i];
        [items addObject:item];
    }
    
    [self.valuePicker setItems:items];
    [self.valuePicker setSelectedItemIndex:10]; // เริ่มที่ 50%
    [self.valuePicker focus]; // เปิดใช้ Digital Crown
}

- (IBAction)pickerDidChange:(NSInteger)value {
    NSLog(@"Picker value: %ld", (long)value);
    // อัพเดท UI
}
```

### 3.6 WKInterfaceSlider

```objc
@property (weak, nonatomic) IBOutlet WKInterfaceSlider *volumeSlider;

- (void)setupSlider {
    [self.volumeSlider setNumberOfSteps:20];
    [self.volumeSlider setValue:0.5];
    [self.volumeSlider setColor:[UIColor systemBlueColor]];
}

- (IBAction)sliderValueChanged:(float)value {
    NSLog(@"Slider: %.2f", value);
}
```

---

## 4. WatchConnectivity Framework

WatchConnectivity เป็น framework สำหรับการสื่อสารระหว่าง iPhone และ Apple Watch

### 4.1 WCSession States

```
Paired: iPhone และ Watch จับคู่กันแล้ว
Activated: Session พร้อมใช้งาน
Reachable: อุปกรณ์อีกข้างเข้าถึงได้ทันที (foreground)
```

### 4.2 iOS Side - WCSession Setup

```objc
// WatchSessionManager.h (iOS)
#import <WatchConnectivity/WatchConnectivity.h>

@interface WatchSessionManager : NSObject <WCSessionDelegate>

+ (instancetype)shared;
- (void)startSession;
- (void)sendDataToWatch:(NSDictionary *)data;
- (void)updateApplicationContext:(NSDictionary *)context;

@end
```

```objc
// WatchSessionManager.m (iOS)
@interface WatchSessionManager ()

@property (strong, nonatomic) WCSession *session;

@end

@implementation WatchSessionManager

+ (instancetype)shared {
    static WatchSessionManager *instance = nil;
    static dispatch_once_t onceToken;
    dispatch_once(&onceToken, ^{
        instance = [[self alloc] init];
    });
    return instance;
}

- (void)startSession {
    if (![WCSession isSupported]) {
        NSLog(@"WatchConnectivity not supported");
        return;
    }
    
    self.session = [WCSession defaultSession];
    self.session.delegate = self;
    [self.session activateSession];
    
    NSLog(@"Watch session started");
}

// ส่งข้อมูลแบบ real-time (ต้องการทั้งคู่ active)
- (void)sendMessageToWatch:(NSDictionary *)message 
              withReply:(void(^)(NSDictionary *reply))replyHandler {
    
    if (!self.session.isReachable) {
        NSLog(@"Watch not reachable");
        return;
    }
    
    [self.session sendMessage:message
                 replyHandler:^(NSDictionary<NSString *,id> *replyMessage) {
        NSLog(@"Watch replied: %@", replyMessage);
        if (replyHandler) {
            replyHandler(replyMessage);
        }
    } errorHandler:^(NSError *error) {
        NSLog(@"Send message error: %@", error.localizedDescription);
    }];
}

// อัพเดท application context (sync เมื่อ watch เปิดแอป)
- (void)updateApplicationContext:(NSDictionary *)context {
    NSError *error;
    BOOL success = [self.session updateApplicationContext:context error:&error];
    
    if (!success) {
        NSLog(@"Failed to update context: %@", error.localizedDescription);
    } else {
        NSLog(@"Application context updated: %@", context.allKeys);
    }
}

// ส่ง user info (แบบ queued)
- (void)transferUserInfo:(NSDictionary *)userInfo {
    WCSessionUserInfoTransfer *transfer = [self.session transferUserInfo:userInfo];
    NSLog(@"User info transfer started: %@", transfer.userInfo.allKeys);
}

// ส่งไฟล์ไปยัง Watch
- (void)transferFile:(NSURL *)fileURL metadata:(NSDictionary *)metadata {
    WCSessionFileTransfer *transfer = [self.session transferFile:fileURL metadata:metadata];
    NSLog(@"File transfer started: %@", fileURL.lastPathComponent);
}

#pragma mark - WCSessionDelegate

- (void)session:(WCSession *)session 
    activationDidCompleteWithState:(WCSessionActivationState)activationState 
                             error:(NSError *)error {
    
    NSString *stateStr;
    switch (activationState) {
        case WCSessionActivationStateActivated: stateStr = @"Activated"; break;
        case WCSessionActivationStateInactive: stateStr = @"Inactive"; break;
        case WCSessionActivationStateNotActivated: stateStr = @"Not Activated"; break;
    }
    
    NSLog(@"WCSession activation: %@, error: %@", stateStr, error);
}

- (void)sessionDidBecomeInactive:(WCSession *)session {
    NSLog(@"Watch session inactive");
}

- (void)sessionDidDeactivate:(WCSession *)session {
    NSLog(@"Watch session deactivated");
    [self.session activateSession]; // reactivate สำหรับ watch switching
}

- (void)sessionWatchStateDidChange:(WCSession *)session {
    NSLog(@"Watch state: paired=%d, installed=%d, reachable=%d",
          session.isPaired, session.isWatchAppInstalled, session.isReachable);
}

// รับข้อความจาก Watch
- (void)session:(WCSession *)session 
    didReceiveMessage:(NSDictionary<NSString *,id> *)message 
         replyHandler:(void (^)(NSDictionary<NSString *,id> *))replyHandler {
    
    NSLog(@"Message from Watch: %@", message);
    
    NSString *action = message[@"action"];
    
    if ([action isEqualToString:@"getData"]) {
        // ส่งข้อมูลกลับไป
        NSArray *data = [[DataManager shared] getLatestData];
        replyHandler(@{@"data": data});
        
    } else if ([action isEqualToString:@"syncHealth"]) {
        // sync health data
        [[HealthKitManager shared] syncHealthData:^(BOOL success) {
            replyHandler(@{@"success": @(success)});
        }];
    }
}

// รับ user info จาก Watch
- (void)session:(WCSession *)session 
    didReceiveUserInfo:(NSDictionary<NSString *,id> *)userInfo {
    NSLog(@"User info from Watch: %@", userInfo);
}

// รับไฟล์จาก Watch
- (void)session:(WCSession *)session 
    didReceiveFile:(WCSessionFile *)file {
    NSLog(@"File from Watch: %@, metadata: %@", 
          file.fileURL.lastPathComponent, file.metadata);
}

@end
```

---

## 5. WCSession - การส่งข้อมูล

### 5.1 ประเภทการส่งข้อมูล

```objc
// 1. sendMessage - real-time, ต้องการทั้งคู่ active
// ใช้เมื่อ: ต้องการ response ทันที, ทั้งคู่อยู่ foreground
[session sendMessage:@{@"key": @"value"}
        replyHandler:^(NSDictionary *reply) { ... }
        errorHandler:^(NSError *error) { ... }];

// 2. transferUserInfo - guaranteed delivery, queued
// ใช้เมื่อ: ไม่ต้องการ response ทันที, ข้อมูลสำคัญ
[session transferUserInfo:@{@"key": @"value"}];

// 3. updateApplicationContext - ข้อมูลล่าสุดเท่านั้น
// ใช้เมื่อ: sync state ปัจจุบัน, ถ้าส่งหลายครั้งจะ overwrite
[session updateApplicationContext:@{@"key": @"value"} error:nil];

// 4. transferFile - ส่งไฟล์ขนาดใหญ่
// ใช้เมื่อ: ส่งรูปภาพ, audio, data files
[session transferFile:fileURL metadata:@{@"type": @"photo"}];

// 5. transferCurrentComplicationUserInfo - อัพเดท complication ทันที
// ใช้เมื่อ: อัพเดท watch face complication
if ([session isComplicationEnabled]) {
    [session transferCurrentComplicationUserInfo:@{@"steps": @(10000)}];
}
```

### 5.2 Watch Side - รับข้อมูล

```objc
// WatchSessionDelegate.m (watchOS)
- (void)session:(WCSession *)session 
    didReceiveMessage:(NSDictionary<NSString *,id> *)message {
    
    dispatch_async(dispatch_get_main_queue(), ^{
        NSLog(@"Message from iPhone: %@", message);
        
        // อัพเดท UI
        if (message[@"title"]) {
            [self.titleLabel setText:message[@"title"]];
        }
    });
}

- (void)session:(WCSession *)session 
    didReceiveApplicationContext:(NSDictionary<NSString *,id> *)applicationContext {
    
    dispatch_async(dispatch_get_main_queue(), ^{
        NSLog(@"App context updated: %@", applicationContext);
        // บันทึก context สำหรับแสดงผล
        [[NSUserDefaults standardUserDefaults] setObject:applicationContext 
                                                  forKey:@"watchAppContext"];
    });
}

- (void)session:(WCSession *)session 
    didReceiveUserInfo:(NSDictionary<NSString *,id> *)userInfo {
    
    dispatch_async(dispatch_get_main_queue(), ^{
        // ประมวลผล user info
        if (userInfo[@"workoutData"]) {
            [[WorkoutManager shared] processWorkoutData:userInfo[@"workoutData"]];
        }
    });
}
```

---

## 6. Complication Development (การพัฒนา Complications)

Complications คือ widget เล็กๆ บน Watch face ที่แสดงข้อมูลจากแอป

### 6.1 ประเภทของ Complications

```objc
// Complication families
CLKComplicationFamilyModularSmall    // กล่องเล็กบน Modular face
CLKComplicationFamilyModularLarge    // กล่องใหญ่บน Modular face  
CLKComplicationFamilyUtilitarianSmall // Utilitarian small
CLKComplicationFamilyUtilitarianSmallFlat // Utilitarian flat
CLKComplicationFamilyUtilitarianLarge // Utilitarian large
CLKComplicationFamilyCircularSmall   // วงกลมเล็กบน Utility/Simple face
CLKComplicationFamilyExtraLarge      // วงกลมใหญ่บน X-Large face
CLKComplicationFamilyGraphicCorner   // มุมบน Infograph face
CLKComplicationFamilyGraphicCircular // วงกลมบน Infograph face
CLKComplicationFamilyGraphicRectangular // สี่เหลี่ยมบน Infograph face
CLKComplicationFamilyGraphicBezel    // เส้นขอบ Infograph face
CLKComplicationFamilyGraphicExtraLarge // Extra large Graphic
```

### 6.2 ComplicationController

```objc
// ComplicationController.h
#import <ClockKit/ClockKit.h>

@interface ComplicationController : NSObject <CLKComplicationDataSource>

@end
```

```objc
// ComplicationController.m
#import "ComplicationController.h"

@implementation ComplicationController

#pragma mark - Timeline Configuration

// Timeline กี่รายการล่วงหน้า
- (void)getTimelineEndDateForComplication:(CLKComplication *)complication 
                              withHandler:(void (^)(NSDate *))handler {
    // ให้ข้อมูลได้ล่วงหน้า 8 ชั่วโมง
    handler([NSDate dateWithTimeIntervalSinceNow:8 * 3600]);
}

// ข้อมูลหมดอายุหรือไม่
- (void)getPrivacyBehaviorForComplication:(CLKComplication *)complication 
                              withHandler:(void (^)(CLKComplicationPrivacyBehavior))handler {
    handler(CLKComplicationPrivacyBehaviorShowOnLockScreen);
}

#pragma mark - Timeline Population

// Current time entry
- (void)getCurrentTimelineEntryForComplication:(CLKComplication *)complication 
                                   withHandler:(void(^)(CLKComplicationTimelineEntry *))handler {
    
    CLKComplicationTimelineEntry *entry = [self createEntryForDate:[NSDate date] 
                                                       complication:complication];
    handler(entry);
}

// Future entries
- (void)getTimelineEntriesForComplication:(CLKComplication *)complication 
                                afterDate:(NSDate *)date 
                                    limit:(NSUInteger)limit 
                              withHandler:(void(^)(NSArray<CLKComplicationTimelineEntry *> *))handler {
    
    NSMutableArray *entries = [NSMutableArray array];
    
    // สร้าง entries สำหรับทุกชั่วโมงในอนาคต
    NSDate *entryDate = [NSDate dateWithTimeIntervalSinceDate:date timeInterval:3600];
    
    for (NSUInteger i = 0; i < limit && i < 8; i++) {
        CLKComplicationTimelineEntry *entry = [self createEntryForDate:entryDate 
                                                          complication:complication];
        if (entry) {
            [entries addObject:entry];
        }
        entryDate = [NSDate dateWithTimeIntervalSinceDate:entryDate timeInterval:3600];
    }
    
    handler(entries);
}

- (CLKComplicationTimelineEntry *)createEntryForDate:(NSDate *)date 
                                         complication:(CLKComplication *)complication {
    
    CLKComplicationTemplate *template = [self createTemplateForFamily:complication.family 
                                                                 date:date];
    if (!template) return nil;
    
    return [CLKComplicationTimelineEntry entryWithDate:date complicationTemplate:template];
}

- (CLKComplicationTemplate *)createTemplateForFamily:(CLKComplicationFamily)family 
                                                date:(NSDate *)date {
    
    // ข้อมูลสำหรับแสดงใน complication
    NSInteger steps = [self getStepsForDate:date];
    NSInteger goal = 10000;
    float progress = (float)steps / goal;
    
    switch (family) {
        case CLKComplicationFamilyModularSmall: {
            CLKComplicationTemplateModularSmallStackText *template = 
                [[CLKComplicationTemplateModularSmallStackText alloc] init];
            template.line1TextProvider = [CLKSimpleTextProvider textProviderWithText:@"ก้าว"];
            template.line2TextProvider = [CLKSimpleTextProvider 
                textProviderWithText:[NSString stringWithFormat:@"%ld", (long)steps]];
            template.highlightLine2 = YES;
            return template;
        }
            
        case CLKComplicationFamilyModularLarge: {
            CLKComplicationTemplateModularLargeColumns *template = 
                [[CLKComplicationTemplateModularLargeColumns alloc] init];
            template.row1Column1TextProvider = [CLKSimpleTextProvider textProviderWithText:@"ก้าว"];
            template.row1Column2TextProvider = [CLKSimpleTextProvider 
                textProviderWithText:[NSString stringWithFormat:@"%ld", (long)steps]];
            template.row2Column1TextProvider = [CLKSimpleTextProvider textProviderWithText:@"เป้าหมาย"];
            template.row2Column2TextProvider = [CLKSimpleTextProvider 
                textProviderWithText:[NSString stringWithFormat:@"%ld", (long)goal]];
            return template;
        }
            
        case CLKComplicationFamilyCircularSmall: {
            CLKComplicationTemplateCircularSmallRingText *template = 
                [[CLKComplicationTemplateCircularSmallRingText alloc] init];
            template.textProvider = [CLKSimpleTextProvider 
                textProviderWithText:[NSString stringWithFormat:@"%ldK", (long)(steps/1000)]];
            template.fillFraction = progress;
            template.ringStyle = CLKComplicationRingStyleClosed;
            return template;
        }
            
        case CLKComplicationFamilyGraphicCircular: {
            if (@available(watchOS 5.0, *)) {
                CLKComplicationTemplateGraphicCircularClosedGaugeText *template = 
                    [[CLKComplicationTemplateGraphicCircularClosedGaugeText alloc] init];
                template.centerTextProvider = [CLKSimpleTextProvider 
                    textProviderWithText:[NSString stringWithFormat:@"%ldK", (long)(steps/1000)]];
                template.gaugeProvider = [CLKSimpleGaugeProvider 
                    gaugeProviderWithStyle:CLKGaugeProviderStyleRing
                             gaugeColors:@[[UIColor greenColor], [UIColor yellowColor], [UIColor redColor]]
                      gaugeColorLocations:@[@0.0, @0.5, @1.0]
                            fillFraction:progress];
                return template;
            }
            return nil;
        }
            
        default:
            return nil;
    }
}

- (NSInteger)getStepsForDate:(NSDate *)date {
    // ดึงข้อมูลจาก HealthKit หรือ cache
    return [[NSUserDefaults standardUserDefaults] integerForKey:@"todaySteps"] ?: 7500;
}

#pragma mark - Placeholder Templates

- (void)getPlaceholderTemplateForComplication:(CLKComplication *)complication 
                                  withHandler:(void (^)(CLKComplicationTemplate *))handler {
    
    CLKComplicationTemplate *template = [self createTemplateForFamily:complication.family 
                                                                 date:[NSDate date]];
    handler(template);
}

#pragma mark - Localized Sample Data

- (void)getLocalizableSampleTemplateForComplication:(CLKComplication *)complication 
                                        withHandler:(void (^)(CLKComplicationTemplate *))handler {
    handler([self createTemplateForFamily:complication.family date:[NSDate date]]);
}

@end
```

---

## 7. ClockKit - อัพเดท Complications

```objc
// อัพเดท complication เมื่อข้อมูลเปลี่ยน
- (void)reloadComplications {
    CLKComplicationServer *server = [CLKComplicationServer sharedInstance];
    
    for (CLKComplication *complication in server.activeComplications) {
        [server reloadTimelineForComplication:complication];
        NSLog(@"Reloaded complication: %ld", (long)complication.family);
    }
}

// อัพเดทจาก iPhone ผ่าน WatchConnectivity
// ใน iOS app
- (void)sendComplicationUpdate:(NSDictionary *)data {
    WCSession *session = [WCSession defaultSession];
    
    if ([session isComplicationEnabled] && session.isReachable) {
        // ส่งทันที (เครดิต จำกัด 50 ครั้ง/วัน)
        [session transferCurrentComplicationUserInfo:data];
        NSLog(@"Complication update sent");
    } else {
        // ส่งแบบ queued
        [session transferUserInfo:data];
    }
}

// ใน watchOS - รับอัพเดท complication
- (void)session:(WCSession *)session 
    didReceiveUserInfo:(NSDictionary<NSString *,id> *)userInfo {
    
    if (userInfo[@"steps"]) {
        NSInteger steps = [userInfo[@"steps"] integerValue];
        [[NSUserDefaults standardUserDefaults] setInteger:steps forKey:@"todaySteps"];
        
        // รีโหลด complication
        CLKComplicationServer *server = [CLKComplicationServer sharedInstance];
        for (CLKComplication *complication in server.activeComplications) {
            [server reloadTimelineForComplication:complication];
        }
    }
}
```

---

## 8. Background Updates on Watch (อัพเดทพื้นหลัง)

```objc
// ExtensionDelegate.m

// ตั้งเวลา background refresh
- (void)scheduleNextBackgroundRefresh {
    // watchOS อนุญาตให้ refresh ทุก 30 นาที
    NSDate *nextRefresh = [NSDate dateWithTimeIntervalSinceNow:30 * 60];
    
    [[WKExtension sharedExtension] 
     scheduleBackgroundRefreshWithPreferredFireDate:nextRefresh
     userInfo:@{@"reason": @"dataRefresh"}
     scheduledCompletion:^(NSError *error) {
        if (error) {
            NSLog(@"Failed to schedule: %@", error);
        } else {
            NSLog(@"Next refresh at: %@", nextRefresh);
        }
    }];
}

// จัดการ background task
- (void)handleAppRefreshTask:(WKApplicationRefreshBackgroundTask *)task {
    NSLog(@"Background refresh started");
    
    // ตั้งเวลาครั้งต่อไป
    [self scheduleNextBackgroundRefresh];
    
    // ดึงข้อมูลผ่าน URL session
    NSURL *url = [NSURL URLWithString:@"https://api.example.com/watch/data"];
    NSURLSession *session = [NSURLSession sharedSession];
    
    NSURLSessionDataTask *dataTask = [session dataTaskWithURL:url 
                                           completionHandler:^(NSData *data, NSURLResponse *response, NSError *error) {
        if (data && !error) {
            NSDictionary *json = [NSJSONSerialization JSONObjectWithData:data options:0 error:nil];
            
            // อัพเดทข้อมูล
            [[NSUserDefaults standardUserDefaults] setObject:json forKey:@"watchData"];
            
            // รีโหลด complication
            dispatch_async(dispatch_get_main_queue(), ^{
                CLKComplicationServer *server = [CLKComplicationServer sharedInstance];
                for (CLKComplication *comp in server.activeComplications) {
                    [server reloadTimelineForComplication:comp];
                }
            });
            
            [task setTaskCompletedWithSnapshot:YES];
        } else {
            [task setTaskCompletedWithSnapshot:NO];
        }
    }];
    
    [dataTask resume];
}

// ตั้งเวลา snapshot
- (void)scheduleSnapshot {
    [[WKExtension sharedExtension] scheduleSnapshotRefreshWithPreferredFireDate:[NSDate date]
                                                                       userInfo:nil
                                                          scheduledCompletion:^(NSError *error) {
        NSLog(@"Snapshot refresh scheduled");
    }];
}
```

---

## 9. Workout Apps (แอปออกกำลังกาย)

### 9.1 HealthKit Authorization

```objc
// WorkoutManager.h
#import <HealthKit/HealthKit.h>
#import <WatchKit/WatchKit.h>

@interface WorkoutManager : NSObject <HKWorkoutSessionDelegate, HKLiveWorkoutBuilderDelegate>

+ (instancetype)shared;
- (void)requestAuthorization:(void(^)(BOOL success))completion;
- (void)startWorkout:(HKWorkoutActivityType)activityType;
- (void)pauseWorkout;
- (void)resumeWorkout;
- (void)endWorkout;

@property (copy, nonatomic) void (^heartRateUpdateHandler)(double heartRate);
@property (copy, nonatomic) void (^caloriesUpdateHandler)(double calories);

@end
```

```objc
// WorkoutManager.m
@interface WorkoutManager ()

@property (strong, nonatomic) HKHealthStore *healthStore;
@property (strong, nonatomic) HKWorkoutSession *workoutSession;
@property (strong, nonatomic) HKLiveWorkoutBuilder *builder;
@property (assign, nonatomic) NSTimeInterval elapsedTime;

@end

@implementation WorkoutManager

+ (instancetype)shared {
    static WorkoutManager *instance = nil;
    static dispatch_once_t onceToken;
    dispatch_once(&onceToken, ^{
        instance = [[self alloc] init];
    });
    return instance;
}

- (instancetype)init {
    self = [super init];
    if (self) {
        _healthStore = [[HKHealthStore alloc] init];
    }
    return self;
}

- (void)requestAuthorization:(void(^)(BOOL success))completion {
    if (![HKHealthStore isHealthDataAvailable]) {
        completion(NO);
        return;
    }
    
    // ประเภทข้อมูลที่ต้องการอ่าน
    NSSet *typesToRead = [NSSet setWithObjects:
        [HKObjectType quantityTypeForIdentifier:HKQuantityTypeIdentifierHeartRate],
        [HKObjectType quantityTypeForIdentifier:HKQuantityTypeIdentifierActiveEnergyBurned],
        [HKObjectType quantityTypeForIdentifier:HKQuantityTypeIdentifierDistanceWalkingRunning],
        [HKObjectType workoutType],
        nil];
    
    // ประเภทข้อมูลที่ต้องการเขียน
    NSSet *typesToShare = [NSSet setWithObjects:
        [HKObjectType workoutType],
        [HKObjectType quantityTypeForIdentifier:HKQuantityTypeIdentifierActiveEnergyBurned],
        nil];
    
    [self.healthStore requestAuthorizationToShareTypes:typesToShare
                                            readTypes:typesToRead
                                           completion:^(BOOL success, NSError *error) {
        dispatch_async(dispatch_get_main_queue(), ^{
            if (success) {
                NSLog(@"HealthKit authorization granted");
            } else {
                NSLog(@"HealthKit authorization failed: %@", error);
            }
            completion(success);
        });
    }];
}

- (void)startWorkout:(HKWorkoutActivityType)activityType {
    // สร้าง workout configuration
    HKWorkoutConfiguration *config = [[HKWorkoutConfiguration alloc] init];
    config.activityType = activityType;
    config.locationType = HKWorkoutSessionLocationTypeOutdoor;
    
    NSError *error;
    self.workoutSession = [[HKWorkoutSession alloc] initWithHealthStore:self.healthStore
                                                          configuration:config
                                                                  error:&error];
    if (error) {
        NSLog(@"Failed to create workout session: %@", error);
        return;
    }
    
    self.workoutSession.delegate = self;
    
    // สร้าง live workout builder
    self.builder = [self.workoutSession associatedWorkoutBuilder];
    self.builder.delegate = self;
    self.builder.dataSource = [[HKLiveWorkoutDataSource alloc] initWithHealthStore:self.healthStore
                                                                    workoutSession:self.workoutSession];
    
    // เริ่ม workout
    NSDate *startDate = [NSDate date];
    [self.workoutSession startActivityWithDate:startDate];
    [self.builder beginCollectionWithStart:startDate completion:^(BOOL success, NSError *error) {
        if (success) {
            NSLog(@"Workout started: %@", 
                  [self workoutTypeString:activityType]);
        }
    }];
}

- (void)pauseWorkout {
    [self.workoutSession pause];
}

- (void)resumeWorkout {
    [self.workoutSession resume];
}

- (void)endWorkout {
    NSDate *endDate = [NSDate date];
    
    [self.workoutSession stopActivityWithDate:endDate];
    [self.builder endCollectionWithEnd:endDate completion:^(BOOL success, NSError *error) {
        if (success) {
            [self.builder finishWorkoutWithCompletion:^(HKWorkout *workout, NSError *error) {
                if (workout) {
                    NSLog(@"Workout saved: duration=%.0fs, calories=%.0f", 
                          workout.duration,
                          [workout.totalEnergyBurned doubleValueForUnit:[HKUnit kilocalorieUnit]]);
                }
            }];
        }
    }];
}

- (NSString *)workoutTypeString:(HKWorkoutActivityType)type {
    switch (type) {
        case HKWorkoutActivityTypeRunning: return @"Running";
        case HKWorkoutActivityTypeCycling: return @"Cycling";
        case HKWorkoutActivityTypeWalking: return @"Walking";
        case HKWorkoutActivityTypeSwimming: return @"Swimming";
        default: return @"Workout";
    }
}

#pragma mark - HKWorkoutSessionDelegate

- (void)workoutSession:(HKWorkoutSession *)workoutSession 
      didChangeTo:(HKWorkoutSessionState)toState 
         fromState:(HKWorkoutSessionState)fromState 
               date:(NSDate *)date {
    
    NSLog(@"Workout state: %ld -> %ld", (long)fromState, (long)toState);
    
    dispatch_async(dispatch_get_main_queue(), ^{
        [[NSNotificationCenter defaultCenter] 
         postNotificationName:@"WorkoutStateChanged"
         object:nil
         userInfo:@{@"state": @(toState)}];
    });
}

- (void)workoutSession:(HKWorkoutSession *)workoutSession 
      didFailWithError:(NSError *)error {
    NSLog(@"Workout session failed: %@", error);
}

#pragma mark - HKLiveWorkoutBuilderDelegate

- (void)workoutBuilder:(HKLiveWorkoutBuilder *)workoutBuilder 
    didCollectDataOfTypes:(NSSet<HKSampleType *> *)collectedTypes {
    
    // อัพเดท heart rate
    HKQuantityType *heartRateType = [HKObjectType quantityTypeForIdentifier:HKQuantityTypeIdentifierHeartRate];
    
    if ([collectedTypes containsObject:heartRateType]) {
        HKQuantity *quantity = [workoutBuilder.elapsedTime > 0 ? 
            [workoutBuilder statisticsForType:heartRateType] : nil mostRecentQuantity];
        
        if (quantity) {
            double heartRate = [quantity doubleValueForUnit:
                [HKUnit unitFromString:@"count/min"]];
            
            dispatch_async(dispatch_get_main_queue(), ^{
                if (self.heartRateUpdateHandler) {
                    self.heartRateUpdateHandler(heartRate);
                }
            });
        }
    }
    
    // อัพเดท calories
    HKQuantityType *caloriesType = [HKObjectType quantityTypeForIdentifier:HKQuantityTypeIdentifierActiveEnergyBurned];
    
    if ([collectedTypes containsObject:caloriesType]) {
        HKQuantity *quantity = [[workoutBuilder statisticsForType:caloriesType] sumQuantity];
        
        if (quantity) {
            double calories = [quantity doubleValueForUnit:[HKUnit kilocalorieUnit]];
            
            dispatch_async(dispatch_get_main_queue(), ^{
                if (self.caloriesUpdateHandler) {
                    self.caloriesUpdateHandler(calories);
                }
            });
        }
    }
}

- (void)workoutBuilderDidCollectEvent:(HKLiveWorkoutBuilder *)workoutBuilder {
    NSLog(@"Workout event collected");
}

@end
```

### 9.2 Workout Interface Controller

```objc
// WorkoutInterfaceController.m
@interface WorkoutInterfaceController ()

@property (weak, nonatomic) IBOutlet WKInterfaceLabel *heartRateLabel;
@property (weak, nonatomic) IBOutlet WKInterfaceLabel *caloriesLabel;
@property (weak, nonatomic) IBOutlet WKInterfaceLabel *durationLabel;
@property (weak, nonatomic) IBOutlet WKInterfaceTimer *durationTimer;
@property (weak, nonatomic) IBOutlet WKInterfaceButton *pauseButton;

@property (strong, nonatomic) NSTimer *displayTimer;
@property (assign, nonatomic) BOOL isPaused;

@end

@implementation WorkoutInterfaceController

- (void)awakeWithContext:(id)context {
    [super awakeWithContext:context];
    
    [self setupWorkoutHandlers];
}

- (void)willActivate {
    [super willActivate];
    
    // ขอ authorization และเริ่ม workout
    [[WorkoutManager shared] requestAuthorization:^(BOOL success) {
        if (success) {
            [[WorkoutManager shared] startWorkout:HKWorkoutActivityTypeRunning];
            [self.durationTimer start];
        }
    }];
}

- (void)setupWorkoutHandlers {
    WorkoutManager *manager = [WorkoutManager shared];
    
    manager.heartRateUpdateHandler = ^(double heartRate) {
        dispatch_async(dispatch_get_main_queue(), ^{
            [self.heartRateLabel setText:[NSString stringWithFormat:@"❤️ %.0f bpm", heartRate]];
            
            // เปลี่ยนสีตาม heart rate zone
            UIColor *color = [self colorForHeartRate:heartRate];
            [self.heartRateLabel setTextColor:color];
        });
    };
    
    manager.caloriesUpdateHandler = ^(double calories) {
        dispatch_async(dispatch_get_main_queue(), ^{
            [self.caloriesLabel setText:[NSString stringWithFormat:@"🔥 %.0f kcal", calories]];
        });
    };
}

- (UIColor *)colorForHeartRate:(double)heartRate {
    if (heartRate < 100) return [UIColor greenColor];
    if (heartRate < 140) return [UIColor yellowColor];
    if (heartRate < 170) return [UIColor orangeColor];
    return [UIColor redColor];
}

- (IBAction)pauseButtonTapped {
    if (self.isPaused) {
        [[WorkoutManager shared] resumeWorkout];
        [self.durationTimer start];
        [self.pauseButton setTitle:@"หยุด"];
        self.isPaused = NO;
    } else {
        [[WorkoutManager shared] pauseWorkout];
        [self.durationTimer stop];
        [self.pauseButton setTitle:@"ดำเนินต่อ"];
        self.isPaused = YES;
    }
}

- (IBAction)endWorkoutTapped {
    [[WorkoutManager shared] endWorkout];
    [self.durationTimer stop];
    
    // แสดง summary
    [self pushControllerWithName:@"WorkoutSummaryController" 
                         context:@{@"completed": @YES}];
}

@end
```

---

## 10. แบบฝึกหัด (Practice Exercises)

### Exercise 1: Watch Step Counter

```objc
// StepCounterInterfaceController.h
@interface StepCounterInterfaceController : WKInterfaceController

@end

// StepCounterInterfaceController.m
@interface StepCounterInterfaceController () <WCSessionDelegate>

@property (weak, nonatomic) IBOutlet WKInterfaceLabel *stepsLabel;
@property (weak, nonatomic) IBOutlet WKInterfaceLabel *goalLabel;
@property (weak, nonatomic) IBOutlet WKInterfaceGroup *progressGroup;
@property (weak, nonatomic) IBOutlet WKInterfacePicker *goalPicker;

@property (assign, nonatomic) NSInteger currentSteps;
@property (assign, nonatomic) NSInteger dailyGoal;

@end

@implementation StepCounterInterfaceController

- (void)awakeWithContext:(id)context {
    [super awakeWithContext:context];
    
    self.dailyGoal = 10000;
    self.currentSteps = [[NSUserDefaults standardUserDefaults] integerForKey:@"steps"];
    
    [self setupGoalPicker];
    [self updateDisplay];
    
    if ([WCSession isSupported]) {
        [WCSession defaultSession].delegate = self;
        [[WCSession defaultSession] activateSession];
    }
}

- (void)willActivate {
    [super willActivate];
    [self fetchStepsFromHealthKit];
}

- (void)setupGoalPicker {
    NSArray *goals = @[@5000, @7500, @10000, @12500, @15000];
    NSMutableArray *items = [NSMutableArray array];
    
    for (NSNumber *goal in goals) {
        WKPickerItem *item = [[WKPickerItem alloc] init];
        item.title = [NSString stringWithFormat:@"%@ ก้าว", goal];
        [items addObject:item];
    }
    
    [self.goalPicker setItems:items];
    [self.goalPicker setSelectedItemIndex:2]; // 10000
}

- (void)fetchStepsFromHealthKit {
    HKHealthStore *store = [[HKHealthStore alloc] init];
    HKQuantityType *stepsType = [HKObjectType quantityTypeForIdentifier:HKQuantityTypeIdentifierStepCount];
    
    NSDate *startOfDay = [[NSCalendar currentCalendar] startOfDayForDate:[NSDate date]];
    NSPredicate *predicate = [HKQuery predicateForSamplesWithStartDate:startOfDay
                                                               endDate:[NSDate date]
                                                               options:HKQueryOptionStrictEndDate];
    
    HKStatisticsQuery *query = [[HKStatisticsQuery alloc] 
        initWithQuantityType:stepsType
        quantitySamplePredicate:predicate
        options:HKStatisticsOptionCumulativeSum
        completionHandler:^(HKStatisticsQuery *query, HKStatistics *result, NSError *error) {
        
        if (result && !error) {
            double steps = [[result sumQuantity] doubleValueForUnit:[HKUnit countUnit]];
            
            dispatch_async(dispatch_get_main_queue(), ^{
                self.currentSteps = (NSInteger)steps;
                [[NSUserDefaults standardUserDefaults] setInteger:self.currentSteps forKey:@"steps"];
                [self updateDisplay];
            });
        }
    }];
    
    [store executeQuery:query];
}

- (void)updateDisplay {
    [self.stepsLabel setText:[NSString stringWithFormat:@"%ld", (long)self.currentSteps]];
    [self.goalLabel setText:[NSString stringWithFormat:@"เป้า: %ld", (long)self.dailyGoal]];
    
    float progress = MIN(1.0, (float)self.currentSteps / self.dailyGoal);
    [self.progressGroup setWidth:140 * progress];
    
    // เปลี่ยนสีตาม progress
    if (progress >= 1.0) {
        [self.progressGroup setBackgroundColor:[UIColor systemGreenColor]];
    } else if (progress >= 0.7) {
        [self.progressGroup setBackgroundColor:[UIColor systemYellowColor]];
    } else {
        [self.progressGroup setBackgroundColor:[UIColor systemBlueColor]];
    }
}

#pragma mark - WCSessionDelegate

- (void)session:(WCSession *)session 
    activationDidCompleteWithState:(WCSessionActivationState)activationState 
                             error:(NSError *)error {}

- (void)session:(WCSession *)session 
    didReceiveApplicationContext:(NSDictionary<NSString *,id> *)applicationContext {
    
    dispatch_async(dispatch_get_main_queue(), ^{
        if (applicationContext[@"steps"]) {
            self.currentSteps = [applicationContext[@"steps"] integerValue];
            [self updateDisplay];
        }
    });
}

@end
```

### Exercise 2: Watch Timer App

```objc
// TimerInterfaceController.m
@interface TimerInterfaceController ()

@property (weak, nonatomic) IBOutlet WKInterfaceTimer *timerDisplay;
@property (weak, nonatomic) IBOutlet WKInterfaceLabel *statusLabel;
@property (weak, nonatomic) IBOutlet WKInterfaceButton *startButton;

@property (strong, nonatomic) NSDate *endDate;
@property (strong, nonatomic) NSTimer *checkTimer;
@property (assign, nonatomic) BOOL isRunning;

@end

@implementation TimerInterfaceController

- (IBAction)startStopTapped {
    if (self.isRunning) {
        [self stopTimer];
    } else {
        [self startTimer:60]; // 1 นาที
    }
}

- (void)startTimer:(NSTimeInterval)duration {
    self.endDate = [NSDate dateWithTimeIntervalSinceNow:duration];
    self.isRunning = YES;
    
    [self.timerDisplay setDate:self.endDate];
    [self.timerDisplay start];
    
    [self.startButton setTitle:@"หยุด"];
    [self.statusLabel setText:@"กำลังนับถอยหลัง"];
    
    // ตรวจสอบว่าหมดเวลาหรือยัง
    self.checkTimer = [NSTimer scheduledTimerWithTimeInterval:1.0
                                                     repeats:YES
                                                       block:^(NSTimer *timer) {
        if ([[NSDate date] isGreaterThanOrEqualTo:self.endDate]) {
            [self timerFinished];
        }
    }];
    
    // สั่นเมื่อเริ่ม
    [[WKExtension sharedExtension] scheduleSnapshotRefreshWithPreferredFireDate:self.endDate
                                                                       userInfo:nil
                                                          scheduledCompletion:nil];
}

- (void)stopTimer {
    [self.timerDisplay stop];
    [self.checkTimer invalidate];
    self.checkTimer = nil;
    self.isRunning = NO;
    
    [self.startButton setTitle:@"เริ่ม"];
    [self.statusLabel setText:@"หยุดแล้ว"];
}

- (void)timerFinished {
    [self stopTimer];
    [self.statusLabel setText:@"⏰ หมดเวลา!"];
    
    // สั่น watch
    [WKInterfaceDevice.currentDevice playHaptic:WKHapticTypeNotification];
    
    NSLog(@"Timer finished!");
}

@end
```

---

## สรุป (Summary)

ในบทนี้เราได้เรียนรู้:

1. **watchOS App Structure** - โครงสร้างแอป Watch และ ExtensionDelegate
2. **WKInterfaceController** - Controller หลักสำหรับ Watch UI
3. **Watch UI Controls** - Label, Button, Table, Image, Picker, Slider
4. **WatchConnectivity** - Framework สำหรับสื่อสาร iPhone ↔ Watch
5. **WCSession** - การส่งข้อมูล 4 รูปแบบ
6. **Complications** - Widget บน Watch face พร้อม ClockKit
7. **Background Updates** - การอัพเดทข้อมูลใน background
8. **Workout Apps** - HealthKit integration สำหรับออกกำลังกาย

### Best Practices สำหรับ watchOS

- ออกแบบ UI ให้เรียบง่าย เข้าใจได้ใน 2-3 วินาที
- ใช้ haptic feedback อย่างเหมาะสม
- Cache ข้อมูลไว้เสมอ เพราะ connection อาจไม่สม่ำเสมอ
- อัพเดท complication เมื่อข้อมูลเปลี่ยน
- ประหยัดแบตเตอรี่ด้วยการลด background refresh ที่ไม่จำเป็น
- ทดสอบบน device จริง เพราะ simulator มีข้อจำกัด

---

*ต่อไป: Part 84 - tvOS Development (การพัฒนาแอปสำหรับ Apple TV)*
