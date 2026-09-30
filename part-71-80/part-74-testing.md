# Part 74: Testing ใน Objective-C ด้วย XCTest

## บทนำ

การทดสอบซอฟต์แวร์ (Software Testing) คือหัวใจสำคัญของการพัฒนาซอฟต์แวร์ที่มีคุณภาพ XCTest เป็น framework การทดสอบที่ Apple จัดให้มาพร้อมกับ Xcode ซึ่งรองรับทั้ง Unit Testing, UI Testing, และ Performance Testing บทนี้จะพาคุณเรียนรู้การใช้งาน XCTest อย่างครบถ้วน

---

## 74.1 Unit Testing กับ XCTest

### ทำไมต้องเขียน Unit Tests?

Unit Tests ช่วยให้:
1. **ตรวจจับ bug ได้เร็ว**: รู้ทันทีเมื่อ code เปลี่ยนแล้วทำให้อะไรพัง
2. **เป็น documentation**: Tests บอกวิธีใช้งาน class/method ที่ถูกต้อง
3. **Refactoring ได้อย่างมั่นใจ**: มั่นใจว่า refactor แล้วยังทำงานได้ถูกต้อง
4. **ออกแบบดีขึ้น**: การเขียน test บังคับให้ออกแบบ API ที่ testable

### โครงสร้างของ XCTest Project

```
MyApp/
├── MyApp/              (Source code)
│   ├── Models/
│   ├── ViewControllers/
│   └── Services/
├── MyAppTests/         (Unit Tests)
│   ├── ModelTests/
│   ├── ViewControllerTests/
│   └── ServiceTests/
└── MyAppUITests/       (UI Tests)
    └── MyAppUITests.m
```

### การเพิ่ม Test Target

1. เปิด Xcode project
2. ไปที่ File > New > Target
3. เลือก "Unit Testing Bundle"
4. ตั้งชื่อ และ target ที่ต้องการ test

---

## 74.2 XCTestCase

XCTestCase คือ base class สำหรับ test ทุก test class ต้อง subclass จาก XCTestCase

### โครงสร้างพื้นฐาน

```objc
#import <XCTest/XCTest.h>
#import "Calculator.h"

// Test class ต้อง subclass XCTestCase
@interface CalculatorTests : XCTestCase

// Properties สำหรับ test fixtures
@property (nonatomic, strong) Calculator *calculator;

@end

@implementation CalculatorTests

// setUp เรียกก่อน test method ทุก method
- (void)setUp {
    [super setUp];
    self.calculator = [[Calculator alloc] init];
}

// tearDown เรียกหลัง test method ทุก method
- (void)tearDown {
    self.calculator = nil;
    [super tearDown];
}

// Test methods ต้องขึ้นต้นด้วย "test"
- (void)testAddition {
    NSInteger result = [self.calculator add:2 to:3];
    XCTAssertEqual(result, 5);
}

- (void)testSubtraction {
    NSInteger result = [self.calculator subtract:3 from:10];
    XCTAssertEqual(result, 7);
}

- (void)testMultiplication {
    NSInteger result = [self.calculator multiply:4 by:5];
    XCTAssertEqual(result, 20);
}

- (void)testDivision {
    double result = [self.calculator divide:10 by:4];
    XCTAssertEqualWithAccuracy(result, 2.5, 0.001);
}

- (void)testDivisionByZero {
    XCTAssertThrows([self.calculator divide:10 by:0]);
}

@end
```

---

## 74.3 Test Methods (testXxx Naming Convention)

### กฎการตั้งชื่อ Test Method

```objc
// รูปแบบ: test + [UnitOfWork] + [Scenario] + [ExpectedBehavior]
// หรือ: test + [MethodName] + [Condition] + [Result]

// ตัวอย่าง
- (void)testAdd_WithPositiveNumbers_ReturnsCorrectSum { }
- (void)testAdd_WithNegativeNumbers_ReturnsCorrectSum { }
- (void)testAdd_WithZero_ReturnsSameNumber { }
- (void)testDivide_ByZero_ThrowsException { }
- (void)testSave_WithValidData_ReturnsTrue { }
- (void)testSave_WithDuplicateEmail_ReturnsFalse { }
```

### โครงสร้าง Arrange-Act-Assert (AAA Pattern)

```objc
- (void)testRegisterUser_WithValidData_CreatesUserSuccessfully {
    // Arrange - เตรียมข้อมูล
    UserService *service = [[UserService alloc] initWithDatabase:[[MockDatabase alloc] init]];
    UserRegistrationForm *form = [[UserRegistrationForm alloc] init];
    form.name = @"Jane Doe";
    form.email = @"jane@example.com";
    form.password = @"SecurePass123!";
    
    // Act - เรียก method ที่ต้องการทดสอบ
    User *result = [service registerUser:form error:nil];
    
    // Assert - ตรวจสอบผลลัพธ์
    XCTAssertNotNil(result, @"User should be created");
    XCTAssertEqualObjects(result.name, @"Jane Doe");
    XCTAssertEqualObjects(result.email, @"jane@example.com");
    XCTAssertNotNil(result.userId, @"User should have an ID");
}
```

### Given-When-Then (GWT Pattern)

```objc
- (void)testShoppingCart_WhenItemAdded_TotalIncreases {
    // Given - สถานการณ์เริ่มต้น
    ShoppingCart *cart = [[ShoppingCart alloc] init];
    Product *product = [[Product alloc] initWithName:@"Apple" price:1.50];
    
    // When - การกระทำ
    [cart addProduct:product quantity:3];
    
    // Then - ผลที่คาดหวัง
    XCTAssertEqual(cart.totalItems, 3);
    XCTAssertEqualWithAccuracy(cart.totalPrice, 4.50, 0.001);
}
```

---

