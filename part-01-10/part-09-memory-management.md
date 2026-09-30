# ตอนที่ 9: Memory Management - การจัดการหน่วยความจำ

## บทนำ

Memory Management (การจัดการหน่วยความจำ) เป็นหนึ่งในหัวข้อที่สำคัญที่สุดในการเขียนโปรแกรม Objective-C การจัดการหน่วยความจำที่ผิดพลาดจะทำให้เกิด memory leaks (รั่วไหล) หรือ crashes ได้ บทนี้จะอธิบายทั้ง MRC (Manual Reference Counting) และ ARC (Automatic Reference Counting) อย่างละเอียด

---

## 9.1 Stack vs Heap Memory

### หน่วยความจำทั้งสองประเภท

```
Stack Memory (หน่วยความจำ Stack):
- จัดการอัตโนมัติโดย compiler
- ตัวแปร local, function parameters
- ขนาดเล็ก แต่เร็วมาก
- ถูกคืนอัตโนมัติเมื่อออกจาก scope

Heap Memory (หน่วยความจำ Heap):
- จัดการด้วยตัวโปรแกรมเอง (alloc/free)
- Objects ที่สร้างด้วย alloc/init
- ขนาดใหญ่กว่า แต่ช้ากว่า
- ต้องคืนเองหรือให้ ARC จัดการ
```

```objc
#import <Foundation/Foundation.h>

void stackDemo() {
    // ตัวแปรเหล่านี้อยู่บน Stack
    int x = 10;                  // Stack
    double pi = 3.14159;         // Stack
    char buffer[100];            // Stack array (ขนาดคงที่)
    
    NSLog(@"Stack variables: x=%d, pi=%.5f", x, pi);
    
    // เมื่อฟังก์ชันจบ ตัวแปรเหล่านี้จะถูกลบออกจาก stack อัตโนมัติ
}

void heapDemo() {
    // Objects สร้างบน Heap ด้วย alloc
    NSString *str = [[NSString alloc] initWithString:@"Hello"];  // Heap
    NSMutableArray *arr = [[NSMutableArray alloc] init];          // Heap
    
    NSLog(@"Heap objects: %@, count=%lu", str, arr.count);
    
    // ด้วย ARC: ตัวแปร str และ arr เป็น strong references
    // เมื่อออกจาก scope ARC จะลด reference count
    // ถ้าเป็น 0 จะ deallocate อัตโนมัติ
}

int main(int argc, const char * argv[]) {
    @autoreleasepool {
        
        NSLog(@"=== Stack vs Heap ===");
        stackDemo();
        heapDemo();
        
        // การใช้ sizeof กับตัวแปร Stack
        int stackArr[10];
        NSLog(@"\nStack array size: %lu bytes", sizeof(stackArr));
        NSLog(@"Stack array elements: %lu", sizeof(stackArr) / sizeof(stackArr[0]));
        
        // Heap allocation
        int *heapArr = malloc(10 * sizeof(int));
        if (heapArr != NULL) {
            for (int i = 0; i < 10; i++) {
                heapArr[i] = i * i;
            }
            NSLog(@"\nHeap array:");
            for (int i = 0; i < 10; i++) {
                NSLog(@"  [%d] = %d", i, heapArr[i]);
            }
            free(heapArr);  // ต้อง free เอง! (C-style memory)
            heapArr = NULL;  // ป้องกัน dangling pointer
        }
        
        NSLog(@"\nMemory hierarchy (เร็ว→ช้า):");
        NSLog(@"1. CPU Registers");
        NSLog(@"2. CPU Cache (L1/L2/L3)");
        NSLog(@"3. RAM (Stack & Heap อยู่ที่นี่)");
        NSLog(@"4. SSD/HDD");
    }
    return 0;
}
```

---

## 9.2 Manual Reference Counting (MRC)

> **หมายเหตุ**: MRC เป็น mode เดิมก่อนมี ARC ในปัจจุบันแทบไม่ใช้แล้ว แต่ต้องเข้าใจเพื่อรู้วิธีทำงานของ ARC

### แนวคิด Reference Counting

```
เมื่อ object ถูกสร้าง: reference count = 1
เมื่อมีใคร retain: reference count + 1
เมื่อมีใคร release: reference count - 1
เมื่อ reference count = 0: object ถูก deallocate
```

### การใช้ retain, release, autorelease

