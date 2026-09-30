# ตอนที่ 36: NSNotificationCenter

## บทนำ

`NSNotificationCenter` เป็นกลไกการสื่อสารแบบ broadcast ใน Objective-C ที่ช่วยให้ objects ในแอปพลิเคชันสามารถสื่อสารกันได้โดยไม่ต้องรู้จักกันโดยตรง มันเป็นการ implement Observer Pattern แบบ loose coupling ซึ่งเป็นหนึ่งในรูปแบบที่ใช้กันมากที่สุดใน iOS และ macOS development

## Notifications คืออะไร?

คิดว่า NSNotificationCenter เป็นเหมือน "กระดานข่าว" (bulletin board) ที่:
- Object หนึ่งโพสต์ข่าว (post notification)
- Objects อื่นๆ ที่สนใจรับข่าว (observe notification)
- ทั้งสองฝ่ายไม่จำเป็นต้องรู้จักกันโดยตรง

```
┌─────────────────────────────────────────────────────────────────┐
│                    NSNotificationCenter                          │
│                                                                  │
│  Poster ──────► postNotification ──────► Observer 1             │
│                      (broadcast)  ──────► Observer 2             │
│                                   ──────► Observer 3             │
└─────────────────────────────────────────────────────────────────┘
```

---

## 36.1 postNotificationName:object:

### การส่ง Notification พื้นฐาน

```objc
#import <Foundation/Foundation.h>

// ส่ง notification แบบง่ายที่สุด
[[NSNotificationCenter defaultCenter] postNotificationName:@"UserDidLogin" 
                                                    object:nil];

// ส่งพร้อม sender object
[[NSNotificationCenter defaultCenter] postNotificationName:@"DataDidUpdate" 
                                                    object:self]; // self = sender

// ตัวอย่างจริง: Login Manager
@interface LoginManager : NSObject

- (void)loginWithUsername:(NSString *)username password:(NSString *)password;
- (void)logout;

@end

@implementation LoginManager

- (void)loginWithUsername:(NSString *)username password:(NSString *)password {
    // ทำ login logic...
    NSLog(@"กำลัง login: %@", username);
    
    // จำลองการ login สำเร็จ
    BOOL loginSuccess = YES;
    
    if (loginSuccess) {
        // แจ้งทั่วทั้ง app ว่า user login แล้ว
        [[NSNotificationCenter defaultCenter] 
         postNotificationName:@"UserDidLoginNotification" 
                       object:self]; // self = LoginManager
        
        NSLog(@"Login สำเร็จ");
    } else {
        [[NSNotificationCenter defaultCenter] 
         postNotificationName:@"UserLoginFailedNotification" 
                       object:self];
        
        NSLog(@"Login ล้มเหลว");
    }
}

- (void)logout {
    // ทำ logout logic...
    
    // แจ้งทั่วทั้ง app ว่า user logout แล้ว
    [[NSNotificationCenter defaultCenter] 
     postNotificationName:@"UserDidLogoutNotification" 
                   object:self];
    
    NSLog(@"Logout สำเร็จ");
}

@end
```

---

## 36.2 postNotificationName:object:userInfo:

`userInfo` dictionary ช่วยให้ส่งข้อมูลเพิ่มเติมพร้อม notification ได้

### การส่ง Notification พร้อม userInfo

```objc
// ส่งพร้อมข้อมูล
NSDictionary *userInfo = @{
    @"username": @"alice123",
    @"loginTime": [NSDate date],
    @"deviceType": @"iPhone"
};

[[NSNotificationCenter defaultCenter] 
 postNotificationName:@"UserDidLoginNotification" 
               object:self 
             userInfo:userInfo];
```

### ตัวอย่าง: Data Loading Service

```objc
@interface DataService : NSObject

@property (nonatomic, strong) NSArray *cachedData;

- (void)fetchData;
- (void)updateData:(NSArray *)newData;

@end

@implementation DataService

// ชื่อ notification constants
NSString * const DataServiceWillFetchNotification = @"DataServiceWillFetchNotification";
NSString * const DataServiceDidFetchNotification = @"DataServiceDidFetchNotification";
NSString * const DataServiceFetchFailedNotification = @"DataServiceFetchFailedNotification";
NSString * const DataServiceDataUpdatedNotification = @"DataServiceDataUpdatedNotification";

// userInfo keys
NSString * const DataServiceItemsKey = @"items";
NSString * const DataServiceCountKey = @"count";
NSString * const DataServiceErrorKey = @"error";
NSString * const DataServicePreviousCountKey = @"previousCount";

- (void)fetchData {
    // แจ้งว่ากำลังจะ fetch
    [[NSNotificationCenter defaultCenter] 
     postNotificationName:DataServiceWillFetchNotification 
                   object:self 
                 userInfo:nil];
    
    // จำลองการดึงข้อมูล
    dispatch_async(dispatch_get_global_queue(QOS_CLASS_BACKGROUND, 0), ^{
        [NSThread sleepForTimeInterval:1.0]; // จำลอง network delay
        
        // จำลองข้อมูลที่ได้มา
        NSArray *fetchedData = @[@"Item 1", @"Item 2", @"Item 3", @"Item 4"];
        NSError *fetchError = nil; // ไม่มี error
        
        dispatch_async(dispatch_get_main_queue(), ^{
            if (fetchError) {
                NSDictionary *errorInfo = @{
                    DataServiceErrorKey: fetchError
                };
                [[NSNotificationCenter defaultCenter] 
                 postNotificationName:DataServiceFetchFailedNotification 
                               object:self 
                             userInfo:errorInfo];
            } else {
                self.cachedData = fetchedData;
                NSDictionary *successInfo = @{
                    DataServiceItemsKey: fetchedData,
                    DataServiceCountKey: @(fetchedData.count)
                };
                [[NSNotificationCenter defaultCenter] 
                 postNotificationName:DataServiceDidFetchNotification 
                               object:self 
                             userInfo:successInfo];
            }
        });
    });
}

- (void)updateData:(NSArray *)newData {
    NSUInteger previousCount = self.cachedData.count;
    self.cachedData = newData;
    
    NSDictionary *updateInfo = @{
        DataServiceItemsKey: newData,
        DataServiceCountKey: @(newData.count),
        DataServicePreviousCountKey: @(previousCount)
    };
    
    [[NSNotificationCenter defaultCenter] 
     postNotificationName:DataServiceDataUpdatedNotification 
                   object:self 
                 userInfo:updateInfo];
}

@end
```

