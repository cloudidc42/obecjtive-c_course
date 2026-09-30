# Part 73: Dependency Injection ใน Objective-C

## บทนำ

Dependency Injection (DI) คือหนึ่งในหลักการออกแบบซอฟต์แวร์ที่สำคัญที่สุดในการเขียนโค้ดที่มีคุณภาพ เข้าใจง่าย และทดสอบได้ง่าย บทนี้จะพาคุณทำความเข้าใจแนวคิดนี้อย่างลึกซึ้งพร้อมตัวอย่างการใช้งานจริงใน Objective-C

---

## 73.1 Dependency Injection คืออะไร?

### ความหมายพื้นฐาน

Dependency Injection เป็นเทคนิคที่ทำให้ object ไม่ต้องสร้าง dependency ของตัวเองขึ้นมา แต่ให้ dependency ถูกส่งผ่านมาจากภายนอก (inject เข้ามา)

คำว่า "dependency" หมายถึง object หรือ service ที่ class หนึ่งต้องการเพื่อทำงาน ตัวอย่างเช่น `OrderService` ที่ต้องการ `DatabaseService` เพื่อบันทึกข้อมูล ในกรณีนี้ `OrderService` มี "dependency" คือ `DatabaseService`

### ปัญหาที่เกิดขึ้นโดยไม่ใช้ DI

```objc
// ❌ ไม่ดี: สร้าง dependency ภายในตัวเอง
@interface OrderService : NSObject

- (void)placeOrder:(Order *)order;

@end

@implementation OrderService

- (void)placeOrder:(Order *)order {
    // สร้าง dependency ภายใน - ปัญหา!
    DatabaseService *db = [[DatabaseService alloc] init];
    NetworkService *network = [[NetworkService alloc] init];
    LoggingService *logger = [[LoggingService alloc] init];
    
    [db saveOrder:order];
    [network sendConfirmation:order];
    [logger log:@"Order placed"];
}

@end
```

**ปัญหาของโค้ดด้านบน:**
1. **ทดสอบยาก**: ไม่สามารถแทนที่ `DatabaseService` ด้วย mock ได้เมื่อทดสอบ
2. **ยืดหยุ่นน้อย**: ถ้าต้องการใช้ database ชนิดอื่น ต้องแก้โค้ดใน `OrderService`
3. **Tight coupling**: `OrderService` ขึ้นอยู่กับ concrete class โดยตรง
4. **ละเมิด Single Responsibility Principle**: class รับผิดชอบสร้าง dependency ด้วย

### การแก้ไขด้วย Dependency Injection

```objc
// ✅ ดี: รับ dependency จากภายนอก
@interface OrderService : NSObject

- (instancetype)initWithDatabase:(DatabaseService *)database
                         network:(NetworkService *)network
                         logger:(LoggingService *)logger;

- (void)placeOrder:(Order *)order;

@end

@implementation OrderService {
    DatabaseService *_database;
    NetworkService *_network;
    LoggingService *_logger;
}

- (instancetype)initWithDatabase:(DatabaseService *)database
                         network:(NetworkService *)network
                         logger:(LoggingService *)logger {
    self = [super init];
    if (self) {
        _database = database;
        _network = network;
        _logger = logger;
    }
    return self;
}

- (void)placeOrder:(Order *)order {
    [_database saveOrder:order];
    [_network sendConfirmation:order];
    [_logger log:@"Order placed"];
}

@end
```

---

## 73.2 ประโยชน์ของ Dependency Injection

### 1. Loose Coupling (การเชื่อมต่อแบบหลวม)

เมื่อใช้ DI class ไม่รู้จักและไม่ขึ้นอยู่กับ implementation cụ thể ของ dependency ทำให้สามารถเปลี่ยน implementation ได้โดยไม่กระทบ class ที่ใช้งาน

### 2. Testability (ทดสอบได้ง่าย)

สามารถส่ง mock objects เข้าไปเมื่อทดสอบ ทำให้การทดสอบ unit test ทำได้โดยไม่ต้องพึ่ง real database, network หรือ services อื่น ๆ

### 3. Reusability (ใช้ซ้ำได้)

Class ที่ใช้ DI สามารถนำไปใช้ในบริบทต่าง ๆ ได้ง่ายขึ้น โดยแค่ส่ง dependency ที่เหมาะสมเข้าไป

### 4. Maintainability (ดูแลรักษาง่าย)

การเปลี่ยนแปลง implementation ของ dependency ทำได้ในที่เดียว

---

## 73.3 Constructor Injection (การ Inject ผ่าน Constructor)

Constructor injection คือการส่ง dependency เข้าไปผ่าน initializer ของ object นี่เป็นวิธีที่แนะนำที่สุดเพราะทำให้ dependency ชัดเจนและบังคับให้ระบุ dependency ทั้งหมดเมื่อสร้าง object

### ตัวอย่างพื้นฐาน

```objc
// Protocol สำหรับ dependency
@protocol DatabaseProtocol <NSObject>
- (BOOL)saveUser:(User *)user;
- (User *)findUserById:(NSString *)userId;
- (void)deleteUser:(NSString *)userId;
@end

// Concrete implementation
@interface SQLiteDatabase : NSObject <DatabaseProtocol>
@end

@implementation SQLiteDatabase

- (BOOL)saveUser:(User *)user {
    // บันทึกลง SQLite
    NSLog(@"Saving user to SQLite: %@", user.name);
    return YES;
}

- (User *)findUserById:(NSString *)userId {
    // ค้นหาจาก SQLite
    return nil; // simplified
}

- (void)deleteUser:(NSString *)userId {
    NSLog(@"Deleting user from SQLite: %@", userId);
}

@end

// UserRepository ที่ใช้ Constructor Injection
@interface UserRepository : NSObject

- (instancetype)initWithDatabase:(id<DatabaseProtocol>)database;
- (BOOL)createUser:(User *)user;
- (User *)getUserById:(NSString *)userId;

@end

@implementation UserRepository {
    id<DatabaseProtocol> _database;
}

- (instancetype)initWithDatabase:(id<DatabaseProtocol>)database {
    self = [super init];
    if (self) {
        NSParameterAssert(database != nil);
        _database = database;
    }
    return self;
}

- (BOOL)createUser:(User *)user {
    return [_database saveUser:user];
}

- (User *)getUserById:(NSString *)userId {
    return [_database findUserById:userId];
}

@end
```

