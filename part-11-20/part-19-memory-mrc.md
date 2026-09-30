# ตอนที่ 19: Manual Reference Counting (MRC) - การจัดการ Memory แบบ Manual

## บทนำ

การจัดการ memory เป็นหนึ่งในหัวข้อที่สำคัญที่สุดในการเขียนโปรแกรม Objective-C ในยุคก่อน ARC (Automatic Reference Counting) โปรแกรมเมอร์ต้องจัดการ memory ด้วยตนเองผ่านกลไกที่เรียกว่า **Manual Reference Counting (MRC)** หรือบางครั้งเรียกว่า **Manual Retain-Release (MRR)**

แม้ว่าในปัจจุบัน ARC จะเป็นค่าเริ่มต้น แต่การเข้าใจ MRC เป็นสิ่งสำคัญมากเพราะ:

1. **เข้าใจว่า ARC ทำอะไรให้เรา** - ARC แค่แทรก retain/release อัตโนมัติ ถ้าเข้าใจ MRC จะเข้าใจ ARC ได้ดีขึ้น
2. **ทำงานกับ Core Foundation** - CF objects ยังคงต้องจัดการ memory เอง
3. **Debug memory leaks** - การเข้าใจ reference counting ช่วย debug ได้
4. **Maintain legacy code** - โค้ดเก่าจำนวนมากยังใช้ MRC

---

## 19.1 Reference Counting Concept

### แนวคิดพื้นฐาน

Reference counting (การนับการอ้างอิง) คือกลไกในการติดตามว่ามีผู้ใช้ object กี่คน:

- เมื่อสร้าง object → **retain count = 1**
- เมื่อมีคนต้องการใช้ object → **retain count เพิ่ม 1** (`retain`)
- เมื่อไม่ต้องการใช้แล้ว → **retain count ลด 1** (`release`)
- เมื่อ retain count = 0 → **object ถูกทำลาย** (`dealloc`)

```
┌─────────────────────────────────────────────────────┐
│                 Reference Counting                  │
│                                                     │
│  alloc/init → retain count = 1                     │
│  retain     → retain count++                       │
│  release    → retain count--                       │
│  count = 0  → dealloc (memory freed)               │
└─────────────────────────────────────────────────────┘
```

```objc
// การแสดง retain count (ใช้เพื่อทดสอบเท่านั้น)
NSObject *obj = [[NSObject alloc] init];
NSLog(@"Retain count: %lu", [obj retainCount]);  // 1

[obj retain];
NSLog(@"Retain count: %lu", [obj retainCount]);  // 2

[obj release];
NSLog(@"Retain count: %lu", [obj retainCount]);  // 1

[obj release];
// ตอนนี้ retain count = 0, obj ถูก dealloc
// อย่าเรียก retainCount หลังจากนี้ - dangling pointer!
```

**หมายเหตุ**: อย่าใช้ `retainCount` ใน production code เพราะค่าที่ได้อาจไม่ตรงกับความเป็นจริงเนื่องจาก autorelease pools

---

## 19.2 retain (count +1)

### การ retain Object

`retain` บอก runtime ว่า "ฉันต้องการใช้ object นี้ ห้ามทำลาย":

```objc
// Car.h
@interface Car : NSObject

@property (nonatomic, retain) NSString *model;  // retain ใน MRC
@property (nonatomic, retain) NSString *color;

- (id)initWithModel:(NSString *)model color:(NSString *)color;

@end
```

```objc
// Car.m
@implementation Car

- (id)initWithModel:(NSString *)aModel color:(NSString *)aColor {
    self = [super init];
    if (self) {
        // retain object ที่ต้องการเก็บไว้
        _model = [aModel retain];
        _color = [aColor retain];
    }
    return self;
}

- (void)dealloc {
    // release ทุก object ที่เรา retain
    [_model release];
    [_color release];
    [super dealloc];  // สำคัญ! ต้องเรียก super dealloc เสมอ
}

@end
```

### ตัวอย่างการ retain ใน Method

```objc
// ❌ ผิด: ไม่ retain, อาจเกิด dangling pointer
- (void)setOwner:(Person *)owner {
    _owner = owner;  // ไม่มี retain, ถ้า owner ถูก release ข้างนอก _owner กลายเป็น dangling pointer
}

// ✅ ถูกต้อง: retain ก่อนเก็บ
- (void)setOwner:(Person *)owner {
    [owner retain];
    [_owner release];  // release อันเก่าก่อน
    _owner = owner;
}
```

---

## 19.3 release (count -1)

### การ release Object

`release` บอก runtime ว่า "ฉันไม่ต้องการ object นี้แล้ว":

```objc
// ตัวอย่าง lifecycle ของ object
- (void)processData {
    // alloc-init สร้าง object, retain count = 1
    NSMutableArray *data = [[NSMutableArray alloc] init];
    
    [data addObject:@"item1"];
    [data addObject:@"item2"];
    
    // ทำงานกับ data...
    NSLog(@"Count: %lu", (unsigned long)data.count);
    
    // เสร็จแล้ว release
    [data release];
    // retain count = 0, data ถูก dealloc
    // data ยังชี้ไป address เดิม แต่ memory ถูก free แล้ว (dangling pointer)
    data = nil;  // ✅ ดีกว่า: set เป็น nil เพื่อป้องกัน dangling pointer
}
```

### Release ใน Different Scenarios