---

## 36.3 addObserver:selector:name:object:

### การลงทะเบียนรับ Notification

```objc
// addObserver:selector:name:object: พารามิเตอร์:
// observer - object ที่จะรับ notification
// selector - method ที่จะถูกเรียก
// name - ชื่อ notification (nil = รับทุก notification)
// object - รับเฉพาะจาก object นี้ (nil = รับจากทุก objects)

[[NSNotificationCenter defaultCenter] 
 addObserver:self 
    selector:@selector(handleUserLogin:) 
        name:@"UserDidLoginNotification" 
      object:nil]; // nil = รับจาก sender ใดก็ได้
```

### ตัวอย่าง: Dashboard ที่ตอบสนองต่อ Data Service

```objc
@interface Dashboard : NSObject

- (void)setupNotifications;
- (void)removeNotifications;

@end

@implementation Dashboard

- (void)setupNotifications {
    NSNotificationCenter *nc = [NSNotificationCenter defaultCenter];
    
    // ลงทะเบียนรับ notifications ต่างๆ
    [nc addObserver:self
           selector:@selector(dataWillFetch:)
               name:DataServiceWillFetchNotification
             object:nil];
    
    [nc addObserver:self
           selector:@selector(dataDidFetch:)
               name:DataServiceDidFetchNotification
             object:nil];
    
    [nc addObserver:self
           selector:@selector(fetchFailed:)
               name:DataServiceFetchFailedNotification
             object:nil];
    
    [nc addObserver:self
           selector:@selector(dataDidUpdate:)
               name:DataServiceDataUpdatedNotification
             object:nil];
    
    NSLog(@"Dashboard พร้อมรับ notifications");
}

- (void)removeNotifications {
    [[NSNotificationCenter defaultCenter] removeObserver:self];
}

- (void)dealloc {
    [self removeNotifications];
}

#pragma mark - Notification Handlers

- (void)dataWillFetch:(NSNotification *)notification {
    NSLog(@"📡 กำลังโหลดข้อมูล...");
    // แสดง loading indicator
}

- (void)dataDidFetch:(NSNotification *)notification {
    NSArray *items = notification.userInfo[DataServiceItemsKey];
    NSNumber *count = notification.userInfo[DataServiceCountKey];
    
    NSLog(@"✅ โหลดข้อมูลสำเร็จ: %@ รายการ", count);
    NSLog(@"   รายการ: %@", items);
    
    // อัปเดต UI
    // [self.tableView reloadData];
    // [self hideLoadingIndicator];
}

- (void)fetchFailed:(NSNotification *)notification {
    NSError *error = notification.userInfo[DataServiceErrorKey];
    NSLog(@"❌ โหลดข้อมูลล้มเหลว: %@", error.localizedDescription);
    
    // แสดง error message
}

- (void)dataDidUpdate:(NSNotification *)notification {
    NSArray *items = notification.userInfo[DataServiceItemsKey];
    NSNumber *newCount = notification.userInfo[DataServiceCountKey];
    NSNumber *prevCount = notification.userInfo[DataServicePreviousCountKey];
    
    NSLog(@"🔄 ข้อมูลอัปเดต: %@ -> %@ รายการ", prevCount, newCount);
}

@end
```

### การ Filter โดย Sender Object

```objc
DataService *mainService = [[DataService alloc] init];
DataService *cacheService = [[DataService alloc] init];

// รับเฉพาะจาก mainService
[[NSNotificationCenter defaultCenter] 
 addObserver:self
    selector:@selector(mainDataFetched:)
        name:DataServiceDidFetchNotification
      object:mainService]; // จำกัดเฉพาะ mainService

// รับจากทุก DataService
[[NSNotificationCenter defaultCenter] 
 addObserver:self
    selector:@selector(anyDataFetched:)
        name:DataServiceDidFetchNotification
      object:nil]; // nil = รับจากทุก sender
```

---

## 36.4 removeObserver:

### การลบ Observer

```objc
// วิธีที่ 1: ลบทุก notifications ของ observer นี้
[[NSNotificationCenter defaultCenter] removeObserver:self];

// วิธีที่ 2: ลบเฉพาะ notification ที่ระบุ
[[NSNotificationCenter defaultCenter] 
 removeObserver:self 
            name:@"UserDidLoginNotification" 
          object:nil];

// วิธีที่ 3: ลบเฉพาะจาก sender ที่ระบุ
[[NSNotificationCenter defaultCenter] 
 removeObserver:self 
            name:DataServiceDidFetchNotification 
          object:specificService]; // ลบเฉพาะจาก specificService
```

### การลบใน dealloc

```objc
@implementation ViewController

- (void)viewDidLoad {
    [super viewDidLoad];
    [[NSNotificationCenter defaultCenter] addObserver:self
                                             selector:@selector(handleNotification:)
                                                 name:@"SomeNotification"
                                               object:nil];
}

- (void)dealloc {
    // ลบทั้งหมดก่อน dealloc เสมอ
    [[NSNotificationCenter defaultCenter] removeObserver:self];
}

- (void)handleNotification:(NSNotification *)notification {
    // handle notification
}

@end
```

---

## 36.5 NSNotification Object

`NSNotification` object ที่ส่งมาใน handler มีข้อมูลต่อไปนี้:

```objc
- (void)handleNotification:(NSNotification *)notification {
    // ชื่อ notification
    NSString *name = notification.name;
    NSLog(@"Notification: %@", name);
    
    // object ที่ส่ง notification (อาจเป็น nil)
    id sender = notification.object;
    NSLog(@"Sender: %@", sender);
    
    // ข้อมูลเพิ่มเติม
    NSDictionary *userInfo = notification.userInfo;
    NSLog(@"UserInfo: %@", userInfo);
    
    // ใช้งาน
    if ([name isEqualToString:@"DataDidUpdate"]) {
        NSArray *newItems = userInfo[@"items"];
        DataService *service = (DataService *)sender;
        NSLog(@"Service %@ อัปเดต %lu รายการ", service, (unsigned long)newItems.count);
    }
}
```

### สร้าง NSNotification Object โดยตรง

```objc
// สร้าง notification object ก่อนแล้วค่อยส่ง
NSDictionary *info = @{@"key": @"value"};
NSNotification *notification = [NSNotification notificationWithName:@"MyNotification"
                                                             object:self
                                                           userInfo:info];

[[NSNotificationCenter defaultCenter] postNotification:notification];
```

---

## 36.6 Custom Notification Names (NSNotificationName)

