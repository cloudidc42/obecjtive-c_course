# Part 13: Encapsulation and Properties ใน Objective-C

## บทนำ

Encapsulation (การห่อหุ้มข้อมูล) เป็นหลักการ OOP ที่ซ่อนรายละเอียดภายในของ object และเปิดเผยเฉพาะ interface ที่จำเป็น Objective-C มีกลไก encapsulation ที่หลากหลายตั้งแต่ระดับ ivar จนถึง Properties สมัยใหม่

---

## 1. Access Control ใน Objective-C

### ระดับการเข้าถึง

Objective-C ไม่มี access modifier แบบ Java/C++ สำหรับ methods แต่มีสำหรับ instance variables

| ระดับ | Keyword | ขอบเขต |
|-------|---------|--------|
| Private | `@private` | เฉพาะ class นั้นเท่านั้น |
| Protected | `@protected` | class + subclasses (ค่าเริ่มต้น) |
| Public | `@public` | ทุกที่ |
| Package | `@package` | ภายใน framework เดียวกัน |

```objc
// AccessControlDemo.h
@interface AccessControlDemo : NSObject {
@private
    NSString *_password;          // เข้าถึงได้เฉพาะใน AccessControlDemo
    NSInteger _loginAttempts;
    NSData   *_encryptedData;

@protected
    NSString *_username;          // เข้าถึงได้ใน class + subclasses
    NSInteger _sessionTimeout;

@public
    NSString *displayName;        // ทุกคนเข้าถึงได้ (ไม่แนะนำ!)

@package
    NSInteger _packageOnlyData;   // ภายใน framework เดียวกัน
}

// Methods ที่ประกาศใน .h จะเป็น "public" โดยปริยาย
- (BOOL)loginWithUsername:(NSString *)username password:(NSString *)password;
- (void)logout;

@end
```

```objc
// ตัวอย่าง: Subclass เข้าถึง @protected ivar
// SecuritySubclass.h
@interface SecuritySubclass : AccessControlDemo

- (void)extendSession;
- (NSString *)getUsername;

@end

@implementation SecuritySubclass

- (void)extendSession {
    _sessionTimeout += 30;  // OK! @protected
    NSLog(@"ขยาย session ออกไป 30 นาที (รวม: %ld)", (long)_sessionTimeout);
}

- (NSString *)getUsername {
    return _username;  // OK! @protected
}

- (void)tryAccessPrivate {
    // _password = @"new";  // ERROR! ไม่สามารถเข้าถึง @private ได้
    // _loginAttempts = 0;  // ERROR!
}

@end
```

---

## 2. @private, @protected, @public สำหรับ ivars

### Best Practices

```objc
// BestPracticeClass.h - ประกาศ ivars ใน @interface หรือ @implementation
// วิธีที่ 1: ประกาศ ivar ใน header (เข้าถึงได้จาก subclass)
@interface BaseEntity : NSObject {
@protected
    // ivars ที่ต้องการให้ subclass เข้าถึงได้
    NSString *_entityID;
    NSDate   *_createdAt;
    NSDate   *_updatedAt;

@private
    // ivars ที่ต้องการซ่อน
    NSMutableDictionary *_metadata;
}

@property (nonatomic, copy, readonly) NSString *entityID;
@property (nonatomic, strong, readonly) NSDate *createdAt;

- (void)updateMetadata:(NSDictionary *)metadata;
- (id)metadataValueForKey:(NSString *)key;

@end
```

```objc
// BestPracticeClass.m
// วิธีที่ 2 (แนะนำ): ประกาศ ivar ใน @implementation เพื่อซ่อนจาก header
@implementation BaseEntity {
    // Private ivars ที่ไม่ต้องแสดงใน header
    NSMutableDictionary *_privateCache;
    NSInteger            _accessCount;
    BOOL                 _isDirty;
}

- (instancetype)init {
    self = [super init];
    if (self) {
        _entityID     = [[NSUUID UUID] UUIDString];
        _createdAt    = [NSDate date];
        _updatedAt    = [NSDate date];
        _metadata     = [NSMutableDictionary dictionary];
        _privateCache = [NSMutableDictionary dictionary];
        _accessCount  = 0;
        _isDirty      = NO;
    }
    return self;
}

- (NSString *)entityID  { return _entityID; }
- (NSDate *)createdAt   { return _createdAt; }

- (void)updateMetadata:(NSDictionary *)metadata {
    [_metadata addEntriesFromDictionary:metadata];
    _updatedAt = [NSDate date];
    _isDirty = YES;
    [_privateCache removeAllObjects];  // invalidate cache
}

- (id)metadataValueForKey:(NSString *)key {
    _accessCount++;
    // ตรวจ cache ก่อน
    id cached = _privateCache[key];
    if (cached) return cached;

    id value = _metadata[key];
    if (value) _privateCache[key] = value;
    return value;
}

@end
```

---

## 3. Properties เป็น Modern Encapsulation Mechanism

### Properties แทนที่ ivars ได้อย่างไร

```objc
// แบบเก่า (ไม่แนะนำ)
@interface OldStyle : NSObject {
    NSString *name;        // @protected โดยค่าเริ่มต้น
    NSInteger age;
}
// getter/setter ต้องเขียนเอง
- (NSString *)name;
- (void)setName:(NSString *)name;
- (NSInteger)age;
- (void)setAge:(NSInteger)age;
@end

// แบบใหม่ (แนะนำ)
@interface NewStyle : NSObject

@property (nonatomic, copy)   NSString *name;   // สร้าง getter/setter อัตโนมัติ
@property (nonatomic, assign) NSInteger age;    // + backing ivar _name, _age

@end
// ใช้ dot syntax: obj.name = @"test"; obj.age = 25;
```

### ตัวอย่าง Encapsulation ที่ดี

```objc
// BankAccount.h - Public interface
@interface BankAccount : NSObject

// Public properties (อ่านได้จากภายนอก)
@property (nonatomic, copy,   readonly) NSString *accountNumber;
@property (nonatomic, copy)             NSString *ownerName;
@property (nonatomic, assign, readonly) double    balance;
@property (nonatomic, strong, readonly) NSDate   *createdDate;

// Public methods
- (instancetype)initWithOwner:(NSString *)ownerName initialBalance:(double)balance;
- (BOOL)depositAmount:(double)amount;
- (BOOL)withdrawAmount:(double)amount;
- (NSString *)miniStatement;

@end
```

