# Part 52: NSUserDefaults ใน Objective-C

## บทนำ (Introduction)

`NSUserDefaults` เป็น API ที่ใช้สำหรับเก็บข้อมูล preferences และ settings ขนาดเล็กของผู้ใช้ในรูปแบบ key-value pairs มันเหมาะสำหรับข้อมูลที่:
- ขนาดเล็ก (ไม่ควรเกิน 1 MB)
- ต้องการเข้าถึงบ่อยๆ
- เป็นค่า preferences เช่น settings, last state, user preferences

NSUserDefaults จัดเก็บข้อมูลไว้ใน `.plist` file ใน app's Library/Preferences directory

---

## 52.1 NSUserDefaults พื้นฐาน

### การเข้าถึง Standard User Defaults

```objc
// เข้าถึง shared instance
NSUserDefaults *defaults = [NSUserDefaults standardUserDefaults];

// หรือใช้ convenience API (iOS 11+)
[[NSUserDefaults standardUserDefaults] ...];
```

### ประเภทข้อมูลที่รองรับโดยตรง

```objc
NSUserDefaults *defaults = [NSUserDefaults standardUserDefaults];

// String
[defaults setObject:@"สมชาย ใจดี" forKey:@"userName"];

// Integer
[defaults setInteger:25 forKey:@"userAge"];

// Float
[defaults setFloat:1.5f forKey:@"textSizeMultiplier"];

// Double
[defaults setDouble:3.14159 forKey:@"pi"];

// Boolean
[defaults setBool:YES forKey:@"isLoggedIn"];
[defaults setBool:NO forKey:@"isDarkMode"];

// URL
[defaults setURL:[NSURL URLWithString:@"https://example.com"] forKey:@"lastURL"];

// Object (ต้องเป็น plist-compatible: NSString, NSNumber, NSArray, NSDictionary, NSData, NSDate)
[defaults setObject:@[@"item1", @"item2"] forKey:@"recentItems"];
[defaults setObject:[NSDate date] forKey:@"lastOpenDate"];
```

---

## 52.2 การอ่านค่า (Retrieving Values)

```objc
NSUserDefaults *defaults = [NSUserDefaults standardUserDefaults];

// String
NSString *name = [defaults stringForKey:@"userName"];
// คืน nil ถ้าไม่มี

// Integer
NSInteger age = [defaults integerForKey:@"userAge"];
// คืน 0 ถ้าไม่มี

// Float
float multiplier = [defaults floatForKey:@"textSizeMultiplier"];
// คืน 0.0 ถ้าไม่มี

// Double
double piValue = [defaults doubleForKey:@"pi"];

// Boolean
BOOL isLoggedIn = [defaults boolForKey:@"isLoggedIn"];
// คืน NO ถ้าไม่มี

// URL
NSURL *lastURL = [defaults URLForKey:@"lastURL"];

// Object
id object = [defaults objectForKey:@"recentItems"];
// ต้อง cast เอง
NSArray *recentItems = (NSArray *)[defaults objectForKey:@"recentItems"];

// NSData
NSData *data = [defaults dataForKey:@"encodedData"];

// NSArray
NSArray *array = [defaults arrayForKey:@"recentItems"];

// NSDictionary
NSDictionary *dict = [defaults dictionaryForKey:@"userInfo"];

// NSArray of strings
NSArray<NSString *> *stringArray = [defaults stringArrayForKey:@"tags"];
```

### ตรวจสอบว่ามี Key หรือไม่

```objc
NSUserDefaults *defaults = [NSUserDefaults standardUserDefaults];

// ตรวจสอบว่ามี key หรือไม่
id value = [defaults objectForKey:@"someKey"];
if (value != nil) {
    NSLog(@"Key exists: %@", value);
} else {
    NSLog(@"Key does not exist");
}

// สำหรับ BOOL - ต้องระวัง: boolForKey: คืน NO ทั้งเมื่อไม่มี key และเมื่อค่าเป็น NO
// ใช้ objectForKey: ก่อนแล้วค่อย check
BOOL hasSetDarkMode = [defaults objectForKey:@"isDarkMode"] != nil;
```

---

## 52.3 Supported Types (ประเภทที่รองรับ)

NSUserDefaults รองรับ "property list types":

| Type | Method |
|------|--------|
| NSString | `stringForKey:` / `setObject:forKey:` |
| NSNumber | `objectForKey:` |
| NSArray | `arrayForKey:` / `setObject:forKey:` |
| NSDictionary | `dictionaryForKey:` / `setObject:forKey:` |
| NSData | `dataForKey:` / `setObject:forKey:` |
| NSDate | `objectForKey:` / `setObject:forKey:` |
| NSURL | `URLForKey:` / `setURL:forKey:` |
| NSInteger | `integerForKey:` / `setInteger:forKey:` |
| float | `floatForKey:` / `setFloat:forKey:` |
| double | `doubleForKey:` / `setDouble:forKey:` |
| BOOL | `boolForKey:` / `setBool:forKey:` |

### ตัวอย่างการจัดเก็บ Array และ Dictionary

```objc
NSUserDefaults *defaults = [NSUserDefaults standardUserDefaults];

// บันทึก Array of strings
NSArray<NSString *> *recentSearches = @[@"iPhone", @"iPad", @"MacBook"];
[defaults setObject:recentSearches forKey:@"recentSearches"];

// อ่าน
NSArray<NSString *> *stored = [defaults stringArrayForKey:@"recentSearches"];

// บันทึก Dictionary
NSDictionary *userInfo = @{
    @"name": @"สมชาย",
    @"email": @"somchai@email.com",
    @"points": @(1500),
};
[defaults setObject:userInfo forKey:@"userInfo"];

// อ่าน
NSDictionary *storedInfo = [defaults dictionaryForKey:@"userInfo"];
NSString *name = storedInfo[@"name"];

// บันทึก Array of Dictionaries
NSArray *cartItems = @[
    @{@"id": @"001", @"name": @"iPhone", @"price": @(35000)},
    @{@"id": @"002", @"name": @"AirPods", @"price": @(7000)},
];
[defaults setObject:cartItems forKey:@"cartItems"];
```

---

## 52.4 Custom Objects ด้วย NSCoding

เพื่อเก็บ custom objects ต้องแปลงเป็น NSData ก่อน โดยใช้ NSCoding protocol

### NSCoding Protocol