### การใช้ Constants

การใช้ string literals โดยตรงเป็นความเสี่ยง เพราะอาจพิมพ์ผิดได้ ควรใช้ constants แทน:

```objc
// วิธีที่ 1: ประกาศใน header (.h)

// UserNotifications.h
extern NSNotificationName const UserDidLoginNotification;
extern NSNotificationName const UserDidLogoutNotification;
extern NSNotificationName const UserProfileDidUpdateNotification;
extern NSNotificationName const UserPasswordDidChangeNotification;

// userInfo keys
extern NSString * const UserNotificationUsernameKey;
extern NSString * const UserNotificationEmailKey;
extern NSString * const UserNotificationTimestampKey;
```

```objc
// UserNotifications.m
NSNotificationName const UserDidLoginNotification = @"UserDidLoginNotification";
NSNotificationName const UserDidLogoutNotification = @"UserDidLogoutNotification";
NSNotificationName const UserProfileDidUpdateNotification = @"UserProfileDidUpdateNotification";
NSNotificationName const UserPasswordDidChangeNotification = @"UserPasswordDidChangeNotification";

NSString * const UserNotificationUsernameKey = @"username";
NSString * const UserNotificationEmailKey = @"email";
NSString * const UserNotificationTimestampKey = @"timestamp";
```

```objc
// การใช้งาน - ไม่มีโอกาสพิมพ์ผิด
[[NSNotificationCenter defaultCenter] 
 postNotificationName:UserDidLoginNotification 
               object:self 
             userInfo:@{UserNotificationUsernameKey: @"alice"}];

[[NSNotificationCenter defaultCenter] 
 addObserver:self 
    selector:@selector(handleLogin:) 
        name:UserDidLoginNotification 
      object:nil];
```

### วิธีที่ 2: ใช้ Static Constants ใน Implementation File

```objc
// ถ้าไม่ต้องการ expose ออกไปนอก class
@implementation MyClass

static NSNotificationName const InternalDataLoadedNotification = @"InternalDataLoadedNotification";

- (void)loadData {
    // ...
    [[NSNotificationCenter defaultCenter] 
     postNotificationName:InternalDataLoadedNotification 
                   object:self];
}

- (void)setupObservers {
    [[NSNotificationCenter defaultCenter] 
     addObserver:self 
        selector:@selector(dataLoaded:) 
            name:InternalDataLoadedNotification 
          object:nil];
}

@end
```

---

## 36.7 Notification userInfo

### Best Practices สำหรับ userInfo

```objc
// ✅ ดี: ใช้ constants เป็น keys
NSString * const OrderNotificationOrderIDKey = @"orderID";
NSString * const OrderNotificationItemsKey = @"items";
NSString * const OrderNotificationTotalKey = @"total";
NSString * const OrderNotificationStatusKey = @"status";

// ส่ง notification
NSDictionary *orderInfo = @{
    OrderNotificationOrderIDKey: @"ORD-12345",
    OrderNotificationItemsKey: @[@"iPhone", @"AirPods"],
    OrderNotificationTotalKey: @(49900.0),
    OrderNotificationStatusKey: @"confirmed"
};

[[NSNotificationCenter defaultCenter] 
 postNotificationName:@"OrderConfirmedNotification" 
               object:self 
             userInfo:orderInfo];

// รับ notification
- (void)orderConfirmed:(NSNotification *)notification {
    NSString *orderID = notification.userInfo[OrderNotificationOrderIDKey];
    NSArray *items = notification.userInfo[OrderNotificationItemsKey];
    NSNumber *total = notification.userInfo[OrderNotificationTotalKey];
    NSString *status = notification.userInfo[OrderNotificationStatusKey];
    
    NSLog(@"คำสั่งซื้อ %@ ยืนยันแล้ว", orderID);
    NSLog(@"รายการ: %@", items);
    NSLog(@"ยอดรวม: %.2f บาท", [total floatValue]);
    NSLog(@"สถานะ: %@", status);
}
```

### userInfo พร้อม Type-Safe Extraction

```objc
@interface NotificationHelper : NSObject

// Helper methods สำหรับดึงข้อมูลจาก notification อย่างปลอดภัย
+ (NSString *)stringForKey:(NSString *)key 
                inNotification:(NSNotification *)notification;
+ (NSInteger)integerForKey:(NSString *)key 
                inNotification:(NSNotification *)notification;
+ (CGFloat)floatForKey:(NSString *)key 
             inNotification:(NSNotification *)notification;

@end

@implementation NotificationHelper

+ (NSString *)stringForKey:(NSString *)key 
                inNotification:(NSNotification *)notification {
    id value = notification.userInfo[key];
    if ([value isKindOfClass:[NSString class]]) {
        return value;
    }
    return nil;
}

+ (NSInteger)integerForKey:(NSString *)key 
                inNotification:(NSNotification *)notification {
    id value = notification.userInfo[key];
    if ([value isKindOfClass:[NSNumber class]]) {
        return [(NSNumber *)value integerValue];
    }
    return 0;
}

+ (CGFloat)floatForKey:(NSString *)key 
             inNotification:(NSNotification *)notification {
    id value = notification.userInfo[key];
    if ([value isKindOfClass:[NSNumber class]]) {
        return [(NSNumber *)value floatValue];
    }
    return 0.0;
}

@end
```

---

## 36.8 addObserverForName:object:queue:usingBlock:

วิธีนี้ใช้ block แทน selector ทำให้โค้ดกระชับกว่า

```objc
// สังเกต: คืนค่า id<NSObject> ที่ต้องเก็บไว้เพื่อ remove ทีหลัง
id<NSObject> observer = [[NSNotificationCenter defaultCenter] 
    addObserverForName:@"UserDidLoginNotification"
                object:nil
                 queue:[NSOperationQueue mainQueue] // เรียก block ใน main queue
            usingBlock:^(NSNotification *notification) {
    NSString *username = notification.userInfo[@"username"];
    NSLog(@"User login: %@", username);
    // อัปเดต UI
}];
```

### การ Remove Block-Based Observer

