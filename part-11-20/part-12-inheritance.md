# Part 12: Inheritance (การสืบทอด) ใน Objective-C

## บทนำ

Inheritance หรือการสืบทอดเป็นหลักการสำคัญของ OOP ที่ช่วยให้ class สามารถรับเอาคุณสมบัติและพฤติกรรมของ class อื่นมาใช้งานได้ Objective-C รองรับ **single inheritance** เท่านั้น (แต่ใช้ Protocols เพื่อจำลอง multiple inheritance)

---

## 1. Single Inheritance ใน Objective-C

### ไวยากรณ์พื้นฐาน

```objc
// รูปแบบ: @interface SubClass : SuperClass
@interface SubClass : SuperClass

// เพิ่ม properties และ methods ใหม่
// Override methods จาก SuperClass

@end
```

### ลำดับชั้นของ Class

```
NSObject (root class)
    └── Animal
            ├── Dog
            │     ├── GoldenRetriever
            │     └── Poodle
            ├── Cat
            └── Bird
                    ├── Parrot
                    └── Eagle
```

```objc
// Animal.h - Base class
#import <Foundation/Foundation.h>

@interface Animal : NSObject

@property (nonatomic, copy)   NSString *name;
@property (nonatomic, assign) NSInteger age;
@property (nonatomic, copy)   NSString *species;

- (instancetype)initWithName:(NSString *)name
                         age:(NSInteger)age
                     species:(NSString *)species NS_DESIGNATED_INITIALIZER;

- (void)makeSound;
- (void)eat:(NSString *)food;
- (void)sleep;
- (NSString *)basicInfo;

@end
```

```objc
// Animal.m
#import "Animal.h"

@implementation Animal

- (instancetype)initWithName:(NSString *)name
                         age:(NSInteger)age
                     species:(NSString *)species {
    self = [super init];
    if (self) {
        _name    = [name    copy];
        _age     = age;
        _species = [species copy];
    }
    return self;
}

- (instancetype)init {
    return [self initWithName:@"Unknown" age:0 species:@"Unknown"];
}

- (void)makeSound {
    NSLog(@"%@ (%@) ส่งเสียง...", _name, _species);
}

- (void)eat:(NSString *)food {
    NSLog(@"%@ กำลังกิน %@", _name, food);
}

- (void)sleep {
    NSLog(@"%@ กำลังนอนหลับ 💤", _name);
}

- (NSString *)basicInfo {
    return [NSString stringWithFormat:@"%@ | สปีชีส์: %@ | อายุ: %ld ปี",
            _name, _species, (long)_age];
}

- (NSString *)description {
    return [NSString stringWithFormat:@"<%@: %@>", [self class], [self basicInfo]];
}

@end
```

---

## 2. @interface SubClass : SuperClass

### การสร้าง Subclass

```objc
// Dog.h - Subclass ของ Animal
#import "Animal.h"

@interface Dog : Animal

// Properties เพิ่มเติม (ไม่มีใน Animal)
@property (nonatomic, copy)   NSString *breed;        // สายพันธุ์
@property (nonatomic, assign) BOOL      isVaccinated;  // ฉีดวัคซีนแล้วหรือยัง
@property (nonatomic, strong) NSString *owner;

// Designated initializer ของ Dog
- (instancetype)initWithName:(NSString *)name
                         age:(NSInteger)age
                       breed:(NSString *)breed
                       owner:(NSString *)owner NS_DESIGNATED_INITIALIZER;

// Methods เพิ่มเติม
- (void)fetch;
- (void)sit;
- (void)shake;
- (void)bark;
- (NSString *)dogInfo;

@end
```

```objc
// Dog.m
#import "Dog.h"

@implementation Dog

// Designated initializer ของ Dog
- (instancetype)initWithName:(NSString *)name
                         age:(NSInteger)age
                       breed:(NSString *)breed
                       owner:(NSString *)owner {
    // เรียก designated initializer ของ superclass (Animal)
    self = [super initWithName:name age:age species:@"สุนัข"];
    if (self) {
        _breed        = [breed copy];
        _owner        = [owner copy];
        _isVaccinated = NO;
    }
    return self;
}

// Override init - ต้องเรียก designated initializer ของ class นี้
- (instancetype)init {
    return [self initWithName:@"Unnamed Dog" age:0 breed:@"Mixed" owner:@"Unknown"];
}

// สามารถ override methods ของ superclass ได้
- (void)makeSound {
    // เรียก superclass method ก่อน (optional)
    // [super makeSound];
    [self bark];
}

- (void)bark {
    NSLog(@"%@ (%@): โฮ้ง! โฮ้ง! 🐕", self.name, _breed);
}

- (void)fetch {
    NSLog(@"%@ วิ่งไปเก็บลูกบอล! 🎾", self.name);
}

- (void)sit {
    NSLog(@"%@ นั่งลง 🐾", self.name);
}

- (void)shake {
    NSLog(@"%@ ยื่นมือให้จับ 🤝", self.name);
}

- (NSString *)dogInfo {
    return [NSString stringWithFormat:
            @"%@\nสายพันธุ์: %@ | เจ้าของ: %@ | วัคซีน: %@",
            [self basicInfo], _breed, _owner, _isVaccinated ? @"ฉีดแล้ว" : @"ยังไม่ฉีด"];
}

- (NSString *)description {
    return [NSString stringWithFormat:@"<Dog: %@>", [self dogInfo]];
}

@end
```

```objc
// Cat.h - อีก subclass ของ Animal
#import "Animal.h"

@interface Cat : Animal

@property (nonatomic, copy)   NSString *furColor;     // สีขน
@property (nonatomic, assign) BOOL      isIndoor;      // แมวบ้านหรือแมวจร
@property (nonatomic, assign) NSInteger lives;         // จำนวนชีวิต (มีมุก 9 ชีวิต)

- (instancetype)initWithName:(NSString *)name
                         age:(NSInteger)age
                    furColor:(NSString *)furColor
                    isIndoor:(BOOL)isIndoor NS_DESIGNATED_INITIALIZER;

- (void)meow;
- (void)purr;
- (void)hiss;
- (void)climb;

@end
```

```objc
// Cat.m
#import "Cat.h"

@implementation Cat

- (instancetype)initWithName:(NSString *)name
                         age:(NSInteger)age
                    furColor:(NSString *)furColor
                    isIndoor:(BOOL)isIndoor {
    self = [super initWithName:name age:age species:@"แมว"];
    if (self) {
        _furColor = [furColor copy];
        _isIndoor = isIndoor;
        _lives = 9;
    }
    return self;
}

- (instancetype)init {
    return [self initWithName:@"Unnamed Cat" age:0 furColor:@"orange" isIndoor:YES];
}

- (void)makeSound {
    [self meow];
}

- (void)meow {
    NSLog(@"%@: เหมียว~ 🐱", self.name);
}

- (void)purr {
    NSLog(@"%@: purrrr... 😸", self.name);
}

- (void)hiss {
    NSLog(@"%@: ฮือออ! 😾", self.name);
}

- (void)climb {
    NSLog(@"%@ ปีนขึ้นไปบนต้นไม้ 🌲", self.name);
}

- (NSString *)description {
    return [NSString stringWithFormat:
            @"<Cat: %@ | ขน: %@ | %@>",
            [self basicInfo], _furColor, _isIndoor ? @"แมวบ้าน" : @"แมวจร"];
}

@end
```

