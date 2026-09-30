# Part 11: Classes and Objects ใน Objective-C (Deep Dive)

## บทนำ

Objective-C เป็นภาษา Object-Oriented Programming (OOP) ที่สร้างบน C โดยเพิ่ม message-passing และ object model เข้ามา ใน Part นี้เราจะเจาะลึกเรื่อง Classes และ Objects ซึ่งเป็นหัวใจสำคัญของการเขียนโปรแกรมด้วย Objective-C

---

## 1. Class Definition: @interface และ @implementation

### โครงสร้างพื้นฐานของ Class

ใน Objective-C การกำหนด class แบ่งออกเป็น 2 ส่วน:

1. **@interface** - ประกาศ (Declaration) ว่า class มีอะไรบ้าง
2. **@implementation** - นิยาม (Definition) ว่าแต่ละ method ทำงานอย่างไร

```objc
// ไฟล์ Person.h - Header file (Declaration)
#import <Foundation/Foundation.h>

@interface Person : NSObject {
    // Instance Variables (ivars) ที่ประกาศใน @interface จะเป็น @protected โดยค่าเริ่มต้น
    NSString *_name;
    NSInteger _age;
}

// Properties
@property (nonatomic, strong) NSString *name;
@property (nonatomic, assign) NSInteger age;

// Method declarations
- (instancetype)initWithName:(NSString *)name age:(NSInteger)age;
- (void)introduce;
- (NSString *)fullDescription;

@end
```

```objc
// ไฟล์ Person.m - Implementation file
#import "Person.h"

@implementation Person

// @synthesize สร้าง getter/setter อัตโนมัติ (ใน modern Objective-C ไม่จำเป็นต้องเขียน)
@synthesize name = _name;
@synthesize age = _age;

- (instancetype)initWithName:(NSString *)name age:(NSInteger)age {
    self = [super init];
    if (self) {
        _name = [name copy];
        _age = age;
    }
    return self;
}

- (void)introduce {
    NSLog(@"สวัสดี ฉันชื่อ %@ อายุ %ld ปี", _name, (long)_age);
}

- (NSString *)fullDescription {
    return [NSString stringWithFormat:@"Person: %@ (อายุ %ld)", _name, (long)_age];
}

@end
```

```objc
// main.m - การใช้งาน
int main(int argc, const char * argv[]) {
    @autoreleasepool {
        Person *person = [[Person alloc] initWithName:@"สมชาย" age:25];
        [person introduce];
        NSLog(@"%@", [person fullDescription]);
    }
    return 0;
}
```

**Output:**
```
สวัสดี ฉันชื่อ สมชาย อายุ 25 ปี
Person: สมชาย (อายุ 25)
```

---

## 2. Instance Variables (ivars)

### Access Modifiers สำหรับ ivars

Objective-C มี 3 ระดับการเข้าถึง ivar:

| Modifier | ความหมาย |
|----------|---------|
| `@private` | เข้าถึงได้เฉพาะใน class นั้นเท่านั้น |
| `@protected` | เข้าถึงได้ใน class และ subclass (ค่าเริ่มต้น) |
| `@public` | เข้าถึงได้จากทุกที่ (ไม่แนะนำ) |

```objc
// IvarDemo.h
#import <Foundation/Foundation.h>

@interface IvarDemo : NSObject {
@private
    NSString *_secretCode;      // เข้าถึงได้เฉพาะใน IvarDemo เท่านั้น
    NSInteger _privateCounter;

@protected
    NSString *_displayName;     // subclass สามารถเข้าถึงได้
    CGFloat _score;

@public
    NSString *publicInfo;       // ทุกคนเข้าถึงได้ (ไม่แนะนำ)
}

- (void)demonstrateAccess;

@end
```

```objc
// IvarDemo.m
#import "IvarDemo.h"

@implementation IvarDemo

- (instancetype)init {
    self = [super init];
    if (self) {
        _secretCode = @"XK-9401";     // @private - OK ใน class เดียวกัน
        _privateCounter = 0;
        _displayName = @"Demo Object"; // @protected - OK ใน class เดียวกัน
        _score = 95.5;
        self->publicInfo = @"ข้อมูลสาธารณะ"; // @public
    }
    return self;
}

- (void)demonstrateAccess {
    // ภายใน class สามารถเข้าถึง ivar ทุกระดับได้
    NSLog(@"Secret: %@", _secretCode);
    NSLog(@"Name: %@", _displayName);
    NSLog(@"Score: %.1f", _score);
    NSLog(@"Public: %@", self->publicInfo);
}

@end
```

```objc
// ChildClass.h - Subclass ที่สืบทอดจาก IvarDemo
@interface ChildClass : IvarDemo

- (void)accessParentIvars;

@end
```

```objc
// ChildClass.m
#import "ChildClass.h"

@implementation ChildClass

- (void)accessParentIvars {
    // สามารถเข้าถึง @protected ivar ของ parent ได้
    _displayName = @"ชื่อที่แก้ไขโดย Child";  // OK!
    _score = 88.0;                              // OK!

    // ไม่สามารถเข้าถึง @private ได้
    // _secretCode = @"NEW";  // ERROR! Cannot access @private ivar

    NSLog(@"displayName from child: %@", _displayName);
}

@end
```

```objc
// การใช้งาน @public ivar (ไม่แนะนำ)
IvarDemo *demo = [[IvarDemo alloc] init];
// เข้าถึง @public ivar โดยตรง (anti-pattern)
demo->publicInfo = @"แก้ไขจากภายนอก";
NSLog(@"%@", demo->publicInfo);
```

> **หมายเหตุ:** ในการพัฒนาจริง ควรใช้ Properties แทน ivars โดยตรง เพราะ properties ให้ encapsulation ที่ดีกว่า

---

## 3. Properties (@property)

Properties เป็นกลไกสมัยใหม่ของ Objective-C สำหรับการเข้าถึง instance variables

### Property Attributes

#### 3.1 Memory Management Attributes

```objc
@interface PropertyDemo : NSObject

// strong - ถือ strong reference, object จะไม่ถูก deallocate ตราบที่มี strong ref
@property (nonatomic, strong) NSString *title;
@property (nonatomic, strong) NSMutableArray *items;

// weak - ถือ weak reference, จะเป็น nil เมื่อ object ถูก deallocate
@property (nonatomic, weak) id delegate;
@property (nonatomic, weak) NSObject *parent;

// copy - คัดลอก object แทนที่จะถือ reference (ดีสำหรับ NSString, NSArray)
@property (nonatomic, copy) NSString *name;
@property (nonatomic, copy) NSArray *data;

// assign - ใช้สำหรับ primitive types (int, float, BOOL, etc.)
@property (nonatomic, assign) NSInteger count;
@property (nonatomic, assign) BOOL isActive;
@property (nonatomic, assign) CGFloat price;

// unsafe_unretained - เหมือน weak แต่ไม่ zeroed (อันตราย, ไม่แนะนำ)
@property (nonatomic, unsafe_unretained) id unsafeRef;

@end
```

#### 3.2 Atomicity Attributes

```objc
@interface AtomicityDemo : NSObject

// nonatomic - เร็วกว่า, ไม่ thread-safe (ใช้ส่วนใหญ่ใน iOS/macOS)
@property (nonatomic, strong) NSString *fastString;

// atomic - ช้ากว่า, thread-safe สำหรับการ get/set (ค่าเริ่มต้น)
@property (atomic, strong) NSString *threadSafeString;

@end
```

#### 3.3 Access Attributes