```objc
// เก็บ token ไว้ใน property
@interface MyViewController : NSObject
@property (nonatomic, strong) id<NSObject> loginObserverToken;
@property (nonatomic, strong) id<NSObject> logoutObserverToken;
@end

@implementation MyViewController

- (void)setupObservers {
    // Login observer
    self.loginObserverToken = [[NSNotificationCenter defaultCenter]
        addObserverForName:UserDidLoginNotification
                    object:nil
                     queue:[NSOperationQueue mainQueue]
                usingBlock:^(NSNotification *notification) {
        NSLog(@"User logged in");
    }];
    
    // Logout observer
    self.logoutObserverToken = [[NSNotificationCenter defaultCenter]
        addObserverForName:UserDidLogoutNotification
                    object:nil
                     queue:[NSOperationQueue mainQueue]
                usingBlock:^(NSNotification *note) {
        NSLog(@"User logged out");
    }];
}

- (void)removeObservers {
    [[NSNotificationCenter defaultCenter] removeObserver:self.loginObserverToken];
    [[NSNotificationCenter defaultCenter] removeObserver:self.logoutObserverToken];
    self.loginObserverToken = nil;
    self.logoutObserverToken = nil;
}

- (void)dealloc {
    [self removeObservers];
}

@end
```

### ประโยชน์ของ Block-Based Observer

```objc
// สามารถกำหนด queue ที่จะเรียก block ได้
// [NSOperationQueue mainQueue] - เรียกใน main thread (สำหรับ UI)
// [NSOperationQueue currentQueue] - เรียกใน thread ที่กำลังทำงาน
// [[NSOperationQueue alloc] init] - เรียกใน background thread

// ตัวอย่าง: process ใน background แต่อัปเดต UI ใน main thread
id<NSObject> token = [[NSNotificationCenter defaultCenter]
    addObserverForName:@"LargeDataReceived"
                object:nil
                 queue:[[NSOperationQueue alloc] init] // background queue
            usingBlock:^(NSNotification *notification) {
    // ประมวลผลข้อมูลใน background
    NSArray *data = notification.userInfo[@"data"];
    NSArray *processed = [self processData:data]; // งานหนัก
    
    // อัปเดต UI ใน main thread
    [[NSOperationQueue mainQueue] addOperationWithBlock:^{
        [self updateUIWithData:processed];
    }];
}];
```

---

## 36.9 NSNotificationCenter vs Delegates vs KVO

### เปรียบเทียบ 3 Patterns

```
┌────────────────────────────────────────────────────────────────────────┐
│           NSNotificationCenter vs Delegates vs KVO                      │
├──────────────────┬──────────────────┬────────────────┬──────────────── │
│ Feature          │ Notification     │ Delegate       │ KVO              │
├──────────────────┼──────────────────┼────────────────┼──────────────── │
│ Coupling         │ Loose (ไม่รู้จัก)│ Tight (รู้จัก)│ Medium          │
│ Receivers        │ 1 to Many        │ 1 to 1         │ 1 to Many       │
│ Data passing     │ userInfo dict    │ Parameters     │ change dict      │
│ Type safety      │ ❌ (dictionary)  │ ✅ (protocol)  │ ❌ (dictionary)  │
│ Property track   │ ❌               │ ❌             │ ✅               │
│ Thread safety    │ ⚠️               │ ⚠️             │ ⚠️              │
│ Setup complexity │ Simple           │ Medium         │ Medium           │
│ Performance      │ Good             │ Best           │ Good             │
└──────────────────┴──────────────────┴────────────────┴──────────────────┘
```

### เมื่อไหรควรใช้อะไร?

```objc
// ใช้ NSNotificationCenter เมื่อ:
// 1. หลาย objects ต้องการรับเหตุการณ์เดียวกัน
// 2. ต้องการ loose coupling (ไม่รู้จักกัน)
// 3. App-wide events (login, logout, app lifecycle)
// 4. Cross-module communication

// ตัวอย่าง: User logout - ทุกส่วนของแอปต้องอัปเดต
[[NSNotificationCenter defaultCenter] 
 postNotificationName:@"UserLogoutNotification" 
               object:nil];
// -> NavigationController รับ (reset stack)
// -> DatabaseManager รับ (clear cached data)
// -> NetworkManager รับ (cancel pending requests)
// -> AnalyticsManager รับ (log event)

// ใช้ Delegate เมื่อ:
// 1. 1-to-1 relationship
// 2. ต้องการ return value จาก delegate
// 3. ต้องการ type safety (protocol)
// 4. เช่น UITableViewDelegate, UIScrollViewDelegate

// ใช้ KVO เมื่อ:
// 1. ต้องการ observe property ของ object
// 2. ต้องการค่าเก่าและใหม่
// 3. เช่น observe model.isLoading เพื่ออัปเดต spinner
```

---

## 36.10 Thread Safety กับ Notifications

### ปัญหา Thread Safety

```objc
// ❌ ปัญหา: post ใน background, observe ใน main thread
// NSNotificationCenter ส่ง notification ใน thread ที่ post
// ถ้าต้องอัปเดต UI จะต้อง dispatch ไป main thread เอง

// Background thread:
dispatch_async(dispatch_get_global_queue(QOS_CLASS_BACKGROUND, 0), ^{
    NSArray *data = [self fetchLargeDataset];
    
    // notification ถูกส่งใน background thread!
    [[NSNotificationCenter defaultCenter] 
     postNotificationName:@"DataFetched" 
                   object:nil 
                 userInfo:@{@"data": data}];
});

// Main thread observer - อาจ crash ถ้าอัปเดต UI โดยตรง
- (void)dataFetched:(NSNotification *)notification {
    // ❌ นี่อาจทำงานใน background thread!
    // self.tableView.reloadData(); // CRASH!
    
    // ✅ ต้อง dispatch ไป main thread
    dispatch_async(dispatch_get_main_queue(), ^{
        NSArray *data = notification.userInfo[@"data"];
        NSLog(@"อัปเดต UI ด้วย %lu รายการ", (unsigned long)data.count);
        // self.tableView.reloadData(); // OK
    });
}
```

### วิธีที่ 1: Post ใน Main Thread เสมอ

```objc
// บังคับ post ใน main thread
void postNotificationOnMainThread(NSString *name, id object, NSDictionary *userInfo) {
    if ([NSThread isMainThread]) {
        [[NSNotificationCenter defaultCenter] 
         postNotificationName:name 
                       object:object 
                     userInfo:userInfo];
    } else {
        dispatch_async(dispatch_get_main_queue(), ^{
            [[NSNotificationCenter defaultCenter] 
             postNotificationName:name 
                           object:object 
                         userInfo:userInfo];
        });
    }
}
```

### วิธีที่ 2: ใช้ Block-Based Observer พร้อม Queue