```objc
// Scenario 1: Method ที่สร้าง object เองต้อง release เอง
- (void)createAndUse {
    NSString *str = [[NSString alloc] initWithString:@"Hello"];
    NSLog(@"%@", str);
    [str release];  // ✅ เราสร้าง เราต้อง release
}

// Scenario 2: Object ที่ได้มาจากข้างนอก ไม่ต้อง release (ถ้าไม่ได้ retain)
- (void)useExternalObject:(NSString *)str {
    NSLog(@"%@", str);
    // ❌ อย่า release ที่นี่ เราไม่ได้ retain มัน
}

// Scenario 3: ถ้า retain ก็ต้อง release
- (void)cacheObject:(NSString *)str {
    [str retain];  // เราเพิ่ม count
    _cachedString = str;
    // ... ต้องมี release ที่ไหนสักที่
}

- (void)clearCache {
    [_cachedString release];
    _cachedString = nil;
}
```

---

## 19.4 autorelease

### autorelease คืออะไร

`autorelease` เลื่อน release ออกไปจนกระทั่ง autorelease pool ถูก drain:

```objc
// ไม่ใช้ autorelease - caller ต้อง release
- (NSString *)createString {
    NSString *str = [[NSString alloc] initWithFormat:@"Count: %d", _count];
    return str;  // retain count = 1, caller ต้อง release!
}

// ใช้ autorelease - ง่ายกว่า, caller ไม่ต้อง release
- (NSString *)createStringAutorelease {
    NSString *str = [[NSString alloc] initWithFormat:@"Count: %d", _count];
    return [str autorelease];  // จะถูก release เมื่อ pool drain
}
```

```objc
// การเรียกใช้
NSString *result = [obj createStringAutorelease];
// ใช้ result ได้อย่างอิสระ
NSLog(@"%@", result);
// ไม่ต้อง release result
// result จะถูก release เมื่อ autorelease pool drain
```

---

## 19.5 @autoreleasepool

### Autorelease Pool คืออะไร

Autorelease pool เก็บรวบรวม objects ที่ถูก autorelease ไว้ แล้ว release ทั้งหมดพร้อมกันเมื่อ pool ถูก drain:

```objc
// การสร้าง autorelease pool แบบ modern
@autoreleasepool {
    NSString *str1 = [NSString stringWithFormat:@"Hello %d", 1];
    NSString *str2 = [NSString stringWithFormat:@"World %d", 2];
    // str1 และ str2 ถูก autorelease อยู่ใน pool นี้
    NSLog(@"%@ %@", str1, str2);
}
// ออกจาก block: pool drain, str1 และ str2 ถูก release
```

### Nested Autorelease Pools

```objc
@autoreleasepool {
    // Outer pool
    NSString *outer = [NSString stringWithString:@"Outer"];
    
    @autoreleasepool {
        // Inner pool
        NSString *inner = [NSString stringWithString:@"Inner"];
        // inner อยู่ใน inner pool
    }
    // inner pool drain: inner ถูก release
    // outer ยังคงอยู่
    
    NSLog(@"%@", outer);
}
// outer pool drain: outer ถูก release
```

### ใช้ @autoreleasepool ใน Loops

```objc
// ❌ ปัญหา: memory สะสมในลูป
for (int i = 0; i < 1000000; i++) {
    NSString *str = [NSString stringWithFormat:@"Item %d", i];
    // str ถูก autorelease แต่ pool ยังไม่ drain
    // memory เพิ่มขึ้นเรื่อยๆ จนกว่า run loop จะรอบใหม่
}

// ✅ ถูกต้อง: drain pool ในแต่ละ iteration
for (int i = 0; i < 1000000; i++) {
    @autoreleasepool {
        NSString *str = [NSString stringWithFormat:@"Item %d", i];
        // ทำงานกับ str
    }
    // pool drain ทุก iteration, memory ถูก reclaim
}
```

### Main Function และ Autorelease Pool

```objc
// main.m
int main(int argc, char *argv[]) {
    @autoreleasepool {
        // ทุก autorelease object ใน main thread จะอยู่ใน pool นี้
        return UIApplicationMain(argc, argv, nil, NSStringFromClass([AppDelegate class]));
    }
    // pool drain เมื่อ app terminate
}
```

---

## 19.6 alloc-init Pattern

### Pattern มาตรฐานในการสร้าง Object

```objc
// Pattern พื้นฐาน
ClassName *obj = [[ClassName alloc] init];
// alloc: จัดสรร memory, retain count = 1
// init: initialize state
// ต้อง release เมื่อเสร็จ
[obj release];
```

### Custom Initializers

```objc
// Person.h
@interface Person : NSObject

@property (nonatomic, retain) NSString *name;
@property (nonatomic, assign) NSInteger age;

- (id)initWithName:(NSString *)name age:(NSInteger)age;

@end
```

```objc
// Person.m
@implementation Person

- (id)init {
    return [self initWithName:@"Unknown" age:0];
}

- (id)initWithName:(NSString *)name age:(NSInteger)age {
    self = [super init];
    if (self) {
        _name = [name retain];   // retain string
        _age = age;              // assign primitive
    }
    return self;
}

- (void)dealloc {
    [_name release];  // release ทุก retained object
    [super dealloc];
}

@end
```

```objc
// การใช้งาน
Person *person = [[Person alloc] initWithName:@"สมชาย" age:30];
NSLog(@"Name: %@, Age: %ld", person.name, (long)person.age);
[person release];
person = nil;
```