## 74.4 Assertions (XCTAssert Functions)

XCTest มี assertion functions หลายประเภทสำหรับการตรวจสอบผลลัพธ์

### Basic Assertions

```objc
// XCTAssert - ตรวจสอบว่า expression เป็น true
XCTAssert(condition);
XCTAssert(condition, @"Custom failure message");

// XCTAssertTrue / XCTAssertFalse
XCTAssertTrue(value == 5);
XCTAssertFalse(array.isEmpty);

// XCTAssertNil / XCTAssertNotNil
XCTAssertNil(result);
XCTAssertNotNil(object);

// XCTFail - บังคับให้ test fail
XCTFail(@"This test should not reach here");
```

### Equality Assertions

```objc
// XCTAssertEqual - สำหรับ primitive types (int, float, bool, etc.)
XCTAssertEqual(result, 42);
XCTAssertEqual(count, 0);

// XCTAssertEqualObjects - สำหรับ Objective-C objects (ใช้ isEqual:)
XCTAssertEqualObjects(string1, string2);
XCTAssertEqualObjects(array1, array2);
XCTAssertEqualObjects(dict1, dict2);

// XCTAssertNotEqual / XCTAssertNotEqualObjects
XCTAssertNotEqual(result, 0);
XCTAssertNotEqualObjects(object1, object2);

// XCTAssertEqualWithAccuracy - สำหรับ floating point (มี tolerance)
XCTAssertEqualWithAccuracy(3.14159, M_PI, 0.00001);

// XCTAssertGreaterThan, XCTAssertLessThan, etc.
XCTAssertGreaterThan(result, 0);
XCTAssertLessThan(result, 100);
XCTAssertGreaterThanOrEqual(result, 0);
XCTAssertLessThanOrEqual(result, 100);
```

### Exception Assertions

```objc
// XCTAssertThrows - ตรวจสอบว่า code throws exception
XCTAssertThrows([object methodThatShouldThrow]);

// XCTAssertThrowsSpecific - ตรวจสอบ exception type ที่ระบุ
XCTAssertThrowsSpecific([object method], NSException);

// XCTAssertThrowsSpecificNamed - ตรวจสอบ exception name
XCTAssertThrowsSpecificNamed([object method], 
                              NSException, 
                              NSRangeException);

// XCTAssertNoThrow - ตรวจสอบว่าไม่ throw exception
XCTAssertNoThrow([object safeMethod]);
```

### ตัวอย่างการใช้ Assertions อย่างครอบคลุม

```objc
@interface StringUtilsTests : XCTestCase
@end

@implementation StringUtilsTests

- (void)testTrimWhitespace {
    NSString *input = @"  Hello World  ";
    NSString *result = [StringUtils trim:input];
    
    XCTAssertNotNil(result);
    XCTAssertEqualObjects(result, @"Hello World");
    XCTAssertFalse([result hasPrefix:@" "]);
    XCTAssertFalse([result hasSuffix:@" "]);
}

- (void)testValidateEmail {
    // Valid emails
    XCTAssertTrue([StringUtils isValidEmail:@"user@example.com"]);
    XCTAssertTrue([StringUtils isValidEmail:@"user.name@domain.co.th"]);
    XCTAssertTrue([StringUtils isValidEmail:@"user+tag@example.org"]);
    
    // Invalid emails
    XCTAssertFalse([StringUtils isValidEmail:@"notanemail"]);
    XCTAssertFalse([StringUtils isValidEmail:@"@nodomain.com"]);
    XCTAssertFalse([StringUtils isValidEmail:@"noatsign"]);
    XCTAssertFalse([StringUtils isValidEmail:nil]);
    XCTAssertFalse([StringUtils isValidEmail:@""]);
}

- (void)testParseCurrency {
    XCTAssertEqualWithAccuracy([StringUtils parseCurrency:@"$1,234.56"], 1234.56, 0.001);
    XCTAssertEqualWithAccuracy([StringUtils parseCurrency:@"฿100.00"], 100.0, 0.001);
    
    // Edge cases
    XCTAssertEqualWithAccuracy([StringUtils parseCurrency:@"0"], 0.0, 0.001);
    XCTAssertEqualWithAccuracy([StringUtils parseCurrency:@"$0.00"], 0.0, 0.001);
}

- (void)testParseInvalidCurrency {
    XCTAssertThrows([StringUtils parseCurrency:@"not-a-number"]);
    XCTAssertThrows([StringUtils parseCurrency:nil]);
}

@end
```

---

## 74.5 setUp และ tearDown

setUp และ tearDown ช่วยจัดการ test fixtures เพื่อไม่ให้ code ซ้ำซ้อน

### Instance Setup/Teardown

```objc
@interface DatabaseTests : XCTestCase

@property (nonatomic, strong) TestDatabase *database;
@property (nonatomic, strong) NSManagedObjectContext *context;

@end

@implementation DatabaseTests

// เรียกก่อน test method ทุก method
- (void)setUp {
    [super setUp];
    
    // สร้าง in-memory database สำหรับ testing
    self.database = [[TestDatabase alloc] initWithInMemoryStore];
    self.context = self.database.mainContext;
    
    // เพิ่ม test data
    [self seedTestData];
}

// เรียกหลัง test method ทุก method
- (void)tearDown {
    // ล้าง test data
    [self database cleanAll];
    self.context = nil;
    self.database = nil;
    
    [super tearDown];
}

- (void)seedTestData {
    // สร้าง test users
    for (int i = 0; i < 5; i++) {
        UserEntity *user = [NSEntityDescription insertNewObjectForEntityForName:@"UserEntity"
                                                       inManagedObjectContext:self.context];
        user.userId = [NSString stringWithFormat:@"user-%d", i];
        user.name = [NSString stringWithFormat:@"Test User %d", i];
        user.email = [NSString stringWithFormat:@"user%d@test.com", i];
    }
    
    NSError *error;
    [self.context save:&error];
}

- (void)testFetchAllUsers {
    NSArray *users = [self.database fetchAllUsers];
    XCTAssertEqual(users.count, 5);
}

- (void)testFetchUserById {
    UserEntity *user = [self.database fetchUserById:@"user-0"];
    XCTAssertNotNil(user);
    XCTAssertEqualObjects(user.name, @"Test User 0");
}

@end
```

