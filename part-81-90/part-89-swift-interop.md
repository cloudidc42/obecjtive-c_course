# ตอนที่ 89: Objective-C และ Swift Interoperability

## บทนำ

ในปัจจุบัน Apple ได้แนะนำ Swift เป็นภาษาหลักสำหรับการพัฒนา iOS และ macOS อย่างไรก็ตาม Objective-C ยังคงมีบทบาทสำคัญในโปรเจกต์ขนาดใหญ่ที่มีโค้ดเก่า (legacy code) จำนวนมาก ความสามารถในการใช้งาน Objective-C และ Swift ร่วมกันในโปรเจกต์เดียวกันเรียกว่า **Interoperability** หรือการทำงานร่วมกันระหว่างสองภาษา

บทนี้จะครอบคลุมทุกแง่มุมของการทำงานร่วมกันระหว่าง Objective-C และ Swift ตั้งแต่พื้นฐานไปจนถึงเทคนิคขั้นสูง

---

## 89.1 ภาพรวมของ Interoperability

### ทำไมต้องใช้ทั้งสองภาษา?

```
โปรเจกต์ iOS จริงในองค์กร
├── Legacy Objective-C Code (70%)
│   ├── Models ที่ใช้มาหลายปี
│   ├── Networking layer เก่า
│   └── Custom UI Components
└── Swift Code ใหม่ (30%)
    ├── Feature ใหม่ๆ
    ├── SwiftUI views
    └── Swift Concurrency
```

ในสถานการณ์จริง นักพัฒนามักพบสถานการณ์เหล่านี้:

1. **โปรเจกต์เก่าที่ต้องการเพิ่มฟีเจอร์ใหม่ด้วย Swift** - ไม่ต้องเขียน Objective-C ใหม่ทั้งหมด
2. **Library ที่เขียนด้วย Objective-C** - ต้องการใช้ใน Swift project
3. **การ Migration แบบค่อยเป็นค่อยไป** - แปลงโค้ดทีละส่วน

### กลไกหลักของ Interoperability

Swift และ Objective-C สามารถทำงานร่วมกันผ่านกลไกหลัก 2 อย่าง:

1. **Bridging Header** - ให้ Swift เห็น Objective-C code
2. **Generated Swift Header** (`-Swift.h`) - ให้ Objective-C เห็น Swift code

---

## 89.2 Bridging Header

### การสร้าง Bridging Header

Bridging Header คือไฟล์ `.h` ที่ Swift compiler ใช้เพื่อ "เห็น" โค้ด Objective-C

**วิธีสร้างอัตโนมัติ:**

เมื่อคุณเพิ่มไฟล์ Objective-C ลงใน Swift project (หรือในทางกลับกัน) Xcode จะถามว่าต้องการสร้าง Bridging Header หรือไม่

```
Xcode Alert:
"Would you like to configure an Objective-C bridging header?"
[Don't Create] [Create Bridging Header]
```

**วิธีสร้างด้วยตนเอง:**

1. สร้างไฟล์ใหม่ชื่อ `ProjectName-Bridging-Header.h`
2. ไปที่ Build Settings > Swift Compiler - General
3. ตั้งค่า **Objective-C Bridging Header** เป็น path ของไฟล์

### ตัวอย่าง Bridging Header

```objc
// MyApp-Bridging-Header.h
// ไฟล์นี้เป็น entry point สำหรับ Swift ที่จะ import Objective-C classes

// Import framework หลัก
#import <Foundation/Foundation.h>
#import <UIKit/UIKit.h>

// Import Objective-C headers ที่ต้องการให้ Swift ใช้ได้
#import "UserManager.h"
#import "NetworkLayer.h"
#import "CacheManager.h"
#import "LegacyDataModel.h"
#import "CustomUIComponents/CustomButton.h"
#import "Utilities/DateFormatter+Extensions.h"

// Import third-party Objective-C libraries
#import <AFNetworking/AFNetworking.h>
#import <SDWebImage/SDWebImage.h>
```

### ตัวอย่าง Objective-C Class ที่จะใช้ใน Swift

```objc
// UserManager.h
#import <Foundation/Foundation.h>

@class User;

@interface UserManager : NSObject

@property (nonatomic, strong, readonly) User *currentUser;
@property (nonatomic, assign, getter=isLoggedIn) BOOL loggedIn;

+ (instancetype)sharedManager;
- (void)loginWithUsername:(NSString *)username 
                 password:(NSString *)password
               completion:(void (^)(BOOL success, NSError *error))completion;
- (void)logout;
- (NSArray<User *> *)getAllUsers;

@end
```

```objc
// UserManager.m
#import "UserManager.h"
#import "User.h"

@interface UserManager ()
@property (nonatomic, strong, readwrite) User *currentUser;
@property (nonatomic, assign, readwrite, getter=isLoggedIn) BOOL loggedIn;
@property (nonatomic, strong) NSMutableArray<User *> *users;
@end

@implementation UserManager

+ (instancetype)sharedManager {
    static UserManager *instance = nil;
    static dispatch_once_t onceToken;
    dispatch_once(&onceToken, ^{
        instance = [[UserManager alloc] init];
    });
    return instance;
}

- (instancetype)init {
    if (self = [super init]) {
        _users = [NSMutableArray array];
        _loggedIn = NO;
    }
    return self;
}

- (void)loginWithUsername:(NSString *)username 
                 password:(NSString *)password
               completion:(void (^)(BOOL success, NSError *error))completion {
    // Simulate login
    dispatch_async(dispatch_get_global_queue(DISPATCH_QUEUE_PRIORITY_DEFAULT, 0), ^{
        // Perform login logic
        BOOL success = [username isEqualToString:@"admin"] && 
                       [password isEqualToString:@"password"];
        
        dispatch_async(dispatch_get_main_queue(), ^{
            if (success) {
                self.loggedIn = YES;
                self.currentUser = [[User alloc] initWithUsername:username];
                if (completion) completion(YES, nil);
            } else {
                NSError *error = [NSError errorWithDomain:@"UserManagerError"
                                                    code:401
                                                userInfo:@{NSLocalizedDescriptionKey: @"Invalid credentials"}];
                if (completion) completion(NO, error);
            }
        });
    });
}

- (void)logout {
    self.loggedIn = NO;
    self.currentUser = nil;
}

- (NSArray<User *> *)getAllUsers {
    return [self.users copy];
}

@end
```

### การใช้ Objective-C Class จาก Swift

```swift
// ViewController.swift
import UIKit

class ViewController: UIViewController {
    
    override func viewDidLoad() {
        super.viewDidLoad()
        
        // ใช้ Objective-C class ได้เลยหลังจาก import ใน Bridging Header
        let userManager = UserManager.shared()
        
        // เรียกใช้ method ด้วย Swift syntax
        userManager.login(withUsername: "admin", password: "password") { success, error in
            if success {
                print("Login successful!")
                if let currentUser = userManager.currentUser {
                    print("Welcome, \(currentUser.username ?? "Unknown")")
                }
            } else if let error = error {
                print("Login failed: \(error.localizedDescription)")
            }
        }
        
        // ใช้ property
        if userManager.isLoggedIn {
            print("User is logged in")
        }
        
        // ใช้ lightweight generics
        let users: [User] = userManager.getAllUsers()
        print("Total users: \(users.count)")
    }
}
```