### Designated Initializer

```objc
// Vehicle.h
@interface Vehicle : NSObject

- (id)initWithMake:(NSString *)make
             model:(NSString *)model
              year:(NSInteger)year;

// Designated initializer

@end
```

```objc
// Vehicle.m
@implementation Vehicle

// Designated initializer
- (id)initWithMake:(NSString *)make model:(NSString *)model year:(NSInteger)year {
    self = [super init];
    if (self) {
        _make = [make retain];
        _model = [model retain];
        _year = year;
    }
    return self;
}

// Convenience initializer เรียก designated initializer
- (id)initWithMake:(NSString *)make model:(NSString *)model {
    return [self initWithMake:make model:model year:2024];
}

// init ก็เรียก designated initializer
- (id)init {
    return [self initWithMake:@"Unknown" model:@"Unknown" year:0];
}

- (void)dealloc {
    [_make release];
    [_model release];
    [super dealloc];
}

@end
```

---

## 19.7 The Ownership Rules (NARC Rule)

NARC เป็นตัวย่อที่ช่วยจำ rule ของ memory ownership:

- **N** - `new` (alloc, new, copy, mutableCopy)
- **A** - `alloc`
- **R** - `retain`
- **C** - `copy`

### กฎ: ถ้าใช้ N, A, R, หรือ C เพื่อได้ object มา → ต้อง release มัน

```objc
- (void)demonstrateNARCRule {
    // N - new
    NSObject *obj1 = [NSObject new];  // retain count = 1
    [obj1 release];  // ✅ ต้อง release เพราะใช้ new
    
    // A - alloc
    NSString *str1 = [[NSString alloc] initWithString:@"Hello"];  // retain count = 1
    [str1 release];  // ✅ ต้อง release เพราะใช้ alloc
    
    // R - retain
    NSString *existingStr = @"World";  // literal, ไม่ต้อง release
    NSString *str2 = [existingStr retain];  // retain count เพิ่ม
    [str2 release];  // ✅ ต้อง release เพราะใช้ retain
    
    // C - copy
    NSMutableString *mutable = [NSMutableString stringWithString:@"Test"];
    NSString *str3 = [mutable copy];  // retain count = 1
    [str3 release];  // ✅ ต้อง release เพราะใช้ copy
}

- (void)demonstrateNoOwnership {
    // ไม่ใช้ NARC - ไม่ต้อง release
    NSString *str = [NSString stringWithString:@"Hello"];  // autorelease, ไม่ต้อง release
    NSArray *arr = [NSArray arrayWithObject:@"item"];      // autorelease, ไม่ต้อง release
    NSDate *now = [NSDate date];                           // autorelease, ไม่ต้อง release
    
    NSLog(@"%@ %@ %@", str, arr, now);
    // ❌ อย่า release str, arr, now - เราไม่ได้ own มัน
}
```

---

## 19.8 Convenience Constructors และ autorelease

### Convenience Constructors

Class methods ที่ขึ้นต้นด้วยชื่อ class (ไม่ใช่ `alloc`/`new`) มักจะ return autoreleased objects:

```objc
// NSString convenience constructors
NSString *str1 = [NSString stringWithFormat:@"Hello %@", name];  // autoreleased
NSString *str2 = [NSString stringWithContentsOfFile:@"/path/to/file" 
                                           encoding:NSUTF8StringEncoding 
                                              error:nil];  // autoreleased

// NSArray
NSArray *arr1 = [NSArray arrayWithObjects:@"a", @"b", @"c", nil];  // autoreleased
NSArray *arr2 = [NSArray arrayWithArray:otherArray];  // autoreleased

// NSDictionary
NSDictionary *dict = [NSDictionary dictionaryWithObject:@"value" forKey:@"key"];  // autoreleased
```

### สร้าง Convenience Constructor ของตัวเอง

```objc
// Car.h
@interface Car : NSObject

@property (nonatomic, retain) NSString *make;
@property (nonatomic, retain) NSString *model;
@property (nonatomic, assign) NSInteger year;

+ (id)carWithMake:(NSString *)make model:(NSString *)model year:(NSInteger)year;
- (id)initWithMake:(NSString *)make model:(NSString *)model year:(NSInteger)year;

@end
```

```objc
// Car.m
@implementation Car

// Convenience constructor - returns autoreleased object
+ (id)carWithMake:(NSString *)make model:(NSString *)model year:(NSInteger)year {
    // alloc-init แล้ว autorelease
    return [[[self alloc] initWithMake:make model:model year:year] autorelease];
    //                                                               ^^^^^^^^^^^
    // สำคัญ! เพิ่ม autorelease เพื่อให้ caller ไม่ต้อง release
}

// Designated initializer
- (id)initWithMake:(NSString *)make model:(NSString *)model year:(NSInteger)year {
    self = [super init];
    if (self) {
        _make = [make retain];
        _model = [model retain];
        _year = year;
    }
    return self;
}

- (void)dealloc {
    [_make release];
    [_model release];
    [super dealloc];
}

@end
```

```objc
// การใช้งาน
// ไม่ต้อง release
Car *car1 = [Car carWithMake:@"Toyota" model:@"Camry" year:2023];

// ต้อง release
Car *car2 = [[Car alloc] initWithMake:@"Honda" model:@"Civic" year:2023];
[car2 release];
```