### Constructor Injection กับหลาย Dependencies

```objc
@interface PaymentProcessor : NSObject

- (instancetype)initWithPaymentGateway:(id<PaymentGatewayProtocol>)gateway
                            fraudCheck:(id<FraudCheckProtocol>)fraudCheck
                               logging:(id<LoggingProtocol>)logging
                             analytics:(id<AnalyticsProtocol>)analytics;

- (PaymentResult *)processPayment:(Payment *)payment;

@end

@implementation PaymentProcessor {
    id<PaymentGatewayProtocol> _gateway;
    id<FraudCheckProtocol> _fraudCheck;
    id<LoggingProtocol> _logging;
    id<AnalyticsProtocol> _analytics;
}

- (instancetype)initWithPaymentGateway:(id<PaymentGatewayProtocol>)gateway
                            fraudCheck:(id<FraudCheckProtocol>)fraudCheck
                               logging:(id<LoggingProtocol>)logging
                             analytics:(id<AnalyticsProtocol>)analytics {
    self = [super init];
    if (self) {
        _gateway = gateway;
        _fraudCheck = fraudCheck;
        _logging = logging;
        _analytics = analytics;
    }
    return self;
}

- (PaymentResult *)processPayment:(Payment *)payment {
    [_logging log:[NSString stringWithFormat:@"Processing payment: %@", payment.transactionId]];
    
    if (![_fraudCheck isValidPayment:payment]) {
        [_logging log:@"Payment rejected: fraud detected"];
        [_analytics trackEvent:@"payment_fraud_detected"];
        return [PaymentResult failedResult:@"Fraud detected"];
    }
    
    PaymentResult *result = [_gateway charge:payment];
    [_analytics trackEvent:result.success ? @"payment_success" : @"payment_failed"];
    
    return result;
}

@end
```

---

## 73.4 Property Injection (การ Inject ผ่าน Property)

Property injection คือการส่ง dependency เข้าไปผ่าน property ของ object วิธีนี้มีประโยชน์เมื่อ dependency เป็น optional หรือสามารถเปลี่ยนแปลงได้หลังจากสร้าง object แล้ว

### ตัวอย่าง Property Injection

```objc
@interface EmailNotifier : NSObject

// Optional dependency - ถ้าไม่ตั้ง จะใช้ default logger
@property (nonatomic, strong) id<LoggingProtocol> logger;

// Required dependency - ต้องตั้งก่อนใช้งาน
@property (nonatomic, strong) id<EmailServiceProtocol> emailService;

- (BOOL)sendWelcomeEmail:(NSString *)emailAddress;

@end

@implementation EmailNotifier

- (instancetype)init {
    self = [super init];
    if (self) {
        // Default logger - สามารถ override ได้
        _logger = [[ConsoleLogger alloc] init];
    }
    return self;
}

- (BOOL)sendWelcomeEmail:(NSString *)emailAddress {
    if (!_emailService) {
        [_logger log:@"Error: emailService not configured"];
        return NO;
    }
    
    [_logger log:[NSString stringWithFormat:@"Sending welcome email to: %@", emailAddress]];
    return [_emailService sendEmail:emailAddress
                           subject:@"Welcome!"
                              body:@"Welcome to our platform!"];
}

@end

// การใช้งาน
EmailNotifier *notifier = [[EmailNotifier alloc] init];
notifier.emailService = [[SMTPEmailService alloc] init]; // Inject dependency
notifier.logger = [[FileLogger alloc] init]; // Override default logger

[notifier sendWelcomeEmail:@"user@example.com"];
```

### Property Injection กับ Lazy Loading

```objc
@interface ReportGenerator : NSObject

@property (nonatomic, strong) id<DataFormatterProtocol> formatter;
@property (nonatomic, strong) id<ChartGeneratorProtocol> chartGenerator;
@property (nonatomic, strong) id<PDFExporterProtocol> pdfExporter;

- (NSData *)generateReport:(ReportData *)data;

@end

@implementation ReportGenerator

- (id<DataFormatterProtocol>)formatter {
    if (!_formatter) {
        // Lazy default ถ้าไม่มีการ inject
        _formatter = [[DefaultDataFormatter alloc] init];
    }
    return _formatter;
}

- (id<ChartGeneratorProtocol>)chartGenerator {
    if (!_chartGenerator) {
        _chartGenerator = [[DefaultChartGenerator alloc] init];
    }
    return _chartGenerator;
}

- (id<PDFExporterProtocol>)pdfExporter {
    if (!_pdfExporter) {
        _pdfExporter = [[DefaultPDFExporter alloc] init];
    }
    return _pdfExporter;
}

- (NSData *)generateReport:(ReportData *)data {
    NSString *formattedData = [self.formatter format:data];
    UIImage *chart = [self.chartGenerator generateChart:data];
    return [self.pdfExporter exportToPDF:formattedData chart:chart];
}

@end
```

---

## 73.5 Method Injection (การ Inject ผ่าน Method)

Method injection คือการส่ง dependency เข้าไปเป็น parameter ของ method โดยตรง วิธีนี้เหมาะกับกรณีที่ต้องการ dependency เฉพาะสำหรับการเรียก method นั้น ๆ

### ตัวอย่าง Method Injection

