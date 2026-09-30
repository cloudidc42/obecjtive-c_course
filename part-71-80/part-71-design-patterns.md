# Part 71: Design Patterns ใน Objective-C

## บทนำ

Design Patterns คือแนวทางการแก้ปัญหาที่ใช้กันบ่อยในการออกแบบซอฟต์แวร์ เป็น "สูตรสำเร็จ" ที่ผ่านการพิสูจน์มาแล้วว่าใช้งานได้จริง แต่ไม่ใช่โค้ดที่ copy-paste ได้เลย ต้องปรับให้เข้ากับบริบทของแต่ละโปรเจกต์

Design Patterns แบ่งออกเป็น 3 หมวดหลัก:
1. **Creational Patterns** - เกี่ยวกับการสร้าง Object
2. **Structural Patterns** - เกี่ยวกับโครงสร้างของ Object
3. **Behavioral Patterns** - เกี่ยวกับพฤติกรรมและการสื่อสารระหว่าง Object

---

# หมวดที่ 1: Creational Patterns

## 71.1 Singleton Pattern

**แนวคิด**: รับประกันว่า class มี instance เพียงตัวเดียวตลอดชีวิตของแอป และมี global point of access

**เมื่อใช้**: Database connection, Configuration manager, Logger, Cache

```objc
// DatabaseManager.h
@interface DatabaseManager : NSObject

@property (nonatomic, strong, readonly) NSString *dbPath;

+ (instancetype)sharedManager;

// ป้องกันการสร้าง instance โดยตรง
+ (instancetype)new NS_UNAVAILABLE;
- (instancetype)init NS_UNAVAILABLE;

- (void)executeQuery:(NSString *)query;
- (NSArray *)fetchData:(NSString *)query;

@end

// DatabaseManager.m
@implementation DatabaseManager

+ (instancetype)sharedManager {
    static DatabaseManager *instance = nil;
    static dispatch_once_t onceToken;
    dispatch_once(&onceToken, ^{
        instance = [[self alloc] initPrivate];
    });
    return instance;
}

- (instancetype)initPrivate {
    self = [super init];
    if (self) {
        _dbPath = [[NSSearchPathForDirectoriesInDomains(NSDocumentDirectory, NSUserDomainMask, YES) firstObject]
                   stringByAppendingPathComponent:@"app.db"];
        [self setupDatabase];
    }
    return self;
}

- (void)setupDatabase {
    NSLog(@"Database initialized at: %@", self.dbPath);
}

- (void)executeQuery:(NSString *)query {
    NSLog(@"Executing: %@", query);
}

- (NSArray *)fetchData:(NSString *)query {
    NSLog(@"Fetching with: %@", query);
    return @[];
}

@end

// การใช้งาน
DatabaseManager *db = [DatabaseManager sharedManager];
[db executeQuery:@"INSERT INTO users VALUES (...)"];
```

**ข้อควรระวัง**:
- Singleton ทำให้ test ยากขึ้น (ต้องใช้ dependency injection แทน)
- ระวัง thread safety เมื่อมีการแก้ไข shared state
- ควรใช้ `dispatch_once` เพื่อ thread-safe initialization

---

## 71.2 Factory Method Pattern

**แนวคิด**: กำหนด interface สำหรับสร้าง object แต่ให้ subclass ตัดสินใจว่าจะสร้าง class ไหน

**เมื่อใช้**: ต้องการสร้าง objects ที่แตกต่างกันตาม input, Plugin architecture

```objc
// Abstract Payment Processor
@interface PaymentProcessor : NSObject

@property (nonatomic, strong) NSString *processorName;

// Factory Method - subclass จะ override
+ (instancetype)createProcessor;

- (void)processPayment:(double)amount currency:(NSString *)currency;
- (BOOL)validateCard:(NSString *)cardNumber;

@end

@implementation PaymentProcessor

+ (instancetype)createProcessor {
    return [[self alloc] init];
}

- (void)processPayment:(double)amount currency:(NSString *)currency {
    // Abstract - subclass ต้อง override
    [NSException raise:NSInternalInconsistencyException
                format:@"Subclass must implement processPayment:currency:"];
}

- (BOOL)validateCard:(NSString *)cardNumber {
    return NO; // Default
}

@end

// Concrete Implementations
@interface StripeProcessor : PaymentProcessor
@end

@implementation StripeProcessor

+ (instancetype)createProcessor {
    StripeProcessor *processor = [[self alloc] init];
    processor.processorName = @"Stripe";
    return processor;
}

- (void)processPayment:(double)amount currency:(NSString *)currency {
    NSLog(@"Stripe: Processing %.2f %@", amount, currency);
    // Stripe-specific logic
}

- (BOOL)validateCard:(NSString *)cardNumber {
    // Luhn algorithm implementation
    return cardNumber.length >= 13 && cardNumber.length <= 19;
}

@end

@interface PayPalProcessor : PaymentProcessor
@end

@implementation PayPalProcessor

+ (instancetype)createProcessor {
    PayPalProcessor *processor = [[self alloc] init];
    processor.processorName = @"PayPal";
    return processor;
}

- (void)processPayment:(double)amount currency:(NSString *)currency {
    NSLog(@"PayPal: Processing %.2f %@", amount, currency);
    // PayPal-specific logic
}

@end

// Factory
@interface PaymentProcessorFactory : NSObject

typedef NS_ENUM(NSInteger, PaymentMethod) {
    PaymentMethodStripe,
    PaymentMethodPayPal,
    PaymentMethodApplePay
};

+ (PaymentProcessor *)processorForMethod:(PaymentMethod)method;

@end

@implementation PaymentProcessorFactory

+ (PaymentProcessor *)processorForMethod:(PaymentMethod)method {
    switch (method) {
        case PaymentMethodStripe:
            return [StripeProcessor createProcessor];
        case PaymentMethodPayPal:
            return [PayPalProcessor createProcessor];
        case PaymentMethodApplePay:
            return [StripeProcessor createProcessor]; // ใช้ Stripe สำหรับ Apple Pay
        default:
            return nil;
    }
}

@end

// การใช้งาน
PaymentProcessor *processor = [PaymentProcessorFactory processorForMethod:PaymentMethodStripe];
[processor processPayment:99.99 currency:@"USD"];
```

---

## 71.3 Abstract Factory Pattern

**แนวคิด**: จัดกลุ่มของ Factories ที่เกี่ยวข้องกัน โดยไม่ระบุ concrete classes

**เมื่อใช้**: UI Theming (Dark/Light mode), Cross-platform components