```objc
@interface AccessDemo : NSObject

// readwrite - มีทั้ง getter และ setter (ค่าเริ่มต้น)
@property (nonatomic, strong, readwrite) NSString *editableName;

// readonly - มีแต่ getter เท่านั้น
@property (nonatomic, strong, readonly) NSString *fixedID;
@property (nonatomic, assign, readonly) NSInteger creationTimestamp;

@end
```

### ตัวอย่างการใช้ Property แบบครบถ้วน

```objc
// Product.h
#import <Foundation/Foundation.h>

@interface Product : NSObject

@property (nonatomic, copy)   NSString   *name;
@property (nonatomic, copy)   NSString   *productID;
@property (nonatomic, assign) double      price;
@property (nonatomic, assign) NSInteger   quantity;
@property (nonatomic, strong) NSDate     *createdDate;
@property (nonatomic, weak)   id          category;
@property (nonatomic, assign, readonly) double totalValue;
@property (nonatomic, copy,   readonly) NSString *formattedPrice;

- (instancetype)initWithName:(NSString *)name
                   productID:(NSString *)productID
                       price:(double)price;

@end
```

```objc
// Product.m
#import "Product.h"

@implementation Product

- (instancetype)initWithName:(NSString *)name
                   productID:(NSString *)productID
                       price:(double)price {
    self = [super init];
    if (self) {
        _name = [name copy];
        _productID = [productID copy];
        _price = price;
        _quantity = 0;
        _createdDate = [NSDate date];
    }
    return self;
}

// Computed readonly property - totalValue
- (double)totalValue {
    return _price * _quantity;
}

// Computed readonly property - formattedPrice
- (NSString *)formattedPrice {
    return [NSString stringWithFormat:@"฿%.2f", _price];
}

- (NSString *)description {
    return [NSString stringWithFormat:
            @"Product{name=%@, id=%@, price=%.2f, qty=%ld, total=%.2f}",
            _name, _productID, _price, (long)_quantity, self.totalValue];
}

@end
```

```objc
// การใช้งาน Product
Product *laptop = [[Product alloc] initWithName:@"MacBook Pro"
                                      productID:@"MBP-001"
                                          price:59900.0];
laptop.quantity = 5;

NSLog(@"ชื่อสินค้า: %@", laptop.name);
NSLog(@"ราคา: %@", laptop.formattedPrice);
NSLog(@"จำนวน: %ld", (long)laptop.quantity);
NSLog(@"มูลค่ารวม: %.2f", laptop.totalValue);
NSLog(@"%@", laptop);
```

**Output:**
```
ชื่อสินค้า: MacBook Pro
ราคา: ฿59900.00
จำนวน: 5
มูลค่ารวม: 299500.00
Product{name=MacBook Pro, id=MBP-001, price=59900.00, qty=5, total=299500.00}
```

---

## 4. @synthesize และ @dynamic

### @synthesize

`@synthesize` บอก compiler ให้สร้าง getter/setter อัตโนมัติ

```objc
// ใน modern Objective-C (Xcode 4.4+) ไม่จำเป็นต้องเขียน @synthesize
// แต่สามารถใช้เพื่อกำหนด backing ivar ที่ต้องการได้

@implementation MyClass

// สร้าง getter/setter สำหรับ name โดยใช้ _name เป็น backing ivar (ค่าเริ่มต้น)
@synthesize name = _name;

// สร้าง getter/setter สำหรับ age โดยใช้ myAge เป็น backing ivar (กำหนดเอง)
@synthesize age = myAge;

// สร้าง getter/setter สำหรับ score โดยใช้ score (ไม่มี underscore) เป็น backing ivar
@synthesize score;

@end
```

### @dynamic

`@dynamic` บอก compiler ว่า getter/setter จะถูกจัดเตรียมที่ runtime (ไม่ให้สร้างอัตโนมัติ)

```objc
// DynamicProperty.h
@interface DynamicProperty : NSObject

@property (nonatomic, strong) NSString *dynamicName;
@property (nonatomic, assign) NSInteger dynamicValue;

@end
```

```objc
// DynamicProperty.m
#import "DynamicProperty.h"
#import <objc/runtime.h>

@implementation DynamicProperty

// บอก compiler ว่าจะสร้าง accessor เอง
@dynamic dynamicName;
@dynamic dynamicValue;

// Custom getter โดยใช้ Associated Objects
- (NSString *)dynamicName {
    return objc_getAssociatedObject(self, @selector(dynamicName));
}

- (void)setDynamicName:(NSString *)dynamicName {
    objc_setAssociatedObject(self,
                             @selector(dynamicName),
                             dynamicName,
                             OBJC_ASSOCIATION_COPY_NONATOMIC);
}

- (NSInteger)dynamicValue {
    NSNumber *value = objc_getAssociatedObject(self, @selector(dynamicValue));
    return [value integerValue];
}

- (void)setDynamicValue:(NSInteger)dynamicValue {
    objc_setAssociatedObject(self,
                             @selector(dynamicValue),
                             @(dynamicValue),
                             OBJC_ASSOCIATION_RETAIN_NONATOMIC);
}

@end
```

---

## 5. Instance Methods (-) และ Class Methods (+)

### Instance Methods (-)

Instance methods เรียกผ่าน instance (object) ของ class

```objc
// MethodsDemo.h
@interface Calculator : NSObject

@property (nonatomic, assign) double result;

// Instance methods (-)
- (double)add:(double)value;
- (double)subtract:(double)value;
- (double)multiply:(double)value;
- (double)divide:(double)value;
- (void)reset;
- (NSString *)formattedResult;

@end
```

```objc
// MethodsDemo.m
@implementation Calculator

- (instancetype)init {
    self = [super init];
    if (self) {
        _result = 0.0;
    }
    return self;
}

- (double)add:(double)value {
    _result += value;
    return _result;
}

- (double)subtract:(double)value {
    _result -= value;
    return _result;
}

- (double)multiply:(double)value {
    _result *= value;
    return _result;
}

- (double)divide:(double)value {
    if (value == 0) {
        NSLog(@"ข้อผิดพลาด: ไม่สามารถหารด้วยศูนย์ได้");
        return _result;
    }
    _result /= value;
    return _result;
}

- (void)reset {
    _result = 0.0;
}

- (NSString *)formattedResult {
    return [NSString stringWithFormat:@"ผลลัพธ์ = %.4f", _result];
}

@end
```

### Class Methods (+)

Class methods เรียกผ่าน class โดยตรง ไม่ต้องสร้าง instance

```objc
// MathUtils.h
@interface MathUtils : NSObject

// Class methods (+) - Factory methods และ utility methods
+ (double)circleAreaWithRadius:(double)radius;
+ (double)rectangleAreaWithWidth:(double)width height:(double)height;
+ (double)pythagoreanHypotenuseWithA:(double)a b:(double)b;
+ (NSArray *)fibonacciSequenceUpTo:(NSInteger)count;
+ (BOOL)isPrime:(NSInteger)number;

// Class method สำหรับสร้าง shared instance (Singleton pattern)
+ (instancetype)sharedInstance;

@end
```