```objc
@interface DataProcessor : NSObject

// Method injection - dependency ส่งผ่านแต่ละ method call
- (NSArray *)processData:(NSArray *)data
          withFormatter:(id<DataFormatterProtocol>)formatter;

- (BOOL)exportData:(NSArray *)data
        toExporter:(id<DataExporterProtocol>)exporter;

- (NSArray *)filterData:(NSArray *)data
          usingPredicate:(id<PredicateProtocol>)predicate;

@end

@implementation DataProcessor

- (NSArray *)processData:(NSArray *)data
          withFormatter:(id<DataFormatterProtocol>)formatter {
    NSMutableArray *result = [NSMutableArray array];
    for (id item in data) {
        [result addObject:[formatter format:item]];
    }
    return [result copy];
}

- (BOOL)exportData:(NSArray *)data
        toExporter:(id<DataExporterProtocol>)exporter {
    return [exporter export:data];
}

- (NSArray *)filterData:(NSArray *)data
         usingPredicate:(id<PredicateProtocol>)predicate {
    NSMutableArray *filtered = [NSMutableArray array];
    for (id item in data) {
        if ([predicate evaluate:item]) {
            [filtered addObject:item];
        }
    }
    return [filtered copy];
}

@end

// การใช้งาน
DataProcessor *processor = [[DataProcessor alloc] init];
NSArray *rawData = @[@"item1", @"item2", @"item3"];

// ส่ง formatter ที่ต้องการ
JSONFormatter *jsonFormatter = [[JSONFormatter alloc] init];
NSArray *jsonData = [processor processData:rawData withFormatter:jsonFormatter];

// หรือใช้ formatter อื่น
XMLFormatter *xmlFormatter = [[XMLFormatter alloc] init];
NSArray *xmlData = [processor processData:rawData withFormatter:xmlFormatter];
```

### เปรียบเทียบทั้ง 3 แบบ

| วิธี | เมื่อใช้ | ข้อดี | ข้อเสีย |
|------|---------|-------|---------|
| Constructor | Required dependencies | ชัดเจน, immutable | หลาย param ทำให้ยาว |
| Property | Optional/changeable deps | ยืดหยุ่น | อาจลืมตั้งค่า |
| Method | Per-call dependencies | มีประสิทธิภาพ | เพิ่ม param ให้ method |

---

## 73.6 Service Locator vs Dependency Injection

Service Locator เป็น pattern อีกแบบหนึ่งที่ใช้จัดการ dependencies แต่มีความแตกต่างกับ DI อย่างมีนัยสำคัญ

### Service Locator Pattern

```objc
// Service Locator - Anti-pattern ในหลาย ๆ กรณี
@interface ServiceLocator : NSObject

+ (instancetype)sharedLocator;

- (void)registerService:(id)service forProtocol:(Protocol *)protocol;
- (id)serviceForProtocol:(Protocol *)protocol;

@end

@implementation ServiceLocator {
    NSMutableDictionary *_services;
}

+ (instancetype)sharedLocator {
    static ServiceLocator *instance;
    static dispatch_once_t onceToken;
    dispatch_once(&onceToken, ^{
        instance = [[ServiceLocator alloc] init];
    });
    return instance;
}

- (instancetype)init {
    self = [super init];
    if (self) {
        _services = [NSMutableDictionary dictionary];
    }
    return self;
}

- (void)registerService:(id)service forProtocol:(Protocol *)protocol {
    NSString *key = NSStringFromProtocol(protocol);
    _services[key] = service;
}

- (id)serviceForProtocol:(Protocol *)protocol {
    NSString *key = NSStringFromProtocol(protocol);
    return _services[key];
}

@end

// การใช้งาน Service Locator - ปัญหาคือซ่อน dependencies
@implementation OrderService

- (void)placeOrder:(Order *)order {
    // Dependencies ไม่ชัดเจนจาก method signature
    id<DatabaseProtocol> db = [[ServiceLocator sharedLocator] 
                                serviceForProtocol:@protocol(DatabaseProtocol)];
    id<NetworkProtocol> network = [[ServiceLocator sharedLocator] 
                                   serviceForProtocol:@protocol(NetworkProtocol)];
    
    [db saveOrder:order];
    [network sendConfirmation:order];
}

@end
```

### ปัญหาของ Service Locator

```objc
// ❌ ปัญหาที่ 1: Dependencies ซ่อนอยู่
// ดู method signature แล้วไม่รู้ว่า class ต้องการอะไร
- (void)processPayment:(Payment *)payment {
    // ต้องไป lookup ว่าใช้ service อะไรบ้าง
    id<GatewayProtocol> gateway = [[ServiceLocator sharedLocator] 
                                    serviceForProtocol:@protocol(GatewayProtocol)];
    // ...
}

// ❌ ปัญหาที่ 2: ยากต่อการทดสอบ
// ต้อง configure ServiceLocator ก่อนทุก test
- (void)testProcessPayment {
    // ต้อง register mock service ใน global locator
    MockGateway *mockGateway = [[MockGateway alloc] init];
    [[ServiceLocator sharedLocator] registerService:mockGateway 
                                        forProtocol:@protocol(GatewayProtocol)];
    // ถ้า test อื่นลืม cleanup จะเกิด side effects
}
```

### เปรียบเทียบ Service Locator vs DI

```objc
// Service Locator - Dependencies ซ่อนอยู่
@implementation UserController

- (void)createUser:(UserDTO *)dto {
    // ไม่รู้ว่า class นี้ต้องการ services อะไร จน runtime
    id<UserService> service = [[ServiceLocator sharedLocator] 
                               serviceForProtocol:@protocol(UserService)];
    [service createUser:dto];
}

@end

// Dependency Injection - Dependencies ชัดเจน
@interface UserController : NSObject

// ชัดเจนมากว่า class นี้ต้องการ UserService
- (instancetype)initWithUserService:(id<UserService>)userService;

@end

@implementation UserController {
    id<UserService> _userService;
}

- (instancetype)initWithUserService:(id<UserService>)userService {
    self = [super init];
    if (self) {
        _userService = userService;
    }
    return self;
}

- (void)createUser:(UserDTO *)dto {
    [_userService createUser:dto];
}

@end
```

