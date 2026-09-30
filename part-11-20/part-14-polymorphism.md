# ส่วนที่ 14: Polymorphism ใน Objective-C

## บทนำ

Polymorphism (พหุสัณฐาน) เป็นหนึ่งในหลักการสำคัญของการเขียนโปรแกรมเชิงวัตถุ (Object-Oriented Programming) ที่ช่วยให้เราสามารถใช้ object ของคลาสต่างๆ ผ่าน interface เดียวกันได้ ใน Objective-C นั้น polymorphism ทำงานผ่าน dynamic dispatch ซึ่งเป็นกลไกที่ runtime จะตัดสินใจว่าจะเรียก method ใดในขณะที่โปรแกรมกำลังทำงาน

---

## 14.1 Dynamic Dispatch คืออะไร?

### แนวคิดพื้นฐาน

ใน Objective-C ทุก method call ไม่ได้ถูก compile ให้เป็น direct function call เหมือนภาษา C แต่จะถูกแปลงเป็น **message send** ที่ส่งไปยัง object ในขณะ runtime

```objc
// การเรียก method ปกติ
[myObject doSomething];

// สิ่งที่เกิดขึ้นจริงเบื้องหลัง (pseudo-code)
objc_msgSend(myObject, @selector(doSomething));
```

### Static Dispatch vs Dynamic Dispatch

**Static Dispatch** (ใน C/C++):
- Compiler รู้ว่าจะเรียก function ใดตั้งแต่ compile time
- เร็วกว่าเพราะไม่ต้องหา method ตอน runtime
- ขาดความยืดหยุ่น

**Dynamic Dispatch** (ใน Objective-C):
- Runtime ตัดสินใจว่าจะเรียก method ใด
- ช้ากว่าเล็กน้อยแต่มีความยืดหยุ่นสูง
- เป็นรากฐานของ polymorphism

```objc
// ตัวอย่าง Dynamic Dispatch
@interface Animal : NSObject
- (void)makeSound;
@end

@interface Dog : Animal
- (void)makeSound;
@end

@interface Cat : Animal
- (void)makeSound;
@end

@implementation Animal
- (void)makeSound {
    NSLog(@"...");
}
@end

@implementation Dog
- (void)makeSound {
    NSLog(@"Woof!");
}
@end

@implementation Cat
- (void)makeSound {
    NSLog(@"Meow!");
}
@end

// การใช้งาน
Animal *animal;

animal = [[Dog alloc] init];
[animal makeSound]; // แสดง "Woof!" - runtime รู้ว่าเป็น Dog

animal = [[Cat alloc] init];
[animal makeSound]; // แสดง "Meow!" - runtime รู้ว่าเป็น Cat
```

---

## 14.2 Method Overriding และ Polymorphism

### การ Override Method

Method overriding คือการที่ subclass นิยาม method ที่มีชื่อเหมือนกับ superclass ใหม่

```objc
// ไฟล์: Shape.h
@interface Shape : NSObject

@property (nonatomic, assign) NSString *color;

- (double)area;
- (double)perimeter;
- (void)draw;
- (NSString *)description;

@end

// ไฟล์: Shape.m
@implementation Shape

- (double)area {
    return 0.0; // Base implementation
}

- (double)perimeter {
    return 0.0; // Base implementation
}

- (void)draw {
    NSLog(@"Drawing a %@ shape", self.color);
}

- (NSString *)description {
    return [NSString stringWithFormat:@"Shape(color=%@, area=%.2f)", 
            self.color, [self area]];
}

@end
```

```objc
// ไฟล์: Circle.h
@interface Circle : Shape

@property (nonatomic, assign) double radius;

- (instancetype)initWithRadius:(double)radius color:(NSString *)color;

@end

// ไฟล์: Circle.m
#import "Circle.h"
#import <math.h>

@implementation Circle

- (instancetype)initWithRadius:(double)radius color:(NSString *)color {
    self = [super init];
    if (self) {
        _radius = radius;
        self.color = color;
    }
    return self;
}

// Override method จาก Shape
- (double)area {
    return M_PI * _radius * _radius;
}

- (double)perimeter {
    return 2 * M_PI * _radius;
}

- (void)draw {
    NSLog(@"Drawing a %@ circle with radius %.2f", self.color, _radius);
}

- (NSString *)description {
    return [NSString stringWithFormat:@"Circle(color=%@, radius=%.2f, area=%.2f)", 
            self.color, _radius, [self area]];
}

@end
```

```objc
// ไฟล์: Rectangle.h
@interface Rectangle : Shape

@property (nonatomic, assign) double width;
@property (nonatomic, assign) double height;

- (instancetype)initWithWidth:(double)width 
                       height:(double)height 
                        color:(NSString *)color;

@end

// ไฟล์: Rectangle.m
@implementation Rectangle

- (instancetype)initWithWidth:(double)width 
                       height:(double)height 
                        color:(NSString *)color {
    self = [super init];
    if (self) {
        _width = width;
        _height = height;
        self.color = color;
    }
    return self;
}

- (double)area {
    return _width * _height;
}

- (double)perimeter {
    return 2 * (_width + _height);
}

- (void)draw {
    NSLog(@"Drawing a %@ rectangle (%.2f x %.2f)", self.color, _width, _height);
}

- (NSString *)description {
    return [NSString stringWithFormat:@"Rectangle(%.2f x %.2f, area=%.2f)", 
            _width, _height, [self area]];
}

@end
```