```objc
// Abstract UI Factory
@protocol UIComponentFactory <NSObject>

- (UIButton *)createPrimaryButton;
- (UILabel *)createTitleLabel;
- (UITextField *)createInputField;
- (UIColor *)createBackgroundColor;

@end

// Light Theme Factory
@interface LightThemeFactory : NSObject <UIComponentFactory>
@end

@implementation LightThemeFactory

- (UIButton *)createPrimaryButton {
    UIButton *button = [UIButton buttonWithType:UIButtonTypeSystem];
    [button setBackgroundColor:[UIColor systemBlueColor]];
    [button setTitleColor:[UIColor whiteColor] forState:UIControlStateNormal];
    button.layer.cornerRadius = 8;
    return button;
}

- (UILabel *)createTitleLabel {
    UILabel *label = [[UILabel alloc] init];
    label.textColor = [UIColor blackColor];
    label.font = [UIFont boldSystemFontOfSize:24];
    return label;
}

- (UITextField *)createInputField {
    UITextField *field = [[UITextField alloc] init];
    field.backgroundColor = [UIColor systemGray6Color];
    field.textColor = [UIColor blackColor];
    field.layer.cornerRadius = 8;
    return field;
}

- (UIColor *)createBackgroundColor {
    return [UIColor whiteColor];
}

@end

// Dark Theme Factory
@interface DarkThemeFactory : NSObject <UIComponentFactory>
@end

@implementation DarkThemeFactory

- (UIButton *)createPrimaryButton {
    UIButton *button = [UIButton buttonWithType:UIButtonTypeSystem];
    [button setBackgroundColor:[UIColor systemIndigoColor]];
    [button setTitleColor:[UIColor whiteColor] forState:UIControlStateNormal];
    button.layer.cornerRadius = 8;
    return button;
}

- (UILabel *)createTitleLabel {
    UILabel *label = [[UILabel alloc] init];
    label.textColor = [UIColor whiteColor];
    label.font = [UIFont boldSystemFontOfSize:24];
    return label;
}

- (UITextField *)createInputField {
    UITextField *field = [[UITextField alloc] init];
    field.backgroundColor = [UIColor systemGray3Color];
    field.textColor = [UIColor whiteColor];
    field.layer.cornerRadius = 8;
    return field;
}

- (UIColor *)createBackgroundColor {
    return [UIColor systemBackgroundColor];
}

@end

// Client code
@interface LoginViewController : UIViewController

@property (nonatomic, strong) id<UIComponentFactory> uiFactory;

@end

@implementation LoginViewController

- (void)setupWithFactory:(id<UIComponentFactory>)factory {
    self.uiFactory = factory;
    self.view.backgroundColor = [factory createBackgroundColor];
    
    UILabel *title = [factory createTitleLabel];
    title.text = @"เข้าสู่ระบบ";
    [self.view addSubview:title];
    
    UITextField *emailField = [factory createInputField];
    emailField.placeholder = @"อีเมล";
    [self.view addSubview:emailField];
    
    UIButton *loginButton = [factory createPrimaryButton];
    [loginButton setTitle:@"เข้าสู่ระบบ" forState:UIControlStateNormal];
    [self.view addSubview:loginButton];
}

@end

// การใช้งาน
LoginViewController *loginVC = [[LoginViewController alloc] init];
BOOL isDarkMode = YES;
id<UIComponentFactory> factory = isDarkMode ? [[DarkThemeFactory alloc] init] : [[LightThemeFactory alloc] init];
[loginVC setupWithFactory:factory];
```

---

## 71.4 Builder Pattern

**แนวคิด**: แยกกระบวนการสร้าง object ที่ซับซ้อนออกจาก representation ของมัน

**เมื่อใช้**: สร้าง object ที่มี optional parameters มาก, Alert Builder, Request Builder

```objc
// Network Request Builder
@interface NetworkRequest : NSObject

@property (nonatomic, strong, readonly) NSURL *url;
@property (nonatomic, strong, readonly) NSString *method;
@property (nonatomic, strong, readonly) NSDictionary *headers;
@property (nonatomic, strong, readonly) NSDictionary *parameters;
@property (nonatomic, strong, readonly) NSData *body;
@property (nonatomic, assign, readonly) NSTimeInterval timeout;
@property (nonatomic, assign, readonly) BOOL useCache;

@end

// Builder
@interface NetworkRequestBuilder : NSObject

- (NetworkRequestBuilder *)withURL:(NSString *)urlString;
- (NetworkRequestBuilder *)withMethod:(NSString *)method;
- (NetworkRequestBuilder *)withHeader:(NSString *)value forKey:(NSString *)key;
- (NetworkRequestBuilder *)withHeaders:(NSDictionary *)headers;
- (NetworkRequestBuilder *)withParameter:(id)value forKey:(NSString *)key;
- (NetworkRequestBuilder *)withJSONBody:(NSDictionary *)body;
- (NetworkRequestBuilder *)withTimeout:(NSTimeInterval)timeout;
- (NetworkRequestBuilder *)enableCache;
- (NetworkRequest *)build;

@end

@interface NetworkRequest ()

@property (nonatomic, strong) NSURL *url;
@property (nonatomic, strong) NSString *method;
@property (nonatomic, strong) NSMutableDictionary *mutableHeaders;
@property (nonatomic, strong) NSMutableDictionary *mutableParameters;
@property (nonatomic, strong) NSData *body;
@property (nonatomic, assign) NSTimeInterval timeout;
@property (nonatomic, assign) BOOL useCache;

@end

@implementation NetworkRequestBuilder {
    NetworkRequest *_request;
}

- (instancetype)init {
    self = [super init];
    if (self) {
        _request = [[NetworkRequest alloc] init];
        _request.method = @"GET";
        _request.timeout = 30.0;
        _request.mutableHeaders = [NSMutableDictionary dictionary];
        _request.mutableParameters = [NSMutableDictionary dictionary];
    }
    return self;
}

- (NetworkRequestBuilder *)withURL:(NSString *)urlString {
    _request.url = [NSURL URLWithString:urlString];
    return self;
}

- (NetworkRequestBuilder *)withMethod:(NSString *)method {
    _request.method = method.uppercaseString;
    return self;
}

- (NetworkRequestBuilder *)withHeader:(NSString *)value forKey:(NSString *)key {
    _request.mutableHeaders[key] = value;
    return self;
}

- (NetworkRequestBuilder *)withHeaders:(NSDictionary *)headers {
    [_request.mutableHeaders addEntriesFromDictionary:headers];
    return self;
}

- (NetworkRequestBuilder *)withParameter:(id)value forKey:(NSString *)key {
    _request.mutableParameters[key] = value;
    return self;
}

- (NetworkRequestBuilder *)withJSONBody:(NSDictionary *)body {
    _request.body = [NSJSONSerialization dataWithJSONObject:body options:0 error:nil];
    [self withHeader:@"application/json" forKey:@"Content-Type"];
    return self;
}

- (NetworkRequestBuilder *)withTimeout:(NSTimeInterval)timeout {
    _request.timeout = timeout;
    return self;
}

- (NetworkRequestBuilder *)enableCache {
    _request.useCache = YES;
    return self;
}

- (NetworkRequest *)build {
    // Validate
    NSAssert(_request.url != nil, @"URL is required");
    
    // Copy mutable to immutable
    _request.mutableHeaders = [_request.mutableHeaders copy];
    _request.mutableParameters = [_request.mutableParameters copy];
    
    return _request;
}

@end

// การใช้งาน - Method Chaining
NetworkRequest *request = [[[[[NetworkRequestBuilder alloc] init]
    withURL:@"https://api.example.com/users"]
    withMethod:@"POST"]
    withJSONBody:@{@"name": @"John", @"email": @"john@example.com"}]
    withHeader:@"Bearer token123" forKey:@"Authorization"]
    withTimeout:60.0]
    build];

NSLog(@"URL: %@", request.url);
NSLog(@"Method: %@", request.method);
```

---

## 71.5 Prototype Pattern

**แนวคิด**: สร้าง object ใหม่โดยการ copy จาก prototype object ที่มีอยู่แล้ว

**เมื่อใช้**: สร้าง object ที่ initialization มี cost สูง, Template documents