```objc
// Person.h
@interface Person : NSObject <NSCoding, NSSecureCoding>
@property (nonatomic, copy) NSString *name;
@property (nonatomic, assign) NSInteger age;
@property (nonatomic, copy) NSString *email;
@end

// Person.m
@implementation Person

// NSSecureCoding
+ (BOOL)supportsSecureCoding {
    return YES;
}

// Encode (บันทึก)
- (void)encodeWithCoder:(NSCoder *)coder {
    [coder encodeObject:self.name forKey:@"name"];
    [coder encodeInteger:self.age forKey:@"age"];
    [coder encodeObject:self.email forKey:@"email"];
}

// Decode (โหลด)
- (instancetype)initWithCoder:(NSCoder *)coder {
    self = [super init];
    if (self) {
        _name = [coder decodeObjectOfClass:[NSString class] forKey:@"name"];
        _age = [coder decodeIntegerForKey:@"age"];
        _email = [coder decodeObjectOfClass:[NSString class] forKey:@"email"];
    }
    return self;
}

// Convenience initializer
- (instancetype)initWithName:(NSString *)name age:(NSInteger)age email:(NSString *)email {
    self = [super init];
    if (self) {
        _name = name;
        _age = age;
        _email = email;
    }
    return self;
}

- (NSString *)description {
    return [NSString stringWithFormat:@"Person{name=%@, age=%ld, email=%@}", 
            self.name, (long)self.age, self.email];
}

@end
```

### บันทึกและโหลด Custom Objects

```objc
- (void)saveCustomObject {
    Person *person = [[Person alloc] initWithName:@"สมชาย ใจดี" 
                                              age:30 
                                            email:@"somchai@email.com"];
    
    // แปลงเป็น NSData (iOS 11+ ใช้ NSKeyedArchiver)
    NSError *error;
    NSData *data = [NSKeyedArchiver archivedDataWithRootObject:person 
                                        requiringSecureCoding:YES 
                                                        error:&error];
    
    if (data && !error) {
        [[NSUserDefaults standardUserDefaults] setObject:data forKey:@"currentUser"];
        NSLog(@"✅ Saved person: %@", person);
    } else {
        NSLog(@"❌ Archive error: %@", error);
    }
}

- (Person *)loadCustomObject {
    NSData *data = [[NSUserDefaults standardUserDefaults] dataForKey:@"currentUser"];
    
    if (!data) {
        NSLog(@"No person data found");
        return nil;
    }
    
    NSError *error;
    Person *person = [NSKeyedUnarchiver unarchivedObjectOfClass:[Person class] 
                                                       fromData:data 
                                                          error:&error];
    
    if (person && !error) {
        NSLog(@"✅ Loaded person: %@", person);
        return person;
    } else {
        NSLog(@"❌ Unarchive error: %@", error);
        return nil;
    }
}
```

### บันทึก Array ของ Custom Objects

```objc
- (void)savePersonArray:(NSArray<Person *> *)persons {
    NSError *error;
    NSData *data = [NSKeyedArchiver archivedDataWithRootObject:persons 
                                        requiringSecureCoding:YES 
                                                        error:&error];
    
    if (data) {
        [[NSUserDefaults standardUserDefaults] setObject:data forKey:@"persons"];
    }
}

- (NSArray<Person *> *)loadPersonArray {
    NSData *data = [[NSUserDefaults standardUserDefaults] dataForKey:@"persons"];
    if (!data) return @[];
    
    NSError *error;
    NSArray<Person *> *persons = [NSKeyedUnarchiver unarchivedArrayOfObjectsOfClass:[Person class] 
                                                                            fromData:data 
                                                                               error:&error];
    return persons ?: @[];
}
```

---

## 52.5 NSUserDefaults Suites

Suites ใช้เมื่อต้องการ user defaults ที่แยกจาก standard defaults หรือแชร์ระหว่าง app extensions

```objc
// สร้าง custom suite
NSUserDefaults *customDefaults = [[NSUserDefaults alloc] initWithSuiteName:@"com.myapp.settings"];

[customDefaults setObject:@"custom value" forKey:@"key"];
NSString *value = [customDefaults stringForKey:@"key"];

// ลบ suite
[customDefaults removeSuiteNamed:@"com.myapp.settings"];
```

### App Groups User Defaults

ใช้แชร์ข้อมูลระหว่างแอปและ app extensions (เช่น Today Widget, Share Extension)

```objc
// ต้องเปิด App Groups ใน Capabilities ก่อน
// App Group Identifier: "group.com.yourcompany.yourapp"

// ใน Main App
NSUserDefaults *sharedDefaults = [[NSUserDefaults alloc] 
    initWithSuiteName:@"group.com.yourcompany.yourapp"];

[sharedDefaults setObject:@"shared data" forKey:@"sharedKey"];
[sharedDefaults setBool:YES forKey:@"isPremiumUser"];
[sharedDefaults setInteger:42 forKey:@"badgeCount"];

// ใน Widget Extension
NSUserDefaults *widgetDefaults = [[NSUserDefaults alloc] 
    initWithSuiteName:@"group.com.yourcompany.yourapp"];

NSString *data = [widgetDefaults stringForKey:@"sharedKey"];
BOOL isPremium = [widgetDefaults boolForKey:@"isPremiumUser"];
NSInteger badge = [widgetDefaults integerForKey:@"badgeCount"];
```

---

## 52.6 synchronize (deprecated แต่ควรรู้)

```objc
// เดิม: ต้องเรียก synchronize เพื่อบังคับ flush ไป disk
[[NSUserDefaults standardUserDefaults] synchronize];

// ปัจจุบัน: iOS จะ save อัตโนมัติในช่วงเวลาที่เหมาะสม
// synchronize ถูก deprecated ใน iOS 12+
// Apple ไม่แนะนำให้เรียกอีกต่อไป เพราะไม่จำเป็น

// อย่างไรก็ตาม synchronize ยังทำงานได้ (ไม่ crash)
// และบางครั้ง developers ยังเรียกเพื่อ backward compatibility
```

### ทำไม synchronize ถึง deprecated?

```
- iOS จัดการการบันทึก user defaults อัตโนมัติ
- ระบบ flush ข้อมูลอย่างสม่ำเสมอและเมื่อ app เข้าสู่ background
- การเรียก synchronize บ่อยๆ ทำให้ I/O มาก ส่งผลให้ battery drain
- ตั้งแต่ iOS 12 Apple รับประกันว่าข้อมูลจะถูกบันทึกก่อน app ถูก terminate
```

---

## 52.7 User Defaults vs Keychain

| Feature | NSUserDefaults | Keychain |
|---------|---------------|----------|
| ความปลอดภัย | ต่ำ (plain text) | สูง (encrypted) |
| ใช้สำหรับ | Preferences, settings | Passwords, tokens, keys |
| ขนาด | ไม่จำกัดมาก | จำกัด |
| App Groups | รองรับ | รองรับ (ต้องตั้งค่า) |
| iCloud sync | รองรับ (NSUbiquitousKeyValueStore) | รองรับ (kSecAttrSynchronizable) |
| Backup | รวมใน iTunes backup | เลือกได้ |
| After install | ล้างเมื่อ uninstall | เก็บอยู่แม้ uninstall |