---

## 89.3 การใช้ Swift จาก Objective-C (Generated Header)

### Xcode Generated Header

เมื่อ Swift class ถูก compile Xcode จะสร้างไฟล์ header อัตโนมัติชื่อ `ProductModuleName-Swift.h` ที่ Objective-C สามารถ import ได้

```objc
// MyObjCClass.m
// Import generated Swift header
#import "MyApp-Swift.h"

@implementation MyObjCClass

- (void)useSwiftClass {
    // สร้าง Swift object
    SwiftUserService *service = [[SwiftUserService alloc] init];
    [service fetchUsersWithCompletion:^(NSArray *users, NSError *error) {
        NSLog(@"Got %lu users", (unsigned long)users.count);
    }];
}

@end
```

### ข้อกำหนดสำหรับ Swift Classes ที่ Objective-C จะใช้

Swift class ต้องมี:
1. Inherit จาก `NSObject` หรือ subclass ของมัน
2. มี `@objc` annotation หรือ `@objcMembers`
3. ไม่ใช้ Swift-only features (เช่น Struct, Enum with associated values)

```swift
// SwiftUserService.swift
import Foundation

// วิธีที่ 1: ใช้ @objc กับแต่ละ member
@objc class SwiftUserService: NSObject {
    
    @objc var isActive: Bool = false
    @objc private(set) var lastUpdated: Date?
    
    @objc func fetchUsers(completion: @escaping ([Any]?, Error?) -> Void) {
        // Implementation
        DispatchQueue.global().async {
            // Fetch users
            let users = [["name": "John"], ["name": "Jane"]]
            DispatchQueue.main.async {
                self.lastUpdated = Date()
                completion(users, nil)
            }
        }
    }
    
    @objc func updateUser(_ userID: String, name: String) -> Bool {
        // Update logic
        return true
    }
}
```

---

## 89.4 @objc และ @objcMembers

### @objc Annotation

`@objc` ทำให้ Swift declaration สามารถเข้าถึงได้จาก Objective-C และ Objective-C runtime

```swift
// SwiftComponents.swift
import Foundation
import UIKit

// @objc กับ class
@objc class AnalyticsManager: NSObject {
    
    // @objc กับ property
    @objc var sessionID: String = UUID().uuidString
    @objc var eventCount: Int = 0
    
    // @objc กับ method ธรรมดา
    @objc func trackEvent(_ eventName: String) {
        eventCount += 1
        print("Tracking: \(eventName) (Session: \(sessionID))")
    }
    
    // @objc กับ method ที่มี parameters
    @objc func trackEvent(_ eventName: String, 
                          properties: [String: Any]) {
        eventCount += 1
        print("Tracking: \(eventName) with \(properties.count) properties")
    }
    
    // @objc กับ optional method (สำหรับ protocol)
    @objc optional func optionalMethod()
    
    // @objc กับ computed property
    @objc var formattedSessionInfo: String {
        return "Session \(sessionID): \(eventCount) events"
    }
}

// @objc Protocol
@objc protocol AnalyticsDelegate: AnyObject {
    func analyticsManager(_ manager: AnalyticsManager, 
                          didTrackEvent eventName: String)
    @objc optional func analyticsManagerDidReset(_ manager: AnalyticsManager)
}
```

### @objcMembers

`@objcMembers` ทำให้ทุก member ของ class เข้าถึงได้จาก Objective-C โดยไม่ต้องใส่ `@objc` แต่ละตัว

```swift
// DataProcessor.swift
import Foundation

// @objcMembers ทำให้ทุก member เป็น @objc อัตโนมัติ
@objcMembers
class DataProcessor: NSObject {
    
    // ไม่ต้องใส่ @objc แยก
    var processorID: String = UUID().uuidString
    var isProcessing: Bool = false
    var processedCount: Int = 0
    
    func startProcessing() {
        isProcessing = true
        print("Processing started")
    }
    
    func stopProcessing() {
        isProcessing = false
        print("Processing stopped")
    }
    
    func processData(_ data: Data) -> Bool {
        processedCount += 1
        // Process data
        return true
    }
    
    // ยกเว้น member ที่ไม่ต้องการให้ Objective-C เข้าถึง
    // ใช้ @nonobjc
    @nonobjc func swiftOnlyMethod() {
        // This won't be visible to Objective-C
        print("Swift only!")
    }
    
    // หรือ Swift-only types ก็จะถูก exclude อัตโนมัติ
    func methodWithSwiftEnum(result: Result<String, Error>) {
        // Result type ไม่สามารถ bridge ไป ObjC ได้
        // จะถูก exclude อัตโนมัติ
    }
}
```

### การใช้ใน Objective-C

```objc
// MyViewController.m
#import "MyApp-Swift.h"

@implementation MyViewController

- (void)viewDidLoad {
    [super viewDidLoad];
    
    // ใช้ Swift class
    AnalyticsManager *analytics = [[AnalyticsManager alloc] init];
    analytics.sessionID = @"custom-session-123";
    
    [analytics trackEvent:@"page_view"];
    [analytics trackEvent:@"button_tap" 
               properties:@{@"button_id": @"submit", @"screen": @"home"}];
    
    NSLog(@"%@", analytics.formattedSessionInfo);
    
    // ใช้ DataProcessor
    DataProcessor *processor = [[DataProcessor alloc] init];
    [processor startProcessing];
    
    NSData *data = [@"test data" dataUsingEncoding:NSUTF8StringEncoding];
    BOOL success = [processor processData:data];
    NSLog(@"Processed: %@, count: %ld", success ? @"YES" : @"NO", 
          (long)processor.processedCount);
}

@end
```

---

## 89.5 NS_SWIFT_NAME Macro

`NS_SWIFT_NAME` ช่วยให้กำหนดชื่อที่ดีกว่าสำหรับ Swift เมื่อ import จาก Objective-C

### ปัญหาที่แก้ไขได้ด้วย NS_SWIFT_NAME

```objc
// ชื่อ Objective-C ที่ดูไม่ดีใน Swift
- (void)URLForPath:(NSString *)path completion:(void(^)(NSURL *url))completion;

// Swift จะแปลงเป็น:
// func url(forPath path: String, completion: (URL) -> Void)
// ซึ่งไม่สวยงาม
```

### ตัวอย่างการใช้ NS_SWIFT_NAME