---

## 3. Overriding Methods

### การ Override Methods จาก Superclass

```objc
// Bird.h
#import "Animal.h"

@interface Bird : Animal

@property (nonatomic, copy)   NSString *wingColor;
@property (nonatomic, assign) BOOL      canFly;
@property (nonatomic, assign) double    wingspan;  // ความกว้างปีก (เมตร)

- (instancetype)initWithName:(NSString *)name
                         age:(NSInteger)age
                   wingColor:(NSString *)wingColor
                      canFly:(BOOL)canFly
                    wingspan:(double)wingspan NS_DESIGNATED_INITIALIZER;

- (void)sing;
- (void)fly;
- (void)land;

@end
```

```objc
// Bird.m
#import "Bird.h"

@implementation Bird

- (instancetype)initWithName:(NSString *)name
                         age:(NSInteger)age
                   wingColor:(NSString *)wingColor
                      canFly:(BOOL)canFly
                    wingspan:(double)wingspan {
    self = [super initWithName:name age:age species:@"นก"];
    if (self) {
        _wingColor = [wingColor copy];
        _canFly    = canFly;
        _wingspan  = wingspan;
    }
    return self;
}

- (instancetype)init {
    return [self initWithName:@"Unnamed Bird" age:0
                    wingColor:@"brown" canFly:YES wingspan:0.3];
}

// Override makeSound จาก Animal
- (void)makeSound {
    [self sing];
}

// Override eat: จาก Animal - เพิ่ม behavior
- (void)eat:(NSString *)food {
    [super eat:food];  // เรียก Animal's eat ก่อน
    NSLog(@"  (%@ จิกกิน %@)", self.name, food);
}

- (void)sing {
    NSLog(@"%@: จิ้กๆ จ้อๆ 🎵", self.name);
}

- (void)fly {
    if (_canFly) {
        NSLog(@"%@ บินขึ้นสู่ท้องฟ้า ✈️ (ปีกกว้าง %.1f ม.)", self.name, _wingspan);
    } else {
        NSLog(@"%@ บินไม่ได้ แต่วิ่งเร็ว! 🏃", self.name);
    }
}

- (void)land {
    NSLog(@"%@ ร่อนลงจอด 🛬", self.name);
}

@end
```

### Subclass ของ Subclass (Deep Inheritance)

```objc
// Parrot.h - Dog <- Animal <- NSObject pattern
#import "Bird.h"

@interface Parrot : Bird

@property (nonatomic, copy) NSString *favoriteWord;  // คำที่ชอบพูด
@property (nonatomic, assign) NSInteger vocabularySize; // จำนวนคำที่รู้

- (instancetype)initWithName:(NSString *)name
                         age:(NSInteger)age
                 favoriteWord:(NSString *)favoriteWord NS_DESIGNATED_INITIALIZER;

- (void)speak;
- (void)learn:(NSString *)newWord;

@end

@implementation Parrot

- (instancetype)initWithName:(NSString *)name
                         age:(NSInteger)age
                 favoriteWord:(NSString *)favoriteWord {
    // เรียก designated initializer ของ Bird
    self = [super initWithName:name age:age
                     wingColor:@"green" canFly:YES wingspan:0.4];
    if (self) {
        _favoriteWord   = [favoriteWord copy];
        _vocabularySize = 1;
        self.species = @"นกแก้ว";  // Override species ผ่าน setter
    }
    return self;
}

- (instancetype)init {
    return [self initWithName:@"Polly" age:1 favoriteWord:@"สวัสดี"];
}

// Override makeSound - สายโซ่ลึกถึง 3 ระดับ
- (void)makeSound {
    [self speak];
}

// Override sing จาก Bird
- (void)sing {
    [super sing];  // เรียก Bird's sing ก่อน
    NSLog(@"%@: %@! %@! %@!", self.name, _favoriteWord, _favoriteWord, _favoriteWord);
}

- (void)speak {
    NSLog(@"%@: '%@' (รู้ %ld คำ)", self.name, _favoriteWord, (long)_vocabularySize);
}

- (void)learn:(NSString *)newWord {
    _vocabularySize++;
    _favoriteWord = [newWord copy];
    NSLog(@"%@ เรียนรู้คำใหม่: '%@' (รวม %ld คำ)", self.name, newWord, (long)_vocabularySize);
}

@end
```

---

## 4. Calling super

### ทำไมต้องเรียก super

```objc
// SuperCallDemo.h
@interface Vehicle : NSObject

@property (nonatomic, assign) double speed;
@property (nonatomic, assign) BOOL   isRunning;

- (void)start;
- (void)stop;
- (NSString *)status;

@end

@interface Car : Vehicle

@property (nonatomic, assign) NSInteger gearPosition;
@property (nonatomic, assign) BOOL      airConditionOn;

- (void)shiftToGear:(NSInteger)gear;

@end

@interface ElectricCar : Car

@property (nonatomic, assign) double batteryLevel;  // 0.0 - 1.0

@end
```

```objc
// SuperCallDemo.m
@implementation Vehicle

- (void)start {
    _isRunning = YES;
    _speed = 0;
    NSLog(@"[Vehicle] เริ่มต้นระบบยานพาหนะ");
}

- (void)stop {
    _isRunning = NO;
    _speed = 0;
    NSLog(@"[Vehicle] หยุดยานพาหนะ");
}

- (NSString *)status {
    return [NSString stringWithFormat:@"ความเร็ว: %.1f | สถานะ: %@",
            _speed, _isRunning ? @"ทำงาน" : @"หยุด"];
}

@end

@implementation Car

- (void)start {
    [super start];  // ต้องเรียก super ก่อนเสมอสำหรับ start
    _gearPosition = 1;
    NSLog(@"[Car] เปลี่ยนเกียร์ไปที่ 1");
}

- (void)stop {
    _gearPosition = 0;
    NSLog(@"[Car] เข้าเกียร์ว่าง");
    [super stop];   // เรียก super หลัง - ก็ได้เช่นกัน
}

- (void)shiftToGear:(NSInteger)gear {
    if (!self.isRunning) {
        NSLog(@"[Car] ต้องสตาร์ทก่อนเปลี่ยนเกียร์");
        return;
    }
    _gearPosition = gear;
    NSLog(@"[Car] เกียร์ %ld", (long)gear);
}

- (NSString *)status {
    NSString *superStatus = [super status];  // ใช้ข้อมูลจาก super
    return [NSString stringWithFormat:@"%@ | เกียร์: %ld | แอร์: %@",
            superStatus, (long)_gearPosition, _airConditionOn ? @"เปิด" : @"ปิด"];
}

@end

@implementation ElectricCar

- (instancetype)init {
    self = [super init];
    if (self) {
        _batteryLevel = 1.0;
    }
    return self;
}

- (void)start {
    if (_batteryLevel <= 0) {
        NSLog(@"[ElectricCar] แบตหมด! ชาร์จก่อนนะ 🔋");
        return;
    }
    [super start];  // เรียก Car's start (ซึ่งเรียก Vehicle's start ต่อ)
    NSLog(@"[ElectricCar] มอเตอร์ไฟฟ้าพร้อม ⚡");
}

- (void)stop {
    [super stop];   // เรียก Car's stop (ซึ่งเรียก Vehicle's stop ต่อ)
    NSLog(@"[ElectricCar] ชาร์จพลังงานขณะหยุด 🔄");
    _batteryLevel = MIN(1.0, _batteryLevel + 0.02);
}

- (NSString *)status {
    NSString *superStatus = [super status];
    return [NSString stringWithFormat:@"%@ | แบต: %.0f%%",
            superStatus, _batteryLevel * 100];
}

@end
```