```objc
// MathUtils.m
#import "MathUtils.h"

@implementation MathUtils

+ (double)circleAreaWithRadius:(double)radius {
    return M_PI * radius * radius;
}

+ (double)rectangleAreaWithWidth:(double)width height:(double)height {
    return width * height;
}

+ (double)pythagoreanHypotenuseWithA:(double)a b:(double)b {
    return sqrt(a * a + b * b);
}

+ (NSArray *)fibonacciSequenceUpTo:(NSInteger)count {
    NSMutableArray *sequence = [NSMutableArray array];
    if (count <= 0) return sequence;

    NSInteger a = 0, b = 1;
    for (NSInteger i = 0; i < count; i++) {
        [sequence addObject:@(a)];
        NSInteger temp = a + b;
        a = b;
        b = temp;
    }
    return [sequence copy];
}

+ (BOOL)isPrime:(NSInteger)number {
    if (number < 2) return NO;
    if (number == 2) return YES;
    if (number % 2 == 0) return NO;

    for (NSInteger i = 3; i <= (NSInteger)sqrt((double)number); i += 2) {
        if (number % i == 0) return NO;
    }
    return YES;
}

// Singleton pattern ด้วย Class method
static MathUtils *_sharedInstance = nil;

+ (instancetype)sharedInstance {
    static dispatch_once_t onceToken;
    dispatch_once(&onceToken, ^{
        _sharedInstance = [[self alloc] init];
    });
    return _sharedInstance;
}

@end
```

```objc
// การใช้งาน
int main(int argc, const char * argv[]) {
    @autoreleasepool {
        // Instance methods
        Calculator *calc = [[Calculator alloc] init];
        [calc add:100];
        [calc add:50];
        [calc subtract:25];
        NSLog(@"%@", [calc formattedResult]);  // ผลลัพธ์ = 125.0000

        // Class methods - ไม่ต้องสร้าง instance
        double area = [MathUtils circleAreaWithRadius:5.0];
        NSLog(@"พื้นที่วงกลม radius=5: %.4f", area);

        double hyp = [MathUtils pythagoreanHypotenuseWithA:3.0 b:4.0];
        NSLog(@"ด้านตรงข้ามมุมฉาก (3-4-?): %.1f", hyp);

        NSArray *fib = [MathUtils fibonacciSequenceUpTo:10];
        NSLog(@"Fibonacci 10 ตัวแรก: %@", fib);

        NSLog(@"13 เป็นจำนวนเฉพาะ: %@", [MathUtils isPrime:13] ? @"ใช่" : @"ไม่ใช่");
        NSLog(@"15 เป็นจำนวนเฉพาะ: %@", [MathUtils isPrime:15] ? @"ใช่" : @"ไม่ใช่");
    }
    return 0;
}
```

---

## 6. self และ super

### self

`self` คือ pointer ที่ชี้ไปยัง instance ปัจจุบัน (เหมือน `this` ใน Java/C++)

```objc
@interface SelfDemo : NSObject

@property (nonatomic, strong) NSString *name;
@property (nonatomic, assign) NSInteger value;

- (instancetype)initWithName:(NSString *)name value:(NSInteger)value;
- (void)updateWithName:(NSString *)name value:(NSInteger)value;
- (SelfDemo *)copyWithModifiedValue:(NSInteger)newValue;

@end

@implementation SelfDemo

- (instancetype)initWithName:(NSString *)name value:(NSInteger)value {
    self = [super init];
    if (self) {
        // self ใช้สำหรับเรียก properties/methods ของ object นี้
        self.name = name;
        self.value = value;

        // หรือเข้า ivar โดยตรง (เร็วกว่า)
        // _name = name;
        // _value = value;
    }
    return self;
}

- (void)updateWithName:(NSString *)name value:(NSInteger)value {
    // ใช้ self เพื่อเรียก setter method
    self.name = name;
    self.value = value;

    // เรียก method อื่นของ object นี้ผ่าน self
    [self logUpdate];
}

- (void)logUpdate {
    NSLog(@"อัพเดต: %@ = %ld", self.name, (long)self.value);
}

- (SelfDemo *)copyWithModifiedValue:(NSInteger)newValue {
    // สร้าง instance ใหม่ของ class เดียวกันผ่าน self class
    SelfDemo *copy = [[self class] alloc];
    copy = [copy initWithName:self.name value:newValue];
    return copy;
}

// Class method ที่ใช้ self เพื่ออ้างถึง class
+ (instancetype)defaultInstance {
    return [[self alloc] initWithName:@"Default" value:0];
}

@end
```

### super

`super` ใช้สำหรับเรียก method ของ superclass

```objc
// SuperDemo.h
@interface Animal : NSObject

@property (nonatomic, copy) NSString *name;
@property (nonatomic, assign) NSInteger age;

- (instancetype)initWithName:(NSString *)name age:(NSInteger)age;
- (void)makeSound;
- (NSString *)description;

@end

@interface Dog : Animal

@property (nonatomic, copy) NSString *breed;

- (instancetype)initWithName:(NSString *)name age:(NSInteger)age breed:(NSString *)breed;
- (void)fetch;

@end
```

```objc
// SuperDemo.m
@implementation Animal

- (instancetype)initWithName:(NSString *)name age:(NSInteger)age {
    self = [super init];  // เรียก NSObject's init
    if (self) {
        _name = [name copy];
        _age = age;
    }
    return self;
}

- (void)makeSound {
    NSLog(@"%@ ส่งเสียง...", _name);
}

- (NSString *)description {
    return [NSString stringWithFormat:@"Animal{name=%@, age=%ld}", _name, (long)_age];
}

@end

@implementation Dog

- (instancetype)initWithName:(NSString *)name age:(NSInteger)age breed:(NSString *)breed {
    // เรียก designated initializer ของ superclass (Animal)
    self = [super initWithName:name age:age];
    if (self) {
        _breed = [breed copy];
    }
    return self;
}

- (void)makeSound {
    // เรียก method ของ superclass ก่อน
    [super makeSound];
    // แล้วเพิ่ม behavior ของตัวเอง
    NSLog(@"%@ เห่า: โฮ้ง โฮ้ง!", self.name);
}

- (void)fetch {
    NSLog(@"%@ (%@) วิ่งไปเก็บลูกบอล!", self.name, _breed);
}

- (NSString *)description {
    // ใช้ super description แล้วเพิ่มข้อมูล Dog
    NSString *superDesc = [super description];
    return [NSString stringWithFormat:@"Dog{%@, breed=%@}", superDesc, _breed];
}

@end
```

---

## 7. Init Methods และ Designated Initializer Pattern

### Designated Initializer

Designated initializer คือ init method หลักที่ทำ initialization ทั้งหมด

```objc
// Rectangle.h
#import <Foundation/Foundation.h>

@interface Rectangle : NSObject

@property (nonatomic, assign) CGFloat width;
@property (nonatomic, assign) CGFloat height;
@property (nonatomic, copy)   NSString *color;

// Designated initializer - เป็น init method หลัก
- (instancetype)initWithWidth:(CGFloat)width
                       height:(CGFloat)height
                        color:(NSString *)color NS_DESIGNATED_INITIALIZER;

// Convenience initializers - เรียก designated initializer ต่อ
- (instancetype)initWithWidth:(CGFloat)width height:(CGFloat)height;
- (instancetype)initWithSide:(CGFloat)side;  // สี่เหลี่ยมจัตุรัส

// Class factory methods
+ (instancetype)rectangleWithWidth:(CGFloat)width height:(CGFloat)height;
+ (instancetype)squareWithSide:(CGFloat)side;

// Computed properties
@property (nonatomic, assign, readonly) CGFloat area;
@property (nonatomic, assign, readonly) CGFloat perimeter;

@end
```