```objc
// NetworkService.h
#import <Foundation/Foundation.h>

@interface NetworkService : NSObject

// เปลี่ยนชื่อ method ให้สวยขึ้นใน Swift
- (void)fetchDataForURL:(NSURL *)url 
            completion:(void (^)(NSData *data, NSError *error))completion
    NS_SWIFT_NAME(fetch(url:completion:));

// เปลี่ยนชื่อ class method
+ (instancetype)serviceWithBaseURL:(NSURL *)baseURL
    NS_SWIFT_NAME(init(baseURL:));

// เปลี่ยนชื่อ property
@property (nonatomic, copy) NSString *baseURLString 
    NS_SWIFT_NAME(baseURL);

// ซ่อน method จาก Swift (ถ้า Swift มี version ที่ดีกว่า)
- (NSArray *)getAllItemsWithError:(NSError **)error
    NS_SWIFT_NAME(getItems());

@end

// Renaming a free function
NSString *NSStringFromMyType(MyType type) NS_SWIFT_NAME(getter:MyType.description(self:));
```

```objc
// ตัวอย่างละเอียดกว่า
// APIClient.h

NS_ASSUME_NONNULL_BEGIN

typedef NS_ENUM(NSInteger, APIErrorCode) {
    APIErrorCodeUnknown = 0,
    APIErrorCodeNetworkFailure,
    APIErrorCodeInvalidResponse,
    APIErrorCodeAuthRequired
} NS_SWIFT_NAME(APIClient.ErrorCode);

@interface APIClient : NSObject

// เปลี่ยน factory method เป็น initializer ใน Swift
+ (instancetype)clientWithAPIKey:(NSString *)apiKey
    NS_SWIFT_NAME(init(apiKey:));

// เปลี่ยนให้ดูเป็น Swift idiom มากขึ้น
- (void)getUser:(NSString *)userID 
     completion:(void (^)(NSDictionary * _Nullable user, NSError * _Nullable error))completion
    NS_SWIFT_NAME(getUser(id:completion:));

// เปลี่ยน class method ให้เป็น static property
+ (APIClient *)defaultClient NS_SWIFT_NAME(default);

// Rename enum case
typedef NS_ENUM(NSInteger, RequestMethod) {
    RequestMethodGET,
    RequestMethodPOST,
    RequestMethodPUT,
    RequestMethodDELETE
} NS_SWIFT_NAME(HTTPMethod);

@end

NS_ASSUME_NONNULL_END
```

### การใช้งานใน Swift หลัง NS_SWIFT_NAME

```swift
// Swift usage
import Foundation

let client = APIClient(apiKey: "my-api-key")  // แทน APIClient.client(withAPIKey:)
let defaultClient = APIClient.default          // แทน APIClient.defaultClient()

// Method ที่สวยงามกว่า
client.getUser(id: "user123") { user, error in
    if let user = user {
        print("Got user: \(user)")
    }
}

// Service ที่ชื่อดีขึ้น
let service = NetworkService(baseURL: URL(string: "https://api.example.com")!)
service.fetch(url: URL(string: "https://api.example.com/data")!) { data, error in
    print("Fetched \(data?.count ?? 0) bytes")
}

// Enum ที่สวยงาม
let method: HTTPMethod = .get  // ดีกว่า RequestMethodGET
let errorCode = APIClient.ErrorCode.networkFailure
```

---

## 89.6 NS_REFINED_FOR_SWIFT

`NS_REFINED_FOR_SWIFT` ใช้เมื่อต้องการสร้าง Swift wrapper ที่ดีกว่าสำหรับ Objective-C method

```objc
// DataService.h
#import <Foundation/Foundation.h>

@interface DataService : NSObject

// Method นี้จะถูก prefix ด้วย __ ใน Swift
// เพื่อให้ Swift extension สามารถ override ด้วย version ที่ดีกว่า
- (nullable NSData *)dataForKey:(NSString *)key 
                          error:(NSError **)error
    NS_REFINED_FOR_SWIFT;

- (BOOL)setData:(NSData *)data 
         forKey:(NSString *)key
          error:(NSError **)error
    NS_REFINED_FOR_SWIFT;

// Method ที่ return multiple values
- (void)getBoundsWithMinX:(CGFloat *)minX 
                     minY:(CGFloat *)minY
                     maxX:(CGFloat *)maxX
                     maxY:(CGFloat *)maxY
    NS_REFINED_FOR_SWIFT;

@end
```

```swift
// DataService+Swift.swift
// Swift extension ที่ให้ API ที่ดีกว่า

extension DataService {
    
    // Swift version ที่ใช้ throws แทน NSError pointer
    func data(forKey key: String) throws -> Data {
        var error: NSError?
        // เรียก Objective-C method ด้วย __ prefix
        guard let data = __data(forKey: key, error: &error) else {
            throw error ?? NSError(domain: "DataServiceError", code: -1)
        }
        return data
    }
    
    // Swift version ที่ใช้ throws
    func setData(_ data: Data, forKey key: String) throws {
        var error: NSError?
        let success = __setData(data, forKey: key, error: &error)
        if !success {
            throw error ?? NSError(domain: "DataServiceError", code: -1)
        }
    }
    
    // Swift version ที่ return tuple แทน multiple out parameters
    var bounds: (minX: CGFloat, minY: CGFloat, maxX: CGFloat, maxY: CGFloat) {
        var minX: CGFloat = 0, minY: CGFloat = 0
        var maxX: CGFloat = 0, maxY: CGFloat = 0
        __getBoundsWithMinX(&minX, minY: &minY, maxX: &maxX, maxY: &maxY)
        return (minX, minY, maxX, maxY)
    }
}

// การใช้งาน
let service = DataService()

// Swift way (cleaner)
do {
    let data = try service.data(forKey: "profile")
    print("Data size: \(data.count)")
    
    try service.setData(data, forKey: "profile_backup")
    
    let (minX, minY, maxX, maxY) = service.bounds
    print("Bounds: (\(minX), \(minY)) to (\(maxX), \(maxY))")
} catch {
    print("Error: \(error)")
}
```

---

## 89.7 Nullability Annotations

### ปัญหาก่อน Nullability Annotations

ก่อนที่ Apple จะเพิ่ม nullability annotations ทุก Objective-C pointer จะถูก import เป็น Swift optional (`?`) ซึ่งทำให้ต้องใช้ optional binding ทุกที่

```swift
// โดยไม่มี nullability annotations
let name: String? = user.name  // ต้องเป็น optional เสมอ
if let name = name {           // ต้อง unwrap ทุกครั้ง
    print(name)
}
```

### Nullable และ Nonnull

```objc
// UserProfile.h
#import <Foundation/Foundation.h>

@interface UserProfile : NSObject

// nonnull - ค่าไม่มีทาง nil (ใน Swift เป็น non-optional)
@property (nonatomic, copy, nonnull) NSString *username;
@property (nonatomic, copy, nonnull) NSString *email;

// nullable - ค่าอาจเป็น nil ได้ (ใน Swift เป็น optional)
@property (nonatomic, copy, nullable) NSString *displayName;
@property (nonatomic, copy, nullable) NSString *bio;
@property (nonatomic, strong, nullable) NSURL *avatarURL;
@property (nonatomic, strong, nullable) NSDate *birthDate;

// Method parameters
- (void)updateBio:(nullable NSString *)bio;

// Return value
- (nonnull NSString *)formattedUsername;
- (nullable NSString *)formattedDisplayName;

// Completion blocks
- (void)fetchAvatarWithCompletion:(void (^ _Nullable)(UIImage * _Nullable image, NSError * _Nullable error))completion;

@end
```