```objc
// ทดสอบ super chain
int main(int argc, const char * argv[]) {
    @autoreleasepool {
        ElectricCar *tesla = [[ElectricCar alloc] init];

        NSLog(@"=== สตาร์ท ===");
        [tesla start];
        // Output:
        // [Vehicle] เริ่มต้นระบบยานพาหนะ
        // [Car] เปลี่ยนเกียร์ไปที่ 1
        // [ElectricCar] มอเตอร์ไฟฟ้าพร้อม ⚡

        NSLog(@"\n=== หยุด ===");
        [tesla stop];
        // Output:
        // [Car] เข้าเกียร์ว่าง
        // [Vehicle] หยุดยานพาหนะ
        // [ElectricCar] ชาร์จพลังงานขณะหยุด 🔄

        NSLog(@"\nสถานะ: %@", [tesla status]);
    }
    return 0;
}
```

---

## 5. super vs self

### ความแตกต่างระหว่าง self และ super

```objc
@interface Shape : NSObject

- (NSString *)shapeName;
- (void)draw;
- (void)drawWithColor:(NSString *)color;

@end

@implementation Shape

- (NSString *)shapeName {
    return @"รูปทรงทั่วไป";
}

- (void)draw {
    NSLog(@"วาด %@", [self shapeName]);  // [self ...] - dynamic dispatch
}

- (void)drawWithColor:(NSString *)color {
    NSLog(@"วาด %@ ด้วยสี %@", [self shapeName], color);
}

@end

@interface Circle : Shape

@property (nonatomic, assign) CGFloat radius;

@end

@implementation Circle

- (NSString *)shapeName {
    return [NSString stringWithFormat:@"วงกลม (r=%.1f)", _radius];
}

- (void)draw {
    // self = dynamic dispatch -> จะเรียก Circle's shapeName
    NSLog(@"[self] draw ใน Circle, shapeName = %@", [self shapeName]);

    // super = static dispatch -> จะเรียก Shape's draw
    // แต่ใน Shape's draw มี [self shapeName] -> ยังเป็น Circle's shapeName!
    [super draw];
}

- (void)demonstrateSelfVsSuper {
    NSLog(@"--- self vs super ---");
    NSLog(@"[self shapeName] = %@", [self shapeName]);    // Circle's
    NSLog(@"[super shapeName] = %@", [super shapeName]);  // Shape's

    // [self draw] -> Circle's draw (ซ้ำไม่หยุด ถ้าไม่ระวัง)
    // [super draw] -> Shape's draw แต่ [self shapeName] ใน Shape ยังเป็น Circle's
}

@end
```

```objc
// การทดสอบ self vs super
Circle *c = [[Circle alloc] init];
c.radius = 5.0;

[c draw];
[c demonstrateSelfVsSuper];
```

---

## 6. Method Lookup Chain

### กลไกการค้นหา Method

เมื่อส่ง message ไปยัง object, Objective-C จะค้นหา method ตามลำดับ:

```
[object method]
    ↓
ตรวจสอบ object's class (ClassName)
    ↓ ไม่พบ
ตรวจสอบ superclass
    ↓ ไม่พบ
ตรวจสอบ superclass ของ superclass
    ...
    ↓ ไม่พบจนถึง NSObject
doesNotRecognizeSelector: (NSException)
```

```objc
// MethodLookupDemo
@interface A : NSObject
- (void)methodA;
- (void)commonMethod;
@end

@interface B : A
- (void)methodB;
- (void)commonMethod;  // Override
@end

@interface C : B
- (void)methodC;
// ไม่ override commonMethod
@end

@implementation A
- (void)methodA    { NSLog(@"A's methodA"); }
- (void)commonMethod { NSLog(@"A's commonMethod"); }
@end

@implementation B
- (void)methodB    { NSLog(@"B's methodB"); }
- (void)commonMethod { NSLog(@"B's commonMethod (override)"); }
@end

@implementation C
- (void)methodC    { NSLog(@"C's methodC"); }
@end
```

```objc
// ทดสอบ method lookup
C *obj = [[C alloc] init];

[obj methodC];       // C -> พบ methodC ใน C
[obj methodB];       // C -> ไม่พบ -> B -> พบ methodB ใน B
[obj methodA];       // C -> ไม่พบ -> B -> ไม่พบ -> A -> พบ methodA ใน A
[obj commonMethod];  // C -> ไม่พบ -> B -> พบ commonMethod ใน B (ไม่ไปถึง A)
```

---

## 7. isKindOfClass: และ isMemberOfClass:

### ตรวจสอบประเภทของ Object

| Method | ความหมาย |
|--------|---------|
| `isKindOfClass:` | ตรวจว่าเป็น instance ของ class นั้น **หรือ subclass** ของ class นั้น |
| `isMemberOfClass:` | ตรวจว่าเป็น instance ของ class นั้น **โดยตรง** เท่านั้น |