```objc
// ตัวอย่างนี้เป็น MRC (ไม่ใช้ใน ARC)
// ต้อง compile ด้วย flag: -fno-objc-arc

#import <Foundation/Foundation.h>

@interface Dog : NSObject {
    NSString *_name;
}
- (instancetype)initWithName:(NSString *)name;
- (NSString *)name;
- (void)bark;
@end

@implementation Dog

- (instancetype)initWithName:(NSString *)name {
    self = [super init];
    if (self) {
        _name = [name retain];  // retain string
        NSLog(@"[Dog init] '%@' (retainCount: %lu)",
              _name, (unsigned long)[self retainCount]);
    }
    return self;
}

- (NSString *)name {
    return _name;
}

- (void)bark {
    NSLog(@"%@: Woof!", _name);
}

- (void)dealloc {
    NSLog(@"[Dog dealloc] '%@' ถูก deallocate", _name);
    [_name release];  // ต้อง release ทุก object ที่ retain
    [super dealloc];
}

@end

// MRC Example (compile-time flag needed: -fno-objc-arc)
void mrcExample() {
    // สร้าง object: retainCount = 1
    Dog *dog = [[Dog alloc] initWithName:@"Rex"];
    NSLog(@"retainCount หลัง alloc: %lu", (unsigned long)[dog retainCount]);
    
    // retain: retainCount + 1 = 2
    Dog *anotherRef = [dog retain];
    NSLog(@"retainCount หลัง retain: %lu", (unsigned long)[dog retainCount]);
    
    // release: retainCount - 1 = 1
    [anotherRef release];
    NSLog(@"retainCount หลัง release: %lu", (unsigned long)[dog retainCount]);
    
    // release: retainCount - 1 = 0 → deallocate
    [dog release];
    // dog ถูก deallocate แล้ว อย่าใช้ต่อ!
    dog = nil;  // ดีที่สุดคือ set เป็น nil
}
```

### กฎ MRC (Memory Management Rules)

```objc
/*
กฎการจัดการ memory ใน MRC:
1. ถ้าคุณ alloc/new/copy/mutableCopy → คุณต้อง release
2. ถ้าคุณ retain → คุณต้อง release
3. ถ้าคุณไม่ใช่เจ้าของ object → ห้าม release

ชื่อ method ที่บอกว่าคุณเป็นเจ้าของ (ต้อง release):
- alloc
- new
- copy
- mutableCopy

ชื่อ method ที่คืน autorelease object (ไม่ต้อง release):
- ชื่ออื่นทั้งหมด เช่น stringWithFormat:, arrayWithObjects:
*/

// ตัวอย่างที่ถูกต้อง:
NSString *s1 = [[NSString alloc] initWithString:@"Hello"];  // ต้อง release
[s1 release];

NSString *s2 = [NSString stringWithString:@"Hello"];  // autorelease, ไม่ต้อง release

NSMutableArray *arr = [[NSMutableArray alloc] init];  // ต้อง release
[arr release];

// ตัวอย่าง Memory Leak:
// void leak() {
//     NSString *s = [[NSString alloc] initWithString:@"Leak!"];
//     // ไม่ release! → Memory Leak
// }

// ตัวอย่าง Over-release (Crash):
// void overRelease() {
//     NSString *s = [NSString stringWithString:@"Hello"];  // autorelease
//     [s release];  // Error! ไม่ได้ retain ก่อน
// }
```

### autorelease Pool

```objc
#import <Foundation/Foundation.h>

// autorelease: ลงทะเบียน object ให้ถูก release ทีหลัง
// โดย autorelease pool จะ drain เมื่อถึงจุดที่กำหนด

NSString *createString(NSString *prefix, int number) {
    // ส่งคืน autorelease object
    return [NSString stringWithFormat:@"%@-%d", prefix, number];
}

int main(int argc, const char * argv[]) {
    // @autoreleasepool: สร้าง autorelease pool
    // objects ที่ autorelease จะถูก release เมื่อ pool ถูก drain
    @autoreleasepool {
        
        NSLog(@"=== autorelease Pool ===");
        
        // ทุก autorelease object จะถูก retain จนกว่า pool จะ drain
        NSString *str = [NSString stringWithFormat:@"Hello %d", 42];
        NSLog(@"String: %@", str);
        
        // Nested autorelease pools
        for (int i = 0; i < 3; i++) {
            @autoreleasepool {
                // Objects ใน inner pool จะถูก drain เมื่อจบ iteration
                NSString *temp = [NSString stringWithFormat:@"Item %d", i];
                NSLog(@"Temp: %@", temp);
                // temp ถูก release ที่นี่ ไม่สะสมใน outer pool
            }
        }
        
        NSLog(@"\nสำคัญ: ใช้ @autoreleasepool ใน:");
        NSLog(@"1. main() ของโปรแกรม");
        NSLog(@"2. Loop ที่สร้าง objects จำนวนมาก");
        NSLog(@"3. Threads ใหม่ทุกตัว");
        NSLog(@"4. Background operations");
    }
    return 0;
}
```

---

## 9.3 ARC - Automatic Reference Counting

ARC เป็นระบบที่ compiler จัดการ reference counting ให้โดยอัตโนมัติ คุณไม่ต้องเขียน retain/release เอง

### ARC ทำงานอย่างไร

```objc
// สิ่งที่คุณเขียน (ARC):
NSString *str = [[NSString alloc] initWithString:@"Hello"];
// str ไม่ใช้แล้ว

// สิ่งที่ ARC แทรก retain/release ให้:
NSString *str = [[NSString alloc] initWithString:@"Hello"];
// ... ใช้ str ...
[str release];  // ARC แทรกให้อัตโนมัติ
```