```objc
// BankAccount.m - Private implementation details
// Class Extension เพื่อ declare private properties/methods
@interface BankAccount ()

// Private properties (เขียนได้จากภายใน class)
@property (nonatomic, copy,   readwrite) NSString *accountNumber;
@property (nonatomic, assign, readwrite) double    balance;
@property (nonatomic, strong) NSMutableArray      *transactionLog;
@property (nonatomic, strong) NSDate              *lastTransactionDate;

// Private methods
- (NSString *)generateAccountNumber;
- (void)logTransaction:(NSString *)description amount:(double)amount;
- (BOOL)validateAmount:(double)amount;

@end

@implementation BankAccount

- (instancetype)initWithOwner:(NSString *)ownerName initialBalance:(double)balance {
    self = [super init];
    if (self) {
        _ownerName          = [ownerName copy];
        _balance            = MAX(0, balance);
        _accountNumber      = [self generateAccountNumber];
        _createdDate        = [NSDate date];
        _transactionLog     = [NSMutableArray array];
        _lastTransactionDate = [NSDate date];

        if (balance > 0) {
            [self logTransaction:@"เปิดบัญชี" amount:balance];
        }
    }
    return self;
}

// Private method - ซ่อนจาก public interface
- (NSString *)generateAccountNumber {
    return [NSString stringWithFormat:@"TH%08lu", (unsigned long)arc4random_uniform(99999999)];
}

- (BOOL)validateAmount:(double)amount {
    if (amount <= 0) {
        NSLog(@"ข้อผิดพลาด: จำนวนเงินต้องมากกว่า 0");
        return NO;
    }
    if (amount > 1000000) {
        NSLog(@"ข้อผิดพลาด: เกินวงเงินต่อครั้ง (1,000,000 บาท)");
        return NO;
    }
    return YES;
}

- (void)logTransaction:(NSString *)description amount:(double)amount {
    NSString *entry = [NSString stringWithFormat:@"%@ | %@ | ฿%.2f | คงเหลือ: ฿%.2f",
                       [NSDate date], description, amount, _balance];
    [_transactionLog addObject:entry];
    _lastTransactionDate = [NSDate date];
}

- (BOOL)depositAmount:(double)amount {
    if (![self validateAmount:amount]) return NO;
    _balance += amount;
    [self logTransaction:@"ฝากเงิน" amount:amount];
    return YES;
}

- (BOOL)withdrawAmount:(double)amount {
    if (![self validateAmount:amount]) return NO;
    if (amount > _balance) {
        NSLog(@"ยอดเงินไม่เพียงพอ (มี: ฿%.2f)", _balance);
        return NO;
    }
    _balance -= amount;
    [self logTransaction:@"ถอนเงิน" amount:amount];
    return YES;
}

- (NSString *)miniStatement {
    NSMutableString *stmt = [NSMutableString string];
    [stmt appendFormat:@"บัญชี: %@\n", _accountNumber];
    [stmt appendFormat:@"ยอดคงเหลือ: ฿%.2f\n", _balance];
    [stmt appendString:@"รายการล่าสุด:\n"];
    NSArray *recent = [_transactionLog count] > 5 ?
                      [_transactionLog subarrayWithRange:
                       NSMakeRange(_transactionLog.count - 5, 5)] :
                      _transactionLog;
    for (NSString *entry in recent) {
        [stmt appendFormat:@"  %@\n", entry];
    }
    return stmt;
}

@end
```

---

## 4. Property Attributes ครบถ้วน

### 4.1 Atomicity: atomic vs nonatomic

```objc
// AtomicityDemo.h
@interface AtomicityDemo : NSObject

// atomic (ค่าเริ่มต้น) - thread-safe สำหรับ get/set
// - ใช้ lock ป้องกัน race condition
// - ช้ากว่า nonatomic ประมาณ 2-10x
// - ไม่ได้ทำให้ทั้ง object เป็น thread-safe!
@property (atomic, strong)    NSString  *atomicString;
@property (atomic, assign)    NSInteger  atomicCount;

// nonatomic - ไม่ thread-safe แต่เร็วกว่า
// - ใช้ส่วนใหญ่ใน iOS/macOS เพราะ UI ทำงานบน main thread
// - ถ้าต้องการ thread-safety ต้องจัดการเองด้วย GCD หรือ locks
@property (nonatomic, strong) NSString  *nonatomicString;
@property (nonatomic, assign) NSInteger  nonatomicCount;

@end
```

```objc
// ทดสอบ atomic property กับ multi-threading
@implementation AtomicityDemo

- (void)testAtomicSafety {
    self.atomicCount = 0;

    dispatch_queue_t queue = dispatch_get_global_queue(DISPATCH_QUEUE_PRIORITY_DEFAULT, 0);
    dispatch_group_t group = dispatch_group_create();

    // หลาย threads เขียนพร้อมกัน
    for (int i = 0; i < 100; i++) {
        dispatch_group_async(group, queue, ^{
            // atomic ป้องกัน get/set จาก multiple threads
            self.atomicCount = self.atomicCount + 1;  // ยังอาจมี race condition!
        });
    }

    dispatch_group_wait(group, DISPATCH_TIME_FOREVER);
    NSLog(@"atomic count: %ld (อาจไม่ใช่ 100 เพราะ read-modify-write ไม่ atomic)", (long)self.atomicCount);
}

// Thread-safe increment ที่ถูกต้อง ใช้ GCD
- (void)threadSafeIncrement {
    static dispatch_queue_t serialQueue;
    static dispatch_once_t onceToken;
    dispatch_once(&onceToken, ^{
        serialQueue = dispatch_queue_create("com.example.counter", DISPATCH_QUEUE_SERIAL);
    });

    dispatch_sync(serialQueue, ^{
        self.atomicCount++;
    });
}

@end
```

### 4.2 Access: readonly vs readwrite

```objc
// AccessAttributesDemo.h
@interface Person : NSObject

// Public interface: readonly
@property (nonatomic, copy,   readonly) NSString   *personID;      // สร้างครั้งเดียว
@property (nonatomic, copy,   readonly) NSDate     *birthDate;     // ไม่เปลี่ยนแปลง
@property (nonatomic, assign, readonly) NSInteger   age;            // คำนวณจาก birthDate
@property (nonatomic, assign, readonly) BOOL        isAdult;        // computed

// readwrite (ค่าเริ่มต้น) - มีทั้ง getter และ setter
@property (nonatomic, copy)             NSString   *name;
@property (nonatomic, copy)             NSString   *email;

@end
```

```objc
// Person.m - ใช้ class extension เพื่อ redeclare readonly เป็น readwrite
@interface Person ()
@property (nonatomic, copy,   readwrite) NSString *personID;
@property (nonatomic, copy,   readwrite) NSDate   *birthDate;
@end

@implementation Person

- (instancetype)initWithName:(NSString *)name birthDate:(NSDate *)birthDate {
    self = [super init];
    if (self) {
        _personID  = [[NSUUID UUID] UUIDString];    // set ครั้งเดียว
        _birthDate = [birthDate copy];               // set ครั้งเดียว
        _name      = [name copy];
        _email     = @"";
    }
    return self;
}

// Computed readonly property
- (NSInteger)age {
    NSCalendar *calendar = [NSCalendar currentCalendar];
    NSDateComponents *components = [calendar components:NSCalendarUnitYear
                                               fromDate:_birthDate
                                                 toDate:[NSDate date]
                                                options:0];
    return components.year;
}

// Computed readonly property
- (BOOL)isAdult {
    return self.age >= 18;
}

@end
```

```objc
// การใช้งาน
Person *p = [[Person alloc] initWithName:@"สมชาย"
                               birthDate:[NSDate dateWithTimeIntervalSince1970:631152000]];

NSLog(@"ID: %@", p.personID);
NSLog(@"อายุ: %ld", (long)p.age);
NSLog(@"ผู้ใหญ่: %@", p.isAdult ? @"ใช่" : @"ไม่");

// p.personID = @"NEW";  // ERROR! personID เป็น readonly จากภายนอก
p.name = @"สมชายใหม่";   // OK! name เป็น readwrite
```

---

## 5. Memory Management Attributes

### 5.1 strong

```objc
@interface StrongDemo : NSObject

// strong - เพิ่ม retain count, object จะอยู่ตราบที่มี strong reference
@property (nonatomic, strong) NSString      *title;        // สำหรับ objects
@property (nonatomic, strong) NSArray       *items;
@property (nonatomic, strong) NSMutableArray *mutableItems;
@property (nonatomic, strong) NSDictionary  *config;
@property (nonatomic, strong) UIViewController *viewController;  // strong reference

@end

@implementation StrongDemo

- (void)demonstrateStrong {
    NSMutableArray *array = [NSMutableArray arrayWithObjects:@"a", @"b", @"c", nil];

    // strong property ถือ strong reference
    self.mutableItems = array;

    // แม้ local variable array หมด scope, object ยังอยู่เพราะ self.mutableItems ถือไว้
    array = nil;
    NSLog(@"ยังเข้าถึงได้: %@", self.mutableItems);  // ยังทำงานได้
}

@end
```

### 5.2 weak