---

## 73.7 Protocol-Based DI ใน Objective-C

Protocol เป็นเครื่องมือที่ทรงพลังใน Objective-C สำหรับทำ DI เพราะช่วยกำหนด interface โดยไม่ผูกกับ concrete implementation

### การออกแบบ Protocol สำหรับ DI

```objc
// กำหนด Protocol สำหรับแต่ละ service
@protocol DatabaseServiceProtocol <NSObject>

- (BOOL)save:(id)entity;
- (id)findById:(NSString *)entityId;
- (NSArray *)findAll;
- (BOOL)delete:(NSString *)entityId;
- (BOOL)update:(id)entity;

@optional
- (void)beginTransaction;
- (void)commitTransaction;
- (void)rollbackTransaction;

@end

@protocol CacheServiceProtocol <NSObject>

- (void)setValue:(id)value forKey:(NSString *)key;
- (id)valueForKey:(NSString *)key;
- (void)removeValueForKey:(NSString *)key;
- (void)clearAll;
- (BOOL)hasValueForKey:(NSString *)key;

@end

@protocol NetworkServiceProtocol <NSObject>

- (void)GET:(NSString *)endpoint
 completion:(void(^)(id response, NSError *error))completion;

- (void)POST:(NSString *)endpoint
         body:(NSDictionary *)body
   completion:(void(^)(id response, NSError *error))completion;

- (void)PUT:(NSString *)endpoint
        body:(NSDictionary *)body
  completion:(void(^)(id response, NSError *error))completion;

- (void)DELETE:(NSString *)endpoint
    completion:(void(^)(BOOL success, NSError *error))completion;

@end
```

### Repository Pattern กับ Protocol-Based DI

```objc
// Abstract Repository Protocol
@protocol UserRepositoryProtocol <NSObject>

- (void)saveUser:(User *)user 
      completion:(void(^)(BOOL success, NSError *error))completion;

- (void)getUserById:(NSString *)userId 
         completion:(void(^)(User *user, NSError *error))completion;

- (void)getAllUsers:(void(^)(NSArray<User *> *users, NSError *error))completion;

- (void)updateUser:(User *)user 
        completion:(void(^)(BOOL success, NSError *error))completion;

- (void)deleteUser:(NSString *)userId 
        completion:(void(^)(BOOL success, NSError *error))completion;

@end

// Implementation สำหรับ Core Data
@interface CoreDataUserRepository : NSObject <UserRepositoryProtocol>

- (instancetype)initWithManagedObjectContext:(NSManagedObjectContext *)context;

@end

@implementation CoreDataUserRepository {
    NSManagedObjectContext *_context;
}

- (instancetype)initWithManagedObjectContext:(NSManagedObjectContext *)context {
    self = [super init];
    if (self) {
        _context = context;
    }
    return self;
}

- (void)saveUser:(User *)user 
      completion:(void(^)(BOOL success, NSError *error))completion {
    [_context performBlock:^{
        // สร้าง Core Data entity
        UserEntity *entity = [NSEntityDescription insertNewObjectForEntityForName:@"UserEntity"
                                                           inManagedObjectContext:self->_context];
        entity.userId = user.userId;
        entity.name = user.name;
        entity.email = user.email;
        
        NSError *error;
        BOOL success = [self->_context save:&error];
        
        dispatch_async(dispatch_get_main_queue(), ^{
            completion(success, error);
        });
    }];
}

- (void)getUserById:(NSString *)userId 
         completion:(void(^)(User *user, NSError *error))completion {
    [_context performBlock:^{
        NSFetchRequest *request = [NSFetchRequest fetchRequestWithEntityName:@"UserEntity"];
        request.predicate = [NSPredicate predicateWithFormat:@"userId == %@", userId];
        
        NSError *error;
        NSArray *results = [self->_context executeFetchRequest:request error:&error];
        
        User *user = nil;
        if (results.count > 0) {
            UserEntity *entity = results[0];
            user = [[User alloc] initWithId:entity.userId name:entity.name email:entity.email];
        }
        
        dispatch_async(dispatch_get_main_queue(), ^{
            completion(user, error);
        });
    }];
}

// ... implement other methods

@end

// Implementation สำหรับ In-Memory (สำหรับ Testing)
@interface InMemoryUserRepository : NSObject <UserRepositoryProtocol>
@end

@implementation InMemoryUserRepository {
    NSMutableDictionary<NSString *, User *> *_store;
}

- (instancetype)init {
    self = [super init];
    if (self) {
        _store = [NSMutableDictionary dictionary];
    }
    return self;
}

- (void)saveUser:(User *)user 
      completion:(void(^)(BOOL success, NSError *error))completion {
    _store[user.userId] = user;
    completion(YES, nil);
}

- (void)getUserById:(NSString *)userId 
         completion:(void(^)(User *user, NSError *error))completion {
    completion(_store[userId], nil);
}

// ... implement other methods

@end
```

### Service Layer กับ Protocol-Based DI