```objc
// ไฟล์: Triangle.h
@interface Triangle : Shape

@property (nonatomic, assign) double sideA;
@property (nonatomic, assign) double sideB;
@property (nonatomic, assign) double sideC;

- (instancetype)initWithSides:(double)a b:(double)b c:(double)c color:(NSString *)color;

@end

// ไฟล์: Triangle.m
#import "Triangle.h"
#import <math.h>

@implementation Triangle

- (instancetype)initWithSides:(double)a b:(double)b c:(double)c color:(NSString *)color {
    self = [super init];
    if (self) {
        _sideA = a;
        _sideB = b;
        _sideC = c;
        self.color = color;
    }
    return self;
}

- (double)area {
    // Heron's formula
    double s = (_sideA + _sideB + _sideC) / 2.0;
    return sqrt(s * (s - _sideA) * (s - _sideB) * (s - _sideC));
}

- (double)perimeter {
    return _sideA + _sideB + _sideC;
}

- (void)draw {
    NSLog(@"Drawing a %@ triangle (%.2f, %.2f, %.2f)", 
          self.color, _sideA, _sideB, _sideC);
}

- (NSString *)description {
    return [NSString stringWithFormat:@"Triangle(sides=%.2f,%.2f,%.2f area=%.2f)", 
            _sideA, _sideB, _sideC, [self area]];
}

@end
```

### การใช้ Polymorphism กับ Shape Hierarchy

```objc
// main.m
#import <Foundation/Foundation.h>
#import "Shape.h"
#import "Circle.h"
#import "Rectangle.h"
#import "Triangle.h"

int main(int argc, const char * argv[]) {
    @autoreleasepool {
        // สร้าง array ของ shapes ต่างชนิด
        NSArray *shapes = @[
            [[Circle alloc] initWithRadius:5.0 color:@"red"],
            [[Rectangle alloc] initWithWidth:4.0 height:6.0 color:@"blue"],
            [[Triangle alloc] initWithSides:3.0 b:4.0 c:5.0 color:@"green"],
            [[Circle alloc] initWithRadius:3.0 color:@"yellow"],
            [[Rectangle alloc] initWithWidth:10.0 height:2.0 color:@"purple"]
        ];
        
        double totalArea = 0.0;
        double totalPerimeter = 0.0;
        
        NSLog(@"=== Shape Report ===");
        
        // Polymorphism ในการทำงาน!
        // ทุก shape ถูกเรียก method เดียวกัน แต่ทำงานต่างกัน
        for (Shape *shape in shapes) {
            [shape draw];
            NSLog(@"  Area: %.2f", [shape area]);
            NSLog(@"  Perimeter: %.2f", [shape perimeter]);
            NSLog(@"  %@", shape);
            NSLog(@"---");
            
            totalArea += [shape area];
            totalPerimeter += [shape perimeter];
        }
        
        NSLog(@"\nTotal Area: %.2f", totalArea);
        NSLog(@"Total Perimeter: %.2f", totalPerimeter);
    }
    return 0;
}
```

**ผลลัพธ์:**
```
=== Shape Report ===
Drawing a red circle with radius 5.00
  Area: 78.54
  Perimeter: 31.42
  Circle(color=red, radius=5.00, area=78.54)
---
Drawing a blue rectangle (4.00 x 6.00)
  Area: 24.00
  Perimeter: 20.00
  Rectangle(4.00 x 6.00, area=24.00)
---
Drawing a green triangle (3.00, 4.00, 5.00)
  Area: 6.00
  Perimeter: 12.00
  Triangle(sides=3.00,4.00,5.00 area=6.00)
---
...
```

---

## 14.3 id Type และ Type Checking

### id Type คืออะไร?

`id` เป็น type พิเศษใน Objective-C ที่สามารถเก็บ reference ไปยัง object ใดๆ ก็ได้ โดยไม่จำกัดชนิด

```objc
id myObject; // สามารถเก็บ object ชนิดใดก็ได้

myObject = [[NSString alloc] initWithString:@"Hello"];
NSLog(@"%@", myObject); // Hello

myObject = [[NSArray alloc] init];
NSLog(@"%@", myObject); // ()

myObject = [[NSNumber alloc] initWithInt:42];
NSLog(@"%@", myObject); // 42
```

### ความแตกต่างระหว่าง id และ NSObject*

```objc
// id: ไม่มี compile-time type checking
id obj = @"Hello";
[obj nonExistentMethod]; // Compiler ไม่แจ้ง warning!

// NSObject*: มี compile-time type checking บางส่วน
NSObject *obj2 = @"Hello";
[obj2 nonExistentMethod]; // Compiler แจ้ง warning/error!
```

### Type Checking ด้วย isKindOfClass: และ isMemberOfClass:

```objc
@interface Vehicle : NSObject
@end

@interface Car : Vehicle
@end

@interface Truck : Vehicle
@end

@implementation Vehicle @end
@implementation Car @end
@implementation Truck @end

// การตรวจสอบ type
Car *myCar = [[Car alloc] init];

// isKindOfClass: ตรวจสอบว่าเป็น instance ของคลาสนั้น หรือ subclass ของมัน
NSLog(@"isKindOfClass Car: %@", 
      [myCar isKindOfClass:[Car class]] ? @"YES" : @"NO");      // YES
NSLog(@"isKindOfClass Vehicle: %@", 
      [myCar isKindOfClass:[Vehicle class]] ? @"YES" : @"NO");  // YES
NSLog(@"isKindOfClass NSObject: %@", 
      [myCar isKindOfClass:[NSObject class]] ? @"YES" : @"NO"); // YES
NSLog(@"isKindOfClass Truck: %@", 
      [myCar isKindOfClass:[Truck class]] ? @"YES" : @"NO");    // NO

// isMemberOfClass: ตรวจสอบว่าเป็น instance ของคลาสนั้นเท่านั้น (ไม่รวม subclass)
NSLog(@"isMemberOfClass Car: %@", 
      [myCar isMemberOfClass:[Car class]] ? @"YES" : @"NO");    // YES
NSLog(@"isMemberOfClass Vehicle: %@", 
      [myCar isMemberOfClass:[Vehicle class]] ? @"YES" : @"NO"); // NO
```

### การใช้ Type Checking อย่างปลอดภัย

```objc
NSArray *objects = @[@"Hello", @42, @[@"a", @"b"], @{@"key": @"value"}];

for (id obj in objects) {
    if ([obj isKindOfClass:[NSString class]]) {
        NSString *str = (NSString *)obj;
        NSLog(@"String: %@, length: %lu", str, [str length]);
    } else if ([obj isKindOfClass:[NSNumber class]]) {
        NSNumber *num = (NSNumber *)obj;
        NSLog(@"Number: %@, value: %d", num, [num intValue]);
    } else if ([obj isKindOfClass:[NSArray class]]) {
        NSArray *arr = (NSArray *)obj;
        NSLog(@"Array with %lu elements", [arr count]);
    } else if ([obj isKindOfClass:[NSDictionary class]]) {
        NSDictionary *dict = (NSDictionary *)obj;
        NSLog(@"Dictionary with %lu keys", [dict count]);
    }
}
```

### respondsToSelector: สำหรับ Duck Typing

```objc
// ตรวจสอบว่า object มี method นี้ก่อนเรียก
id unknownObject = getSomeObject(); // ไม่รู้ว่าเป็น type อะไร

if ([unknownObject respondsToSelector:@selector(processData)]) {
    [unknownObject processData];
}

if ([unknownObject respondsToSelector:@selector(getName)]) {
    NSString *name = [unknownObject getName];
    NSLog(@"Name: %@", name);
}
```

---

## 14.4 instancetype

### ปัญหาของ id ใน init Methods

ก่อน Objective-C จะมี `instancetype` เราต้องใช้ `id` ใน factory methods และ init methods ซึ่งทำให้ compiler ไม่สามารถตรวจสอบ type ได้

```objc
// แบบเก่า - ใช้ id
@interface Animal : NSObject
+ (id)createAnimal; // ไม่มี type info
- (id)init;         // ไม่มี type info
@end

// ปัญหา: compiler ไม่รู้ว่า return type คืออะไร
Animal *a = [Animal createAnimal];
// [a specificAnimalMethod]; // อาจเกิด warning
```

### การใช้ instancetype

`instancetype` บอก compiler ว่า method จะ return instance ของคลาสที่เรียก method นั้น

```objc
@interface Animal : NSObject

@property (nonatomic, copy) NSString *name;
@property (nonatomic, assign) NSInteger age;

+ (instancetype)animalWithName:(NSString *)name age:(NSInteger)age;
- (instancetype)initWithName:(NSString *)name age:(NSInteger)age;

@end

@implementation Animal

+ (instancetype)animalWithName:(NSString *)name age:(NSInteger)age {
    return [[self alloc] initWithName:name age:age];
}

- (instancetype)initWithName:(NSString *)name age:(NSInteger)age {
    self = [super init];
    if (self) {
        _name = [name copy];
        _age = age;
    }
    return self;
}

@end

@interface Dog : Animal
- (void)bark;
@end

@implementation Dog
- (void)bark {
    NSLog(@"%@ says: Woof!", self.name);
}
@end

// การใช้งาน - instancetype ช่วย type checking
Dog *dog = [Dog animalWithName:@"Rex" age:3];
[dog bark]; // Compiler รู้ว่า dog เป็น Dog จึงไม่มี warning
```

### instancetype กับ Inheritance