```objc
// Rectangle.m
#import "Rectangle.h"

@implementation Rectangle

// Designated initializer
- (instancetype)initWithWidth:(CGFloat)width
                       height:(CGFloat)height
                        color:(NSString *)color {
    self = [super init];
    if (self) {
        // ตั้งค่า ivar โดยตรงใน init (ไม่ผ่าน setter เพื่อ safety)
        _width = width;
        _height = height;
        _color = [color copy];
    }
    return self;
}

// Convenience initializer เรียก designated initializer
- (instancetype)initWithWidth:(CGFloat)width height:(CGFloat)height {
    return [self initWithWidth:width height:height color:@"white"];
}

// Convenience initializer เรียก designated initializer
- (instancetype)initWithSide:(CGFloat)side {
    return [self initWithWidth:side height:side color:@"white"];
}

// ต้อง override init ให้เรียก designated initializer ของ class นี้
- (instancetype)init {
    return [self initWithWidth:1.0 height:1.0 color:@"white"];
}

// Factory class methods
+ (instancetype)rectangleWithWidth:(CGFloat)width height:(CGFloat)height {
    return [[self alloc] initWithWidth:width height:height];
}

+ (instancetype)squareWithSide:(CGFloat)side {
    return [[self alloc] initWithSide:side];
}

// Computed readonly properties
- (CGFloat)area {
    return _width * _height;
}

- (CGFloat)perimeter {
    return 2 * (_width + _height);
}

- (NSString *)description {
    return [NSString stringWithFormat:
            @"Rectangle{%.1f x %.1f, color=%@, area=%.2f}",
            _width, _height, _color, self.area];
}

@end
```

```objc
// การใช้งาน
Rectangle *r1 = [[Rectangle alloc] initWithWidth:10 height:5 color:@"blue"];
Rectangle *r2 = [[Rectangle alloc] initWithWidth:8 height:3];
Rectangle *r3 = [[Rectangle alloc] initWithSide:6];
Rectangle *r4 = [Rectangle rectangleWithWidth:12 height:4];
Rectangle *r5 = [Rectangle squareWithSide:7];

NSLog(@"%@", r1);  // Rectangle{10.0 x 5.0, color=blue, area=50.00}
NSLog(@"%@", r2);  // Rectangle{8.0 x 3.0, color=white, area=24.00}
NSLog(@"พื้นที่ r3: %.1f", r3.area);   // พื้นที่ r3: 36.0
NSLog(@"เส้นรอบวง r4: %.1f", r4.perimeter);  // เส้นรอบวง r4: 32.0
```

---

## 8. dealloc Method

`dealloc` ถูกเรียกโดยอัตโนมัติเมื่อ object ถูกลบออกจากหน่วยความจำ

```objc
// ResourceManager.h
@interface ResourceManager : NSObject

@property (nonatomic, strong) NSString *name;
@property (nonatomic, strong) NSMutableArray *resources;

- (instancetype)initWithName:(NSString *)name;
- (void)addResource:(NSString *)resource;

@end
```

```objc
// ResourceManager.m
#import "ResourceManager.h"

@implementation ResourceManager

- (instancetype)initWithName:(NSString *)name {
    self = [super init];
    if (self) {
        _name = [name copy];
        _resources = [NSMutableArray array];
        NSLog(@"[%@] เริ่มต้น ResourceManager", _name);
        // สมมติว่าเปิด file หรือ network connection
    }
    return self;
}

- (void)addResource:(NSString *)resource {
    [_resources addObject:resource];
    NSLog(@"[%@] เพิ่ม resource: %@", _name, resource);
}

- (void)dealloc {
    // ทำความสะอาดทรัพยากรที่ไม่ได้อยู่ใน ARC
    NSLog(@"[%@] กำลัง deallocate...", _name);

    // ใน ARC ไม่ต้องเรียก [super dealloc] ด้วยตนเอง
    // ARC จะจัดการให้อัตโนมัติ

    // แต่ถ้าใช้ manual memory management ต้องเรียก:
    // [super dealloc];
}

@end
```

```objc
// การใช้งาน - ARC จัดการ dealloc อัตโนมัติ
@autoreleasepool {
    ResourceManager *mgr = [[ResourceManager alloc] initWithName:@"FileManager"];
    [mgr addResource:@"file1.txt"];
    [mgr addResource:@"file2.txt"];
    // เมื่อออกจาก scope นี้ mgr จะถูก deallocate
    NSLog(@"กำลังออกจาก scope...");
}
// Output: [FileManager] กำลัง deallocate...
```

---

## 9. Class Methods สำหรับ Factory Patterns

Factory pattern ใช้ class methods เพื่อสร้าง object ที่มี configuration ต่างกัน

```objc
// Color.h
@interface Color : NSObject

@property (nonatomic, assign, readonly) CGFloat red;
@property (nonatomic, assign, readonly) CGFloat green;
@property (nonatomic, assign, readonly) CGFloat blue;
@property (nonatomic, assign, readonly) CGFloat alpha;

// Designated initializer
- (instancetype)initWithRed:(CGFloat)red
                      green:(CGFloat)green
                       blue:(CGFloat)blue
                      alpha:(CGFloat)alpha;

// Factory methods - สร้าง Color ที่กำหนดไว้ล่วงหน้า
+ (instancetype)redColor;
+ (instancetype)greenColor;
+ (instancetype)blueColor;
+ (instancetype)blackColor;
+ (instancetype)whiteColor;
+ (instancetype)clearColor;

// Factory method สำหรับ custom color
+ (instancetype)colorWithRed:(CGFloat)red
                       green:(CGFloat)green
                        blue:(CGFloat)blue;

+ (instancetype)colorWithHexString:(NSString *)hexString;

// Instance methods
- (Color *)colorWithAlpha:(CGFloat)alpha;
- (NSString *)hexString;

@end
```

```objc
// Color.m
#import "Color.h"

@implementation Color

- (instancetype)initWithRed:(CGFloat)red
                      green:(CGFloat)green
                       blue:(CGFloat)blue
                      alpha:(CGFloat)alpha {
    self = [super init];
    if (self) {
        _red   = MAX(0.0, MIN(1.0, red));
        _green = MAX(0.0, MIN(1.0, green));
        _blue  = MAX(0.0, MIN(1.0, blue));
        _alpha = MAX(0.0, MIN(1.0, alpha));
    }
    return self;
}

+ (instancetype)redColor {
    return [[self alloc] initWithRed:1.0 green:0.0 blue:0.0 alpha:1.0];
}

+ (instancetype)greenColor {
    return [[self alloc] initWithRed:0.0 green:1.0 blue:0.0 alpha:1.0];
}

+ (instancetype)blueColor {
    return [[self alloc] initWithRed:0.0 green:0.0 blue:1.0 alpha:1.0];
}

+ (instancetype)blackColor {
    return [[self alloc] initWithRed:0.0 green:0.0 blue:0.0 alpha:1.0];
}

+ (instancetype)whiteColor {
    return [[self alloc] initWithRed:1.0 green:1.0 blue:1.0 alpha:1.0];
}

+ (instancetype)clearColor {
    return [[self alloc] initWithRed:0.0 green:0.0 blue:0.0 alpha:0.0];
}

+ (instancetype)colorWithRed:(CGFloat)red
                       green:(CGFloat)green
                        blue:(CGFloat)blue {
    return [[self alloc] initWithRed:red green:green blue:blue alpha:1.0];
}

+ (instancetype)colorWithHexString:(NSString *)hexString {
    NSString *cleaned = [hexString stringByReplacingOccurrencesOfString:@"#"
                                                             withString:@""];
    if (cleaned.length != 6) return [self blackColor];

    unsigned int rgbValue = 0;
    NSScanner *scanner = [NSScanner scannerWithString:cleaned];
    [scanner scanHexInt:&rgbValue];

    CGFloat r = ((rgbValue & 0xFF0000) >> 16) / 255.0;
    CGFloat g = ((rgbValue & 0x00FF00) >>  8) / 255.0;
    CGFloat b =  (rgbValue & 0x0000FF)        / 255.0;

    return [[self alloc] initWithRed:r green:g blue:b alpha:1.0];
}

- (Color *)colorWithAlpha:(CGFloat)alpha {
    return [[Color alloc] initWithRed:_red
                                green:_green
                                 blue:_blue
                                alpha:alpha];
}

- (NSString *)hexString {
    NSInteger r = (NSInteger)(_red   * 255);
    NSInteger g = (NSInteger)(_green * 255);
    NSInteger b = (NSInteger)(_blue  * 255);
    return [NSString stringWithFormat:@"#%02lX%02lX%02lX",
            (long)r, (long)g, (long)b];
}

- (NSString *)description {
    return [NSString stringWithFormat:
            @"Color(r=%.3f, g=%.3f, b=%.3f, a=%.3f) %@",
            _red, _green, _blue, _alpha, [self hexString]];
}

@end
```