### Class Setup/Teardown (setUpWithError/tearDownWithError)

```objc
@interface APIIntegrationTests : XCTestCase

@property (nonatomic, strong) MockServer *server;
@property (nonatomic, strong) APIClient *client;

@end

@implementation APIIntegrationTests

// เรียกครั้งเดียวก่อน test ทั้งหมดใน class
+ (void)setUp {
    [super setUp];
    // Setup ที่ expensive ที่ทำครั้งเดียวได้
    NSLog(@"Setting up APIIntegrationTests class");
}

// เรียกครั้งเดียวหลัง test ทั้งหมดใน class
+ (void)tearDown {
    NSLog(@"Tearing down APIIntegrationTests class");
    [super tearDown];
}

// setUpWithError - รองรับ error handling (iOS 14+)
- (void)setUpWithError:(NSError **)error {
    [super setUpWithError:error];
    
    self.server = [[MockServer alloc] init];
    NSError *serverError;
    [self.server startOnPort:8080 error:&serverError];
    
    if (serverError) {
        *error = serverError;
        return;
    }
    
    self.client = [[APIClient alloc] initWithBaseURL:@"http://localhost:8080"];
}

// tearDownWithError
- (void)tearDownWithError:(NSError **)error {
    [self.server stop];
    self.server = nil;
    self.client = nil;
    
    [super tearDownWithError:error];
}

@end
```

### Async Setup/Teardown

```objc
@implementation AsyncServiceTests

- (void)setUp {
    [super setUp];
    
    self.service = [[AsyncService alloc] init];
    
    XCTestExpectation *expectation = [self expectationWithDescription:@"Service ready"];
    
    [self.service prepareForTesting:^{
        [expectation fulfill];
    }];
    
    [self waitForExpectationsWithTimeout:5.0 handler:nil];
}

@end
```

---

## 74.6 Test Doubles: Mocks, Stubs, Fakes

Test doubles คือ objects ที่ใช้แทน real objects ในการทดสอบ

### ประเภทของ Test Doubles

**Stub**: ส่งคืน hard-coded values
**Mock**: ตรวจสอบว่า methods ถูกเรียกอย่างถูกต้อง
**Fake**: มี working implementation แต่ simplified
**Spy**: เหมือน mock แต่ยัง delegate ไป real implementation

### Stub Implementation

```objc
// Stub - ส่งคืน predetermined values
@interface StubWeatherAPI : NSObject <WeatherAPIProtocol>

@property (nonatomic, strong) WeatherData *weatherToReturn;
@property (nonatomic, strong) NSError *errorToReturn;

@end

@implementation StubWeatherAPI

- (void)fetchWeatherForCity:(NSString *)city 
                 completion:(void(^)(WeatherData *data, NSError *error))completion {
    // ส่งคืนค่าที่กำหนดไว้ล่วงหน้า
    completion(self.weatherToReturn, self.errorToReturn);
}

@end

// การใช้งาน Stub
- (void)testLoadWeather_DisplaysTemperature {
    // Arrange
    StubWeatherAPI *stubAPI = [[StubWeatherAPI alloc] init];
    WeatherData *fakeWeather = [[WeatherData alloc] init];
    fakeWeather.temperature = 25.0;
    fakeWeather.cityName = @"Bangkok";
    stubAPI.weatherToReturn = fakeWeather;
    
    WeatherViewModel *viewModel = [[WeatherViewModel alloc] initWithAPI:stubAPI];
    
    // Act
    [viewModel loadWeatherForCity:@"Bangkok" completion:^(WeatherData *data, NSError *error) {
        // Assert
        XCTAssertEqualWithAccuracy(data.temperature, 25.0, 0.001);
        XCTAssertEqualObjects(data.cityName, @"Bangkok");
    }];
}
```

### Mock Implementation

```objc
// Mock - ตรวจสอบ interaction
@interface MockEmailSender : NSObject <EmailSenderProtocol>

// Tracking
@property (nonatomic, assign) NSInteger sendEmailCallCount;
@property (nonatomic, strong) NSMutableArray<NSString *> *sentToAddresses;
@property (nonatomic, strong) NSString *lastSubject;
@property (nonatomic, strong) NSString *lastBody;

// Configuration
@property (nonatomic, assign) BOOL shouldSucceed;

@end

@implementation MockEmailSender

- (instancetype)init {
    self = [super init];
    if (self) {
        _sendEmailCallCount = 0;
        _sentToAddresses = [NSMutableArray array];
        _shouldSucceed = YES;
    }
    return self;
}

- (BOOL)sendEmail:(NSString *)to subject:(NSString *)subject body:(NSString *)body {
    _sendEmailCallCount++;
    [_sentToAddresses addObject:to];
    _lastSubject = subject;
    _lastBody = body;
    return _shouldSucceed;
}

@end

// การใช้งาน Mock
- (void)testRegistration_SendsWelcomeEmail {
    // Arrange
    MockEmailSender *mockEmail = [[MockEmailSender alloc] init];
    UserService *service = [[UserService alloc] initWithEmailSender:mockEmail];
    
    UserRegistrationForm *form = [[UserRegistrationForm alloc] init];
    form.email = @"new@user.com";
    form.name = @"New User";
    
    // Act
    [service registerUser:form error:nil];
    
    // Assert - ตรวจสอบ interaction
    XCTAssertEqual(mockEmail.sendEmailCallCount, 1, @"Should send exactly one email");
    XCTAssertEqualObjects([mockEmail.sentToAddresses firstObject], @"new@user.com");
    XCTAssertTrue([mockEmail.lastSubject containsString:@"Welcome"]);
}
```