```objc
// กำหนด queue ใน addObserverForName:
id<NSObject> token = [[NSNotificationCenter defaultCenter]
    addObserverForName:@"DataFetched"
                object:nil
                 queue:[NSOperationQueue mainQueue] // บังคับ main thread
            usingBlock:^(NSNotification *notification) {
    // block นี้จะทำงานใน main thread เสมอ
    NSArray *data = notification.userInfo[@"data"];
    NSLog(@"อัปเดต UI: %@", [NSThread isMainThread] ? @"main" : @"background");
    // self.tableView.reloadData(); // safe!
}];
```

---

## 36.11 Strong/Weak Observers

### ปัญหา: Observer ที่ถูก Dealloc

```objc
// NSNotificationCenter ไม่ retain observer
// ถ้า observer ถูก dealloc แต่ยังไม่ได้ remove -> crash!

@implementation SomeClass

- (void)startObserving {
    [[NSNotificationCenter defaultCenter] 
     addObserver:self // NSNotificationCenter ไม่ retain self
        selector:@selector(handleNotification:)
            name:@"SomeNotification"
          object:nil];
}

// ถ้า SomeClass ถูก dealloc และ notification ถูกส่งมา -> crash
// เพราะ NSNotificationCenter ยังมี pointer ไป self ที่ถูก dealloc แล้ว

- (void)dealloc {
    // ต้อง remove ก่อน dealloc เสมอ!
    [[NSNotificationCenter defaultCenter] removeObserver:self];
}

@end
```

### Block Observer กับ Retain Cycle

```objc
// ❌ Retain cycle: self retain token, token retain self
@interface MyClass : NSObject
@property (nonatomic, strong) id<NSObject> observerToken;
@end

- (void)badSetup {
    self.observerToken = [[NSNotificationCenter defaultCenter]
        addObserverForName:@"SomeNotification"
                    object:nil
                     queue:nil
                usingBlock:^(NSNotification *note) {
        // ❌ Retain cycle!
        [self handleNotification:note]; // self retain cycle ผ่าน token
    }];
}

// ✅ ใช้ weakSelf
- (void)goodSetup {
    __weak typeof(self) weakSelf = self;
    
    self.observerToken = [[NSNotificationCenter defaultCenter]
        addObserverForName:@"SomeNotification"
                    object:nil
                     queue:nil
                usingBlock:^(NSNotification *note) {
        __strong typeof(weakSelf) strongSelf = weakSelf;
        if (strongSelf) {
            [strongSelf handleNotification:note];
        }
    }];
}
```

---

## 36.12 ตัวอย่างจริง: App Event System

```objc
// AppNotifications.h
// Notification names
extern NSNotificationName const AppUserDidAuthenticateNotification;
extern NSNotificationName const AppUserDidSignOutNotification;
extern NSNotificationName const AppNetworkStatusChangedNotification;
extern NSNotificationName const AppDataSyncCompletedNotification;
extern NSNotificationName const AppThemeChangedNotification;

// userInfo keys
extern NSString * const AppNotifUserIDKey;
extern NSString * const AppNotifUsernameKey;
extern NSString * const AppNotifNetworkStatusKey;
extern NSString * const AppNotifSyncCountKey;
extern NSString * const AppNotifThemeKey;
```

```objc
// AppNotifications.m
NSNotificationName const AppUserDidAuthenticateNotification = @"AppUserDidAuthenticate";
NSNotificationName const AppUserDidSignOutNotification = @"AppUserDidSignOut";
NSNotificationName const AppNetworkStatusChangedNotification = @"AppNetworkStatusChanged";
NSNotificationName const AppDataSyncCompletedNotification = @"AppDataSyncCompleted";
NSNotificationName const AppThemeChangedNotification = @"AppThemeChanged";

NSString * const AppNotifUserIDKey = @"userID";
NSString * const AppNotifUsernameKey = @"username";
NSString * const AppNotifNetworkStatusKey = @"networkStatus";
NSString * const AppNotifSyncCountKey = @"syncCount";
NSString * const AppNotifThemeKey = @"theme";
```

```objc
// AuthenticationService.m
@implementation AuthenticationService

- (void)loginWithCredentials:(NSDictionary *)credentials {
    // ทำ authentication...
    BOOL success = YES; // จำลอง
    
    if (success) {
        [[NSNotificationCenter defaultCenter]
         postNotificationName:AppUserDidAuthenticateNotification
                       object:self
                     userInfo:@{
                         AppNotifUserIDKey: @"user123",
                         AppNotifUsernameKey: credentials[@"username"] ?: @"Unknown"
                     }];
    }
}

- (void)signOut {
    [[NSNotificationCenter defaultCenter]
     postNotificationName:AppUserDidSignOutNotification
                   object:self
                 userInfo:nil];
}

@end
```

```objc
// NavigationManager.m - จัดการ navigation เมื่อ auth state เปลี่ยน
@implementation NavigationManager

- (void)setupNotifications {
    [[NSNotificationCenter defaultCenter]
     addObserver:self
        selector:@selector(userDidAuthenticate:)
            name:AppUserDidAuthenticateNotification
          object:nil];
    
    [[NSNotificationCenter defaultCenter]
     addObserver:self
        selector:@selector(userDidSignOut:)
            name:AppUserDidSignOutNotification
          object:nil];
}

- (void)userDidAuthenticate:(NSNotification *)notification {
    NSString *username = notification.userInfo[AppNotifUsernameKey];
    NSLog(@"Nav: User '%@' logged in -> navigate to home", username);
    // [self showHomeScreen];
}

- (void)userDidSignOut:(NSNotification *)notification {
    NSLog(@"Nav: User signed out -> navigate to login");
    // [self showLoginScreen];
}

- (void)dealloc {
    [[NSNotificationCenter defaultCenter] removeObserver:self];
}

@end
```

```objc
// DatabaseManager.m - clear cache เมื่อ sign out
@implementation DatabaseManager

- (void)setupNotifications {
    [[NSNotificationCenter defaultCenter]
     addObserver:self
        selector:@selector(userDidSignOut:)
            name:AppUserDidSignOutNotification
          object:nil];
}

- (void)userDidSignOut:(NSNotification *)notification {
    NSLog(@"DB: Clearing user data...");
    [self clearUserCache];
    [self clearUserPreferences];
    NSLog(@"DB: User data cleared");
}

- (void)clearUserCache { /* ... */ }
- (void)clearUserPreferences { /* ... */ }

- (void)dealloc {
    [[NSNotificationCenter defaultCenter] removeObserver:self];
}

@end
```