---

## 10. description Method

`description` คือ method ที่ถูกเรียกเมื่อใช้ `%@` ใน NSLog หรือ string formatting

```objc
// Employee.h
@interface Employee : NSObject

@property (nonatomic, copy)   NSString   *employeeID;
@property (nonatomic, copy)   NSString   *firstName;
@property (nonatomic, copy)   NSString   *lastName;
@property (nonatomic, copy)   NSString   *department;
@property (nonatomic, assign) double      salary;

@end
```

```objc
// Employee.m
@implementation Employee

- (NSString *)description {
    // รูปแบบที่อ่านง่ายสำหรับ debugging
    return [NSString stringWithFormat:
            @"<Employee: %@ | %@ %@ | %@ | ฿%.2f>",
            _employeeID, _firstName, _lastName, _department, _salary];
}

// debugDescription - แสดงเมื่อ debug ใน debugger (LLDB)
- (NSString *)debugDescription {
    return [NSString stringWithFormat:
            @"Employee{\n"
            @"  id: %@\n"
            @"  name: %@ %@\n"
            @"  dept: %@\n"
            @"  salary: %.2f\n"
            @"}",
            _employeeID, _firstName, _lastName, _department, _salary];
}

@end
```

---

## 11. isEqual: และ hash Methods

ถ้า override `isEqual:` ต้อง override `hash` ด้วยเสมอ เพราะ objects ที่ equal ต้องมี hash เท่ากัน

```objc
// Point.h
@interface Point : NSObject <NSCopying>

@property (nonatomic, assign) CGFloat x;
@property (nonatomic, assign) CGFloat y;

- (instancetype)initWithX:(CGFloat)x y:(CGFloat)y;

@end
```

```objc
// Point.m
@implementation Point

- (instancetype)initWithX:(CGFloat)x y:(CGFloat)y {
    self = [super init];
    if (self) {
        _x = x;
        _y = y;
    }
    return self;
}

// Override isEqual:
- (BOOL)isEqual:(id)object {
    // 1. ตรวจว่าเป็น object เดียวกันหรือไม่ (pointer equality)
    if (self == object) return YES;

    // 2. ตรวจประเภทของ object
    if (![object isKindOfClass:[Point class]]) return NO;

    // 3. เปรียบเทียบ properties
    Point *other = (Point *)object;
    return (fabs(_x - other.x) < 1e-10) && (fabs(_y - other.y) < 1e-10);
}

// Override hash - objects ที่ equal ต้องมี hash เท่ากัน
- (NSUInteger)hash {
    // ใช้ XOR และ bit shifting เพื่อสร้าง hash ที่ดี
    return (NSUInteger)(_x * 1000) ^ ((NSUInteger)(_y * 1000) << 16);
}

// NSCopying protocol
- (id)copyWithZone:(NSZone *)zone {
    Point *copy = [[Point allocWithZone:zone] init];
    copy->_x = _x;
    copy->_y = _y;
    return copy;
}

- (NSString *)description {
    return [NSString stringWithFormat:@"Point(%.2f, %.2f)", _x, _y];
}

@end
```

```objc
// การทดสอบ isEqual: และ hash
Point *p1 = [[Point alloc] initWithX:3.0 y:4.0];
Point *p2 = [[Point alloc] initWithX:3.0 y:4.0];
Point *p3 = [[Point alloc] initWithX:1.0 y:2.0];

NSLog(@"p1 == p2: %@", [p1 isEqual:p2] ? @"ใช่" : @"ไม่");  // ใช่
NSLog(@"p1 == p3: %@", [p1 isEqual:p3] ? @"ใช่" : @"ไม่");  // ไม่
NSLog(@"hash p1: %lu", (unsigned long)[p1 hash]);
NSLog(@"hash p2: %lu", (unsigned long)[p2 hash]);  // hash เดียวกับ p1

// ใช้ใน NSDictionary และ NSSet ได้อย่างถูกต้อง
NSSet *pointSet = [NSSet setWithObjects:p1, p2, p3, nil];
NSLog(@"จำนวน unique points: %lu", (unsigned long)pointSet.count);  // 2
```

---

## 12. NSCopying Protocol

NSCopying protocol ทำให้ object สามารถถูก copy ได้ด้วย `[obj copy]`

```objc
// Address.h
#import <Foundation/Foundation.h>

@interface Address : NSObject <NSCopying, NSMutableCopying>

@property (nonatomic, copy) NSString *street;
@property (nonatomic, copy) NSString *city;
@property (nonatomic, copy) NSString *province;
@property (nonatomic, copy) NSString *zipCode;

- (instancetype)initWithStreet:(NSString *)street
                          city:(NSString *)city
                      province:(NSString *)province
                       zipCode:(NSString *)zipCode;

@end

// Mutable version
@interface MutableAddress : Address

@property (nonatomic, copy, readwrite) NSString *street;
@property (nonatomic, copy, readwrite) NSString *city;
@property (nonatomic, copy, readwrite) NSString *province;
@property (nonatomic, copy, readwrite) NSString *zipCode;

@end
```

```objc
// Address.m
@implementation Address

- (instancetype)initWithStreet:(NSString *)street
                          city:(NSString *)city
                      province:(NSString *)province
                       zipCode:(NSString *)zipCode {
    self = [super init];
    if (self) {
        _street   = [street   copy];
        _city     = [city     copy];
        _province = [province copy];
        _zipCode  = [zipCode  copy];
    }
    return self;
}

// NSCopying - สร้าง immutable copy
- (id)copyWithZone:(NSZone *)zone {
    Address *copy = [[Address allocWithZone:zone]
                      initWithStreet:_street
                                city:_city
                            province:_province
                             zipCode:_zipCode];
    return copy;
}

// NSMutableCopying - สร้าง mutable copy
- (id)mutableCopyWithZone:(NSZone *)zone {
    MutableAddress *copy = [[MutableAddress allocWithZone:zone]
                             initWithStreet:_street
                                       city:_city
                                   province:_province
                                    zipCode:_zipCode];
    return copy;
}

- (NSString *)description {
    return [NSString stringWithFormat:
            @"%@, %@, %@ %@", _street, _city, _province, _zipCode];
}

@end

@implementation MutableAddress
// Properties ถูก override เป็น readwrite ใน .h แล้ว
@end
```

---

## 13. ตัวอย่างครบถ้วน: BankAccount Class