### Fake Implementation

```objc
// Fake - Working implementation ที่ simplified (in-memory storage)
@interface FakeUserRepository : NSObject <UserRepositoryProtocol>

@property (nonatomic, strong, readonly) NSArray<User *> *allUsers;

@end

@implementation FakeUserRepository {
    NSMutableDictionary<NSString *, User *> *_storage;
}

- (instancetype)init {
    self = [super init];
    if (self) {
        _storage = [NSMutableDictionary dictionary];
    }
    return self;
}

- (NSArray<User *> *)allUsers {
    return [_storage allValues];
}

- (void)saveUser:(User *)user completion:(void(^)(BOOL, NSError *))completion {
    _storage[user.userId] = user;
    completion(YES, nil);
}

- (void)getUserById:(NSString *)userId completion:(void(^)(User *, NSError *))completion {
    completion(_storage[userId], nil);
}

- (void)getAllUsers:(void(^)(NSArray<User *> *, NSError *))completion {
    completion(self.allUsers, nil);
}

- (void)updateUser:(User *)user completion:(void(^)(BOOL, NSError *))completion {
    if (_storage[user.userId]) {
        _storage[user.userId] = user;
        completion(YES, nil);
    } else {
        NSError *error = [NSError errorWithDomain:@"NotFound" code:404 userInfo:nil];
        completion(NO, error);
    }
}

- (void)deleteUser:(NSString *)userId completion:(void(^)(BOOL, NSError *))completion {
    if (_storage[userId]) {
        [_storage removeObjectForKey:userId];
        completion(YES, nil);
    } else {
        NSError *error = [NSError errorWithDomain:@"NotFound" code:404 userInfo:nil];
        completion(NO, error);
    }
}

@end
```

### Spy Implementation

```objc
// Spy - Delegates to real implementation แต่ track calls
@interface SpyLogger : NSObject <LoggerProtocol>

@property (nonatomic, strong) id<LoggerProtocol> realLogger;
@property (nonatomic, strong) NSMutableArray<NSString *> *loggedMessages;
@property (nonatomic, assign) NSInteger infoCallCount;
@property (nonatomic, assign) NSInteger errorCallCount;
@property (nonatomic, assign) NSInteger warningCallCount;

- (instancetype)initWithRealLogger:(id<LoggerProtocol>)logger;

@end

@implementation SpyLogger

- (instancetype)initWithRealLogger:(id<LoggerProtocol>)logger {
    self = [super init];
    if (self) {
        _realLogger = logger;
        _loggedMessages = [NSMutableArray array];
    }
    return self;
}

- (void)info:(NSString *)format, ... {
    _infoCallCount++;
    va_list args;
    va_start(args, format);
    NSString *message = [[NSString alloc] initWithFormat:format arguments:args];
    va_end(args);
    
    [_loggedMessages addObject:message];
    [_realLogger info:message]; // Delegate to real logger
}

- (void)error:(NSString *)format, ... {
    _errorCallCount++;
    // ... similar implementation
}

@end
```

---

## 74.7 OCMock Introduction

OCMock เป็น third-party library ที่ทรงพลังสำหรับสร้าง mock objects ใน Objective-C

### การติดตั้ง OCMock

```ruby
# Podfile
pod 'OCMock', '~> 3.9'
```

### OCMock Basics

```objc
#import <OCMock/OCMock.h>

@interface OrderServiceOCMockTests : XCTestCase
@end

@implementation OrderServiceOCMockTests

- (void)testPlaceOrder_WithValidPayment_ProcessesPayment {
    // สร้าง mock ด้วย OCMock
    id<PaymentGatewayProtocol> mockGateway = OCMProtocolMock(@protocol(PaymentGatewayProtocol));
    id<InventoryProtocol> mockInventory = OCMProtocolMock(@protocol(InventoryProtocol));
    
    // กำหนด behavior ของ mock (stub)
    Order *order = [[Order alloc] initWithTotal:99.99];
    PaymentResult *successResult = [[PaymentResult alloc] initWithSuccess:YES];
    
    OCMStub([mockGateway processPayment:order]).andReturn(successResult);
    OCMStub([mockInventory checkAvailability:order]).andReturn(YES);
    
    // สร้าง service พร้อม mocks
    OrderService *service = [[OrderService alloc] initWithGateway:mockGateway
                                                        inventory:mockInventory];
    
    // Act
    BOOL result = [service placeOrder:order];
    
    // Assert - ตรวจสอบ return value
    XCTAssertTrue(result);
    
    // Assert - ตรวจสอบว่า methods ถูกเรียก
    OCMVerify([mockGateway processPayment:order]);
    OCMVerify([mockInventory checkAvailability:order]);
}

- (void)testPlaceOrder_WhenOutOfStock_DoesNotProcessPayment {
    id<PaymentGatewayProtocol> mockGateway = OCMProtocolMock(@protocol(PaymentGatewayProtocol));
    id<InventoryProtocol> mockInventory = OCMProtocolMock(@protocol(InventoryProtocol));
    
    Order *order = [[Order alloc] initWithTotal:99.99];
    
    OCMStub([mockInventory checkAvailability:order]).andReturn(NO);
    
    // Expect processPayment ไม่ถูกเรียก
    OCMReject([mockGateway processPayment:[OCMArg any]]);
    
    OrderService *service = [[OrderService alloc] initWithGateway:mockGateway
                                                        inventory:mockInventory];
    
    BOOL result = [service placeOrder:order];
    
    XCTAssertFalse(result);
    OCMVerifyAll(mockGateway); // ตรวจสอบว่า OCMReject ไม่ถูก violate
}

@end
```