```objc
// ❌ อย่าเก็บข้อมูลสำคัญใน NSUserDefaults
[[NSUserDefaults standardUserDefaults] setObject:@"my_secret_token" forKey:@"authToken"];  // ❌

// ✅ ใช้ Keychain สำหรับข้อมูลสำคัญ
// ใช้ KeychainWrapper หรือ Security framework
```

### Keychain Access ตัวอย่างอย่างง่าย

```objc
// เพิ่ม value เข้า Keychain
- (void)saveToKeychain:(NSString *)value forKey:(NSString *)key {
    NSData *data = [value dataUsingEncoding:NSUTF8StringEncoding];
    
    NSDictionary *query = @{
        (__bridge NSString *)kSecClass: (__bridge NSString *)kSecClassGenericPassword,
        (__bridge NSString *)kSecAttrAccount: key,
        (__bridge NSString *)kSecValueData: data,
        (__bridge NSString *)kSecAttrAccessible: 
            (__bridge NSString *)kSecAttrAccessibleWhenUnlocked,
    };
    
    // ลบของเก่าก่อน
    SecItemDelete((__bridge CFDictionaryRef)query);
    
    // เพิ่มใหม่
    OSStatus status = SecItemAdd((__bridge CFDictionaryRef)query, nil);
    
    if (status == errSecSuccess) {
        NSLog(@"✅ Saved to keychain");
    } else {
        NSLog(@"❌ Keychain error: %d", (int)status);
    }
}

// อ่านจาก Keychain
- (NSString *)loadFromKeychain:(NSString *)key {
    NSDictionary *query = @{
        (__bridge NSString *)kSecClass: (__bridge NSString *)kSecClassGenericPassword,
        (__bridge NSString *)kSecAttrAccount: key,
        (__bridge NSString *)kSecReturnData: @YES,
        (__bridge NSString *)kSecMatchLimit: (__bridge NSString *)kSecMatchLimitOne,
    };
    
    CFDataRef result = nil;
    OSStatus status = SecItemCopyMatching((__bridge CFDictionaryRef)query, 
                                          (CFTypeRef *)&result);
    
    if (status == errSecSuccess && result) {
        NSData *data = (__bridge_transfer NSData *)result;
        return [[NSString alloc] initWithData:data encoding:NSUTF8StringEncoding];
    }
    
    return nil;
}
```

---

## 52.8 Default Values ด้วย registerDefaults:

`registerDefaults:` ใช้ตั้งค่า default values เมื่อ key ยังไม่ถูกตั้งค่า (ไม่ overwrite ค่าที่มีอยู่)

```objc
// AppDelegate.m
- (BOOL)application:(UIApplication *)application 
    didFinishLaunchingWithOptions:(NSDictionary *)launchOptions {
    
    [self registerDefaultValues];
    return YES;
}

- (void)registerDefaultValues {
    NSDictionary *defaults = @{
        @"isDarkMode": @NO,
        @"fontSize": @(16.0),
        @"language": @"th",
        @"notificationsEnabled": @YES,
        @"soundEnabled": @YES,
        @"maxSearchResults": @(20),
        @"autoSave": @YES,
        @"lastSyncDate": [NSDate distantPast],
        @"theme": @"light",
        @"appLaunchCount": @(0),
    };
    
    [[NSUserDefaults standardUserDefaults] registerDefaults:defaults];
    
    NSLog(@"Default values registered");
}
```

### ประโยชน์ของ registerDefaults:

```objc
// ก่อน register defaults:
// [defaults boolForKey:@"notificationsEnabled"] → NO (ค่าเริ่มต้นของ BOOL)

// หลัง register defaults:
// [defaults boolForKey:@"notificationsEnabled"] → YES (ค่าที่เรากำหนด)

// สำคัญ: registerDefaults ไม่ทับค่าที่ user ตั้งไว้แล้ว!
[[NSUserDefaults standardUserDefaults] setBool:NO forKey:@"notificationsEnabled"];
// ตอนนี้จะคืน NO แม้ว่า default เป็น YES
```

---

## 52.9 การ Observe การเปลี่ยนแปลง User Defaults

### NSNotificationCenter

```objc
// Register observer
- (void)viewDidLoad {
    [super viewDidLoad];
    
    [[NSNotificationCenter defaultCenter] addObserver:self
                                             selector:@selector(userDefaultsChanged:)
                                                 name:NSUserDefaultsDidChangeNotification
                                               object:[NSUserDefaults standardUserDefaults]];
}

- (void)userDefaultsChanged:(NSNotification *)notification {
    // เรียกทุกครั้งที่ user defaults เปลี่ยน
    // notification.object = NSUserDefaults instance
    
    NSUserDefaults *defaults = notification.object;
    BOOL isDarkMode = [defaults boolForKey:@"isDarkMode"];
    
    dispatch_async(dispatch_get_main_queue(), ^{
        [self updateThemeForDarkMode:isDarkMode];
    });
}

- (void)dealloc {
    [[NSNotificationCenter defaultCenter] removeObserver:self];
}
```

### KVO (Key-Value Observing)

```objc
// Observe specific key
static void *ThemeContext = &ThemeContext;

- (void)startObservingTheme {
    [[NSUserDefaults standardUserDefaults] addObserver:self
                                            forKeyPath:@"isDarkMode"
                                               options:NSKeyValueObservingOptionNew | NSKeyValueObservingOptionOld
                                               context:ThemeContext];
}

- (void)observeValueForKeyPath:(NSString *)keyPath 
                      ofObject:(id)object
                        change:(NSDictionary *)change
                       context:(void *)context {
    if (context == ThemeContext) {
        BOOL newValue = [change[NSKeyValueChangeNewKey] boolValue];
        BOOL oldValue = [change[NSKeyValueChangeOldKey] boolValue];
        
        if (newValue != oldValue) {
            NSLog(@"Dark mode changed from %@ to %@", 
                  oldValue ? @"YES" : @"NO",
                  newValue ? @"YES" : @"NO");
            [self applyTheme:newValue];
        }
    } else {
        [super observeValueForKeyPath:keyPath 
                             ofObject:object 
                               change:change 
                              context:context];
    }
}

- (void)stopObservingTheme {
    [[NSUserDefaults standardUserDefaults] removeObserver:self 
                                               forKeyPath:@"isDarkMode" 
                                                  context:ThemeContext];
}
```

---

## 52.10 App Groups User Defaults

ใช้แชร์ user defaults ระหว่างแอปหลักและ extensions

### ขั้นตอนการตั้งค่า

1. ใน Xcode > Project Settings > Signing & Capabilities
2. กด "+" เพิ่ม App Groups capability
3. กำหนด App Group Identifier: `group.com.company.appname`
4. ทำเช่นเดียวกันใน Extension target