```objc
// WeakDemo.h
@protocol WeakDelegateProtocol <NSObject>
- (void)didFinishLoading:(id)sender;
@end

@interface WeakDemo : NSObject

// weak - ไม่เพิ่ม retain count, จะเป็น nil เมื่อ object ถูก deallocate
// ใช้สำหรับ: delegate, parent reference, เพื่อหลีกเลี่ยง retain cycle
@property (nonatomic, weak) id<WeakDelegateProtocol> delegate;
@property (nonatomic, weak) WeakDemo *parent;          // parent-child ที่ไม่เป็น cycle
@property (nonatomic, weak) NSObject *observer;        // weak observer

@end
```

```objc
// Retain Cycle ตัวอย่าง (ปัญหา)
@interface PersonNode : NSObject

@property (nonatomic, strong) NSString     *name;
@property (nonatomic, strong) PersonNode  *friend;    // strong -> retain cycle ถ้า A.friend = B, B.friend = A

@end

// วิธีแก้: ใช้ weak สำหรับ relationship ที่ไม่ควร retain
@interface PersonNodeFixed : NSObject

@property (nonatomic, strong) NSString             *name;
@property (nonatomic, weak)   PersonNodeFixed *friend;   // weak -> ไม่มี retain cycle

@end
```

```objc
// ตัวอย่าง retain cycle ใน block
@interface BlockRetainCycle : NSObject

@property (nonatomic, copy)   NSString *name;
@property (nonatomic, copy)   void (^completionBlock)(void);

- (void)setupBlock;

@end

@implementation BlockRetainCycle

- (void)setupBlock {
    // ❌ Retain cycle: self -> completionBlock -> self
    self.completionBlock = ^{
        NSLog(@"Name: %@", self.name);  // strong capture ของ self
    };

    // ✅ วิธีแก้: ใช้ weak reference
    __weak typeof(self) weakSelf = self;
    self.completionBlock = ^{
        __strong typeof(weakSelf) strongSelf = weakSelf;
        if (strongSelf) {
            NSLog(@"Name: %@", strongSelf.name);
        }
    };
}

@end
```

### 5.3 copy

```objc
// CopyDemo.h
@interface CopyDemo : NSObject

// copy - คัดลอก value แทนที่จะถือ reference
// สำคัญมากสำหรับ NSString เพราะอาจได้รับ NSMutableString
@property (nonatomic, copy) NSString      *name;       // ใช้ copy สำหรับ NSString
@property (nonatomic, copy) NSArray       *items;      // ใช้ copy สำหรับ NSArray
@property (nonatomic, copy) NSDictionary  *config;     // ใช้ copy สำหรับ NSDictionary
@property (nonatomic, copy) void (^block)(void);       // ใช้ copy สำหรับ blocks (บังคับ)

@end
```

```objc
// ทำไมต้องใช้ copy สำหรับ NSString
@implementation CopyDemo

- (void)demonstrateCopyImportance {
    NSMutableString *mutable = [NSMutableString stringWithString:@"ต้นฉบับ"];

    // ❌ ถ้าใช้ strong: เก็บ reference ไปยัง mutable string เดิม
    // self.name = mutable;  // ถ้า strong
    // [mutable appendString:@" แก้ไขแล้ว"];
    // NSLog(@"%@", self.name);  // "ต้นฉบับ แก้ไขแล้ว" !!

    // ✅ ถ้าใช้ copy: สร้าง immutable copy
    self.name = mutable;  // copy สร้าง NSString ใหม่จาก NSMutableString
    [mutable appendString:@" แก้ไขแล้ว"];
    NSLog(@"self.name: %@", self.name);    // "ต้นฉบับ" - ไม่ถูกแก้ไข!
    NSLog(@"mutable: %@", mutable);         // "ต้นฉบับ แก้ไขแล้ว"
}

@end
```

### 5.4 assign

```objc
// AssignDemo.h
@interface AssignDemo : NSObject

// assign - ไม่ retain/release, เก็บ value โดยตรง
// ใช้สำหรับ: primitive types, C structs, enums
@property (nonatomic, assign) NSInteger   count;
@property (nonatomic, assign) double      price;
@property (nonatomic, assign) BOOL        isActive;
@property (nonatomic, assign) CGFloat     width;
@property (nonatomic, assign) CGRect      frame;      // C struct
@property (nonatomic, assign) NSRange     range;      // C struct

// ห้ามใช้ assign กับ Objective-C objects! (ใช้ weak แทน)
// @property (nonatomic, assign) NSObject *obj;  // อันตราย! dangling pointer

@end
```

### 5.5 unsafe_unretained

```objc
// UnsafeUnretainedDemo
@interface UnsafeUnretainedDemo : NSObject

// unsafe_unretained - เหมือน assign สำหรับ object types
// ไม่ zeroed เมื่อ object ถูก deallocate -> dangling pointer!
// ใช้กับ: legacy code ก่อน ARC, หรือเมื่อ weak ไม่รองรับ (เช่น __unsafe_unretained)
@property (nonatomic, unsafe_unretained) NSObject *legacyRef;

@end

// ใช้ weak แทน unsafe_unretained เสมอ ยกเว้นมีเหตุผลพิเศษ
```

### 5.6 retain (Pre-ARC)

```objc
// retain ใช้ใน manual memory management (MRC) ก่อน ARC
// ใน ARC ใช้ strong แทน
@interface PreARCClass : NSObject

@property (nonatomic, retain) NSString *name;  // MRC equivalent ของ strong

@end

// ใน ARC ให้ใช้:
@interface ModernClass : NSObject
@property (nonatomic, strong) NSString *name;  // ✅
@end
```

---

## 6. Custom Getter/Setter

### Custom Getter

```objc
// CustomAccessorDemo.h
@interface Temperature : NSObject

@property (nonatomic, assign) double celsius;

// Custom getter names (เมื่อต้องการชื่อ method ที่ต่างออกไป)
@property (nonatomic, assign, getter=isBoiling) BOOL boiling;
@property (nonatomic, assign, getter=isFreezing) BOOL freezing;

// Computed readonly
@property (nonatomic, assign, readonly) double fahrenheit;
@property (nonatomic, assign, readonly) double kelvin;

@end
```

```objc
// Temperature.m
@implementation Temperature

- (instancetype)initWithCelsius:(double)celsius {
    self = [super init];
    if (self) {
        _celsius = celsius;
    }
    return self;
}

// Custom getter สำหรับ fahrenheit (computed)
- (double)fahrenheit {
    return _celsius * 9.0 / 5.0 + 32.0;
}

// Custom getter สำหรับ kelvin (computed)
- (double)kelvin {
    return _celsius + 273.15;
}

// Custom getter พร้อม validation/transformation
- (BOOL)isBoiling {
    return _celsius >= 100.0;
}

- (BOOL)isFreezing {
    return _celsius <= 0.0;
}

// Custom setter พร้อม validation
- (void)setCelsius:(double)celsius {
    if (celsius < -273.15) {
        NSLog(@"ข้อผิดพลาด: อุณหภูมิต่ำกว่า absolute zero (-273.15°C)");
        _celsius = -273.15;
    } else {
        _celsius = celsius;
    }
}

- (NSString *)description {
    return [NSString stringWithFormat:
            @"%.2f°C = %.2f°F = %.2fK",
            _celsius, self.fahrenheit, self.kelvin];
}

@end
```