```objc
// Document Prototype
@interface DocumentTemplate : NSObject <NSCopying>

@property (nonatomic, strong) NSString *title;
@property (nonatomic, strong) NSString *content;
@property (nonatomic, strong) NSMutableDictionary *metadata;
@property (nonatomic, strong) UIFont *titleFont;
@property (nonatomic, strong) UIColor *backgroundColor;

- (instancetype)deepCopy;

@end

@implementation DocumentTemplate

- (id)copyWithZone:(NSZone *)zone {
    DocumentTemplate *copy = [[[self class] allocWithZone:zone] init];
    
    // Shallow copy - pointer is copied
    copy.title = self.title;
    copy.content = self.content;
    copy.titleFont = self.titleFont;
    copy.backgroundColor = self.backgroundColor;
    
    // Deep copy - create new mutable dictionary
    copy.metadata = [self.metadata mutableCopy];
    
    return copy;
}

- (instancetype)deepCopy {
    return [self copy];
}

@end

// Template Registry
@interface TemplateRegistry : NSObject

+ (instancetype)sharedRegistry;
- (void)registerTemplate:(DocumentTemplate *)template withName:(NSString *)name;
- (DocumentTemplate *)cloneTemplateWithName:(NSString *)name;

@end

@implementation TemplateRegistry {
    NSMutableDictionary<NSString *, DocumentTemplate *> *_templates;
}

+ (instancetype)sharedRegistry {
    static TemplateRegistry *instance = nil;
    static dispatch_once_t onceToken;
    dispatch_once(&onceToken, ^{
        instance = [[TemplateRegistry alloc] init];
    });
    return instance;
}

- (instancetype)init {
    self = [super init];
    if (self) {
        _templates = [NSMutableDictionary dictionary];
    }
    return self;
}

- (void)registerTemplate:(DocumentTemplate *)template withName:(NSString *)name {
    _templates[name] = template;
}

- (DocumentTemplate *)cloneTemplateWithName:(NSString *)name {
    return [_templates[name] deepCopy];
}

@end

// การใช้งาน
DocumentTemplate *reportTemplate = [[DocumentTemplate alloc] init];
reportTemplate.title = @"รายงานประจำเดือน";
reportTemplate.content = @"เนื้อหาเริ่มต้น...";
reportTemplate.metadata = [@{@"type": @"report", @"version": @"1.0"} mutableCopy];

[[TemplateRegistry sharedRegistry] registerTemplate:reportTemplate withName:@"monthly-report"];

// สร้าง document จาก template
DocumentTemplate *newDoc = [[TemplateRegistry sharedRegistry] cloneTemplateWithName:@"monthly-report"];
newDoc.title = @"รายงานเดือนมกราคม 2567";
newDoc.metadata[@"date"] = @"2024-01";
```

---

# หมวดที่ 2: Structural Patterns

## 71.6 Adapter Pattern

**แนวคิด**: แปลง interface ของ class หนึ่งให้เป็น interface ที่ client ต้องการ

**เมื่อใช้**: Integrate third-party libraries, Legacy code integration

```objc
// Target Interface (ที่เราต้องการใช้)
@protocol AnalyticsService <NSObject>
- (void)trackEvent:(NSString *)eventName properties:(NSDictionary *)properties;
- (void)identifyUser:(NSString *)userID traits:(NSDictionary *)traits;
- (void)setUserProperty:(NSString *)value forKey:(NSString *)key;
@end

// Adaptee - Library เดิมที่มี interface ต่างออกไป
@interface LegacyAnalytics : NSObject

- (void)logAction:(NSString *)action withData:(NSDictionary *)data userId:(NSString *)userId;
- (void)setProperty:(NSString *)key value:(NSString *)value forUser:(NSString *)userId;

@end

@implementation LegacyAnalytics

- (void)logAction:(NSString *)action withData:(NSDictionary *)data userId:(NSString *)userId {
    NSLog(@"[Legacy] Action: %@ Data: %@ User: %@", action, data, userId);
}

- (void)setProperty:(NSString *)key value:(NSString *)value forUser:(NSString *)userId {
    NSLog(@"[Legacy] Set %@=%@ for user %@", key, value, userId);
}

@end

// Adapter
@interface LegacyAnalyticsAdapter : NSObject <AnalyticsService>

- (instancetype)initWithLegacyAnalytics:(LegacyAnalytics *)legacy;

@end

@interface LegacyAnalyticsAdapter ()
@property (nonatomic, strong) LegacyAnalytics *legacy;
@property (nonatomic, strong) NSString *currentUserID;
@end

@implementation LegacyAnalyticsAdapter

- (instancetype)initWithLegacyAnalytics:(LegacyAnalytics *)legacy {
    self = [super init];
    if (self) {
        _legacy = legacy;
        _currentUserID = @"anonymous";
    }
    return self;
}

- (void)trackEvent:(NSString *)eventName properties:(NSDictionary *)properties {
    [self.legacy logAction:eventName withData:properties userId:self.currentUserID];
}

- (void)identifyUser:(NSString *)userID traits:(NSDictionary *)traits {
    self.currentUserID = userID;
    for (NSString *key in traits) {
        [self.legacy setProperty:key value:[traits[key] description] forUser:userID];
    }
}

- (void)setUserProperty:(NSString *)value forKey:(NSString *)key {
    [self.legacy setProperty:key value:value forUser:self.currentUserID];
}

@end

// การใช้งาน
LegacyAnalytics *legacy = [[LegacyAnalytics alloc] init];
id<AnalyticsService> analytics = [[LegacyAnalyticsAdapter alloc] initWithLegacyAnalytics:legacy];

[analytics trackEvent:@"button_click" properties:@{@"button": @"buy_now"}];
[analytics identifyUser:@"user_123" traits:@{@"name": @"John", @"plan": @"premium"}];
```

---

## 71.7 Decorator Pattern

**แนวคิด**: เพิ่ม behavior ให้กับ object โดยไม่เปลี่ยน class เดิม

**เมื่อใช้**: Logging wrapper, Caching layer, Network retry

```objc
// Component Protocol
@protocol DataService <NSObject>
- (NSData *)fetchDataFromURL:(NSURL *)url;
- (void)saveData:(NSData *)data toURL:(NSURL *)url;
@end

// Concrete Component
@interface NetworkDataService : NSObject <DataService>
@end

@implementation NetworkDataService

- (NSData *)fetchDataFromURL:(NSURL *)url {
    NSLog(@"Fetching from network: %@", url);
    // จริงๆ ต้องเรียก network
    return [@"Network Data" dataUsingEncoding:NSUTF8StringEncoding];
}

- (void)saveData:(NSData *)data toURL:(NSURL *)url {
    NSLog(@"Saving to network: %@", url);
}

@end

// Base Decorator
@interface DataServiceDecorator : NSObject <DataService>

@property (nonatomic, strong) id<DataService> wrappedService;

- (instancetype)initWithService:(id<DataService>)service;

@end

@implementation DataServiceDecorator

- (instancetype)initWithService:(id<DataService>)service {
    self = [super init];
    if (self) {
        _wrappedService = service;
    }
    return self;
}

- (NSData *)fetchDataFromURL:(NSURL *)url {
    return [self.wrappedService fetchDataFromURL:url];
}

- (void)saveData:(NSData *)data toURL:(NSURL *)url {
    [self.wrappedService saveData:data toURL:url];
}

@end

// Logging Decorator
@interface LoggingDataService : DataServiceDecorator
@end

@implementation LoggingDataService

- (NSData *)fetchDataFromURL:(NSURL *)url {
    NSDate *start = [NSDate date];
    NSLog(@"[LOG] Starting fetch: %@", url);
    
    NSData *data = [super fetchDataFromURL:url];
    
    NSTimeInterval elapsed = [[NSDate date] timeIntervalSinceDate:start];
    NSLog(@"[LOG] Fetch completed in %.3fs, size: %lu bytes", elapsed, (unsigned long)data.length);
    
    return data;
}

@end

// Caching Decorator
@interface CachingDataService : DataServiceDecorator

@property (nonatomic, strong) NSCache *cache;

@end

@implementation CachingDataService

- (instancetype)initWithService:(id<DataService>)service {
    self = [super initWithService:service];
    if (self) {
        _cache = [[NSCache alloc] init];
        _cache.countLimit = 100;
    }
    return self;
}

- (NSData *)fetchDataFromURL:(NSURL *)url {
    NSString *key = url.absoluteString;
    NSData *cached = [self.cache objectForKey:key];
    
    if (cached) {
        NSLog(@"[CACHE] Cache hit: %@", url);
        return cached;
    }
    
    NSData *data = [super fetchDataFromURL:url];
    [self.cache setObject:data forKey:key];
    NSLog(@"[CACHE] Cached: %@", url);
    
    return data;
}

@end

// Retry Decorator
@interface RetryDataService : DataServiceDecorator

@property (nonatomic, assign) NSInteger maxRetries;

- (instancetype)initWithService:(id<DataService>)service maxRetries:(NSInteger)maxRetries;

@end

@implementation RetryDataService

- (instancetype)initWithService:(id<DataService>)service maxRetries:(NSInteger)maxRetries {
    self = [super initWithService:service];
    if (self) {
        _maxRetries = maxRetries;
    }
    return self;
}

- (NSData *)fetchDataFromURL:(NSURL *)url {
    NSData *data = nil;
    NSInteger attempts = 0;
    
    while (!data && attempts < self.maxRetries) {
        attempts++;
        NSLog(@"[RETRY] Attempt %ld/%ld", (long)attempts, (long)self.maxRetries);
        data = [super fetchDataFromURL:url];
        
        if (!data) {
            [NSThread sleepForTimeInterval:attempts]; // exponential backoff
        }
    }
    
    return data;
}

@end

// การใช้งาน - ซ้อน decorators
id<DataService> baseService = [[NetworkDataService alloc] init];
id<DataService> withLogging = [[LoggingDataService alloc] initWithService:baseService];
id<DataService> withCache = [[CachingDataService alloc] initWithService:withLogging];
id<DataService> withRetry = [[RetryDataService alloc] initWithService:withCache maxRetries:3];

NSURL *url = [NSURL URLWithString:@"https://api.example.com/data"];
NSData *data = [withRetry fetchDataFromURL:url];
```