```objc
// Service ที่ใช้ Protocol-Based DI
@interface UserService : NSObject

- (instancetype)initWithRepository:(id<UserRepositoryProtocol>)repository
                            logger:(id<LoggerProtocol>)logger
                         validator:(id<UserValidatorProtocol>)validator;

- (void)registerUser:(UserRegistrationDTO *)dto
          completion:(void(^)(User *user, NSError *error))completion;

- (void)updateUserProfile:(UserProfileDTO *)dto
               completion:(void(^)(BOOL success, NSError *error))completion;

@end

@implementation UserService {
    id<UserRepositoryProtocol> _repository;
    id<LoggerProtocol> _logger;
    id<UserValidatorProtocol> _validator;
}

- (instancetype)initWithRepository:(id<UserRepositoryProtocol>)repository
                            logger:(id<LoggerProtocol>)logger
                         validator:(id<UserValidatorProtocol>)validator {
    self = [super init];
    if (self) {
        _repository = repository;
        _logger = logger;
        _validator = validator;
    }
    return self;
}

- (void)registerUser:(UserRegistrationDTO *)dto
          completion:(void(^)(User *user, NSError *error))completion {
    [_logger info:@"Registering new user: %@", dto.email];
    
    // Validate
    NSError *validationError = [_validator validateRegistration:dto];
    if (validationError) {
        [_logger error:@"Validation failed: %@", validationError.localizedDescription];
        completion(nil, validationError);
        return;
    }
    
    // Create user
    User *newUser = [[User alloc] initWithId:[[NSUUID UUID] UUIDString]
                                       name:dto.name
                                      email:dto.email];
    
    [_repository saveUser:newUser completion:^(BOOL success, NSError *error) {
        if (success) {
            [self->_logger info:@"User registered successfully: %@", newUser.userId];
            completion(newUser, nil);
        } else {
            [self->_logger error:@"Failed to save user: %@", error.localizedDescription];
            completion(nil, error);
        }
    }];
}

@end
```

---

## 73.8 Building a Simple DI Container

DI Container (หรือ IoC Container) คือ framework ที่ช่วยจัดการการสร้างและ inject dependencies โดยอัตโนมัติ

### Simple DI Container Implementation

```objc
// DI Container
@interface DIContainer : NSObject

+ (instancetype)sharedContainer;

// ลงทะเบียน service
- (void)registerClass:(Class)implementation 
          forProtocol:(Protocol *)protocol;

- (void)registerInstance:(id)instance 
             forProtocol:(Protocol *)protocol;

- (void)registerFactory:(id(^)(DIContainer *container))factory 
            forProtocol:(Protocol *)protocol;

// สร้าง service
- (id)resolve:(Protocol *)protocol;
- (id)resolveClass:(Class)aClass;

@end

// Registration types
typedef NS_ENUM(NSInteger, DIRegistrationType) {
    DIRegistrationTypeTransient,  // สร้างใหม่ทุกครั้ง
    DIRegistrationTypeSingleton,  // สร้างครั้งเดียว ใช้ซ้ำ
};

// Registration descriptor
@interface DIRegistration : NSObject

@property (nonatomic, assign) DIRegistrationType type;
@property (nonatomic, strong) Class implementationClass;
@property (nonatomic, strong) id instance;
@property (nonatomic, copy) id(^factory)(DIContainer *container);

@end

@implementation DIRegistration
@end

@implementation DIContainer {
    NSMutableDictionary<NSString *, DIRegistration *> *_registrations;
    NSMutableDictionary<NSString *, id> *_singletons;
}

+ (instancetype)sharedContainer {
    static DIContainer *instance;
    static dispatch_once_t onceToken;
    dispatch_once(&onceToken, ^{
        instance = [[DIContainer alloc] init];
    });
    return instance;
}

- (instancetype)init {
    self = [super init];
    if (self) {
        _registrations = [NSMutableDictionary dictionary];
        _singletons = [NSMutableDictionary dictionary];
    }
    return self;
}

- (void)registerClass:(Class)implementation 
          forProtocol:(Protocol *)protocol {
    [self registerClass:implementation 
            forProtocol:protocol 
                   type:DIRegistrationTypeTransient];
}

- (void)registerClass:(Class)implementation 
          forProtocol:(Protocol *)protocol 
                 type:(DIRegistrationType)type {
    DIRegistration *reg = [[DIRegistration alloc] init];
    reg.implementationClass = implementation;
    reg.type = type;
    
    NSString *key = NSStringFromProtocol(protocol);
    _registrations[key] = reg;
}

- (void)registerInstance:(id)instance 
             forProtocol:(Protocol *)protocol {
    DIRegistration *reg = [[DIRegistration alloc] init];
    reg.instance = instance;
    reg.type = DIRegistrationTypeSingleton;
    
    NSString *key = NSStringFromProtocol(protocol);
    _registrations[key] = reg;
    _singletons[key] = instance;
}

- (void)registerFactory:(id(^)(DIContainer *container))factory 
            forProtocol:(Protocol *)protocol {
    DIRegistration *reg = [[DIRegistration alloc] init];
    reg.factory = factory;
    reg.type = DIRegistrationTypeTransient;
    
    NSString *key = NSStringFromProtocol(protocol);
    _registrations[key] = reg;
}

- (id)resolve:(Protocol *)protocol {
    NSString *key = NSStringFromProtocol(protocol);
    DIRegistration *reg = _registrations[key];
    
    if (!reg) {
        NSLog(@"Warning: No registration found for protocol: %@", key);
        return nil;
    }
    
    // Singleton check
    if (reg.type == DIRegistrationTypeSingleton) {
        id existing = _singletons[key];
        if (existing) {
            return existing;
        }
    }
    
    // Create instance
    id instance;
    if (reg.instance) {
        instance = reg.instance;
    } else if (reg.factory) {
        instance = reg.factory(self);
    } else if (reg.implementationClass) {
        instance = [[reg.implementationClass alloc] init];
    }
    
    // Cache singleton
    if (reg.type == DIRegistrationTypeSingleton && instance) {
        _singletons[key] = instance;
    }
    
    return instance;
}

@end

// การใช้งาน DI Container
void setupDIContainer() {
    DIContainer *container = [DIContainer sharedContainer];
    
    // Register services
    [container registerClass:[CoreDataUserRepository class] 
                 forProtocol:@protocol(UserRepositoryProtocol)];
    
    [container registerFactory:^id(DIContainer *c) {
        id<UserRepositoryProtocol> repo = [c resolve:@protocol(UserRepositoryProtocol)];
        id<LoggerProtocol> logger = [c resolve:@protocol(LoggerProtocol)];
        id<UserValidatorProtocol> validator = [c resolve:@protocol(UserValidatorProtocol)];
        return [[UserService alloc] initWithRepository:repo
                                               logger:logger
                                            validator:validator];
    } forProtocol:@protocol(UserServiceProtocol)];
    
    [container registerClass:[ConsoleLogger class] 
                 forProtocol:@protocol(LoggerProtocol)];
    
    [container registerClass:[DefaultUserValidator class] 
                 forProtocol:@protocol(UserValidatorProtocol)];
}

// ใช้ใน AppDelegate
- (BOOL)application:(UIApplication *)application 
didFinishLaunchingWithOptions:(NSDictionary *)launchOptions {
    setupDIContainer();
    
    // Resolve services
    id<UserServiceProtocol> userService = [[DIContainer sharedContainer] 
                                           resolve:@protocol(UserServiceProtocol)];
    
    // ใช้งาน service
    UserRegistrationDTO *dto = [[UserRegistrationDTO alloc] init];
    dto.name = @"John Doe";
    dto.email = @"john@example.com";
    
    [userService registerUser:dto completion:^(User *user, NSError *error) {
        if (user) {
            NSLog(@"User registered: %@", user.userId);
        }
    }];
    
    return YES;
}
```