---

## 19.9 copy vs mutableCopy

### copy - สร้าง immutable copy

```objc
NSMutableString *original = [NSMutableString stringWithString:@"Hello"];

// copy สร้าง NSString (immutable)
NSString *immutableCopy = [original copy];
// immutableCopy เป็น NSString
// [immutableCopy appendString:@" World"];  // Error! NSString ไม่มี appendString:

// ต้อง release เพราะใช้ copy (NARC rule)
[immutableCopy release];
```

### mutableCopy - สร้าง mutable copy

```objc
NSString *original = @"Hello";

// mutableCopy สร้าง NSMutableString
NSMutableString *mutableCopy = [original mutableCopy];
[mutableCopy appendString:@" World"];
NSLog(@"%@", mutableCopy);  // Hello World

// ต้อง release เพราะใช้ mutableCopy (NARC rule)
[mutableCopy release];
```

### Shallow vs Deep Copy กับ Collections

```objc
NSArray *original = @[@[@1, @2], @[@3, @4]];

// copy - shallow copy: copy แค่ array ชั้นนอก
NSArray *shallowCopy = [original copy];
// shallowCopy เป็น array ใหม่ แต่ elements เป็น pointer เดิม

// Deep copy ต้องทำเอง
NSMutableArray *deepCopy = [NSMutableArray array];
for (id item in original) {
    if ([item respondsToSelector:@selector(mutableCopy)]) {
        [deepCopy addObject:[item mutableCopy]];
    }
}

[shallowCopy release];
// deepCopy เป็น autoreleased (จาก [NSMutableArray array])
// inner arrays ที่ mutableCopy ต้อง release เอง ❌
// ต้องระวัง! mutableCopy ใน loop ทำให้มี leak
```

```objc
// ✅ ถูกต้อง: Deep copy ด้วย archiving
NSArray *deepCopyCorrect = [NSKeyedUnarchiver unarchiveObjectWithData:
                            [NSKeyedArchiver archivedDataWithRootObject:original]];
// deepCopyCorrect เป็น autoreleased
```

---

## 19.10 Memory Leaks และ Dangling Pointers

### Memory Leak

Memory leak เกิดเมื่อ retain count ไม่เคยถึง 0 ทำให้ object ไม่ถูก deallocate:

```objc
// ❌ Memory Leak: alloc แต่ไม่ release
- (void)memoryLeakExample {
    for (int i = 0; i < 1000; i++) {
        NSString *str = [[NSString alloc] initWithFormat:@"Item %d", i];
        // ใช้ str...
        // ลืม release str!  ← LEAK!
    }
}

// ✅ ถูกต้อง
- (void)noLeakExample {
    for (int i = 0; i < 1000; i++) {
        NSString *str = [[NSString alloc] initWithFormat:@"Item %d", i];
        // ใช้ str...
        [str release];  // ← ต้องมี
    }
}
```

### Leak จาก Property Setter ที่ไม่ถูกต้อง

```objc
// ❌ Memory Leak ใน setter
@implementation BadObject

- (void)setName:(NSString *)name {
    _name = [name retain];  // เก่า _name ไม่ได้ release!
}

// ✅ ถูกต้อง
- (void)setName:(NSString *)name {
    [name retain];         // retain อันใหม่ก่อน
    [_name release];       // release อันเก่า
    _name = name;          // เก็บ pointer
}

// หรือแบบนี้ก็ได้
- (void)setName:(NSString *)name {
    if (_name != name) {   // check ก่อนว่าไม่ใช่ object เดิม
        [_name release];   // release อันเก่า
        _name = [name retain]; // retain อันใหม่
    }
}

@end
```

### Dangling Pointer

Dangling pointer คือ pointer ที่ชี้ไปยัง memory ที่ถูก free แล้ว:

```objc
// ❌ Dangling pointer
NSString *str = [[NSString alloc] initWithString:@"Hello"];
[str release];
// str ยังชี้ไปยัง memory เดิม แต่ memory นั้นถูก free แล้ว
NSLog(@"%@", str);  // CRASH! หรือ garbage data

// ✅ ถูกต้อง: ตั้งเป็น nil หลัง release
NSString *str2 = [[NSString alloc] initWithString:@"World"];
[str2 release];
str2 = nil;  // ← ป้องกัน dangling pointer
NSLog(@"%@", str2);  // ปลอดภัย: พิมพ์ "(null)"
```

### Over-Release

```objc
// ❌ Over-release: release มากกว่าที่ควร
NSString *str = [NSString stringWithString:@"Hello"];  // autoreleased
[str release];  // CRASH! release object ที่เราไม่ได้ own

// ❌ Over-release อีกแบบ
NSString *str2 = [[NSString alloc] initWithString:@"World"];
[str2 release];
[str2 release];  // CRASH! release สองครั้ง
```

---

## 19.11 NSZombie สำหรับ Debugging

### เปิดใช้งาน NSZombie

NSZombie เป็น debug tool ที่แปลง deallocated object ให้เป็น "zombie" แทนที่จะ free memory จริงๆ เมื่อมีการส่ง message ไปยัง zombie จะ crash พร้อม error message ที่ชัดเจน

**วิธีเปิด NSZombie ใน Xcode:**
1. Product → Scheme → Edit Scheme
2. เลือก Run
3. Diagnostics tab
4. เปิด "Enable Zombie Objects"