```objc
#import <Foundation/Foundation.h>

@interface Person : NSObject
@property (nonatomic, strong) NSString *name;
@property (nonatomic, assign) NSInteger age;

- (instancetype)initWithName:(NSString *)name age:(NSInteger)age;
@end

@implementation Person

- (instancetype)initWithName:(NSString *)name age:(NSInteger)age {
    self = [super init];
    if (self) {
        _name = name;
        _age = age;
        NSLog(@"[+] สร้าง Person: %@", _name);
    }
    return self;
}

- (void)dealloc {
    NSLog(@"[-] deallocate Person: %@", _name);
    // ARC: ไม่ต้องเรียก [super dealloc]
}

- (NSString *)description {
    return [NSString stringWithFormat:@"Person{%@, อายุ %ld}", _name, (long)_age];
}

@end

int main(int argc, const char * argv[]) {
    @autoreleasepool {
        
        NSLog(@"=== ARC Demo ===\n");
        
        NSLog(@"--- Scope 1 เริ่ม ---");
        {
            Person *p1 = [[Person alloc] initWithName:@"สมชาย" age:25];
            NSLog(@"ใช้ p1: %@", p1);
            // p1 ออกจาก scope → ARC เรียก release → dealloc
        }
        NSLog(@"--- Scope 1 จบ ---\n");
        
        NSLog(@"--- Scope 2 เริ่ม ---");
        Person *p2 = nil;
        {
            Person *inner = [[Person alloc] initWithName:@"มณี" age:30];
            p2 = inner;  // p2 เป็น strong reference → retainCount = 2
            NSLog(@"inner และ p2 ชี้ที่เดียวกัน: %@", inner);
            // inner ออกจาก scope → retainCount = 1 (p2 ยังชี้อยู่)
        }
        NSLog(@"p2 ยังมีชีวิตอยู่: %@", p2);
        p2 = nil;  // p2 = nil → retainCount = 0 → dealloc
        NSLog(@"--- Scope 2 จบ ---\n");
        
        NSLog(@"--- Array ของ objects ---");
        NSMutableArray *people = [NSMutableArray array];
        {
            Person *p3 = [[Person alloc] initWithName:@"อนุชา" age:22];
            [people addObject:p3];  // array retain p3 → retainCount = 2
            // p3 ออกจาก scope → retainCount = 1 (array ยังเก็บไว้)
        }
        NSLog(@"Array ยังมี: %@", people[0]);
        [people removeAllObjects];  // array release → retainCount = 0 → dealloc
        NSLog(@"--- Array จบ ---");
    }
    return 0;
}
```

---

## 9.4 Strong vs Weak References

### Strong Reference

```objc
// Strong reference: เพิ่ม reference count
// object จะไม่ถูก deallocate ตราบที่มี strong reference ชี้ถึง

@interface Car : NSObject
@property (nonatomic, strong) NSString *brand;    // strong
@property (nonatomic, strong) NSString *model;    // strong
@property (nonatomic, assign) NSInteger year;     // assign (primitive)
@end

@implementation Car
- (NSString *)description {
    return [NSString stringWithFormat:@"%ld %@ %@", (long)_year, _brand, _model];
}
@end
```

### Weak Reference

```objc
// Weak reference: ไม่เพิ่ม reference count
// ถ้า object ถูก deallocate → weak reference จะเป็น nil อัตโนมัติ
// ใช้เพื่อป้องกัน retain cycle

@interface Owner : NSObject
@property (nonatomic, strong) NSString *name;
@property (nonatomic, strong) NSObject *ownedObject;  // strong
@end

@implementation Owner
@end

@interface Owned : NSObject
@property (nonatomic, weak) Owner *owner;  // weak ป้องกัน retain cycle
@end

@implementation Owned
@end
```

### Unsafe_unretained vs Weak

```objc
// unsafe_unretained: ไม่เพิ่ม count, ไม่ nil อัตโนมัติ
// → Dangling pointer ถ้า object ถูก deallocate!
// ใช้เฉพาะเมื่อจำเป็นจริงๆ

@interface Legacy : NSObject
@property (nonatomic, unsafe_unretained) NSObject *legacyRef;  // อันตราย!
@end

// สรุป:
// strong  = ถือเป็นเจ้าของ, เพิ่ม retain count
// weak    = ไม่ถือเป็นเจ้าของ, ไม่เพิ่ม count, nil ถ้า deallocate
// assign  = สำหรับ primitive types (int, float, BOOL, etc.)
// copy    = copy object เมื่อ assign (ใช้กับ NSString, NSArray เป็นต้น)
// unsafe_unretained = เหมือน weak แต่ไม่ nil อัตโนมัติ (เสี่ยง crash)
```

---

## 9.5 Retain Cycles

Retain Cycle เกิดขึ้นเมื่อ objects สองตัว (หรือมากกว่า) ถือ strong reference ซึ่งกันและกัน ทำให้ reference count ไม่สามารถเป็น 0 ได้ และ object จะไม่ถูก deallocate

### ตัวอย่าง Retain Cycle

