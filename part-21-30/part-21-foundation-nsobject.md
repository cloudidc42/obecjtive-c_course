# ตอนที่ 21 - Foundation Framework และ NSObject

## บทนำ

Foundation Framework เป็นหัวใจหลักของการพัฒนา Objective-C และ macOS/iOS applications Framework นี้มาพร้อมกับคลาสพื้นฐานที่จำเป็นสำหรับการพัฒนาแอปพลิเคชัน เช่น การจัดการ string, collection, date, URL, file system และอีกมากมาย

ในบทนี้เราจะเรียนรู้เกี่ยวกับ:
- Foundation Framework และความสำคัญ
- NSObject ซึ่งเป็น root class ของทุกคลาส
- Methods สำคัญของ NSObject
- Runtime introspection
- Method swizzling เบื้องต้น

---

## 21.1 Foundation Framework Overview

### Foundation Framework คืออะไร?

Foundation Framework คือ framework ระดับ low-level ที่ให้บริการ:
- **Data types**: NSString, NSNumber, NSDate, NSData
- **Collections**: NSArray, NSDictionary, NSSet
- **File management**: NSFileManager
- **Networking**: NSURL, NSURLSession
- **Threading**: NSThread, NSOperationQueue
- **Notifications**: NSNotificationCenter
- **Memory management**: NSAutoreleasePool (ARC จัดการแทนปัจจุบัน)
- **Object introspection**: การตรวจสอบ type และ behavior ของ objects ณ runtime

### การ import Foundation

```objc
#import <Foundation/Foundation.h>
```

เมื่อ import แล้ว คุณจะเข้าถึงคลาสทั้งหมดใน Foundation ได้ทันที

### โครงสร้างของ Foundation Framework

```
Foundation Framework
├── Value Objects
│   ├── NSString / NSMutableString
│   ├── NSNumber
│   ├── NSDate
│   └── NSData / NSMutableData
├── Collections
│   ├── NSArray / NSMutableArray
│   ├── NSDictionary / NSMutableDictionary
│   ├── NSSet / NSMutableSet
│   └── NSOrderedSet / NSMutableOrderedSet
├── OS Services
│   ├── NSFileManager
│   ├── NSBundle
│   └── NSProcessInfo
└── Networking
    ├── NSURL
    ├── NSURLRequest
    └── NSURLSession
```

---

## 21.2 NSObject - Root Class ของทุกคลาส

### NSObject คืออะไร?

`NSObject` เป็น root class ของ Objective-C class hierarchy เกือบทุกคลาสใน Objective-C จะ inherit มาจาก NSObject ไม่ทางตรงก็ทางอ้อม

```objc
@interface MyClass : NSObject
// MyClass รับ inherit จาก NSObject
@end
```

NSObject ให้ functionality พื้นฐาน:
- Memory management (retain, release, autorelease)
- Object identity และ equality
- Introspection (class, isKindOfClass:, etc.)
- Message forwarding
- Key-Value Coding (KVC)

### Declaration ของ NSObject

```objc
@interface NSObject <NSObject> {
    Class isa; // pointer to class object
}

// Class methods
+ (void)load;
+ (void)initialize;
+ (instancetype)new;
+ (instancetype)alloc;
+ (instancetype)allocWithZone:(struct _NSZone *)zone;

// Instance methods
- (instancetype)init;
- (void)dealloc;
- (id)copy;
- (id)mutableCopy;

@end
```

---

## 21.3 Methods พื้นฐานของ NSObject

### 21.3.1 init และ dealloc

`init` คือ initializer พื้นฐาน และ `dealloc` คือ method ที่เรียกเมื่อ object ถูก deallocated

```objc
#import <Foundation/Foundation.h>

@interface Person : NSObject

@property (nonatomic, strong) NSString *name;
@property (nonatomic, assign) NSInteger age;

- (instancetype)initWithName:(NSString *)name age:(NSInteger)age;

@end

@implementation Person

- (instancetype)initWithName:(NSString *)name age:(NSInteger)age {
    // ต้องเรียก [super init] ก่อนเสมอ
    self = [super init];
    if (self) {
        _name = name;
        _age = age;
        NSLog(@"Person '%@' created", _name);
    }
    return self;
}

- (instancetype)init {
    // Delegate ไปยัง designated initializer
    return [self initWithName:@"Unknown" age:0];
}

- (void)dealloc {
    // ใน ARC ไม่ต้องเรียก [super dealloc]
    // แต่สามารถทำ cleanup ได้ที่นี่
    NSLog(@"Person '%@' deallocated", _name);
}

@end

int main(int argc, const char * argv[]) {
    @autoreleasepool {
        Person *person = [[Person alloc] initWithName:@"สมชาย" age:30];
        NSLog(@"Name: %@, Age: %ld", person.name, (long)person.age);
        // person จะถูก dealloc เมื่อออกจาก scope
    }
    return 0;
}
```

**Output:**
```
Person 'สมชาย' created
Name: สมชาย, Age: 30
Person 'สมชาย' deallocated
```

### Designated Initializer Pattern

```objc
@interface Vehicle : NSObject

@property (nonatomic, strong) NSString *brand;
@property (nonatomic, assign) NSInteger year;
@property (nonatomic, strong) NSString *color;

// Designated initializer
- (instancetype)initWithBrand:(NSString *)brand
                         year:(NSInteger)year
                        color:(NSString *)color NS_DESIGNATED_INITIALIZER;

// Convenience initializers
- (instancetype)initWithBrand:(NSString *)brand year:(NSInteger)year;
- (instancetype)initWithBrand:(NSString *)brand;

@end

@implementation Vehicle

// Designated initializer ต้องเรียก [super init] หรือ super's designated initializer
- (instancetype)initWithBrand:(NSString *)brand
                         year:(NSInteger)year
                        color:(NSString *)color {
    self = [super init];
    if (self) {
        _brand = brand;
        _year = year;
        _color = color;
    }
    return self;
}

// Convenience initializers ต้อง delegate ไปยัง designated initializer
- (instancetype)initWithBrand:(NSString *)brand year:(NSInteger)year {
    return [self initWithBrand:brand year:year color:@"White"];
}

- (instancetype)initWithBrand:(NSString *)brand {
    return [self initWithBrand:brand year:2024 color:@"White"];
}

// ต้อง override init เพื่อไป delegate ให้ designated initializer
- (instancetype)init {
    return [self initWithBrand:@"Unknown" year:2024 color:@"White"];
}

@end

int main(int argc, const char * argv[]) {
    @autoreleasepool {
        Vehicle *car1 = [[Vehicle alloc] initWithBrand:@"Toyota"
                                                  year:2023
                                                 color:@"Red"];
        Vehicle *car2 = [[Vehicle alloc] initWithBrand:@"Honda" year:2022];
        Vehicle *car3 = [[Vehicle alloc] initWithBrand:@"BMW"];
        
        NSLog(@"Car1: %@ %ld %@", car1.brand, (long)car1.year, car1.color);
        NSLog(@"Car2: %@ %ld %@", car2.brand, (long)car2.year, car2.color);
        NSLog(@"Car3: %@ %ld %@", car3.brand, (long)car3.year, car3.color);
    }
    return 0;
}
```