### OCMock Advanced Features

```objc
// Argument Matchers
OCMStub([mockService fetchData:[OCMArg any]]).andReturn(someData);
OCMStub([mockService search:[OCMArg checkWithBlock:^BOOL(id value) {
    return [value length] > 0; // Custom matcher
}]]).andReturn(results);

// Partial Mocks - mock specific methods of real objects
Calculator *realCalculator = [[Calculator alloc] init];
id partialMock = OCMPartialMock(realCalculator);

OCMStub([partialMock sqrt:4.0]).andReturn(2.0);

// ยังใช้งาน real implementation ของ methods อื่น ๆ
XCTAssertEqual([partialMock add:2 to:3], 5); // Real implementation

// Class Mocks
id classMock = OCMClassMock([NSURLSession class]);
OCMStub([classMock sharedSession]).andReturn(mockSession);

// Verify call count
OCMVerify(times(2), [mockService processItem:[OCMArg any]]);
OCMVerify(atLeast(1), [mockService sendNotification:[OCMArg any]]);
OCMVerify(never(), [mockService deleteData]);
```

---

## 74.8 Asynchronous Testing

การทดสอบ async operations ต้องใช้ XCTestExpectation

### XCTestExpectation

```objc
@implementation NetworkServiceTests

- (void)testFetchUserData_ReturnsUserData {
    // สร้าง expectation
    XCTestExpectation *expectation = [self expectationWithDescription:@"Fetch user data"];
    
    NetworkService *service = [[NetworkService alloc] init];
    
    [service fetchUserWithId:@"123" completion:^(User *user, NSError *error) {
        // Assert ใน completion block
        XCTAssertNotNil(user);
        XCTAssertNil(error);
        XCTAssertEqualObjects(user.userId, @"123");
        
        // Fulfill เมื่อ async operation เสร็จ
        [expectation fulfill];
    }];
    
    // รอ expectation ให้เสร็จ (timeout = 10 วินาที)
    [self waitForExpectationsWithTimeout:10.0 handler:^(NSError *error) {
        if (error) {
            NSLog(@"Timeout Error: %@", error);
        }
    }];
}

- (void)testDownloadImage_CachesResult {
    XCTestExpectation *firstLoad = [self expectationWithDescription:@"First image load"];
    XCTestExpectation *secondLoad = [self expectationWithDescription:@"Second image load (cached)"];
    
    ImageLoader *loader = [[ImageLoader alloc] init];
    NSURL *imageURL = [NSURL URLWithString:@"https://example.com/image.jpg"];
    
    __block NSTimeInterval firstLoadTime = 0;
    NSDate *start = [NSDate date];
    
    [loader loadImage:imageURL completion:^(UIImage *image) {
        firstLoadTime = [[NSDate date] timeIntervalSinceDate:start];
        XCTAssertNotNil(image);
        [firstLoad fulfill];
        
        // Load again - should be from cache (faster)
        NSDate *cacheStart = [NSDate date];
        [loader loadImage:imageURL completion:^(UIImage *cachedImage) {
            NSTimeInterval cacheLoadTime = [[NSDate date] timeIntervalSinceDate:cacheStart];
            XCTAssertNotNil(cachedImage);
            XCTAssertLessThan(cacheLoadTime, firstLoadTime); // Cache should be faster
            [secondLoad fulfill];
        }];
    }];
    
    [self waitForExpectationsWithTimeout:30.0 handler:nil];
}

@end
```

### Multiple Expectations

```objc
- (void)testBatchOperation_ProcessesAllItems {
    NSArray *items = @[@"item1", @"item2", @"item3"];
    
    // สร้าง expectation สำหรับแต่ละ item
    NSMutableArray *expectations = [NSMutableArray array];
    for (NSString *item in items) {
        XCTestExpectation *exp = [self expectationWithDescription:
                                   [NSString stringWithFormat:@"Process %@", item]];
        [expectations addObject:exp];
    }
    
    BatchProcessor *processor = [[BatchProcessor alloc] init];
    
    __block NSInteger processedCount = 0;
    [processor processItems:items 
             itemCompletion:^(NSString *item, NSInteger index) {
        XCTAssertEqualObjects(item, items[index]);
        [expectations[index] fulfill];
        processedCount++;
    } 
            batchCompletion:^{
        XCTAssertEqual(processedCount, items.count);
    }];
    
    [self waitForExpectationsWithTimeout:10.0 handler:nil];
}
```

---

## 74.9 Integration Tests

Integration Tests ทดสอบการทำงานร่วมกันของหลาย components

```objc
@interface UserRegistrationIntegrationTests : XCTestCase

@property (nonatomic, strong) UserService *userService;
@property (nonatomic, strong) EmailService *emailService;
@property (nonatomic, strong) TestDatabase *database;

@end

@implementation UserRegistrationIntegrationTests

- (void)setUp {
    [super setUp];
    
    // ใช้ real components แต่กับ test database
    self.database = [[TestDatabase alloc] initWithInMemoryStore];
    
    // Email service ที่ไม่ส่ง email จริง
    self.emailService = [[TestEmailService alloc] init];
    
    id<UserRepositoryProtocol> repository = [[CoreDataUserRepository alloc] 
                                              initWithContext:self.database.context];
    
    self.userService = [[UserService alloc] initWithRepository:repository
                                                  emailService:self.emailService];
}

- (void)testFullRegistrationFlow {
    // Arrange
    UserRegistrationForm *form = [[UserRegistrationForm alloc] init];
    form.name = @"Integration Test User";
    form.email = @"integration@test.com";
    form.password = @"SecurePass123!";
    
    XCTestExpectation *expectation = [self expectationWithDescription:@"Registration complete"];
    
    // Act
    [self.userService registerUser:form completion:^(User *user, NSError *error) {
        // Assert registration
        XCTAssertNotNil(user);
        XCTAssertNil(error);
        XCTAssertEqualObjects(user.email, @"integration@test.com");
        
        // Assert user was saved to database
        [self.database fetchUserByEmail:@"integration@test.com" 
                             completion:^(User *savedUser, NSError *dbError) {
            XCTAssertNotNil(savedUser);
            XCTAssertEqualObjects(savedUser.userId, user.userId);
            
            // Assert welcome email was sent
            XCTAssertEqual(self.emailService.sentEmailCount, 1);
            XCTAssertEqualObjects(self.emailService.lastRecipient, @"integration@test.com");
            
            [expectation fulfill];
        }];
    }];
    
    [self waitForExpectationsWithTimeout:10.0 handler:nil];
}

@end
```