```objc
Animal  *animal = [[Animal  alloc] initWithName:@"สัตว์" age:1 species:@"?"];
Dog     *dog    = [[Dog     alloc] initWithName:@"ออดี้" age:2 breed:@"Lab" owner:@"คุณสมชาย"];
Cat     *cat    = [[Cat     alloc] initWithName:@"มิ้ว"  age:1 furColor:@"ส้ม" isIndoor:YES];
Parrot  *parrot = [[Parrot  alloc] initWithName:@"โพลลี่" age:3 favoriteWord:@"สวัสดี"];

// isKindOfClass: - ตรวจ class หรือ subclass
NSLog(@"dog isKindOfClass Animal:  %@", [dog isKindOfClass:[Animal class]]  ? @"YES" : @"NO");  // YES
NSLog(@"dog isKindOfClass Dog:     %@", [dog isKindOfClass:[Dog class]]     ? @"YES" : @"NO");  // YES
NSLog(@"dog isKindOfClass Cat:     %@", [dog isKindOfClass:[Cat class]]     ? @"YES" : @"NO");  // NO
NSLog(@"dog isKindOfClass NSObject:%@", [dog isKindOfClass:[NSObject class]]? @"YES" : @"NO");  // YES

// isMemberOfClass: - ตรวจ class โดยตรงเท่านั้น
NSLog(@"dog isMemberOfClass Animal:%@", [dog isMemberOfClass:[Animal class]]? @"YES" : @"NO");  // NO!
NSLog(@"dog isMemberOfClass Dog:   %@", [dog isMemberOfClass:[Dog class]]   ? @"YES" : @"NO");  // YES

// parrot ซึ่งเป็น subclass ของ Bird ซึ่งเป็น subclass ของ Animal
NSLog(@"parrot isKindOfClass Animal:%@", [parrot isKindOfClass:[Animal class]]? @"YES" : @"NO");// YES
NSLog(@"parrot isKindOfClass Bird:  %@", [parrot isKindOfClass:[Bird class]]  ? @"YES" : @"NO");// YES
NSLog(@"parrot isMemberOfClass Bird:%@", [parrot isMemberOfClass:[Bird class]]? @"YES" : @"NO");// NO
NSLog(@"parrot isMemberOfClass Parrot:%@",[parrot isMemberOfClass:[Parrot class]]?@"YES":@"NO");// YES
```

### การใช้งานจริง: Polymorphism

```objc
NSArray *zoo = @[
    [[Dog    alloc] initWithName:@"โบ"     age:3 breed:@"Shiba"  owner:@"สมชาย"],
    [[Cat    alloc] initWithName:@"มิ้ว"   age:2 furColor:@"ขาว" isIndoor:YES],
    [[Bird   alloc] initWithName:@"ทวีป"   age:1 wingColor:@"น้ำเงิน" canFly:YES wingspan:0.5],
    [[Parrot alloc] initWithName:@"โพลลี่" age:4 favoriteWord:@"ฮัลโหล"],
];

for (Animal *animal in zoo) {
    NSLog(@"\n--- %@ ---", animal.name);
    [animal makeSound];   // Polymorphism - เรียก method ที่ถูกต้องตาม class จริง
    [animal eat:@"อาหาร"];

    // ตรวจ type ก่อนใช้ method เฉพาะ
    if ([animal isKindOfClass:[Dog class]]) {
        Dog *d = (Dog *)animal;
        [d fetch];
    } else if ([animal isKindOfClass:[Bird class]]) {
        Bird *b = (Bird *)animal;
        [b fly];
    }
}
```

---

## 8. respondsToSelector:

ตรวจสอบว่า object มี method นั้นหรือไม่ ก่อนเรียกใช้

```objc
// RespondsToSelectorDemo
@protocol Flyable <NSObject>
- (void)fly;
@optional
- (void)land;
@end

Animal *animals[] = {
    [Animal  class],
    [Dog     class],
    [Bird    class],
};

// ตรวจสอบ method แต่ละตัวก่อนเรียก
for (Animal *animal in @[
    [[Dog   alloc] initWithName:@"โบ"    age:2 breed:@"Lab" owner:@"สมชาย"],
    [[Bird  alloc] initWithName:@"ทวีป"  age:1 wingColor:@"น้ำเงิน" canFly:YES wingspan:0.5],
    [[Cat   alloc] initWithName:@"มิ้ว"  age:1 furColor:@"ส้ม" isIndoor:YES],
]) {
    NSLog(@"\n[%@] %@:", [animal class], animal.name);

    // ตรวจ method ก่อนเรียก
    if ([animal respondsToSelector:@selector(fly)]) {
        NSLog(@"  -> สามารถบินได้");
        [(Bird *)animal fly];
    } else {
        NSLog(@"  -> บินไม่ได้");
    }

    if ([animal respondsToSelector:@selector(bark)]) {
        [(Dog *)animal bark];
    }

    if ([animal respondsToSelector:@selector(meow)]) {
        [(Cat *)animal meow];
    }
}
```

```objc
// ใช้ respondsToSelector: กับ delegate pattern
@protocol DataSourceDelegate <NSObject>

@required
- (NSInteger)numberOfItems;

@optional
- (NSString *)titleForItemAtIndex:(NSInteger)index;
- (void)didSelectItemAtIndex:(NSInteger)index;

@end

@interface DataManager : NSObject

@property (nonatomic, weak) id<DataSourceDelegate> delegate;

- (void)loadData;

@end

@implementation DataManager

- (void)loadData {
    if (!_delegate) return;

    NSInteger count = [_delegate numberOfItems];
    NSLog(@"จำนวน items: %ld", (long)count);

    for (NSInteger i = 0; i < count; i++) {
        // ตรวจก่อนเรียก optional method
        if ([_delegate respondsToSelector:@selector(titleForItemAtIndex:)]) {
            NSString *title = [_delegate titleForItemAtIndex:i];
            NSLog(@"  item %ld: %@", (long)i, title);
        }
    }
}

@end
```

---

## 9. Dynamic Dispatch

Objective-C ใช้ Dynamic Dispatch ซึ่งหมายความว่า method ที่จะเรียกถูกกำหนดที่ **runtime** ไม่ใช่ compile time

```objc
// DynamicDispatchDemo
@interface Shape : NSObject
- (double)area;
- (NSString *)shapeName;
@end

@interface Circle : Shape
@property (nonatomic, assign) double radius;
- (instancetype)initWithRadius:(double)radius;
@end

@interface Rectangle : Shape
@property (nonatomic, assign) double width;
@property (nonatomic, assign) double height;
- (instancetype)initWithWidth:(double)width height:(double)height;
@end

@interface Triangle : Shape
@property (nonatomic, assign) double base;
@property (nonatomic, assign) double height;
- (instancetype)initWithBase:(double)base height:(double)height;
@end
```

```objc
@implementation Shape
- (double)area { return 0.0; }
- (NSString *)shapeName { return @"รูปทรง"; }
@end

@implementation Circle
- (instancetype)initWithRadius:(double)radius {
    self = [super init];
    if (self) { _radius = radius; }
    return self;
}
- (double)area { return M_PI * _radius * _radius; }
- (NSString *)shapeName { return @"วงกลม"; }
@end

@implementation Rectangle
- (instancetype)initWithWidth:(double)width height:(double)height {
    self = [super init];
    if (self) { _width = width; _height = height; }
    return self;
}
- (double)area { return _width * _height; }
- (NSString *)shapeName { return @"สี่เหลี่ยม"; }
@end

@implementation Triangle
- (instancetype)initWithBase:(double)base height:(double)height {
    self = [super init];
    if (self) { _base = base; _height = height; }
    return self;
}
- (double)area { return 0.5 * _base * _height; }
- (NSString *)shapeName { return @"สามเหลี่ยม"; }
@end
```

