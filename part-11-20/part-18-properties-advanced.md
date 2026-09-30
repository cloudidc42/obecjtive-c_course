# ตอนที่ 18: Properties ขั้นสูง (Advanced Properties)

## บทนำ

ในตอนที่ผ่านมาเราได้เรียนรู้พื้นฐานของ Properties ใน Objective-C กันแล้ว ในตอนนี้เราจะเจาะลึกไปยังฟีเจอร์ขั้นสูงที่ทำให้ Properties มีความสามารถมากขึ้น ทั้ง thread-safe properties, computed properties, observable properties, copy semantics, weak references, property สำหรับ blocks, struct properties, class properties, และ KVO-compliant properties

Properties ขั้นสูงเหล่านี้เป็นหัวใจสำคัญของการเขียนโปรแกรม Objective-C ที่มีคุณภาพ เพราะช่วยให้โค้ดมีความปลอดภัย, ยืดหยุ่น, และบำรุงรักษาได้ง่าย

---

## 18.1 Property Synthesis: Auto vs Manual

### Auto Synthesis (การสังเคราะห์อัตโนมัติ)

ตั้งแต่ Xcode 4.4 เป็นต้นมา Objective-C รองรับ auto synthesis ซึ่งหมายความว่าคุณไม่จำเป็นต้องเขียน `@synthesize` เพื่อสร้าง backing instance variable (ivar) สำหรับ property

```objc
// Person.h
@interface Person : NSObject

@property (nonatomic, strong) NSString *name;
@property (nonatomic, assign) NSInteger age;
@property (nonatomic, copy) NSString *email;

@end
```

```objc
// Person.m - ไม่จำเป็นต้องเขียน @synthesize
@implementation Person

// Auto synthesis จะสร้าง ivar ชื่อ _name, _age, _email ให้อัตโนมัติ
// และสร้าง getter/setter ให้ด้วย

@end
```

เมื่อใช้ auto synthesis ตัว compiler จะสร้าง instance variable ในรูป `_propertyName` และสร้าง getter/setter methods ให้อัตโนมัติ

### Manual Synthesis (การสังเคราะห์ด้วยตนเอง)

บางครั้งเราจำเป็นต้องควบคุมชื่อของ backing ivar หรือต้องการใช้ชื่อ ivar ที่กำหนดเอง นั่นคือเวลาที่เราใช้ `@synthesize`

```objc
// Vehicle.h
@interface Vehicle : NSObject

@property (nonatomic, strong) NSString *model;
@property (nonatomic, assign) NSInteger year;
@property (nonatomic, assign) double fuelLevel;

@end
```

```objc
// Vehicle.m
@implementation Vehicle

// กำหนดชื่อ ivar เอง
@synthesize model = _vehicleModel;
@synthesize year = _manufacturingYear;
@synthesize fuelLevel = _currentFuelLevel;

- (NSString *)description {
    return [NSString stringWithFormat:@"%@ (%ld) - Fuel: %.1f%%", 
            _vehicleModel, (long)_manufacturingYear, _currentFuelLevel];
}

@end
```

```objc
// การใช้งาน
Vehicle *car = [[Vehicle alloc] init];
car.model = @"Toyota Camry";
car.year = 2023;
car.fuelLevel = 85.5;
NSLog(@"%@", car); // Toyota Camry (2023) - Fuel: 85.5%
```

### เมื่อไหร่ควรใช้ Manual Synthesis

1. **ต้องการชื่อ ivar แตกต่างจาก `_propertyName`**
2. **Override ทั้ง getter และ setter** - ในกรณีนี้ auto synthesis จะไม่สร้าง ivar ให้ เราต้องใช้ `@synthesize` หรือประกาศ ivar เอง
3. **Inherit property** - เมื่อ subclass ต้องการ synthesize property ของ parent class

```objc
// Animal.h
@interface Animal : NSObject
@property (nonatomic, strong) NSString *sound;
@end

// Dog.h
@interface Dog : Animal
@end

// Dog.m - ต้องระบุ @synthesize ถ้าต้องการใช้ ivar ใน subclass
@implementation Dog
@synthesize sound = _dogSound;

- (void)makeSound {
    NSLog(@"Dog says: %@", _dogSound);
}
@end
```

---

## 18.2 Backing iVar Naming Convention

### หลักการตั้งชื่อ Backing iVar

การตั้งชื่อ backing ivar แบบมาตรฐานใน Objective-C คือการใช้ underscore (`_`) นำหน้าชื่อ property:

```objc
@interface BankAccount : NSObject {
    // ถ้าต้องการประกาศ ivar เองก็ทำได้
    NSString *_accountNumber;
    double _balance;
    NSMutableArray *_transactions;
}

@property (nonatomic, copy) NSString *accountNumber;
@property (nonatomic, assign) double balance;
@property (nonatomic, strong) NSMutableArray *transactions;

@end
```

```objc
@implementation BankAccount

- (instancetype)initWithAccountNumber:(NSString *)number initialBalance:(double)amount {
    self = [super init];
    if (self) {
        // เข้าถึง backing ivar โดยตรง (ใน initializer ควรทำแบบนี้)
        _accountNumber = [number copy];
        _balance = amount;
        _transactions = [[NSMutableArray alloc] init];
    }
    return self;
}

- (void)deposit:(double)amount {
    if (amount > 0) {
        _balance += amount;
        [_transactions addObject:@{@"type": @"deposit", @"amount": @(amount)}];
    }
}

- (void)withdraw:(double)amount {
    if (amount > 0 && amount <= _balance) {
        _balance -= amount;
        [_transactions addObject:@{@"type": @"withdrawal", @"amount": @(amount)}];
    }
}

@end
```

### ทำไมต้องเข้าถึง iVar โดยตรงใน Initializer และ Dealloc

```objc
@implementation Person

- (instancetype)initWithName:(NSString *)name {
    self = [super init];
    if (self) {
        // ✅ ถูกต้อง: เข้าถึง ivar โดยตรงใน initializer
        _name = [name copy];
        
        // ❌ ไม่แนะนำ: ใช้ setter ใน initializer
        // self.name = name;  // อาจเกิดปัญหากับ subclass ที่ override setter
    }
    return self;
}

- (void)dealloc {
    // ✅ ถูกต้อง: ใช้ ivar โดยตรงใน dealloc (สำหรับ MRC)
    // [_name release];
    
    // ❌ ไม่แนะนำ: ใช้ setter ใน dealloc
    // self.name = nil;  // อาจ trigger side effects ที่ไม่ต้องการ
}

@end
```

---

## 18.3 Thread-Safe Properties ด้วย Locks

### ปัญหาของ Non-Thread-Safe Properties

ค่าเริ่มต้น property attribute `nonatomic` ไม่ thread-safe ถ้ามีหลาย thread เข้าถึงพร้อมกันอาจเกิด race condition:

```objc
// ❌ ไม่ thread-safe
@interface Counter : NSObject
@property (nonatomic, assign) NSInteger count;
@end

// Thread 1 และ Thread 2 อาจอ่าน/เขียน count พร้อมกัน ทำให้ค่าผิดพลาด
```