```objc
// UserDefaultsManager.h - Wrapper class
@interface UserDefaultsManager : NSObject

// Standard defaults (main app only)
+ (NSUserDefaults *)standard;

// Shared defaults (main app + extensions)  
+ (NSUserDefaults *)shared;

// Convenience methods
+ (void)setSharedValue:(id)value forKey:(NSString *)key;
+ (id)sharedValueForKey:(NSString *)key;

@end

// UserDefaultsManager.m
static NSString *const kAppGroupID = @"group.com.mycompany.myapp";

@implementation UserDefaultsManager

+ (NSUserDefaults *)standard {
    return [NSUserDefaults standardUserDefaults];
}

+ (NSUserDefaults *)shared {
    NSUserDefaults *defaults = [[NSUserDefaults alloc] initWithSuiteName:kAppGroupID];
    NSAssert(defaults != nil, @"App Group not configured: %@", kAppGroupID);
    return defaults;
}

+ (void)setSharedValue:(id)value forKey:(NSString *)key {
    [[self shared] setObject:value forKey:key];
}

+ (id)sharedValueForKey:(NSString *)key {
    return [[self shared] objectForKey:key];
}

@end
```

### ตัวอย่างการใช้งาน App Groups

```objc
// Main App - บันทึกข้อมูลที่แชร์กับ Widget
- (void)updateSharedData {
    NSUserDefaults *shared = [UserDefaultsManager shared];
    
    // ข้อมูลที่ Widget ต้องการแสดง
    [shared setObject:@"สมชาย ใจดี" forKey:@"userName"];
    [shared setInteger:1250 forKey:@"totalPoints"];
    [shared setObject:[NSDate date] forKey:@"lastUpdateDate"];
    
    NSArray *recentItems = @[@"Item 1", @"Item 2", @"Item 3"];
    [shared setObject:recentItems forKey:@"recentItems"];
    
    NSLog(@"Shared data updated");
}

// Widget Extension - อ่านข้อมูลที่แชร์
- (void)loadSharedData {
    NSUserDefaults *shared = [[NSUserDefaults alloc] 
        initWithSuiteName:@"group.com.mycompany.myapp"];
    
    NSString *userName = [shared stringForKey:@"userName"];
    NSInteger points = [shared integerForKey:@"totalPoints"];
    NSDate *lastUpdate = [shared objectForKey:@"lastUpdateDate"];
    NSArray *recentItems = [shared arrayForKey:@"recentItems"];
    
    NSLog(@"User: %@, Points: %ld", userName, (long)points);
}
```

---

## 52.11 ตัวอย่างครบถ้วน - Settings Manager

```objc
// SettingsManager.h
@interface SettingsManager : NSObject

// Singleton
+ (instancetype)shared;

// Theme
@property (nonatomic, assign) BOOL isDarkMode;
@property (nonatomic, copy) NSString *accentColor;

// Notifications
@property (nonatomic, assign) BOOL pushNotificationsEnabled;
@property (nonatomic, assign) BOOL emailNotificationsEnabled;
@property (nonatomic, assign) BOOL soundEnabled;

// Display
@property (nonatomic, assign) CGFloat fontSizeMultiplier;
@property (nonatomic, assign) NSInteger defaultLanguageIndex;

// Account
@property (nonatomic, copy, nullable) NSString *username;
@property (nonatomic, copy, nullable) NSString *email;
@property (nonatomic, assign) NSInteger totalLoginCount;
@property (nonatomic, strong, nullable) NSDate *lastLoginDate;

// Methods
- (void)resetToDefaults;
- (NSDictionary *)exportSettings;
- (void)importSettings:(NSDictionary *)settings;
- (void)incrementLoginCount;

@end

// SettingsManager.m
@implementation SettingsManager

// Keys
static NSString *const kDarkModeKey = @"settings.darkMode";
static NSString *const kAccentColorKey = @"settings.accentColor";
static NSString *const kPushNotifKey = @"settings.pushNotifications";
static NSString *const kEmailNotifKey = @"settings.emailNotifications";
static NSString *const kSoundKey = @"settings.sound";
static NSString *const kFontSizeKey = @"settings.fontSize";
static NSString *const kLanguageKey = @"settings.language";
static NSString *const kUsernameKey = @"settings.username";
static NSString *const kEmailKey = @"settings.email";
static NSString *const kLoginCountKey = @"settings.loginCount";
static NSString *const kLastLoginKey = @"settings.lastLogin";

+ (instancetype)shared {
    static SettingsManager *instance = nil;
    static dispatch_once_t onceToken;
    dispatch_once(&onceToken, ^{
        instance = [[SettingsManager alloc] init];
    });
    return instance;
}

- (instancetype)init {
    self = [super init];
    if (self) {
        [self registerDefaults];
    }
    return self;
}

- (void)registerDefaults {
    [[NSUserDefaults standardUserDefaults] registerDefaults:@{
        kDarkModeKey: @NO,
        kAccentColorKey: @"blue",
        kPushNotifKey: @YES,
        kEmailNotifKey: @YES,
        kSoundKey: @YES,
        kFontSizeKey: @(1.0),
        kLanguageKey: @(0),
        kLoginCountKey: @(0),
    }];
}

#pragma mark - Properties

- (BOOL)isDarkMode {
    return [[NSUserDefaults standardUserDefaults] boolForKey:kDarkModeKey];
}

- (void)setIsDarkMode:(BOOL)isDarkMode {
    [[NSUserDefaults standardUserDefaults] setBool:isDarkMode forKey:kDarkModeKey];
}

- (NSString *)accentColor {
    return [[NSUserDefaults standardUserDefaults] stringForKey:kAccentColorKey] ?: @"blue";
}

- (void)setAccentColor:(NSString *)accentColor {
    [[NSUserDefaults standardUserDefaults] setObject:accentColor forKey:kAccentColorKey];
}

- (BOOL)pushNotificationsEnabled {
    return [[NSUserDefaults standardUserDefaults] boolForKey:kPushNotifKey];
}

- (void)setPushNotificationsEnabled:(BOOL)enabled {
    [[NSUserDefaults standardUserDefaults] setBool:enabled forKey:kPushNotifKey];
}

- (BOOL)emailNotificationsEnabled {
    return [[NSUserDefaults standardUserDefaults] boolForKey:kEmailNotifKey];
}

- (void)setEmailNotificationsEnabled:(BOOL)enabled {
    [[NSUserDefaults standardUserDefaults] setBool:enabled forKey:kEmailNotifKey];
}

- (BOOL)soundEnabled {
    return [[NSUserDefaults standardUserDefaults] boolForKey:kSoundKey];
}

- (void)setSoundEnabled:(BOOL)enabled {
    [[NSUserDefaults standardUserDefaults] setBool:enabled forKey:kSoundKey];
}

- (CGFloat)fontSizeMultiplier {
    return [[NSUserDefaults standardUserDefaults] floatForKey:kFontSizeKey];
}

- (void)setFontSizeMultiplier:(CGFloat)multiplier {
    [[NSUserDefaults standardUserDefaults] setFloat:multiplier forKey:kFontSizeKey];
}

- (NSInteger)defaultLanguageIndex {
    return [[NSUserDefaults standardUserDefaults] integerForKey:kLanguageKey];
}

- (void)setDefaultLanguageIndex:(NSInteger)index {
    [[NSUserDefaults standardUserDefaults] setInteger:index forKey:kLanguageKey];
}

- (NSString *)username {
    return [[NSUserDefaults standardUserDefaults] stringForKey:kUsernameKey];
}

- (void)setUsername:(NSString *)username {
    [[NSUserDefaults standardUserDefaults] setObject:username forKey:kUsernameKey];
}

- (NSString *)email {
    return [[NSUserDefaults standardUserDefaults] stringForKey:kEmailKey];
}

- (void)setEmail:(NSString *)email {
    [[NSUserDefaults standardUserDefaults] setObject:email forKey:kEmailKey];
}

- (NSInteger)totalLoginCount {
    return [[NSUserDefaults standardUserDefaults] integerForKey:kLoginCountKey];
}

- (NSDate *)lastLoginDate {
    return [[NSUserDefaults standardUserDefaults] objectForKey:kLastLoginKey];
}

#pragma mark - Methods

- (void)incrementLoginCount {
    NSInteger current = self.totalLoginCount;
    [[NSUserDefaults standardUserDefaults] setInteger:current + 1 forKey:kLoginCountKey];
    [[NSUserDefaults standardUserDefaults] setObject:[NSDate date] forKey:kLastLoginKey];
}

- (void)resetToDefaults {
    NSUserDefaults *defaults = [NSUserDefaults standardUserDefaults];
    
    NSArray *keysToRemove = @[
        kDarkModeKey, kAccentColorKey, kPushNotifKey, kEmailNotifKey,
        kSoundKey, kFontSizeKey, kLanguageKey
    ];
    
    for (NSString *key in keysToRemove) {
        [defaults removeObjectForKey:key];
    }
    
    [self registerDefaults];
    NSLog(@"Settings reset to defaults");
}

- (NSDictionary *)exportSettings {
    return @{
        @"darkMode": @(self.isDarkMode),
        @"accentColor": self.accentColor ?: @"blue",
        @"pushNotifications": @(self.pushNotificationsEnabled),
        @"emailNotifications": @(self.emailNotificationsEnabled),
        @"sound": @(self.soundEnabled),
        @"fontSize": @(self.fontSizeMultiplier),
        @"language": @(self.defaultLanguageIndex),
    };
}

- (void)importSettings:(NSDictionary *)settings {
    if (settings[@"darkMode"]) {
        self.isDarkMode = [settings[@"darkMode"] boolValue];
    }
    if (settings[@"accentColor"]) {
        self.accentColor = settings[@"accentColor"];
    }
    if (settings[@"pushNotifications"]) {
        self.pushNotificationsEnabled = [settings[@"pushNotifications"] boolValue];
    }
    if (settings[@"emailNotifications"]) {
        self.emailNotificationsEnabled = [settings[@"emailNotifications"] boolValue];
    }
    if (settings[@"sound"]) {
        self.soundEnabled = [settings[@"sound"] boolValue];
    }
    if (settings[@"fontSize"]) {
        self.fontSizeMultiplier = [settings[@"fontSize"] floatValue];
    }
    if (settings[@"language"]) {
        self.defaultLanguageIndex = [settings[@"language"] integerValue];
    }
}

@end
```