```objc
// Dynamic dispatch ในการทำงาน
NSArray<Shape *> *shapes = @[
    [[Circle    alloc] initWithRadius:5],
    [[Rectangle alloc] initWithWidth:4 height:6],
    [[Triangle  alloc] initWithBase:3 height:8],
    [[Circle    alloc] initWithRadius:2.5],
];

double totalArea = 0;
for (Shape *shape in shapes) {
    // Runtime เรียก method ที่ถูกต้องสำหรับแต่ละ class
    double area = [shape area];
    totalArea += area;
    NSLog(@"%@ : พื้นที่ = %.4f", [shape shapeName], area);
}
NSLog(@"พื้นที่รวมทั้งหมด: %.4f", totalArea);
```

---

## 10. Abstract Classes Pattern

Objective-C ไม่มี abstract class แบบ native แต่สามารถจำลองได้

### วิธีที่ 1: NSException ใน Base Class

```objc
// AbstractShape.h
@interface AbstractShape : NSObject

// "Abstract" methods - subclass ต้อง override
- (double)area;
- (double)perimeter;
- (void)draw;

// Concrete methods ที่ใช้ "abstract" methods
- (NSString *)shapeReport;
- (BOOL)isLargerThan:(AbstractShape *)other;

@end
```

```objc
// AbstractShape.m
@implementation AbstractShape

- (double)area {
    // บังคับให้ subclass override
    [NSException raise:NSInternalInconsistencyException
                format:@"%@ ต้อง override method 'area'", [self class]];
    return 0;
}

- (double)perimeter {
    [NSException raise:NSInternalInconsistencyException
                format:@"%@ ต้อง override method 'perimeter'", [self class]];
    return 0;
}

- (void)draw {
    [NSException raise:NSInternalInconsistencyException
                format:@"%@ ต้อง override method 'draw'", [self class]];
}

// Concrete method ที่ใช้ abstract methods
- (NSString *)shapeReport {
    return [NSString stringWithFormat:
            @"รูปทรง: %@\nพื้นที่: %.4f\nเส้นรอบวง: %.4f",
            [self class], [self area], [self perimeter]];
}

- (BOOL)isLargerThan:(AbstractShape *)other {
    return [self area] > [other area];
}

@end
```

```objc
// ConcreteCircle.h - Concrete implementation
@interface ConcreteCircle : AbstractShape

@property (nonatomic, assign) double radius;

- (instancetype)initWithRadius:(double)radius;

@end

@implementation ConcreteCircle

- (instancetype)initWithRadius:(double)radius {
    self = [super init];
    if (self) { _radius = radius; }
    return self;
}

- (double)area      { return M_PI * _radius * _radius; }
- (double)perimeter { return 2 * M_PI * _radius; }
- (void)draw        { NSLog(@"วาดวงกลม radius=%.1f ○", _radius); }

@end
```

### วิธีที่ 2: Protocol แทน Abstract Class

```objc
// ใช้ Protocol เพื่อกำหนด "abstract interface"
@protocol Drawable <NSObject>

@required
- (void)draw;
- (CGRect)boundingBox;

@optional
- (void)drawWithColor:(NSString *)color;

@end

@protocol Measurable <NSObject>

@required
- (double)area;
- (double)perimeter;

@end

// Class ที่ conform ทั้งสอง protocol = ทำหน้าที่เหมือน abstract class
@interface ConcreteShape : NSObject <Drawable, Measurable>
@end
```

---

## 11. Class Hierarchy: Animal -> Dog, Cat, Bird (ครบถ้วน)

```objc
// สรุปลำดับชั้น Animal hierarchy พร้อม polymorphism

// Animal.h (ดูด้านบน)
// Dog.h (ดูด้านบน)
// Cat.h (ดูด้านบน)
// Bird.h (ดูด้านบน)
// Parrot.h (ดูด้านบน)

// ZooManager.h
@interface ZooManager : NSObject

@property (nonatomic, strong, readonly) NSArray<Animal *> *animals;

- (void)addAnimal:(Animal *)animal;
- (void)removeAnimalNamed:(NSString *)name;
- (void)feedAllAnimals;
- (void)makeAllSounds;
- (NSArray<Animal *> *)animalsOfClass:(Class)animalClass;
- (void)printZooReport;

@end
```

```objc
// ZooManager.m
#import "ZooManager.h"

@interface ZooManager ()
@property (nonatomic, strong) NSMutableArray<Animal *> *mutableAnimals;
@end

@implementation ZooManager

- (instancetype)init {
    self = [super init];
    if (self) {
        _mutableAnimals = [NSMutableArray array];
    }
    return self;
}

- (NSArray<Animal *> *)animals {
    return [_mutableAnimals copy];
}

- (void)addAnimal:(Animal *)animal {
    [_mutableAnimals addObject:animal];
    NSLog(@"ยินดีต้อนรับ %@ เข้าสู่สวนสัตว์!", animal.name);
}

- (void)removeAnimalNamed:(NSString *)name {
    Animal *toRemove = nil;
    for (Animal *a in _mutableAnimals) {
        if ([a.name isEqualToString:name]) {
            toRemove = a;
            break;
        }
    }
    if (toRemove) {
        [_mutableAnimals removeObject:toRemove];
        NSLog(@"%@ ออกจากสวนสัตว์แล้ว", name);
    }
}

- (void)feedAllAnimals {
    NSLog(@"\n=== เวลาให้อาหาร ===");
    for (Animal *animal in _mutableAnimals) {
        NSString *food;
        if ([animal isKindOfClass:[Dog class]]) {
            food = @"กระดูก";
        } else if ([animal isKindOfClass:[Cat class]]) {
            food = @"ปลาทู";
        } else if ([animal isKindOfClass:[Bird class]]) {
            food = @"เมล็ดพืช";
        } else {
            food = @"อาหารทั่วไป";
        }
        [animal eat:food];
    }
}

- (void)makeAllSounds {
    NSLog(@"\n=== เสียงในสวนสัตว์ ===");
    for (Animal *animal in _mutableAnimals) {
        [animal makeSound];
    }
}

- (NSArray<Animal *> *)animalsOfClass:(Class)animalClass {
    NSMutableArray *result = [NSMutableArray array];
    for (Animal *a in _mutableAnimals) {
        if ([a isKindOfClass:animalClass]) {
            [result addObject:a];
        }
    }
    return [result copy];
}

- (void)printZooReport {
    NSLog(@"\n====== รายงานสวนสัตว์ ======");
    NSLog(@"จำนวนสัตว์ทั้งหมด: %lu", (unsigned long)_mutableAnimals.count);
    NSLog(@"สุนัข: %lu ตัว", (unsigned long)[self animalsOfClass:[Dog class]].count);
    NSLog(@"แมว: %lu ตัว",   (unsigned long)[self animalsOfClass:[Cat class]].count);
    NSLog(@"นก: %lu ตัว",    (unsigned long)[self animalsOfClass:[Bird class]].count);
    NSLog(@"\nรายชื่อสัตว์:");
    for (Animal *animal in _mutableAnimals) {
        NSLog(@"  - %@", animal);
    }
    NSLog(@"============================");
}

@end
```