```objc
// BankAccount.h
#import <Foundation/Foundation.h>

typedef NS_ENUM(NSInteger, BankAccountType) {
    BankAccountTypeSavings,   // ออมทรัพย์
    BankAccountTypeCurrent,   // กระแสรายวัน
    BankAccountTypeFixed      // ฝากประจำ
};

typedef NS_ENUM(NSInteger, TransactionType) {
    TransactionTypeDeposit,   // ฝาก
    TransactionTypeWithdrawal, // ถอน
    TransactionTypeTransfer   // โอน
};

@interface Transaction : NSObject

@property (nonatomic, assign, readonly) TransactionType type;
@property (nonatomic, assign, readonly) double          amount;
@property (nonatomic, assign, readonly) double          balanceAfter;
@property (nonatomic, strong, readonly) NSDate         *date;
@property (nonatomic, copy,   readonly) NSString       *note;

+ (instancetype)transactionWithType:(TransactionType)type
                             amount:(double)amount
                       balanceAfter:(double)balanceAfter
                               note:(NSString *)note;

@end

@interface BankAccount : NSObject <NSCopying>

@property (nonatomic, copy,   readonly) NSString          *accountNumber;
@property (nonatomic, copy)             NSString          *ownerName;
@property (nonatomic, assign, readonly) BankAccountType    type;
@property (nonatomic, assign, readonly) double             balance;
@property (nonatomic, strong, readonly) NSArray<Transaction *> *transactions;

// Designated initializer
- (instancetype)initWithAccountNumber:(NSString *)accountNumber
                            ownerName:(NSString *)ownerName
                                 type:(BankAccountType)type
                        initialBalance:(double)initialBalance
    NS_DESIGNATED_INITIALIZER;

// Factory methods
+ (instancetype)savingsAccountForOwner:(NSString *)ownerName;
+ (instancetype)currentAccountForOwner:(NSString *)ownerName;

// Operations
- (BOOL)depositAmount:(double)amount note:(NSString *)note;
- (BOOL)withdrawAmount:(double)amount note:(NSString *)note;
- (BOOL)transferAmount:(double)amount toAccount:(BankAccount *)targetAccount note:(NSString *)note;

// Info
- (NSString *)accountStatement;
- (NSString *)accountTypeString;

@end
```

```objc
// BankAccount.m
#import "BankAccount.h"

// =================== Transaction ===================
@implementation Transaction {
    TransactionType _type;
    double          _amount;
    double          _balanceAfter;
    NSDate         *_date;
    NSString       *_note;
}

+ (instancetype)transactionWithType:(TransactionType)type
                             amount:(double)amount
                       balanceAfter:(double)balanceAfter
                               note:(NSString *)note {
    Transaction *t = [[Transaction alloc] init];
    t->_type = type;
    t->_amount = amount;
    t->_balanceAfter = balanceAfter;
    t->_date = [NSDate date];
    t->_note = [note copy];
    return t;
}

- (TransactionType)type         { return _type; }
- (double)amount                { return _amount; }
- (double)balanceAfter          { return _balanceAfter; }
- (NSDate *)date                { return _date; }
- (NSString *)note              { return _note; }

- (NSString *)description {
    NSString *typeStr;
    switch (_type) {
        case TransactionTypeDeposit:    typeStr = @"ฝาก";  break;
        case TransactionTypeWithdrawal: typeStr = @"ถอน";  break;
        case TransactionTypeTransfer:   typeStr = @"โอน";  break;
    }
    return [NSString stringWithFormat:@"[%@] %@ ฿%.2f (คงเหลือ: ฿%.2f) - %@",
            typeStr, _date, _amount, _balanceAfter, _note];
}

@end

// =================== BankAccount ===================
@interface BankAccount ()
@property (nonatomic, assign, readwrite) double balance;
@property (nonatomic, strong) NSMutableArray<Transaction *> *mutableTransactions;
@end

@implementation BankAccount

static NSInteger _accountCounter = 1000;

- (instancetype)initWithAccountNumber:(NSString *)accountNumber
                            ownerName:(NSString *)ownerName
                                 type:(BankAccountType)type
                        initialBalance:(double)initialBalance {
    self = [super init];
    if (self) {
        _accountNumber = [accountNumber copy];
        _ownerName = [ownerName copy];
        _type = type;
        _balance = MAX(0, initialBalance);
        _mutableTransactions = [NSMutableArray array];

        if (initialBalance > 0) {
            Transaction *t = [Transaction transactionWithType:TransactionTypeDeposit
                                                       amount:initialBalance
                                                 balanceAfter:_balance
                                                         note:@"เปิดบัญชี"];
            [_mutableTransactions addObject:t];
        }
    }
    return self;
}

- (instancetype)init {
    NSString *accNum = [NSString stringWithFormat:@"ACC%06ld", (long)(++_accountCounter)];
    return [self initWithAccountNumber:accNum
                             ownerName:@"ไม่ระบุชื่อ"
                                  type:BankAccountTypeSavings
                         initialBalance:0];
}

+ (instancetype)savingsAccountForOwner:(NSString *)ownerName {
    NSString *accNum = [NSString stringWithFormat:@"SAV%06ld", (long)(++_accountCounter)];
    return [[self alloc] initWithAccountNumber:accNum
                                     ownerName:ownerName
                                          type:BankAccountTypeSavings
                                 initialBalance:0];
}

+ (instancetype)currentAccountForOwner:(NSString *)ownerName {
    NSString *accNum = [NSString stringWithFormat:@"CUR%06ld", (long)(++_accountCounter)];
    return [[self alloc] initWithAccountNumber:accNum
                                     ownerName:ownerName
                                          type:BankAccountTypeCurrent
                                 initialBalance:0];
}

- (NSArray<Transaction *> *)transactions {
    return [_mutableTransactions copy];
}

- (BOOL)depositAmount:(double)amount note:(NSString *)note {
    if (amount <= 0) {
        NSLog(@"จำนวนเงินฝากต้องมากกว่า 0");
        return NO;
    }

    _balance += amount;
    Transaction *t = [Transaction transactionWithType:TransactionTypeDeposit
                                               amount:amount
                                         balanceAfter:_balance
                                                 note:note ?: @"ฝากเงิน"];
    [_mutableTransactions addObject:t];
    NSLog(@"[%@] ฝากเงิน ฿%.2f สำเร็จ คงเหลือ: ฿%.2f", _accountNumber, amount, _balance);
    return YES;
}

- (BOOL)withdrawAmount:(double)amount note:(NSString *)note {
    if (amount <= 0) {
        NSLog(@"จำนวนเงินถอนต้องมากกว่า 0");
        return NO;
    }
    if (amount > _balance) {
        NSLog(@"[%@] ยอดเงินไม่พอ (มี: ฿%.2f, ถอน: ฿%.2f)", _accountNumber, _balance, amount);
        return NO;
    }

    _balance -= amount;
    Transaction *t = [Transaction transactionWithType:TransactionTypeWithdrawal
                                               amount:amount
                                         balanceAfter:_balance
                                                 note:note ?: @"ถอนเงิน"];
    [_mutableTransactions addObject:t];
    NSLog(@"[%@] ถอนเงิน ฿%.2f สำเร็จ คงเหลือ: ฿%.2f", _accountNumber, amount, _balance);
    return YES;
}

- (BOOL)transferAmount:(double)amount toAccount:(BankAccount *)targetAccount note:(NSString *)note {
    if (!targetAccount) {
        NSLog(@"ไม่พบบัญชีปลายทาง");
        return NO;
    }

    NSString *withdrawNote = [NSString stringWithFormat:@"โอนไป %@: %@",
                              targetAccount.accountNumber, note ?: @""];
    BOOL success = [self withdrawAmount:amount note:withdrawNote];

    if (success) {
        NSString *depositNote = [NSString stringWithFormat:@"รับโอนจาก %@: %@",
                                 _accountNumber, note ?: @""];
        [targetAccount depositAmount:amount note:depositNote];
    }

    return success;
}

- (NSString *)accountTypeString {
    switch (_type) {
        case BankAccountTypeSavings:  return @"ออมทรัพย์";
        case BankAccountTypeCurrent:  return @"กระแสรายวัน";
        case BankAccountTypeFixed:    return @"ฝากประจำ";
    }
}

- (NSString *)accountStatement {
    NSMutableString *stmt = [NSMutableString string];
    [stmt appendString:@"===================================\n"];
    [stmt appendFormat:@"สถานะบัญชี\n"];
    [stmt appendFormat:@"เลขบัญชี  : %@\n", _accountNumber];
    [stmt appendFormat:@"ชื่อเจ้าของ: %@\n", _ownerName];
    [stmt appendFormat:@"ประเภท    : %@\n", [self accountTypeString]];
    [stmt appendFormat:@"ยอดคงเหลือ: ฿%.2f\n", _balance];
    [stmt appendString:@"-----------------------------------\n"];
    [stmt appendString:@"รายการทำธุรกรรม:\n"];
    for (Transaction *t in _mutableTransactions) {
        [stmt appendFormat:@"  %@\n", t];
    }
    [stmt appendString:@"==================================="];
    return stmt;
}

- (id)copyWithZone:(NSZone *)zone {
    BankAccount *copy = [[BankAccount allocWithZone:zone]
                          initWithAccountNumber:_accountNumber
                                      ownerName:_ownerName
                                           type:_type
                                  initialBalance:0];
    copy->_balance = _balance;
    copy->_mutableTransactions = [_mutableTransactions mutableCopy];
    return copy;
}

- (NSString *)description {
    return [NSString stringWithFormat:
            @"BankAccount[%@ | %@ | %@ | ฿%.2f]",
            _accountNumber, _ownerName, [self accountTypeString], _balance];
}

@end
```