```objc
// ตัวอย่าง Custom Getter/Setter ที่ซับซ้อน
@interface UserProfile : NSObject

@property (nonatomic, copy)   NSString *firstName;
@property (nonatomic, copy)   NSString *lastName;
@property (nonatomic, assign) NSInteger birthYear;

// Custom getter - computed
@property (nonatomic, copy, readonly) NSString *fullName;
@property (nonatomic, assign, readonly) NSInteger age;
@property (nonatomic, copy, readonly) NSString *initials;

@end

@implementation UserProfile

// Custom getter สำหรับ fullName
- (NSString *)fullName {
    if (_firstName && _lastName) {
        return [NSString stringWithFormat:@"%@ %@", _firstName, _lastName];
    } else if (_firstName) {
        return _firstName;
    } else if (_lastName) {
        return _lastName;
    }
    return @"ไม่มีชื่อ";
}

// Custom getter สำหรับ age
- (NSInteger)age {
    NSInteger currentYear = [[NSCalendar currentCalendar]
                              component:NSCalendarUnitYear
                              fromDate:[NSDate date]];
    return currentYear - _birthYear;
}

// Custom getter สำหรับ initials
- (NSString *)initials {
    NSMutableString *initials = [NSMutableString string];
    if (_firstName.length > 0) {
        [initials appendString:[[_firstName substringToIndex:1] uppercaseString]];
    }
    if (_lastName.length > 0) {
        [initials appendString:[[_lastName substringToIndex:1] uppercaseString]];
    }
    return initials;
}

// Custom setter สำหรับ firstName - normalize
- (void)setFirstName:(NSString *)firstName {
    // Capitalize first letter, trim whitespace
    NSString *trimmed = [firstName stringByTrimmingCharactersInSet:
                         [NSCharacterSet whitespaceAndNewlineCharacterSet]];
    if (trimmed.length > 0) {
        _firstName = [trimmed capitalizedString];
    } else {
        _firstName = nil;
    }
}

// Custom setter สำหรับ birthYear - validate
- (void)setBirthYear:(NSInteger)birthYear {
    NSInteger currentYear = [[NSCalendar currentCalendar]
                              component:NSCalendarUnitYear
                              fromDate:[NSDate date]];
    if (birthYear < 1900 || birthYear > currentYear) {
        NSLog(@"ปีเกิดไม่ถูกต้อง");
        return;
    }
    _birthYear = birthYear;
}

@end
```

---

## 7. Lazy Initialization

Lazy initialization สร้าง object เมื่อต้องการครั้งแรกเท่านั้น ประหยัดหน่วยความจำและเวลาเริ่มต้น

```objc
// LazyInitDemo.h
@interface LazyInitDemo : NSObject

@property (nonatomic, strong) NSMutableArray  *expensiveArray;
@property (nonatomic, strong) NSDictionary    *configData;
@property (nonatomic, strong) NSDateFormatter *dateFormatter;

- (void)doWork;

@end
```

```objc
// LazyInitDemo.m
@implementation LazyInitDemo

// Lazy getter สำหรับ expensiveArray
- (NSMutableArray *)expensiveArray {
    if (_expensiveArray == nil) {
        NSLog(@"สร้าง expensiveArray ครั้งแรก...");
        _expensiveArray = [NSMutableArray array];
        // เติมข้อมูลเริ่มต้น (expensive operation)
        for (int i = 0; i < 1000; i++) {
            [_expensiveArray addObject:@(i * i)];
        }
    }
    return _expensiveArray;
}

// Lazy getter สำหรับ configData
- (NSDictionary *)configData {
    if (_configData == nil) {
        NSLog(@"โหลด configData จากไฟล์...");
        // จำลองการโหลดจากไฟล์
        _configData = @{
            @"maxRetry": @3,
            @"timeout":  @30,
            @"baseURL":  @"https://api.example.com",
            @"version":  @"1.0"
        };
    }
    return _configData;
}

// Lazy getter สำหรับ dateFormatter (expensive to create)
- (NSDateFormatter *)dateFormatter {
    if (_dateFormatter == nil) {
        _dateFormatter = [[NSDateFormatter alloc] init];
        _dateFormatter.dateFormat = @"yyyy-MM-dd HH:mm:ss";
        _dateFormatter.locale = [NSLocale localeWithLocaleIdentifier:@"th_TH"];
    }
    return _dateFormatter;
}

- (void)doWork {
    // dateFormatter สร้างครั้งเดียว ใช้หลายครั้ง
    NSString *now = [self.dateFormatter stringFromDate:[NSDate date]];
    NSLog(@"เวลาปัจจุบัน: %@", now);

    // expensiveArray สร้างเมื่อต้องการครั้งแรก
    NSLog(@"จำนวน items: %lu", (unsigned long)self.expensiveArray.count);

    // configData โหลดครั้งเดียว
    NSLog(@"Version: %@", self.configData[@"version"]);
}

@end
```

```objc
// Thread-safe Lazy Initialization
@interface ThreadSafeManager : NSObject

@property (nonatomic, strong, readonly) NSOperationQueue *backgroundQueue;
@property (nonatomic, strong, readonly) NSCache          *imageCache;

@end

@implementation ThreadSafeManager {
    NSOperationQueue *_backgroundQueue;
    NSCache          *_imageCache;
    dispatch_once_t   _queueOnce;
    dispatch_once_t   _cacheOnce;
}

- (NSOperationQueue *)backgroundQueue {
    dispatch_once(&_queueOnce, ^{
        _backgroundQueue = [[NSOperationQueue alloc] init];
        _backgroundQueue.maxConcurrentOperationCount = 4;
        _backgroundQueue.name = @"com.example.background";
    });
    return _backgroundQueue;
}

- (NSCache *)imageCache {
    dispatch_once(&_cacheOnce, ^{
        _imageCache = [[NSCache alloc] init];
        _imageCache.countLimit = 100;
        _imageCache.totalCostLimit = 50 * 1024 * 1024;  // 50 MB
    });
    return _imageCache;
}

@end
```

---

## 8. Class Extensions สำหรับ Private Properties

Class Extension (ก็คือ Anonymous Category) เป็นวิธีหลักในการ declare private properties/methods

```objc
// APIClient.h - Public interface เท่านั้น
@interface APIClient : NSObject

@property (nonatomic, copy, readonly) NSURL *baseURL;

- (instancetype)initWithBaseURL:(NSURL *)baseURL;
- (void)fetchDataFromEndpoint:(NSString *)endpoint
                   completion:(void(^)(NSDictionary *data, NSError *error))completion;
- (void)cancelAllRequests;

@end
```

```objc
// APIClient.m - Private implementation
// Class Extension ใน .m file เพื่อ declare private members
@interface APIClient ()

// Private properties - ไม่ต้องการให้ภายนอกเห็น
@property (nonatomic, copy,   readwrite) NSURL            *baseURL;
@property (nonatomic, strong)            NSURLSession      *session;
@property (nonatomic, strong)            NSMutableArray    *activeTasks;
@property (nonatomic, strong)            NSMutableDictionary *headers;
@property (nonatomic, assign)            BOOL               isAuthenticated;

// Private method declarations
- (NSURLRequest *)buildRequestForEndpoint:(NSString *)endpoint;
- (void)handleResponse:(NSData *)data
                 error:(NSError *)error
            completion:(void(^)(NSDictionary *data, NSError *error))completion;
- (void)addDefaultHeaders:(NSMutableURLRequest *)request;

@end

@implementation APIClient

- (instancetype)initWithBaseURL:(NSURL *)baseURL {
    self = [super init];
    if (self) {
        _baseURL          = [baseURL copy];
        _session          = [NSURLSession sharedSession];
        _activeTasks      = [NSMutableArray array];
        _headers          = [NSMutableDictionary dictionary];
        _isAuthenticated  = NO;

        // Default headers
        _headers[@"Content-Type"] = @"application/json";
        _headers[@"Accept"]       = @"application/json";
    }
    return self;
}

// Private method - ซ่อนจาก public interface
- (NSURLRequest *)buildRequestForEndpoint:(NSString *)endpoint {
    NSURL *url = [NSURL URLWithString:endpoint relativeToURL:_baseURL];
    NSMutableURLRequest *request = [NSMutableURLRequest requestWithURL:url];
    [self addDefaultHeaders:request];
    return request;
}

- (void)addDefaultHeaders:(NSMutableURLRequest *)request {
    for (NSString *key in _headers) {
        [request setValue:_headers[key] forHTTPHeaderField:key];
    }
}

- (void)handleResponse:(NSData *)data
                 error:(NSError *)error
            completion:(void(^)(NSDictionary *data, NSError *error))completion {
    if (error) {
        dispatch_async(dispatch_get_main_queue(), ^{
            completion(nil, error);
        });
        return;
    }

    NSError *parseError;
    NSDictionary *json = [NSJSONSerialization JSONObjectWithData:data
                                                         options:0
                                                           error:&parseError];
    dispatch_async(dispatch_get_main_queue(), ^{
        completion(json, parseError);
    });
}

- (void)fetchDataFromEndpoint:(NSString *)endpoint
                   completion:(void(^)(NSDictionary *data, NSError *error))completion {
    NSURLRequest *request = [self buildRequestForEndpoint:endpoint];

    NSURLSessionDataTask *task = [_session dataTaskWithRequest:request
                                             completionHandler:^(NSData *data,
                                                                 NSURLResponse *response,
                                                                 NSError *error) {
        [self handleResponse:data error:error completion:completion];
    }];

    [_activeTasks addObject:task];
    [task resume];
}

- (void)cancelAllRequests {
    for (NSURLSessionTask *task in _activeTasks) {
        [task cancel];
    }
    [_activeTasks removeAllObjects];
}

@end
```