### Settings View Controller

```objc
// SettingsViewController.m
@interface SettingsViewController () <UITableViewDataSource, UITableViewDelegate>
@property (nonatomic, strong) UITableView *tableView;
@end

@implementation SettingsViewController

- (void)viewDidLoad {
    [super viewDidLoad];
    self.title = @"ตั้งค่า";
    
    self.tableView = [[UITableView alloc] initWithFrame:CGRectZero style:UITableViewStyleInsetGrouped];
    self.tableView.translatesAutoresizingMaskIntoConstraints = NO;
    self.tableView.dataSource = self;
    self.tableView.delegate = self;
    [self.view addSubview:self.tableView];
    
    [NSLayoutConstraint activateConstraints:@[
        [self.tableView.topAnchor constraintEqualToAnchor:self.view.topAnchor],
        [self.tableView.leadingAnchor constraintEqualToAnchor:self.view.leadingAnchor],
        [self.tableView.trailingAnchor constraintEqualToAnchor:self.view.trailingAnchor],
        [self.tableView.bottomAnchor constraintEqualToAnchor:self.view.bottomAnchor],
    ]];
    
    // Observe settings changes
    [[NSNotificationCenter defaultCenter] addObserver:self
                                             selector:@selector(settingsChanged:)
                                                 name:NSUserDefaultsDidChangeNotification
                                               object:nil];
}

- (void)settingsChanged:(NSNotification *)notification {
    dispatch_async(dispatch_get_main_queue(), ^{
        [self.tableView reloadData];
    });
}

- (NSInteger)numberOfSectionsInTableView:(UITableView *)tableView {
    return 3;  // Appearance, Notifications, Account
}

- (NSString *)tableView:(UITableView *)tableView titleForHeaderInSection:(NSInteger)section {
    switch (section) {
        case 0: return @"การแสดงผล";
        case 1: return @"การแจ้งเตือน";
        case 2: return @"บัญชีผู้ใช้";
        default: return nil;
    }
}

- (NSInteger)tableView:(UITableView *)tableView numberOfRowsInSection:(NSInteger)section {
    switch (section) {
        case 0: return 2;  // Dark Mode, Font Size
        case 1: return 2;  // Push, Sound
        case 2: return 2;  // Username, Reset
        default: return 0;
    }
}

- (UITableViewCell *)tableView:(UITableView *)tableView 
         cellForRowAtIndexPath:(NSIndexPath *)indexPath {
    
    SettingsManager *settings = [SettingsManager shared];
    
    if (indexPath.section == 0) {
        if (indexPath.row == 0) {
            return [self toggleCellWithTitle:@"โหมดมืด"
                                      value:settings.isDarkMode
                                   selector:@selector(darkModeToggled:)];
        } else {
            return [self sliderCellWithTitle:@"ขนาดตัวอักษร"
                                      value:settings.fontSizeMultiplier
                                   selector:@selector(fontSizeChanged:)];
        }
    } else if (indexPath.section == 1) {
        if (indexPath.row == 0) {
            return [self toggleCellWithTitle:@"การแจ้งเตือน"
                                      value:settings.pushNotificationsEnabled
                                   selector:@selector(notificationsToggled:)];
        } else {
            return [self toggleCellWithTitle:@"เสียง"
                                      value:settings.soundEnabled
                                   selector:@selector(soundToggled:)];
        }
    } else {
        if (indexPath.row == 0) {
            UITableViewCell *cell = [[UITableViewCell alloc] 
                initWithStyle:UITableViewCellStyleValue1 
                reuseIdentifier:@"InfoCell"];
            cell.textLabel.text = @"ชื่อผู้ใช้";
            cell.detailTextLabel.text = settings.username ?: @"ไม่ได้ตั้งค่า";
            cell.accessoryType = UITableViewCellAccessoryDisclosureIndicator;
            return cell;
        } else {
            UITableViewCell *cell = [[UITableViewCell alloc] 
                initWithStyle:UITableViewCellStyleDefault 
                reuseIdentifier:@"ResetCell"];
            cell.textLabel.text = @"รีเซ็ตค่าเริ่มต้น";
            cell.textLabel.textColor = [UIColor systemRedColor];
            return cell;
        }
    }
}

- (void)tableView:(UITableView *)tableView didSelectRowAtIndexPath:(NSIndexPath *)indexPath {
    [tableView deselectRowAtIndexPath:indexPath animated:YES];
    
    if (indexPath.section == 2) {
        if (indexPath.row == 0) {
            [self editUsername];
        } else {
            [self confirmResetSettings];
        }
    }
}

#pragma mark - Toggle Actions

- (void)darkModeToggled:(UISwitch *)sender {
    [SettingsManager shared].isDarkMode = sender.isOn;
}

- (void)notificationsToggled:(UISwitch *)sender {
    [SettingsManager shared].pushNotificationsEnabled = sender.isOn;
}

- (void)soundToggled:(UISwitch *)sender {
    [SettingsManager shared].soundEnabled = sender.isOn;
}

- (void)fontSizeChanged:(UISlider *)sender {
    [SettingsManager shared].fontSizeMultiplier = sender.value;
}

#pragma mark - Alert Dialogs

- (void)editUsername {
    UIAlertController *alert = [UIAlertController 
        alertControllerWithTitle:@"แก้ไขชื่อผู้ใช้"
        message:nil
        preferredStyle:UIAlertControllerStyleAlert];
    
    [alert addTextFieldWithConfigurationHandler:^(UITextField *textField) {
        textField.placeholder = @"ชื่อผู้ใช้";
        textField.text = [SettingsManager shared].username;
    }];
    
    UIAlertAction *save = [UIAlertAction actionWithTitle:@"บันทึก"
                                                   style:UIAlertActionStyleDefault
                                                 handler:^(UIAlertAction *action) {
        NSString *newName = alert.textFields.firstObject.text;
        if (newName.length > 0) {
            [SettingsManager shared].username = newName;
            [self.tableView reloadData];
        }
    }];
    
    [alert addAction:save];
    [alert addAction:[UIAlertAction actionWithTitle:@"ยกเลิก"
                                              style:UIAlertActionStyleCancel
                                            handler:nil]];
    
    [self presentViewController:alert animated:YES completion:nil];
}

- (void)confirmResetSettings {
    UIAlertController *alert = [UIAlertController 
        alertControllerWithTitle:@"รีเซ็ตการตั้งค่า?"
        message:@"การตั้งค่าทั้งหมดจะกลับเป็นค่าเริ่มต้น"
        preferredStyle:UIAlertControllerStyleAlert];
    
    [alert addAction:[UIAlertAction actionWithTitle:@"รีเซ็ต"
                                              style:UIAlertActionStyleDestructive
                                            handler:^(UIAlertAction *action) {
        [[SettingsManager shared] resetToDefaults];
    }]];
    
    [alert addAction:[UIAlertAction actionWithTitle:@"ยกเลิก"
                                              style:UIAlertActionStyleCancel
                                            handler:nil]];
    
    [self presentViewController:alert animated:YES completion:nil];
}

#pragma mark - Helper Methods

- (UITableViewCell *)toggleCellWithTitle:(NSString *)title 
                                   value:(BOOL)value 
                                selector:(SEL)selector {
    UITableViewCell *cell = [[UITableViewCell alloc] 
        initWithStyle:UITableViewCellStyleDefault 
        reuseIdentifier:@"ToggleCell"];
    
    cell.textLabel.text = title;
    cell.selectionStyle = UITableViewCellSelectionStyleNone;
    
    UISwitch *toggle = [[UISwitch alloc] init];
    toggle.on = value;
    [toggle addTarget:self action:selector forControlEvents:UIControlEventValueChanged];
    cell.accessoryView = toggle;
    
    return cell;
}

- (UITableViewCell *)sliderCellWithTitle:(NSString *)title 
                                   value:(CGFloat)value 
                                selector:(SEL)selector {
    UITableViewCell *cell = [[UITableViewCell alloc] 
        initWithStyle:UITableViewCellStyleDefault 
        reuseIdentifier:@"SliderCell"];
    
    cell.textLabel.text = title;
    cell.selectionStyle = UITableViewCellSelectionStyleNone;
    
    UISlider *slider = [[UISlider alloc] init];
    slider.minimumValue = 0.5f;
    slider.maximumValue = 2.0f;
    slider.value = value;
    slider.frame = CGRectMake(0, 0, 150, 31);
    [slider addTarget:self action:selector forControlEvents:UIControlEventValueChanged];
    cell.accessoryView = slider;
    
    return cell;
}

- (void)dealloc {
    [[NSNotificationCenter defaultCenter] removeObserver:self];
}

@end
```