```objc
@interface Person : NSObject

@property (nonatomic, copy) NSString *firstName;
@property (nonatomic, copy) NSString *lastName;

+ (instancetype)personWithFirstName:(NSString *)first lastName:(NSString *)last;

@end

@implementation Person

+ (instancetype)personWithFirstName:(NSString *)first lastName:(NSString *)last {
    Person *person = [[self alloc] init]; // ใช้ self แทน Person เพื่อรองรับ subclass
    person.firstName = first;
    person.lastName = last;
    return person;
}

@end

@interface Employee : Person

@property (nonatomic, copy) NSString *department;

@end

@implementation Employee
@end

// instancetype ทำให้ทำงานได้ถูกต้อง
Employee *emp = [Employee personWithFirstName:@"John" lastName:@"Doe"];
emp.department = @"Engineering"; // ทำได้! compiler รู้ว่าเป็น Employee
```

---

## 14.5 Duck Typing ใน Objective-C

### แนวคิด Duck Typing

"ถ้ามันเดินเหมือนเป็ด และร้องเหมือนเป็ด มันก็คือเป็ด"

Duck typing คือแนวคิดที่ไม่สนใจ type ของ object แต่สนใจว่า object นั้นสามารถทำสิ่งที่เราต้องการได้หรือไม่

```objc
// ตัวอย่าง Duck Typing
@protocol Drawable
- (void)draw;
@end

// ไม่จำเป็นต้อง conform to protocol
@interface Circle : NSObject
- (void)draw;
@end

@interface Square : NSObject
- (void)draw;
@end

@interface TextLabel : NSObject
- (void)draw;
@end

@implementation Circle
- (void)draw { NSLog(@"Drawing circle"); }
@end

@implementation Square
- (void)draw { NSLog(@"Drawing square"); }
@end

@implementation TextLabel
- (void)draw { NSLog(@"Drawing text label"); }
@end

// Duck typing: ทุก object ที่มี -draw method สามารถถูกเรียกได้
void drawAll(NSArray *objects) {
    for (id obj in objects) {
        if ([obj respondsToSelector:@selector(draw)]) {
            [obj draw]; // เรียก draw โดยไม่สนใจว่า obj เป็น type ใด
        }
    }
}

int main(int argc, const char * argv[]) {
    @autoreleasepool {
        NSArray *drawables = @[
            [[Circle alloc] init],
            [[Square alloc] init],
            [[TextLabel alloc] init],
            @"NotDrawable" // NSString ไม่มี draw method
        ];
        
        drawAll(drawables);
        // Drawing circle
        // Drawing square
        // Drawing text label
        // (NSString ถูกข้ามไป)
    }
    return 0;
}
```

---

## 14.6 Message Sending (objc_msgSend)

### กลไกเบื้องหลัง

ทุก method call ใน Objective-C จะถูกแปลงเป็นการเรียก `objc_msgSend` ที่ runtime

```objc
// Code ที่เราเขียน
[myObject doSomethingWith:argument];

// สิ่งที่ compiler แปลงเป็น
objc_msgSend(myObject, @selector(doSomethingWith:), argument);
```

### Message Sending Process

```
1. Runtime รับ message (receiver + selector + arguments)
   ↓
2. ดึง class ของ receiver จาก isa pointer
   ↓  
3. ค้นหา method ใน method table ของ class
   ↓
4. ถ้าไม่พบ ให้ไต่ขึ้น superclass chain
   ↓
5. ถ้าพบ execute method
   ถ้าไม่พบ เรียก doesNotRecognizeSelector: (throws NSException)
```

### ตัวอย่างการเข้าใจ Message Sending

```objc
#import <objc/runtime.h>
#import <objc/message.h>

@interface Calculator : NSObject
- (double)add:(double)a to:(double)b;
- (double)multiply:(double)a by:(double)b;
@end

@implementation Calculator
- (double)add:(double)a to:(double)b {
    return a + b;
}
- (double)multiply:(double)a by:(double)b {
    return a * b;
}
@end

int main(int argc, const char * argv[]) {
    @autoreleasepool {
        Calculator *calc = [[Calculator alloc] init];
        
        // วิธีปกติ
        double result1 = [calc add:3.0 to:4.0];
        NSLog(@"3 + 4 = %.1f", result1); // 7.0
        
        // ผ่าน performSelector (dynamic message send)
        SEL addSelector = @selector(add:to:);
        // Note: สำหรับ method ที่ return non-object ต้องระวัง
        // นี้เป็นเพียงตัวอย่างเชิงแนวคิด
        
        // ตรวจสอบว่า method มีอยู่
        if ([calc respondsToSelector:addSelector]) {
            NSLog(@"Calculator has add:to: method");
        }
        
        // ดู method list ด้วย runtime API
        unsigned int methodCount;
        Method *methods = class_copyMethodList([Calculator class], &methodCount);
        
        NSLog(@"Methods in Calculator:");
        for (unsigned int i = 0; i < methodCount; i++) {
            NSLog(@"  - %@", NSStringFromSelector(method_getName(methods[i])));
        }
        
        free(methods);
    }
    return 0;
}
```

---

## 14.7 Runtime Method Lookup

### Method Cache

Objective-C runtime ใช้ method cache เพื่อเพิ่มประสิทธิภาพ เมื่อ method ถูกเรียกครั้งแรก runtime จะ cache ผลการค้นหาไว้