### Atomic Properties

การใช้ `atomic` attribute (ค่าเริ่มต้น) ทำให้ getter/setter เป็น thread-safe แต่ช้ากว่า:

```objc
@interface SafeCounter : NSObject
@property (atomic, assign) NSInteger count;  // thread-safe แต่ช้า
@end
```

**ข้อควรระวัง**: `atomic` ทำให้ getter/setter แต่ละ operation เป็น atomic แต่ไม่ทำให้ compound operation (เช่น increment) เป็น atomic:

```objc
// ยังไม่ thread-safe สำหรับ compound operation
SafeCounter *counter = [[SafeCounter alloc] init];
// Thread 1: counter.count = counter.count + 1;  // read + write = not atomic!
// Thread 2: counter.count = counter.count + 1;  // race condition!
```

### Thread-Safe Properties ด้วย NSLock

```objc
// ThreadSafeDataStore.h
@interface ThreadSafeDataStore : NSObject

@property (nonatomic, strong, readonly) NSArray *items;

- (void)addItem:(id)item;
- (void)removeItem:(id)item;

@end
```

```objc
// ThreadSafeDataStore.m
@interface ThreadSafeDataStore ()
@property (nonatomic, strong) NSMutableArray *mutableItems;
@property (nonatomic, strong) NSLock *lock;
@end

@implementation ThreadSafeDataStore

- (instancetype)init {
    self = [super init];
    if (self) {
        _mutableItems = [[NSMutableArray alloc] init];
        _lock = [[NSLock alloc] init];
    }
    return self;
}

- (NSArray *)items {
    [_lock lock];
    NSArray *copy = [_mutableItems copy];
    [_lock unlock];
    return copy;
}

- (void)addItem:(id)item {
    [_lock lock];
    [_mutableItems addObject:item];
    [_lock unlock];
}

- (void)removeItem:(id)item {
    [_lock lock];
    [_mutableItems removeObject:item];
    [_lock unlock];
}

@end
```

### Thread-Safe Properties ด้วย GCD (Grand Central Dispatch)

วิธีที่นิยมกว่าคือการใช้ GCD dispatch queue:

```objc
// ThreadSafeCache.h
@interface ThreadSafeCache : NSObject

- (void)setValue:(id)value forKey:(NSString *)key;
- (id)valueForKey:(NSString *)key;
- (void)removeValueForKey:(NSString *)key;

@end
```

```objc
// ThreadSafeCache.m
@interface ThreadSafeCache ()
@property (nonatomic, strong) NSMutableDictionary *cache;
@property (nonatomic, strong) dispatch_queue_t queue;
@end

@implementation ThreadSafeCache

- (instancetype)init {
    self = [super init];
    if (self) {
        _cache = [[NSMutableDictionary alloc] init];
        // Concurrent queue สำหรับ read หลายคนพร้อมกัน, write ทีละคน
        _queue = dispatch_queue_create("com.example.ThreadSafeCache", DISPATCH_QUEUE_CONCURRENT);
    }
    return self;
}

- (void)setValue:(id)value forKey:(NSString *)key {
    // Write ใช้ barrier เพื่อให้มีเพียง thread เดียวเขียนได้
    dispatch_barrier_async(_queue, ^{
        self->_cache[key] = value;
    });
}

- (id)valueForKey:(NSString *)key {
    // Read ทำได้พร้อมกันหลาย thread
    __block id value;
    dispatch_sync(_queue, ^{
        value = self->_cache[key];
    });
    return value;
}

- (void)removeValueForKey:(NSString *)key {
    dispatch_barrier_async(_queue, ^{
        [self->_cache removeObjectForKey:key];
    });
}

@end
```

### Thread-Safe Property ด้วย @synchronized

```objc
@interface SharedResource : NSObject

@property (nonatomic, strong) NSString *data;

@end

@implementation SharedResource

- (NSString *)data {
    @synchronized(self) {
        return _data;
    }
}

- (void)setData:(NSString *)data {
    @synchronized(self) {
        if (_data != data) {
            _data = [data copy];
            // ทำ side effects ที่นี่ได้อย่างปลอดภัย
        }
    }
}

@end
```

---

## 18.4 Computed Properties (Properties with Custom Getters)

Computed properties คือ property ที่ค่าของมันคำนวณจาก property หรือ ivar อื่นๆ แทนที่จะเก็บค่าไว้โดยตรง

### ตัวอย่าง: Temperature Converter

```objc
// Temperature.h
@interface Temperature : NSObject

@property (nonatomic, assign) double celsius;

// Computed properties - คำนวณจาก celsius
@property (nonatomic, assign) double fahrenheit;  // setter แปลงกลับ
@property (nonatomic, readonly) double kelvin;    // read-only computed

@end
```

```objc
// Temperature.m
@implementation Temperature

- (double)fahrenheit {
    return (_celsius * 9.0 / 5.0) + 32.0;
}

- (void)setFahrenheit:(double)fahrenheit {
    _celsius = (fahrenheit - 32.0) * 5.0 / 9.0;
}

- (double)kelvin {
    return _celsius + 273.15;
}

@end
```

```objc
// การใช้งาน
Temperature *temp = [[Temperature alloc] init];
temp.celsius = 100.0;
NSLog(@"Celsius: %.1f", temp.celsius);      // 100.0
NSLog(@"Fahrenheit: %.1f", temp.fahrenheit); // 212.0
NSLog(@"Kelvin: %.1f", temp.kelvin);         // 373.1

temp.fahrenheit = 32.0;
NSLog(@"Celsius: %.1f", temp.celsius);  // 0.0
```

### ตัวอย่าง: FullName จาก firstName + lastName

```objc
// Person.h
@interface Person : NSObject

@property (nonatomic, copy) NSString *firstName;
@property (nonatomic, copy) NSString *lastName;
@property (nonatomic, readonly) NSString *fullName;  // computed
@property (nonatomic, readonly) NSString *initials;  // computed

@end
```

```objc
// Person.m
@implementation Person

- (NSString *)fullName {
    if (self.firstName && self.lastName) {
        return [NSString stringWithFormat:@"%@ %@", self.firstName, self.lastName];
    } else if (self.firstName) {
        return self.firstName;
    } else if (self.lastName) {
        return self.lastName;
    }
    return @"";
}

- (NSString *)initials {
    NSMutableString *initials = [NSMutableString string];
    if (self.firstName.length > 0) {
        [initials appendString:[self.firstName substringToIndex:1]];
    }
    if (self.lastName.length > 0) {
        [initials appendString:[self.lastName substringToIndex:1]];
    }
    return [initials uppercaseString];
}

@end
```

```objc
// การใช้งาน
Person *person = [[Person alloc] init];
person.firstName = @"สมชาย";
person.lastName = @"ใจดี";
NSLog(@"Full Name: %@", person.fullName);  // สมชาย ใจดี
NSLog(@"Initials: %@", person.initials);   // สใ
```

### ตัวอย่าง: Rectangle ด้วย Computed Properties