---

## 52.12 UserDefaults Utility Class

```objc
// UserDefaultsKeys.h - Constants
@interface UserDefaultsKeys : NSObject

// User
extern NSString * const kUserIsLoggedIn;
extern NSString * const kUserID;
extern NSString * const kUserName;
extern NSString * const kUserEmail;
extern NSString * const kUserToken;  // ❌ ใช้ Keychain แทน!

// App State
extern NSString * const kAppLaunchCount;
extern NSString * const kLastLaunchDate;
extern NSString * const kAppVersion;
extern NSString * const kHasSeenOnboarding;

// Settings
extern NSString * const kSettingsDarkMode;
extern NSString * const kSettingsLanguage;
extern NSString * const kSettingsFontSize;

// Cache
extern NSString * const kCacheLastFetchDate;

@end

// UserDefaultsKeys.m
@implementation UserDefaultsKeys

NSString * const kUserIsLoggedIn = @"user.isLoggedIn";
NSString * const kUserID = @"user.id";
NSString * const kUserName = @"user.name";
NSString * const kUserEmail = @"user.email";

NSString * const kAppLaunchCount = @"app.launchCount";
NSString * const kLastLaunchDate = @"app.lastLaunchDate";
NSString * const kAppVersion = @"app.version";
NSString * const kHasSeenOnboarding = @"app.hasSeenOnboarding";

NSString * const kSettingsDarkMode = @"settings.darkMode";
NSString * const kSettingsLanguage = @"settings.language";
NSString * const kSettingsFontSize = @"settings.fontSize";

NSString * const kCacheLastFetchDate = @"cache.lastFetchDate";

@end
```