หรือเพิ่ม environment variable `NSZombieEnabled = YES`

### ตัวอย่างการใช้ NSZombie

```objc
// ❌ Bug: ส่ง message ไปยัง released object
- (void)demonstrateBug {
    NSString *str = [[NSString alloc] initWithString:@"Hello"];
    [str release];
    
    // ไม่ได้ตั้งเป็น nil หลัง release
    // ต่อมาใน code ส่ง message ไปยัง str ที่ released แล้ว
    NSLog(@"Length: %lu", (unsigned long)str.length);
    // โดยปกติ: crash แบบ mysterious หรือ garbage
    // ด้วย NSZombie: crash พร้อม message:
    // "message sent to deallocated instance 0x..."
}
```

```objc
// หลังจาก enable NSZombie และ run:
// *** -[NSString length]: message sent to deallocated instance 0x100206b60
// ทำให้รู้ว่า object ไหนถูก over-released
```

### ใช้ malloc_history และ Instruments

```bash
# ดู allocation history ของ object ที่ address เฉพาะ
malloc_history <pid> <address>
```

---

## 19.12 retain/release Rules สำหรับ Properties

### Synthesized Setter Patterns ใน MRC

เมื่อประกาศ property ด้วย `retain` attribute ใน MRC, synthesized setter จะทำตาม pattern ที่ถูกต้อง:

```objc
// เมื่อประกาศ:
@property (nonatomic, retain) NSString *name;

// Synthesized setter ทำงานแบบนี้ (เทียบเท่า):
- (void)setName:(NSString *)name {
    if (_name != name) {
        [_name release];
        _name = [name retain];
    }
}

// Synthesized getter:
- (NSString *)name {
    return _name;
}
```

### Manual Setter Patterns

```objc
// Pattern 1: Standard retain pattern
- (void)setDelegate:(id<MyDelegate>)delegate {
    if (_delegate != delegate) {
        [_delegate release];
        _delegate = [delegate retain];
    }
}

// Pattern 2: สำหรับ assign (primitives หรือ weak refs)
- (void)setAge:(NSInteger)age {
    _age = age;  // ไม่ retain primitives
}

// Pattern 3: สำหรับ copy
- (void)setTitle:(NSString *)title {
    if (_title != title) {
        [_title release];
        _title = [title copy];  // copy แทน retain
    }
}
```

---

## 19.13 Setter Patterns สำหรับ Retained Objects

### Full Example: Custom Setters

```objc
// Document.h
@interface Document : NSObject

@property (nonatomic, retain) NSString *title;
@property (nonatomic, retain) NSMutableArray *pages;
@property (nonatomic, assign) BOOL isDirty;
@property (nonatomic, copy) NSString *author;  // copy สำหรับ string

@end
```

```objc
// Document.m
@implementation Document

- (id)init {
    self = [super init];
    if (self) {
        _title = [[NSString alloc] initWithString:@"Untitled"];
        _pages = [[NSMutableArray alloc] init];
        _isDirty = NO;
        _author = [[NSString alloc] init];
    }
    return self;
}

// Custom setter สำหรับ retain property
- (void)setTitle:(NSString *)title {
    if (_title != title) {
        [_title release];
        _title = [title retain];
        _isDirty = YES;  // side effect
    }
}

// Custom setter สำหรับ copy property
- (void)setAuthor:(NSString *)author {
    if (_author != author) {
        [_author release];
        _author = [author copy];  // copy ไม่ใช่ retain
    }
}

// Custom setter สำหรับ array - replace ทั้ง array
- (void)setPages:(NSMutableArray *)pages {
    if (_pages != pages) {
        [_pages release];
        _pages = [pages retain];
        _isDirty = YES;
    }
}

- (void)dealloc {
    [_title release];
    [_pages release];
    [_author release];
    [super dealloc];
}

@end
```

---

## 19.14 Complete Example ด้วย Proper Memory Management

### ตัวอย่างโปรแกรม Address Book แบบ MRC

```objc
// Contact.h
@interface Contact : NSObject

@property (nonatomic, retain) NSString *firstName;
@property (nonatomic, retain) NSString *lastName;
@property (nonatomic, retain) NSString *phoneNumber;
@property (nonatomic, retain) NSString *email;
@property (nonatomic, retain) NSMutableArray *addresses;

+ (id)contactWithFirstName:(NSString *)firstName
                  lastName:(NSString *)lastName
               phoneNumber:(NSString *)phoneNumber;

- (id)initWithFirstName:(NSString *)firstName
               lastName:(NSString *)lastName
            phoneNumber:(NSString *)phoneNumber;

- (void)addAddress:(NSString *)address;
- (NSString *)fullName;

@end
```