**Output:**
```
Car1: Toyota 2023 Red
Car2: Honda 2022 White
Car3: BMW 2024 White
```

---

### 21.3.2 copy และ mutableCopy

`copy` สร้าง immutable copy ของ object ส่วน `mutableCopy` สร้าง mutable copy

```objc
#import <Foundation/Foundation.h>

// สำหรับ copy ต้อง implement NSCopying protocol
// สำหรับ mutableCopy ต้อง implement NSMutableCopying protocol

@interface Student : NSObject <NSCopying>

@property (nonatomic, strong) NSString *name;
@property (nonatomic, assign) NSInteger grade;
@property (nonatomic, strong) NSMutableArray<NSString *> *courses;

- (instancetype)initWithName:(NSString *)name grade:(NSInteger)grade;

@end

@implementation Student

- (instancetype)initWithName:(NSString *)name grade:(NSInteger)grade {
    self = [super init];
    if (self) {
        _name = name;
        _grade = grade;
        _courses = [NSMutableArray array];
    }
    return self;
}

// ต้อง implement copyWithZone: สำหรับ NSCopying
- (id)copyWithZone:(NSZone *)zone {
    Student *copy = [[[self class] allocWithZone:zone] init];
    copy.name = [_name copy]; // deep copy สำหรับ NSString
    copy.grade = _grade;
    copy.courses = [_courses mutableCopy]; // deep copy สำหรับ array
    return copy;
}

@end

int main(int argc, const char * argv[]) {
    @autoreleasepool {
        Student *student1 = [[Student alloc] initWithName:@"สมหญิง" grade:10];
        [student1.courses addObject:@"คณิตศาสตร์"];
        [student1.courses addObject:@"วิทยาศาสตร์"];
        
        // Shallow copy vs Deep copy
        Student *student2 = [student1 copy]; // NSCopying
        student2.name = @"วิชาญ";
        [student2.courses addObject:@"ภาษาอังกฤษ"];
        
        NSLog(@"Student1: %@, Courses: %@", student1.name, student1.courses);
        NSLog(@"Student2: %@, Courses: %@", student2.name, student2.courses);
        
        // สังเกตว่า courses ของ student1 ไม่ได้รับผลกระทบจาก student2
    }
    return 0;
}
```

**Shallow Copy vs Deep Copy:**

```objc
int main(int argc, const char * argv[]) {
    @autoreleasepool {
        NSMutableArray *original = [NSMutableArray arrayWithObjects:
                                    @"item1", @"item2", @"item3", nil];
        
        // Shallow copy - สร้าง array ใหม่ แต่ elements ชี้ไปที่ object เดิม
        NSMutableArray *shallowCopy = [original mutableCopy];
        
        // Deep copy - สร้าง array ใหม่ และ copy elements ด้วย
        NSArray *deepCopy = [[NSArray alloc] initWithArray:original
                                                 copyItems:YES];
        
        [original addObject:@"item4"];
        NSLog(@"Original: %@", original);      // มี item4
        NSLog(@"Shallow Copy: %@", shallowCopy); // ไม่มี item4
        NSLog(@"Deep Copy: %@", deepCopy);       // ไม่มี item4
    }
    return 0;
}
```

---

### 21.3.3 description และ debugDescription

`description` ใช้เมื่อ print object ด้วย `%@` ส่วน `debugDescription` ใช้ใน debugger

```objc
@interface Product : NSObject

@property (nonatomic, strong) NSString *name;
@property (nonatomic, assign) double price;
@property (nonatomic, strong) NSString *category;

@end

@implementation Product

// Override description สำหรับ NSLog
- (NSString *)description {
    return [NSString stringWithFormat:@"Product{name='%@', price=%.2f, category='%@'}",
            _name, _price, _category];
}

// Override debugDescription สำหรับ LLDB
- (NSString *)debugDescription {
    return [NSString stringWithFormat:
            @"<%@: %p>\n"
            @"  name = '%@'\n"
            @"  price = %.2f\n"
            @"  category = '%@'",
            NSStringFromClass([self class]), self,
            _name, _price, _category];
}

@end

int main(int argc, const char * argv[]) {
    @autoreleasepool {
        Product *p = [[Product alloc] init];
        p.name = @"MacBook Pro";
        p.price = 59900.00;
        p.category = @"Electronics";
        
        // เรียก description
        NSLog(@"%@", p);
        
        // เรียก debugDescription โดยตรง
        NSLog(@"%@", [p debugDescription]);
        
        // ใน NSArray จะเรียก description ของแต่ละ element
        NSArray *products = @[p];
        NSLog(@"Array: %@", products);
    }
    return 0;
}
```

**Output:**
```
Product{name='MacBook Pro', price=59900.00, category='Electronics'}
<Product: 0x...>
  name = 'MacBook Pro'
  price = 59900.00
  category = 'Electronics'
```

---

### 21.3.4 isEqual: และ hash

`isEqual:` ใช้เปรียบเทียบ content ของ objects และ `hash` ใช้สำหรับ hash-based collections