```objc
// AnalyticsManager.m - log analytics events
@implementation AnalyticsManager

- (void)setupNotifications {
    NSNotificationCenter *nc = [NSNotificationCenter defaultCenter];
    
    __weak typeof(self) weakSelf = self;
    
    [nc addObserverForName:AppUserDidAuthenticateNotification
                   object:nil
                    queue:[NSOperationQueue mainQueue]
               usingBlock:^(NSNotification *note) {
        NSString *userID = note.userInfo[AppNotifUserIDKey];
        [weakSelf logEvent:@"user_login" properties:@{@"user_id": userID ?: @"unknown"}];
    }];
    
    [nc addObserverForName:AppUserDidSignOutNotification
                   object:nil
                    queue:[NSOperationQueue mainQueue]
               usingBlock:^(NSNotification *note) {
        [weakSelf logEvent:@"user_logout" properties:nil];
    }];
    
    [nc addObserverForName:AppDataSyncCompletedNotification
                   object:nil
                    queue:[NSOperationQueue mainQueue]
               usingBlock:^(NSNotification *note) {
        NSNumber *count = note.userInfo[AppNotifSyncCountKey];
        [weakSelf logEvent:@"data_sync" properties:@{@"count": count ?: @0}];
    }];
}

- (void)logEvent:(NSString *)event properties:(NSDictionary *)props {
    NSLog(@"📊 Analytics: %@ | %@", event, props ?: @{});
}

@end
```

---

## 36.13 ตัวอย่างจริง: Theme Manager

```objc
// ThemeManager.h
typedef NS_ENUM(NSInteger, AppTheme) {
    AppThemeLight,
    AppThemeDark,
    AppThemeSystem
};

@interface ThemeManager : NSObject

@property (nonatomic, assign) AppTheme currentTheme;
@property (class, nonatomic, readonly) ThemeManager *sharedManager;

- (void)applyTheme:(AppTheme)theme;

@end

extern NSNotificationName const ThemeManagerThemeChangedNotification;
extern NSString * const ThemeManagerThemeKey;
extern NSString * const ThemeManagerPreviousThemeKey;
```

```objc
// ThemeManager.m
#import "ThemeManager.h"

NSNotificationName const ThemeManagerThemeChangedNotification = @"ThemeManagerThemeChanged";
NSString * const ThemeManagerThemeKey = @"theme";
NSString * const ThemeManagerPreviousThemeKey = @"previousTheme";

@implementation ThemeManager

+ (ThemeManager *)sharedManager {
    static ThemeManager *shared = nil;
    static dispatch_once_t onceToken;
    dispatch_once(&onceToken, ^{
        shared = [[ThemeManager alloc] init];
    });
    return shared;
}

- (instancetype)init {
    self = [super init];
    if (self) {
        _currentTheme = AppThemeLight;
    }
    return self;
}

- (void)applyTheme:(AppTheme)theme {
    if (theme == self.currentTheme) return;
    
    AppTheme previousTheme = self.currentTheme;
    self.currentTheme = theme;
    
    NSString *themeStr = [self stringForTheme:theme];
    NSString *prevStr = [self stringForTheme:previousTheme];
    
    NSLog(@"🎨 Theme เปลี่ยน: %@ -> %@", prevStr, themeStr);
    
    [[NSNotificationCenter defaultCenter]
     postNotificationName:ThemeManagerThemeChangedNotification
                   object:self
                 userInfo:@{
                     ThemeManagerThemeKey: @(theme),
                     ThemeManagerPreviousThemeKey: @(previousTheme)
                 }];
}

- (NSString *)stringForTheme:(AppTheme)theme {
    switch (theme) {
        case AppThemeLight: return @"Light";
        case AppThemeDark: return @"Dark";
        case AppThemeSystem: return @"System";
    }
    return @"Unknown";
}

@end
```

```objc
// UIComponent ที่ตอบสนองต่อ theme
@interface ThemeAwareView : NSObject

@property (nonatomic, strong) NSString *name;

- (instancetype)initWithName:(NSString *)name;

@end

@implementation ThemeAwareView

- (instancetype)initWithName:(NSString *)name {
    self = [super init];
    if (self) {
        _name = name;
        [self setupThemeObserver];
    }
    return self;
}

- (void)setupThemeObserver {
    [[NSNotificationCenter defaultCenter]
     addObserver:self
        selector:@selector(themeChanged:)
            name:ThemeManagerThemeChangedNotification
          object:nil];
    
    // Apply current theme immediately
    [self applyTheme:[ThemeManager sharedManager].currentTheme];
}

- (void)themeChanged:(NSNotification *)notification {
    AppTheme newTheme = [notification.userInfo[ThemeManagerThemeKey] integerValue];
    [self applyTheme:newTheme];
}

- (void)applyTheme:(AppTheme)theme {
    switch (theme) {
        case AppThemeLight:
            NSLog(@"[%@] พื้นหลังสีขาว, ตัวหนังสือสีดำ", self.name);
            break;
        case AppThemeDark:
            NSLog(@"[%@] พื้นหลังสีดำ, ตัวหนังสือสีขาว", self.name);
            break;
        case AppThemeSystem:
            NSLog(@"[%@] ใช้ตาม system setting", self.name);
            break;
    }
}

- (void)dealloc {
    [[NSNotificationCenter defaultCenter] removeObserver:self];
}

@end

// การใช้งาน
ThemeAwareView *header = [[ThemeAwareView alloc] initWithName:@"Header"];
ThemeAwareView *content = [[ThemeAwareView alloc] initWithName:@"Content"];
ThemeAwareView *footer = [[ThemeAwareView alloc] initWithName:@"Footer"];

ThemeManager *themeManager = [ThemeManager sharedManager];
[themeManager applyTheme:AppThemeDark];  // ทุก view จะอัปเดตพร้อมกัน
[themeManager applyTheme:AppThemeLight]; // ทุก view อัปเดตอีกครั้ง
```

---

## 36.14 ตัวอย่างจริง: Shopping Cart System