---

## 71.8 Facade Pattern

**แนวคิด**: ให้ interface ที่ง่ายขึ้นสำหรับระบบย่อยที่ซับซ้อน

**เมื่อใช้**: ลดความซับซ้อน, เปิด API ที่ง่ายต่อการใช้งาน

```objc
// Complex Subsystems
@interface VideoEncoder : NSObject
- (NSData *)encode:(NSData *)rawVideo format:(NSString *)format quality:(float)quality;
@end

@interface ThumbnailGenerator : NSObject
- (UIImage *)generateFromVideo:(NSData *)video atTime:(NSTimeInterval)time;
@end

@interface CDNUploader : NSObject
- (NSString *)uploadData:(NSData *)data withMetadata:(NSDictionary *)metadata;
@end

@interface NotificationService : NSObject
- (void)notifyUploadComplete:(NSString *)url forUserID:(NSString *)userID;
@end

@implementation VideoEncoder
- (NSData *)encode:(NSData *)rawVideo format:(NSString *)format quality:(float)quality {
    NSLog(@"Encoding video: %@ at %.1f quality", format, quality);
    return rawVideo; // Simplified
}
@end

@implementation ThumbnailGenerator
- (UIImage *)generateFromVideo:(NSData *)video atTime:(NSTimeInterval)time {
    NSLog(@"Generating thumbnail at %.1fs", time);
    return [UIImage new]; // Simplified
}
@end

@implementation CDNUploader
- (NSString *)uploadData:(NSData *)data withMetadata:(NSDictionary *)metadata {
    NSLog(@"Uploading to CDN");
    return @"https://cdn.example.com/video/123"; // Simplified
}
@end

@implementation NotificationService
- (void)notifyUploadComplete:(NSString *)url forUserID:(NSString *)userID {
    NSLog(@"Notifying user %@: video uploaded to %@", userID, url);
}
@end

// Facade
@interface VideoUploadFacade : NSObject

- (NSString *)uploadVideo:(NSData *)videoData
                forUserID:(NSString *)userID
               completion:(void(^)(NSString *url, NSError *error))completion;

@end

@implementation VideoUploadFacade {
    VideoEncoder *_encoder;
    ThumbnailGenerator *_thumbnailGen;
    CDNUploader *_uploader;
    NotificationService *_notifier;
}

- (instancetype)init {
    self = [super init];
    if (self) {
        _encoder = [[VideoEncoder alloc] init];
        _thumbnailGen = [[ThumbnailGenerator alloc] init];
        _uploader = [[CDNUploader alloc] init];
        _notifier = [[NotificationService alloc] init];
    }
    return self;
}

- (NSString *)uploadVideo:(NSData *)videoData
                forUserID:(NSString *)userID
               completion:(void(^)(NSString *url, NSError *error))completion {
    
    // Step 1: Encode
    NSData *encoded = [_encoder encode:videoData format:@"H.264" quality:0.8];
    
    // Step 2: Generate thumbnail
    UIImage *thumbnail = [_thumbnailGen generateFromVideo:encoded atTime:1.0];
    
    // Step 3: Upload
    NSString *url = [_uploader uploadData:encoded withMetadata:@{
        @"userID": userID,
        @"thumbnail": @"base64-thumbnail-data"
    }];
    
    // Step 4: Notify
    [_notifier notifyUploadComplete:url forUserID:userID];
    
    if (completion) {
        completion(url, nil);
    }
    
    return url;
}

@end

// การใช้งาน - ง่ายมาก!
VideoUploadFacade *facade = [[VideoUploadFacade alloc] init];
[facade uploadVideo:rawVideoData forUserID:@"user_123" completion:^(NSString *url, NSError *error) {
    if (url) {
        NSLog(@"Video available at: %@", url);
    }
}];
```

---

## 71.9 Composite Pattern

**แนวคิด**: จัดกลุ่ม objects ในโครงสร้าง tree เพื่อให้ client จัดการ individual objects และ composite objects แบบเดียวกัน

```objc
// Component
@protocol FileSystemItem <NSObject>

@property (nonatomic, strong) NSString *name;
@property (nonatomic, assign, readonly) NSInteger size;

- (void)display:(NSInteger)indent;

@end

// Leaf
@interface File : NSObject <FileSystemItem>

@property (nonatomic, strong) NSString *name;
@property (nonatomic, assign) NSInteger fileSize;

- (instancetype)initWithName:(NSString *)name size:(NSInteger)size;

@end

@implementation File

- (instancetype)initWithName:(NSString *)name size:(NSInteger)size {
    self = [super init];
    if (self) {
        _name = name;
        _fileSize = size;
    }
    return self;
}

- (NSInteger)size { return self.fileSize; }

- (void)display:(NSInteger)indent {
    NSString *padding = [@"" stringByPaddingToLength:indent * 2 withString:@" " startingAtIndex:0];
    NSLog(@"%@📄 %@ (%ld bytes)", padding, self.name, (long)self.fileSize);
}

@end

// Composite
@interface Folder : NSObject <FileSystemItem>

@property (nonatomic, strong) NSString *name;
@property (nonatomic, strong) NSMutableArray<id<FileSystemItem>> *children;

- (instancetype)initWithName:(NSString *)name;
- (void)addItem:(id<FileSystemItem>)item;
- (void)removeItem:(id<FileSystemItem>)item;

@end

@implementation Folder

- (instancetype)initWithName:(NSString *)name {
    self = [super init];
    if (self) {
        _name = name;
        _children = [NSMutableArray array];
    }
    return self;
}

- (void)addItem:(id<FileSystemItem>)item {
    [self.children addObject:item];
}

- (void)removeItem:(id<FileSystemItem>)item {
    [self.children removeObject:item];
}

- (NSInteger)size {
    NSInteger total = 0;
    for (id<FileSystemItem> item in self.children) {
        total += item.size;
    }
    return total;
}

- (void)display:(NSInteger)indent {
    NSString *padding = [@"" stringByPaddingToLength:indent * 2 withString:@" " startingAtIndex:0];
    NSLog(@"%@📁 %@ (%ld bytes)", padding, self.name, (long)self.size);
    
    for (id<FileSystemItem> item in self.children) {
        [item display:indent + 1];
    }
}

@end

// การใช้งาน
Folder *root = [[Folder alloc] initWithName:@"Documents"];

File *readme = [[File alloc] initWithName:@"README.md" size:1024];
Folder *src = [[Folder alloc] initWithName:@"src"];

File *main = [[File alloc] initWithName:@"main.m" size:4096];
File *utils = [[File alloc] initWithName:@"utils.m" size:2048];

[src addItem:main];
[src addItem:utils];

[root addItem:readme];
[root addItem:src];

[root display:0];
// 📁 Documents (7168 bytes)
//   📄 README.md (1024 bytes)
//   📁 src (6144 bytes)
//     📄 main.m (4096 bytes)
//     📄 utils.m (2048 bytes)
```