---

## 73.9 Testing กับ Dependency Injection

หนึ่งในประโยชน์หลักของ DI คือทำให้การทดสอบง่ายขึ้นมาก เพราะสามารถส่ง mock objects เข้าไปแทน real dependencies

### สร้าง Mock Objects

```objc
// Mock Database สำหรับ Testing
@interface MockDatabase : NSObject <DatabaseServiceProtocol>

@property (nonatomic, strong) NSMutableArray *savedItems;
@property (nonatomic, assign) BOOL shouldFailOnSave;
@property (nonatomic, strong) NSError *errorToReturn;
@property (nonatomic, assign) NSInteger saveCallCount;

@end

@implementation MockDatabase

- (instancetype)init {
    self = [super init];
    if (self) {
        _savedItems = [NSMutableArray array];
        _shouldFailOnSave = NO;
        _saveCallCount = 0;
    }
    return self;
}

- (BOOL)save:(id)entity {
    _saveCallCount++;
    
    if (_shouldFailOnSave) {
        return NO;
    }
    
    [_savedItems addObject:entity];
    return YES;
}

- (id)findById:(NSString *)entityId {
    for (id item in _savedItems) {
        if ([[item valueForKey:@"id"] isEqualToString:entityId]) {
            return item;
        }
    }
    return nil;
}

- (NSArray *)findAll {
    return [_savedItems copy];
}

- (BOOL)delete:(NSString *)entityId {
    id toRemove = [self findById:entityId];
    if (toRemove) {
        [_savedItems removeObject:toRemove];
        return YES;
    }
    return NO;
}

- (BOOL)update:(id)entity {
    NSString *entityId = [entity valueForKey:@"id"];
    id existing = [self findById:entityId];
    if (existing) {
        NSInteger index = [_savedItems indexOfObject:existing];
        _savedItems[index] = entity;
        return YES;
    }
    return NO;
}

@end

// Unit Tests โดยใช้ DI
#import <XCTest/XCTest.h>

@interface UserServiceTests : XCTestCase

@property (nonatomic, strong) UserService *userService;
@property (nonatomic, strong) MockDatabase *mockDatabase;
@property (nonatomic, strong) MockLogger *mockLogger;
@property (nonatomic, strong) MockValidator *mockValidator;

@end

@implementation UserServiceTests

- (void)setUp {
    [super setUp];
    
    // สร้าง mocks
    self.mockDatabase = [[MockDatabase alloc] init];
    self.mockLogger = [[MockLogger alloc] init];
    self.mockValidator = [[MockValidator alloc] init];
    
    // Inject mocks เข้า service
    self.userService = [[UserService alloc] 
                        initWithRepository:self.mockDatabase
                                   logger:self.mockLogger
                                validator:self.mockValidator];
}

- (void)tearDown {
    self.userService = nil;
    self.mockDatabase = nil;
    self.mockLogger = nil;
    self.mockValidator = nil;
    [super tearDown];
}

- (void)testRegisterUser_WithValidData_SavesUser {
    // Arrange
    UserRegistrationDTO *dto = [[UserRegistrationDTO alloc] init];
    dto.name = @"John Doe";
    dto.email = @"john@example.com";
    
    self.mockValidator.validationResult = nil; // No error = valid
    
    // Act
    __block User *registeredUser;
    __block NSError *registrationError;
    
    XCTestExpectation *expectation = [self expectationWithDescription:@"Registration complete"];
    
    [self.userService registerUser:dto completion:^(User *user, NSError *error) {
        registeredUser = user;
        registrationError = error;
        [expectation fulfill];
    }];
    
    [self waitForExpectationsWithTimeout:1.0 handler:nil];
    
    // Assert
    XCTAssertNotNil(registeredUser);
    XCTAssertNil(registrationError);
    XCTAssertEqual(self.mockDatabase.saveCallCount, 1);
    XCTAssertEqual(self.mockDatabase.savedItems.count, 1);
}

- (void)testRegisterUser_WithInvalidEmail_ReturnsError {
    // Arrange
    UserRegistrationDTO *dto = [[UserRegistrationDTO alloc] init];
    dto.name = @"John Doe";
    dto.email = @"invalid-email";
    
    NSError *validationError = [NSError errorWithDomain:@"ValidationError" 
                                                    code:400 
                                                userInfo:@{NSLocalizedDescriptionKey: @"Invalid email"}];
    self.mockValidator.validationResult = validationError;
    
    // Act
    __block NSError *registrationError;
    
    XCTestExpectation *expectation = [self expectationWithDescription:@"Registration complete"];
    
    [self.userService registerUser:dto completion:^(User *user, NSError *error) {
        registrationError = error;
        [expectation fulfill];
    }];
    
    [self waitForExpectationsWithTimeout:1.0 handler:nil];
    
    // Assert
    XCTAssertNotNil(registrationError);
    XCTAssertEqual(self.mockDatabase.saveCallCount, 0); // ไม่ควร save
}

- (void)testRegisterUser_WhenDatabaseFails_ReturnsError {
    // Arrange
    UserRegistrationDTO *dto = [[UserRegistrationDTO alloc] init];
    dto.name = @"John Doe";
    dto.email = @"john@example.com";
    
    self.mockValidator.validationResult = nil;
    self.mockDatabase.shouldFailOnSave = YES; // จำลอง database error
    
    // Act
    __block NSError *registrationError;
    
    XCTestExpectation *expectation = [self expectationWithDescription:@"Registration complete"];
    
    [self.userService registerUser:dto completion:^(User *user, NSError *error) {
        registrationError = error;
        [expectation fulfill];
    }];
    
    [self waitForExpectationsWithTimeout:1.0 handler:nil];
    
    // Assert
    XCTAssertNotNil(registrationError);
}

@end
```