```objc
// Contact.m
@implementation Contact

+ (id)contactWithFirstName:(NSString *)firstName
                  lastName:(NSString *)lastName
               phoneNumber:(NSString *)phoneNumber {
    return [[[self alloc] initWithFirstName:firstName
                                   lastName:lastName
                                phoneNumber:phoneNumber] autorelease];
}

- (id)initWithFirstName:(NSString *)firstName
               lastName:(NSString *)lastName
            phoneNumber:(NSString *)phoneNumber {
    self = [super init];
    if (self) {
        _firstName = [firstName retain];
        _lastName = [lastName retain];
        _phoneNumber = [phoneNumber retain];
        _email = nil;  // nil เริ่มต้น
        _addresses = [[NSMutableArray alloc] init];
    }
    return self;
}

- (void)setFirstName:(NSString *)firstName {
    if (_firstName != firstName) {
        [_firstName release];
        _firstName = [firstName retain];
    }
}

- (void)setLastName:(NSString *)lastName {
    if (_lastName != lastName) {
        [_lastName release];
        _lastName = [lastName retain];
    }
}

- (void)setPhoneNumber:(NSString *)phoneNumber {
    if (_phoneNumber != phoneNumber) {
        [_phoneNumber release];
        _phoneNumber = [phoneNumber retain];
    }
}

- (void)setEmail:(NSString *)email {
    if (_email != email) {
        [_email release];
        _email = [email retain];
    }
}

- (void)addAddress:(NSString *)address {
    [_addresses addObject:address];  // NSMutableArray retain เอง
}

- (NSString *)fullName {
    return [NSString stringWithFormat:@"%@ %@", _firstName, _lastName];
    // stringWithFormat: returns autoreleased object, caller ไม่ต้อง release
}

- (NSString *)description {
    return [NSString stringWithFormat:@"Contact: %@ - %@", self.fullName, _phoneNumber];
}

- (void)dealloc {
    // Release ทุก retained property ตามลำดับ
    [_firstName release];  _firstName = nil;
    [_lastName release];   _lastName = nil;
    [_phoneNumber release]; _phoneNumber = nil;
    [_email release];      _email = nil;
    [_addresses release];  _addresses = nil;
    
    [super dealloc];  // ต้องเรียก super dealloc เสมอ!
}

@end
```

```objc
// AddressBook.h
@interface AddressBook : NSObject

- (void)addContact:(Contact *)contact;
- (void)removeContact:(Contact *)contact;
- (Contact *)findContactWithLastName:(NSString *)lastName;
- (NSArray *)allContacts;

@end
```

```objc
// AddressBook.m
@implementation AddressBook {
    NSMutableArray *_contacts;
}

- (id)init {
    self = [super init];
    if (self) {
        _contacts = [[NSMutableArray alloc] init];
    }
    return self;
}

- (void)addContact:(Contact *)contact {
    if (contact && ![_contacts containsObject:contact]) {
        [_contacts addObject:contact];  // array retain contact
    }
}

- (void)removeContact:(Contact *)contact {
    [_contacts removeObject:contact];  // array release contact
}

- (Contact *)findContactWithLastName:(NSString *)lastName {
    for (Contact *contact in _contacts) {
        if ([contact.lastName isEqualToString:lastName]) {
            return contact;  // ไม่ต้อง retain เพราะแค่ return reference
        }
    }
    return nil;
}

- (NSArray *)allContacts {
    return [[_contacts copy] autorelease];  // return immutable copy, autoreleased
}

- (void)dealloc {
    [_contacts release];  // release array, array จะ release contacts ด้วย
    [super dealloc];
}

@end
```

```objc
// การใช้งานแบบ MRC
int main() {
    @autoreleasepool {
        // สร้าง Address Book
        AddressBook *book = [[AddressBook alloc] init];
        
        // สร้าง Contacts ด้วย convenience constructor (autoreleased)
        Contact *contact1 = [Contact contactWithFirstName:@"สมชาย"
                                                 lastName:@"ใจดี"
                                              phoneNumber:@"081-234-5678"];
        contact1.email = @"somchai@example.com";
        [contact1 addAddress:@"123 ถนนสุขุมวิท กรุงเทพ"];
        
        // สร้าง Contact ด้วย alloc-init (ต้อง release)
        Contact *contact2 = [[Contact alloc] initWithFirstName:@"สมหญิง"
                                                       lastName:@"รักดี"
                                                   phoneNumber:@"082-345-6789"];
        
        [book addContact:contact1];
        [book addContact:contact2];
        
        // contact1 ไม่ต้อง release (autoreleased)
        // contact2 ต้อง release (alloc-init)
        [contact2 release];
        contact2 = nil;
        
        // ค้นหา contact
        Contact *found = [book findContactWithLastName:@"ใจดี"];
        if (found) {
            NSLog(@"Found: %@", found.description);
        }
        
        // แสดงรายชื่อทั้งหมด
        NSArray *allContacts = [book allContacts];
        // allContacts เป็น autoreleased copy
        for (Contact *c in allContacts) {
            NSLog(@"%@", c.description);
        }
        
        // release book
        [book release];
        book = nil;
        
        // ตอน autorelease pool drain:
        // - allContacts array จะถูก release
        // - contact1 จะถูก release (และ dealloc เพราะ book ก็ release แล้ว)
    }
    
    return 0;
}
```

---

## 19.15 Common Mistakes และวิธีหลีกเลี่ยง

### Mistake 1: ลืม Release

```objc
// ❌ Leak
- (void)processImage {
    UIImage *image = [[UIImage alloc] initWithContentsOfFile:@"/path/to/image.jpg"];
    // process image...
    // ลืม release!
}

// ✅ ถูกต้อง
- (void)processImage {
    UIImage *image = [[UIImage alloc] initWithContentsOfFile:@"/path/to/image.jpg"];
    // process image...
    [image release];
    image = nil;
}
```

### Mistake 2: Release Object ที่ไม่ได้ Own