### AppLaunchManager

```objc
// AppLaunchManager.h
@interface AppLaunchManager : NSObject
+ (instancetype)shared;
@property (nonatomic, readonly) NSInteger launchCount;
@property (nonatomic, readonly, nullable) NSDate *lastLaunchDate;
@property (nonatomic, readonly) BOOL isFirstLaunch;
@property (nonatomic, readonly) BOOL hasSeenOnboarding;
@property (nonatomic, getter=hasSeenOnboarding) BOOL seenOnboarding;
- (void)recordLaunch;
@end

// AppLaunchManager.m
@implementation AppLaunchManager

+ (instancetype)shared {
    static AppLaunchManager *instance = nil;
    static dispatch_once_t onceToken;
    dispatch_once(&onceToken, ^{
        instance = [[AppLaunchManager alloc] init];
    });
    return instance;
}

- (NSInteger)launchCount {
    return [[NSUserDefaults standardUserDefaults] integerForKey:kAppLaunchCount];
}

- (NSDate *)lastLaunchDate {
    return [[NSUserDefaults standardUserDefaults] objectForKey:kLastLaunchDate];
}

- (BOOL)isFirstLaunch {
    return self.launchCount == 0;
}

- (BOOL)hasSeenOnboarding {
    return [[NSUserDefaults standardUserDefaults] boolForKey:kHasSeenOnboarding];
}

- (void)setSeenOnboarding:(BOOL)seen {
    [[NSUserDefaults standardUserDefaults] setBool:seen forKey:kHasSeenOnboarding];
}

- (void)recordLaunch {
    NSUserDefaults *defaults = [NSUserDefaults standardUserDefaults];
    
    NSInteger count = [defaults integerForKey:kAppLaunchCount];
    [defaults setInteger:count + 1 forKey:kAppLaunchCount];
    [defaults setObject:[NSDate date] forKey:kLastLaunchDate];
    
    // ตรวจสอบ app version
    NSString *currentVersion = [[NSBundle mainBundle] objectForInfoDictionaryKey:@"CFBundleShortVersionString"];
    NSString *savedVersion = [defaults stringForKey:kAppVersion];
    
    if (![currentVersion isEqualToString:savedVersion]) {
        [defaults setObject:currentVersion forKey:kAppVersion];
        
        if (savedVersion) {
            NSLog(@"App updated from %@ to %@", savedVersion, currentVersion);
            [[NSNotificationCenter defaultCenter] postNotificationName:@"AppDidUpdateNotification" 
                                                                object:nil 
                                                              userInfo:@{
                @"oldVersion": savedVersion,
                @"newVersion": currentVersion
            }];
        }
    }
}

@end
```

---

## 52.13 การลบข้อมูล (Removing Values)

```objc
NSUserDefaults *defaults = [NSUserDefaults standardUserDefaults];

// ลบ single key
[defaults removeObjectForKey:@"userName"];

// ลบทุก key ของ app (ควรระวัง!)
- (void)clearAllUserDefaults {
    NSString *appDomain = [[NSBundle mainBundle] bundleIdentifier];
    [[NSUserDefaults standardUserDefaults] removePersistentDomainForName:appDomain];
    NSLog(@"All user defaults cleared");
}

// ลบเฉพาะบาง keys (ปลอดภัยกว่า)
- (void)clearUserSession {
    NSUserDefaults *defaults = [NSUserDefaults standardUserDefaults];
    NSArray *sessionKeys = @[
        @"user.isLoggedIn",
        @"user.id", 
        @"user.name",
        @"user.email",
    ];
    
    for (NSString *key in sessionKeys) {
        [defaults removeObjectForKey:key];
    }
}
```

---

## 52.14 Type Safety Helper Category

```objc
// NSUserDefaults+TypeSafe.h
@interface NSUserDefaults (TypeSafe)

// Safe getters with default values
- (NSString *)safeStringForKey:(NSString *)key defaultValue:(NSString *)defaultValue;
- (NSInteger)safeIntegerForKey:(NSString *)key defaultValue:(NSInteger)defaultValue;
- (BOOL)safeBoolForKey:(NSString *)key defaultValue:(BOOL)defaultValue;
- (double)safeDoubleForKey:(NSString *)key defaultValue:(double)defaultValue;
- (NSDate *)safeDateForKey:(NSString *)key defaultValue:(nullable NSDate *)defaultValue;
- (NSArray *)safeArrayForKey:(NSString *)key defaultValue:(NSArray *)defaultValue;

@end

// NSUserDefaults+TypeSafe.m
@implementation NSUserDefaults (TypeSafe)

- (NSString *)safeStringForKey:(NSString *)key defaultValue:(NSString *)defaultValue {
    NSString *value = [self stringForKey:key];
    return value ?: defaultValue;
}

- (NSInteger)safeIntegerForKey:(NSString *)key defaultValue:(NSInteger)defaultValue {
    if ([self objectForKey:key] == nil) return defaultValue;
    return [self integerForKey:key];
}

- (BOOL)safeBoolForKey:(NSString *)key defaultValue:(BOOL)defaultValue {
    if ([self objectForKey:key] == nil) return defaultValue;
    return [self boolForKey:key];
}

- (double)safeDoubleForKey:(NSString *)key defaultValue:(double)defaultValue {
    if ([self objectForKey:key] == nil) return defaultValue;
    return [self doubleForKey:key];
}

- (NSDate *)safeDateForKey:(NSString *)key defaultValue:(nullable NSDate *)defaultValue {
    NSDate *value = [self objectForKey:key];
    return value ?: defaultValue;
}

- (NSArray *)safeArrayForKey:(NSString *)key defaultValue:(NSArray *)defaultValue {
    NSArray *value = [self arrayForKey:key];
    return value ?: defaultValue;
}

@end

// การใช้งาน
NSString *name = [[NSUserDefaults standardUserDefaults] 
    safeStringForKey:@"userName" 
    defaultValue:@"ไม่ระบุชื่อ"];

BOOL isDark = [[NSUserDefaults standardUserDefaults] 
    safeBoolForKey:@"isDarkMode" 
    defaultValue:NO];
```

---

## 52.15 แบบฝึกหัด (Practice Exercises)

### แบบฝึกหัดที่ 1: Onboarding Flow

สร้าง onboarding flow ที่แสดงแค่ครั้งแรกที่เปิดแอป:

```objc
@implementation AppDelegate

- (BOOL)application:(UIApplication *)application 
    didFinishLaunchingWithOptions:(NSDictionary *)launchOptions {
    
    [self setupWindow];
    return YES;
}

- (void)setupWindow {
    UIViewController *rootVC;
    
    BOOL hasSeenOnboarding = [[NSUserDefaults standardUserDefaults] 
        boolForKey:@"hasSeenOnboarding"];
    
    if (hasSeenOnboarding) {
        rootVC = [[MainViewController alloc] init];
    } else {
        OnboardingViewController *onboarding = [[OnboardingViewController alloc] init];
        onboarding.completionHandler = ^{
            [[NSUserDefaults standardUserDefaults] setBool:YES forKey:@"hasSeenOnboarding"];
            // Transition to main
        };
        rootVC = onboarding;
    }
    
    self.window.rootViewController = rootVC;
}

@end
```

### แบบฝึกหัดที่ 2: Recent Searches

สร้าง recent searches feature:

```objc
@implementation RecentSearchManager

static NSString * const kRecentSearchesKey = @"recentSearches";
static NSInteger const kMaxRecentSearches = 10;

+ (NSArray<NSString *> *)recentSearches {
    return [[NSUserDefaults standardUserDefaults] stringArrayForKey:kRecentSearchesKey] ?: @[];
}

+ (void)addSearchTerm:(NSString *)term {
    if (term.length == 0) return;
    
    NSMutableArray *searches = [[self recentSearches] mutableCopy];
    
    // ลบถ้ามีอยู่แล้ว (เพื่อย้ายขึ้นมาด้านบน)
    [searches removeObject:term];
    
    // เพิ่มที่ต้น
    [searches insertObject:term atIndex:0];
    
    // จำกัดจำนวน
    if (searches.count > kMaxRecentSearches) {
        searches = [[searches subarrayWithRange:NSMakeRange(0, kMaxRecentSearches)] mutableCopy];
    }
    
    [[NSUserDefaults standardUserDefaults] setObject:searches forKey:kRecentSearchesKey];
}

+ (void)removeSearchTerm:(NSString *)term {
    NSMutableArray *searches = [[self recentSearches] mutableCopy];
    [searches removeObject:term];
    [[NSUserDefaults standardUserDefaults] setObject:searches forKey:kRecentSearchesKey];
}

+ (void)clearAllSearches {
    [[NSUserDefaults standardUserDefaults] removeObjectForKey:kRecentSearchesKey];
}

@end
```

### แบบฝึกหัดที่ 3: User Preferences with Custom Object

สร้าง `UserProfile` class ที่ implement NSCoding และบันทึก/โหลดด้วย NSUserDefaults:

```objc
// UserProfile.h
@interface UserProfile : NSObject <NSSecureCoding>
@property (nonatomic, copy) NSString *userID;
@property (nonatomic, copy) NSString *displayName;
@property (nonatomic, copy) NSString *bio;
@property (nonatomic, assign) NSInteger followersCount;
@property (nonatomic, assign) BOOL isVerified;
@property (nonatomic, copy) NSArray<NSString *> *interests;
@property (nonatomic, copy) NSDate *joinDate;
+ (nullable instancetype)loadCurrentProfile;
- (void)saveAsCurrentProfile;
@end

// UserProfile.m
@implementation UserProfile

+ (BOOL)supportsSecureCoding { return YES; }

- (void)encodeWithCoder:(NSCoder *)coder {
    [coder encodeObject:self.userID forKey:@"userID"];
    [coder encodeObject:self.displayName forKey:@"displayName"];
    [coder encodeObject:self.bio forKey:@"bio"];
    [coder encodeInteger:self.followersCount forKey:@"followersCount"];
    [coder encodeBool:self.isVerified forKey:@"isVerified"];
    [coder encodeObject:self.interests forKey:@"interests"];
    [coder encodeObject:self.joinDate forKey:@"joinDate"];
}

- (instancetype)initWithCoder:(NSCoder *)coder {
    self = [super init];
    if (self) {
        _userID = [coder decodeObjectOfClass:[NSString class] forKey:@"userID"];
        _displayName = [coder decodeObjectOfClass:[NSString class] forKey:@"displayName"];
        _bio = [coder decodeObjectOfClass:[NSString class] forKey:@"bio"];
        _followersCount = [coder decodeIntegerForKey:@"followersCount"];
        _isVerified = [coder decodeBoolForKey:@"isVerified"];
        _interests = [coder decodeObjectOfClasses:[NSSet setWithObjects:[NSArray class], [NSString class], nil] 
                                           forKey:@"interests"];
        _joinDate = [coder decodeObjectOfClass:[NSDate class] forKey:@"joinDate"];
    }
    return self;
}

+ (nullable instancetype)loadCurrentProfile {
    NSData *data = [[NSUserDefaults standardUserDefaults] dataForKey:@"currentUserProfile"];
    if (!data) return nil;
    
    NSError *error;
    UserProfile *profile = [NSKeyedUnarchiver unarchivedObjectOfClass:[UserProfile class] 
                                                             fromData:data 
                                                                error:&error];
    if (error) {
        NSLog(@"Error loading profile: %@", error);
        return nil;
    }
    return profile;
}

- (void)saveAsCurrentProfile {
    NSError *error;
    NSData *data = [NSKeyedArchiver archivedDataWithRootObject:self 
                                        requiringSecureCoding:YES 
                                                        error:&error];
    if (data && !error) {
        [[NSUserDefaults standardUserDefaults] setObject:data forKey:@"currentUserProfile"];
    } else {
        NSLog(@"Error saving profile: %@", error);
    }
}

@end
```

---

## สรุป (Summary)

- **NSUserDefaults** เหมาะสำหรับเก็บ preferences, settings ขนาดเล็ก
- รองรับ property list types: NSString, NSNumber, NSArray, NSDictionary, NSData, NSDate, NSURL
- ใช้ `registerDefaults:` เพื่อตั้ง default values (ไม่ทับค่าที่มีอยู่)
- Custom objects ต้องใช้ **NSCoding** แปลงเป็น NSData ก่อนบันทึก
- **NSSecureCoding** แนะนำมากกว่า NSCoding เพื่อความปลอดภัย
- `synchronize` deprecated ใน iOS 12 ไม่จำเป็นต้องเรียกอีกต่อไป
- **NSUserDefaults Suites** ใช้แยก defaults หรือแชร์ระหว่าง extensions
- **App Groups** ใช้แชร์ข้อมูลระหว่างแอปหลักและ extensions
- **อย่า** เก็บ passwords, tokens, sensitive data ใน NSUserDefaults - ใช้ **Keychain** แทน
- ใช้ **NSUserDefaultsDidChangeNotification** หรือ **KVO** เพื่อ observe การเปลี่ยนแปลง
- ไม่ควรเก็บข้อมูลขนาดใหญ่ (ควรน้อยกว่า 1 MB) ใช้ Core Data หรือ Files API แทน
- ใช้ **constants** สำหรับ keys เพื่อป้องกัน typos และง่ายต่อการ refactor