---

## 73.10 Practical Examples: Building a Weather App

ตัวอย่างการใช้ DI ในการสร้าง Weather App จริง ๆ

```objc
// Protocols
@protocol WeatherAPIProtocol <NSObject>
- (void)fetchWeatherForCity:(NSString *)city 
                 completion:(void(^)(WeatherData *data, NSError *error))completion;
@end

@protocol LocationServiceProtocol <NSObject>
- (void)getCurrentLocation:(void(^)(CLLocation *location, NSError *error))completion;
@end

@protocol WeatherCacheProtocol <NSObject>
- (void)cacheWeather:(WeatherData *)data forCity:(NSString *)city;
- (WeatherData *)cachedWeatherForCity:(NSString *)city;
- (BOOL)isCacheValidForCity:(NSString *)city;
@end

// Real implementations
@interface OpenWeatherMapAPI : NSObject <WeatherAPIProtocol>
- (instancetype)initWithAPIKey:(NSString *)apiKey;
@end

@implementation OpenWeatherMapAPI {
    NSString *_apiKey;
    NSURLSession *_session;
}

- (instancetype)initWithAPIKey:(NSString *)apiKey {
    self = [super init];
    if (self) {
        _apiKey = apiKey;
        _session = [NSURLSession sharedSession];
    }
    return self;
}

- (void)fetchWeatherForCity:(NSString *)city 
                 completion:(void(^)(WeatherData *data, NSError *error))completion {
    NSString *urlString = [NSString stringWithFormat:
                           @"https://api.openweathermap.org/data/2.5/weather?q=%@&appid=%@",
                           city, _apiKey];
    
    NSURL *url = [NSURL URLWithString:[urlString stringByAddingPercentEncodingWithAllowedCharacters:[NSCharacterSet URLQueryAllowedCharacterSet]]];
    
    [[_session dataTaskWithURL:url completionHandler:^(NSData *data, NSURLResponse *response, NSError *error) {
        if (error) {
            completion(nil, error);
            return;
        }
        
        NSError *parseError;
        NSDictionary *json = [NSJSONSerialization JSONObjectWithData:data options:0 error:&parseError];
        
        if (parseError) {
            completion(nil, parseError);
            return;
        }
        
        WeatherData *weatherData = [WeatherData weatherFromDictionary:json];
        completion(weatherData, nil);
    }] resume];
}

@end

// Weather ViewModel ที่ใช้ DI
@interface WeatherViewModel : NSObject

- (instancetype)initWithWeatherAPI:(id<WeatherAPIProtocol>)api
                   locationService:(id<LocationServiceProtocol>)locationService
                             cache:(id<WeatherCacheProtocol>)cache;

- (void)loadWeather:(void(^)(WeatherData *data, NSError *error))completion;
- (void)loadWeatherForCity:(NSString *)city 
               completion:(void(^)(WeatherData *data, NSError *error))completion;

@end

@implementation WeatherViewModel {
    id<WeatherAPIProtocol> _api;
    id<LocationServiceProtocol> _locationService;
    id<WeatherCacheProtocol> _cache;
}

- (instancetype)initWithWeatherAPI:(id<WeatherAPIProtocol>)api
                   locationService:(id<LocationServiceProtocol>)locationService
                             cache:(id<WeatherCacheProtocol>)cache {
    self = [super init];
    if (self) {
        _api = api;
        _locationService = locationService;
        _cache = cache;
    }
    return self;
}

- (void)loadWeather:(void(^)(WeatherData *data, NSError *error))completion {
    [_locationService getCurrentLocation:^(CLLocation *location, NSError *error) {
        if (error) {
            completion(nil, error);
            return;
        }
        
        // ใช้ location เพื่อหาชื่อ city
        NSString *city = [self cityNameFromLocation:location];
        [self loadWeatherForCity:city completion:completion];
    }];
}

- (void)loadWeatherForCity:(NSString *)city 
               completion:(void(^)(WeatherData *data, NSError *error))completion {
    // ตรวจสอบ cache ก่อน
    if ([_cache isCacheValidForCity:city]) {
        WeatherData *cached = [_cache cachedWeatherForCity:city];
        completion(cached, nil);
        return;
    }
    
    // Fetch จาก API
    [_api fetchWeatherForCity:city completion:^(WeatherData *data, NSError *error) {
        if (data) {
            [self->_cache cacheWeather:data forCity:city];
        }
        completion(data, error);
    }];
}

- (NSString *)cityNameFromLocation:(CLLocation *)location {
    // Simplified - ในจริงจะใช้ CLGeocoder
    return @"Bangkok";
}

@end

// App setup
void setupWeatherApp() {
    // Production setup
    OpenWeatherMapAPI *api = [[OpenWeatherMapAPI alloc] initWithAPIKey:@"YOUR_API_KEY"];
    CLLocationManager *locationManager = [[CLLocationManager alloc] init];
    CoreDataWeatherCache *cache = [[CoreDataWeatherCache alloc] init];
    
    WeatherViewModel *viewModel = [[WeatherViewModel alloc] 
                                   initWithWeatherAPI:api
                                      locationService:locationManager
                                                cache:cache];
    
    // Use viewModel...
}

void setupWeatherAppForTesting() {
    // Test setup - ใช้ mocks
    MockWeatherAPI *mockAPI = [[MockWeatherAPI alloc] init];
    MockLocationService *mockLocation = [[MockLocationService alloc] init];
    MockWeatherCache *mockCache = [[MockWeatherCache alloc] init];
    
    // Configure mocks
    mockAPI.weatherToReturn = [WeatherData testWeather];
    mockLocation.locationToReturn = [[CLLocation alloc] initWithLatitude:13.7 longitude:100.5];
    
    WeatherViewModel *viewModel = [[WeatherViewModel alloc] 
                                   initWithWeatherAPI:mockAPI
                                      locationService:mockLocation
                                                cache:mockCache];
    
    // Test viewModel...
}
```