```objc
// ❌ Over-release
- (void)handleResult:(NSArray *)result {
    // result เป็น parameter, เราไม่ได้ retain มัน
    [result release];  // CRASH!
}

// ✅ ถูกต้อง
- (void)handleResult:(NSArray *)result {
    // ถ้าต้องการเก็บไว้ ให้ retain
    _savedResult = [result retain];
    // ไม่งั้นแค่ใช้โดยไม่ต้อง release
}
```

### Mistake 3: Setter ที่ผิดพลาด

```objc
// ❌ Memory leak ใน setter
- (void)setDelegate:(id<Delegate>)delegate {
    _delegate = [delegate retain];  // _delegate เก่าไม่ได้ release!
}

// ❌ Potential crash: release ก่อน retain เมื่อ delegate == _delegate
- (void)setDelegate:(id<Delegate>)delegate {
    [_delegate release];  // ถ้า delegate == _delegate, release แล้ว...
    _delegate = [delegate retain];  // ...แล้ว retain object ที่อาจ dealloc แล้ว
}

// ✅ ถูกต้อง
- (void)setDelegate:(id<Delegate>)delegate {
    if (_delegate != delegate) {
        [_delegate release];
        _delegate = [delegate retain];
    }
}
```

### Mistake 4: dealloc ไม่ครบ

```objc
// ❌ Memory leak ใน dealloc
- (void)dealloc {
    [_name release];
    // ลืม release _email, _address!
    [super dealloc];
}

// ✅ ถูกต้อง: release ทุก retained property
- (void)dealloc {
    [_name release];
    [_email release];
    [_address release];
    [_phoneNumber release];
    [super dealloc];
}
```

### Mistake 5: ลืม [super dealloc]

```objc
// ❌ Memory leak: ลืม super dealloc
- (void)dealloc {
    [_name release];
    // ลืม [super dealloc]!
}

// ✅ ถูกต้อง
- (void)dealloc {
    [_name release];
    [super dealloc];  // ต้องเรียกเสมอ และต้องเป็น statement สุดท้าย
}
```

### Mistake 6: Retain Cycle

```objc
// ❌ Retain cycle
@interface Parent : NSObject
@property (nonatomic, retain) Child *child;  // Parent retain Child
@end

@interface Child : NSObject
@property (nonatomic, retain) Parent *parent;  // Child retain Parent = CYCLE!
@end

// ✅ ถูกต้อง: ใช้ assign สำหรับ backward reference ใน MRC
@interface Child : NSObject
@property (nonatomic, assign) Parent *parent;  // ไม่ retain = ไม่มี cycle
@end
```

### Mistake 7: ใช้ Object หลัง Release

```objc
// ❌ Use after free
NSString *str = [[NSString alloc] initWithString:@"Hello"];
[str release];
NSLog(@"%lu", (unsigned long)str.length);  // CRASH หรือ garbage

// ✅ ถูกต้อง
NSString *str2 = [[NSString alloc] initWithString:@"Hello"];
[str2 release];
str2 = nil;
if (str2) {  // nil check ก่อน
    NSLog(@"%lu", (unsigned long)str2.length);  // ไม่เข้า block
}
```

---

## แบบฝึกหัด (Practice Exercises)

### ข้อ 1: Stack Data Structure แบบ MRC

สร้าง class `Stack` ที่จัดการ memory ด้วย MRC อย่างถูกต้อง:

```objc
// เฉลย
@interface Stack : NSObject {
    NSMutableArray *_items;
}

- (void)push:(id)item;
- (id)pop;
- (id)peek;
- (NSUInteger)count;
- (BOOL)isEmpty;

@end

@implementation Stack

- (id)init {
    self = [super init];
    if (self) {
        _items = [[NSMutableArray alloc] init];
    }
    return self;
}

- (void)push:(id)item {
    [_items addObject:item];  // array retains item
}

- (id)pop {
    if ([_items count] == 0) return nil;
    
    id top = [[_items lastObject] retain];  // retain ก่อน remove
    [_items removeLastObject];              // array releases item
    return [top autorelease];               // autorelease เพื่อให้ caller ใช้ได้
}

- (id)peek {
    return [_items lastObject];  // return without retaining, autoreleased by array
}

- (NSUInteger)count { return [_items count]; }
- (BOOL)isEmpty { return [_items count] == 0; }

- (void)dealloc {
    [_items release];
    [super dealloc];
}

@end
```

### ข้อ 2: Observer Pattern ด้วย MRC

```objc
// เฉลย
@protocol EventObserver <NSObject>
- (void)didReceiveEvent:(NSString *)event data:(id)data;
@end

@interface EventBus : NSObject

+ (id)sharedBus;
- (void)subscribe:(id<EventObserver>)observer forEvent:(NSString *)event;
- (void)unsubscribe:(id<EventObserver>)observer forEvent:(NSString *)event;
- (void)emit:(NSString *)event data:(id)data;

@end

@implementation EventBus {
    NSMutableDictionary *_observers;
}

static EventBus *_sharedBus = nil;

+ (id)sharedBus {
    @synchronized(self) {
        if (!_sharedBus) {
            _sharedBus = [[EventBus alloc] init];
            // Note: singleton ไม่ถูก release
        }
    }
    return _sharedBus;
}

- (id)init {
    self = [super init];
    if (self) {
        _observers = [[NSMutableDictionary alloc] init];
    }
    return self;
}

- (void)subscribe:(id<EventObserver>)observer forEvent:(NSString *)event {
    NSMutableArray *list = [_observers objectForKey:event];
    if (!list) {
        list = [NSMutableArray array];
        [_observers setObject:list forKey:event];
    }
    if (![list containsObject:observer]) {
        [list addObject:observer];  // retain observer
    }
}

- (void)unsubscribe:(id<EventObserver>)observer forEvent:(NSString *)event {
    NSMutableArray *list = [_observers objectForKey:event];
    [list removeObject:observer];  // release observer
}

- (void)emit:(NSString *)event data:(id)data {
    NSArray *observers = [[_observers objectForKey:event] copy];
    for (id<EventObserver> obs in observers) {
        [obs didReceiveEvent:event data:data];
    }
    [observers release];
}

- (void)dealloc {
    [_observers release];
    [super dealloc];
}

@end
```