---

## 71.10 Proxy Pattern

**แนวคิด**: ให้ placeholder หรือ surrogate แทน object จริง เพื่อควบคุมการ access

**เมื่อใช้**: Lazy loading, Access control, Remote proxy, Virtual proxy

```objc
// Service Interface
@protocol ImageService <NSObject>
- (UIImage *)loadImageFromURL:(NSURL *)url;
@end

// Real Subject
@interface RealImageService : NSObject <ImageService>
@end

@implementation RealImageService
- (UIImage *)loadImageFromURL:(NSURL *)url {
    NSLog(@"Loading image from: %@", url);
    NSData *data = [NSData dataWithContentsOfURL:url];
    return [UIImage imageWithData:data];
}
@end

// Lazy Loading Proxy
@interface LazyImageProxy : NSObject <ImageService>

- (instancetype)initWithURL:(NSURL *)url;

@end

@interface LazyImageProxy ()
@property (nonatomic, strong) NSURL *imageURL;
@property (nonatomic, strong) UIImage *cachedImage;
@property (nonatomic, strong) RealImageService *realService;
@end

@implementation LazyImageProxy

- (instancetype)initWithURL:(NSURL *)url {
    self = [super init];
    if (self) {
        _imageURL = url;
        // ยังไม่สร้าง real service หรือโหลดรูป
    }
    return self;
}

- (UIImage *)loadImageFromURL:(NSURL *)url {
    if (!self.cachedImage) {
        // สร้างเมื่อต้องการเท่านั้น
        if (!self.realService) {
            self.realService = [[RealImageService alloc] init];
        }
        NSLog(@"[Proxy] First load - fetching real image");
        self.cachedImage = [self.realService loadImageFromURL:url];
    } else {
        NSLog(@"[Proxy] Cache hit - returning cached image");
    }
    return self.cachedImage;
}

@end
```

---

# หมวดที่ 3: Behavioral Patterns

## 71.11 Observer Pattern

**แนวคิด**: กำหนด one-to-many dependency ระหว่าง objects เมื่อ object เปลี่ยนแปลง objects ที่ขึ้นกับมันจะได้รับแจ้งและ update อัตโนมัติ

iOS มีการ implement Observer หลายรูปแบบ: NSNotificationCenter, KVO, Delegate

```objc
// Custom Observer Pattern
@protocol StockObserver <NSObject>
- (void)stockPrice:(NSString *)symbol changedTo:(double)price change:(double)change;
@end

@interface StockMarket : NSObject

+ (instancetype)sharedMarket;
- (void)addObserver:(id<StockObserver>)observer;
- (void)removeObserver:(id<StockObserver>)observer;
- (void)updatePrice:(double)price forSymbol:(NSString *)symbol;

@end

@interface StockMarket ()
@property (nonatomic, strong) NSHashTable<id<StockObserver>> *observers;
@property (nonatomic, strong) NSMutableDictionary<NSString *, NSNumber *> *prices;
@end

@implementation StockMarket

+ (instancetype)sharedMarket {
    static StockMarket *market = nil;
    static dispatch_once_t token;
    dispatch_once(&token, ^{
        market = [[StockMarket alloc] init];
    });
    return market;
}

- (instancetype)init {
    self = [super init];
    if (self) {
        // NSHashTable.weakObjects prevents retain cycles
        _observers = [NSHashTable weakObjectsHashTable];
        _prices = [NSMutableDictionary dictionary];
    }
    return self;
}

- (void)addObserver:(id<StockObserver>)observer {
    [self.observers addObject:observer];
}

- (void)removeObserver:(id<StockObserver>)observer {
    [self.observers removeObject:observer];
}

- (void)updatePrice:(double)price forSymbol:(NSString *)symbol {
    double oldPrice = [self.prices[symbol] doubleValue];
    self.prices[symbol] = @(price);
    double change = price - oldPrice;
    
    for (id<StockObserver> observer in self.observers) {
        [observer stockPrice:symbol changedTo:price change:change];
    }
}

@end

// Concrete Observer
@interface StockWidget : UIView <StockObserver>

@property (nonatomic, strong) NSString *watchedSymbol;

@end

@implementation StockWidget

- (void)stockPrice:(NSString *)symbol changedTo:(double)price change:(double)change {
    if ([symbol isEqualToString:self.watchedSymbol]) {
        dispatch_async(dispatch_get_main_queue(), ^{
            NSLog(@"[Widget] %@: %.2f (%+.2f)", symbol, price, change);
            // Update UI
        });
    }
}

@end

// การใช้งาน
StockWidget *widget = [[StockWidget alloc] initWithFrame:CGRectMake(0, 0, 200, 50)];
widget.watchedSymbol = @"AAPL";

[[StockMarket sharedMarket] addObserver:widget];
[[StockMarket sharedMarket] updatePrice:182.50 forSymbol:@"AAPL"];
[[StockMarket sharedMarket] updatePrice:183.20 forSymbol:@"AAPL"];
```

---

## 71.12 Strategy Pattern

**แนวคิด**: กำหนดกลุ่ม algorithm แต่ละตัว encapsulate ไว้ และทำให้แลกเปลี่ยนกันได้

**เมื่อใช้**: Sorting algorithms, Validation strategies, Payment methods

```objc
// Sort Strategy
@protocol SortStrategy <NSObject>
- (NSArray *)sort:(NSArray *)array;
@end

@interface BubbleSortStrategy : NSObject <SortStrategy>
@end

@implementation BubbleSortStrategy
- (NSArray *)sort:(NSArray *)array {
    NSMutableArray *arr = [array mutableCopy];
    NSInteger n = arr.count;
    
    for (NSInteger i = 0; i < n - 1; i++) {
        for (NSInteger j = 0; j < n - i - 1; j++) {
            if ([arr[j] compare:arr[j + 1]] == NSOrderedDescending) {
                [arr exchangeObjectAtIndex:j withObjectAtIndex:j + 1];
            }
        }
    }
    return arr;
}
@end

@interface QuickSortStrategy : NSObject <SortStrategy>
@end

@implementation QuickSortStrategy
- (NSArray *)sort:(NSArray *)array {
    // ใช้ built-in sort สำหรับตัวอย่าง
    return [array sortedArrayUsingComparator:^NSComparisonResult(id a, id b) {
        return [a compare:b];
    }];
}
@end

// Context
@interface DataSorter : NSObject

@property (nonatomic, strong) id<SortStrategy> strategy;

- (instancetype)initWithStrategy:(id<SortStrategy>)strategy;
- (NSArray *)sortData:(NSArray *)data;

@end

@implementation DataSorter

- (instancetype)initWithStrategy:(id<SortStrategy>)strategy {
    self = [super init];
    if (self) {
        _strategy = strategy;
    }
    return self;
}

- (NSArray *)sortData:(NSArray *)data {
    return [self.strategy sort:data];
}

@end

// การใช้งาน
NSArray *data = @[@5, @2, @8, @1, @9, @3];

DataSorter *sorter = [[DataSorter alloc] initWithStrategy:[[QuickSortStrategy alloc] init]];
NSArray *sorted = [sorter sortData:data];
NSLog(@"Sorted: %@", sorted);

// เปลี่ยน strategy ได้ runtime
sorter.strategy = [[BubbleSortStrategy alloc] init];
sorted = [sorter sortData:data];
```

---

## 71.13 Command Pattern

**แนวคิด**: Encapsulate request เป็น object ทำให้สามารถ queue, log หรือ undo ได้

**เมื่อใช้**: Undo/Redo, Transaction, Macro recording