```objc
// Rectangle.h
@interface Rectangle : NSObject

@property (nonatomic, assign) double width;
@property (nonatomic, assign) double height;

// Computed properties
@property (nonatomic, readonly) double area;
@property (nonatomic, readonly) double perimeter;
@property (nonatomic, readonly) double diagonal;
@property (nonatomic, readonly) BOOL isSquare;

@end
```

```objc
// Rectangle.m
#import <math.h>

@implementation Rectangle

- (double)area {
    return _width * _height;
}

- (double)perimeter {
    return 2.0 * (_width + _height);
}

- (double)diagonal {
    return sqrt(_width * _width + _height * _height);
}

- (BOOL)isSquare {
    return fabs(_width - _height) < 0.0001;  // เปรียบเทียบ floating point
}

@end
```

---

## 18.5 Observable Properties (Properties with Side Effects in Setters)

Observable properties คือ property ที่ setter ของมันมี side effects นอกจากการเก็บค่า เช่น การ notify observers, การ validate ค่า, หรือการ update UI

### ตัวอย่าง: Property พร้อม Validation

```objc
// Product.h
@interface Product : NSObject

@property (nonatomic, copy) NSString *name;
@property (nonatomic, assign) double price;
@property (nonatomic, assign) NSInteger stockCount;

@end
```

```objc
// Product.m
@implementation Product

- (void)setPrice:(double)price {
    if (price < 0) {
        NSLog(@"Warning: Price cannot be negative, setting to 0");
        _price = 0;
    } else {
        _price = price;
    }
}

- (void)setStockCount:(NSInteger)stockCount {
    NSInteger oldCount = _stockCount;
    _stockCount = MAX(0, stockCount);
    
    // Side effect: แจ้งเตือนเมื่อ stock ต่ำ
    if (_stockCount < 10 && oldCount >= 10) {
        NSLog(@"⚠️ Low stock alert for %@: %ld items remaining", _name, (long)_stockCount);
    }
    
    // Side effect: แจ้งเตือนเมื่อหมด
    if (_stockCount == 0 && oldCount > 0) {
        NSLog(@"🚫 Out of stock: %@", _name);
    }
}

@end
```

### ตัวอย่าง: Observable Property ด้วย Delegate Pattern

```objc
// TemperatureSensor.h
@class TemperatureSensor;

@protocol TemperatureSensorDelegate <NSObject>
@optional
- (void)temperatureSensor:(TemperatureSensor *)sensor 
      didChangeTemperature:(double)newTemperature;
- (void)temperatureSensor:(TemperatureSensor *)sensor 
      didExceedThreshold:(double)threshold;
@end

@interface TemperatureSensor : NSObject

@property (nonatomic, weak) id<TemperatureSensorDelegate> delegate;
@property (nonatomic, assign) double currentTemperature;
@property (nonatomic, assign) double warningThreshold;

@end
```

```objc
// TemperatureSensor.m
@implementation TemperatureSensor

- (void)setCurrentTemperature:(double)currentTemperature {
    double oldTemperature = _currentTemperature;
    _currentTemperature = currentTemperature;
    
    // Notify delegate ถ้าค่าเปลี่ยน
    if (fabs(oldTemperature - currentTemperature) > 0.01) {
        if ([_delegate respondsToSelector:@selector(temperatureSensor:didChangeTemperature:)]) {
            [_delegate temperatureSensor:self didChangeTemperature:currentTemperature];
        }
    }
    
    // แจ้งเตือนถ้าเกิน threshold
    if (currentTemperature > _warningThreshold && oldTemperature <= _warningThreshold) {
        if ([_delegate respondsToSelector:@selector(temperatureSensor:didExceedThreshold:)]) {
            [_delegate temperatureSensor:self didExceedThreshold:_warningThreshold];
        }
    }
}

@end
```

### ตัวอย่าง: Property ที่ Update Multiple State

```objc
// LoginForm.h
@interface LoginForm : NSObject

@property (nonatomic, copy) NSString *username;
@property (nonatomic, copy) NSString *password;
@property (nonatomic, readonly) BOOL isValid;
@property (nonatomic, readonly) NSString *validationMessage;

@end
```

```objc
// LoginForm.m
@interface LoginForm ()
@property (nonatomic, assign) BOOL usernameValid;
@property (nonatomic, assign) BOOL passwordValid;
@end

@implementation LoginForm

- (void)setUsername:(NSString *)username {
    _username = [username copy];
    // Side effect: validate และ update state
    _usernameValid = username.length >= 3;
    [self updateValidationState];
}

- (void)setPassword:(NSString *)password {
    _password = [password copy];
    // Side effect: validate และ update state
    _passwordValid = password.length >= 8;
    [self updateValidationState];
}

- (void)updateValidationState {
    // คำนวณ validation message
    if (!_usernameValid && !_passwordValid) {
        _validationMessage = @"Username ต้องมีอย่างน้อย 3 ตัวอักษร และ Password ต้องมีอย่างน้อย 8 ตัวอักษร";
    } else if (!_usernameValid) {
        _validationMessage = @"Username ต้องมีอย่างน้อย 3 ตัวอักษร";
    } else if (!_passwordValid) {
        _validationMessage = @"Password ต้องมีอย่างน้อย 8 ตัวอักษร";
    } else {
        _validationMessage = @"ข้อมูลถูกต้อง";
    }
}

- (BOOL)isValid {
    return _usernameValid && _passwordValid;
}

@end
```

---

## 18.6 Copy Semantics

### ทำไม NSString และ NSArray ควรใช้ `copy`

เมื่อ property มี attribute เป็น `strong` แทน `copy` อาจทำให้เกิดปัญหาได้เมื่อกำหนดค่า mutable object เข้าไป:

```objc
// ❌ ปัญหา: ใช้ strong กับ NSString
@interface Document : NSObject
@property (nonatomic, strong) NSString *title;
@end

// การใช้งานที่มีปัญหา
NSMutableString *mutableTitle = [NSMutableString stringWithString:@"My Document"];
Document *doc = [[Document alloc] init];
doc.title = mutableTitle;

NSLog(@"Before: %@", doc.title);  // My Document
[mutableTitle appendString:@" - Draft"];
NSLog(@"After: %@", doc.title);   // My Document - Draft  ← ค่าเปลี่ยนโดยไม่ตั้งใจ!
```

```objc
// ✅ ถูกต้อง: ใช้ copy กับ NSString
@interface Document : NSObject
@property (nonatomic, copy) NSString *title;  // ← copy!
@end

// ตอนนี้ doc จะมี copy ของ string แยกต่างหาก
NSMutableString *mutableTitle = [NSMutableString stringWithString:@"My Document"];
Document *doc = [[Document alloc] init];
doc.title = mutableTitle;

NSLog(@"Before: %@", doc.title);  // My Document
[mutableTitle appendString:@" - Draft"];
NSLog(@"After: %@", doc.title);   // My Document  ← ค่าไม่เปลี่ยน!
```

### Copy กับ NSArray และ NSDictionary