### Class Extension สำหรับ Redeclaring readonly เป็น readwrite

```objc
// Config.h - ภายนอกอ่านได้อย่างเดียว
@interface Config : NSObject

@property (nonatomic, copy,   readonly) NSString  *appName;
@property (nonatomic, copy,   readonly) NSString  *version;
@property (nonatomic, assign, readonly) BOOL       isDevelopment;
@property (nonatomic, strong, readonly) NSDictionary *settings;

+ (instancetype)sharedConfig;

@end
```

```objc
// Config.m - ภายใน class เขียนได้
@interface Config ()  // Class extension

@property (nonatomic, copy,   readwrite) NSString     *appName;
@property (nonatomic, copy,   readwrite) NSString     *version;
@property (nonatomic, assign, readwrite) BOOL          isDevelopment;
@property (nonatomic, strong, readwrite) NSDictionary *settings;

- (void)loadFromBundle;

@end

@implementation Config

static Config *_sharedConfig = nil;

+ (instancetype)sharedConfig {
    static dispatch_once_t onceToken;
    dispatch_once(&onceToken, ^{
        _sharedConfig = [[Config alloc] init];
        [_sharedConfig loadFromBundle];
    });
    return _sharedConfig;
}

- (void)loadFromBundle {
    // โหลดจาก Info.plist
    NSDictionary *info = [[NSBundle mainBundle] infoDictionary];
    _appName       = info[@"CFBundleDisplayName"] ?: @"MyApp";
    _version       = info[@"CFBundleShortVersionString"] ?: @"1.0";
    _isDevelopment = [[info[@"BuildConfiguration"] ?: @"Release"] isEqual:@"Debug"];
    _settings      = info[@"AppSettings"] ?: @{};
}

@end
```

---

## 9. Property vs ivar Performance

```objc
// PerformanceComparisonDemo
@interface PerformanceDemo : NSObject {
    NSInteger _directIvar;      // direct ivar access
}

@property (nonatomic, assign) NSInteger propertyValue;
@property (nonatomic, assign) NSInteger anotherProperty;

@end

@implementation PerformanceDemo

// ภายใน class:
// _ivar   -> เร็วที่สุด (direct memory access)
// self->_ivar -> เหมือนกับ _ivar
// self.property -> ช้ากว่า (ผ่าน method call / objc_msgSend)

- (void)performanceComparison {
    NSInteger result = 0;
    NSInteger iterations = 1000000;

    // การเข้าถึง ivar โดยตรง (เร็วที่สุด)
    NSDate *start = [NSDate date];
    for (NSInteger i = 0; i < iterations; i++) {
        _directIvar = i;
        result += _directIvar;
    }
    NSLog(@"Direct ivar: %.4f วินาที", -[start timeIntervalSinceNow]);

    // การเข้าถึงผ่าน property (ช้ากว่าเล็กน้อย)
    start = [NSDate date];
    for (NSInteger i = 0; i < iterations; i++) {
        self.propertyValue = i;
        result += self.propertyValue;
    }
    NSLog(@"Property: %.4f วินาที", -[start timeIntervalSinceNow]);
}

// คำแนะนำการใช้:
// ใน init/dealloc: เข้าถึง ivar โดยตรง (_ivar)
// ใน methods อื่น: ใช้ property (self.property) - encapsulation ดีกว่า
// ถ้าต้องการประสิทธิภาพสูง (inner loops): ใช้ local variable แทน

- (void)optimizedMethod {
    // Cache property ใน local variable สำหรับ inner loop
    NSMutableArray *items = self.expensiveItems;  // อ่านครั้งเดียว
    NSInteger count = items.count;                 // cache count

    for (NSInteger i = 0; i < count; i++) {
        // ใช้ local variables แทน property calls ซ้ำๆ
        id item = items[i];
        [self processItem:item];
    }
}

@property (nonatomic, strong) NSMutableArray *expensiveItems;
- (void)processItem:(id)item {}

@end
```

---

## 10. KVC (Key-Value Coding) Compliance

Properties ที่ทำ KVC compliant ได้จะทำให้ใช้ `valueForKey:` และ `setValue:forKey:` ได้

```objc
// KVCDemo.h
@interface Person : NSObject

@property (nonatomic, copy)   NSString  *name;
@property (nonatomic, assign) NSInteger  age;
@property (nonatomic, copy)   NSString  *email;
@property (nonatomic, strong) NSArray   *skills;

@end
```

```objc
// KVCDemo.m
@implementation Person

// KVC ต้องการ getter ชื่อตรงกับ property name
// และ setter ชื่อ set<PropertyName>:
// ซึ่ง @property สร้างให้อัตโนมัติ -> KVC compliant โดยอัตโนมัติ

@end
```

```objc
// การใช้งาน KVC
Person *p = [[Person alloc] init];

// valueForKey: - เทียบเท่า p.name
[p setValue:@"สมชาย" forKey:@"name"];
[p setValue:@(25)    forKey:@"age"];
[p setValue:@"somchai@example.com" forKey:@"email"];

NSString *name = [p valueForKey:@"name"];  // เทียบเท่า p.name
NSInteger age  = [[p valueForKey:@"age"] integerValue];
NSLog(@"ชื่อ: %@, อายุ: %ld", name, (long)age);

// Key Path
NSDictionary *data = @{
    @"name":  @"สมหญิง",
    @"age":   @(30),
    @"email": @"somying@example.com"
};

// ตั้งค่าหลาย property พร้อมกัน
[p setValuesForKeysWithDictionary:data];
NSLog(@"อัพเดต: %@ (อายุ %ld)", p.name, (long)p.age);

// NSDictionary จาก object
NSDictionary *dict = [p dictionaryWithValuesForKeys:@[@"name", @"age", @"email"]];
NSLog(@"เป็น Dictionary: %@", dict);
```

### KVC กับ Array

```objc
// KVC operator
NSArray *people = @[
    ({Person *p = [Person new]; p.name = @"สมชาย"; p.age = 25; p; }),
    ({Person *p = [Person new]; p.name = @"สมหญิง"; p.age = 30; p; }),
    ({Person *p = [Person new]; p.name = @"สมพงษ์"; p.age = 22; p; }),
];

// ดึง array ของ names
NSArray *names = [people valueForKey:@"name"];
NSLog(@"ชื่อทั้งหมด: %@", names);

// ใช้ KVC operators
NSNumber *avgAge   = [people valueForKeyPath:@"@avg.age"];
NSNumber *maxAge   = [people valueForKeyPath:@"@max.age"];
NSNumber *minAge   = [people valueForKeyPath:@"@min.age"];
NSNumber *sumAge   = [people valueForKeyPath:@"@sum.age"];
NSNumber *countP   = [people valueForKeyPath:@"@count"];

NSLog(@"อายุเฉลี่ย: %.1f", [avgAge doubleValue]);
NSLog(@"อายุมากสุด: %ld", [maxAge longValue]);
NSLog(@"จำนวนคน: %ld", [countP longValue]);
```