```objc
// Command Protocol
@protocol Command <NSObject>
- (void)execute;
- (void)undo;
@end

// Receiver
@interface TextEditor : NSObject

@property (nonatomic, strong) NSMutableString *text;

- (void)insert:(NSString *)text atPosition:(NSUInteger)position;
- (void)delete:(NSUInteger)length atPosition:(NSUInteger)position;

@end

@implementation TextEditor

- (instancetype)init {
    self = [super init];
    if (self) {
        _text = [NSMutableString string];
    }
    return self;
}

- (void)insert:(NSString *)text atPosition:(NSUInteger)position {
    [self.text insertString:text atIndex:position];
}

- (void)delete:(NSUInteger)length atPosition:(NSUInteger)position {
    [self.text deleteCharactersInRange:NSMakeRange(position, length)];
}

@end

// Concrete Commands
@interface InsertTextCommand : NSObject <Command>

- (instancetype)initWithEditor:(TextEditor *)editor text:(NSString *)text position:(NSUInteger)position;

@end

@interface InsertTextCommand ()
@property (nonatomic, strong) TextEditor *editor;
@property (nonatomic, strong) NSString *text;
@property (nonatomic, assign) NSUInteger position;
@end

@implementation InsertTextCommand

- (instancetype)initWithEditor:(TextEditor *)editor text:(NSString *)text position:(NSUInteger)position {
    self = [super init];
    if (self) {
        _editor = editor;
        _text = text;
        _position = position;
    }
    return self;
}

- (void)execute {
    [self.editor insert:self.text atPosition:self.position];
    NSLog(@"Insert '%@' at %lu: '%@'", self.text, (unsigned long)self.position, self.editor.text);
}

- (void)undo {
    [self.editor delete:self.text.length atPosition:self.position];
    NSLog(@"Undo insert: '%@'", self.editor.text);
}

@end

// Command Manager (Invoker)
@interface CommandManager : NSObject

- (void)executeCommand:(id<Command>)command;
- (void)undo;
- (void)redo;
- (BOOL)canUndo;
- (BOOL)canRedo;

@end

@interface CommandManager ()
@property (nonatomic, strong) NSMutableArray<id<Command>> *history;
@property (nonatomic, assign) NSInteger currentIndex;
@end

@implementation CommandManager

- (instancetype)init {
    self = [super init];
    if (self) {
        _history = [NSMutableArray array];
        _currentIndex = -1;
    }
    return self;
}

- (void)executeCommand:(id<Command>)command {
    // ลบ redo history
    if (self.currentIndex < (NSInteger)self.history.count - 1) {
        [self.history removeObjectsInRange:
            NSMakeRange(self.currentIndex + 1, self.history.count - self.currentIndex - 1)];
    }
    
    [command execute];
    [self.history addObject:command];
    self.currentIndex++;
}

- (void)undo {
    if ([self canUndo]) {
        [self.history[self.currentIndex] undo];
        self.currentIndex--;
    }
}

- (void)redo {
    if ([self canRedo]) {
        self.currentIndex++;
        [self.history[self.currentIndex] execute];
    }
}

- (BOOL)canUndo { return self.currentIndex >= 0; }
- (BOOL)canRedo { return self.currentIndex < (NSInteger)self.history.count - 1; }

@end

// การใช้งาน
TextEditor *editor = [[TextEditor alloc] init];
CommandManager *manager = [[CommandManager alloc] init];

[manager executeCommand:[[InsertTextCommand alloc] initWithEditor:editor text:@"Hello" position:0]];
[manager executeCommand:[[InsertTextCommand alloc] initWithEditor:editor text:@" World" position:5]];
// "Hello World"

[manager undo];
// "Hello"

[manager redo];
// "Hello World"
```

---

## 71.14 Template Method Pattern

**แนวคิด**: กำหนด skeleton ของ algorithm ใน base class แต่ให้ subclass override บางขั้นตอน

```objc
// Abstract Data Export
@interface DataExporter : NSObject

// Template Method - ขั้นตอนหลัก
- (void)exportData:(NSArray *)data toPath:(NSString *)path;

// Steps ที่ subclass ต้อง override
- (NSData *)convertData:(NSArray *)data;
- (NSString *)fileExtension;

// Hooks - optional override
- (void)beforeExport:(NSArray *)data;
- (void)afterExport:(NSString *)path;

@end

@implementation DataExporter

// Template Method
- (void)exportData:(NSArray *)data toPath:(NSString *)path {
    [self beforeExport:data]; // Hook
    
    NSLog(@"Preparing export...");
    NSData *convertedData = [self convertData:data]; // Abstract step
    
    NSString *fullPath = [path stringByAppendingPathExtension:[self fileExtension]];
    [convertedData writeToFile:fullPath atomically:YES];
    
    NSLog(@"Exported %lu items to: %@", (unsigned long)data.count, fullPath);
    [self afterExport:fullPath]; // Hook
}

// Abstract - subclass ต้อง override
- (NSData *)convertData:(NSArray *)data {
    [NSException raise:NSInternalInconsistencyException format:@"Subclass must implement convertData:"];
    return nil;
}

- (NSString *)fileExtension {
    [NSException raise:NSInternalInconsistencyException format:@"Subclass must implement fileExtension"];
    return nil;
}

// Hooks - default เป็นว่าง
- (void)beforeExport:(NSArray *)data { }
- (void)afterExport:(NSString *)path { }

@end

// Concrete Exporters
@interface JSONExporter : DataExporter
@end

@implementation JSONExporter

- (NSData *)convertData:(NSArray *)data {
    return [NSJSONSerialization dataWithJSONObject:data options:NSJSONWritingPrettyPrinted error:nil];
}

- (NSString *)fileExtension { return @"json"; }

- (void)afterExport:(NSString *)path {
    NSLog(@"JSON export complete: %@", path);
}

@end

@interface CSVExporter : DataExporter
@end

@implementation CSVExporter

- (NSData *)convertData:(NSArray *)data {
    NSMutableString *csv = [NSMutableString string];
    
    if ([data.firstObject isKindOfClass:[NSDictionary class]]) {
        // Header
        NSDictionary *first = data.firstObject;
        [csv appendString:[[first.allKeys sortedArrayUsingSelector:@selector(compare:)] componentsJoinedByString:@","]];
        [csv appendString:@"\n"];
        
        // Rows
        for (NSDictionary *row in data) {
            NSArray *sortedKeys = [row.allKeys sortedArrayUsingSelector:@selector(compare:)];
            NSArray *values = [sortedKeys valueForKey:@"self"];
            [csv appendString:[[row objectsForKeys:values notFoundMarker:@""] componentsJoinedByString:@","]];
            [csv appendString:@"\n"];
        }
    }
    
    return [csv dataUsingEncoding:NSUTF8StringEncoding];
}

- (NSString *)fileExtension { return @"csv"; }

@end

// การใช้งาน
NSArray *users = @[
    @{@"name": @"Alice", @"age": @30},
    @{@"name": @"Bob", @"age": @25}
];

NSString *documentsPath = NSSearchPathForDirectoriesInDomains(NSDocumentDirectory, NSUserDomainMask, YES).firstObject;

DataExporter *jsonExporter = [[JSONExporter alloc] init];
[jsonExporter exportData:users toPath:[documentsPath stringByAppendingPathComponent:@"users"]];

DataExporter *csvExporter = [[CSVExporter alloc] init];
[csvExporter exportData:users toPath:[documentsPath stringByAppendingPathComponent:@"users"]];
```

---

## 71.15 Chain of Responsibility Pattern

**แนวคิด**: ส่ง request ผ่าน chain ของ handlers โดย handler แต่ละตัวตัดสินใจว่าจะจัดการหรือส่งต่อ