```objc
@interface Point : NSObject

@property (nonatomic, assign) NSInteger x;
@property (nonatomic, assign) NSInteger y;

- (instancetype)initWithX:(NSInteger)x y:(NSInteger)y;

@end

@implementation Point

- (instancetype)initWithX:(NSInteger)x y:(NSInteger)y {
    self = [super init];
    if (self) {
        _x = x;
        _y = y;
    }
    return self;
}

// Override isEqual: เพื่อ compare content
- (BOOL)isEqual:(id)other {
    // ตรวจสอบ identity ก่อน (optimization)
    if (self == other) return YES;
    
    // ตรวจสอบว่าเป็น class เดียวกัน
    if (![other isKindOfClass:[Point class]]) return NO;
    
    Point *otherPoint = (Point *)other;
    return _x == otherPoint.x && _y == otherPoint.y;
}

// ต้อง override hash เมื่อ override isEqual:
// กฎ: ถ้า a.isEqual(b) == YES แล้ว a.hash == b.hash ต้องเป็นจริงด้วย
- (NSUInteger)hash {
    return (_x * 31) ^ _y; // simple hash function
}

- (NSString *)description {
    return [NSString stringWithFormat:@"Point(%ld, %ld)", (long)_x, (long)_y];
}

@end

int main(int argc, const char * argv[]) {
    @autoreleasepool {
        Point *p1 = [[Point alloc] initWithX:3 y:4];
        Point *p2 = [[Point alloc] initWithX:3 y:4];
        Point *p3 = [[Point alloc] initWithX:5 y:6];
        
        // == เปรียบเทียบ pointer (identity)
        NSLog(@"p1 == p2 (pointer): %@", (p1 == p2) ? @"YES" : @"NO");  // NO
        
        // isEqual: เปรียบเทียบ content
        NSLog(@"p1 isEqual: p2: %@", [p1 isEqual:p2] ? @"YES" : @"NO"); // YES
        NSLog(@"p1 isEqual: p3: %@", [p1 isEqual:p3] ? @"YES" : @"NO"); // NO
        
        // hash ของ p1 และ p2 ต้องเท่ากัน
        NSLog(@"p1 hash: %lu", (unsigned long)[p1 hash]);
        NSLog(@"p2 hash: %lu", (unsigned long)[p2 hash]);
        
        // ใช้ใน NSSet
        NSSet *pointSet = [NSSet setWithObjects:p1, p2, p3, nil];
        NSLog(@"Set count: %lu", (unsigned long)pointSet.count); // 2 (p1 และ p2 เหมือนกัน)
        
        // ใช้ใน NSDictionary as key
        NSMutableDictionary *dict = [NSMutableDictionary dictionary];
        dict[p1] = @"First Point";
        NSLog(@"Dict with p2 key: %@", dict[p2]); // "First Point"
    }
    return 0;
}
```

**Output:**
```
p1 == p2 (pointer): NO
p1 isEqual: p2: YES
p1 isEqual: p3: NO
p1 hash: 97
p2 hash: 97
Set count: 2
Dict with p2 key: First Point
```

---

### 21.3.5 respondsToSelector: และ performSelector:

ใช้ตรวจสอบว่า object มี method นั้นหรือไม่ และเรียก method โดย selector

```objc
#import <Foundation/Foundation.h>

@interface Animal : NSObject
- (void)makeSound;
- (void)eat;
@end

@interface Dog : Animal
- (void)fetch;
- (void)bark;
@end

@interface Cat : Animal
- (void)purr;
@end

@implementation Animal
- (void)makeSound { NSLog(@"..."); }
- (void)eat { NSLog(@"กำลังกิน"); }
@end

@implementation Dog
- (void)makeSound { NSLog(@"โฮ่ง โฮ่ง!"); }
- (void)fetch { NSLog(@"ไปเอาของมาแล้ว!"); }
- (void)bark { NSLog(@"เห่า!"); }
@end

@implementation Cat
- (void)makeSound { NSLog(@"เมี้ยว~"); }
- (void)purr { NSLog(@"กรุ๊ กรุ๊~"); }
@end

int main(int argc, const char * argv[]) {
    @autoreleasepool {
        NSArray *animals = @[
            [[Dog alloc] init],
            [[Cat alloc] init],
            [[Dog alloc] init]
        ];
        
        for (Animal *animal in animals) {
            // ตรวจสอบก่อนเรียก method
            if ([animal respondsToSelector:@selector(fetch)]) {
                [animal performSelector:@selector(fetch)];
            }
            
            if ([animal respondsToSelector:@selector(purr)]) {
                [animal performSelector:@selector(purr)];
            }
            
            // เรียก method ที่มีอยู่
            [animal makeSound];
        }
        
        // performSelector with argument
        Dog *dog = [[Dog alloc] init];
        SEL barkSel = @selector(bark);
        
        if ([dog respondsToSelector:barkSel]) {
            [dog performSelector:barkSel];
        }
        
        // performSelector with delay
        [dog performSelector:@selector(bark)
                  withObject:nil
                  afterDelay:1.0];
        
        NSLog(@"รอ 2 วินาที...");
        [[NSRunLoop mainRunLoop] runUntilDate:[NSDate dateWithTimeIntervalSinceNow:2]];
    }
    return 0;
}
```

**performSelector กับ arguments:**

```objc
@interface Calculator : NSObject
- (NSNumber *)addNumber:(NSNumber *)a toNumber:(NSNumber *)b;
- (void)printMessage:(NSString *)message;
@end

@implementation Calculator
- (NSNumber *)addNumber:(NSNumber *)a toNumber:(NSNumber *)b {
    return @([a integerValue] + [b integerValue]);
}

- (void)printMessage:(NSString *)message {
    NSLog(@"Message: %@", message);
}
@end

int main(int argc, const char * argv[]) {
    @autoreleasepool {
        Calculator *calc = [[Calculator alloc] init];
        
        // performSelector: กับ 1 argument
        [calc performSelector:@selector(printMessage:) withObject:@"สวัสดี!"];
        
        // performSelector: กับ 2 arguments
        NSNumber *result = [calc performSelector:@selector(addNumber:toNumber:)
                                      withObject:@5
                                      withObject:@3];
        NSLog(@"Result: %@", result);
        
        // ใช้ NSInvocation สำหรับ arguments มากกว่า 2 ตัว
        SEL selector = @selector(addNumber:toNumber:);
        NSMethodSignature *signature = [calc methodSignatureForSelector:selector];
        NSInvocation *invocation = [NSInvocation invocationWithMethodSignature:signature];
        
        [invocation setTarget:calc];
        [invocation setSelector:selector];
        
        NSNumber *a = @10;
        NSNumber *b = @20;
        [invocation setArgument:&a atIndex:2]; // index 0=self, 1=_cmd, 2+=args
        [invocation setArgument:&b atIndex:3];
        [invocation invoke];
        
        NSNumber *invResult;
        [invocation getReturnValue:&invResult];
        NSLog(@"NSInvocation result: %@", invResult);
    }
    return 0;
}
```

---

## 21.4 Class Inspection Methods

### 21.4.1 class, superclass, isKindOfClass:, isMemberOfClass:

```objc
#import <Foundation/Foundation.h>

@interface Shape : NSObject
@end

@interface Circle : Shape
@end

@interface Rectangle : Shape
@end

@interface Square : Rectangle
@end

@implementation Shape @end
@implementation Circle @end
@implementation Rectangle @end
@implementation Square @end

int main(int argc, const char * argv[]) {
    @autoreleasepool {
        Square *square = [[Square alloc] init];
        
        // class - คืน class ของ object
        NSLog(@"Class: %@", [square class]);  // Square
        NSLog(@"Class name: %@", NSStringFromClass([square class]));  // "Square"
        
        // superclass - คืน superclass
        NSLog(@"Superclass: %@", [square superclass]);  // Rectangle
        
        // isKindOfClass: - true ถ้าเป็น instance ของ class หรือ subclass
        NSLog(@"isKindOfClass Square: %@",
              [square isKindOfClass:[Square class]] ? @"YES" : @"NO");     // YES
        NSLog(@"isKindOfClass Rectangle: %@",
              [square isKindOfClass:[Rectangle class]] ? @"YES" : @"NO"); // YES
        NSLog(@"isKindOfClass Shape: %@",
              [square isKindOfClass:[Shape class]] ? @"YES" : @"NO");     // YES
        NSLog(@"isKindOfClass NSObject: %@",
              [square isKindOfClass:[NSObject class]] ? @"YES" : @"NO");  // YES
        NSLog(@"isKindOfClass Circle: %@",
              [square isKindOfClass:[Circle class]] ? @"YES" : @"NO");    // NO
        
        // isMemberOfClass: - true เฉพาะ class ตรงๆ เท่านั้น
        NSLog(@"\nisMemberOfClass Square: %@",
              [square isMemberOfClass:[Square class]] ? @"YES" : @"NO");     // YES
        NSLog(@"isMemberOfClass Rectangle: %@",
              [square isMemberOfClass:[Rectangle class]] ? @"YES" : @"NO"); // NO
        NSLog(@"isMemberOfClass Shape: %@",
              [square isMemberOfClass:[Shape class]] ? @"YES" : @"NO");     // NO
        
        // ตัวอย่างการใช้งาน: Type checking
        NSArray *shapes = @[
            [[Circle alloc] init],
            [[Square alloc] init],
            [[Rectangle alloc] init],
            [[Square alloc] init]
        ];
        
        NSLog(@"\nShapes in array:");
        for (Shape *shape in shapes) {
            if ([shape isMemberOfClass:[Square class]]) {
                NSLog(@"  Square (exact)");
            } else if ([shape isKindOfClass:[Rectangle class]]) {
                NSLog(@"  Rectangle (or subclass)");
            } else if ([shape isKindOfClass:[Circle class]]) {
                NSLog(@"  Circle");
            }
        }
    }
    return 0;
}
```

**Output:**
```
Class: Square
Class name: Square
Superclass: Rectangle
isKindOfClass Square: YES
isKindOfClass Rectangle: YES
isKindOfClass Shape: YES
isKindOfClass NSObject: YES
isKindOfClass Circle: NO

isMemberOfClass Square: YES
isMemberOfClass Rectangle: NO
isMemberOfClass Shape: NO

Shapes in array:
  Circle
  Square (exact)
  Rectangle (or subclass)
  Square (exact)
```

---

### 21.4.2 conformsToProtocol:

ตรวจสอบว่า object หรือ class implement protocol นั้นหรือไม่

```objc
#import <Foundation/Foundation.h>

// กำหนด protocols
@protocol Drawable
- (void)draw;
@end

@protocol Resizable
- (void)resizeToWidth:(CGFloat)width height:(CGFloat)height;
@end

@protocol Printable <NSObject>
- (NSString *)printableDescription;
@end

@interface Circle : NSObject <Drawable, Printable>
@property (nonatomic, assign) CGFloat radius;
@end

@interface Square : NSObject <Drawable, Resizable, Printable>
@property (nonatomic, assign) CGFloat side;
@end

@implementation Circle
- (void)draw { NSLog(@"Drawing circle with radius %.1f", _radius); }
- (NSString *)printableDescription {
    return [NSString stringWithFormat:@"Circle(radius=%.1f)", _radius];
}
@end

@implementation Square
- (void)draw { NSLog(@"Drawing square with side %.1f", _side); }
- (void)resizeToWidth:(CGFloat)width height:(CGFloat)height {
    _side = MIN(width, height);
    NSLog(@"Resized square to %.1f", _side);
}
- (NSString *)printableDescription {
    return [NSString stringWithFormat:@"Square(side=%.1f)", _side];
}
@end

int main(int argc, const char * argv[]) {
    @autoreleasepool {
        NSArray *shapes = @[
            [[Circle alloc] init],
            [[Square alloc] init]
        ];
        
        for (id shape in shapes) {
            NSLog(@"--- %@ ---", NSStringFromClass([shape class]));
            
            // ตรวจสอบ protocol
            if ([shape conformsToProtocol:@protocol(Drawable)]) {
                NSLog(@"  Supports Drawable");
                [(id<Drawable>)shape draw];
            }
            
            if ([shape conformsToProtocol:@protocol(Resizable)]) {
                NSLog(@"  Supports Resizable");
                [(id<Resizable>)shape resizeToWidth:100 height:80];
            }
            
            if ([shape conformsToProtocol:@protocol(Printable)]) {
                NSLog(@"  Description: %@",
                      [(id<Printable>)shape printableDescription]);
            }
        }
        
        // ตรวจสอบที่ class level
        NSLog(@"\nClass level checks:");
        NSLog(@"Circle conformsToProtocol Drawable: %@",
              [Circle conformsToProtocol:@protocol(Drawable)] ? @"YES" : @"NO");
        NSLog(@"Circle conformsToProtocol Resizable: %@",
              [Circle conformsToProtocol:@protocol(Resizable)] ? @"YES" : @"NO");
        NSLog(@"Square conformsToProtocol Resizable: %@",
              [Square conformsToProtocol:@protocol(Resizable)] ? @"YES" : @"NO");
    }
    return 0;
}
```

---

## 21.5 +load และ +initialize

### +load: Method

`+load` เรียกเมื่อ class หรือ category ถูก load เข้า runtime (ก่อน main() ด้วยซ้ำ)

```objc
#import <Foundation/Foundation.h>
#import <objc/runtime.h>

@interface MyClass : NSObject
@end

@interface MyClass (Extras)
@end

@implementation MyClass

// เรียกเมื่อ class ถูก load
+ (void)load {
    NSLog(@"MyClass +load called");
    // ใช้สำหรับ: method swizzling, registration, etc.
}

@end

@implementation MyClass (Extras)

// Category's +load ก็เรียกเหมือนกัน
+ (void)load {
    NSLog(@"MyClass (Extras) +load called");
}

@end

// สำคัญ: ลำดับการเรียก +load
// 1. Superclass ก่อน subclass เสมอ
// 2. Class ก่อน category ของมัน
// 3. ลำดับระหว่าง unrelated classes ไม่แน่นอน
```