```objc
// Shopping Cart Notifications
NSNotificationName const CartItemAddedNotification = @"CartItemAdded";
NSNotificationName const CartItemRemovedNotification = @"CartItemRemoved";
NSNotificationName const CartClearedNotification = @"CartCleared";
NSNotificationName const CartCheckoutStartedNotification = @"CartCheckoutStarted";
NSNotificationName const CartCheckoutCompletedNotification = @"CartCheckoutCompleted";

NSString * const CartNotifItemKey = @"item";
NSString * const CartNotifQuantityKey = @"quantity";
NSString * const CartNotifTotalKey = @"total";
NSString * const CartNotifItemCountKey = @"itemCount";

@interface ShoppingCartItem : NSObject
@property (nonatomic, strong) NSString *productID;
@property (nonatomic, strong) NSString *name;
@property (nonatomic, assign) CGFloat price;
@property (nonatomic, assign) NSInteger quantity;
@end

@implementation ShoppingCartItem
@end

@interface ShoppingCart : NSObject

@property (nonatomic, strong) NSMutableArray *items;
@property (nonatomic, readonly) CGFloat total;
@property (nonatomic, readonly) NSInteger itemCount;

- (void)addItem:(ShoppingCartItem *)item;
- (void)removeItem:(ShoppingCartItem *)item;
- (void)clearCart;
- (void)checkout;

@end

@implementation ShoppingCart

- (instancetype)init {
    self = [super init];
    if (self) {
        _items = [NSMutableArray array];
    }
    return self;
}

- (CGFloat)total {
    CGFloat sum = 0;
    for (ShoppingCartItem *item in self.items) {
        sum += item.price * item.quantity;
    }
    return sum;
}

- (NSInteger)itemCount {
    NSInteger count = 0;
    for (ShoppingCartItem *item in self.items) {
        count += item.quantity;
    }
    return count;
}

- (void)addItem:(ShoppingCartItem *)item {
    [self.items addObject:item];
    
    [[NSNotificationCenter defaultCenter]
     postNotificationName:CartItemAddedNotification
                   object:self
                 userInfo:@{
                     CartNotifItemKey: item,
                     CartNotifQuantityKey: @(item.quantity),
                     CartNotifTotalKey: @(self.total),
                     CartNotifItemCountKey: @(self.itemCount)
                 }];
}

- (void)removeItem:(ShoppingCartItem *)item {
    [self.items removeObject:item];
    
    [[NSNotificationCenter defaultCenter]
     postNotificationName:CartItemRemovedNotification
                   object:self
                 userInfo:@{
                     CartNotifItemKey: item,
                     CartNotifTotalKey: @(self.total),
                     CartNotifItemCountKey: @(self.itemCount)
                 }];
}

- (void)clearCart {
    [self.items removeAllObjects];
    
    [[NSNotificationCenter defaultCenter]
     postNotificationName:CartClearedNotification
                   object:self
                 userInfo:nil];
}

- (void)checkout {
    [[NSNotificationCenter defaultCenter]
     postNotificationName:CartCheckoutStartedNotification
                   object:self
                 userInfo:@{
                     CartNotifTotalKey: @(self.total),
                     CartNotifItemCountKey: @(self.itemCount)
                 }];
    
    // จำลองการ checkout
    dispatch_after(dispatch_time(DISPATCH_TIME_NOW, (int64_t)(1.0 * NSEC_PER_SEC)), 
                   dispatch_get_main_queue(), ^{
        [[NSNotificationCenter defaultCenter]
         postNotificationName:CartCheckoutCompletedNotification
                       object:self
                     userInfo:@{
                         CartNotifTotalKey: @(self.total)
                     }];
        [self clearCart];
    });
}

@end

// UI Badge Counter
@interface CartBadgeView : NSObject
@property (nonatomic, assign) NSInteger badgeCount;
@end

@implementation CartBadgeView

- (instancetype)init {
    self = [super init];
    if (self) {
        [self setupObservers];
    }
    return self;
}

- (void)setupObservers {
    NSNotificationCenter *nc = [NSNotificationCenter defaultCenter];
    
    [nc addObserver:self selector:@selector(itemAdded:) 
               name:CartItemAddedNotification object:nil];
    [nc addObserver:self selector:@selector(itemRemoved:) 
               name:CartItemRemovedNotification object:nil];
    [nc addObserver:self selector:@selector(cartCleared:) 
               name:CartClearedNotification object:nil];
    [nc addObserver:self selector:@selector(checkoutCompleted:) 
               name:CartCheckoutCompletedNotification object:nil];
}

- (void)itemAdded:(NSNotification *)notification {
    self.badgeCount = [notification.userInfo[CartNotifItemCountKey] integerValue];
    [self updateBadge];
}

- (void)itemRemoved:(NSNotification *)notification {
    self.badgeCount = [notification.userInfo[CartNotifItemCountKey] integerValue];
    [self updateBadge];
}

- (void)cartCleared:(NSNotification *)notification {
    self.badgeCount = 0;
    [self updateBadge];
}

- (void)checkoutCompleted:(NSNotification *)notification {
    CGFloat total = [notification.userInfo[CartNotifTotalKey] floatValue];
    NSLog(@"🛍️ Checkout เสร็จสิ้น ยอดรวม: %.2f บาท", total);
}

- (void)updateBadge {
    if (self.badgeCount > 0) {
        NSLog(@"🛒 Badge: %ld รายการ", (long)self.badgeCount);
    } else {
        NSLog(@"🛒 Badge: ว่าง");
    }
}

- (void)dealloc {
    [[NSNotificationCenter defaultCenter] removeObserver:self];
}

@end

// การใช้งาน
ShoppingCart *cart = [[ShoppingCart alloc] init];
CartBadgeView *badge = [[CartBadgeView alloc] init];

ShoppingCartItem *item1 = [[ShoppingCartItem alloc] init];
item1.name = @"iPhone 15";
item1.price = 39900;
item1.quantity = 1;

ShoppingCartItem *item2 = [[ShoppingCartItem alloc] init];
item2.name = @"AirPods Pro";
item2.price = 9490;
item2.quantity = 2;

[cart addItem:item1];  // Badge: 1 รายการ
[cart addItem:item2];  // Badge: 3 รายการ
[cart removeItem:item1]; // Badge: 2 รายการ
[cart checkout]; // เริ่ม checkout
```

---

## 36.15 System Notifications

Apple จัดเตรียม system notifications จำนวนมากที่เราสามารถ observe ได้

### UIApplication Notifications (iOS)