```objc
// Handler Protocol
@protocol RequestHandler <NSObject>
- (void)setNext:(id<RequestHandler>)handler;
- (void)handleRequest:(NSInteger)amount;
@end

@interface BaseHandler : NSObject <RequestHandler>
@property (nonatomic, strong) id<RequestHandler> nextHandler;
@end

@implementation BaseHandler

- (void)setNext:(id<RequestHandler>)handler {
    self.nextHandler = handler;
}

- (void)handleRequest:(NSInteger)amount {
    if (self.nextHandler) {
        [self.nextHandler handleRequest:amount];
    }
}

@end

// Concrete Handlers (Expense Approval)
@interface TeamLeadHandler : BaseHandler
@end

@implementation TeamLeadHandler
- (void)handleRequest:(NSInteger)amount {
    if (amount <= 1000) {
        NSLog(@"Team Lead อนุมัติ %ld บาท", (long)amount);
    } else {
        NSLog(@"Team Lead ส่งต่อ (%ld เกิน 1,000)", (long)amount);
        [super handleRequest:amount];
    }
}
@end

@interface ManagerHandler : BaseHandler
@end

@implementation ManagerHandler
- (void)handleRequest:(NSInteger)amount {
    if (amount <= 10000) {
        NSLog(@"Manager อนุมัติ %ld บาท", (long)amount);
    } else {
        NSLog(@"Manager ส่งต่อ (%ld เกิน 10,000)", (long)amount);
        [super handleRequest:amount];
    }
}
@end

@interface DirectorHandler : BaseHandler
@end

@implementation DirectorHandler
- (void)handleRequest:(NSInteger)amount {
    if (amount <= 100000) {
        NSLog(@"Director อนุมัติ %ld บาท", (long)amount);
    } else {
        NSLog(@"ต้องการอนุมัติจาก Board: %ld บาท", (long)amount);
    }
}
@end

// การใช้งาน
TeamLeadHandler *teamLead = [[TeamLeadHandler alloc] init];
ManagerHandler *manager = [[ManagerHandler alloc] init];
DirectorHandler *director = [[DirectorHandler alloc] init];

[teamLead setNext:manager];
[manager setNext:director];

[teamLead handleRequest:500];     // Team Lead อนุมัติ
[teamLead handleRequest:5000];    // Manager อนุมัติ
[teamLead handleRequest:50000];   // Director อนุมัติ
[teamLead handleRequest:500000];  // ต้องการ Board
```

---

## 71.16 State Pattern

**แนวคิด**: อนุญาตให้ object เปลี่ยน behavior เมื่อ state เปลี่ยน เหมือนว่า object เปลี่ยน class

```objc
// State Protocol
@protocol TrafficLightState <NSObject>
- (void)handleState:(id)light;
- (NSString *)color;
- (NSTimeInterval)duration;
@end

@interface TrafficLight : NSObject

@property (nonatomic, strong) id<TrafficLightState> currentState;

- (void)change;
- (NSString *)currentColor;

@end

// Concrete States
@interface RedState : NSObject <TrafficLightState>
@end
@interface YellowState : NSObject <TrafficLightState>
@end
@interface GreenState : NSObject <TrafficLightState>
@end

@implementation RedState
- (void)handleState:(TrafficLight *)light {
    NSLog(@"🔴 หยุด! (%0.f วินาที)", self.duration);
    light.currentState = [[GreenState alloc] init];
}
- (NSString *)color { return @"Red"; }
- (NSTimeInterval)duration { return 30.0; }
@end

@implementation YellowState
- (void)handleState:(TrafficLight *)light {
    NSLog(@"🟡 เตรียมพร้อม! (%0.f วินาที)", self.duration);
    light.currentState = [[RedState alloc] init];
}
- (NSString *)color { return @"Yellow"; }
- (NSTimeInterval)duration { return 5.0; }
@end

@implementation GreenState
- (void)handleState:(TrafficLight *)light {
    NSLog(@"🟢 ไปได้! (%0.f วินาที)", self.duration);
    light.currentState = [[YellowState alloc] init];
}
- (NSString *)color { return @"Green"; }
- (NSTimeInterval)duration { return 25.0; }
@end

@implementation TrafficLight

- (instancetype)init {
    self = [super init];
    if (self) {
        _currentState = [[RedState alloc] init];
    }
    return self;
}

- (void)change {
    [self.currentState handleState:self];
}

- (NSString *)currentColor {
    return self.currentState.color;
}

@end

// การใช้งาน
TrafficLight *light = [[TrafficLight alloc] init];
[light change]; // Red -> Green
[light change]; // Green -> Yellow
[light change]; // Yellow -> Red
[light change]; // Red -> Green
```

---

## 71.17 Mediator Pattern

**แนวคิด**: กำหนด object ที่ encapsulate วิธีที่ objects กลุ่มหนึ่งโต้ตอบกัน ลด coupling ระหว่าง colleagues

```objc
// Mediator Protocol
@protocol ChatMediator <NSObject>
- (void)sendMessage:(NSString *)message from:(id)sender;
- (void)addUser:(id)user;
@end

// Colleague
@interface ChatUser : NSObject

@property (nonatomic, strong) NSString *name;
@property (nonatomic, weak) id<ChatMediator> mediator;

- (instancetype)initWithName:(NSString *)name mediator:(id<ChatMediator>)mediator;
- (void)sendMessage:(NSString *)message;
- (void)receiveMessage:(NSString *)message from:(NSString *)sender;

@end

@implementation ChatUser

- (instancetype)initWithName:(NSString *)name mediator:(id<ChatMediator>)mediator {
    self = [super init];
    if (self) {
        _name = name;
        _mediator = mediator;
    }
    return self;
}

- (void)sendMessage:(NSString *)message {
    NSLog(@"%@ ส่ง: %@", self.name, message);
    [self.mediator sendMessage:message from:self];
}

- (void)receiveMessage:(NSString *)message from:(NSString *)sender {
    NSLog(@"%@ ได้รับจาก %@: %@", self.name, sender, message);
}

@end

// Concrete Mediator
@interface ChatRoom : NSObject <ChatMediator>

@property (nonatomic, strong) NSMutableArray<ChatUser *> *users;

@end

@implementation ChatRoom

- (instancetype)init {
    self = [super init];
    if (self) {
        _users = [NSMutableArray array];
    }
    return self;
}

- (void)addUser:(ChatUser *)user {
    [self.users addObject:user];
}

- (void)sendMessage:(NSString *)message from:(ChatUser *)sender {
    for (ChatUser *user in self.users) {
        if (user != sender) {
            [user receiveMessage:message from:sender.name];
        }
    }
}

@end

// การใช้งาน
ChatRoom *room = [[ChatRoom alloc] init];

ChatUser *alice = [[ChatUser alloc] initWithName:@"Alice" mediator:room];
ChatUser *bob = [[ChatUser alloc] initWithName:@"Bob" mediator:room];
ChatUser *charlie = [[ChatUser alloc] initWithName:@"Charlie" mediator:room];

[room addUser:alice];
[room addUser:bob];
[room addUser:charlie];

[alice sendMessage:@"สวัสดีทุกคน!"];
[bob sendMessage:@"สวัสดี Alice!"];
```

---

## 71.18 Iterator Pattern

**แนวคิด**: ให้วิธีเข้าถึงสมาชิกของ collection โดยไม่เปิดเผย representation ภายใน

```objc
// Iterator Protocol
@protocol Iterator <NSObject>
- (BOOL)hasNext;
- (id)next;
- (void)reset;
@end

// Number Range Iterator
@interface NumberRangeIterator : NSObject <Iterator>

- (instancetype)initWithStart:(NSInteger)start end:(NSInteger)end step:(NSInteger)step;

@end

@interface NumberRangeIterator ()
@property (nonatomic, assign) NSInteger start;
@property (nonatomic, assign) NSInteger end;
@property (nonatomic, assign) NSInteger step;
@property (nonatomic, assign) NSInteger current;
@end

@implementation NumberRangeIterator

- (instancetype)initWithStart:(NSInteger)start end:(NSInteger)end step:(NSInteger)step {
    self = [super init];
    if (self) {
        _start = start;
        _end = end;
        _step = step;
        _current = start;
    }
    return self;
}

- (BOOL)hasNext { return self.current <= self.end; }

- (id)next {
    if (![self hasNext]) return nil;
    NSNumber *value = @(self.current);
    self.current += self.step;
    return value;
}

- (void)reset { self.current = self.start; }

@end

// การใช้งาน
id<Iterator> evenNumbers = [[NumberRangeIterator alloc] initWithStart:0 end:20 step:2];

while ([evenNumbers hasNext]) {
    NSNumber *num = [evenNumbers next];
    NSLog(@"%@", num);
}
// 0, 2, 4, 6, 8, 10, 12, 14, 16, 18, 20
```