```objc
// แนวคิดของ Method Cache (pseudo-code)
struct objc_class {
    struct objc_class *superclass;
    cache_t cache; // Method cache
    method_list_t *methods;
    // ...
};

// การค้นหา method:
// 1. ตรวจสอบ cache ก่อน (เร็วมาก)
// 2. ถ้าไม่ใน cache ค้นหาใน method list
// 3. ถ้าไม่พบ ขึ้นไปที่ superclass
// 4. Cache ผลลัพธ์ไว้
```

### Method Forwarding

เมื่อ method ไม่พบ runtime จะให้โอกาสแก้ไขก่อน throw exception

```objc
@interface SmartProxy : NSObject

@property (nonatomic, strong) id target;

- (instancetype)initWithTarget:(id)target;

@end

@implementation SmartProxy

- (instancetype)initWithTarget:(id)target {
    self = [super init];
    if (self) {
        _target = target;
    }
    return self;
}

// Step 1: ลอง resolve method dynamically
+ (BOOL)resolveInstanceMethod:(SEL)sel {
    NSLog(@"resolveInstanceMethod: %@", NSStringFromSelector(sel));
    return [super resolveInstanceMethod:sel];
}

// Step 2: ลอง forward ไปยัง object อื่น
- (id)forwardingTargetForSelector:(SEL)aSelector {
    NSLog(@"forwardingTargetForSelector: %@", NSStringFromSelector(aSelector));
    if ([_target respondsToSelector:aSelector]) {
        return _target; // Forward ไปยัง target
    }
    return [super forwardingTargetForSelector:aSelector];
}

// Step 3: Full forwarding
- (NSMethodSignature *)methodSignatureForSelector:(SEL)aSelector {
    NSMethodSignature *sig = [_target methodSignatureForSelector:aSelector];
    if (sig) return sig;
    return [super methodSignatureForSelector:aSelector];
}

- (void)forwardInvocation:(NSInvocation *)anInvocation {
    if ([_target respondsToSelector:[anInvocation selector]]) {
        [anInvocation invokeWithTarget:_target];
    } else {
        [super forwardInvocation:anInvocation];
    }
}

@end

// การใช้งาน
NSString *str = @"Hello World";
SmartProxy *proxy = [[SmartProxy alloc] initWithTarget:str];

// เมื่อ proxy ไม่มี method นี้ จะ forward ไปยัง str
NSLog(@"%@", [proxy description]); // Forward ไปยัง NSString
NSLog(@"%lu", (unsigned long)[proxy length]); // ใช้ผ่าน proxy
```

---

## 14.8 ตัวอย่าง Polymorphism เชิงปฏิบัติ: ระบบชำระเงิน

```objc
// ไฟล์: PaymentMethod.h
@interface PaymentMethod : NSObject

@property (nonatomic, copy) NSString *name;
@property (nonatomic, assign) double balance;

- (instancetype)initWithName:(NSString *)name balance:(double)balance;
- (BOOL)canProcessAmount:(double)amount;
- (BOOL)processPayment:(double)amount;
- (NSString *)paymentDetails;

@end

// ไฟล์: PaymentMethod.m
@implementation PaymentMethod

- (instancetype)initWithName:(NSString *)name balance:(double)balance {
    self = [super init];
    if (self) {
        _name = [name copy];
        _balance = balance;
    }
    return self;
}

- (BOOL)canProcessAmount:(double)amount {
    return _balance >= amount;
}

- (BOOL)processPayment:(double)amount {
    if ([self canProcessAmount:amount]) {
        _balance -= amount;
        return YES;
    }
    return NO;
}

- (NSString *)paymentDetails {
    return [NSString stringWithFormat:@"%@ (Balance: ฿%.2f)", _name, _balance];
}

@end
```

```objc
// ไฟล์: CreditCard.h
@interface CreditCard : PaymentMethod

@property (nonatomic, assign) double creditLimit;
@property (nonatomic, assign) double currentDebt;
@property (nonatomic, copy) NSString *cardNumber;

- (instancetype)initWithName:(NSString *)name 
                  creditLimit:(double)limit
                   cardNumber:(NSString *)number;

@end

// ไฟล์: CreditCard.m
@implementation CreditCard

- (instancetype)initWithName:(NSString *)name 
                  creditLimit:(double)limit
                   cardNumber:(NSString *)number {
    self = [super initWithName:name balance:limit];
    if (self) {
        _creditLimit = limit;
        _currentDebt = 0;
        _cardNumber = [number copy];
    }
    return self;
}

- (BOOL)canProcessAmount:(double)amount {
    // Credit card: available credit = limit - current debt
    double availableCredit = _creditLimit - _currentDebt;
    return availableCredit >= amount;
}

- (BOOL)processPayment:(double)amount {
    if ([self canProcessAmount:amount]) {
        _currentDebt += amount;
        return YES;
    }
    return NO;
}

- (NSString *)paymentDetails {
    return [NSString stringWithFormat:@"Credit Card ***%@ (Limit: ฿%.2f, Debt: ฿%.2f)", 
            [_cardNumber substringFromIndex:[_cardNumber length] - 4],
            _creditLimit, _currentDebt];
}

@end
```