### NS_ASSUME_NONNULL_BEGIN/END

แทนที่จะใส่ `nonnull` ทุกที่ สามารถใช้ macro นี้เพื่อทำให้ทุกอย่างเป็น `nonnull` โดย default

```objc
// CompleteUserProfile.h
#import <Foundation/Foundation.h>

NS_ASSUME_NONNULL_BEGIN  // ทุกอย่างใน block นี้เป็น nonnull โดย default

@interface CompleteUserProfile : NSObject

// nonnull โดย default (ไม่ต้องระบุ)
@property (nonatomic, copy) NSString *userID;
@property (nonatomic, copy) NSString *username;
@property (nonatomic, copy) NSString *email;

// ต้องระบุ nullable อย่างชัดเจน
@property (nonatomic, copy, nullable) NSString *displayName;
@property (nonatomic, copy, nullable) NSString *bio;
@property (nonatomic, strong, nullable) NSURL *avatarURL;

// Method parameters (nonnull โดย default)
- (void)updateEmail:(NSString *)email;

// nullable parameter ต้องระบุชัดเจน
- (void)updateDisplayName:(nullable NSString *)displayName;

// Return values
- (NSString *)formattedInfo;                     // nonnull
- (nullable NSString *)shortBio;                 // nullable
- (NSArray<NSString *> *)allPropertyNames;       // nonnull array ของ nonnull strings

// Block parameters
- (void)performAction:(void (^)(BOOL success))completion;
- (void)fetchData:(void (^ _Nullable)(NSData * _Nullable data, NSError * _Nullable error))completion;

@end

// Function
NSString *formatUserID(NSString *userID);
NSString * _Nullable findUserByEmail(NSString * _Nullable email);

NS_ASSUME_NONNULL_END
```

### ผลลัพธ์ใน Swift

```swift
// Swift view ของ CompleteUserProfile
let profile = CompleteUserProfile()

// nonnull properties - ไม่ต้อง unwrap
let userID: String = profile.userID
let username: String = profile.username
profile.updateEmail("new@email.com")  // parameter ไม่ต้อง optional

// nullable properties - ต้อง handle optional
let displayName: String? = profile.displayName
let bio: String? = profile.bio
let avatarURL: URL? = profile.avatarURL

// Method calls
profile.updateDisplayName(nil)  // OK - parameter เป็น nullable
profile.updateDisplayName("John Doe")  // OK

// Return values
let info: String = profile.formattedInfo  // ไม่ต้อง unwrap!
let shortBio: String? = profile.shortBio  // ต้อง handle optional
let names: [String] = profile.allPropertyNames  // array ชัดเจน

// Block
profile.performAction { success in  // parameter ไม่ต้อง optional
    print("Success: \(success)")
}

profile.fetchData { data, error in  // both optional
    if let data = data {
        print("Got \(data.count) bytes")
    }
}
```

---

## 89.8 Lightweight Generics

Objective-C รองรับ lightweight generics เพื่อให้ Swift มี type information ที่ดีขึ้น

```objc
// TypedContainers.h
#import <Foundation/Foundation.h>

NS_ASSUME_NONNULL_BEGIN

// Generic class
@interface Stack<ObjectType> : NSObject

- (void)push:(ObjectType)object;
- (nullable ObjectType)pop;
- (nullable ObjectType)peek;
@property (nonatomic, readonly) NSUInteger count;
@property (nonatomic, readonly, getter=isEmpty) BOOL empty;

// Typed collections
- (NSArray<ObjectType> *)allObjects;

@end

// ใช้ generics กับ NSArray, NSDictionary, NSSet
@interface UserRepository : NSObject

// Without generics (เดิม)
// @property (nonatomic, strong) NSArray *users;

// With generics (ดีกว่า)
@property (nonatomic, strong) NSArray<User *> *users;
@property (nonatomic, strong) NSDictionary<NSString *, User *> *usersByID;
@property (nonatomic, strong) NSSet<NSString *> *blockedUserIDs;

- (nullable User *)userWithID:(NSString *)userID;
- (NSArray<User *> *)usersMatchingPredicate:(NSPredicate *)predicate;

@end

NS_ASSUME_NONNULL_END
```

```objc
// Stack.m
@implementation Stack {
    NSMutableArray *_storage;
}

- (instancetype)init {
    if (self = [super init]) {
        _storage = [NSMutableArray array];
    }
    return self;
}

- (void)push:(id)object {
    [_storage addObject:object];
}

- (nullable id)pop {
    if (_storage.count == 0) return nil;
    id object = _storage.lastObject;
    [_storage removeLastObject];
    return object;
}

- (nullable id)peek {
    return _storage.lastObject;
}

- (NSUInteger)count {
    return _storage.count;
}

- (BOOL)isEmpty {
    return _storage.count == 0;
}

- (NSArray *)allObjects {
    return [_storage copy];
}

@end
```

### การใช้ Lightweight Generics ใน Swift

```swift
// Swift usage ของ Objective-C generics

// Stack มี type information ที่ดีขึ้น
let stringStack = Stack<String>()
stringStack.push("Hello")
stringStack.push("World")

let top: String? = stringStack.pop()  // ได้ String? ไม่ใช่ Any?
print(top ?? "Empty")

// UserRepository
let repo = UserRepository()
let users: [User] = repo.users  // ได้ [User] ไม่ใช่ [Any]
let usersByID: [String: User] = repo.usersByID  // typed dictionary

if let user = repo.user(withID: "123") {
    print("Found user: \(user.username)")
}

// Typed arrays ใน Objective-C
let nsMutableArray = NSMutableArray()
// vs
let typedArray: NSMutableArray = NSMutableArray()
```

### __covariant และ __contravariant

```objc
// Variance annotations สำหรับ generics
@interface Container<__covariant ObjectType> : NSObject

// __covariant: Container<UIButton> เป็น subtype ของ Container<UIView>
// (ทิศทางเดียวกัน - ปลอดภัยสำหรับ output/read-only)
- (ObjectType)item;
- (NSArray<ObjectType> *)allItems;

@end

@interface Processor<__contravariant ObjectType> : NSObject

// __contravariant: ทิศทางตรงข้าม - ปลอดภัยสำหรับ input/write-only
- (void)process:(ObjectType)item;

@end
```

---

## 89.9 NS_ENUM และ NS_OPTIONS กับ Swift

### NS_ENUM