---

## 71.19 Memento Pattern

**แนวคิด**: Capture และ externalize state ของ object เพื่อ restore ในภายหลัง โดยไม่ละเมิด encapsulation

```objc
// Memento
@interface EditorMemento : NSObject

@property (nonatomic, strong, readonly) NSString *text;
@property (nonatomic, assign, readonly) NSUInteger cursorPosition;
@property (nonatomic, strong, readonly) NSDate *savedAt;

@end

@interface EditorMemento ()
@property (nonatomic, strong) NSString *text;
@property (nonatomic, assign) NSUInteger cursorPosition;
@property (nonatomic, strong) NSDate *savedAt;
@end

@implementation EditorMemento

- (instancetype)initWithText:(NSString *)text cursor:(NSUInteger)cursor {
    self = [super init];
    if (self) {
        _text = [text copy];
        _cursorPosition = cursor;
        _savedAt = [NSDate date];
    }
    return self;
}

@end

// Originator
@interface TextEditorOriginator : NSObject

@property (nonatomic, strong) NSString *text;
@property (nonatomic, assign) NSUInteger cursorPosition;

- (EditorMemento *)save;
- (void)restore:(EditorMemento *)memento;

@end

@implementation TextEditorOriginator

- (EditorMemento *)save {
    return [[EditorMemento alloc] initWithText:self.text cursor:self.cursorPosition];
}

- (void)restore:(EditorMemento *)memento {
    self.text = memento.text;
    self.cursorPosition = memento.cursorPosition;
    NSLog(@"Restored to: '%@' (saved at %@)", self.text, memento.savedAt);
}

@end

// Caretaker
@interface History : NSObject

- (void)save:(EditorMemento *)memento;
- (EditorMemento *)undo;
- (NSInteger)count;

@end

@interface History ()
@property (nonatomic, strong) NSMutableArray<EditorMemento *> *mementos;
@end

@implementation History

- (instancetype)init {
    self = [super init];
    if (self) {
        _mementos = [NSMutableArray array];
    }
    return self;
}

- (void)save:(EditorMemento *)memento {
    [self.mementos addObject:memento];
}

- (EditorMemento *)undo {
    if (self.mementos.count == 0) return nil;
    EditorMemento *last = self.mementos.lastObject;
    [self.mementos removeLastObject];
    return last;
}

- (NSInteger)count { return self.mementos.count; }

@end

// การใช้งาน
TextEditorOriginator *editor2 = [[TextEditorOriginator alloc] init];
History *history = [[History alloc] init];

editor2.text = @"Hello";
[history save:[editor2 save]];

editor2.text = @"Hello World";
[history save:[editor2 save]];

editor2.text = @"Hello World!";

NSLog(@"Current: %@", editor2.text); // Hello World!

[editor2 restore:[history undo]]; // Hello World
[editor2 restore:[history undo]]; // Hello
```

---

## 71.20 Visitor Pattern

**แนวคิด**: กำหนด operation ที่จะทำกับสมาชิกของ object structure โดยไม่เปลี่ยน class

```objc
@class Circle, Rectangle, Triangle;

// Visitor Protocol
@protocol ShapeVisitor <NSObject>
- (double)visitCircle:(Circle *)circle;
- (double)visitRectangle:(Rectangle *)rect;
- (double)visitTriangle:(Triangle *)triangle;
@end

// Shape Protocol
@protocol Shape <NSObject>
- (double)accept:(id<ShapeVisitor>)visitor;
- (NSString *)name;
@end

// Concrete Shapes
@interface Circle : NSObject <Shape>
@property (nonatomic, assign) double radius;
- (instancetype)initWithRadius:(double)radius;
@end

@implementation Circle
- (instancetype)initWithRadius:(double)radius {
    self = [super init]; if (self) { _radius = radius; } return self;
}
- (double)accept:(id<ShapeVisitor>)visitor { return [visitor visitCircle:self]; }
- (NSString *)name { return @"Circle"; }
@end

@interface Rectangle : NSObject <Shape>
@property (nonatomic, assign) double width, height;
- (instancetype)initWithWidth:(double)w height:(double)h;
@end

@implementation Rectangle
- (instancetype)initWithWidth:(double)w height:(double)h {
    self = [super init]; if (self) { _width = w; _height = h; } return self;
}
- (double)accept:(id<ShapeVisitor>)visitor { return [visitor visitRectangle:self]; }
- (NSString *)name { return @"Rectangle"; }
@end

@interface Triangle : NSObject <Shape>
@property (nonatomic, assign) double base, height;
- (instancetype)initWithBase:(double)b height:(double)h;
@end

@implementation Triangle
- (instancetype)initWithBase:(double)b height:(double)h {
    self = [super init]; if (self) { _base = b; _height = h; } return self;
}
- (double)accept:(id<ShapeVisitor>)visitor { return [visitor visitTriangle:self]; }
- (NSString *)name { return @"Triangle"; }
@end

// Area Calculator Visitor
@interface AreaCalculator : NSObject <ShapeVisitor>
@end

@implementation AreaCalculator
- (double)visitCircle:(Circle *)circle { return M_PI * circle.radius * circle.radius; }
- (double)visitRectangle:(Rectangle *)rect { return rect.width * rect.height; }
- (double)visitTriangle:(Triangle *)triangle { return 0.5 * triangle.base * triangle.height; }
@end

// Perimeter Calculator Visitor
@interface PerimeterCalculator : NSObject <ShapeVisitor>
@end

@implementation PerimeterCalculator
- (double)visitCircle:(Circle *)circle { return 2 * M_PI * circle.radius; }
- (double)visitRectangle:(Rectangle *)rect { return 2 * (rect.width + rect.height); }
- (double)visitTriangle:(Triangle *)triangle { return triangle.base * 3; } // สมมติสามเหลี่ยมด้านเท่า
@end

// การใช้งาน
NSArray<id<Shape>> *shapes = @[
    [[Circle alloc] initWithRadius:5],
    [[Rectangle alloc] initWithWidth:4 height:6],
    [[Triangle alloc] initWithBase:3 height:4]
];

AreaCalculator *areaCalc = [[AreaCalculator alloc] init];
PerimeterCalculator *perimCalc = [[PerimeterCalculator alloc] init];

for (id<Shape> shape in shapes) {
    double area = [shape accept:areaCalc];
    double perimeter = [shape accept:perimCalc];
    NSLog(@"%@: Area=%.2f, Perimeter=%.2f", shape.name, area, perimeter);
}
```

---

## แบบฝึกหัด

### แบบฝึกหัดที่ 1
สร้าง Logger ด้วย Singleton ที่รองรับ log levels (DEBUG, INFO, WARNING, ERROR) และ output destinations หลายที่ (Console, File)

### แบบฝึกหัดที่ 2
สร้าง Image Processing Pipeline ด้วย Decorator ที่มี filters: Resize, Grayscale, Blur, Watermark

### แบบฝึกหัดที่ 3
สร้างระบบ Shopping Cart ด้วย Command Pattern ที่รองรับ add/remove item และ undo/redo

### แบบฝึกหัดที่ 4
สร้าง Notification System ด้วย Observer ที่รองรับ filter by type และ priority

---

## สรุป

Design Patterns ช่วยให้โค้ดมีคุณภาพสูงขึ้นในด้าน:
- **Reusability** - ใช้ซ้ำได้ในหลายสถานการณ์
- **Maintainability** - แก้ไขและบำรุงรักษาง่าย
- **Flexibility** - ปรับเปลี่ยนได้ตามความต้องการ
- **Testability** - ทดสอบได้ง่ายกว่า

แต่ควรระวัง Over-engineering - ไม่ใช่ทุกปัญหาต้องการ Design Pattern ให้ใช้เมื่อมันแก้ปัญหาจริงๆ