```objc
@interface Configuration : NSObject

@property (nonatomic, copy) NSArray *allowedFormats;  // ✅ copy
@property (nonatomic, copy) NSDictionary *settings;    // ✅ copy

@end

@implementation Configuration

- (void)setupDefaults {
    NSMutableArray *formats = [NSMutableArray arrayWithObjects:@"jpg", @"png", @"gif", nil];
    self.allowedFormats = formats;  // สร้าง copy อัตโนมัติ
    
    formats = nil;  // อ้างอิงเดิมหายไปก็ไม่กระทบ self.allowedFormats
}

@end
```

### Implementing Copy in Custom Classes

ถ้าต้องการให้ class ของเราสามารถ copy ได้ ต้องทำตาม `NSCopying` protocol:

```objc
// Address.h
@interface Address : NSObject <NSCopying>

@property (nonatomic, copy) NSString *street;
@property (nonatomic, copy) NSString *city;
@property (nonatomic, copy) NSString *country;

@end
```

```objc
// Address.m
@implementation Address

- (id)copyWithZone:(NSZone *)zone {
    Address *copy = [[[self class] allocWithZone:zone] init];
    copy.street = self.street;
    copy.city = self.city;
    copy.country = self.country;
    return copy;
}

@end
```

### Deep Copy vs Shallow Copy

```objc
// ShoppingCart.h
@interface ShoppingCart : NSObject <NSCopying>

@property (nonatomic, copy) NSArray *items;  // shallow copy: copy แค่ array, ไม่ copy items
@property (nonatomic, copy) NSString *cartId;

@end

@implementation ShoppingCart

// Shallow copy
- (id)copyWithZone:(NSZone *)zone {
    ShoppingCart *copy = [[[self class] allocWithZone:zone] init];
    copy->_items = [_items copy];    // shallow copy
    copy->_cartId = [_cartId copy];
    return copy;
}

// Deep copy (ถ้าต้องการ)
- (ShoppingCart *)deepCopy {
    ShoppingCart *copy = [[ShoppingCart alloc] init];
    copy->_cartId = [_cartId copy];
    
    // Deep copy แต่ละ item
    NSMutableArray *copiedItems = [NSMutableArray arrayWithCapacity:_items.count];
    for (id item in _items) {
        if ([item conformsToProtocol:@protocol(NSCopying)]) {
            [copiedItems addObject:[item copy]];
        } else {
            [copiedItems addObject:item];
        }
    }
    copy->_items = [copiedItems copy];
    return copy;
}

@end
```

---

## 18.7 Weak References สำหรับ Delegates

### ทำไม Delegate ควรเป็น weak

```objc
// ❌ ปัญหา: Strong reference ทำให้เกิด retain cycle
@interface NetworkManager : NSObject
@property (nonatomic, strong) id<NetworkManagerDelegate> delegate;  // ← strong!
@end

@interface ViewController : UIViewController
@property (nonatomic, strong) NetworkManager *networkManager;  // ← strong!
@end

// ViewController เก็บ NetworkManager (strong)
// NetworkManager เก็บ ViewController เป็น delegate (strong)
// = Retain cycle! ทั้งคู่ไม่ถูก deallocate
```

```objc
// ✅ ถูกต้อง: Weak reference ป้องกัน retain cycle
@interface NetworkManager : NSObject
@property (nonatomic, weak) id<NetworkManagerDelegate> delegate;  // ← weak!
@end

// ตอนนี้:
// ViewController เก็บ NetworkManager (strong)
// NetworkManager เก็บ ViewController เป็น delegate (weak)
// ไม่มี retain cycle - เมื่อ ViewController ถูก release, NetworkManager จะเห็น delegate เป็น nil
```

### ตัวอย่างการใช้ Weak Delegate

```objc
// DataLoader.h
@protocol DataLoaderDelegate <NSObject>
@required
- (void)dataLoader:(id)loader didLoadData:(NSData *)data;
@optional
- (void)dataLoader:(id)loader didFailWithError:(NSError *)error;
- (void)dataLoaderDidStartLoading:(id)loader;
@end

@interface DataLoader : NSObject

@property (nonatomic, weak) id<DataLoaderDelegate> delegate;
@property (nonatomic, assign, readonly, getter=isLoading) BOOL loading;

- (void)loadDataFromURL:(NSURL *)url;

@end
```

```objc
// DataLoader.m
@interface DataLoader ()
@property (nonatomic, assign) BOOL loading;
@end

@implementation DataLoader

- (void)loadDataFromURL:(NSURL *)url {
    _loading = YES;
    
    // ใช้ weak reference check ก่อน
    if ([_delegate respondsToSelector:@selector(dataLoaderDidStartLoading:)]) {
        [_delegate dataLoaderDidStartLoading:self];
    }
    
    // Simulate async loading
    dispatch_async(dispatch_get_global_queue(DISPATCH_QUEUE_PRIORITY_DEFAULT, 0), ^{
        // โหลดข้อมูล...
        NSData *data = [NSData dataWithContentsOfURL:url];
        
        dispatch_async(dispatch_get_main_queue(), ^{
            self->_loading = NO;
            
            // ตรวจสอบว่า delegate ยังมีอยู่ (weak ref จะเป็น nil ถ้า deallocated)
            if (data) {
                [self->_delegate dataLoader:self didLoadData:data];
            } else {
                if ([self->_delegate respondsToSelector:@selector(dataLoader:didFailWithError:)]) {
                    NSError *error = [NSError errorWithDomain:@"DataLoaderError" 
                                                         code:404 
                                                     userInfo:nil];
                    [self->_delegate dataLoader:self didFailWithError:error];
                }
            }
        });
    });
}

@end
```

---

## 18.8 Strong Reference Cycles กับ Properties

### เข้าใจ Retain Cycles

Retain cycle เกิดเมื่อวัตถุสองชิ้นหรือมากกว่านั้นต่างก็มี strong reference ถึงกัน ทำให้ทั้งคู่ไม่ถูก deallocate:

```objc
// ❌ Retain cycle ระหว่างสอง object
@interface Node : NSObject
@property (nonatomic, strong) NSString *value;
@property (nonatomic, strong) Node *next;      // ← forward strong
@property (nonatomic, strong) Node *previous;  // ← backward strong = cycle!
@end

Node *node1 = [[Node alloc] init];
Node *node2 = [[Node alloc] init];
node1.next = node2;
node2.previous = node1;  // retain cycle!
// เมื่อ node1 และ node2 ออกจาก scope - ทั้งคู่ยังคงอยู่ใน memory!
```

```objc
// ✅ ถูกต้อง: ใช้ weak สำหรับ backward reference
@interface Node : NSObject
@property (nonatomic, strong) NSString *value;
@property (nonatomic, strong) Node *next;      // ← forward: strong (เป็นเจ้าของ)
@property (nonatomic, weak) Node *previous;    // ← backward: weak (ไม่เป็นเจ้าของ)
@end
```

### Parent-Child Relationship