### +initialize: Method

`+initialize` เรียกครั้งเดียวก่อนที่ class จะถูกใช้ครั้งแรก (thread-safe)

```objc
@interface DatabaseManager : NSObject

+ (instancetype)sharedInstance;
- (void)query:(NSString *)sql;

@end

static DatabaseManager *_sharedInstance = nil;
static NSMutableDictionary *_queryCache = nil;

@implementation DatabaseManager

// เรียกครั้งเดียวก่อน class ถูกใช้ครั้งแรก
+ (void)initialize {
    // ต้องตรวจสอบ self == [DatabaseManager class]
    // เพราะ subclass อาจเรียก initialize ของ superclass
    if (self == [DatabaseManager class]) {
        _queryCache = [NSMutableDictionary dictionary];
        NSLog(@"DatabaseManager initialized with cache");
    }
}

+ (instancetype)sharedInstance {
    static dispatch_once_t onceToken;
    dispatch_once(&onceToken, ^{
        _sharedInstance = [[self alloc] init];
    });
    return _sharedInstance;
}

- (void)query:(NSString *)sql {
    if (_queryCache[sql]) {
        NSLog(@"Cache hit: %@", sql);
    } else {
        _queryCache[sql] = @YES;
        NSLog(@"Executing: %@", sql);
    }
}

@end

int main(int argc, const char * argv[]) {
    @autoreleasepool {
        // +initialize จะเรียกตอนนี้
        DatabaseManager *db = [DatabaseManager sharedInstance];
        [db query:@"SELECT * FROM users"];
        [db query:@"SELECT * FROM users"]; // cache hit
        [db query:@"SELECT * FROM products"];
    }
    return 0;
}
```

**ข้อแตกต่าง +load vs +initialize:**

| Feature | +load | +initialize |
|---------|-------|-------------|
| เรียกเมื่อ | Class load เข้า memory | ก่อนใช้ class ครั้งแรก |
| ครั้งที่เรียก | ทุก class และ category | ครั้งเดียว |
| Thread safe | ไม่ (main thread) | ใช่ (protected) |
| Superclass | ไม่ inherit | Inherit (ต้องตรวจ self) |
| ใช้สำหรับ | Method swizzling | One-time setup |

---

## 21.6 Runtime Introspection

### การใช้ Objective-C Runtime

```objc
#import <Foundation/Foundation.h>
#import <objc/runtime.h>

@interface Car : NSObject

@property (nonatomic, strong) NSString *brand;
@property (nonatomic, strong) NSString *model;
@property (nonatomic, assign) NSInteger year;

- (void)startEngine;
- (void)stopEngine;
- (NSString *)getInfo;

@end

@implementation Car

- (void)startEngine { NSLog(@"Engine started"); }
- (void)stopEngine { NSLog(@"Engine stopped"); }
- (NSString *)getInfo {
    return [NSString stringWithFormat:@"%@ %@ (%ld)",
            _brand, _model, (long)_year];
}

@end

void inspectClass(Class cls) {
    NSLog(@"\n=== Inspecting class: %@ ===", NSStringFromClass(cls));
    
    // ดูชื่อ class
    NSLog(@"Class name: %s", class_getName(cls));
    
    // ดู superclass
    Class superclass = class_getSuperclass(cls);
    if (superclass) {
        NSLog(@"Superclass: %s", class_getName(superclass));
    }
    
    // ดู instance methods
    NSLog(@"\nInstance Methods:");
    unsigned int methodCount;
    Method *methods = class_copyMethodList(cls, &methodCount);
    for (unsigned int i = 0; i < methodCount; i++) {
        SEL sel = method_getName(methods[i]);
        NSLog(@"  - %s", sel_getName(sel));
    }
    free(methods);
    
    // ดู properties
    NSLog(@"\nProperties:");
    unsigned int propertyCount;
    objc_property_t *properties = class_copyPropertyList(cls, &propertyCount);
    for (unsigned int i = 0; i < propertyCount; i++) {
        const char *name = property_getName(properties[i]);
        const char *attributes = property_getAttributes(properties[i]);
        NSLog(@"  - %s (attributes: %s)", name, attributes);
    }
    free(properties);
    
    // ดู instance variables
    NSLog(@"\nInstance Variables:");
    unsigned int ivarCount;
    Ivar *ivars = class_copyIvarList(cls, &ivarCount);
    for (unsigned int i = 0; i < ivarCount; i++) {
        const char *name = ivar_getName(ivars[i]);
        const char *type = ivar_getTypeEncoding(ivars[i]);
        NSLog(@"  - %s (type: %s)", name, type);
    }
    free(ivars);
}

int main(int argc, const char * argv[]) {
    @autoreleasepool {
        inspectClass([Car class]);
    }
    return 0;
}
```

### ดู Protocols ที่ Implement

```objc
void inspectProtocols(Class cls) {
    NSLog(@"\nProtocols:");
    unsigned int protocolCount;
    Protocol * __unsafe_unretained *protocols =
        class_copyProtocolList(cls, &protocolCount);
    
    for (unsigned int i = 0; i < protocolCount; i++) {
        NSLog(@"  - %s", protocol_getName(protocols[i]));
    }
    free(protocols);
}
```

### Dynamic Method Resolution

```objc
#import <Foundation/Foundation.h>
#import <objc/runtime.h>

@interface DynamicClass : NSObject
@end

@implementation DynamicClass

// เรียกเมื่อ method ไม่พบ - โอกาสเพิ่ม method dynamically
+ (BOOL)resolveInstanceMethod:(SEL)sel {
    NSString *selName = NSStringFromSelector(sel);
    
    if ([selName hasPrefix:@"dynamic_"]) {
        // เพิ่ม method แบบ dynamic
        IMP dynamicImpl = imp_implementationWithBlock(^(id self) {
            NSLog(@"Dynamic method called: %@", selName);
        });
        
        class_addMethod([self class], sel, dynamicImpl, "v@:");
        return YES;
    }
    
    return [super resolveInstanceMethod:sel];
}

@end

int main(int argc, const char * argv[]) {
    @autoreleasepool {
        DynamicClass *obj = [[DynamicClass alloc] init];
        
        // method เหล่านี้ไม่มีอยู่ แต่จะถูก resolve dynamically
        #pragma clang diagnostic push
        #pragma clang diagnostic ignored "-Wundeclared-selector"
        [obj performSelector:@selector(dynamic_hello)];
        [obj performSelector:@selector(dynamic_greet)];
        #pragma clang diagnostic pop
    }
    return 0;
}
```