```objc
#import <Foundation/Foundation.h>

// BAD EXAMPLE: Retain Cycle
@interface Employee : NSObject
@property (nonatomic, strong) NSString *name;
@property (nonatomic, strong) NSObject *company;  // strong → cycle!
- (instancetype)initWithName:(NSString *)name;
@end

@interface Company : NSObject
@property (nonatomic, strong) NSString *companyName;
@property (nonatomic, strong) NSObject *employee;  // strong → cycle!
- (instancetype)initWithName:(NSString *)name;
@end

@implementation Employee
- (instancetype)initWithName:(NSString *)name {
    self = [super init];
    if (self) { _name = name; }
    return self;
}
- (void)dealloc {
    NSLog(@"Employee '%@' deallocated", _name);
}
@end

@implementation Company
- (instancetype)initWithName:(NSString *)name {
    self = [super init];
    if (self) { _companyName = name; }
    return self;
}
- (void)dealloc {
    NSLog(@"Company '%@' deallocated", _companyName);
}
@end

void retainCycleDemo() {
    Employee *emp = [[Employee alloc] initWithName:@"สมชาย"];
    Company *comp = [[Company alloc] initWithName:@"บริษัท ABC"];
    
    emp.company = comp;   // emp → comp (strong)
    comp.employee = emp;  // comp → emp (strong)
    
    NSLog(@"emp และ comp ถือกันแบบ strong");
    
    // เมื่อออกจาก scope:
    // emp retainCount = 1 (comp ถือ), comp retainCount = 1 (emp ถือ)
    // ไม่มีตัวไหนถึง 0 → ไม่ถูก deallocate! → MEMORY LEAK!
    NSLog(@"dealloc จะไม่ถูกเรียก!");
}

int main(int argc, const char * argv[]) {
    @autoreleasepool {
        NSLog(@"=== Retain Cycle Demo ===");
        retainCycleDemo();
        NSLog(@"(ไม่เห็น dealloc messages ↑ = memory leak)");
    }
    return 0;
}
```

### การแก้ไข Retain Cycle ด้วย weak

```objc
#import <Foundation/Foundation.h>

// GOOD EXAMPLE: แก้ Retain Cycle ด้วย weak
@interface Manager : NSObject
@property (nonatomic, strong) NSString *name;
@property (nonatomic, strong) NSObject *department;  // Manager เป็นเจ้าของ Dept
- (instancetype)initWithName:(NSString *)name;
@end

@interface Department : NSObject
@property (nonatomic, strong) NSString *deptName;
@property (nonatomic, weak) NSObject *manager;  // weak! Dept ไม่เป็นเจ้าของ Manager
- (instancetype)initWithName:(NSString *)name;
@end

@implementation Manager
- (instancetype)initWithName:(NSString *)name {
    self = [super init];
    if (self) { _name = name; }
    return self;
}
- (void)dealloc {
    NSLog(@"Manager '%@' deallocated ✓", _name);
}
@end

@implementation Department
- (instancetype)initWithName:(NSString *)name {
    self = [super init];
    if (self) { _deptName = name; }
    return self;
}
- (void)dealloc {
    NSLog(@"Department '%@' deallocated ✓", _deptName);
}
@end

void fixedCycleDemo() {
    Manager *mgr = [[Manager alloc] initWithName:@"ผู้จัดการ"];
    Department *dept = [[Department alloc] initWithName:@"ฝ่าย IT"];
    
    mgr.department = dept;  // strong: mgr → dept
    dept.manager = mgr;     // weak: dept ⇢ mgr (ไม่เพิ่ม count)
    
    NSLog(@"ตั้งค่า mgr และ dept แบบ break cycle");
    
    // เมื่อออกจาก scope:
    // mgr retainCount = 1 → ออกจาก scope → 0 → dealloc
    //   → mgr.department = nil → dept retainCount = 1 → 0 → dealloc
    NSLog(@"จะเห็น dealloc messages:");
}

int main(int argc, const char * argv[]) {
    @autoreleasepool {
        NSLog(@"=== Fixed Retain Cycle ===");
        fixedCycleDemo();
    }
    return 0;
}
```

### Retain Cycle ใน Blocks

```objc
#import <Foundation/Foundation.h>

@interface Controller : NSObject
@property (nonatomic, strong) NSString *name;
@property (nonatomic, copy) void (^callback)(void);
- (void)setupCallback;
@end

@implementation Controller

- (instancetype)init {
    self = [super init];
    if (self) {
        _name = @"MyController";
    }
    return self;
}

- (void)setupCallbackBAD {
    // BAD: Block capture self → Retain Cycle
    // self strong → block, block strong → self
    self.callback = ^{
        NSLog(@"name: %@", self.name);  // self ถูก capture แบบ strong
    };
}

- (void)setupCallbackGOOD {
    // GOOD: ใช้ weakSelf
    __weak Controller *weakSelf = self;
    self.callback = ^{
        // ใช้ weakSelf แทน self ใน block
        __strong Controller *strongSelf = weakSelf;  // สำคัญ!
        if (strongSelf) {
            NSLog(@"name: %@", strongSelf.name);
        }
    };
}

- (void)dealloc {
    NSLog(@"Controller '%@' deallocated", _name);
}

@end

int main(int argc, const char * argv[]) {
    @autoreleasepool {
        
        NSLog(@"=== Block Retain Cycle ===");
        
        {
            Controller *ctrl = [[Controller alloc] init];
            [ctrl setupCallbackGOOD];
            if (ctrl.callback) {
                ctrl.callback();
            }
            // ctrl ออกจาก scope → dealloc (ถ้าไม่มี cycle)
        }
        NSLog(@"(ควรเห็น dealloc message ↑)");
    }
    return 0;
}
```

---

## 9.6 @autoreleasepool