```objc
// ไฟล์: BankTransfer.h
@interface BankTransfer : PaymentMethod

@property (nonatomic, copy) NSString *accountNumber;
@property (nonatomic, copy) NSString *bankName;
@property (nonatomic, assign) double dailyLimit;
@property (nonatomic, assign) double todayTransferred;

- (instancetype)initWithName:(NSString *)name
                     balance:(double)balance
               accountNumber:(NSString *)account
                    bankName:(NSString *)bank
                  dailyLimit:(double)limit;

@end

// ไฟล์: BankTransfer.m
@implementation BankTransfer

- (instancetype)initWithName:(NSString *)name
                     balance:(double)balance
               accountNumber:(NSString *)account
                    bankName:(NSString *)bank
                  dailyLimit:(double)limit {
    self = [super initWithName:name balance:balance];
    if (self) {
        _accountNumber = [account copy];
        _bankName = [bank copy];
        _dailyLimit = limit;
        _todayTransferred = 0;
    }
    return self;
}

- (BOOL)canProcessAmount:(double)amount {
    BOOL hasBalance = [super canProcessAmount:amount];
    BOOL withinDailyLimit = (_todayTransferred + amount) <= _dailyLimit;
    return hasBalance && withinDailyLimit;
}

- (BOOL)processPayment:(double)amount {
    if ([self canProcessAmount:amount]) {
        self.balance -= amount;
        _todayTransferred += amount;
        return YES;
    }
    return NO;
}

- (NSString *)paymentDetails {
    return [NSString stringWithFormat:@"Bank Transfer %@ (%@, Balance: ฿%.2f, Daily remaining: ฿%.2f)", 
            _bankName, _accountNumber, self.balance, 
            _dailyLimit - _todayTransferred];
}

@end
```

```objc
// ไฟล์: DigitalWallet.h
@interface DigitalWallet : PaymentMethod

@property (nonatomic, copy) NSString *walletId;
@property (nonatomic, assign) double cashbackRate;
@property (nonatomic, assign) double totalCashback;

- (instancetype)initWithName:(NSString *)name
                     balance:(double)balance
                    walletId:(NSString *)walletId
                cashbackRate:(double)rate;

@end

// ไฟล์: DigitalWallet.m
@implementation DigitalWallet

- (instancetype)initWithName:(NSString *)name
                     balance:(double)balance
                    walletId:(NSString *)walletId
                cashbackRate:(double)rate {
    self = [super initWithName:name balance:balance];
    if (self) {
        _walletId = [walletId copy];
        _cashbackRate = rate;
        _totalCashback = 0;
    }
    return self;
}

- (BOOL)processPayment:(double)amount {
    if ([super processPayment:amount]) {
        double cashback = amount * _cashbackRate;
        _totalCashback += cashback;
        self.balance += cashback; // เพิ่ม cashback กลับเข้า wallet
        NSLog(@"💰 Cashback earned: ฿%.2f", cashback);
        return YES;
    }
    return NO;
}

- (NSString *)paymentDetails {
    return [NSString stringWithFormat:@"Digital Wallet %@ (Balance: ฿%.2f, Total cashback: ฿%.2f)", 
            _walletId, self.balance, _totalCashback];
}

@end
```

```objc
// ไฟล์: PaymentProcessor.h
@interface PaymentProcessor : NSObject

- (void)processTransaction:(double)amount 
             usingPayments:(NSArray<PaymentMethod *> *)payments;

- (PaymentMethod *)findBestPaymentFor:(double)amount 
                             from:(NSArray<PaymentMethod *> *)payments;

@end

// ไฟล์: PaymentProcessor.m
@implementation PaymentProcessor

- (void)processTransaction:(double)amount 
             usingPayments:(NSArray<PaymentMethod *> *)payments {
    
    NSLog(@"\n=== Processing Payment of ฿%.2f ===", amount);
    
    PaymentMethod *bestPayment = [self findBestPaymentFor:amount from:payments];
    
    if (bestPayment) {
        NSLog(@"Using: %@", [bestPayment paymentDetails]);
        BOOL success = [bestPayment processPayment:amount];
        
        if (success) {
            NSLog(@"✅ Payment successful!");
            NSLog(@"After payment: %@", [bestPayment paymentDetails]);
        } else {
            NSLog(@"❌ Payment failed!");
        }
    } else {
        NSLog(@"❌ No payment method available for this amount");
    }
}

- (PaymentMethod *)findBestPaymentFor:(double)amount 
                             from:(NSArray<PaymentMethod *> *)payments {
    // หา payment method ที่ดีที่สุด (ให้ priority ให้ Digital Wallet ก่อน)
    for (PaymentMethod *payment in payments) {
        if ([payment isKindOfClass:[DigitalWallet class]] && 
            [payment canProcessAmount:amount]) {
            return payment;
        }
    }
    
    // จากนั้นลอง payment methods อื่นๆ
    for (PaymentMethod *payment in payments) {
        if ([payment canProcessAmount:amount]) {
            return payment;
        }
    }
    
    return nil;
}

@end

// main.m - ทดสอบ Polymorphism ของระบบชำระเงิน
int main(int argc, const char * argv[]) {
    @autoreleasepool {
        // สร้าง payment methods ต่างชนิด
        NSArray *paymentMethods = @[
            [[CreditCard alloc] initWithName:@"Visa Gold" 
                                 creditLimit:50000.0 
                                  cardNumber:@"4111111111111234"],
            [[BankTransfer alloc] initWithName:@"SCB" 
                                      balance:30000.0 
                                accountNumber:@"xxx-x-xxxxx-1" 
                                     bankName:@"SCB" 
                                   dailyLimit:100000.0],
            [[DigitalWallet alloc] initWithName:@"TrueMoney" 
                                       balance:5000.0 
                                      walletId:@"TM-001" 
                                   cashbackRate:0.01]
        ];
        
        // แสดงข้อมูล payment methods ทั้งหมด
        NSLog(@"=== Available Payment Methods ===");
        for (PaymentMethod *pm in paymentMethods) {
            NSLog(@"• %@", [pm paymentDetails]); // Polymorphism!
        }
        
        // ดำเนินการชำระเงิน
        PaymentProcessor *processor = [[PaymentProcessor alloc] init];
        
        [processor processTransaction:500.0 usingPayments:paymentMethods];
        [processor processTransaction:6000.0 usingPayments:paymentMethods];
        [processor processTransaction:200000.0 usingPayments:paymentMethods];
    }
    return 0;
}
```