```objc
// Enumerations.h

// NS_ENUM สำหรับ enums ทั่วไป
typedef NS_ENUM(NSInteger, UserStatus) {
    UserStatusUnknown = 0,
    UserStatusActive,
    UserStatusInactive,
    UserStatusSuspended,
    UserStatusDeleted
} NS_SWIFT_NAME(User.Status);  // ใน Swift เป็น User.Status

// NS_OPTIONS สำหรับ bitmask/flags
typedef NS_OPTIONS(NSUInteger, NotificationPermissions) {
    NotificationPermissionsNone     = 0,
    NotificationPermissionsBadge    = 1 << 0,
    NotificationPermissionsAlert    = 1 << 1,
    NotificationPermissionsSound    = 1 << 2,
    NotificationPermissionsAll      = (NotificationPermissionsBadge | 
                                       NotificationPermissionsAlert | 
                                       NotificationPermissionsSound)
} NS_SWIFT_NAME(NotificationPermissionSet);

// NS_CLOSED_ENUM สำหรับ enum ที่ไม่มีค่าใหม่เพิ่มในอนาคต
// (ช่วยให้ Swift compiler รู้ว่า exhaustive switch ปลอดภัย)
typedef NS_CLOSED_ENUM(NSInteger, Direction) {
    DirectionNorth,
    DirectionSouth,
    DirectionEast,
    DirectionWest
};

// NS_ERROR_ENUM สำหรับ error domains
typedef NS_ERROR_ENUM(NSString *, NetworkErrorDomain) {
    NetworkErrorTimeout = -1001,
    NetworkErrorNoConnection = -1009,
    NetworkErrorInvalidURL = -1003
} NS_SWIFT_NAME(NetworkError);
```

### การใช้งานใน Swift

```swift
// Swift usage ของ NS_ENUM
let status: User.Status = .active  // ไม่ต้อง UserStatusActive

switch status {
case .unknown:
    print("Unknown")
case .active:
    print("Active user")
case .inactive:
    print("Inactive")
case .suspended:
    print("Suspended")
case .deleted:
    print("Deleted")
@unknown default:
    print("New case added")
}

// NS_OPTIONS ใน Swift เป็น OptionSet
var permissions: NotificationPermissionSet = [.badge, .alert, .sound]

// ตรวจสอบ
if permissions.contains(.alert) {
    print("Alert permission granted")
}

// เพิ่ม/ลบ
permissions.insert(.badge)
permissions.remove(.sound)

// NS_CLOSED_ENUM ใน Swift
let direction: Direction = .north
switch direction {
case .north: print("Going north")
case .south: print("Going south")
case .east:  print("Going east")
case .west:  print("Going west")
// ไม่ต้องมี default เพราะ exhaustive!
}

// NS_ERROR_ENUM
do {
    try makeNetworkRequest()
} catch NetworkError.timeout {
    print("Request timed out")
} catch NetworkError.noConnection {
    print("No internet connection")
} catch {
    print("Other error: \(error)")
}
```

---

## 89.10 Bridging Collections

### NSArray ↔ Swift Array

```swift
// Objective-C to Swift
let objcArray: NSArray = ["Apple", "Banana", "Cherry"] as NSArray

// แปลงเป็น Swift Array
let swiftArray: [String] = objcArray as! [String]
// หรือ
let safeSwiftArray = objcArray as? [String] ?? []

// Swift to Objective-C
let fruits = ["Apple", "Banana", "Cherry"]
let objcFruits: NSArray = fruits as NSArray
// หรือ
let mutableFruits: NSMutableArray = NSMutableArray(array: fruits)

// Bridging ทำงานอัตโนมัติเมื่อส่งให้ Objective-C
func sendToObjCMethod(_ array: [String]) {
    // objcMethod รับ NSArray แต่ส่ง [String] ได้เลย
    SomeObjCClass.processStrings(array)  // auto-bridge
}
```

### NSDictionary ↔ Swift Dictionary

```swift
// Objective-C NSDictionary to Swift Dictionary
let objcDict: NSDictionary = ["key1": "value1", "key2": "value2"] as NSDictionary

// แปลงเป็น Swift Dictionary
let swiftDict: [String: String] = objcDict as! [String: String]

// Swift Dictionary to Objective-C
let config: [String: Any] = ["timeout": 30, "retries": 3, "debug": true]
let objcConfig: NSDictionary = config as NSDictionary
let mutableConfig: NSMutableDictionary = NSMutableDictionary(dictionary: config)

// Typed NSDictionary จาก Objective-C
// ObjC: @property NSMutableDictionary<NSString *, User *> *userCache;
let cache = userManager.userCache  // ได้เป็น NSMutableDictionary เดิม
// หรือถ้ามี lightweight generics จะได้ Swift typed
```

### NSSet ↔ Swift Set

```swift
// NSSet bridging
let objcSet: NSSet = NSSet(array: [1, 2, 3, 4, 5])
let swiftSet: Set<Int> = objcSet as! Set<Int>

let swiftStringSet: Set<String> = ["one", "two", "three"]
let objcStringSet: NSSet = swiftStringSet as NSSet

// NSOrderedSet (ไม่มี Swift counterpart ตรงๆ)
let orderedSet = NSOrderedSet(array: ["first", "second", "third"])
let array = orderedSet.array as? [String] ?? []
```

### String Bridging

```swift
// NSString ↔ String bridging เป็นอัตโนมัติ
let swiftString: String = "Hello, World!"
let nsString: NSString = swiftString as NSString

// ใช้ NSString methods ได้บน String
let range = swiftString.range(of: "World")
let uppercased = (swiftString as NSString).uppercased

// ระวัง: NSString methods บางอย่าง return NSString
let replaced = (swiftString as NSString).replacingOccurrences(of: "World", 
                                                               with: "Swift")
// replaced เป็น String (auto-bridge กลับ)
```

---

## 89.11 Error Handling Bridging (NSError ↔ Swift throws)

### NSError ใน Objective-C

```objc
// ErrorHandling.h
#import <Foundation/Foundation.h>

// Error domain constant
extern NSString * const FileProcessingErrorDomain;

// Error codes
typedef NS_ERROR_ENUM(NSString *, FileProcessingError) {
    FileProcessingErrorFileNotFound = 1,
    FileProcessingErrorPermissionDenied,
    FileProcessingErrorInvalidFormat,
    FileProcessingErrorDiskFull
} NS_SWIFT_NAME(FileProcessor.Error);

@interface FileProcessor : NSObject

// Pattern ที่ 1: Return BOOL + NSError** (แปลงเป็น throws ใน Swift)
- (BOOL)processFile:(NSString *)path 
              error:(NSError **)error;

// Pattern ที่ 2: Return object + NSError** (แปลงเป็น throws ใน Swift)
- (nullable NSData *)readFile:(NSString *)path 
                        error:(NSError **)error;

// Pattern ที่ 3: Completion block with error (ยังคงเป็น completion ใน Swift)
- (void)processFileAsync:(NSString *)path 
              completion:(void (^)(BOOL success, NSError * _Nullable error))completion;

@end
```