```objc
// การใช้งาน BankAccount
int main(int argc, const char * argv[]) {
    @autoreleasepool {
        BankAccount *myAccount = [BankAccount savingsAccountForOwner:@"สมชาย ใจดี"];
        BankAccount *friendAccount = [BankAccount savingsAccountForOwner:@"สมหญิง รักเรียน"];

        // ฝากเงิน
        [myAccount depositAmount:50000 note:@"เงินเดือน"];
        [myAccount depositAmount:10000 note:@"โบนัส"];

        // ถอนเงิน
        [myAccount withdrawAmount:5000 note:@"ค่าใช้จ่าย"];

        // โอนเงิน
        [myAccount transferAmount:15000 toAccount:friendAccount note:@"ค่าเช่า"];

        // แสดงสถานะ
        NSLog(@"\n%@", [myAccount accountStatement]);
        NSLog(@"\n%@", [friendAccount accountStatement]);
    }
    return 0;
}
```

---

## 14. ตัวอย่างครบถ้วน: Vehicle Class

```objc
// Vehicle.h
#import <Foundation/Foundation.h>

typedef NS_ENUM(NSInteger, FuelType) {
    FuelTypeGasoline,   // น้ำมันเบนซิน
    FuelTypeDiesel,     // ดีเซล
    FuelTypeElectric,   // ไฟฟ้า
    FuelTypeHybrid      // ไฮบริด
};

@interface Vehicle : NSObject <NSCopying>

@property (nonatomic, copy)   NSString *make;
@property (nonatomic, copy)   NSString *model;
@property (nonatomic, assign) NSInteger year;
@property (nonatomic, assign) FuelType  fuelType;
@property (nonatomic, assign, readonly) double odometer;     // กิโลเมตรที่วิ่งแล้ว
@property (nonatomic, assign, readonly) double fuelLevel;    // 0.0 - 1.0
@property (nonatomic, assign) double   fuelCapacity;         // ลิตร
@property (nonatomic, assign, readonly) BOOL   engineRunning;

- (instancetype)initWithMake:(NSString *)make
                       model:(NSString *)model
                        year:(NSInteger)year
                    fuelType:(FuelType)fuelType
                fuelCapacity:(double)fuelCapacity NS_DESIGNATED_INITIALIZER;

// Factory methods
+ (instancetype)carWithMake:(NSString *)make model:(NSString *)model year:(NSInteger)year;
+ (instancetype)electricCarWithMake:(NSString *)make model:(NSString *)model year:(NSInteger)year;

// Operations
- (BOOL)startEngine;
- (BOOL)stopEngine;
- (BOOL)refuel:(double)liters;
- (BOOL)drive:(double)kilometers;

// Info
- (NSString *)fuelTypeString;
- (NSString *)vehicleInfo;

@end
```