```objc
#import <Foundation/Foundation.h>

int main(int argc, const char * argv[]) {
    @autoreleasepool {
        
        NSLog(@"=== @autoreleasepool ===");
        
        // 1. Basic usage - wraps all code in main
        // ทุกโปรแกรม Objective-C ต้องมี @autoreleasepool ใน main()
        
        // 2. Memory-intensive loops
        NSLog(@"\n--- Loop with autorelease pool ---");
        
        // BAD: ทุก iteration สะสม autorelease objects
        // for (int i = 0; i < 1000000; i++) {
        //     NSString *s = [NSString stringWithFormat:@"String %d", i];
        //     // ทุก s ถูก autorelease แต่ยังไม่ถูก drain
        //     // memory พุ่งขึ้นสูง!
        // }
        
        // GOOD: drain pool ทุก iteration
        for (int i = 0; i < 10; i++) {
            @autoreleasepool {
                NSString *s = [NSString stringWithFormat:@"String %d", i];
                // s autorelease → ถูก drain เมื่อจบ @autoreleasepool ใน iteration นี้
                if (i % 3 == 0) {
                    NSLog(@"String: %@", s);
                }
            }
            // memory ถูก release แล้ว
        }
        
        // 3. Background threads
        NSLog(@"\n--- Thread with autorelease pool ---");
        dispatch_async(dispatch_get_global_queue(DISPATCH_QUEUE_PRIORITY_DEFAULT, 0), ^{
            @autoreleasepool {  // จำเป็นสำหรับ thread ใหม่
                // operations ที่สร้าง autorelease objects
                for (int i = 0; i < 5; i++) {
                    @autoreleasepool {
                        NSString *result = [NSString stringWithFormat:@"Task %d", i];
                        NSLog(@"Thread: %@", result);
                    }
                }
            }
        });
        
        // รอ thread จบ (ตัวอย่างง่ายๆ)
        [NSThread sleepForTimeInterval:0.1];
        
        // 4. Large data processing
        NSLog(@"\n--- Processing large data ---");
        NSMutableArray *results = [NSMutableArray array];
        
        for (int batch = 0; batch < 3; batch++) {
            @autoreleasepool {
                NSMutableArray *batchData = [NSMutableArray array];
                for (int i = 0; i < 100; i++) {
                    // สร้าง object จำนวนมาก
                    NSDictionary *item = @{
                        @"id": @(batch * 100 + i),
                        @"value": [NSString stringWithFormat:@"item_%d_%d", batch, i]
                    };
                    [batchData addObject:item];
                }
                // process batch
                NSLog(@"Batch %d: %lu items", batch, batchData.count);
                [results addObjectsFromArray:batchData];
                // batchData autorelease objects ถูก drain ที่นี่
            }
        }
        NSLog(@"Total results: %lu", results.count);
    }
    return 0;
}
```

---

## 9.7 Memory Leaks และการตรวจสอบ

### ประเภทของ Memory Bugs

```objc
#import <Foundation/Foundation.h>

// 1. Memory Leak: สร้าง object แต่ไม่ release
void demonstrateLeak() {
    // ใน ARC: ยากที่จะเกิด leak จาก basic code
    // แต่ยังเกิดได้จาก:
    // - Retain cycles
    // - C malloc ที่ไม่ free
    // - Core Foundation objects ที่ไม่ release
    
    // C malloc leak:
    int *arr = malloc(100 * sizeof(int));
    // ถ้าไม่ free(arr) → memory leak!
    // ARC ไม่จัดการ C memory!
    
    if (arr) {
        arr[0] = 42;
        NSLog(@"arr[0] = %d", arr[0]);
        free(arr);  // ต้อง free เอง
    }
}

// 2. Core Foundation leak:
void demonstrateCFLeak() {
    // CFString ไม่ถูกจัดการโดย ARC
    CFStringRef cfStr = CFStringCreateWithCString(NULL, "Hello", kCFStringEncodingUTF8);
    
    NSLog(@"CF String: %@", (__bridge NSString *)cfStr);
    
    CFRelease(cfStr);  // ต้อง release เอง!
    // ถ้าลืม → memory leak
}

// 3. Bridging
void demonstrateBridging() {
    NSString *nsStr = @"Hello World";
    
    // __bridge: แปลงโดยไม่เปลี่ยน ownership
    CFStringRef cfStr = (__bridge CFStringRef)nsStr;
    // ARC ยังดูแล nsStr
    
    // __bridge_retained: ย้าย ownership ไปให้ CF
    CFStringRef cfStrOwned = (__bridge_retained CFStringRef)nsStr;
    // ARC จะไม่ดูแล nsStr อีก → ต้อง CFRelease เอง!
    NSLog(@"CF owned: %@", (__bridge NSString *)cfStrOwned);
    CFRelease(cfStrOwned);
    
    // __bridge_transfer: ย้าย ownership กลับมาให้ ARC
    CFStringRef cfNew = CFStringCreateWithCString(NULL, "New String", kCFStringEncodingUTF8);
    NSString *nsNew = (__bridge_transfer NSString *)cfNew;
    // ARC จะดูแล nsNew แล้ว ไม่ต้อง CFRelease
    NSLog(@"NS from CF: %@", nsNew);
}

int main(int argc, const char * argv[]) {
    @autoreleasepool {
        NSLog(@"=== Memory Bugs Demo ===");
        demonstrateLeak();
        demonstrateCFLeak();
        demonstrateBridging();
    }
    return 0;
}
```