---

## 74.10 UI Testing (XCUITest)

XCUITest ช่วยทดสอบ UI ของแอปโดยการ simulate การกดปุ่ม, พิมพ์ข้อความ ฯลฯ

### โครงสร้างพื้นฐาน

```objc
#import <XCTest/XCTest.h>

@interface LoginUITests : XCTestCase

@property (nonatomic, strong) XCUIApplication *app;

@end

@implementation LoginUITests

- (void)setUp {
    [super setUp];
    
    // หยุด test ทันทีเมื่อ fail
    self.continueAfterFailure = NO;
    
    // Launch app
    self.app = [[XCUIApplication alloc] init];
    [self.app launch];
}

- (void)tearDown {
    [self.app terminate];
    [super tearDown];
}

- (void)testLoginWithValidCredentials {
    // ค้นหา UI elements
    XCUIElement *emailField = self.app.textFields[@"emailTextField"];
    XCUIElement *passwordField = self.app.secureTextFields[@"passwordTextField"];
    XCUIElement *loginButton = self.app.buttons[@"loginButton"];
    
    // Assert elements exist
    XCTAssertTrue(emailField.exists);
    XCTAssertTrue(passwordField.exists);
    XCTAssertTrue(loginButton.exists);
    
    // Interact with UI
    [emailField tap];
    [emailField typeText:@"user@example.com"];
    
    [passwordField tap];
    [passwordField typeText:@"password123"];
    
    [loginButton tap];
    
    // Assert navigation to main screen
    XCUIElement *mainScreen = self.app.otherElements[@"mainScreen"];
    XCTAssertTrue([mainScreen waitForExistenceWithTimeout:5.0]);
}

- (void)testLoginWithInvalidCredentials {
    XCUIElement *emailField = self.app.textFields[@"emailTextField"];
    XCUIElement *passwordField = self.app.secureTextFields[@"passwordTextField"];
    XCUIElement *loginButton = self.app.buttons[@"loginButton"];
    
    [emailField tap];
    [emailField typeText:@"wrong@example.com"];
    
    [passwordField tap];
    [passwordField typeText:@"wrongpassword"];
    
    [loginButton tap];
    
    // Assert error message appears
    XCUIElement *errorMessage = self.app.staticTexts[@"errorMessage"];
    XCTAssertTrue([errorMessage waitForExistenceWithTimeout:5.0]);
    XCTAssertTrue([errorMessage.label containsString:@"Invalid"]);
}

- (void)testEmptyFieldsShowValidationErrors {
    XCUIElement *loginButton = self.app.buttons[@"loginButton"];
    [loginButton tap];
    
    // Assert validation errors shown
    XCTAssertTrue(self.app.staticTexts[@"emailErrorLabel"].exists);
    XCTAssertTrue(self.app.staticTexts[@"passwordErrorLabel"].exists);
}

@end
```

### XCUITest กับ Table Views

```objc
@implementation ContactListUITests

- (void)testContactListLoadsData {
    XCUIElement *tableView = self.app.tables[@"contactsTable"];
    XCTAssertTrue(tableView.exists);
    
    // รอให้ data โหลด
    XCUIElement *firstCell = tableView.cells.firstMatch;
    XCTAssertTrue([firstCell waitForExistenceWithTimeout:10.0]);
    
    // Assert มี contacts
    XCTAssertGreaterThan(tableView.cells.count, 0);
}

- (void)testAddNewContact {
    // Tap add button
    [self.app.navigationBars.buttons[@"addButton"] tap];
    
    // Fill form
    [self.app.textFields[@"nameField"] tap];
    [self.app.textFields[@"nameField"] typeText:@"New Contact"];
    
    [self.app.textFields[@"phoneField"] tap];
    [self.app.textFields[@"phoneField"] typeText:@"0891234567"];
    
    // Save
    [self.app.navigationBars.buttons[@"saveButton"] tap];
    
    // Assert new contact appears in list
    XCUIElement *newCell = self.app.tables[@"contactsTable"].cells.staticTexts[@"New Contact"];
    XCTAssertTrue([newCell waitForExistenceWithTimeout:5.0]);
}

- (void)testDeleteContact {
    XCUIElement *tableView = self.app.tables[@"contactsTable"];
    XCUIElement *firstCell = tableView.cells.firstMatch;
    
    // Get initial count
    NSInteger initialCount = tableView.cells.count;
    
    // Swipe to delete
    [firstCell swipeLeft];
    [self.app.buttons[@"Delete"] tap];
    
    // Assert cell count decreased
    XCTAssertEqual(tableView.cells.count, initialCount - 1);
}

@end
```

---

## 74.11 Code Coverage

Code Coverage ช่วยวัดว่า tests ครอบคลุม code ได้มากแค่ไหน

### เปิดใช้ Code Coverage