```objc
// การใช้งาน ZooManager
int main(int argc, const char * argv[]) {
    @autoreleasepool {
        ZooManager *zoo = [[ZooManager alloc] init];

        // เพิ่มสัตว์
        [zoo addAnimal:[[Dog alloc] initWithName:@"โบ"     age:3 breed:@"Shiba"  owner:@"สมชาย"]];
        [zoo addAnimal:[[Dog alloc] initWithName:@"ออดี้"   age:2 breed:@"Lab"    owner:@"สมหญิง"]];
        [zoo addAnimal:[[Cat alloc] initWithName:@"มิ้ว"   age:4 furColor:@"ส้ม" isIndoor:YES]];
        [zoo addAnimal:[[Cat alloc] initWithName:@"ขาว"    age:1 furColor:@"ขาว" isIndoor:NO]];
        [zoo addAnimal:[[Bird alloc] initWithName:@"ทวีป"  age:2 wingColor:@"น้ำเงิน" canFly:YES wingspan:0.4]];
        [zoo addAnimal:[[Parrot alloc] initWithName:@"โพลลี่" age:5 favoriteWord:@"สวัสดี"]];

        // การใช้งาน
        [zoo makeAllSounds];
        [zoo feedAllAnimals];
        [zoo printZooReport];
    }
    return 0;
}
```

---

## 12. Vehicle Hierarchy Example

```objc
// Vehicle.h (Base)
@interface Vehicle : NSObject

@property (nonatomic, copy)   NSString *brand;
@property (nonatomic, assign) NSInteger year;
@property (nonatomic, assign) double    maxSpeed;   // km/h
@property (nonatomic, assign, readonly) double currentSpeed;

- (instancetype)initWithBrand:(NSString *)brand
                         year:(NSInteger)year
                     maxSpeed:(double)maxSpeed;

- (void)accelerateTo:(double)speed;
- (void)brake;
- (NSString *)vehicleType;
- (NSString *)vehicleInfo;

@end

// LandVehicle.h
@interface LandVehicle : Vehicle

@property (nonatomic, assign) NSInteger wheels;
@property (nonatomic, copy)   NSString *driveType;  // FWD, RWD, AWD

@end

// Car.h
@interface Car : LandVehicle

@property (nonatomic, assign) NSInteger doors;
@property (nonatomic, assign) NSInteger passengerCapacity;
@property (nonatomic, copy)   NSString *bodyStyle; // sedan, SUV, hatchback

- (instancetype)initWithBrand:(NSString *)brand
                         year:(NSInteger)year
                    bodyStyle:(NSString *)bodyStyle
                        doors:(NSInteger)doors;

@end

// Truck.h
@interface Truck : LandVehicle

@property (nonatomic, assign) double payloadCapacity;  // ตัน
@property (nonatomic, assign) NSInteger axles;

@end

// Motorcycle.h
@interface Motorcycle : LandVehicle

@property (nonatomic, copy) NSString *type;  // sport, cruiser, touring

@end

// Boat.h
@interface Boat : Vehicle

@property (nonatomic, assign) double length;  // เมตร
@property (nonatomic, copy)   NSString *hullType;

- (void)dropAnchor;
- (void)raiseSail;

@end

// Airplane.h
@interface Airplane : Vehicle

@property (nonatomic, assign) double altitude;       // เมตร
@property (nonatomic, assign) NSInteger engines;
@property (nonatomic, assign) NSInteger passengerCapacity;

- (void)takeoff;
- (void)land;
- (void)cruiseAtAltitude:(double)altitude;

@end
```

```objc
// Implementations
@implementation Vehicle

- (instancetype)initWithBrand:(NSString *)brand
                         year:(NSInteger)year
                     maxSpeed:(double)maxSpeed {
    self = [super init];
    if (self) {
        _brand        = [brand copy];
        _year         = year;
        _maxSpeed     = maxSpeed;
        _currentSpeed = 0;
    }
    return self;
}

- (void)accelerateTo:(double)speed {
    if (speed > _maxSpeed) {
        NSLog(@"[%@] เกินความเร็วสูงสุด! จำกัดที่ %.0f km/h", _brand, _maxSpeed);
        _currentSpeed = _maxSpeed;
    } else {
        _currentSpeed = speed;
        NSLog(@"[%@] เร่งความเร็วถึง %.0f km/h", _brand, _currentSpeed);
    }
}

- (void)brake {
    NSLog(@"[%@] เบรก (%.0f -> 0 km/h)", _brand, _currentSpeed);
    _currentSpeed = 0;
}

- (NSString *)vehicleType { return @"ยานพาหนะ"; }

- (NSString *)vehicleInfo {
    return [NSString stringWithFormat:@"%d %@ (%@) | ความเร็วสูงสุด: %.0f km/h",
            (int)_year, _brand, [self vehicleType], _maxSpeed];
}

- (NSString *)description { return [self vehicleInfo]; }

@end

@implementation LandVehicle
- (NSString *)vehicleType { return @"ยานพาหนะทางบก"; }
@end

@implementation Car

- (instancetype)initWithBrand:(NSString *)brand
                         year:(NSInteger)year
                    bodyStyle:(NSString *)bodyStyle
                        doors:(NSInteger)doors {
    self = [super initWithBrand:brand year:year maxSpeed:220];
    if (self) {
        _bodyStyle = [bodyStyle copy];
        _doors = doors;
        _passengerCapacity = doors <= 2 ? 2 : 5;
        self.wheels = 4;
        self.driveType = @"FWD";
    }
    return self;
}

- (NSString *)vehicleType {
    return [NSString stringWithFormat:@"รถยนต์ (%@)", _bodyStyle];
}

- (NSString *)vehicleInfo {
    return [NSString stringWithFormat:@"%@ | %ld ประตู | %ld ที่นั่ง",
            [super vehicleInfo], (long)_doors, (long)_passengerCapacity];
}

@end

@implementation Airplane

- (instancetype)init {
    self = [super initWithBrand:@"Generic" year:2020 maxSpeed:900];
    if (self) {
        _engines = 2;
        _passengerCapacity = 150;
        _altitude = 0;
    }
    return self;
}

- (void)takeoff {
    _altitude = 10000;
    NSLog(@"[%@] ขึ้นบิน สูง %.0f เมตร", self.brand, _altitude);
    [self accelerateTo:850];
}

- (void)land {
    NSLog(@"[%@] กำลังลงจอด...", self.brand);
    [self brake];
    _altitude = 0;
    NSLog(@"[%@] ลงจอดสำเร็จ", self.brand);
}

- (void)cruiseAtAltitude:(double)altitude {
    _altitude = altitude;
    NSLog(@"[%@] บินที่ระดับ %.0f เมตร", self.brand, altitude);
}

- (NSString *)vehicleType { return @"เครื่องบิน"; }

@end
```

---

## 13. When to Use Inheritance vs Composition

### Inheritance: ใช้เมื่อมีความสัมพันธ์แบบ "IS-A"