### การใช้ Instruments

```
Instruments เป็น tool ใน Xcode สำหรับตรวจหา memory issues

วิธีใช้:
1. Product → Profile (⌘I)
2. เลือก "Leaks" template
3. Run app และทำ actions ต่างๆ
4. ดู Leaks instrument → แสดง leaked objects

Instruments templates ที่มีประโยชน์:
- Leaks: ตรวจหา memory leaks
- Allocations: ดู memory usage ทั้งหมด
- Time Profiler: ดูว่า code ส่วนไหนช้า
- Core Data: ดู database operations
- Network: ดู network requests

การอ่านผล Allocations:
- All Heap & Anonymous VM: memory ทั้งหมด
- Live Bytes: memory ที่กำลังใช้อยู่
- Peak: memory สูงสุดที่เคยใช้
- # Persistent: จำนวน objects ที่ยังมีชีวิต
- # Transient: จำนวน objects ที่ถูก deallocate แล้ว
```

---

## 9.8 Best Practices

### Property Attributes

```objc
#import <Foundation/Foundation.h>

@interface Product : NSObject

// NSString, NSArray, NSDictionary → ใช้ copy ป้องกัน external mutation
@property (nonatomic, copy) NSString *name;
@property (nonatomic, copy) NSArray *tags;

// Mutable types → ถ้าต้องการเก็บแบบ mutable ใช้ strong
@property (nonatomic, strong) NSMutableArray *items;

// Primitive types → ใช้ assign
@property (nonatomic, assign) NSInteger price;
@property (nonatomic, assign) BOOL isAvailable;

// Delegate pattern → ใช้ weak ป้องกัน retain cycle
@property (nonatomic, weak) id<NSObject> delegate;

// Strong ownership
@property (nonatomic, strong) NSObject *owner;

@end

@implementation Product

- (instancetype)init {
    self = [super init];
    if (self) {
        _items = [NSMutableArray array];  // lazy initialization pattern
        _isAvailable = YES;
    }
    return self;
}

- (void)setName:(NSString *)name {
    // copy ป้องกันไม่ให้ external code แก้ไข string ของเรา
    _name = [name copy];
}

@end

int main(int argc, const char * argv[]) {
    @autoreleasepool {
        
        NSLog(@"=== Property Best Practices ===");
        
        Product *p = [[Product alloc] init];
        
        // ทดสอบ copy property
        NSMutableString *mutableName = [NSMutableString stringWithString:@"Widget"];
        p.name = mutableName;
        [mutableName appendString:@" MODIFIED"];  // แก้ไข original
        
        NSLog(@"Product name: %@", p.name);  // ไม่เปลี่ยน เพราะ copy
        NSLog(@"Mutable original: %@", mutableName);
        
        // ทดสอบ copy array
        NSMutableArray *mutableTags = [NSMutableArray arrayWithObjects:@"A", @"B", nil];
        p.tags = mutableTags;
        [mutableTags addObject:@"C"];  // แก้ไข original
        
        NSLog(@"\nProduct tags: %@", p.tags);  // ไม่มี "C" เพราะ copy
        NSLog(@"Mutable original: %@", mutableTags);
    }
    return 0;
}
```

### Singleton Pattern

```objc
#import <Foundation/Foundation.h>

// Singleton - object ที่มีแค่ instance เดียว
@interface DatabaseManager : NSObject

@property (nonatomic, strong) NSString *databasePath;

+ (instancetype)sharedInstance;
- (void)connect;
- (void)disconnect;

@end

@implementation DatabaseManager

+ (instancetype)sharedInstance {
    static DatabaseManager *instance = nil;
    static dispatch_once_t onceToken;
    dispatch_once(&onceToken, ^{
        instance = [[self alloc] init];
    });
    return instance;
}

- (instancetype)init {
    self = [super init];
    if (self) {
        _databasePath = @"/var/db/app.sqlite";
        NSLog(@"DatabaseManager initialized");
    }
    return self;
}

- (void)connect {
    NSLog(@"Connected to: %@", _databasePath);
}

- (void)disconnect {
    NSLog(@"Disconnected from: %@", _databasePath);
}

@end

int main(int argc, const char * argv[]) {
    @autoreleasepool {
        
        NSLog(@"=== Singleton Pattern ===");
        
        DatabaseManager *db1 = [DatabaseManager sharedInstance];
        DatabaseManager *db2 = [DatabaseManager sharedInstance];
        
        NSLog(@"db1 == db2: %@", (db1 == db2) ? @"YES (same instance)" : @"NO");
        
        [db1 connect];
        [db2 disconnect];  // db2 คือ object เดียวกัน
    }
    return 0;
}
```

### Memory Best Practices ทั้งหมด