---

## 14.9 Polymorphism กับ Collections

```objc
// การจัดการ collection ของ objects ที่มี type ต่างกัน

@interface Printable : NSObject
- (NSString *)printableString;
@end

@interface Invoice : Printable
@property (nonatomic, assign) double amount;
@property (nonatomic, copy) NSString *customer;
@end

@interface Receipt : Printable
@property (nonatomic, assign) double paid;
@property (nonatomic, strong) NSDate *date;
@end

@interface Report : Printable
@property (nonatomic, copy) NSString *title;
@property (nonatomic, copy) NSString *content;
@end

@implementation Invoice
- (NSString *)printableString {
    return [NSString stringWithFormat:@"[INVOICE] Customer: %@, Amount: ฿%.2f", 
            self.customer, self.amount];
}
@end

@implementation Receipt
- (NSString *)printableString {
    NSDateFormatter *fmt = [[NSDateFormatter alloc] init];
    fmt.dateStyle = NSDateFormatterShortStyle;
    return [NSString stringWithFormat:@"[RECEIPT] Paid: ฿%.2f on %@", 
            self.paid, [fmt stringFromDate:self.date]];
}
@end

@implementation Report
- (NSString *)printableString {
    return [NSString stringWithFormat:@"[REPORT] %@: %@", 
            self.title, self.content];
}
@end

void printAll(NSArray<Printable *> *items) {
    NSLog(@"\n=== Print Queue ===");
    for (Printable *item in items) {
        NSLog(@"%@", [item printableString]); // Polymorphism!
    }
}
```

---

## 14.10 สรุปหลักการ Polymorphism

### ประโยชน์ของ Polymorphism

1. **Code Reuse**: เขียน code ที่ทำงานกับ base type ได้ โดยไม่ต้องรู้ว่าเป็น subtype ใด
2. **Extensibility**: เพิ่ม subclass ใหม่ได้โดยไม่ต้องแก้ code ที่มีอยู่
3. **Abstraction**: ซ่อนรายละเอียดการ implement จากผู้ใช้
4. **Flexibility**: เปลี่ยน behavior ของ system ได้โดยเปลี่ยนเพียง object ที่ inject เข้าไป

### Design Principles ที่เกี่ยวข้อง

```objc
// Liskov Substitution Principle (LSP)
// Subclass ต้องสามารถใช้แทน superclass ได้โดยไม่ทำให้ program ผิดพลาด

void drawShape(Shape *shape) {
    // ต้องทำงานได้กับทุก subclass ของ Shape
    [shape draw];
    NSLog(@"Area: %.2f", [shape area]);
}

// Open/Closed Principle
// เปิดสำหรับการ extend แต่ปิดสำหรับการแก้ไข
// เพิ่ม Pentagon class ได้โดยไม่ต้องแก้ drawShape
@interface Pentagon : Shape
@end
```

---

## แบบฝึกหัด

### ระดับเริ่มต้น

**แบบฝึกหัดที่ 1**: สร้าง class hierarchy สำหรับ Animal
- Base class: `Animal` มี method `makeSound`, `move`, `eat`
- Subclasses: `Dog`, `Cat`, `Bird`, `Fish`
- แต่ละ subclass override method ตามลักษณะของสัตว์
- สร้าง array ของ Animal และเรียก method ผ่าน polymorphism

**แบบฝึกหัดที่ 2**: Employee Hierarchy
```objc
// สร้าง class hierarchy ต่อไปนี้:
// Employee (base)
//   ├── FullTimeEmployee (เงินเดือน + โบนัส)
//   ├── PartTimeEmployee (ค่าแรงรายชั่วโมง)
//   └── Contractor (ค่าจ้างต่อ project)

// แต่ละคลาสมี:
// - (double)calculateSalary;
// - (NSString *)employeeInfo;

// สร้าง NSArray<Employee *> และคำนวณ total payroll
```