```objc
// ✅ ดี: Dog IS-A Animal
@interface Dog : Animal @end

// ✅ ดี: Car IS-A Vehicle
@interface Car : Vehicle @end

// ❌ ไม่ดี: Engine IS-A Car? ไม่ใช่! Engine เป็นส่วนประกอบของ Car
// @interface Engine : Car @end  // ผิด!
```

### Composition: ใช้เมื่อมีความสัมพันธ์แบบ "HAS-A"

```objc
// ✅ ดี: Car HAS-A Engine (Composition)
@interface Engine : NSObject

@property (nonatomic, assign) NSInteger horsepower;
@property (nonatomic, assign) double    displacement;
@property (nonatomic, copy)   NSString *fuelType;

- (void)start;
- (void)stop;
- (NSString *)engineSpecs;

@end

@interface Transmission : NSObject

@property (nonatomic, assign) NSInteger gears;
@property (nonatomic, copy)   NSString *type;  // manual, automatic, CVT

- (void)shiftUp;
- (void)shiftDown;

@end

@interface ComposedCar : NSObject

@property (nonatomic, copy)   NSString      *brand;
@property (nonatomic, strong) Engine        *engine;        // HAS-A Engine
@property (nonatomic, strong) Transmission  *transmission;  // HAS-A Transmission

- (instancetype)initWithBrand:(NSString *)brand
                       engine:(Engine *)engine
                 transmission:(Transmission *)transmission;

- (void)startCar;
- (void)stopCar;

@end
```

```objc
@implementation ComposedCar

- (instancetype)initWithBrand:(NSString *)brand
                       engine:(Engine *)engine
                 transmission:(Transmission *)transmission {
    self = [super init];
    if (self) {
        _brand        = [brand copy];
        _engine       = engine;
        _transmission = transmission;
    }
    return self;
}

- (void)startCar {
    [_engine start];
    NSLog(@"[%@] พร้อมขับ", _brand);
}

- (void)stopCar {
    [_engine stop];
}

@end
```

### เมื่อควรใช้อะไร

| สถานการณ์ | ใช้ |
|-----------|-----|
| B "เป็น" A อย่างชัดเจน | Inheritance |
| B "มี" A เป็นส่วนประกอบ | Composition |
| ต้องการ override behavior | Inheritance |
| ต้องการเปลี่ยน component ที่ runtime | Composition |
| มีหลาย "roles" ที่ต้องรวมกัน | Protocol + Composition |
| ต้องการ reuse code โดยไม่ต้องการ "เป็น" type นั้น | Composition |

---

## 14. Practical Example: Shape Hierarchy

```objc
// Protocol สำหรับ rendering
@protocol Renderable <NSObject>
- (void)renderWithColor:(NSString *)color;
@optional
- (void)renderWithGradientFrom:(NSString *)startColor to:(NSString *)endColor;
@end

// Base Shape
@interface BaseShape : NSObject <Renderable>

@property (nonatomic, assign) CGPoint   center;
@property (nonatomic, copy)   NSString *fillColor;
@property (nonatomic, copy)   NSString *strokeColor;
@property (nonatomic, assign) CGFloat   strokeWidth;

- (instancetype)initAtCenter:(CGPoint)center;

// Abstract methods (raise exception if not overridden)
- (CGFloat)area;
- (CGFloat)perimeter;
- (CGRect)boundingRect;
- (BOOL)containsPoint:(CGPoint)point;

// Concrete methods
- (void)moveTo:(CGPoint)newCenter;
- (void)moveBy:(CGVector)delta;
- (NSString *)shapeDescription;

@end

// Circle
@interface CircleShape : BaseShape

@property (nonatomic, assign) CGFloat radius;

+ (instancetype)circleAtCenter:(CGPoint)center radius:(CGFloat)radius;

@end

// Rectangle
@interface RectShape : BaseShape

@property (nonatomic, assign) CGFloat width;
@property (nonatomic, assign) CGFloat height;

+ (instancetype)rectAtCenter:(CGPoint)center width:(CGFloat)width height:(CGFloat)height;

@end

// Regular Polygon (สามเหลี่ยม, หกเหลี่ยม ฯลฯ)
@interface RegularPolygon : BaseShape

@property (nonatomic, assign) NSInteger sides;
@property (nonatomic, assign) CGFloat   sideLength;

+ (instancetype)polygonAtCenter:(CGPoint)center sides:(NSInteger)sides sideLength:(CGFloat)length;

@end
```

```objc
@implementation BaseShape

- (instancetype)initAtCenter:(CGPoint)center {
    self = [super init];
    if (self) {
        _center      = center;
        _fillColor   = @"white";
        _strokeColor = @"black";
        _strokeWidth = 1.0;
    }
    return self;
}

- (CGFloat)area {
    [NSException raise:NSInternalInconsistencyException
                format:@"%@ must override 'area'", [self class]];
    return 0;
}

- (CGFloat)perimeter {
    [NSException raise:NSInternalInconsistencyException
                format:@"%@ must override 'perimeter'", [self class]];
    return 0;
}

- (CGRect)boundingRect {
    [NSException raise:NSInternalInconsistencyException
                format:@"%@ must override 'boundingRect'", [self class]];
    return CGRectZero;
}

- (BOOL)containsPoint:(CGPoint)point {
    [NSException raise:NSInternalInconsistencyException
                format:@"%@ must override 'containsPoint:'", [self class]];
    return NO;
}

- (void)moveTo:(CGPoint)newCenter {
    _center = newCenter;
}

- (void)moveBy:(CGVector)delta {
    _center.x += delta.dx;
    _center.y += delta.dy;
}

- (void)renderWithColor:(NSString *)color {
    _fillColor = color;
    NSLog(@"[%@] วาดรูปที่ (%.1f, %.1f) สี %@",
          [self class], _center.x, _center.y, color);
}

- (NSString *)shapeDescription {
    return [NSString stringWithFormat:
            @"%@ at (%.1f, %.1f) | area=%.2f | perimeter=%.2f",
            [self class], _center.x, _center.y, [self area], [self perimeter]];
}

@end

@implementation CircleShape

+ (instancetype)circleAtCenter:(CGPoint)center radius:(CGFloat)radius {
    CircleShape *c = [[self alloc] initAtCenter:center];
    c->_radius = radius;
    return c;
}

- (CGFloat)area      { return M_PI * _radius * _radius; }
- (CGFloat)perimeter { return 2 * M_PI * _radius; }

- (CGRect)boundingRect {
    return CGRectMake(_center.x - _radius, _center.y - _radius,
                      _radius * 2, _radius * 2);
}

- (BOOL)containsPoint:(CGPoint)point {
    CGFloat dx = point.x - _center.x;
    CGFloat dy = point.y - _center.y;
    return (dx * dx + dy * dy) <= (_radius * _radius);
}

@end

@implementation RectShape

+ (instancetype)rectAtCenter:(CGPoint)center width:(CGFloat)width height:(CGFloat)height {
    RectShape *r = [[self alloc] initAtCenter:center];
    r->_width  = width;
    r->_height = height;
    return r;
}

- (CGFloat)area      { return _width * _height; }
- (CGFloat)perimeter { return 2 * (_width + _height); }

- (CGRect)boundingRect {
    return CGRectMake(_center.x - _width/2, _center.y - _height/2, _width, _height);
}

- (BOOL)containsPoint:(CGPoint)point {
    CGRect bounds = [self boundingRect];
    return CGRectContainsPoint(bounds, point);
}

@end

@implementation RegularPolygon

+ (instancetype)polygonAtCenter:(CGPoint)center sides:(NSInteger)sides sideLength:(CGFloat)length {
    RegularPolygon *p = [[self alloc] initAtCenter:center];
    p->_sides      = sides;
    p->_sideLength = length;
    return p;
}

- (CGFloat)area {
    // A = (n * s^2) / (4 * tan(π/n))
    double n = _sides;
    double s = _sideLength;
    return (n * s * s) / (4.0 * tan(M_PI / n));
}

- (CGFloat)perimeter {
    return _sides * _sideLength;
}

- (CGRect)boundingRect {
    // circumradius R = s / (2 * sin(π/n))
    CGFloat R = _sideLength / (2.0 * sin(M_PI / _sides));
    return CGRectMake(_center.x - R, _center.y - R, R * 2, R * 2);
}

- (BOOL)containsPoint:(CGPoint)point {
    // อย่างง่าย: ใช้ bounding circle
    CGFloat R = _sideLength / (2.0 * sin(M_PI / _sides));
    CGFloat dx = point.x - _center.x;
    CGFloat dy = point.y - _center.y;
    return (dx * dx + dy * dy) <= (R * R);
}

@end
```