```objc
// TreeNode.h
@interface TreeNode : NSObject

@property (nonatomic, strong) NSString *data;
@property (nonatomic, strong) NSMutableArray<TreeNode *> *children;  // Parent owns children
@property (nonatomic, weak) TreeNode *parent;  // ← weak: ป้องกัน cycle

@end
```

```objc
// TreeNode.m
@implementation TreeNode

- (instancetype)initWithData:(NSString *)data {
    self = [super init];
    if (self) {
        _data = data;
        _children = [[NSMutableArray alloc] init];
    }
    return self;
}

- (void)addChild:(TreeNode *)child {
    child.parent = self;  // weak reference
    [_children addObject:child];  // strong reference
}

- (void)removeFromParent {
    [_parent.children removeObject:self];
    _parent = nil;
}

@end
```

---

## 18.9 Properties with Block Types

Blocks เป็น first-class citizen ใน Objective-C และสามารถเก็บเป็น property ได้:

### การประกาศ Block Property

```objc
// Button.h
// ประกาศ block type เพื่อความอ่านง่าย
typedef void (^ButtonTapHandler)(void);
typedef void (^ButtonLongPressHandler)(NSTimeInterval duration);
typedef NSString *(^ButtonTitleProvider)(void);

@interface Button : NSObject

@property (nonatomic, copy) ButtonTapHandler onTap;          // ← copy สำหรับ block
@property (nonatomic, copy) ButtonLongPressHandler onLongPress;
@property (nonatomic, copy) ButtonTitleProvider titleProvider;
@property (nonatomic, copy) NSString *title;

- (void)simulateTap;
- (void)simulateLongPress:(NSTimeInterval)duration;

@end
```

**สำคัญ**: Block property ควรใช้ `copy` เสมอ เพราะ block ที่สร้างขึ้นเริ่มอยู่บน stack และ `copy` จะย้ายมาอยู่บน heap

```objc
// Button.m
@implementation Button

- (void)simulateTap {
    if (_onTap) {
        _onTap();
    }
}

- (void)simulateLongPress:(NSTimeInterval)duration {
    if (_onLongPress) {
        _onLongPress(duration);
    }
}

- (NSString *)title {
    if (_titleProvider) {
        return _titleProvider();  // dynamic title
    }
    return _title;
}

@end
```

```objc
// การใช้งาน
Button *button = [[Button alloc] init];

// กำหนด tap handler
button.onTap = ^{
    NSLog(@"Button was tapped!");
};

// กำหนด long press handler
button.onLongPress = ^(NSTimeInterval duration) {
    NSLog(@"Long pressed for %.1f seconds", duration);
};

// Dynamic title
__weak NSString *currentUserName = @"สมชาย";
button.titleProvider = ^NSString *{
    return [NSString stringWithFormat:@"Hello, %@", currentUserName];
};

[button simulateTap];         // Button was tapped!
[button simulateLongPress:2.5]; // Long pressed for 2.5 seconds
NSLog(@"%@", button.title);   // Hello, สมชาย
```

### ระวัง Retain Cycle กับ Block Properties

```objc
// ❌ Retain cycle
@interface ViewController : NSObject
@property (nonatomic, copy) void (^completionHandler)(BOOL success);
@property (nonatomic, strong) NSString *title;
@end

ViewController *vc = [[ViewController alloc] init];
vc.completionHandler = ^(BOOL success) {
    // Block capture vc (strong reference)
    NSLog(@"Title: %@", vc.title);  // ← vc retain block, block retain vc = cycle!
};
```

```objc
// ✅ ถูกต้อง: ใช้ weak-strong dance
ViewController *vc = [[ViewController alloc] init];
__weak ViewController *weakVC = vc;  // weak reference

vc.completionHandler = ^(BOOL success) {
    __strong ViewController *strongVC = weakVC;  // strong ชั่วคราวใน block
    if (strongVC) {
        NSLog(@"Title: %@", strongVC.title);
    }
};
```

---

## 18.10 Struct Properties

Objective-C สามารถใช้ C struct เป็น property ได้ แต่มีข้อควรระวัง:

```objc
// Geometric structs
typedef struct {
    double x;
    double y;
} Point2D;

typedef struct {
    double width;
    double height;
} Size2D;

typedef struct {
    Point2D origin;
    Size2D size;
} Rect2D;

// Shape.h
@interface Shape : NSObject

@property (nonatomic, assign) Point2D position;
@property (nonatomic, assign) Size2D size;
@property (nonatomic, assign) Rect2D bounds;

@end
```

```objc
// Shape.m
@implementation Shape

- (Rect2D)bounds {
    return (Rect2D){.origin = _position, .size = _size};
}

- (void)setBounds:(Rect2D)bounds {
    _position = bounds.origin;
    _size = bounds.size;
}

@end
```

```objc
// การใช้งาน - struct properties ต้องระวังเมื่อ modify member
Shape *shape = [[Shape alloc] init];

// ✅ ถูกต้อง: set ทั้ง struct
Point2D pos = {100.0, 200.0};
shape.position = pos;

// ❌ ไม่ compile: ไม่สามารถ modify member โดยตรงผ่าน property
// shape.position.x = 150.0;  // Error!

// ✅ ถูกต้อง: ต้อง read-modify-write
Point2D newPos = shape.position;
newPos.x = 150.0;
shape.position = newPos;
```

### ใช้ Helper Methods สำหรับ Struct Modification

```objc
// Shape.m เพิ่ม helper methods
@implementation Shape

- (void)moveToX:(double)x y:(double)y {
    _position.x = x;
    _position.y = y;
}

- (void)resizeToWidth:(double)width height:(double)height {
    _size.width = width;
    _size.height = height;
}

- (double)positionX {
    return _position.x;
}

- (double)positionY {
    return _position.y;
}

@end
```

---

## 18.11 Class Properties (+property)

ตั้งแต่ Xcode 8 / Objective-C เป็นต้นมา รองรับ class properties ด้วย `+` prefix:

### Class Properties สำหรับ Shared State

```objc
// AppConfiguration.h
@interface AppConfiguration : NSObject

// Class properties
@property (class, nonatomic, strong) NSString *appVersion;
@property (class, nonatomic, assign) BOOL debugModeEnabled;
@property (class, nonatomic, strong, readonly) AppConfiguration *sharedConfiguration;

// Instance properties
@property (nonatomic, copy) NSString *apiEndpoint;
@property (nonatomic, assign) NSTimeInterval requestTimeout;

@end
```

```objc
// AppConfiguration.m
@implementation AppConfiguration

static NSString *_appVersion = nil;
static BOOL _debugModeEnabled = NO;
static AppConfiguration *_sharedConfiguration = nil;

// Class property getters
+ (NSString *)appVersion {
    return _appVersion;
}

+ (void)setAppVersion:(NSString *)version {
    _appVersion = [version copy];
}

+ (BOOL)debugModeEnabled {
    return _debugModeEnabled;
}

+ (void)setDebugModeEnabled:(BOOL)enabled {
    _debugModeEnabled = enabled;
}

+ (AppConfiguration *)sharedConfiguration {
    static dispatch_once_t onceToken;
    dispatch_once(&onceToken, ^{
        _sharedConfiguration = [[AppConfiguration alloc] init];
        _sharedConfiguration.apiEndpoint = @"https://api.example.com";
        _sharedConfiguration.requestTimeout = 30.0;
    });
    return _sharedConfiguration;
}

@end
```