**แบบฝึกหัดที่ 3**: ใช้ `isKindOfClass:` และ `respondsToSelector:`
```objc
// สร้าง NSArray ที่มี objects หลายชนิด
// วน loop และตรวจสอบ type ก่อนเรียก method เฉพาะของแต่ละ type
NSArray *mixedObjects = @[
    @"Hello",
    @42,
    [[NSDate alloc] init],
    @[@1, @2, @3],
    @{@"key": @"value"}
];
```

### ระดับกลาง

**แบบฝึกหัดที่ 4**: Notification System
```objc
// สร้าง Notification types:
// - EmailNotification (ส่ง email)
// - SMSNotification (ส่ง SMS)  
// - PushNotification (ส่ง push notification)
// - LineNotification (ส่งผ่าน Line)

// Base class: Notification
// - (void)send;
// - (NSString *)notificationInfo;

// สร้าง NotificationService ที่รับ NSArray<Notification *>
// และส่ง notification ทุกชนิดผ่าน polymorphism
```

**แบบฝึกหัดที่ 5**: File System Operations
```objc
// FileSystemItem (base)
//   ├── File (มี content, size)
//   └── Directory (มี children, recursive size)

// Methods:
// - (NSInteger)size;
// - (void)print:(NSInteger)indent;
// - (NSString *)name;
```

**แบบฝึกหัดที่ 6**: Tax Calculator
```objc
// TaxCalculator (base)
//   ├── PersonalTaxCalculator
//   ├── CorporateTaxCalculator
//   └── VATCalculator

// - (double)calculateTax:(double)income;
// - (NSString *)taxType;
// - (void)printTaxReport:(double)income;
```

### ระดับสูง

**แบบฝึกหัดที่ 7**: Rendering Engine
```objc
// สร้าง rendering engine ที่รองรับหลาย format
// Renderer (base)
//   ├── HTMLRenderer
//   ├── PDFRenderer
//   └── MarkdownRenderer

// Content (base)
//   ├── Heading
//   ├── Paragraph
//   └── List

// Renderer รับ array ของ Content และ render เป็น format ที่ต้องการ
```

**แบบฝึกหัดที่ 8**: Plugin System
```objc
// สร้าง plugin system ที่ extensible
// Plugin (base)
//   ├── LogPlugin
//   ├── SecurityPlugin
//   └── CachePlugin

// PluginManager รับ NSArray<Plugin *>
// เรียก execute: บน plugin ทุกตัวตาม event ที่เกิดขึ้น
```

**แบบฝึกหัดที่ 9**: Report Generator
```objc
// สร้าง Report Generator ที่รองรับ data sources ต่างๆ
// DataSource (base)
//   ├── DatabaseSource
//   ├── CSVSource
//   └── JSONSource

// ReportFormatter (base)
//   ├── TableFormatter
//   ├── ChartFormatter
//   └── SummaryFormatter

// ใช้ polymorphism ทั้ง source และ formatter
```

**แบบฝึกหัดที่ 10**: Game Entity System
```objc
// GameEntity (base)
//   ├── Player
//   ├── Enemy (สร้าง subclasses หลายชนิด)
//   ├── Projectile
//   └── PowerUp

// ทุก entity มี:
// - (void)update:(double)deltaTime;
// - (CGRect)boundingBox;
// - (void)onCollisionWith:(GameEntity *)other;

// GameWorld รับ NSMutableArray<GameEntity *>
// และ update/render ทุก entity ผ่าน polymorphism
```

**แบบฝึกหัดที่ 11 (Challenge)**: Builder Pattern with Polymorphism
```objc
// สร้าง Document Builder ที่รองรับหลาย document types
// DocumentBuilder (abstract)
//   ├── ResumeBuilder
//   ├── ContractBuilder
//   └── ReportBuilder

// Director ใช้ DocumentBuilder interface
// สร้าง documents ต่างชนิดผ่าน polymorphism
// แต่ละ Builder build sections ต่างกัน
```

---

## สรุปบทเรียน

ในบทนี้เราได้เรียนรู้:
- **Dynamic Dispatch**: กลไกที่ทำให้ Objective-C มี polymorphism
- **Method Overriding**: วิธีที่ subclass เปลี่ยน behavior ของ superclass
- **id Type**: type ที่ยืดหยุ่นสำหรับเก็บ object ใดๆ
- **instancetype**: การระบุ return type ที่ถูกต้องสำหรับ factory methods
- **Duck Typing**: การตรวจสอบ capability แทนที่จะตรวจสอบ type
- **objc_msgSend**: กลไกเบื้องหลังของการส่ง message
- **Practical Examples**: Shape hierarchy และ Payment system

> **หมายเหตุ**: Polymorphism ทำงานได้ดีที่สุดเมื่อออกแบบ class hierarchy ที่ดี ควรปฏิบัติตาม Liskov Substitution Principle และให้แต่ละ class มีหน้าที่เดียวชัดเจน