---

## 15. แบบฝึกหัด (10+ ข้อ)

### ข้อ 1: Employee Hierarchy
สร้าง class hierarchy:
- `Employee` (base): id, name, baseSalary
- `Manager : Employee`: department, directReports (array of Employee)
- `Engineer : Employee`: skills (array), level (Junior/Mid/Senior)
- `Designer : Employee`: tools (array), portfolio
- Override calculateBonus ใน แต่ละ class

### ข้อ 2: Exception Handling Hierarchy
สร้าง custom exception hierarchy:
- `AppException : NSException`
- `NetworkException : AppException`
- `DatabaseException : AppException`
- `ValidationException : AppException`
- แต่ละ class มี userInfo และ recovery suggestions

### ข้อ 3: UI Component Hierarchy
สร้าง class hierarchy (จำลอง UIKit):
- `UIComponent` (base): frame, isVisible, backgroundColor
- `UILabel : UIComponent`: text, font, textColor
- `UIButton : UIComponent`: title, isEnabled, action
- `UIImageView : UIComponent`: image, contentMode
- Override renderToString ใน แต่ละ class

### ข้อ 4: Food Menu Hierarchy
สร้าง:
- `MenuItem` (base): name, price, calories
- `MainDish : MenuItem`: protein, servingSize
- `SideDish : MenuItem`: portion
- `Beverage : MenuItem`: volume, isAlcoholic
- `Dessert : MenuItem`: sweetness
- Class method สร้าง restaurant menu

### ข้อ 5: File System Hierarchy
สร้าง:
- `FileSystemItem` (base): name, path, size, createdDate
- `File : FileSystemItem`: extension, content
- `Directory : FileSystemItem`: children (array)
- Override description, size (Directory = sum of children)

### ข้อ 6: Sensor Hierarchy
สร้าง:
- `Sensor` (base): sensorID, name, readingInterval
- `TemperatureSensor : Sensor`: currentTemp, unit (C/F/K)
- `HumiditySensor : Sensor`: humidity (0-100%)
- `PressureSensor : Sensor`: pressure (hPa)
- `ComboSensor : Sensor`: ถือ array ของ sensors อื่น

### ข้อ 7: Game Character Hierarchy
สร้าง RPG character system:
- `Character` (base): name, hp, maxHp, level, exp
- `Warrior : Character`: strength, armor, attackMelee
- `Mage : Character`: mana, spells (array), castSpell:
- `Archer : Character`: agility, arrows, shootArrow:
- Abstract method: attack: ใน Character

### ข้อ 8: Bank Account Hierarchy
สร้าง:
- `Account` (base): accountNumber, balance, owner
- `SavingsAccount : Account`: interestRate, applyInterest
- `CheckingAccount : Account`: overdraftLimit, checkNumber
- `FixedDepositAccount : Account`: maturityDate, penalty
- Override withdraw ใน แต่ละ class (ต่างกันตามกฎ)

### ข้อ 9: Notification System
สร้าง:
- `Notification` (base): id, title, message, timestamp
- `EmailNotification : Notification`: recipient, subject, htmlBody
- `SMSNotification : Notification`: phoneNumber, charLimit
- `PushNotification : Notification`: deviceToken, badge, sound
- Override send, format ใน แต่ละ class

### ข้อ 10: Data Transformer Hierarchy
สร้าง:
- `DataTransformer` (base): inputData, outputData
- `JSONTransformer : DataTransformer`: parse, stringify
- `XMLTransformer : DataTransformer`: parse, stringify
- `CSVTransformer : DataTransformer`: parse, stringify, delimiter
- ใช้ method polymorphism

### ข้อ 11 (ท้าทาย): Observer Pattern + Inheritance
สร้าง Observable base class และ subclasses:
- `Observable`: addObserver:, removeObserver:, notifyObservers:
- `StockPrice : Observable`: price, symbol
- `WeatherStation : Observable`: temperature, humidity
- `Observer protocol`: update:

### ข้อ 12 (ท้าทาย): Expression Tree
สร้าง expression evaluation tree:
- `Expression` (base, abstract): evaluate
- `NumberExpression : Expression`: value
- `BinaryExpression : Expression`: left, right (both Expression)
  - `AddExpression`, `SubtractExpression`, `MultiplyExpression`, `DivideExpression`
- `UnaryExpression : Expression`: operand
  - `NegateExpression`, `AbsoluteExpression`

---

## สรุป Part 12

ใน Part นี้เราได้เรียนรู้:

1. **Single Inheritance** - `@interface Sub : Super`
2. **Method Overriding** - override method ของ superclass
3. **Calling super** - เรียก implementation ของ superclass
4. **self vs super** - dynamic vs static dispatch
5. **Method Lookup Chain** - กลไกค้นหา method ที่ runtime
6. **isKindOfClass: / isMemberOfClass:** - ตรวจสอบประเภท
7. **respondsToSelector:** - ตรวจสอบ method ก่อนเรียก
8. **Dynamic Dispatch** - runtime method resolution
9. **Abstract Classes Pattern** - จำลอง abstract class ใน Objective-C
10. **Animal/Vehicle Hierarchy** - ตัวอย่างการสืบทอดที่ดี
11. **Inheritance vs Composition** - เลือกใช้อย่างเหมาะสม

ใน Part 13 จะเรียนเรื่อง **Encapsulation and Properties** การซ่อนข้อมูลและควบคุมการเข้าถึง