```objc
// การใช้งาน
AppConfiguration.appVersion = @"2.1.0";
AppConfiguration.debugModeEnabled = YES;

NSLog(@"App Version: %@", AppConfiguration.appVersion);
NSLog(@"Debug Mode: %@", AppConfiguration.debugModeEnabled ? @"ON" : @"OFF");

AppConfiguration *config = AppConfiguration.sharedConfiguration;
NSLog(@"API: %@", config.apiEndpoint);
```

### Class Properties สำหรับ Counters/Stats

```objc
// Request.h
@interface Request : NSObject

@property (class, nonatomic, assign, readonly) NSInteger totalRequestCount;
@property (class, nonatomic, assign, readonly) NSInteger activeRequestCount;

@property (nonatomic, copy) NSString *endpoint;
@property (nonatomic, assign, readonly, getter=isActive) BOOL active;

- (void)start;
- (void)cancel;

@end
```

```objc
// Request.m
@interface Request ()
@property (nonatomic, assign) BOOL active;
@end

@implementation Request

static NSInteger _totalRequestCount = 0;
static NSInteger _activeRequestCount = 0;

+ (NSInteger)totalRequestCount { return _totalRequestCount; }
+ (NSInteger)activeRequestCount { return _activeRequestCount; }

- (void)start {
    if (!_active) {
        _active = YES;
        _totalRequestCount++;
        _activeRequestCount++;
        NSLog(@"Started request to %@. Active: %ld", _endpoint, (long)_activeRequestCount);
    }
}

- (void)cancel {
    if (_active) {
        _active = NO;
        _activeRequestCount--;
        NSLog(@"Cancelled request to %@. Active: %ld", _endpoint, (long)_activeRequestCount);
    }
}

@end
```

---

## 18.12 KVO-Compliant Properties

Key-Value Observing (KVO) ช่วยให้ object หนึ่งสังเกตการเปลี่ยนแปลงของ property ใน object อื่นได้

### KVO กับ Auto-Synthesized Properties

Properties ที่สร้างด้วย auto synthesis จะ KVO-compliant อัตโนมัติ:

```objc
// Stock.h
@interface Stock : NSObject

@property (nonatomic, copy) NSString *symbol;
@property (nonatomic, assign) double price;
@property (nonatomic, assign) NSInteger volume;

@end

// Stock.m - ไม่ต้องทำอะไรพิเศษ KVO ทำงานอัตโนมัติ
@implementation Stock
@end
```

```objc
// StockObserver.h
@interface StockObserver : NSObject
- (void)startObserving:(Stock *)stock;
- (void)stopObserving:(Stock *)stock;
@end
```

```objc
// StockObserver.m
@implementation StockObserver

- (void)startObserving:(Stock *)stock {
    [stock addObserver:self
            forKeyPath:@"price"
               options:NSKeyValueObservingOptionNew | NSKeyValueObservingOptionOld
               context:NULL];
    
    [stock addObserver:self
            forKeyPath:@"volume"
               options:NSKeyValueObservingOptionNew
               context:NULL];
}

- (void)stopObserving:(Stock *)stock {
    [stock removeObserver:self forKeyPath:@"price"];
    [stock removeObserver:self forKeyPath:@"volume"];
}

- (void)observeValueForKeyPath:(NSString *)keyPath
                      ofObject:(id)object
                        change:(NSDictionary *)change
                       context:(void *)context {
    if ([keyPath isEqualToString:@"price"]) {
        double oldPrice = [change[NSKeyValueChangeOldKey] doubleValue];
        double newPrice = [change[NSKeyValueChangeNewKey] doubleValue];
        double change_pct = ((newPrice - oldPrice) / oldPrice) * 100.0;
        NSLog(@"Price changed: $%.2f → $%.2f (%.1f%%)", oldPrice, newPrice, change_pct);
    } else if ([keyPath isEqualToString:@"volume"]) {
        NSInteger newVolume = [change[NSKeyValueChangeNewKey] integerValue];
        NSLog(@"Volume: %ld", (long)newVolume);
    }
}

@end
```

### Manual KVO Notifications สำหรับ Computed Properties

Computed properties ที่คำนวณจาก property อื่นต้องใช้ `keyPathsForValuesAffecting...` เพื่อบอก KVO:

```objc
// Employee.h
@interface Employee : NSObject

@property (nonatomic, copy) NSString *firstName;
@property (nonatomic, copy) NSString *lastName;
@property (nonatomic, assign) double baseSalary;
@property (nonatomic, assign) double bonus;

// Computed properties - ต้องแจ้ง KVO เองว่า depend on อะไร
@property (nonatomic, readonly) NSString *fullName;
@property (nonatomic, readonly) double totalCompensation;

@end
```

```objc
// Employee.m
@implementation Employee

// บอก KVO ว่า fullName depend on firstName และ lastName
+ (NSSet *)keyPathsForValuesAffectingFullName {
    return [NSSet setWithObjects:@"firstName", @"lastName", nil];
}

// บอก KVO ว่า totalCompensation depend on baseSalary และ bonus
+ (NSSet *)keyPathsForValuesAffectingTotalCompensation {
    return [NSSet setWithObjects:@"baseSalary", @"bonus", nil];
}

- (NSString *)fullName {
    return [NSString stringWithFormat:@"%@ %@", _firstName, _lastName];
}

- (double)totalCompensation {
    return _baseSalary + _bonus;
}

@end
```

### KVO ด้วย Custom Setter (Manual Notification)

```objc
// Gauge.h
@interface Gauge : NSObject

@property (nonatomic, assign) double value;       // Triggers KVO automatically
@property (nonatomic, readonly) NSString *status;  // Computed, depends on value

@end
```

```objc
// Gauge.m
@implementation Gauge

+ (NSSet *)keyPathsForValuesAffectingStatus {
    return [NSSet setWithObject:@"value"];
}

- (NSString *)status {
    if (_value < 25.0) return @"Critical";
    if (_value < 50.0) return @"Warning";
    if (_value < 75.0) return @"Normal";
    return @"Excellent";
}

@end
```

---

## 18.13 Property Attributes สรุป

| Attribute | ความหมาย | ใช้กับ |
|-----------|----------|--------|
| `strong` | เป็นเจ้าของ object | Objects ทั่วไป |
| `weak` | ไม่เป็นเจ้าของ, auto-nil | Delegates, parent refs |
| `copy` | สร้าง copy เมื่อ assign | NSString, NSArray, Blocks |
| `assign` | ค่า pointer/primitive โดยตรง | Primitives (int, double) |
| `atomic` | Thread-safe getter/setter | ค่าเริ่มต้น, ช้ากว่า |
| `nonatomic` | ไม่ thread-safe, เร็วกว่า | ส่วนใหญ่ใช้อันนี้ |
| `readonly` | มีแค่ getter | Computed, protected state |
| `readwrite` | มีทั้ง getter และ setter | ค่าเริ่มต้น |
| `class` | Class-level property | Shared state, singletons |
| `getter=` | ชื่อ getter แบบ custom | BOOL (isEnabled) |
| `setter=` | ชื่อ setter แบบ custom | ไม่ค่อยใช้ |
| `nullable` | อาจเป็น nil | Swift interop |
| `nonnull` | ต้องไม่เป็น nil | Swift interop |