---

## 21.7 Method Swizzling เบื้องต้น

Method Swizzling คือการ swap implementation ของ 2 methods ณ runtime

### ตัวอย่างพื้นฐาน: Swizzling NSString

```objc
#import <Foundation/Foundation.h>
#import <objc/runtime.h>

@implementation NSString (Swizzling)

// Method ใหม่ที่จะ replace uppercaseString
- (NSString *)swizzled_uppercaseString {
    // เรียก original (ที่ตอนนี้ชื่อ swizzled_uppercaseString)
    NSString *result = [self swizzled_uppercaseString];
    NSLog(@"[SWIZZLED] uppercaseString called on: '%@'", self);
    return result;
}

+ (void)load {
    static dispatch_once_t onceToken;
    dispatch_once(&onceToken, ^{
        Class class = [self class];
        
        SEL originalSelector = @selector(uppercaseString);
        SEL swizzledSelector = @selector(swizzled_uppercaseString);
        
        Method originalMethod = class_getInstanceMethod(class, originalSelector);
        Method swizzledMethod = class_getInstanceMethod(class, swizzledSelector);
        
        // ลอง add method ก่อน (กรณีที่ subclass อาจไม่มี method นั้น)
        BOOL didAddMethod = class_addMethod(
            class,
            originalSelector,
            method_getImplementation(swizzledMethod),
            method_getTypeEncoding(swizzledMethod)
        );
        
        if (didAddMethod) {
            class_replaceMethod(
                class,
                swizzledSelector,
                method_getImplementation(originalMethod),
                method_getTypeEncoding(originalMethod)
            );
        } else {
            method_exchangeImplementations(originalMethod, swizzledMethod);
        }
    });
}

@end
```

### Swizzling เพื่อ Logging (Practical Use Case)

```objc
#import <Foundation/Foundation.h>
#import <objc/runtime.h>

@interface UIViewController (Tracking)
@end

@implementation UIViewController (Tracking)

- (void)tracked_viewDidAppear:(BOOL)animated {
    [self tracked_viewDidAppear:animated]; // เรียก original
    NSLog(@"[Analytics] Screen viewed: %@",
          NSStringFromClass([self class]));
}

+ (void)load {
    static dispatch_once_t onceToken;
    dispatch_once(&onceToken, ^{
        Class cls = [self class];
        SEL original = @selector(viewDidAppear:);
        SEL swizzled = @selector(tracked_viewDidAppear:);
        
        Method originalMethod = class_getInstanceMethod(cls, original);
        Method swizzledMethod = class_getInstanceMethod(cls, swizzled);
        
        method_exchangeImplementations(originalMethod, swizzledMethod);
    });
}

@end
```

### ข้อควรระวังในการใช้ Method Swizzling

```objc
// หลักการสำคัญ:
// 1. ทำใน +load เท่านั้น ไม่ใช่ +initialize
// 2. ใช้ dispatch_once เพื่อป้องกัน swizzle สองครั้ง
// 3. ตั้งชื่อ swizzled method ให้ชัดเจน (prefix ด้วย vendor name)
// 4. เรียก original implementation เสมอ (ยกเว้นจงใจไม่ต้องการ)
// 5. ระวัง class hierarchy - swizzle ที่ level ที่ถูกต้อง
```

---

## 21.8 ตัวอย่างปฏิบัติ: Custom NSObject Subclass

### ตัวอย่างที่ 1: Observable Object

```objc
#import <Foundation/Foundation.h>

// Protocol สำหรับ observer
@protocol PropertyObserver <NSObject>
- (void)object:(id)object
       didChangeProperty:(NSString *)propertyName
               fromValue:(id)oldValue
                 toValue:(id)newValue;
@end

@interface ObservableObject : NSObject

- (void)addObserver:(id<PropertyObserver>)observer;
- (void)removeObserver:(id<PropertyObserver>)observer;
- (void)notifyObservers:(NSString *)property
              fromValue:(id)old
                toValue:(id)new;

@end

@implementation ObservableObject {
    NSMutableArray<id<PropertyObserver>> *_observers;
}

- (instancetype)init {
    self = [super init];
    if (self) {
        _observers = [NSMutableArray array];
    }
    return self;
}

- (void)addObserver:(id<PropertyObserver>)observer {
    if (![_observers containsObject:observer]) {
        [_observers addObject:observer];
    }
}

- (void)removeObserver:(id<PropertyObserver>)observer {
    [_observers removeObject:observer];
}

- (void)notifyObservers:(NSString *)property
              fromValue:(id)old
                toValue:(id)new {
    for (id<PropertyObserver> observer in _observers) {
        [observer object:self
       didChangeProperty:property
               fromValue:old
                 toValue:new];
    }
}

@end

// ใช้งาน
@interface BankAccount : ObservableObject

@property (nonatomic, strong) NSString *owner;
@property (nonatomic, assign) double balance;

- (void)deposit:(double)amount;
- (BOOL)withdraw:(double)amount;

@end

@implementation BankAccount

- (void)setBalance:(double)balance {
    double old = _balance;
    _balance = balance;
    [self notifyObservers:@"balance"
                fromValue:@(old)
                  toValue:@(balance)];
}

- (void)deposit:(double)amount {
    self.balance += amount;
}

- (BOOL)withdraw:(double)amount {
    if (amount > _balance) {
        NSLog(@"ยอดเงินไม่เพียงพอ");
        return NO;
    }
    self.balance -= amount;
    return YES;
}

@end

// Observer
@interface AccountLogger : NSObject <PropertyObserver>
@end

@implementation AccountLogger

- (void)object:(id)object
       didChangeProperty:(NSString *)propertyName
               fromValue:(id)oldValue
                 toValue:(id)newValue {
    NSLog(@"[LOG] %@ changed: %@ → %@", propertyName, oldValue, newValue);
}

@end

int main(int argc, const char * argv[]) {
    @autoreleasepool {
        BankAccount *account = [[BankAccount alloc] init];
        account.owner = @"สมชาย";
        account.balance = 0;
        
        AccountLogger *logger = [[AccountLogger alloc] init];
        [account addObserver:logger];
        
        [account deposit:5000];
        [account deposit:3000];
        [account withdraw:2000];
        [account withdraw:10000]; // ล้มเหลว
        
        NSLog(@"ยอดคงเหลือ: %.2f", account.balance);
    }
    return 0;
}
```