```objc
#import <Foundation/Foundation.h>

// Best Practices Collection

@interface BestPracticesDemo : NSObject

// 1. Strong/Weak/Copy ที่เหมาะสม
@property (nonatomic, copy) NSString *title;             // copy for value types
@property (nonatomic, strong) NSMutableArray *data;      // strong for objects you own
@property (nonatomic, weak) id delegate;                  // weak for delegates
@property (nonatomic, assign) NSInteger count;            // assign for primitives

// 2. Lazy initialization
@property (nonatomic, strong) NSMutableDictionary *cache;

@end

@implementation BestPracticesDemo

// Lazy initialization - สร้างเมื่อต้องการใช้จริง
- (NSMutableDictionary *)cache {
    if (!_cache) {
        _cache = [NSMutableDictionary dictionary];
        NSLog(@"Cache initialized lazily");
    }
    return _cache;
}

// 3. ใช้ local variables ใน method ให้ ARC จัดการ
- (void)processData:(NSArray *)input {
    // Local variables ใน method จะถูก release เมื่อ method จบ
    NSMutableArray *processed = [NSMutableArray array];
    
    for (id item in input) {
        // ทำงานกับ item
        [processed addObject:item];
    }
    
    NSLog(@"Processed %lu items", processed.count);
    // processed ถูก release อัตโนมัติเมื่อ method จบ
}

// 4. ระวัง strong self ใน block
- (void)performAsync {
    __weak typeof(self) weakSelf = self;
    
    dispatch_async(dispatch_get_main_queue(), ^{
        __strong typeof(weakSelf) strongSelf = weakSelf;
        if (!strongSelf) return;  // object อาจถูก deallocate แล้ว
        
        NSLog(@"Async complete: %@", strongSelf.title);
    });
}

@end

int main(int argc, const char * argv[]) {
    @autoreleasepool {
        
        NSLog(@"=== Best Practices ===");
        
        BestPracticesDemo *demo = [[BestPracticesDemo alloc] init];
        demo.title = @"Test";
        
        // Lazy init - cache สร้างตอนนี้
        NSLog(@"Cache (first access): %@", demo.cache);
        
        // กำหนดค่าแล้ว cache ไม่สร้างใหม่
        [demo.cache setObject:@"value" forKey:@"key"];
        NSLog(@"Cache: %@", demo.cache);
        
        [demo processData:@[@1, @2, @3]];
        [demo performAsync];
        
        // รอ async
        [NSThread sleepForTimeInterval:0.01];
        
        // 5. nil checks
        NSString *maybeNil = nil;
        NSLog(@"\nSend message to nil: %@", [maybeNil uppercaseString]);  // ปลอดภัย คืน nil
        NSLog(@"Length of nil: %lu", maybeNil.length);  // คืน 0 ปลอดภัย
        
        // 6. ตรวจสอบก่อนใช้ optional values
        NSArray *arr = nil;
        if (arr) {
            NSLog(@"arr has %lu items", arr.count);
        } else {
            NSLog(@"arr is nil");
        }
    }
    return 0;
}
```

---

## 9.9 ตัวอย่าง Memory Management Patterns

### Object Graph

```objc
#import <Foundation/Foundation.h>

// Object graph ที่ซับซ้อน
@class Node;

@interface Node : NSObject
@property (nonatomic, strong) NSString *value;
@property (nonatomic, strong) NSMutableArray<Node *> *children;   // strong: parent → children
@property (nonatomic, weak) Node *parent;                          // weak: child → parent
- (instancetype)initWithValue:(NSString *)value;
- (void)addChild:(Node *)child;
@end

@implementation Node

- (instancetype)initWithValue:(NSString *)value {
    self = [super init];
    if (self) {
        _value = value;
        _children = [NSMutableArray array];
    }
    return self;
}

- (void)addChild:(Node *)child {
    child.parent = self;  // weak reference: child → parent
    [_children addObject:child];  // strong: parent → child
}

- (void)dealloc {
    NSLog(@"Node '%@' deallocated", _value);
}

- (NSString *)description {
    return [NSString stringWithFormat:@"Node(%@)", _value];
}

@end

int main(int argc, const char * argv[]) {
    @autoreleasepool {
        
        NSLog(@"=== Object Graph ===");
        
        Node *root = [[Node alloc] initWithValue:@"Root"];
        Node *child1 = [[Node alloc] initWithValue:@"Child1"];
        Node *child2 = [[Node alloc] initWithValue:@"Child2"];
        Node *grandchild = [[Node alloc] initWithValue:@"Grandchild"];
        
        [root addChild:child1];
        [root addChild:child2];
        [child1 addChild:grandchild];
        
        // Tree: Root → Child1 → Grandchild
        //              → Child2
        
        NSLog(@"Root: %@, children: %@", root, root.children);
        NSLog(@"Child1 parent: %@", child1.parent);  // weak
        NSLog(@"Grandchild parent: %@", grandchild.parent);  // weak
        
        // เมื่อ root ออกจาก scope:
        // root → strong → children → 0 → dealloc children → cascades
    }
    NSLog(@"(ควรเห็น dealloc messages ทั้งหมด)");
    
    return 0;
}
```

### Cache ที่ใช้ NSCache