---

## แบบฝึกหัด (Practice Exercises)

### ข้อ 1: Bank Account ที่ Thread-Safe

สร้าง class `ConcurrentBankAccount` ที่:
- มี property `balance` (double) ที่ thread-safe ด้วย GCD
- มี method `deposit:` และ `withdraw:` ที่ thread-safe
- มี computed property `status` ที่คืนค่า "Empty", "Low", "Normal", หรือ "Rich" ตาม balance
- เพิ่ม KVO support สำหรับ `balance` และ `status`

```objc
// เฉลย
@interface ConcurrentBankAccount : NSObject

@property (nonatomic, assign, readonly) double balance;
@property (nonatomic, readonly) NSString *status;

- (instancetype)initWithBalance:(double)initialBalance;
- (BOOL)deposit:(double)amount;
- (BOOL)withdraw:(double)amount;

@end

@interface ConcurrentBankAccount ()
@property (nonatomic, assign) double balance;  // readwrite ใน extension
@end

@implementation ConcurrentBankAccount {
    dispatch_queue_t _queue;
}

- (instancetype)initWithBalance:(double)initialBalance {
    self = [super init];
    if (self) {
        _balance = initialBalance;
        _queue = dispatch_queue_create("com.bank.account", DISPATCH_QUEUE_CONCURRENT);
    }
    return self;
}

+ (NSSet *)keyPathsForValuesAffectingStatus {
    return [NSSet setWithObject:@"balance"];
}

- (NSString *)status {
    if (_balance == 0) return @"Empty";
    if (_balance < 1000) return @"Low";
    if (_balance < 100000) return @"Normal";
    return @"Rich";
}

- (BOOL)deposit:(double)amount {
    if (amount <= 0) return NO;
    dispatch_barrier_sync(_queue, ^{
        [self willChangeValueForKey:@"balance"];
        self->_balance += amount;
        [self didChangeValueForKey:@"balance"];
    });
    return YES;
}

- (BOOL)withdraw:(double)amount {
    __block BOOL success = NO;
    dispatch_barrier_sync(_queue, ^{
        if (amount > 0 && amount <= self->_balance) {
            [self willChangeValueForKey:@"balance"];
            self->_balance -= amount;
            [self didChangeValueForKey:@"balance"];
            success = YES;
        }
    });
    return success;
}

@end
```

### ข้อ 2: Color ด้วย Computed Properties

สร้าง class `Color` ที่:
- เก็บค่า RGB (0.0 - 1.0)
- มี computed property `hex` (NSString) เช่น `#FF5733`
- มี computed property `brightness` (double) คำนวณจาก RGB
- มี class property `redColor`, `greenColor`, `blueColor`, `whiteColor`, `blackColor`

```objc
// เฉลย
@interface Color : NSObject

@property (nonatomic, assign) double red;
@property (nonatomic, assign) double green;
@property (nonatomic, assign) double blue;
@property (nonatomic, assign) double alpha;

@property (nonatomic, readonly) NSString *hexString;
@property (nonatomic, readonly) double brightness;
@property (nonatomic, readonly) BOOL isDark;

@property (class, nonatomic, readonly) Color *redColor;
@property (class, nonatomic, readonly) Color *greenColor;
@property (class, nonatomic, readonly) Color *blueColor;
@property (class, nonatomic, readonly) Color *whiteColor;
@property (class, nonatomic, readonly) Color *blackColor;

+ (instancetype)colorWithRed:(double)r green:(double)g blue:(double)b;

@end

@implementation Color

+ (instancetype)colorWithRed:(double)r green:(double)g blue:(double)b {
    Color *c = [[Color alloc] init];
    c.red = r; c.green = g; c.blue = b; c.alpha = 1.0;
    return c;
}

+ (Color *)redColor   { return [Color colorWithRed:1 green:0 blue:0]; }
+ (Color *)greenColor { return [Color colorWithRed:0 green:1 blue:0]; }
+ (Color *)blueColor  { return [Color colorWithRed:0 green:0 blue:1]; }
+ (Color *)whiteColor { return [Color colorWithRed:1 green:1 blue:1]; }
+ (Color *)blackColor { return [Color colorWithRed:0 green:0 blue:0]; }

- (NSString *)hexString {
    int r = (int)(_red * 255);
    int g = (int)(_green * 255);
    int b = (int)(_blue * 255);
    return [NSString stringWithFormat:@"#%02X%02X%02X", r, g, b];
}

- (double)brightness {
    // Perceptual brightness formula
    return 0.299 * _red + 0.587 * _green + 0.114 * _blue;
}

- (BOOL)isDark {
    return self.brightness < 0.5;
}

@end
```

### ข้อ 3: Observable Timer

สร้าง class `CountdownTimer` ที่:
- มี property `totalSeconds` (NSInteger)
- มี property `remainingSeconds` (NSInteger, readonly)
- มี computed property `progress` (double, 0.0-1.0)
- มี property `onTick` (block ที่รับ remaining seconds)
- มี property `onComplete` (block ที่ไม่มี argument)
- implement `start`, `pause`, `reset` methods

```objc
// เฉลย
typedef void (^TickHandler)(NSInteger remainingSeconds);
typedef void (^CompleteHandler)(void);

@interface CountdownTimer : NSObject

@property (nonatomic, assign) NSInteger totalSeconds;
@property (nonatomic, assign, readonly) NSInteger remainingSeconds;
@property (nonatomic, readonly) double progress;
@property (nonatomic, copy) TickHandler onTick;
@property (nonatomic, copy) CompleteHandler onComplete;

- (void)start;
- (void)pause;
- (void)reset;

@end

@interface CountdownTimer ()
@property (nonatomic, assign) NSInteger remainingSeconds;
@property (nonatomic, strong) NSTimer *timer;
@end

@implementation CountdownTimer

+ (NSSet *)keyPathsForValuesAffectingProgress {
    return [NSSet setWithObjects:@"remainingSeconds", @"totalSeconds", nil];
}

- (double)progress {
    if (_totalSeconds == 0) return 1.0;
    return 1.0 - ((double)_remainingSeconds / _totalSeconds);
}

- (void)start {
    if (!_timer) {
        if (_remainingSeconds == 0) {
            _remainingSeconds = _totalSeconds;
        }
        _timer = [NSTimer scheduledTimerWithTimeInterval:1.0
                                                  target:self
                                                selector:@selector(tick)
                                                userInfo:nil
                                                 repeats:YES];
    }
}

- (void)pause {
    [_timer invalidate];
    _timer = nil;
}

- (void)reset {
    [self pause];
    _remainingSeconds = _totalSeconds;
}

- (void)tick {
    _remainingSeconds--;
    if (_onTick) _onTick(_remainingSeconds);
    if (_remainingSeconds <= 0) {
        [self pause];
        if (_onComplete) _onComplete();
    }
}

@end
```