---

## 11. ตัวอย่างจากโลกจริง: Configuration Manager

```objc
// AppConfiguration.h
#import <Foundation/Foundation.h>

typedef NS_ENUM(NSInteger, Environment) {
    EnvironmentDevelopment,
    EnvironmentStaging,
    EnvironmentProduction
};

@interface AppConfiguration : NSObject

// Public readonly properties
@property (nonatomic, copy,   readonly) NSString     *appName;
@property (nonatomic, copy,   readonly) NSString     *appVersion;
@property (nonatomic, copy,   readonly) NSString     *buildNumber;
@property (nonatomic, assign, readonly) Environment   environment;
@property (nonatomic, copy,   readonly) NSURL        *apiBaseURL;
@property (nonatomic, assign, readonly) NSTimeInterval networkTimeout;
@property (nonatomic, assign, readonly) BOOL          analyticsEnabled;
@property (nonatomic, assign, readonly) BOOL          crashReportingEnabled;
@property (nonatomic, copy,   readonly) NSString     *environmentName;

// Public methods
+ (instancetype)sharedConfiguration;
- (id)valueForConfigKey:(NSString *)key;
- (BOOL)featureFlagEnabled:(NSString *)flagName;

@end
```

```objc
// AppConfiguration.m
@interface AppConfiguration ()

// Redeclare as readwrite for internal use
@property (nonatomic, copy,   readwrite) NSString     *appName;
@property (nonatomic, copy,   readwrite) NSString     *appVersion;
@property (nonatomic, copy,   readwrite) NSString     *buildNumber;
@property (nonatomic, assign, readwrite) Environment   environment;
@property (nonatomic, copy,   readwrite) NSURL        *apiBaseURL;
@property (nonatomic, assign, readwrite) NSTimeInterval networkTimeout;
@property (nonatomic, assign, readwrite) BOOL          analyticsEnabled;
@property (nonatomic, assign, readwrite) BOOL          crashReportingEnabled;

// Private properties
@property (nonatomic, strong) NSDictionary *rawConfig;
@property (nonatomic, strong) NSDictionary *featureFlags;

// Private methods
- (void)loadConfiguration;
- (void)configureForEnvironment:(Environment)env;
- (NSDictionary *)defaultFeatureFlags;

@end

@implementation AppConfiguration

static AppConfiguration *_sharedConfig = nil;

+ (instancetype)sharedConfiguration {
    static dispatch_once_t onceToken;
    dispatch_once(&onceToken, ^{
        _sharedConfig = [[AppConfiguration alloc] init];
    });
    return _sharedConfig;
}

- (instancetype)init {
    self = [super init];
    if (self) {
        [self loadConfiguration];
    }
    return self;
}

- (void)loadConfiguration {
    // โหลด App Info
    NSDictionary *info = [[NSBundle mainBundle] infoDictionary];
    _appName     = info[@"CFBundleDisplayName"] ?: info[@"CFBundleName"] ?: @"App";
    _appVersion  = info[@"CFBundleShortVersionString"] ?: @"1.0";
    _buildNumber = info[@"CFBundleVersion"] ?: @"1";

    // กำหนด environment
    NSString *envString = [[NSProcessInfo processInfo].environment[@"APP_ENV"] ?: @"production"];
    if ([envString isEqualToString:@"development"]) {
        _environment = EnvironmentDevelopment;
    } else if ([envString isEqualToString:@"staging"]) {
        _environment = EnvironmentStaging;
    } else {
        _environment = EnvironmentProduction;
    }

    [self configureForEnvironment:_environment];

    // โหลด feature flags
    _featureFlags = [self defaultFeatureFlags];
}

- (void)configureForEnvironment:(Environment)env {
    switch (env) {
        case EnvironmentDevelopment:
            _apiBaseURL             = [NSURL URLWithString:@"http://localhost:3000/api/v1"];
            _networkTimeout         = 60.0;
            _analyticsEnabled       = NO;
            _crashReportingEnabled  = NO;
            break;

        case EnvironmentStaging:
            _apiBaseURL             = [NSURL URLWithString:@"https://staging.api.example.com/v1"];
            _networkTimeout         = 30.0;
            _analyticsEnabled       = YES;
            _crashReportingEnabled  = YES;
            break;

        case EnvironmentProduction:
            _apiBaseURL             = [NSURL URLWithString:@"https://api.example.com/v1"];
            _networkTimeout         = 30.0;
            _analyticsEnabled       = YES;
            _crashReportingEnabled  = YES;
            break;
    }
}

- (NSDictionary *)defaultFeatureFlags {
    return @{
        @"newOnboarding":    @YES,
        @"darkMode":         @YES,
        @"betaFeatures":     @(_environment == EnvironmentDevelopment),
        @"offlineMode":      @NO,
        @"advancedSearch":   @YES,
    };
}

// Computed readonly
- (NSString *)environmentName {
    switch (_environment) {
        case EnvironmentDevelopment: return @"Development";
        case EnvironmentStaging:     return @"Staging";
        case EnvironmentProduction:  return @"Production";
    }
}

- (id)valueForConfigKey:(NSString *)key {
    return _rawConfig[key];
}

- (BOOL)featureFlagEnabled:(NSString *)flagName {
    return [_featureFlags[flagName] boolValue];
}

- (NSString *)description {
    return [NSString stringWithFormat:
            @"AppConfig{name=%@, version=%@, env=%@, api=%@}",
            _appName, _appVersion, self.environmentName, _apiBaseURL];
}

@end
```

---

## 12. ตัวอย่างจากโลกจริง: UserSettings

```objc
// UserSettings.h - Encapsulated settings management
#import <Foundation/Foundation.h>

@interface UserSettings : NSObject

// User preferences
@property (nonatomic, copy)   NSString   *language;
@property (nonatomic, copy)   NSString   *theme;           // light, dark, auto
@property (nonatomic, assign) BOOL        notificationsEnabled;
@property (nonatomic, assign) NSInteger   fontSize;
@property (nonatomic, assign) BOOL        biometricEnabled;

// Derived/computed
@property (nonatomic, assign, readonly) BOOL isDarkMode;
@property (nonatomic, copy,   readonly) NSLocale *locale;

// Factory
+ (instancetype)sharedSettings;

// Persistence
- (void)save;
- (void)resetToDefaults;

@end
```