```objc
// Vehicle.m
#import "Vehicle.h"

// Fuel consumption rates (ลิตร/กม.)
static const double kGasolineConsumption = 0.08;
static const double kDieselConsumption   = 0.07;
static const double kElectricConsumption = 0.0;  // ใช้ kWh แต่ simulate เป็น 0
static const double kHybridConsumption   = 0.05;

@implementation Vehicle

- (instancetype)initWithMake:(NSString *)make
                       model:(NSString *)model
                        year:(NSInteger)year
                    fuelType:(FuelType)fuelType
                fuelCapacity:(double)fuelCapacity {
    self = [super init];
    if (self) {
        _make         = [make copy];
        _model        = [model copy];
        _year         = year;
        _fuelType     = fuelType;
        _fuelCapacity = fuelCapacity;
        _fuelLevel    = 0.5;  // เริ่มที่ครึ่งถัง
        _odometer     = 0.0;
        _engineRunning = NO;
    }
    return self;
}

- (instancetype)init {
    return [self initWithMake:@"Unknown"
                        model:@"Unknown"
                         year:2020
                     fuelType:FuelTypeGasoline
                 fuelCapacity:50.0];
}

+ (instancetype)carWithMake:(NSString *)make model:(NSString *)model year:(NSInteger)year {
    return [[self alloc] initWithMake:make
                                model:model
                                 year:year
                             fuelType:FuelTypeGasoline
                         fuelCapacity:50.0];
}

+ (instancetype)electricCarWithMake:(NSString *)make model:(NSString *)model year:(NSInteger)year {
    return [[self alloc] initWithMake:make
                                model:model
                                 year:year
                             fuelType:FuelTypeElectric
                         fuelCapacity:75.0];  // 75 kWh battery
}

- (BOOL)startEngine {
    if (_engineRunning) {
        NSLog(@"[%@ %@] เครื่องยนต์ทำงานอยู่แล้ว", _make, _model);
        return NO;
    }
    if (_fuelLevel <= 0 && _fuelType != FuelTypeElectric) {
        NSLog(@"[%@ %@] เชื้อเพลิงหมด ไม่สามารถสตาร์ทได้", _make, _model);
        return NO;
    }
    _engineRunning = YES;
    NSLog(@"[%@ %@] สตาร์ทเครื่องยนต์สำเร็จ", _make, _model);
    return YES;
}

- (BOOL)stopEngine {
    if (!_engineRunning) {
        NSLog(@"[%@ %@] เครื่องยนต์ไม่ได้ทำงาน", _make, _model);
        return NO;
    }
    _engineRunning = NO;
    NSLog(@"[%@ %@] ดับเครื่องยนต์", _make, _model);
    return YES;
}

- (BOOL)refuel:(double)liters {
    if (liters <= 0) return NO;
    double currentFuel = _fuelLevel * _fuelCapacity;
    double newFuel = MIN(currentFuel + liters, _fuelCapacity);
    _fuelLevel = newFuel / _fuelCapacity;
    NSLog(@"[%@ %@] เติมเชื้อเพลิง %.1f ลิตร (ระดับ: %.0f%%)",
          _make, _model, liters, _fuelLevel * 100);
    return YES;
}

- (BOOL)drive:(double)kilometers {
    if (!_engineRunning) {
        NSLog(@"[%@ %@] ต้องสตาร์ทเครื่องยนต์ก่อน", _make, _model);
        return NO;
    }

    double consumption = 0;
    switch (_fuelType) {
        case FuelTypeGasoline: consumption = kGasolineConsumption; break;
        case FuelTypeDiesel:   consumption = kDieselConsumption;   break;
        case FuelTypeElectric: consumption = kElectricConsumption; break;
        case FuelTypeHybrid:   consumption = kHybridConsumption;   break;
    }

    double fuelNeeded = kilometers * consumption;
    double currentFuel = _fuelLevel * _fuelCapacity;

    if (fuelNeeded > currentFuel && _fuelType != FuelTypeElectric) {
        double maxKm = currentFuel / consumption;
        NSLog(@"[%@ %@] เชื้อเพลิงไม่พอ สามารถวิ่งได้อีกเพียง %.0f กม.",
              _make, _model, maxKm);
        return NO;
    }

    _odometer  += kilometers;
    _fuelLevel -= (fuelNeeded / _fuelCapacity);
    NSLog(@"[%@ %@] วิ่ง %.0f กม. (รวม: %.0f กม., เชื้อเพลิง: %.0f%%)",
          _make, _model, kilometers, _odometer, _fuelLevel * 100);
    return YES;
}

- (NSString *)fuelTypeString {
    switch (_fuelType) {
        case FuelTypeGasoline: return @"เบนซิน";
        case FuelTypeDiesel:   return @"ดีเซล";
        case FuelTypeElectric: return @"ไฟฟ้า";
        case FuelTypeHybrid:   return @"ไฮบริด";
    }
}

- (NSString *)vehicleInfo {
    return [NSString stringWithFormat:
            @"ยานพาหนะ: %d %@ %@\n"
            @"ประเภทเชื้อเพลิง: %@\n"
            @"ระยะทางรวม: %.0f กม.\n"
            @"ระดับเชื้อเพลิง: %.0f%%\n"
            @"สถานะเครื่องยนต์: %@",
            (int)_year, _make, _model,
            [self fuelTypeString],
            _odometer,
            _fuelLevel * 100,
            _engineRunning ? @"ทำงาน" : @"ดับ"];
}

- (id)copyWithZone:(NSZone *)zone {
    Vehicle *copy = [[Vehicle allocWithZone:zone]
                      initWithMake:_make
                             model:_model
                              year:_year
                          fuelType:_fuelType
                      fuelCapacity:_fuelCapacity];
    copy->_fuelLevel = _fuelLevel;
    copy->_odometer  = _odometer;
    return copy;
}

- (NSString *)description {
    return [NSString stringWithFormat:@"Vehicle[%d %@ %@ (%@) %.0f กม.]",
            (int)_year, _make, _model, [self fuelTypeString], _odometer];
}

@end
```

---

## 15. แบบฝึกหัด (10+ ข้อ)

### ข้อ 1: Student Class
สร้าง class `Student` ที่มี:
- Properties: studentID (readonly), firstName, lastName, gpa (readonly, computed)
- Array ของ grades (NSMutableArray)
- Methods: addGrade:, removeLastGrade, calculateGPA, gradeReport

### ข้อ 2: Library Book Class
สร้าง class `LibraryBook` ที่มี:
- Properties: isbn (readonly), title, author, isAvailable (readonly)
- Methods: checkout:, returnBook, bookInfo
- Factory methods: bookWithISBN:title:author:

### ข้อ 3: Temperature Converter Class
สร้าง class `Temperature` ที่มี:
- Property: celsius (กำหนดได้)
- Readonly properties: fahrenheit, kelvin (computed)
- Class methods: temperatureWithCelsius:, temperatureWithFahrenheit:, temperatureWithKelvin:
- isEqualToTemperature: และ override hash

### ข้อ 4: Shopping Cart Class
สร้าง class `ShoppingCart` ที่มี:
- NSMutableArray ของ CartItem
- Methods: addItem:, removeItemAtIndex:, clearCart, total, itemCount
- Cart item: productName, price, quantity, subtotal (computed)

### ข้อ 5: Stack Data Structure
สร้าง class `Stack` ที่มี:
- Generic-like behavior ด้วย id
- Methods: push:, pop, peek, isEmpty, count
- NSCopying implementation

### ข้อ 6: Employee Payroll
สร้าง class `Employee` และ class `Payroll`:
- Employee: id, name, baseSalary, hoursWorked
- Payroll: calculatePay, generatePayslip, addEmployee:, totalPayroll

### ข้อ 7: Matrix Class
สร้าง class `Matrix` ที่มี:
- Properties: rows, columns, data (2D array)
- Methods: initWithRows:columns:, setValue:atRow:column:, valueAtRow:column:
- Class methods: matrixAdd:to:, matrixMultiply:by:
- isEqual:, hash, description

### ข้อ 8: Clock Class
สร้าง class `Clock` ที่มี:
- Properties: hours (0-23), minutes, seconds
- Methods: tick (เพิ่ม 1 วินาที), addSeconds:, description (แสดงใน HH:MM:SS)
- isEqual:, hash
- NSCopying

### ข้อ 9: Inventory System
สร้าง class `Inventory` ที่มี:
- NSMutableDictionary ของ items (key=productCode, value=Product)
- Methods: addProduct:, removeProductWithCode:, updateQuantity:forCode:, lowStockItems, totalValue

### ข้อ 10: Observer Pattern
สร้าง class `EventEmitter` ที่มี:
- NSMutableDictionary ของ event listeners
- Methods: on:handler:, emit:withData:, removeListener:
- ใช้ blocks สำหรับ handler

### ข้อ 11 (ท้าทาย): Linked List
สร้าง class `LinkedList` และ `ListNode`:
- ListNode: value, next
- LinkedList: head, count
- Methods: appendValue:, prependValue:, removeAtIndex:, valueAtIndex:, toArray, description

### ข้อ 12 (ท้าทาย): Date Range
สร้าง class `DateRange` ที่มี:
- Properties: startDate, endDate (ทั้งคู่ NSDate)
- Methods: containsDate:, intersects:, duration (NSTimeInterval), description
- NSCopying, isEqual:, hash

---

## สรุป Part 11

ใน Part นี้เราได้เรียนรู้:

1. **Class Structure** - แบ่งเป็น @interface (header) และ @implementation
2. **Instance Variables** - @private, @protected, @public
3. **Properties** - แทนที่ ivars ด้วย @property พร้อม attributes ต่างๆ
4. **@synthesize / @dynamic** - ควบคุมการสร้าง accessor
5. **Instance vs Class Methods** - (-) สำหรับ instances, (+) สำหรับ class
6. **self และ super** - pointer ไปยัง instance ปัจจุบัน และ superclass
7. **Designated Initializer Pattern** - best practice สำหรับ initialization
8. **dealloc** - ทำความสะอาดทรัพยากรเมื่อ object ถูกลบ
9. **Factory Pattern** - ใช้ class methods สร้าง object
10. **description** - customizing object's string representation
11. **isEqual: และ hash** - value equality สำหรับ collections
12. **NSCopying** - ทำให้ object สามารถ copy ได้

ใน Part 12 จะเรียนเรื่อง **Inheritance** การสืบทอดคุณสมบัติระหว่าง class