```objc
// FileProcessor.m
#import "FileProcessor.h"

NSString * const FileProcessingErrorDomain = @"FileProcessingErrorDomain";

@implementation FileProcessor

- (BOOL)processFile:(NSString *)path error:(NSError **)error {
    if (![[NSFileManager defaultManager] fileExistsAtPath:path]) {
        if (error) {
            *error = [NSError errorWithDomain:FileProcessingErrorDomain
                                         code:FileProcessingErrorFileNotFound
                                     userInfo:@{
                NSLocalizedDescriptionKey: @"File not found",
                NSLocalizedFailureReasonErrorKey: [NSString stringWithFormat:@"No file at path: %@", path]
            }];
        }
        return NO;
    }
    
    // Process file...
    return YES;
}

- (nullable NSData *)readFile:(NSString *)path error:(NSError **)error {
    NSData *data = [NSData dataWithContentsOfFile:path options:0 error:error];
    return data;
}

- (void)processFileAsync:(NSString *)path 
              completion:(void (^)(BOOL success, NSError * _Nullable error))completion {
    dispatch_async(dispatch_get_global_queue(DISPATCH_QUEUE_PRIORITY_DEFAULT, 0), ^{
        NSError *error = nil;
        BOOL success = [self processFile:path error:&error];
        dispatch_async(dispatch_get_main_queue(), ^{
            if (completion) completion(success, error);
        });
    });
}

@end
```

### การใช้งานใน Swift

```swift
// Swift ใช้ Objective-C error handling

let processor = FileProcessor()

// Pattern 1: BOOL + NSError** → throws
do {
    try processor.processFile("/path/to/file.txt")
    print("File processed successfully")
} catch FileProcessor.Error.fileNotFound {
    print("File not found!")
} catch FileProcessor.Error.permissionDenied {
    print("Permission denied!")
} catch {
    print("Error: \(error.localizedDescription)")
}

// Pattern 2: Return value + NSError** → throws + returns value
do {
    let data = try processor.readFile("/path/to/data.bin")
    print("Read \(data.count) bytes")
} catch {
    print("Failed to read: \(error)")
}

// Pattern 3: Completion block (ไม่เปลี่ยน)
processor.processFileAsync("/path/to/file.txt") { success, error in
    if success {
        print("Async processing done")
    } else if let error = error as? FileProcessor.Error {
        switch error {
        case .fileNotFound:
            print("Not found")
        case .permissionDenied:
            print("No permission")
        default:
            print("Other error")
        }
    }
}

// แปลง NSError เป็น typed Error
if let nsError = error as NSError? {
    print("Domain: \(nsError.domain)")
    print("Code: \(nsError.code)")
    print("User info: \(nsError.userInfo)")
}
```

---

## 89.12 Blocks ↔ Closures

### การแปลง Block เป็น Closure

Objective-C blocks และ Swift closures มีความเข้ากันได้โดยธรรมชาติ

```objc
// BlocksAPI.h - Objective-C API ที่ใช้ Blocks

typedef void (^CompletionHandler)(BOOL success, NSError * _Nullable error);
typedef void (^ProgressHandler)(float progress);
typedef NSString * (^TransformBlock)(NSString *input);

@interface DataDownloader : NSObject

// Simple block
- (void)downloadWithCompletion:(CompletionHandler)completion;

// Block ที่มี progress
- (void)downloadURL:(NSURL *)url
           progress:(nullable ProgressHandler)progress
         completion:(CompletionHandler)completion;

// Block ที่ return value
- (NSArray<NSString *> *)transformItems:(NSArray<NSString *> *)items
                              transform:(TransformBlock)transform;

// Nullable block
- (void)performOptionalTask:(nullable void (^)(void))completion;

@end
```

### การใช้งานใน Swift

```swift
// Swift ใช้ Objective-C Blocks

let downloader = DataDownloader()

// Block เป็น trailing closure ใน Swift
downloader.download { success, error in
    print("Done: \(success)")
}

// Named closure variable
let progressHandler: ((Float) -> Void) = { progress in
    print("Progress: \(progress * 100)%")
}

let completionHandler: ((Bool, Error?) -> Void) = { success, error in
    if success {
        print("Downloaded successfully")
    } else {
        print("Error: \(error?.localizedDescription ?? "unknown")")
    }
}

downloader.downloadURL(URL(string: "https://example.com/file")!,
                       progress: progressHandler,
                       completion: completionHandler)

// Transform block
let items = ["hello", "world", "swift"]
let uppercased = downloader.transformItems(items) { input in
    input.uppercased()
}

// Optional block
downloader.performOptionalTask(nil)
downloader.performOptionalTask {
    print("Task completed")
}
```

### @escaping และ Objective-C Blocks

```objc
// EscapingExample.h
@interface AsyncProcessor : NSObject

// Block ที่ถูกเรียกหลังจาก method return (escaping)
- (void)startProcessingWithCompletion:(void (^)(NSArray *results))completion;

@end
```

```swift
// Swift รู้อัตโนมัติว่า block เป็น @escaping
// เพราะ Objective-C blocks ถือเป็น @escaping โดย default

let processor = AsyncProcessor()

// ต้องใช้ [weak self] เพื่อหลีกเลี่ยง retain cycle
processor.startProcessing { [weak self] results in
    guard let self = self else { return }
    self.handleResults(results)
}

// Storing block-as-closure
class MyViewController: UIViewController {
    var pendingBlock: ((Bool) -> Void)?
    
    func setupProcessor() {
        let processor = AsyncProcessor()
        processor.startProcessing { [weak self] results in
            self?.pendingBlock?(results.count > 0)
        }
    }
}
```

---

## 89.13 Mixed-Language Project Setup

### โครงสร้างโปรเจกต์แบบ Mixed

```
MyMixedApp/
├── MyMixedApp-Bridging-Header.h    ← Objective-C → Swift
├── MyMixedApp/
│   ├── AppDelegate.swift
│   ├── SceneDelegate.swift
│   ├── ViewControllers/
│   │   ├── MainViewController.swift    (Swift)
│   │   ├── ProfileViewController.m    (Objective-C)
│   │   └── ProfileViewController.h
│   ├── Models/
│   │   ├── User.h                     (Objective-C)
│   │   ├── User.m
│   │   ├── Post.swift                 (Swift)
│   │   └── Comment.swift
│   ├── Services/
│   │   ├── NetworkService.h           (Objective-C)
│   │   ├── NetworkService.m
│   │   ├── AuthService.swift          (Swift)
│   │   └── CacheService.swift
│   └── Utilities/
│       ├── DateUtility.h             (Objective-C)
│       ├── DateUtility.m
│       └── StringExtension.swift
├── MyMixedApp.xcodeproj
└── Tests/
    ├── ObjCTests.m
    └── SwiftTests.swift
```

### Build Settings สำคัญ

```
Build Settings:
├── Swift Compiler - General
│   ├── Objective-C Bridging Header: "$(SRCROOT)/MyMixedApp/MyMixedApp-Bridging-Header.h"
│   └── Install Objective-C Compatibility Header: YES
├── Build Options
│   └── Always Embed Swift Standard Libraries: YES (for frameworks with Swift)
└── Packaging
    └── Product Module Name: "MyMixedApp" (used in -Swift.h filename)
```

### ตัวอย่าง Mixed-Language Communication

```objc
// LegacyViewController.h - Objective-C ViewController ที่ใช้ Swift services

#import <UIKit/UIKit.h>
@class SwiftAuthService;

@interface LegacyViewController : UIViewController

@property (nonatomic, strong) SwiftAuthService *authService;

@end
```