---

## 73.11 แบบฝึกหัด

### แบบฝึกหัดที่ 1: ออกแบบ Protocol สำหรับ Shopping Cart

สร้าง protocol และ implementation สำหรับ:
- `ProductRepositoryProtocol` - จัดการข้อมูลสินค้า
- `CartServiceProtocol` - จัดการ shopping cart
- `PricingServiceProtocol` - คำนวณราคา รวมส่วนลด
- `PaymentServiceProtocol` - ประมวลผลการชำระเงิน

```objc
// เริ่มต้นด้วย...
@protocol ProductRepositoryProtocol <NSObject>
// TODO: เพิ่ม methods
@end

@protocol CartServiceProtocol <NSObject>
// TODO: เพิ่ม methods
@end

// TODO: Implement CartViewController ที่ใช้ DI
@interface CartViewController : UIViewController
- (instancetype)initWithCartService:(id<CartServiceProtocol>)cartService
                    pricingService:(id<PricingServiceProtocol>)pricingService;
@end
```

### แบบฝึกหัดที่ 2: สร้าง DI Container ที่สมบูรณ์

ขยาย DI Container ที่เราสร้างไว้ให้รองรับ:
1. Singleton scope
2. Transient scope
3. Named registrations (ลงทะเบียนหลาย implementation สำหรับ protocol เดียวกัน)
4. Circular dependency detection

```objc
// TODO: ขยาย DIContainer
@interface DIContainer : NSObject

- (void)registerClass:(Class)implementation 
          forProtocol:(Protocol *)protocol 
                scope:(DIScope)scope;

- (void)registerClass:(Class)implementation 
          forProtocol:(Protocol *)protocol 
                 name:(NSString *)name;

- (id)resolve:(Protocol *)protocol;
- (id)resolve:(Protocol *)protocol name:(NSString *)name;

@end
```

### แบบฝึกหัดที่ 3: Refactor Legacy Code

โค้ดต่อไปนี้ไม่ได้ใช้ DI ให้ทำการ refactor:

```objc
// Legacy code - ต้องทำการ refactor
@implementation NewsViewController

- (void)viewDidLoad {
    [super viewDidLoad];
    
    // ❌ ปัญหา: สร้าง dependencies ภายใน
    NewsAPI *api = [[NewsAPI alloc] initWithBaseURL:@"https://api.news.com"];
    CoreDataCache *cache = [[CoreDataCache alloc] init];
    
    [api fetchLatestNews:^(NSArray *articles) {
        [cache saveArticles:articles];
        [self.tableView reloadData];
    }];
}

@end

// TODO: Refactor ให้ใช้ DI
```

### แบบฝึกหัดที่ 4: Test ด้วย Mock Objects

เขียน unit tests สำหรับ `NotificationService` โดยใช้ mock objects:

```objc
@protocol NotificationSenderProtocol <NSObject>
- (void)sendPushNotification:(NSString *)message toUser:(NSString *)userId;
- (void)sendEmail:(NSString *)subject body:(NSString *)body to:(NSString *)email;
- (void)sendSMS:(NSString *)message to:(NSString *)phoneNumber;
@end

@interface NotificationService : NSObject
- (instancetype)initWithSender:(id<NotificationSenderProtocol>)sender;
- (void)notifyUser:(User *)user aboutOrder:(Order *)order;
@end

// TODO: เขียน MockNotificationSender
// TODO: เขียน tests สำหรับ NotificationService
```

---

## สรุป

Dependency Injection เป็นหลักการที่สำคัญมากในการเขียนโค้ดที่:
- **ทดสอบได้ง่าย**: ส่ง mock objects เข้าไปแทน real dependencies
- **ยืดหยุ่น**: เปลี่ยน implementation ได้โดยไม่กระทบ code ที่ใช้งาน
- **อ่านเข้าใจง่าย**: dependencies ชัดเจนจาก constructor/property
- **ดูแลรักษาง่าย**: แก้ไขในที่เดียว กระทบน้อยที่สุด

ในบทต่อไป (Part 74) เราจะเรียนรู้เรื่อง Testing ด้วย XCTest อย่างละเอียด ซึ่งจะสัมพันธ์กับ DI ที่เราเรียนไปในบทนี้อย่างมาก

---

*จบ Part 73 - Dependency Injection*