```objc
// UserSettings.m
@interface UserSettings ()

@property (nonatomic, strong) NSUserDefaults *defaults;

@end

@implementation UserSettings

static NSString * const kLanguageKey             = @"user.language";
static NSString * const kThemeKey                = @"user.theme";
static NSString * const kNotificationsKey        = @"user.notifications";
static NSString * const kFontSizeKey             = @"user.fontSize";
static NSString * const kBiometricKey            = @"user.biometric";

+ (instancetype)sharedSettings {
    static UserSettings *instance = nil;
    static dispatch_once_t onceToken;
    dispatch_once(&onceToken, ^{
        instance = [[UserSettings alloc] init];
    });
    return instance;
}

- (instancetype)init {
    self = [super init];
    if (self) {
        _defaults = [NSUserDefaults standardUserDefaults];
        [self loadFromDefaults];
    }
    return self;
}

- (void)loadFromDefaults {
    _language              = [_defaults stringForKey:kLanguageKey] ?: @"th";
    _theme                 = [_defaults stringForKey:kThemeKey]    ?: @"auto";
    _notificationsEnabled  = [_defaults boolForKey:kNotificationsKey];
    _fontSize              = [_defaults integerForKey:kFontSizeKey] ?: 16;
    _biometricEnabled      = [_defaults boolForKey:kBiometricKey];
}

// Custom setter สำหรับ language - validate
- (void)setLanguage:(NSString *)language {
    NSArray *supported = @[@"th", @"en", @"zh", @"ja", @"ko"];
    if ([supported containsObject:language]) {
        _language = [language copy];
    } else {
        NSLog(@"ภาษาที่ไม่รองรับ: %@", language);
    }
}

// Custom setter สำหรับ theme - validate
- (void)setTheme:(NSString *)theme {
    NSArray *valid = @[@"light", @"dark", @"auto"];
    _theme = [valid containsObject:theme] ? [theme copy] : @"auto";
}

// Custom setter สำหรับ fontSize - validate range
- (void)setFontSize:(NSInteger)fontSize {
    _fontSize = MAX(12, MIN(24, fontSize));  // clamp 12-24
}

// Computed readonly property
- (BOOL)isDarkMode {
    if ([_theme isEqualToString:@"dark"]) return YES;
    if ([_theme isEqualToString:@"light"]) return NO;
    // auto - ตรวจระบบ
    return NO;  // simplified - จริงๆ ต้องตรวจ UIUserInterfaceStyle
}

// Computed readonly property
- (NSLocale *)locale {
    return [NSLocale localeWithLocaleIdentifier:_language];
}

- (void)save {
    [_defaults setObject:_language              forKey:kLanguageKey];
    [_defaults setObject:_theme                 forKey:kThemeKey];
    [_defaults setBool:_notificationsEnabled    forKey:kNotificationsKey];
    [_defaults setInteger:_fontSize             forKey:kFontSizeKey];
    [_defaults setBool:_biometricEnabled        forKey:kBiometricKey];
    [_defaults synchronize];
    NSLog(@"บันทึกการตั้งค่าสำเร็จ");
}

- (void)resetToDefaults {
    _language             = @"th";
    _theme                = @"auto";
    _notificationsEnabled = YES;
    _fontSize             = 16;
    _biometricEnabled     = NO;
    [self save];
    NSLog(@"รีเซ็ตการตั้งค่าเรียบร้อย");
}

- (NSString *)description {
    return [NSString stringWithFormat:
            @"UserSettings{lang=%@, theme=%@, notifications=%@, fontSize=%ld}",
            _language, _theme, _notificationsEnabled ? @"on" : @"off", (long)_fontSize];
}

@end
```

---

## 13. Property Observer Pattern

```objc
// PropertyObserver.h - ใช้ KVO (Key-Value Observing) กับ Properties
@interface StockPrice : NSObject

@property (nonatomic, copy)   NSString *symbol;
@property (nonatomic, assign) double    price;
@property (nonatomic, assign) double    change;         // การเปลี่ยนแปลง
@property (nonatomic, assign) double    changePercent;  // เปอร์เซ็นต์การเปลี่ยนแปลง
@property (nonatomic, assign, readonly) BOOL isGaining; // กำลังขึ้นหรือเปล่า

- (instancetype)initWithSymbol:(NSString *)symbol price:(double)price;
- (void)updatePrice:(double)newPrice;

@end
```

```objc
// PropertyObserver.m
@interface StockPrice ()
@property (nonatomic, assign) double previousPrice;
@end

@implementation StockPrice

- (instancetype)initWithSymbol:(NSString *)symbol price:(double)price {
    self = [super init];
    if (self) {
        _symbol        = [symbol copy];
        _price         = price;
        _previousPrice = price;
        _change        = 0;
        _changePercent = 0;
    }
    return self;
}

// Custom setter สำหรับ price - คำนวณ change อัตโนมัติ
- (void)setPrice:(double)price {
    _previousPrice = _price;
    _price         = price;
    _change        = _price - _previousPrice;
    _changePercent = _previousPrice != 0 ? (_change / _previousPrice * 100) : 0;
}

- (BOOL)isGaining {
    return _change >= 0;
}

- (void)updatePrice:(double)newPrice {
    self.price = newPrice;  // เรียกผ่าน setter เพื่อคำนวณ change
    NSLog(@"[%@] ราคา: %.2f (%@%.2f / %.2f%%)",
          _symbol, _price,
          self.isGaining ? @"+" : @"",
          _change, _changePercent);
}

@end
```

```objc
// KVO Observer
@interface PriceAlert : NSObject

@property (nonatomic, strong) StockPrice *stock;
@property (nonatomic, assign) double      alertPrice;

- (instancetype)initForStock:(StockPrice *)stock alertAt:(double)price;

@end

@implementation PriceAlert

- (instancetype)initForStock:(StockPrice *)stock alertAt:(double)price {
    self = [super init];
    if (self) {
        _stock      = stock;
        _alertPrice = price;
        // KVO: สังเกต price property
        [stock addObserver:self
                forKeyPath:@"price"
                   options:NSKeyValueObservingOptionNew | NSKeyValueObservingOptionOld
                   context:NULL];
    }
    return self;
}

- (void)observeValueForKeyPath:(NSString *)keyPath
                      ofObject:(id)object
                        change:(NSDictionary *)change
                       context:(void *)context {
    if ([keyPath isEqualToString:@"price"]) {
        double newPrice = [change[NSKeyValueChangeNewKey] doubleValue];
        double oldPrice = [change[NSKeyValueChangeOldKey] doubleValue];

        if (newPrice >= _alertPrice && oldPrice < _alertPrice) {
            NSLog(@"⚠️ แจ้งเตือน: [%@] ราคาถึง %.2f แล้ว!", _stock.symbol, newPrice);
        }
    }
}

- (void)dealloc {
    [_stock removeObserver:self forKeyPath:@"price"];
}

@end
```

---

## 14. Encapsulation Patterns ในโปรเจกต์จริง

### Builder Pattern ด้วย Encapsulation

```objc
// NetworkRequest.h
@interface NetworkRequest : NSObject

@property (nonatomic, copy,   readonly) NSURL              *url;
@property (nonatomic, copy,   readonly) NSString           *method;
@property (nonatomic, strong, readonly) NSDictionary       *headers;
@property (nonatomic, strong, readonly) NSData             *body;
@property (nonatomic, assign, readonly) NSTimeInterval      timeout;

// ไม่มี public initializer - ต้องใช้ Builder
+ (id)builder;

@end

// Builder
@interface NetworkRequestBuilder : NSObject

- (NetworkRequestBuilder *)withURL:(NSURL *)url;
- (NetworkRequestBuilder *)withMethod:(NSString *)method;
- (NetworkRequestBuilder *)withHeader:(NSString *)value forKey:(NSString *)key;
- (NetworkRequestBuilder *)withJSONBody:(NSDictionary *)body;
- (NetworkRequestBuilder *)withTimeout:(NSTimeInterval)timeout;
- (NetworkRequest *)build;

@end
```

```objc
// NetworkRequest.m
@interface NetworkRequest ()

@property (nonatomic, copy,   readwrite) NSURL        *url;
@property (nonatomic, copy,   readwrite) NSString     *method;
@property (nonatomic, strong, readwrite) NSDictionary *headers;
@property (nonatomic, strong, readwrite) NSData       *body;
@property (nonatomic, assign, readwrite) NSTimeInterval timeout;

@end

@implementation NetworkRequest

+ (id)builder {
    return [[NetworkRequestBuilder alloc] init];
}

- (instancetype)init {
    self = [super init];
    if (self) {
        _method  = @"GET";
        _headers = @{};
        _timeout = 30.0;
    }
    return self;
}

@end

@implementation NetworkRequestBuilder {
    NetworkRequest *_request;
}

- (instancetype)init {
    self = [super init];
    if (self) {
        _request = [[NetworkRequest alloc] init];
    }
    return self;
}

- (NetworkRequestBuilder *)withURL:(NSURL *)url {
    _request.url = url;
    return self;
}

- (NetworkRequestBuilder *)withMethod:(NSString *)method {
    _request.method = [method uppercaseString];
    return self;
}

- (NetworkRequestBuilder *)withHeader:(NSString *)value forKey:(NSString *)key {
    NSMutableDictionary *headers = [_request.headers mutableCopy];
    headers[key] = value;
    _request.headers = [headers copy];
    return self;
}

- (NetworkRequestBuilder *)withJSONBody:(NSDictionary *)body {
    NSData *data = [NSJSONSerialization dataWithJSONObject:body options:0 error:nil];
    _request.body = data;
    return self;
}

- (NetworkRequestBuilder *)withTimeout:(NSTimeInterval)timeout {
    _request.timeout = timeout;
    return self;
}

- (NetworkRequest *)build {
    NSAssert(_request.url != nil, @"URL ต้องไม่เป็น nil");
    return _request;
}

@end
```