**Output:**
```
[LOG] balance changed: 0 → 5000
[LOG] balance changed: 5000 → 8000
[LOG] balance changed: 8000 → 6000
ยอดเงินไม่เพียงพอ
ยอดคงเหลือ: 6000.00
```

---

### ตัวอย่างที่ 2: Registry Pattern

```objc
#import <Foundation/Foundation.h>
#import <objc/runtime.h>

// Protocol สำหรับ registerable objects
@protocol Registerable <NSObject>
+ (NSString *)registryKey;
@end

// Central Registry
@interface ObjectRegistry : NSObject

+ (instancetype)sharedRegistry;
- (void)registerClass:(Class)cls;
- (id)createObjectForKey:(NSString *)key;
- (NSArray<NSString *> *)allRegisteredKeys;

@end

@implementation ObjectRegistry {
    NSMutableDictionary<NSString *, Class> *_registry;
}

+ (instancetype)sharedRegistry {
    static ObjectRegistry *instance = nil;
    static dispatch_once_t onceToken;
    dispatch_once(&onceToken, ^{
        instance = [[ObjectRegistry alloc] init];
    });
    return instance;
}

- (instancetype)init {
    self = [super init];
    if (self) {
        _registry = [NSMutableDictionary dictionary];
    }
    return self;
}

- (void)registerClass:(Class)cls {
    if ([cls conformsToProtocol:@protocol(Registerable)]) {
        NSString *key = [cls registryKey];
        _registry[key] = cls;
        NSLog(@"Registered: %@ for key '%@'", NSStringFromClass(cls), key);
    }
}

- (id)createObjectForKey:(NSString *)key {
    Class cls = _registry[key];
    if (cls) {
        return [[cls alloc] init];
    }
    return nil;
}

- (NSArray<NSString *> *)allRegisteredKeys {
    return [_registry allKeys];
}

@end

// Classes ที่ register ตัวเอง
@interface TextPlugin : NSObject <Registerable>
- (void)process:(NSString *)input;
@end

@interface ImagePlugin : NSObject <Registerable>
- (void)process:(NSString *)input;
@end

@implementation TextPlugin
+ (NSString *)registryKey { return @"text"; }
+ (void)load {
    [[ObjectRegistry sharedRegistry] registerClass:self];
}
- (void)process:(NSString *)input {
    NSLog(@"TextPlugin processing: %@", input);
}
@end

@implementation ImagePlugin
+ (NSString *)registryKey { return @"image"; }
+ (void)load {
    [[ObjectRegistry sharedRegistry] registerClass:self];
}
- (void)process:(NSString *)input {
    NSLog(@"ImagePlugin processing: %@", input);
}
@end

int main(int argc, const char * argv[]) {
    @autoreleasepool {
        ObjectRegistry *registry = [ObjectRegistry sharedRegistry];
        NSLog(@"Registered keys: %@", registry.allRegisteredKeys);
        
        // สร้าง object จาก key
        id textPlugin = [registry createObjectForKey:@"text"];
        id imagePlugin = [registry createObjectForKey:@"image"];
        
        if ([textPlugin respondsToSelector:@selector(process:)]) {
            [textPlugin performSelector:@selector(process:) withObject:@"Hello World"];
        }
        
        if ([imagePlugin respondsToSelector:@selector(process:)]) {
            [imagePlugin performSelector:@selector(process:) withObject:@"photo.jpg"];
        }
    }
    return 0;
}
```

---

### ตัวอย่างที่ 3: Introspection-based Serialization

```objc
#import <Foundation/Foundation.h>
#import <objc/runtime.h>

// Category บน NSObject สำหรับ serialize เป็น dictionary
@interface NSObject (Serialization)

- (NSDictionary *)toDictionary;
+ (instancetype)fromDictionary:(NSDictionary *)dict;

@end

@implementation NSObject (Serialization)

- (NSDictionary *)toDictionary {
    NSMutableDictionary *result = [NSMutableDictionary dictionary];
    
    Class cls = [self class];
    while (cls && cls != [NSObject class]) {
        unsigned int count;
        objc_property_t *properties = class_copyPropertyList(cls, &count);
        
        for (unsigned int i = 0; i < count; i++) {
            NSString *name = @(property_getName(properties[i]));
            id value = [self valueForKey:name];
            
            if (value) {
                if ([value isKindOfClass:[NSObject class]] &&
                    ![value isKindOfClass:[NSString class]] &&
                    ![value isKindOfClass:[NSNumber class]] &&
                    ![value isKindOfClass:[NSDate class]]) {
                    result[name] = [value toDictionary];
                } else {
                    result[name] = value;
                }
            }
        }
        free(properties);
        cls = [cls superclass];
    }
    
    return [result copy];
}

+ (instancetype)fromDictionary:(NSDictionary *)dict {
    id instance = [[self alloc] init];
    [dict enumerateKeysAndObjectsUsingBlock:^(id key, id obj, BOOL *stop) {
        if ([instance respondsToSelector:NSSelectorFromString(
                [NSString stringWithFormat:@"set%@%@:",
                 [[key substringToIndex:1] uppercaseString],
                 [key substringFromIndex:1]])]) {
            [instance setValue:obj forKey:key];
        }
    }];
    return instance;
}

@end

// ทดสอบ
@interface Employee : NSObject

@property (nonatomic, strong) NSString *firstName;
@property (nonatomic, strong) NSString *lastName;
@property (nonatomic, assign) NSInteger employeeId;
@property (nonatomic, strong) NSString *department;

@end

@implementation Employee @end

int main(int argc, const char * argv[]) {
    @autoreleasepool {
        Employee *emp = [[Employee alloc] init];
        emp.firstName = @"สมชาย";
        emp.lastName = @"ใจดี";
        emp.employeeId = 1001;
        emp.department = @"Engineering";
        
        // Serialize
        NSDictionary *dict = [emp toDictionary];
        NSLog(@"Serialized: %@", dict);
        
        // Deserialize
        Employee *emp2 = [Employee fromDictionary:dict];
        NSLog(@"Deserialized: %@ %@ (ID: %ld)",
              emp2.firstName, emp2.lastName, (long)emp2.employeeId);
    }
    return 0;
}
```

---

## 21.9 สรุปและแบบฝึกหัด

### สรุป Key Concepts