```objc
#import <Foundation/Foundation.h>

// NSCache: automatic eviction under memory pressure
@interface ImageCache : NSObject
@property (nonatomic, strong) NSCache *cache;
+ (instancetype)sharedCache;
- (NSData *)imageDataForKey:(NSString *)key;
- (void)setImageData:(NSData *)data forKey:(NSString *)key;
@end

@implementation ImageCache

+ (instancetype)sharedCache {
    static ImageCache *instance = nil;
    static dispatch_once_t token;
    dispatch_once(&token, ^{
        instance = [[ImageCache alloc] init];
    });
    return instance;
}

- (instancetype)init {
    self = [super init];
    if (self) {
        _cache = [[NSCache alloc] init];
        _cache.name = @"ImageCache";
        _cache.countLimit = 100;           // จำกัดจำนวน objects
        _cache.totalCostLimit = 50 * 1024 * 1024;  // 50 MB
    }
    return self;
}

- (NSData *)imageDataForKey:(NSString *)key {
    return [_cache objectForKey:key];
}

- (void)setImageData:(NSData *)data forKey:(NSString *)key {
    NSUInteger cost = data.length;
    [_cache setObject:data forKey:key cost:cost];
}

@end

int main(int argc, const char * argv[]) {
    @autoreleasepool {
        
        NSLog(@"=== NSCache Demo ===");
        
        ImageCache *cache = [ImageCache sharedCache];
        
        // เพิ่มข้อมูลใน cache
        for (int i = 0; i < 5; i++) {
            NSString *key = [NSString stringWithFormat:@"image_%d", i];
            NSData *fakeData = [NSData dataWithBytes:"fake image data" length:16];
            [cache setImageData:fakeData forKey:key];
            NSLog(@"Cached: %@", key);
        }
        
        // ดึงข้อมูลจาก cache
        NSData *img = [cache imageDataForKey:@"image_2"];
        if (img) {
            NSLog(@"Cache hit: image_2 (%lu bytes)", img.length);
        }
        
        NSData *missing = [cache imageDataForKey:@"image_999"];
        NSLog(@"Cache miss: %@", missing ? @"found" : @"not found");
        
        // NSCache จะ evict objects อัตโนมัติเมื่อ memory ต่ำ
        // ต่างจาก NSDictionary ที่เก็บตลอดจนกว่าจะ release
    }
    return 0;
}
```

---

## 9.10 สรุปเปรียบเทียบ

| หัวข้อ | MRC | ARC |
|--------|-----|-----|
| retain/release | ทำเอง | Compiler แทรกให้ |
| dealloc | ต้องเรียก [super dealloc] | ไม่ต้อง |
| autorelease | ต้องจัดการเอง | จัดการบางส่วน |
| Retain Cycle | ต้องหลีกเลี่ยงเอง | ยังต้องระวัง |
| Weak references | zeroing weak (iOS 5+) | ใช้ __weak |
| Performance | ดีกว่า GC | ดีมาก (เหมือน MRC) |
| Errors | มาก | น้อยกว่า |

| Qualifier | ความหมาย | ใช้เมื่อ |
|-----------|---------|---------|
| strong | เจ้าของ, เพิ่ม count | default สำหรับ objects |
| weak | ไม่เป็นเจ้าของ, nil อัตโนมัติ | delegates, avoid cycles |
| copy | copy object ตอน assign | NSString, NSArray, blocks |
| assign | ไม่ retain | primitive types |
| __weak | weak local variable | ใน method/block |
| __strong | strong local (default) | ใน block ต้องการ strong |
| __block | อนุญาตให้ block แก้ไขตัวแปรได้ | ใน block ที่ต้องแก้ค่า |

---

## แบบฝึกหัด

### ข้อที่ 1: Lifecycle Observation
```objc
// สร้าง class ที่ log ทุก lifecycle event
// init, dealloc, copy, mutableCopy
// สังเกต behavior ในสถานการณ์ต่างๆ
@interface Observable : NSObject
@property (nonatomic, strong) NSString *identifier;
@end
```

### ข้อที่ 2: ค้นหา Retain Cycle
```objc
// Code นี้มี retain cycle ไหม? ถ้ามี แก้ไขอย่างไร?
@interface A : NSObject
@property (nonatomic, strong) B *b;
@end

@interface B : NSObject
@property (nonatomic, strong) A *a;
@end

// A *objA = [A new];
// B *objB = [B new];
// objA.b = objB;
// objB.a = objA;
```

### ข้อที่ 3: Block Memory
```objc
// แก้ไข retain cycle ใน block นี้
@interface ViewController : NSObject
@property (nonatomic, strong) NSTimer *timer;
- (void)startTimer;
@end

// - (void)startTimer {
//     self.timer = [NSTimer scheduledTimerWithTimeInterval:1.0
//                                                  target:self
//                                                selector:@selector(tick)
//                                                userInfo:nil
//                                                 repeats:YES];
// }
// วิธีไหนที่ดีกว่า? ทำไม?
```

### ข้อที่ 4: Object Pool
```objc
// สร้าง ObjectPool ที่ reuse objects แทนการสร้างใหม่
// เหมาะสำหรับ objects ที่สร้างแพง เช่น database connections
@interface ObjectPool : NSObject
- (id)acquire;
- (void)release:(id)object;
- (NSUInteger)poolSize;
@end
```

### ข้อที่ 5: Autorelease Pool Timing
```objc
// เขียน code ที่แสดงให้เห็นความต่างระหว่าง
// การใช้ @autoreleasepool ใน loop กับไม่ใช้
// วัด memory peak ด้วย instruments หรือ debug output
```

---

*จบบทที่ 9: Memory Management* | ไปยัง [บทที่ 10: Introduction to OOP →](part-10-oop-intro.md)