### ข้อ 4: Type-Safe User Settings

สร้าง class `UserSettings` ที่:
- มี class property `shared` (singleton)
- มี property ต่างๆ เช่น `username`, `fontSize`, `darkModeEnabled`, `language`
- ทุก property ต้อง persist ลง NSUserDefaults อัตโนมัติผ่าน setter
- มี method `resetToDefaults`

```objc
// เฉลยบางส่วน
@interface UserSettings : NSObject

@property (class, nonatomic, strong, readonly) UserSettings *shared;

@property (nonatomic, copy) NSString *username;
@property (nonatomic, assign) double fontSize;
@property (nonatomic, assign) BOOL darkModeEnabled;
@property (nonatomic, copy) NSString *language;

- (void)resetToDefaults;

@end

@implementation UserSettings

static UserSettings *_shared;

+ (UserSettings *)shared {
    static dispatch_once_t onceToken;
    dispatch_once(&onceToken, ^{
        _shared = [[UserSettings alloc] init];
    });
    return _shared;
}

- (NSString *)username {
    return [[NSUserDefaults standardUserDefaults] stringForKey:@"username"] ?: @"Guest";
}

- (void)setUsername:(NSString *)username {
    [[NSUserDefaults standardUserDefaults] setObject:username forKey:@"username"];
}

- (double)fontSize {
    double size = [[NSUserDefaults standardUserDefaults] doubleForKey:@"fontSize"];
    return size > 0 ? size : 16.0;
}

- (void)setFontSize:(double)fontSize {
    [[NSUserDefaults standardUserDefaults] setDouble:fontSize forKey:@"fontSize"];
}

- (BOOL)darkModeEnabled {
    return [[NSUserDefaults standardUserDefaults] boolForKey:@"darkModeEnabled"];
}

- (void)setDarkModeEnabled:(BOOL)enabled {
    [[NSUserDefaults standardUserDefaults] setBool:enabled forKey:@"darkModeEnabled"];
}

- (void)resetToDefaults {
    self.username = @"Guest";
    self.fontSize = 16.0;
    self.darkModeEnabled = NO;
    self.language = @"th";
}

@end
```

### ข้อ 5: Observable Array Wrapper

สร้าง class `ObservableArray` ที่:
- ห่อหุ้ม NSMutableArray
- มี property `count` (readonly)
- มี property `onChange` (block ที่รับ array และ operation type)
- เมื่อเพิ่ม/ลบ element จะ trigger block

```objc
// เฉลย
typedef NS_ENUM(NSInteger, ArrayChangeType) {
    ArrayChangeTypeAdd,
    ArrayChangeTypeRemove,
    ArrayChangeTypeClear,
};

typedef void (^ArrayChangeHandler)(NSArray *array, ArrayChangeType type, NSInteger index);

@interface ObservableArray : NSObject

@property (nonatomic, readonly) NSInteger count;
@property (nonatomic, readonly) NSArray *allObjects;
@property (nonatomic, copy) ArrayChangeHandler onChange;

- (void)addObject:(id)object;
- (void)removeObjectAtIndex:(NSInteger)index;
- (id)objectAtIndex:(NSInteger)index;
- (void)clear;

@end

@implementation ObservableArray {
    NSMutableArray *_items;
}

- (instancetype)init {
    self = [super init];
    if (self) {
        _items = [[NSMutableArray alloc] init];
    }
    return self;
}

- (NSInteger)count { return _items.count; }
- (NSArray *)allObjects { return [_items copy]; }

- (void)addObject:(id)object {
    [_items addObject:object];
    if (_onChange) {
        _onChange([_items copy], ArrayChangeTypeAdd, _items.count - 1);
    }
}

- (void)removeObjectAtIndex:(NSInteger)index {
    if (index >= 0 && index < _items.count) {
        [_items removeObjectAtIndex:index];
        if (_onChange) {
            _onChange([_items copy], ArrayChangeTypeRemove, index);
        }
    }
}

- (id)objectAtIndex:(NSInteger)index {
    return [_items objectAtIndex:index];
}

- (void)clear {
    [_items removeAllObjects];
    if (_onChange) {
        _onChange([_items copy], ArrayChangeTypeClear, -1);
    }
}

@end
```

### ข้อ 6 - 10: แบบฝึกหัดเพิ่มเติม

**ข้อ 6**: สร้าง `GradeCalculator` class ที่มี property `scores` (copy NSArray), `average` (computed), `grade` (computed NSString: A/B/C/D/F), `isPassing` (computed BOOL)

**ข้อ 7**: สร้าง `LinkedList` class ด้วย Node class ที่มี `next` (strong), `previous` (weak) เพื่อหลีกเลี่ยง retain cycle

**ข้อ 8**: สร้าง `EventEmitter` class ที่มี class property `shared`, รองรับการ subscribe/unsubscribe events, และ emit events ด้วย block callbacks

**ข้อ 9**: สร้าง `ThrottledProperty` class ที่ใช้ block property เพื่อ debounce/throttle การเรียก handler (ไม่ให้เรียกถี่เกินไป)

**ข้อ 10**: สร้าง KVO-compliant `ProgressTracker` ที่มี property `currentStep`, `totalSteps`, computed `percentage`, computed `statusMessage`, และ supports multiple observers

---

## สรุป

ในบทนี้เราได้เรียนรู้ Properties ขั้นสูงใน Objective-C ครอบคลุม:

1. **Auto vs Manual Synthesis** - ความแตกต่างและเวลาที่ควรใช้แต่ละแบบ
2. **Backing iVar Naming** - Convention ของ `_propertyName` และเหตุผลในการเข้าถึง ivar โดยตรงใน initializer
3. **Thread-Safe Properties** - การใช้ NSLock, GCD, และ `@synchronized`
4. **Computed Properties** - Custom getters ที่คำนวณค่าจาก state อื่น
5. **Observable Properties** - Setters ที่มี side effects สำหรับ notification และ validation
6. **Copy Semantics** - ทำไม NSString/NSArray/Block ควรใช้ `copy`
7. **Weak Delegates** - ป้องกัน retain cycles ด้วย weak references
8. **Strong Reference Cycles** - เข้าใจและป้องกัน retain cycles
9. **Block Properties** - การเก็บ blocks เป็น properties และ memory management
10. **Struct Properties** - ข้อจำกัดและวิธีใช้งาน
11. **Class Properties** - Shared state และ singleton patterns
12. **KVO-Compliant Properties** - การทำให้ computed properties รองรับ KVO

ตอนต่อไปเราจะเรียนรู้เรื่อง **Manual Reference Counting (MRC)** ซึ่งเป็นระบบจัดการ memory แบบดั้งเดิมของ Objective-C ก่อนที่ ARC จะเข้ามา