| Concept | Method | ใช้สำหรับ |
|---------|--------|-----------|
| Initialization | `-init`, `-initWithX:` | สร้าง object |
| Cleanup | `-dealloc` | ล้างทรัพยากร |
| Copy | `-copy`, `-mutableCopy` | คัดลอก object |
| Description | `-description`, `-debugDescription` | แสดงข้อมูล |
| Equality | `-isEqual:`, `-hash` | เปรียบเทียบ object |
| Type check | `-isKindOfClass:`, `-isMemberOfClass:` | ตรวจสอบ type |
| Method check | `-respondsToSelector:` | ตรวจสอบ method |
| Protocol check | `-conformsToProtocol:` | ตรวจสอบ protocol |
| Class event | `+load` | เมื่อ class load |
| Class init | `+initialize` | ก่อนใช้ครั้งแรก |

### แบบฝึกหัด

**แบบฝึกหัดที่ 1:** สร้างคลาส `Shape` ที่ implement `NSCopying` พร้อม `isEqual:` และ `hash` ที่ถูกต้อง โดยมี properties ได้แก่ `color` และ `borderWidth`

**แบบฝึกหัดที่ 2:** สร้าง category บน `NSObject` ชื่อ `JSONSerializable` ที่ serialize properties ทั้งหมดเป็น JSON string

**แบบฝึกหัดที่ 3:** ใช้ Runtime Introspection เพื่อ list methods ทั้งหมดที่มีใน `NSString` class

**แบบฝึกหัดที่ 4:** สร้าง Singleton ที่ใช้ `+initialize` เพื่อ setup initial state

**แบบฝึกหัดที่ 5:** Implement Method Swizzling บน `viewDidLoad` (UIViewController) เพื่อ log ชื่อ controller ทุกครั้งที่ load

### เฉลยแบบฝึกหัดที่ 1

```objc
#import <Foundation/Foundation.h>

typedef NS_ENUM(NSInteger, ShapeColor) {
    ShapeColorRed,
    ShapeColorGreen,
    ShapeColorBlue,
    ShapeColorYellow
};

@interface Shape : NSObject <NSCopying>

@property (nonatomic, assign) ShapeColor color;
@property (nonatomic, assign) CGFloat borderWidth;

- (instancetype)initWithColor:(ShapeColor)color borderWidth:(CGFloat)borderWidth;

@end

@implementation Shape

- (instancetype)initWithColor:(ShapeColor)color borderWidth:(CGFloat)borderWidth {
    self = [super init];
    if (self) {
        _color = color;
        _borderWidth = borderWidth;
    }
    return self;
}

- (id)copyWithZone:(NSZone *)zone {
    Shape *copy = [[[self class] allocWithZone:zone] init];
    copy.color = _color;
    copy.borderWidth = _borderWidth;
    return copy;
}

- (BOOL)isEqual:(id)other {
    if (self == other) return YES;
    if (![other isKindOfClass:[Shape class]]) return NO;
    Shape *otherShape = (Shape *)other;
    return _color == otherShape.color &&
           fabs(_borderWidth - otherShape.borderWidth) < 0.0001;
}

- (NSUInteger)hash {
    return (_color * 31) ^ (NSUInteger)(_borderWidth * 1000);
}

- (NSString *)description {
    NSArray *colorNames = @[@"Red", @"Green", @"Blue", @"Yellow"];
    return [NSString stringWithFormat:@"Shape(color=%@, border=%.1f)",
            colorNames[_color], _borderWidth];
}

@end

int main(int argc, const char * argv[]) {
    @autoreleasepool {
        Shape *s1 = [[Shape alloc] initWithColor:ShapeColorRed borderWidth:2.0];
        Shape *s2 = [[Shape alloc] initWithColor:ShapeColorRed borderWidth:2.0];
        Shape *s3 = [s1 copy];
        s3.color = ShapeColorBlue;
        
        NSLog(@"s1 = %@", s1);
        NSLog(@"s2 = %@", s2);
        NSLog(@"s3 = %@", s3);
        NSLog(@"s1 == s2: %@", [s1 isEqual:s2] ? @"YES" : @"NO");
        NSLog(@"s1 == s3: %@", [s1 isEqual:s3] ? @"YES" : @"NO");
        
        NSSet *set = [NSSet setWithObjects:s1, s2, s3, nil];
        NSLog(@"Set count: %lu", (unsigned long)set.count); // 2
    }
    return 0;
}
```

---

## 21.10 Best Practices สำหรับ NSObject

### 1. ใช้ NS_DESIGNATED_INITIALIZER

```objc
// ทำ
- (instancetype)initWithName:(NSString *)name NS_DESIGNATED_INITIALIZER;

// ไม่ทำ
- (instancetype)initWithName:(NSString *)name; // ไม่ชัดเจนว่าเป็น designated หรือไม่
```

### 2. ตรวจสอบ self != nil ใน init

```objc
- (instancetype)init {
    self = [super init]; // ต้อง assign กลับ
    if (self) {          // ต้องตรวจสอบ
        // setup
    }
    return self;
}
```

### 3. Override isEqual: และ hash พร้อมกันเสมอ

```objc
// ถ้า override isEqual: ต้อง override hash ด้วยเสมอ
// กฎ: isEqual: returns YES → hash values ต้องเท่ากัน

- (BOOL)isEqual:(id)other { ... }
- (NSUInteger)hash { ... } // ต้องมีด้วย!
```

### 4. description ที่ดีช่วยในการ debug

```objc
- (NSString *)description {
    // ควรแสดงข้อมูลที่เป็นประโยชน์
    return [NSString stringWithFormat:@"<%@: name=%@, value=%@>",
            NSStringFromClass([self class]), _name, _value];
}
```

### 5. ระวัง Method Swizzling

```objc
// ทำ:
// - Swizzle ใน +load เท่านั้น
// - ใช้ dispatch_once
// - เรียก original method เสมอ

// ไม่ทำ:
// - Swizzle ใน +initialize
// - Swizzle หลายครั้ง
// - ลืมเรียก original method
```

---

## บทสรุป

Foundation Framework และ NSObject เป็นรากฐานของ Objective-C development ความเข้าใจที่ดีเกี่ยวกับ:

1. **NSObject's lifecycle** - init, dealloc และ memory management
2. **Object identity vs equality** - isEqual: และ hash
3. **Runtime introspection** - การตรวจสอบ types, methods, properties
4. **Class loading** - +load และ +initialize
5. **Method swizzling** - การ modify behavior ณ runtime

จะทำให้คุณเขียน Objective-C ได้อย่างมีประสิทธิภาพและเข้าใจกลไกลึกๆ ของภาษา

ในบทต่อไปเราจะเรียนรู้เกี่ยวกับ **NSString Advanced** ซึ่งเป็นหนึ่งในคลาสที่ใช้บ่อยที่สุดใน Objective-C