### ข้อ 3: Binary Tree Node ด้วย MRC

```objc
// เฉลย
@interface BinaryTreeNode : NSObject

@property (nonatomic, assign) NSInteger value;
@property (nonatomic, retain) BinaryTreeNode *left;
@property (nonatomic, retain) BinaryTreeNode *right;
@property (nonatomic, assign) BinaryTreeNode *parent;  // weak ใน MRC

+ (id)nodeWithValue:(NSInteger)value;

@end

@implementation BinaryTreeNode

+ (id)nodeWithValue:(NSInteger)value {
    BinaryTreeNode *node = [[[BinaryTreeNode alloc] init] autorelease];
    node.value = value;
    return node;
}

- (void)setLeft:(BinaryTreeNode *)left {
    if (_left != left) {
        [_left release];
        _left = [left retain];
        _left.parent = self;
    }
}

- (void)setRight:(BinaryTreeNode *)right {
    if (_right != right) {
        [_right release];
        _right = [right retain];
        _right.parent = self;
    }
}

- (void)dealloc {
    [_left release];
    [_right release];
    // _parent เป็น assign, ไม่ต้อง release
    [super dealloc];
}

@end
```

### ข้อ 4-10: แบบฝึกหัดเพิ่มเติม

**ข้อ 4**: สร้าง `LRUCache` class ด้วย MRC ที่มี capacity limit และ evict oldest item เมื่อเต็ม

**ข้อ 5**: สร้าง `ObjectPool` class ที่ reuse objects เพื่อประหยัด memory allocation/deallocation

**ข้อ 6**: สร้าง `WeakSet` class ที่เก็บ weak references ไปยัง objects (ใน MRC ใช้ assign) และทำการ cleanup objects ที่ถูก deallocate แล้ว

**ข้อ 7**: แก้ไข memory bugs ต่อไปนี้:
```objc
// Bug 1: 
NSMutableArray *arr = [[NSMutableArray alloc] init];
for (int i = 0; i < 10; i++) {
    NSString *s = [[NSString alloc] initWithFormat:@"%d", i];
    [arr addObject:s];
    // ลืม release s
}
[arr release];

// Bug 2:
@implementation MyClass
- (void)setData:(NSData *)data {
    _data = [data retain];
}
- (void)dealloc {
    [_data release];
    // ลืม super dealloc
}
@end

// Bug 3:
NSString *str = [NSString stringWithFormat:@"Hello"];
NSLog(@"%@", str);
[str release];  // over-release
```

**ข้อ 8**: สร้าง `MemoryPool` allocator ที่จัดสรร memory ด้วยตัวเอง เพื่อลด fragmentation

**ข้อ 9**: เขียน unit tests สำหรับ class ที่ใช้ MRC เพื่อตรวจสอบว่าไม่มี leaks

**ข้อ 10**: Convert class ที่ใช้ MRC ให้กลายเป็น ARC โดยเข้าใจทุก step ที่เปลี่ยนแปลง

---

## สรุป

ในบทนี้เราได้เรียนรู้ Manual Reference Counting (MRC) ครบทุกด้าน:

1. **Reference Counting Concept** - หลักการนับการอ้างอิงและวงจรชีวิตของ object
2. **retain / release** - การเพิ่มและลด retain count
3. **autorelease** - การเลื่อน release ออกไปจนกระทั่ง pool drain
4. **@autoreleasepool** - การสร้างและจัดการ autorelease pools
5. **alloc-init Pattern** - Pattern มาตรฐานในการสร้าง objects
6. **NARC Rule** - N(ew)A(lloc)R(etain)C(opy) คือ rules สำหรับ ownership
7. **Convenience Constructors** - Pattern สำหรับสร้าง autoreleased objects
8. **copy vs mutableCopy** - ความแตกต่างและการใช้งาน
9. **Memory Leaks & Dangling Pointers** - การเข้าใจและป้องกัน
10. **NSZombie** - Tool สำหรับ debug memory problems
11. **Setter Patterns** - Pattern ที่ถูกต้องสำหรับ retained properties
12. **Complete Example** - Address book ที่จัดการ memory ครบถ้วน
13. **Common Mistakes** - ข้อผิดพลาดที่พบบ่อยและวิธีหลีกเลี่ยง

การเข้าใจ MRC เป็นรากฐานสำคัญที่ทำให้เข้าใจ ARC ได้ดีขึ้น ในตอนต่อไปเราจะเรียนรู้ **Automatic Reference Counting (ARC)** ซึ่ง compiler จัดการ retain/release ให้เราอัตโนมัติ