```objc
// การใช้งาน Builder
NetworkRequest *request = [[[[[NetworkRequest builder]
    withURL:[NSURL URLWithString:@"https://api.example.com/users"]]
    withMethod:@"POST"]
    withHeader:@"Bearer token123" forKey:@"Authorization"]
    withJSONBody:@{@"name": @"สมชาย", @"email": @"somchai@example.com"}]
    build];

NSLog(@"URL: %@", request.url);
NSLog(@"Method: %@", request.method);
NSLog(@"Headers: %@", request.headers);
```

---

## 15. แบบฝึกหัด (10+ ข้อ)

### ข้อ 1: Secure Password Manager
สร้าง class `PasswordEntry` ที่มี:
- `@private` ivar สำหรับ raw password
- Properties: username, website, notes (ทั้งหมด copy)
- Custom getter สำหรับ maskedPassword (แสดงแค่ 4 ตัวสุดท้าย)
- Custom setter สำหรับ password (validate: min 8 chars, complexity)
- Class extension สำหรับ encryption/decryption methods

### ข้อ 2: Observable Array
สร้าง class `ObservableArray` ที่มี:
- Private `NSMutableArray` backing store
- Public `NSArray` readonly property (copy)
- Methods ที่ trigger KVO: addObject:, removeObjectAtIndex:, insertObject:atIndex:
- Delegate protocol สำหรับ change notifications

### ข้อ 3: Thread-safe Cache
สร้าง class `ThreadSafeCache` ที่มี:
- Private `NSMutableDictionary` ด้วย `dispatch_queue` สำหรับ thread safety
- Properties: countLimit, totalCostLimit
- Methods: setObject:forKey:cost:, objectForKey:, removeObjectForKey:, removeAllObjects
- Computed readonly properties: count, totalCost

### ข้อ 4: Validated Model
สร้าง class `ContactInfo` ที่มี:
- Properties: firstName, lastName, phone, email, website
- Custom setters ทุกตัวที่ validate input:
  - phone: ต้องเป็นตัวเลข 10 หลัก
  - email: ต้องมี @ และ .
  - website: ต้องขึ้นต้นด้วย http:// หรือ https://
- `isValid` computed property
- `validationErrors` computed property (array ของ error messages)

### ข้อ 5: Immutable Value Object
สร้าง class `Money` ที่ immutable โดยสมบูรณ์:
- `amount` และ `currency` เป็น readonly
- ทุก operation สร้าง object ใหม่ (add:, subtract:, multiply:, convertTo:rate:)
- Implement NSCopying, isEqual:, hash
- Class methods: zeroUSD, zeroBaht

### ข้อ 6: Smart Property with Dependencies
สร้าง class `Rectangle` ที่:
- Properties: width, height, area (computed), perimeter (computed), diagonal (computed)
- Custom setters ที่ notify เมื่อ area/perimeter เปลี่ยน
- Lazy property: `boundingCircle` (Circle object)
- isSquare computed property

### ข้อ 7: Configuration with Defaults
สร้าง class `GameConfig` ที่:
- Properties ทั้งหมดมี default values
- สามารถ load จาก Dictionary ได้ด้วย `loadFromDictionary:`
- สามารถ export เป็น Dictionary ได้ด้วย `toDictionary`
- Private validation method ที่ตรวจสอบ consistency ระหว่าง properties
- Class method `defaultConfig`, `hardMode`, `easyMode`

### ข้อ 8: Lazy Property Chain
สร้าง class `DocumentProcessor` ที่มี:
- Properties ที่ lazy init และ depend กัน:
  - `rawContent` (load จากไฟล์ครั้งแรกที่ใช้)
  - `parsedLines` (แบ่ง rawContent ตาม newline, lazy)
  - `wordCount` (นับคำจาก parsedLines, lazy)
  - `uniqueWords` (NSSet จาก words, lazy)
- เมื่อ reset rawContent ต้อง invalidate ทุก derived properties

### ข้อ 9: Fluent Property Interface
สร้าง class `QueryBuilder` ที่:
- ใช้ properties เป็น builder (return self)
- `from:`, `where:`, `orderBy:`, `limit:`, `offset:`
- `execute` method คืน query string
- ทุก method validate input
- Private `buildQuery` method

### ข้อ 10: Encapsulated State Machine
สร้าง class `OrderStateMachine` ที่:
- State enum: Pending, Processing, Shipped, Delivered, Cancelled
- `currentState` เป็น readonly property
- Private method `canTransitionTo:` ที่ตรวจสอบ valid transitions
- Public methods: `process`, `ship`, `deliver`, `cancel` - เปลี่ยน state เมื่อถูกต้อง
- Delegate protocol สำหรับ state change notifications

### ข้อ 11 (ท้าทาย): Property with Undo/Redo
สร้าง class `UndoableProperty` ที่:
- เก็บ history ของ values
- `value` property ที่เมื่อ set จะ push ค่าเก่าลง history
- `undo`, `redo` methods
- `canUndo`, `canRedo` computed properties
- `maxHistorySize` property (limit history)

### ข้อ 12 (ท้าทาย): Reactive Properties
สร้าง `Observable<T>` pattern:
- `Observable` class ที่ wrap value
- `subscribe:` method รับ block
- เมื่อ value เปลี่ยน ทุก subscriber ถูกแจ้งเตือน
- `unsubscribeAll` method
- ใช้ `NSMutableArray` เก็บ weak references ของ subscribers

---

## สรุป Part 13

ใน Part นี้เราได้เรียนรู้:

1. **Access Control** - @private, @protected, @public, @package สำหรับ ivars
2. **Properties** - กลไก encapsulation สมัยใหม่ที่แทน ivars
3. **Property Attributes ครบถ้วน**:
   - Atomicity: `atomic`, `nonatomic`
   - Access: `readonly`, `readwrite`
   - Memory: `strong`, `weak`, `copy`, `assign`, `unsafe_unretained`
4. **Custom Getter/Setter** - override accessor methods สำหรับ validation และ computation
5. **Lazy Initialization** - สร้าง object เมื่อต้องการครั้งแรก
6. **Class Extensions** - declare private properties/methods ใน .m file
7. **Property vs ivar Performance** - เมื่อควรใช้อะไร
8. **KVC Compliance** - valueForKey: และ setValue:forKey: กับ Properties
9. **Real-world Examples** - BankAccount, Configuration, UserSettings
10. **Builder Pattern** ด้วย Encapsulation

Encapsulation ที่ดีทำให้ code:
- **ปลอดภัยกว่า** - ป้องกัน invalid state
- **อ่านง่ายกว่า** - interface ชัดเจน
- **บำรุงรักษาง่ายกว่า** - เปลี่ยน implementation ได้โดยไม่กระทบ users
- **ทดสอบง่ายกว่า** - mock ได้ง่าย

ใน Part 14 จะเรียนเรื่อง **Polymorphism and Protocols** การใช้ polymorphism อย่างเต็มรูปแบบ