```objc
// iOS UIApplication Notifications
#if TARGET_OS_IOS
NSNotificationName notifications[] = {
    UIApplicationDidFinishLaunchingNotification,     // แอปเปิดสำเร็จ
    UIApplicationWillResignActiveNotification,       // กำลังจะ inactive
    UIApplicationDidBecomeActiveNotification,        // กลับมา active
    UIApplicationDidEnterBackgroundNotification,     // เข้า background
    UIApplicationWillEnterForegroundNotification,    // กลับมา foreground
    UIApplicationWillTerminateNotification,          // กำลังจะปิด
    UIApplicationDidReceiveMemoryWarningNotification // หน่วยความจำเหลือน้อย
};
#endif

// ตัวอย่าง: จัดการ App Lifecycle
[[NSNotificationCenter defaultCenter]
 addObserver:self
    selector:@selector(appDidEnterBackground:)
        name:UIApplicationDidEnterBackgroundNotification
      object:nil];

- (void)appDidEnterBackground:(NSNotification *)notification {
    NSLog(@"App เข้า background - บันทึกข้อมูล...");
    // [self saveCurrentState];
}
```

### NSNotificationCenter + NSUserDefaults

```objc
// สังเกตการเปลี่ยนแปลง NSUserDefaults
[[NSNotificationCenter defaultCenter]
 addObserver:self
    selector:@selector(defaultsChanged:)
        name:NSUserDefaultsDidChangeNotification
      object:nil];

- (void)defaultsChanged:(NSNotification *)notification {
    NSLog(@"NSUserDefaults เปลี่ยนแปลง");
    // reload settings
}
```

### Keyboard Notifications (iOS)

```objc
// ตัวอย่างจริง: จัดการ keyboard appearance
[[NSNotificationCenter defaultCenter]
 addObserver:self
    selector:@selector(keyboardWillShow:)
        name:UIKeyboardWillShowNotification
      object:nil];

[[NSNotificationCenter defaultCenter]
 addObserver:self
    selector:@selector(keyboardWillHide:)
        name:UIKeyboardWillHideNotification
      object:nil];

- (void)keyboardWillShow:(NSNotification *)notification {
    NSDictionary *info = notification.userInfo;
    CGRect keyboardFrame = [info[UIKeyboardFrameEndUserInfoKey] CGRectValue];
    NSTimeInterval duration = [info[UIKeyboardAnimationDurationUserInfoKey] doubleValue];
    
    NSLog(@"Keyboard สูง: %.0f px, duration: %.2f s", keyboardFrame.size.height, duration);
    // ปรับ scrollView.contentInset ให้พอดี
}

- (void)keyboardWillHide:(NSNotification *)notification {
    NSLog(@"Keyboard ซ่อน");
    // reset scrollView.contentInset
}
```

---

## แบบฝึกหัด

### แบบฝึกหัดที่ 1: Event Bus
สร้าง EventBus ที่รองรับ typed events:

```objc
// ออกแบบ EventBus ที่:
// - รองรับ event types ต่างๆ
// - ส่งข้อมูล typed แทน userInfo dictionary
// - Auto-remove observers เมื่อ observer ถูก dealloc
// - รองรับ one-time observers (ดูครั้งเดียวแล้วลบ)

EventBus *bus = [EventBus sharedBus];

// ลงทะเบียน
[bus on:@"orderCreated" handler:^(id data) {
    Order *order = (Order *)data;
    NSLog(@"Order: %@", order.orderID);
}];

// ส่ง event
[bus emit:@"orderCreated" data:newOrder];
```

### แบบฝึกหัดที่ 2: Undo/Redo System
สร้าง Undo/Redo system โดยใช้ notifications:

```objc
// ประกาศ notifications:
// DocumentWillPerformUndoNotification
// DocumentDidPerformUndoNotification
// DocumentWillPerformRedoNotification
// DocumentDidPerformRedoNotification
// DocumentDidChangeNotification

// สร้าง:
// - UndoManager ที่ post notifications เมื่อ undo/redo
// - DocumentViewController ที่ observe และอัปเดต UI
// - UndoHistoryPanel ที่แสดง list ของ undo operations
```

### แบบฝึกหัดที่ 3: Real-time Score Board
สร้าง score board สำหรับเกมที่:
- Players หลายคน post notification เมื่อได้คะแนน
- Score board observe ทุก notifications
- แสดง leaderboard อัปเดตแบบ real-time

```objc
// Notifications:
// PlayerScoredNotification
// PlayerJoinedNotification
// PlayerLeftNotification
// GameStartedNotification
// GameEndedNotification

// Classes:
// GamePlayer - post notifications เมื่อทำคะแนน
// ScoreBoard - observe และแสดงอันดับ
// GameController - จัดการ game lifecycle
```

### แบบฝึกหัดที่ 4: Network Status Monitor
สร้าง network monitor ที่แจ้ง app ทั้งหมดเมื่อ connectivity เปลี่ยน:

```objc
typedef NS_ENUM(NSInteger, NetworkStatus) {
    NetworkStatusNotReachable,
    NetworkStatusReachableViaWiFi,
    NetworkStatusReachableViaCellular,
};

extern NSNotificationName const NetworkStatusChangedNotification;
extern NSString * const NetworkStatusKey;
extern NSString * const NetworkPreviousStatusKey;

@interface NetworkMonitor : NSObject
@property (class, readonly) NetworkMonitor *sharedMonitor;
@property (readonly) NetworkStatus currentStatus;
- (void)startMonitoring;
- (void)stopMonitoring;
@end

// Components ที่ต้องตอบสนอง:
// - OfflineBanner: แสดง/ซ่อนเมื่อ offline
// - RequestQueue: retry requests เมื่อกลับมา online
// - CacheManager: switch mode ตาม network status
```

---

## สรุป

`NSNotificationCenter` เป็นเครื่องมือสำคัญสำหรับการสื่อสารระหว่าง objects ใน Objective-C:

1. **Broadcast Pattern** - ส่งข้อความไปยัง observers หลายตัวพร้อมกัน
2. **Loose Coupling** - sender ไม่ต้องรู้จัก receivers
3. **userInfo** - ส่งข้อมูลเพิ่มเติมพร้อม notification
4. **Constants** - ใช้ NSNotificationName constants เสมอ
5. **Thread Safety** - ระวัง thread ที่ post กับที่ handle

**ข้อควรระวัง:**
- ลบ observer ใน `dealloc` เสมอ
- หลีกเลี่ยง retain cycles ในblock-based observers
- ระวัง thread ที่ post notification vs thread ที่ต้องการอัปเดต UI

**เลือกใช้ NSNotificationCenter เมื่อ:**
- ต้องการ broadcast ไปหลาย objects
- ต้องการ loose coupling ระหว่าง modules
- App-wide events เช่น login/logout, theme change, network status
- ไม่ต้องการ return value จาก receivers