1. ไปที่ Product > Scheme > Edit Scheme
2. เลือก Test
3. ติ๊ก "Gather coverage for all targets"

### ตีความผลลัพธ์

```objc
// Code Coverage ควรอยู่ที่เท่าไร?
// - Critical business logic: 90%+
// - ViewControllers: 60-70%
// - Models: 80%+
// - Utilities: 70%+

// โค้ดที่ควรมี Coverage
@implementation TaxCalculator

// ✅ ควรมี test ครอบคลุมทุก branch
- (double)calculateTax:(double)income {
    if (income <= 150000) {        // Branch 1
        return 0;
    } else if (income <= 300000) { // Branch 2
        return (income - 150000) * 0.05;
    } else if (income <= 500000) { // Branch 3
        return 7500 + (income - 300000) * 0.10;
    } else if (income <= 750000) { // Branch 4
        return 27500 + (income - 500000) * 0.15;
    } else if (income <= 1000000) { // Branch 5
        return 65000 + (income - 750000) * 0.20;
    } else {                        // Branch 6
        return 115000 + (income - 1000000) * 0.25;
    }
}

@end

// Tests ที่ครอบคลุมทุก branch
- (void)testCalculateTax_FirstBracket {
    XCTAssertEqualWithAccuracy([calc calculateTax:100000], 0, 0.01);
}
- (void)testCalculateTax_SecondBracket {
    XCTAssertEqualWithAccuracy([calc calculateTax:200000], 2500, 0.01);
}
// ... test แต่ละ bracket
```

---

## 74.12 Test-Driven Development (TDD)

TDD คือการเขียน test ก่อน implementation โดยทำตาม Red-Green-Refactor cycle

### TDD Cycle

```
Red → Green → Refactor
เขียน test ที่ fail → ทำให้ pass → ปรับปรุง code
```

### ตัวอย่าง TDD

```objc
// Step 1: Red - เขียน test ที่ fail
- (void)testPasswordValidator_StrongPassword_ReturnsTrue {
    // PasswordValidator ยังไม่ได้สร้าง - test จะ compile error ก่อน
    PasswordValidator *validator = [[PasswordValidator alloc] init];
    BOOL result = [validator isStrong:@"MyStr0ng!Pass"];
    XCTAssertTrue(result);
}

// Step 2: Green - เขียน implementation ให้ test ผ่าน (minimal)
@implementation PasswordValidator

- (BOOL)isStrong:(NSString *)password {
    return YES; // Simplest implementation ที่ทำให้ test ผ่าน
}

@end

// Step 3: เขียน test เพิ่มที่ทำให้ fail อีกครั้ง
- (void)testPasswordValidator_WeakPassword_ReturnsFalse {
    PasswordValidator *validator = [[PasswordValidator alloc] init];
    XCTAssertFalse([validator isStrong:@"password"]); // ตอนนี้ fail เพราะ return YES เสมอ
}

// Step 4: ปรับ implementation
@implementation PasswordValidator

- (BOOL)isStrong:(NSString *)password {
    if (password.length < 8) return NO;
    
    BOOL hasUppercase = [[NSPredicate predicateWithFormat:@"SELF MATCHES %@", @".*[A-Z].*"] 
                          evaluateWithObject:password];
    BOOL hasLowercase = [[NSPredicate predicateWithFormat:@"SELF MATCHES %@", @".*[a-z].*"] 
                          evaluateWithObject:password];
    BOOL hasNumber = [[NSPredicate predicateWithFormat:@"SELF MATCHES %@", @".*[0-9].*"] 
                       evaluateWithObject:password];
    BOOL hasSpecial = [[NSPredicate predicateWithFormat:@"SELF MATCHES %@", @".*[^A-Za-z0-9].*"] 
                        evaluateWithObject:password];
    
    return hasUppercase && hasLowercase && hasNumber && hasSpecial;
}

@end

// Step 5: เพิ่ม tests ให้ครอบคลุม
- (void)testPasswordValidator_AllRules {
    PasswordValidator *validator = [[PasswordValidator alloc] init];
    
    // Valid passwords
    XCTAssertTrue([validator isStrong:@"MyStr0ng!Pass"]);
    XCTAssertTrue([validator isStrong:@"C0mpl3x#P@ss"]);
    
    // Too short
    XCTAssertFalse([validator isStrong:@"Sh0rt!"]);
    
    // No uppercase
    XCTAssertFalse([validator isStrong:@"mystr0ng!pass"]);
    
    // No lowercase
    XCTAssertFalse([validator isStrong:@"MYSTR0NG!PASS"]);
    
    // No number
    XCTAssertFalse([validator isStrong:@"MyStrong!Pass"]);
    
    // No special character
    XCTAssertFalse([validator isStrong:@"MyStr0ngPass"]);
}
```

---

## 74.13 Behavior-Driven Development (BDD)

BDD เน้นการเขียน tests ในแบบที่อ่านเข้าใจง่าย เหมือนภาษาธรรมชาติ

### ตัวอย่าง BDD Style ด้วย Kiwi

```objc
// ใช้ Kiwi framework สำหรับ BDD
#import "Kiwi.h"
#import "ShoppingCart.h"

SPEC_BEGIN(ShoppingCartSpec)

describe(@"ShoppingCart", ^{
    context(@"when empty", ^{
        __block ShoppingCart *cart;
        
        beforeEach(^{
            cart = [[ShoppingCart alloc] init];
        });
        
        it(@"should have zero items", ^{
            [[theValue(cart.itemCount) should] equal:theValue(0)];
        });
        
        it(@"should have zero total", ^{
            [[theValue(cart.total) should] equal:theValue(0.0) withDelta:0.001];
        });
    });
    
    context(@"when adding items", ^{
        __block ShoppingCart *cart;
        __block Product *product;
        
        beforeEach(^{
            cart = [[ShoppingCart alloc] init];
            product = [[Product alloc] initWithName:@"Book" price:99.0];
        });
        
        it(@"should increase item count", ^{
            [cart addProduct:product];
            [[theValue(cart.itemCount) should] equal:theValue(1)];
        });
        
        it(@"should update total price", ^{
            [cart addProduct:product];
            [[theValue(cart.total) should] equal:theValue(99.0) withDelta:0.001];
        });
        
        context(@"with quantity", ^{
            it(@"should multiply price by quantity", ^{
                [cart addProduct:product quantity:3];
                [[theValue(cart.total) should] equal:theValue(297.0) withDelta:0.001];
            });
        });
    });
});

SPEC_END
```