```objc
// LegacyViewController.m
#import "LegacyViewController.h"
#import "MyMixedApp-Swift.h"  // Import Swift generated header

@implementation LegacyViewController

- (void)viewDidLoad {
    [super viewDidLoad];
    
    self.authService = [[SwiftAuthService alloc] init];
    
    // ใช้ Swift service จาก Objective-C
    [self.authService loginWithEmail:@"user@example.com" 
                            password:@"password123"
                          completion:^(BOOL success, NSError * _Nullable error) {
        if (success) {
            NSLog(@"Login success!");
        } else {
            NSLog(@"Login failed: %@", error.localizedDescription);
        }
    }];
}

@end
```

```swift
// SwiftAuthService.swift - Swift service ที่ Objective-C เรียกใช้

import Foundation

@objc class SwiftAuthService: NSObject {
    
    @objc var isAuthenticated: Bool = false
    @objc var currentUserID: String?
    
    @objc func login(withEmail email: String,
                     password: String,
                     completion: @escaping (Bool, Error?) -> Void) {
        // Swift async networking (แต่ expose เป็น completion block)
        Task {
            do {
                let userID = try await performLogin(email: email, password: password)
                isAuthenticated = true
                currentUserID = userID
                completion(true, nil)
            } catch {
                completion(false, error)
            }
        }
    }
    
    private func performLogin(email: String, password: String) async throws -> String {
        // Actual login logic
        try await Task.sleep(nanoseconds: 1_000_000_000)
        return "user_\(email.hashValue)"
    }
    
    @objc func logout() {
        isAuthenticated = false
        currentUserID = nil
    }
}
```

---

## 89.14 Migration Strategies จาก Objective-C ไป Swift

### Strategy 1: Incremental Migration (แนะนำ)

```
Phase 1: เพิ่ม Swift ทีละส่วน (ไม่แตะ Objective-C เดิม)
├── เพิ่มฟีเจอร์ใหม่ด้วย Swift
├── สร้าง Swift wrappers สำหรับ Objective-C classes
└── Swift extensions สำหรับ Objective-C classes

Phase 2: แปลง Models (เริ่มจากส่วนที่แยกส่วน)
├── แปลง Data models เป็น Swift structs
├── แปลง Enums
└── แปลง Constants

Phase 3: แปลง Services (Business Logic)
├── แปลง Networking layer
├── แปลง Data parsing
└── แปลง Utility classes

Phase 4: แปลง ViewControllers (ส่วนที่ซับซ้อนที่สุด)
├── เริ่มจาก ViewControllers ที่เล็กที่สุด
├── แปลง Views และ Cells
└── สุดท้ายแปลง AppDelegate และ main files
```

### ตัวอย่าง Migration ของ Model

```objc
// เดิม: Objective-C User model
// User.h (Objective-C)
@interface User : NSObject <NSCoding>
@property (nonatomic, copy) NSString *userID;
@property (nonatomic, copy) NSString *username;
@property (nonatomic, copy, nullable) NSString *email;
@property (nonatomic, strong) NSDate *createdAt;
@property (nonatomic, assign) NSInteger followersCount;
@end
```

```swift
// ใหม่: Swift User model
// User.swift (Swift)
struct User: Codable {
    let userID: String
    let username: String
    let email: String?
    let createdAt: Date
    let followersCount: Int
    
    enum CodingKeys: String, CodingKey {
        case userID = "user_id"
        case username
        case email
        case createdAt = "created_at"
        case followersCount = "followers_count"
    }
}

// Objective-C Compatibility wrapper (ช่วง Transition)
@objc class UserObjCBridge: NSObject {
    let user: User
    
    @objc init(userID: String, username: String, email: String?, followersCount: Int) {
        self.user = User(
            userID: userID,
            username: username,
            email: email,
            createdAt: Date(),
            followersCount: followersCount
        )
    }
    
    @objc var userID: String { user.userID }
    @objc var username: String { user.username }
    @objc var email: String? { user.email }
    @objc var followersCount: Int { user.followersCount }
}
```

### Strategy 2: Parallel Development

```swift
// สร้าง Swift version ควบคู่กับ Objective-C
// แล้ว gradually เปลี่ยน callers ทีละส่วน

// Objective-C version ยังอยู่
// + Swift version ใหม่

// NetworkService.swift (Swift version)
class NetworkService {
    func fetch<T: Decodable>(_ type: T.Type, from url: URL) async throws -> T {
        let (data, _) = try await URLSession.shared.data(from: url)
        return try JSONDecoder().decode(type, from: data)
    }
}

// Legacy Objective-C callers ยังใช้ NetworkServiceObjC.h ได้
// New Swift code ใช้ NetworkService.swift
```

### Best Practices สำหรับ Migration

```swift
// 1. สร้าง Swift Extension สำหรับ Objective-C class ก่อน Migration

// Extension เพิ่ม functionality ใหม่ด้วย Swift
extension NSArray {
    func firstWhere(_ predicate: (Any) -> Bool) -> Any? {
        return first { predicate($0) }
    }
}

// 2. ใช้ typealiases เพื่อให้ง่ายต่อการ migrate
typealias LegacyUser = User  // ชั่วคราวระหว่าง migration

// 3. Protocol-based bridging
protocol UserProtocol {
    var userID: String { get }
    var username: String { get }
    var email: String? { get }
}

// Objective-C class conform to Swift protocol
extension LegacyUser: UserProtocol {
    var userID: String { self.userId ?? "" }
}

// Swift struct ก็ conform to protocol เดียวกัน
struct NewUser: UserProtocol {
    let userID: String
    let username: String  
    let email: String?
}
```

---

## 89.15 แนวปฏิบัติที่ดีในการทำ Interoperability

### 1. Naming Conventions

```objc
// Objective-C: ใช้ verbose naming
- (void)setUserName:(NSString *)userName;
- (NSString *)getUserName;

// ดีกว่า: ใช้ property
@property (nonatomic, copy) NSString *userName;
```

```swift
// Swift: ใช้ concise naming
var userName: String
```

### 2. จัดการ Memory ที่ถูกต้อง

```objc
// ใน Objective-C: ระวัง retain cycles ใน blocks
@interface MyClass : NSObject
@property (nonatomic, copy) void (^completionBlock)(void);
@end

@implementation MyClass
- (void)setup {
    // ระวัง: self retain block, block retain self = retain cycle!
    __weak typeof(self) weakSelf = self;
    self.completionBlock = ^{
        __strong typeof(weakSelf) strongSelf = weakSelf;
        if (strongSelf) {
            [strongSelf doSomething];
        }
    };
}
@end
```

```swift
// Swift: ใช้ [weak self]
class MyClass {
    var completionBlock: (() -> Void)?
    
    func setup() {
        completionBlock = { [weak self] in
            self?.doSomething()
        }
    }
}
```

### 3. Thread Safety