---

## 74.14 Performance Testing

XCTest มี `measure` block สำหรับ performance testing

```objc
@implementation SortingPerformanceTests

- (void)testSortingPerformance_SmallArray {
    NSArray *array = [self generateRandomArray:100];
    
    [self measureBlock:^{
        // code ที่ต้องการวัด performance
        [array sortedArrayUsingComparator:^NSComparisonResult(id a, id b) {
            return [a compare:b];
        }];
    }];
}

- (void)testSortingPerformance_LargeArray {
    NSArray *array = [self generateRandomArray:10000];
    
    [self measureBlock:^{
        [array sortedArrayUsingComparator:^NSComparisonResult(id a, id b) {
            return [a compare:b];
        }];
    }];
}

- (NSArray *)generateRandomArray:(NSInteger)count {
    NSMutableArray *array = [NSMutableArray arrayWithCapacity:count];
    for (NSInteger i = 0; i < count; i++) {
        [array addObject:@(arc4random_uniform(1000000))];
    }
    return [array copy];
}

// เปรียบเทียบ 2 algorithms
- (void)testBubbleSortPerformance {
    NSArray *array = [self generateRandomArray:1000];
    
    [self measureBlock:^{
        [SortingAlgorithms bubbleSort:[array mutableCopy]];
    }];
}

- (void)testQuickSortPerformance {
    NSArray *array = [self generateRandomArray:1000];
    
    [self measureBlock:^{
        [SortingAlgorithms quickSort:[array mutableCopy]];
    }];
}

// กำหนด baseline
- (void)testImageProcessingPerformance {
    UIImage *testImage = [UIImage imageNamed:@"test_image"];
    
    [self measureMetrics:@[XCTPerformanceMetric_WallClockTime]
     automaticallyStartMeasuring:NO
                      forBlock:^{
        // Setup
        [self startMeasuring];
        
        // Code ที่ต้องการวัด
        UIImage *processedImage = [ImageProcessor applyFilter:testImage 
                                                     filterName:@"blur"];
        
        [self stopMeasuring];
        
        // Verify result outside measurement
        XCTAssertNotNil(processedImage);
    }];
}

@end
```

---

## 74.15 แบบฝึกหัด

### แบบฝึกหัดที่ 1: TDD สร้าง Calculator

ใช้ TDD เขียน tests ก่อน แล้วค่อย implement:
- บวก, ลบ, คูณ, หาร
- ตรวจสอบ division by zero
- รองรับ decimal numbers
- Memory functions (M+, M-, MR, MC)

```objc
// เริ่มด้วย test เหล่านี้
- (void)testAdd_PositiveNumbers { }
- (void)testAdd_NegativeNumbers { }
- (void)testDivide_ByZero_ThrowsException { }
- (void)testMemoryStore_StoresAndRetrieves { }
```

### แบบฝึกหัดที่ 2: Mock-Based Tests

สร้าง `PaymentService` พร้อม tests ที่ใช้ mocks:

```objc
@protocol CreditCardAPIProtocol <NSObject>
- (void)chargeCard:(CreditCard *)card 
            amount:(double)amount 
        completion:(void(^)(ChargeResult *result, NSError *error))completion;
@end

// TODO: สร้าง MockCreditCardAPI
// TODO: เขียน tests ครอบคลุม:
//   - successful payment
//   - declined card
//   - network error
//   - invalid card number
```

### แบบฝึกหัดที่ 3: UI Tests สำหรับ Form Validation

```objc
// TODO: เขียน UI tests สำหรับ registration form:
// - กรอก email ที่ไม่ valid -> error message แสดง
// - password ไม่ตรงกัน -> error message แสดง
// - กรอกครบ valid -> สำเร็จ, ไปหน้า home
```

### แบบฝึกหัดที่ 4: Performance Test

```objc
// TODO: สร้าง performance tests สำหรับ
// - CoreData fetch operations
// - JSON parsing
// - Image resizing
// แล้วหา bottleneck และ optimize
```

---

## สรุป

บทนี้ครอบคลุมการทดสอบใน Objective-C อย่างครบถ้วน:

- **XCTestCase**: โครงสร้างพื้นฐานของ testing
- **Assertions**: ตรวจสอบผลลัพธ์ด้วย XCTAssert functions
- **setUp/tearDown**: จัดการ test fixtures
- **Test Doubles**: Mocks, Stubs, Fakes สำหรับการ isolate dependencies
- **OCMock**: Library ทรงพลังสำหรับสร้าง mocks
- **Async Testing**: ใช้ XCTestExpectation
- **UI Testing**: ทดสอบ UI ด้วย XCUITest
- **Code Coverage**: วัดความครอบคลุมของ tests
- **TDD/BDD**: methodology การพัฒนาโดยใช้ tests นำ
- **Performance Testing**: วัด performance ด้วย measureBlock

ในบทต่อไป (Part 75) เราจะเรียนรู้เรื่อง Performance Optimization ซึ่งสัมพันธ์กับ Performance Testing ที่เราเรียนในบทนี้

---

*จบ Part 74 - Testing กับ XCTest*