```objc
// Objective-C: ใช้ dispatch queues
@interface ThreadSafeCache : NSObject
@end

@implementation ThreadSafeCache {
    NSMutableDictionary *_cache;
    dispatch_queue_t _queue;
}

- (instancetype)init {
    if (self = [super init]) {
        _cache = [NSMutableDictionary dictionary];
        _queue = dispatch_queue_create("com.app.cache", DISPATCH_QUEUE_CONCURRENT);
    }
    return self;
}

- (id)objectForKey:(NSString *)key {
    __block id object;
    dispatch_sync(_queue, ^{
        object = self->_cache[key];
    });
    return object;
}

- (void)setObject:(id)object forKey:(NSString *)key {
    dispatch_barrier_async(self->_queue, ^{
        self->_cache[key] = object;
    });
}

@end
```

### 4. Testing Mixed-Language Code

```swift
// SwiftTests.swift
import XCTest
@testable import MyMixedApp

class InteropTests: XCTestCase {
    
    func testObjectiveCClassFromSwift() {
        // Test Objective-C class in Swift test
        let userManager = UserManager.shared()
        XCTAssertNotNil(userManager)
        XCTAssertFalse(userManager.isLoggedIn)
    }
    
    func testSwiftClassFromObjCPerspective() {
        // Test Swift class that will be used from Objective-C
        let service = SwiftAuthService()
        XCTAssertFalse(service.isAuthenticated)
        XCTAssertNil(service.currentUserID)
    }
    
    func testBridgingCollections() {
        let objcArray: NSArray = ["a", "b", "c"] as NSArray
        let swiftArray = objcArray as? [String]
        XCTAssertEqual(swiftArray, ["a", "b", "c"])
        
        let swiftDict = ["key": "value"]
        let objcDict = swiftDict as NSDictionary
        XCTAssertEqual(objcDict["key"] as? String, "value")
    }
}
```

---

## 89.16 แบบฝึกหัด

### แบบฝึกหัดที่ 1: สร้าง Bridging Header

สร้างโปรเจกต์ใหม่แบบ Mixed-Language:

```
งาน:
1. สร้าง iOS project ด้วย Swift
2. เพิ่ม Objective-C class ชื่อ "LegacyCalculator" ที่มี:
   - method บวก ลบ คูณ หาร
   - property สำหรับเก็บผลลัพธ์ล่าสุด
3. สร้าง Bridging Header
4. เรียกใช้ LegacyCalculator จาก Swift ViewController
5. แสดงผลลัพธ์บน UILabel
```

```objc
// LegacyCalculator.h (สร้างไฟล์นี้)
#import <Foundation/Foundation.h>

NS_ASSUME_NONNULL_BEGIN

@interface LegacyCalculator : NSObject

@property (nonatomic, readonly) double lastResult;
@property (nonatomic, readonly) NSString *lastOperation;

- (double)add:(double)a to:(double)b;
- (double)subtract:(double)b from:(double)a;
- (double)multiply:(double)a by:(double)b;
- (nullable NSNumber *)divide:(double)a by:(double)b error:(NSError **)error;
- (void)reset;
- (NSString *)historyDescription;

@end

NS_ASSUME_NONNULL_END
```

### แบบฝึกหัดที่ 2: @objc Protocol

```swift
// สร้าง Swift protocol ที่ Objective-C ใช้ได้

// 1. สร้าง @objc protocol สำหรับ data source
@objc protocol TableDataSource: AnyObject {
    var numberOfItems: Int { get }
    func item(at index: Int) -> Any
    @objc optional func titleForSection(_ section: Int) -> String?
}

// 2. สร้าง Swift class ที่ implement protocol
class UserListDataSource: NSObject, TableDataSource {
    // ... implement ที่นี่
}

// 3. สร้าง Objective-C class ที่ใช้ protocol
// ObjCTableViewController.h
// @property (nonatomic, weak) id<TableDataSource> dataSource;
```

### แบบฝึกหัดที่ 3: Error Bridging

```objc
// สร้าง Objective-C API ที่ใช้ NSError
// แล้ว wrap ด้วย Swift throws

// FileValidator.h
typedef NS_ERROR_ENUM(NSString *, FileValidationError) {
    FileValidationErrorFileTooLarge = 1,
    FileValidationErrorInvalidFormat,
    FileValidationErrorMissingPermission
} NS_SWIFT_NAME(FileValidator.ValidationError);

@interface FileValidator : NSObject
- (BOOL)validateFile:(NSString *)path 
           maxSizeMB:(NSInteger)maxSizeMB 
               error:(NSError **)error;
@end
```

```swift
// Swift extension ที่ให้ throw API
extension FileValidator {
    func validate(filePath: String, maxSizeMB: Int) throws {
        // Implement using __validateFile:maxSizeMB:error:
        // (ถ้าใช้ NS_REFINED_FOR_SWIFT)
    }
}
```

### แบบฝึกหัดที่ 4: Migration Challenge

```
โจทย์: Migration Project
มี Objective-C app ที่มี:
- UserModel (NSObject subclass)
- PostModel (NSObject subclass)
- NetworkManager (NSObject, singleton)
- MainViewController (UIViewController)

งาน:
1. แปลง UserModel และ PostModel เป็น Swift structs
2. สร้าง Swift NetworkManager ใหม่ (async/await)
3. สร้าง Compatibility bridges
4. ทดสอบว่า Objective-C code เดิมยังทำงานได้
5. Migrate MainViewController เป็น Swift ในขั้นสุดท้าย
```

---

## สรุป

ในบทนี้เราได้เรียนรู้:

1. **Bridging Header** - วิธีให้ Swift เห็น Objective-C code
2. **Generated Header** - วิธีให้ Objective-C เห็น Swift code
3. **@objc และ @objcMembers** - การ expose Swift ให้ Objective-C runtime
4. **NS_SWIFT_NAME** - การปรับแต่งชื่อสำหรับ Swift
5. **NS_REFINED_FOR_SWIFT** - การสร้าง Swift wrappers ที่ดีกว่า
6. **Nullability** - การบอก Swift เรื่อง null safety
7. **Lightweight Generics** - Type safety สำหรับ collections
8. **NS_ENUM และ NS_OPTIONS** - Enumerations ที่ใช้ร่วมกันได้ดี
9. **Collection Bridging** - การแปลง NSArray/NSDictionary/NSSet
10. **Error Bridging** - NSError กับ Swift throws
11. **Block/Closure Bridging** - Objective-C blocks กับ Swift closures
12. **Mixed-Language Setup** - การตั้งค่าโปรเจกต์แบบผสม
13. **Migration Strategies** - กลยุทธ์การย้ายโค้ดจาก Objective-C ไป Swift

---

## แหล่งข้อมูลเพิ่มเติม

- [Apple Documentation: Importing Objective-C into Swift](https://developer.apple.com/documentation/swift/importing-objective-c-into-swift)
- [Apple Documentation: Importing Swift into Objective-C](https://developer.apple.com/documentation/swift/importing-swift-into-objective-c)
- [Swift and Objective-C Interoperability WWDC Sessions](https://developer.apple.com/videos/frameworks/swift/)
- [Using Swift with Cocoa and Objective-C](https://developer.apple.com/library/archive/documentation/Swift/Conceptual/BuildingCocoaApps/)

---

*บทต่อไป: ตอนที่ 90 - CI/CD และ Deployment สำหรับ iOS*
