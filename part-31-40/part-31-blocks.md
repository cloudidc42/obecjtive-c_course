# Part 31: Blocks ใน Objective-C

## บทนำ

**Blocks** เป็น feature ที่ Apple เพิ่มเข้ามาใน Objective-C (ตั้งแต่ iOS 4 / macOS 10.6) Blocks คือ **anonymous functions** (หรือ closures) ที่สามารถ:
- เก็บ code เป็นค่า (first-class value)
- ส่งเป็น argument ให้ function
- return จาก function
- **capture** ตัวแปรจาก surrounding scope

Blocks เป็น foundation สำหรับ GCD (Grand Central Dispatch), completion handlers, callbacks, và functional programming patterns ใน Objective-C

---

## 31.1 Blocks คืออะไร?

Block เป็น self-contained unit of code ที่:
- มี code body (เหมือน function)
- อ้างอิงตัวแปรจาก scope ที่สร้าง (capturing)
- เก็บไว้ในตัวแปรได้
- ส่งเป็น argument ได้
- return ค่าได้

ใน Swift เรียกว่า **Closure** ใน Python เรียกว่า **Lambda** ใน JavaScript เรียกว่า **Arrow function** หรือ **anonymous function**

---

## 31.2 Block Syntax

### 31.2.1 รูปแบบทั่วไป

```
^returnType(parameterTypes) { body }
```

### 31.2.2 ตัวอย่างพื้นฐาน

```objc
#import <Foundation/Foundation.h>

int main(int argc, const char * argv[]) {
    @autoreleasepool {
        // Block ที่ไม่มี parameter ไม่มี return value
        void (^sayHello)(void) = ^{
            NSLog(@"สวัสดีจาก Block!");
        };
        sayHello(); // เรียกใช้งาน
        
        // Block ที่รับ parameter
        void (^greet)(NSString *) = ^(NSString *name) {
            NSLog(@"สวัสดี %@!", name);
        };
        greet(@"สมชาย");
        greet(@"สมหญิง");
        
        // Block ที่ return ค่า
        NSInteger (^add)(NSInteger, NSInteger) = ^(NSInteger a, NSInteger b) {
            return a + b;
        };
        NSLog(@"3 + 4 = %ld", (long)add(3, 4));
        
        // Block ที่ return BOOL
        BOOL (^isEven)(NSInteger) = ^(NSInteger n) {
            return n % 2 == 0;
        };
        NSLog(@"10 เป็นเลขคู่? %@", isEven(10) ? @"ใช่" : @"ไม่");
        NSLog(@"7 เป็นเลขคู่? %@",  isEven(7)  ? @"ใช่" : @"ไม่");
        
        // Block ที่ return NSString
        NSString *(^describe)(NSInteger) = ^(NSInteger n) {
            if (n > 0) return @"บวก";
            if (n < 0) return @"ลบ";
            return @"ศูนย์";
        };
        NSLog(@"5: %@", describe(5));
        NSLog(@"-3: %@", describe(-3));
        NSLog(@"0: %@", describe(0));
    }
    return 0;
}
```

### 31.2.3 Inline Block (ไม่เก็บในตัวแปร)

```objc
#import <Foundation/Foundation.h>

// ฟังก์ชันที่รับ block
void doTwice(void (^operation)(void)) {
    operation();
    operation();
}

int main(int argc, const char * argv[]) {
    @autoreleasepool {
        // ใช้ block แบบ inline โดยไม่เก็บในตัวแปร
        doTwice(^{
            NSLog(@"ทำซ้ำ!");
        });
        
        // หรือเรียก block โดยตรงทันที
        NSInteger result = (^(NSInteger x) { return x * x; })(5); // 5^2
        NSLog(@"5 squared = %ld", (long)result);
    }
    return 0;
}
```

---

## 31.3 Capturing Variables

Block สามารถ "จับ" (capture) ตัวแปรจาก enclosing scope ได้ มีสองแบบ:

### 31.3.1 Capture by Value (default)

```objc
#import <Foundation/Foundation.h>

int main(int argc, const char * argv[]) {
    @autoreleasepool {
        NSInteger x = 10;
        
        // Block capture x ด้วยค่าตอนที่ block ถูกสร้าง
        void (^printX)(void) = ^{
            NSLog(@"x = %ld", (long)x);
        };
        
        printX(); // x = 10
        
        x = 20; // เปลี่ยนค่า x ภายนอก
        printX(); // x ยังเป็น 10 อยู่! (captured by value)
        
        NSLog(@"x ภายนอก: %ld", (long)x); // 20
        
        // ถ้าพยายามแก้ไข x ในblock จะ compile error
        // void (^tryModify)(void) = ^{
        //     x = 30; // ERROR: Variable is not assignable (missing __block type specifier)
        // };
    }
    return 0;
}
```

### 31.3.2 Capture by Reference ด้วย __block

```objc
#import <Foundation/Foundation.h>

int main(int argc, const char * argv[]) {
    @autoreleasepool {
        __block NSInteger counter = 0;
        
        // Block สามารถแก้ไข counter ได้
        void (^increment)(void) = ^{
            counter++;
        };
        
        void (^printCounter)(void) = ^{
            NSLog(@"counter = %ld", (long)counter);
        };
        
        printCounter(); // 0
        increment();
        increment();
        increment();
        printCounter(); // 3
        
        NSLog(@"counter ภายนอก: %ld", (long)counter); // 3 (เปลี่ยนไปด้วย!)
        
        // __block กับ NSMutableArray
        __block NSMutableArray *results = [NSMutableArray array];
        
        void (^addValue)(NSInteger) = ^(NSInteger n) {
            [results addObject:@(n)];
        };
        
        addValue(1);
        addValue(2);
        addValue(3);
        NSLog(@"Results: %@", results); // [1, 2, 3]
    }
    return 0;
}
```

### 31.3.3 Capture ตัวแปรประเภทต่าง ๆ

```objc
#import <Foundation/Foundation.h>

int main(int argc, const char * argv[]) {
    @autoreleasepool {
        // Primitive types - captured by value
        int a = 5;
        double b = 3.14;
        char c = 'A';
        
        void (^showPrimitives)(void) = ^{
            NSLog(@"a=%d, b=%.2f, c=%c", a, b, c);
        };
        a = 99; // ไม่กระทบ block
        showPrimitives(); // แสดง 5, 3.14, A
        
        // Objective-C objects - captured as strong reference
        NSString *str = @"สวัสดี";
        NSMutableArray *arr = [NSMutableArray array];
        
        void (^showObjects)(void) = ^{
            NSLog(@"str: %@, arr count: %lu", str, (unsigned long)arr.count);
        };
        
        [arr addObject:@"one"]; // arr.count จะเป็น 1 ใน block
        showObjects(); // str: สวัสดี, arr count: 1
        
        // C struct - captured by value
        CGPoint point = CGPointMake(10, 20);
        void (^showPoint)(void) = ^{
            NSLog(@"point: (%.0f, %.0f)", point.x, point.y);
        };
        showPoint(); // (10, 20)
    }
    return 0;
}
```

---

## 31.4 Block as Function Parameter

### 31.4.1 รับ block เป็น parameter

```objc
#import <Foundation/Foundation.h>

// ฟังก์ชันที่รับ block เป็น parameter
void repeatAction(NSInteger times, void (^action)(NSInteger index)) {
    for (NSInteger i = 0; i < times; i++) {
        action(i);
    }
}

// filter array ด้วย predicate block
NSArray* filterArray(NSArray *array, BOOL (^predicate)(id element)) {
    NSMutableArray *result = [NSMutableArray array];
    for (id element in array) {
        if (predicate(element)) {
            [result addObject:element];
        }
    }
    return [result copy];
}

// transform array ด้วย transform block
NSArray* mapArray(NSArray *array, id (^transform)(id element)) {
    NSMutableArray *result = [NSMutableArray array];
    for (id element in array) {
        [result addObject:transform(element)];
    }
    return [result copy];
}

// reduce array
id reduceArray(NSArray *array, id initialValue, id (^accumulate)(id acc, id element)) {
    id acc = initialValue;
    for (id element in array) {
        acc = accumulate(acc, element);
    }
    return acc;
}

int main(int argc, const char * argv[]) {
    @autoreleasepool {
        // ใช้ repeatAction
        repeatAction(3, ^(NSInteger i) {
            NSLog(@"ครั้งที่ %ld", (long)(i + 1));
        });
        
        // filter เลขคู่
        NSArray *numbers = @[@1, @2, @3, @4, @5, @6, @7, @8, @9, @10];
        NSArray *evens = filterArray(numbers, ^BOOL(id element) {
            return [element integerValue] % 2 == 0;
        });
        NSLog(@"เลขคู่: %@", evens);
        
        // map: คูณแต่ละตัวด้วย 2
        NSArray *doubled = mapArray(numbers, ^id(id element) {
            return @([element integerValue] * 2);
        });
        NSLog(@"คูณ 2: %@", doubled);
        
        // reduce: หาผลรวม
        NSNumber *sum = reduceArray(numbers, @0, ^id(id acc, id element) {
            return @([acc integerValue] + [element integerValue]);
        });
        NSLog(@"ผลรวม: %@", sum);
        
        // filter strings
        NSArray *names = @[@"สมชาย", @"Alice", @"สมหญิง", @"Bob", @"ประสาน"];
        NSArray *thaiNames = filterArray(names, ^BOOL(id element) {
            // ตรวจสอบว่าเป็นภาษาไทยหรือไม่ (simplified check)
            NSString *s = element;
            unichar firstChar = [s characterAtIndex:0];
            return firstChar >= 0x0E00 && firstChar <= 0x0E7F; // Thai unicode range
        });
        NSLog(@"ชื่อไทย: %@", thaiNames);
    }
    return 0;
}
```

### 31.4.2 Completion Handler Pattern

```objc
#import <Foundation/Foundation.h>

// Completion handler เป็น pattern ที่ใช้บ่อยมากใน iOS development

// Simulate async operation with completion block
void loadDataAsync(NSString *url,
                   void (^completion)(NSData *data, NSError *error)) {
    // จำลองการโหลดข้อมูล
    NSLog(@"กำลังโหลดจาก: %@", url);
    
    if ([url containsString:@"error"]) {
        NSError *error = [NSError errorWithDomain:@"com.net"
                                             code:404
                                         userInfo:@{
            NSLocalizedDescriptionKey: @"ไม่พบข้อมูล"
        }];
        completion(nil, error);
    } else {
        NSData *mockData = [url dataUsingEncoding:NSUTF8StringEncoding];
        completion(mockData, nil);
    }
}

// ฟังก์ชันที่ return ผลลัพธ์แบบ async
void processFile(NSString *path,
                 void (^onSuccess)(NSDictionary *result),
                 void (^onFailure)(NSError *error)) {
    if (path.length == 0) {
        NSError *error = [NSError errorWithDomain:@"com.file" code:1
                                         userInfo:@{NSLocalizedDescriptionKey: @"Path ว่างเปล่า"}];
        onFailure(error);
        return;
    }
    
    NSDictionary *result = @{
        @"path": path,
        @"size": @1024,
        @"processed": @YES
    };
    onSuccess(result);
}

int main(int argc, const char * argv[]) {
    @autoreleasepool {
        // ใช้ completion block
        loadDataAsync(@"https://api.example.com/data", ^(NSData *data, NSError *error) {
            if (error) {
                NSLog(@"Error: %@", error.localizedDescription);
            } else {
                NSLog(@"โหลดสำเร็จ: %lu bytes", (unsigned long)data.length);
            }
        });
        
        // ใช้ success/failure blocks
        processFile(@"/path/to/file.txt",
                    ^(NSDictionary *result) {
                        NSLog(@"สำเร็จ: %@", result[@"path"]);
                    },
                    ^(NSError *error) {
                        NSLog(@"ล้มเหลว: %@", error.localizedDescription);
                    });
        
        processFile(@"", // path ว่าง
                    ^(NSDictionary *result) {
                        NSLog(@"สำเร็จ: %@", result);
                    },
                    ^(NSError *error) {
                        NSLog(@"คาดหวัง error: %@", error.localizedDescription);
                    });
    }
    return 0;
}
```

---

## 31.5 Storing Blocks in Properties

### 31.5.1 Block ใน property

```objc
#import <Foundation/Foundation.h>

// Class ที่ใช้ block ใน property
@interface Button : NSObject

@property (nonatomic, copy) void (^onTap)(void);
@property (nonatomic, copy) void (^onLongPress)(CGPoint location);
@property (nonatomic, copy) NSString *(^titleProvider)(void);

- (void)simulateTap;
- (void)simulateLongPressAtPoint:(CGPoint)point;

@end

@implementation Button

- (void)simulateTap {
    if (self.onTap) {
        self.onTap();
    } else {
        NSLog(@"Button tapped (no handler)");
    }
}

- (void)simulateLongPressAtPoint:(CGPoint)point {
    if (self.onLongPress) {
        self.onLongPress(point);
    }
}

- (NSString *)currentTitle {
    if (self.titleProvider) {
        return self.titleProvider();
    }
    return @"Button";
}

@end

int main(int argc, const char * argv[]) {
    @autoreleasepool {
        Button *button = [[Button alloc] init];
        
        // กำหนด block handlers
        button.onTap = ^{
            NSLog(@"Button ถูกกด!");
        };
        
        button.onLongPress = ^(CGPoint location) {
            NSLog(@"Long press ที่ (%.0f, %.0f)", location.x, location.y);
        };
        
        __block NSInteger tapCount = 0;
        button.onTap = ^{
            tapCount++;
            NSLog(@"Button ถูกกดครั้งที่ %ld", (long)tapCount);
        };
        
        button.titleProvider = ^NSString *{
            return [NSString stringWithFormat:@"กดแล้ว %ld ครั้ง", (long)tapCount];
        };
        
        // จำลองการกด
        [button simulateTap];
        [button simulateTap];
        [button simulateLongPressAtPoint:CGPointMake(100, 200)];
        NSLog(@"Title: %@", [button currentTitle]);
    }
    return 0;
}
```

### 31.5.2 Block ใน Collection

```objc
#import <Foundation/Foundation.h>

int main(int argc, const char * argv[]) {
    @autoreleasepool {
        // เก็บ block ใน NSArray
        NSArray<void (^)(void)> *actions = @[
            ^{ NSLog(@"Action 1: print date"); },
            ^{ NSLog(@"Action 2: do something"); },
            ^{ NSLog(@"Action 3: cleanup"); },
        ];
        
        // เรียกใช้ทีละ action
        for (void (^action)(void) in actions) {
            action();
        }
        
        // เก็บ block ใน NSDictionary
        NSDictionary<NSString *, void (^)(NSString *)> *handlers = @{
            @"greet": ^(NSString *name) { NSLog(@"สวัสดี %@", name); },
            @"farewell": ^(NSString *name) { NSLog(@"ลาก่อน %@", name); },
            @"shout": ^(NSString *name) { NSLog(@"%@!!!", [name uppercaseString]); },
        };
        
        NSString *action = @"greet";
        void (^handler)(NSString *) = handlers[action];
        if (handler) {
            handler(@"สมชาย");
        }
    }
    return 0;
}
```

---

## 31.6 typedef สำหรับ Block Types

### 31.6.1 ทำไมต้องใช้ typedef

Block syntax อาจอ่านยากเมื่อซับซ้อนขึ้น `typedef` ช่วยให้อ่านง่ายขึ้น:

```objc
#import <Foundation/Foundation.h>

// ก่อนใช้ typedef - ยาวและอ่านยาก
- (void)performTask:(void (^)(BOOL success, NSError *error))completion;
@property (nonatomic, copy) void (^onCompletion)(BOOL success, NSError *error);

// หลังใช้ typedef - อ่านง่ายกว่า
typedef void (^CompletionBlock)(BOOL success, NSError *error);
typedef void (^VoidBlock)(void);
typedef BOOL (^PredicateBlock)(id element);
typedef id (^TransformBlock)(id element);
typedef NSString *(^StringProviderBlock)(void);
typedef void (^ProgressBlock)(float progress, NSString *message);

// ใช้ใน interface
@interface TaskManager : NSObject

@property (nonatomic, copy) CompletionBlock completionHandler;
@property (nonatomic, copy) ProgressBlock progressHandler;

- (void)startTaskWithCompletion:(CompletionBlock)completion;
- (void)startTaskWithProgress:(ProgressBlock)progress
                   completion:(CompletionBlock)completion;

@end

@implementation TaskManager

- (void)startTaskWithCompletion:(CompletionBlock)completion {
    NSLog(@"Task started...");
    // จำลองงานเสร็จ
    if (completion) {
        completion(YES, nil);
    }
}

- (void)startTaskWithProgress:(ProgressBlock)progress
                   completion:(CompletionBlock)completion {
    NSLog(@"Task started with progress tracking...");
    
    // จำลอง progress
    for (float p = 0.0f; p <= 1.0f; p += 0.25f) {
        if (progress) {
            progress(p, [NSString stringWithFormat:@"%.0f%% เสร็จแล้ว", p * 100]);
        }
    }
    
    if (completion) {
        completion(YES, nil);
    }
}

@end

int main(int argc, const char * argv[]) {
    @autoreleasepool {
        TaskManager *manager = [[TaskManager alloc] init];
        
        // ใช้ typedef ทำให้อ่านง่าย
        CompletionBlock completion = ^(BOOL success, NSError *error) {
            if (success) {
                NSLog(@"Task เสร็จแล้ว!");
            } else {
                NSLog(@"Task ล้มเหลว: %@", error.localizedDescription);
            }
        };
        
        [manager startTaskWithCompletion:completion];
        
        [manager startTaskWithProgress:^(float progress, NSString *message) {
            NSLog(@"Progress: %@", message);
        } completion:^(BOOL success, NSError *error) {
            NSLog(@"ทำทุกอย่างเสร็จแล้ว!");
        }];
    }
    return 0;
}
```

---

## 31.7 Memory Management และ Retain Cycles

### 31.7.1 Block กับ Memory

Block มีสามตำแหน่งใน memory:
- **Stack block** - สร้างบน stack, ถูก deallocate เมื่อออกจาก scope
- **Heap block** - copy ไปอยู่บน heap (เมื่อ assign ให้ property หรือ copy)
- **Global block** - block ที่ไม่ capture ตัวแปรใด ๆ (อยู่ใน data segment)

```objc
#import <Foundation/Foundation.h>

int main(int argc, const char * argv[]) {
    @autoreleasepool {
        // Stack block - สร้างเมื่อ assign ให้ variable
        void (^stackBlock)(void) = ^{
            NSLog(@"Stack block");
        };
        
        // เมื่อ assign ให้ property หรือ copy -> ไปอยู่บน heap
        void (^heapBlock)(void) = [stackBlock copy];
        
        // Global block - ไม่ capture ตัวแปรใด ๆ
        void (^globalBlock)(void) = ^{
            NSLog(@"Global block (no captures)");
        };
        
        stackBlock();
        heapBlock();
        globalBlock();
        
        NSLog(@"stackBlock class: %@", [stackBlock class]);
        NSLog(@"heapBlock class:  %@", [heapBlock class]);
        NSLog(@"globalBlock class: %@", [globalBlock class]);
    }
    return 0;
}
```

### 31.7.2 Retain Cycle ปัญหาสำคัญ!

```objc
#import <Foundation/Foundation.h>

@interface MyViewController : NSObject

@property (nonatomic, copy) void (^fetchAction)(void);
@property (nonatomic, strong) NSMutableArray *data;

- (void)setupBadWay;    // retain cycle!
- (void)setupGoodWay;   // ถูกต้อง

@end

@implementation MyViewController

- (instancetype)init {
    self = [super init];
    if (self) {
        _data = [NSMutableArray array];
    }
    return self;
}

- (void)setupBadWay {
    // ❌ RETAIN CYCLE!
    // self -> fetchAction (block) -> self
    // self จะไม่ถูก deallocate เลย!
    self.fetchAction = ^{
        NSLog(@"Data count: %lu", (unsigned long)self.data.count); // retain cycle!
        [self.data addObject:@"item"]; // อีก retain cycle!
    };
}

- (void)setupGoodWay {
    // ✅ ใช้ __weak self เพื่อหลีกเลี่ยง retain cycle
    __weak typeof(self) weakSelf = self;
    
    self.fetchAction = ^{
        // ตรวจสอบว่า weakSelf ยังอยู่ก่อน
        __strong typeof(weakSelf) strongSelf = weakSelf;
        if (!strongSelf) {
            NSLog(@"ViewController ถูก deallocate แล้ว");
            return;
        }
        
        NSLog(@"Data count: %lu", (unsigned long)strongSelf.data.count);
        [strongSelf.data addObject:@"item"];
    };
}

- (void)dealloc {
    NSLog(@"MyViewController ถูก deallocate");
}

@end

int main(int argc, const char * argv[]) {
    @autoreleasepool {
        // สาธิต retain cycle
        @autoreleasepool {
            MyViewController *vc = [[MyViewController alloc] init];
            [vc setupGoodWay]; // ถ้าใช้ setupBadWay จะไม่มี dealloc
            vc.fetchAction();
        }
        // MyViewController ควร dealloc ที่นี่ (ถ้า setupGoodWay)
        NSLog(@"หลัง inner autoreleasepool");
    }
    return 0;
}
```

### 31.7.3 __weak / __strong dance

```objc
#import <Foundation/Foundation.h>

@interface DataService : NSObject

@property (nonatomic, strong) NSString *serverURL;
@property (nonatomic, copy) void (^periodicTask)(void);

- (void)start;
- (void)process:(NSString *)data;

@end

@implementation DataService

- (instancetype)initWithURL:(NSString *)url {
    self = [super init];
    if (self) {
        _serverURL = url;
    }
    return self;
}

- (void)start {
    // __weak/__strong dance pattern
    __weak DataService *weakSelf = self;
    
    self.periodicTask = ^{
        // สร้าง strong reference ชั่วคราว
        DataService *strongSelf = weakSelf;
        
        // ถ้า strongSelf เป็น nil แปลว่า object ถูก deallocate แล้ว
        if (!strongSelf) return;
        
        // ใช้ strongSelf แทน self - ป้องกัน deallocate ระหว่าง block ทำงาน
        NSLog(@"Connecting to: %@", strongSelf.serverURL);
        [strongSelf process:@"periodic data"];
    };
}

- (void)process:(NSString *)data {
    NSLog(@"Processing: %@", data);
}

- (void)dealloc {
    NSLog(@"DataService deallocated");
}

@end

int main(int argc, const char * argv[]) {
    @autoreleasepool {
        DataService *service = [[DataService alloc] initWithURL:@"https://api.example.com"];
        [service start];
        service.periodicTask();
        
        // service จะถูก dealloc เมื่อออกจาก autoreleasepool
    }
    return 0;
}
```

---

## 31.8 NSArray Block-based Methods

NSArray มี methods ที่ใช้ block หลายตัว:

```objc
#import <Foundation/Foundation.h>

int main(int argc, const char * argv[]) {
    @autoreleasepool {
        NSArray *numbers = @[@5, @3, @8, @1, @9, @2, @7, @4, @6];
        NSArray *names = @[@"สมชาย", @"Alice", @"ประสาน", @"Bob", @"มานี"];
        
        // enumerateObjectsUsingBlock: - วน loop พร้อม index
        [numbers enumerateObjectsUsingBlock:^(id obj, NSUInteger idx, BOOL *stop) {
            NSLog(@"[%lu] = %@", (unsigned long)idx, obj);
            if ([obj integerValue] == 8) {
                *stop = YES; // หยุด enumerate
            }
        }];
        
        // sortedArrayUsingComparator: - เรียงลำดับด้วย block
        NSArray *sortedAsc = [numbers sortedArrayUsingComparator:^NSComparisonResult(id a, id b) {
            return [a compare:b]; // น้อยไปมาก
        }];
        NSLog(@"เรียง น้อย->มาก: %@", sortedAsc);
        
        NSArray *sortedDesc = [numbers sortedArrayUsingComparator:^NSComparisonResult(id a, id b) {
            return [b compare:a]; // มากไปน้อย
        }];
        NSLog(@"เรียง มาก->น้อย: %@", sortedDesc);
        
        // เรียงชื่อตามความยาว
        NSArray *namesByLength = [names sortedArrayUsingComparator:^NSComparisonResult(id a, id b) {
            NSInteger lenA = [(NSString *)a length];
            NSInteger lenB = [(NSString *)b length];
            if (lenA < lenB) return NSOrderedAscending;
            if (lenA > lenB) return NSOrderedDescending;
            return NSOrderedSame;
        }];
        NSLog(@"เรียงตามความยาว: %@", namesByLength);
        
        // indexesOfObjectsPassingTest: - หา indexes ของ elements ที่ผ่านเงื่อนไข
        NSIndexSet *evenIndexes = [numbers indexesOfObjectsPassingTest:^BOOL(id obj,
                                                                              NSUInteger idx,
                                                                              BOOL *stop) {
            return [obj integerValue] % 2 == 0;
        }];
        NSLog(@"Indexes ของเลขคู่: %@", evenIndexes);
        
        // filteredArrayUsingPredicate: (ใช้ NSPredicate แต่เปรียบเทียบกับ block)
        NSArray *filtered = [numbers filteredArrayUsingPredicate:
                             [NSPredicate predicateWithBlock:^BOOL(id obj, NSDictionary *bindings) {
            return [obj integerValue] > 5;
        }]];
        NSLog(@"เลขที่มากกว่า 5: %@", filtered);
        
        // makeObjectsPerformSelector: vs enumerateObjectsUsingBlock:
        // Block version:
        NSMutableArray *upperNames = [NSMutableArray array];
        [names enumerateObjectsUsingBlock:^(id obj, NSUInteger idx, BOOL *stop) {
            [upperNames addObject:[(NSString *)obj uppercaseString]];
        }];
        NSLog(@"ชื่อตัวใหญ่: %@", upperNames);
    }
    return 0;
}
```

---

## 31.9 NSDictionary Block-based Methods

```objc
#import <Foundation/Foundation.h>

int main(int argc, const char * argv[]) {
    @autoreleasepool {
        NSDictionary *scores = @{
            @"Alice": @95,
            @"Bob": @72,
            @"Charlie": @88,
            @"Diana": @91,
            @"Eve": @65,
        };
        
        // enumerateKeysAndObjectsUsingBlock:
        NSLog(@"คะแนนทั้งหมด:");
        [scores enumerateKeysAndObjectsUsingBlock:^(id key, id obj, BOOL *stop) {
            NSLog(@"  %-10s: %@", [(NSString *)key UTF8String], obj);
        }];
        
        // หา keys ที่ผ่านเงื่อนไข
        NSSet *highScorers = [scores keysOfEntriesPassingTest:^BOOL(id key, id obj, BOOL *stop) {
            return [obj integerValue] >= 90;
        }];
        NSLog(@"คะแนน >= 90: %@", highScorers);
        
        // เรียง keys ด้วย comparator
        NSArray *sortedByScore = [scores keysSortedByValueUsingComparator:
                                  ^NSComparisonResult(id a, id b) {
            return [b compare:a]; // มากไปน้อย
        }];
        NSLog(@"เรียงตามคะแนน (มาก->น้อย):");
        for (NSString *name in sortedByScore) {
            NSLog(@"  %-10s: %@", [name UTF8String], scores[name]);
        }
    }
    return 0;
}
```

---

## 31.10 Block Patterns ที่ใช้บ่อย

### 31.10.1 Builder Pattern

```objc
#import <Foundation/Foundation.h>

@interface RequestBuilder : NSObject

@property (nonatomic, strong) NSString *url;
@property (nonatomic, strong) NSString *method;
@property (nonatomic, strong) NSDictionary *headers;
@property (nonatomic, strong) NSData *body;

typedef void (^RequestBuilderBlock)(RequestBuilder *builder);

+ (instancetype)buildWith:(RequestBuilderBlock)builderBlock;

@end

@implementation RequestBuilder

+ (instancetype)buildWith:(RequestBuilderBlock)builderBlock {
    RequestBuilder *builder = [[self alloc] init];
    builder.method = @"GET"; // default
    if (builderBlock) {
        builderBlock(builder);
    }
    return builder;
}

- (NSString *)description {
    return [NSString stringWithFormat:@"[%@] %@ headers:%lu body:%lu bytes",
            self.method, self.url,
            (unsigned long)self.headers.count,
            (unsigned long)self.body.length];
}

@end

int main(int argc, const char * argv[]) {
    @autoreleasepool {
        // ใช้ builder pattern กับ block
        RequestBuilder *getRequest = [RequestBuilder buildWith:^(RequestBuilder *b) {
            b.url = @"https://api.example.com/users";
            b.method = @"GET";
            b.headers = @{@"Accept": @"application/json"};
        }];
        
        RequestBuilder *postRequest = [RequestBuilder buildWith:^(RequestBuilder *b) {
            b.url = @"https://api.example.com/users";
            b.method = @"POST";
            b.headers = @{
                @"Content-Type": @"application/json",
                @"Authorization": @"Bearer token123"
            };
            b.body = [@"{\"name\":\"สมชาย\"}" dataUsingEncoding:NSUTF8StringEncoding];
        }];
        
        NSLog(@"GET: %@", getRequest);
        NSLog(@"POST: %@", postRequest);
    }
    return 0;
}
```

### 31.10.2 Observer/Event Handler Pattern

```objc
#import <Foundation/Foundation.h>

typedef void (^EventHandler)(NSDictionary *eventInfo);

@interface EventEmitter : NSObject

- (void)on:(NSString *)event handler:(EventHandler)handler;
- (void)emit:(NSString *)event info:(NSDictionary *)info;
- (void)off:(NSString *)event;

@end

@implementation EventEmitter {
    NSMutableDictionary<NSString *, NSMutableArray<EventHandler> *> *_handlers;
}

- (instancetype)init {
    self = [super init];
    if (self) {
        _handlers = [NSMutableDictionary dictionary];
    }
    return self;
}

- (void)on:(NSString *)event handler:(EventHandler)handler {
    if (!_handlers[event]) {
        _handlers[event] = [NSMutableArray array];
    }
    [_handlers[event] addObject:[handler copy]];
}

- (void)emit:(NSString *)event info:(NSDictionary *)info {
    NSArray *handlers = [_handlers[event] copy];
    for (EventHandler handler in handlers) {
        handler(info ?: @{});
    }
}

- (void)off:(NSString *)event {
    [_handlers removeObjectForKey:event];
}

@end

int main(int argc, const char * argv[]) {
    @autoreleasepool {
        EventEmitter *emitter = [[EventEmitter alloc] init];
        
        // ลงทะเบียน handlers
        [emitter on:@"data" handler:^(NSDictionary *info) {
            NSLog(@"Handler 1 received data: %@", info[@"value"]);
        }];
        
        [emitter on:@"data" handler:^(NSDictionary *info) {
            NSLog(@"Handler 2 - processing: %@", info[@"value"]);
        }];
        
        [emitter on:@"error" handler:^(NSDictionary *info) {
            NSLog(@"Error occurred: %@", info[@"message"]);
        }];
        
        // emit events
        [emitter emit:@"data" info:@{@"value": @"Hello, World!"}];
        [emitter emit:@"error" info:@{@"message": @"Something went wrong"}];
        [emitter emit:@"unknown" info:nil]; // ไม่มี handler - ไม่มีอะไรเกิดขึ้น
    }
    return 0;
}
```

### 31.10.3 Middleware / Pipeline Pattern

```objc
#import <Foundation/Foundation.h>

typedef NSString *(^TransformBlock)(NSString *input);
typedef BOOL (^FilterBlock)(NSString *input);

@interface Pipeline : NSObject

- (Pipeline *)addTransform:(TransformBlock)transform;
- (Pipeline *)addFilter:(FilterBlock)filter;
- (NSArray<NSString *> *)process:(NSArray<NSString *> *)inputs;

@end

@implementation Pipeline {
    NSMutableArray *_steps; // array ของ blocks
}

- (instancetype)init {
    self = [super init];
    if (self) {
        _steps = [NSMutableArray array];
    }
    return self;
}

- (Pipeline *)addTransform:(TransformBlock)transform {
    [_steps addObject:@{@"type": @"transform", @"block": [transform copy]}];
    return self;
}

- (Pipeline *)addFilter:(FilterBlock)filter {
    [_steps addObject:@{@"type": @"filter", @"block": [filter copy]}];
    return self;
}

- (NSArray<NSString *> *)process:(NSArray<NSString *> *)inputs {
    NSMutableArray *current = [inputs mutableCopy];
    
    for (NSDictionary *step in _steps) {
        NSMutableArray *next = [NSMutableArray array];
        
        if ([step[@"type"] isEqualToString:@"transform"]) {
            TransformBlock transform = step[@"block"];
            for (NSString *item in current) {
                [next addObject:transform(item)];
            }
        } else if ([step[@"type"] isEqualToString:@"filter"]) {
            FilterBlock filter = step[@"block"];
            for (NSString *item in current) {
                if (filter(item)) {
                    [next addObject:item];
                }
            }
        }
        
        current = next;
    }
    
    return [current copy];
}

@end

int main(int argc, const char * argv[]) {
    @autoreleasepool {
        NSArray *words = @[@"  Hello  ", @"world", @"  OBJECTIVE-C  ",
                            @"blocks", @"are", @"AWESOME  "];
        
        Pipeline *pipeline = [[Pipeline alloc] init];
        
        // trim whitespace
        [pipeline addTransform:^NSString *(NSString *s) {
            return [s stringByTrimmingCharactersInSet:[NSCharacterSet whitespaceCharacterSet]];
        }];
        
        // convert to lowercase
        [pipeline addTransform:^NSString *(NSString *s) {
            return [s lowercaseString];
        }];
        
        // filter: เก็บเฉพาะคำที่ยาวกว่า 3 ตัวอักษร
        [pipeline addFilter:^BOOL(NSString *s) {
            return s.length > 3;
        }];
        
        // เพิ่ม prefix
        [pipeline addTransform:^NSString *(NSString *s) {
            return [@">>> " stringByAppendingString:s];
        }];
        
        NSArray *result = [pipeline process:words];
        NSLog(@"Input:  %@", words);
        NSLog(@"Output: %@", result);
    }
    return 0;
}
```

---

## 31.11 GCD Basics กับ Blocks

(รายละเอียดเต็มใน Part 32)

```objc
#import <Foundation/Foundation.h>

int main(int argc, const char * argv[]) {
    @autoreleasepool {
        // dispatch_async - รัน block ใน background thread
        dispatch_queue_t backgroundQueue = dispatch_get_global_queue(DISPATCH_QUEUE_PRIORITY_DEFAULT, 0);
        dispatch_queue_t mainQueue = dispatch_get_main_queue();
        
        NSLog(@"เริ่มต้น - thread: %@", [NSThread currentThread]);
        
        // รัน block แบบ async
        dispatch_async(backgroundQueue, ^{
            // ทำงานหนักใน background
            NSLog(@"Background work - thread: %@", [NSThread currentThread]);
            
            NSString *result = @"ผลลัพธ์จาก background";
            
            // อัพเดท UI ใน main thread
            dispatch_async(mainQueue, ^{
                NSLog(@"Update UI: %@ - thread: %@",
                      result, [NSThread currentThread]);
            });
        });
        
        // dispatch_after - รัน block หลังจาก delay
        dispatch_after(dispatch_time(DISPATCH_TIME_NOW, (int64_t)(1.0 * NSEC_PER_SEC)),
                       mainQueue, ^{
            NSLog(@"หลังจาก 1 วินาที");
        });
        
        // รอให้ async tasks เสร็จ (ใน real app ใช้ dispatch_group หรือ completion block)
        [[NSRunLoop mainRunLoop] runUntilDate:[NSDate dateWithTimeIntervalSinceNow:2.0]];
    }
    return 0;
}
```

---

## 31.12 Block กับ Recursive Calls

```objc
#import <Foundation/Foundation.h>

int main(int argc, const char * argv[]) {
    @autoreleasepool {
        // Block ที่เรียกตัวเองซ้ำ (recursive) - ต้องใช้ __block
        __block NSInteger (^factorial)(NSInteger);
        factorial = ^NSInteger(NSInteger n) {
            if (n <= 1) return 1;
            return n * factorial(n - 1);
        };
        
        NSLog(@"5! = %ld", (long)factorial(5));   // 120
        NSLog(@"10! = %ld", (long)factorial(10)); // 3628800
        
        // fibonacci
        __block NSInteger (^fib)(NSInteger);
        fib = ^NSInteger(NSInteger n) {
            if (n <= 1) return n;
            return fib(n-1) + fib(n-2);
        };
        
        for (NSInteger i = 0; i < 10; i++) {
            NSLog(@"fib(%ld) = %ld", (long)i, (long)fib(i));
        }
    }
    return 0;
}
```

---

## 31.13 Block ใน Protocol / Delegate Alternative

```objc
#import <Foundation/Foundation.h>

// แทนที่จะใช้ delegate protocol สามารถใช้ blocks ได้
@interface Downloader : NSObject

// Block-based callbacks (alternative to delegate)
typedef void (^DownloadProgressBlock)(float progress);
typedef void (^DownloadCompletionBlock)(NSData *data, NSError *error);

- (void)downloadURL:(NSString *)url
           progress:(DownloadProgressBlock)progress
         completion:(DownloadCompletionBlock)completion;

@end

@implementation Downloader

- (void)downloadURL:(NSString *)url
           progress:(DownloadProgressBlock)progress
         completion:(DownloadCompletionBlock)completion {
    
    NSLog(@"Starting download: %@", url);
    
    // จำลอง download
    for (float p = 0; p <= 1.0f; p += 0.2f) {
        if (progress) progress(p);
    }
    
    if ([url containsString:@"fail"]) {
        NSError *error = [NSError errorWithDomain:@"com.download" code:404
                                         userInfo:@{NSLocalizedDescriptionKey: @"ไม่พบ URL"}];
        if (completion) completion(nil, error);
    } else {
        NSData *mockData = [url dataUsingEncoding:NSUTF8StringEncoding];
        if (completion) completion(mockData, nil);
    }
}

@end

int main(int argc, const char * argv[]) {
    @autoreleasepool {
        Downloader *dl = [[Downloader alloc] init];
        
        [dl downloadURL:@"https://example.com/file.zip"
               progress:^(float progress) {
            NSLog(@"Progress: %.0f%%", progress * 100);
        } completion:^(NSData *data, NSError *error) {
            if (data) {
                NSLog(@"Downloaded %lu bytes", (unsigned long)data.length);
            } else {
                NSLog(@"Download failed: %@", error.localizedDescription);
            }
        }];
    }
    return 0;
}
```

---

## แบบฝึกหัด (Practice Exercises)

### แบบฝึกหัดที่ 1: Block Basics
สร้าง block ที่รับ array ของตัวเลขและคืนค่าผลรวม

```objc
// เฉลย
NSInteger (^sumArray)(NSArray *) = ^NSInteger(NSArray *numbers) {
    NSInteger total = 0;
    for (NSNumber *n in numbers) {
        total += [n integerValue];
    }
    return total;
};

NSArray *nums = @[@1, @2, @3, @4, @5];
NSLog(@"ผลรวม: %ld", (long)sumArray(nums)); // 15
```

### แบบฝึกหัดที่ 2: Capturing ตัวแปร
สร้าง counter function factory ที่ return block

```objc
// สร้าง counter ที่เริ่มจากค่าที่กำหนด
void (^)(void) makeCounter(NSInteger start, NSInteger step) {
    // ไม่สามารถเขียนแบบนี้ได้ใน C - ต้องใช้ typedef
}

// แก้ด้วย typedef
typedef void (^CounterBlock)(void);

CounterBlock makeCounter(NSInteger start, NSInteger step) {
    __block NSInteger current = start;
    return ^{
        NSLog(@"Count: %ld", (long)current);
        current += step;
    };
}

// ทดสอบ
CounterBlock countBy1 = makeCounter(0, 1);
CounterBlock countBy5 = makeCounter(100, 5);

countBy1(); // 0
countBy1(); // 1
countBy1(); // 2

countBy5(); // 100
countBy5(); // 105
```

### แบบฝึกหัดที่ 3: Filter และ Map
เขียน function filter และ map สำหรับ NSArray ที่ใช้ block

```objc
// Filter
NSArray* filter(NSArray *arr, BOOL (^predicate)(id)) {
    NSMutableArray *result = [NSMutableArray array];
    for (id item in arr) {
        if (predicate(item)) [result addObject:item];
    }
    return [result copy];
}

// Map
NSArray* map(NSArray *arr, id (^transform)(id)) {
    NSMutableArray *result = [NSMutableArray array];
    for (id item in arr) {
        [result addObject:transform(item)];
    }
    return [result copy];
}

// ทดสอบ
NSArray *numbers = @[@1, @2, @3, @4, @5, @6, @7, @8, @9, @10];

// เอาเลขคู่แล้วคูณ 10
NSArray *evenDoubled = map(
    filter(numbers, ^BOOL(id n) { return [n integerValue] % 2 == 0; }),
    ^id(id n) { return @([n integerValue] * 10); }
);

NSLog(@"เลขคู่คูณ 10: %@", evenDoubled); // [20, 40, 60, 80, 100]
```

### แบบฝึกหัดที่ 4: Memoize
สร้าง memoize function ที่ cache ผลลัพธ์ของ block

```objc
typedef NSInteger (^IntTransform)(NSInteger);

IntTransform memoize(IntTransform block) {
    __block NSMutableDictionary *cache = [NSMutableDictionary dictionary];
    return ^NSInteger(NSInteger n) {
        NSNumber *key = @(n);
        if (cache[key]) {
            NSLog(@"Cache hit for %ld", (long)n);
            return [cache[key] integerValue];
        }
        NSInteger result = block(n);
        cache[key] = @(result);
        return result;
    };
}

// ทดสอบ - slow fibonacci ที่ถูก memoize
__block IntTransform memoFib;
IntTransform slowFib = ^NSInteger(NSInteger n) {
    if (n <= 1) return n;
    return memoFib(n-1) + memoFib(n-2);
};
memoFib = memoize(slowFib);

NSLog(@"fib(10) = %ld", (long)memoFib(10));
NSLog(@"fib(10) again = %ld", (long)memoFib(10)); // จาก cache
```

### แบบฝึกหัดที่ 5: Debounce
สร้าง debounce function ที่จะรัน block เฉพาะเมื่อไม่มีการเรียกซ้ำใน interval ที่กำหนด

```objc
typedef dispatch_block_t VoidBlock;

VoidBlock debounce(double delay, void (^block)(void)) {
    __block BOOL pending = NO;
    return ^{
        if (!pending) {
            pending = YES;
            dispatch_after(dispatch_time(DISPATCH_TIME_NOW, (int64_t)(delay * NSEC_PER_SEC)),
                           dispatch_get_main_queue(), ^{
                pending = NO;
                block();
            });
        }
    };
}

// ทดสอบ - เรียก 5 ครั้ง แต่ควรรัน block เพียงครั้งเดียว
VoidBlock debouncedSearch = debounce(0.3, ^{
    NSLog(@"Search executed!");
});

debouncedSearch(); // trigger 1
debouncedSearch(); // trigger 2 - ยกเลิก trigger 1
debouncedSearch(); // trigger 3 - ยกเลิก trigger 2
// รอ 0.3 วินาที -> "Search executed!" (ครั้งเดียว)
```

### แบบฝึกหัดที่ 6: Once
สร้าง function ที่ให้ block ทำงานได้เพียงครั้งเดียว

```objc
typedef void (^ActionBlock)(void);

ActionBlock once(ActionBlock block) {
    __block BOOL called = NO;
    return ^{
        if (!called) {
            called = YES;
            block();
        } else {
            NSLog(@"Block already called, ignoring");
        }
    };
}

// ทดสอบ
ActionBlock initAction = once(^{
    NSLog(@"Initialization! (ควรเห็นครั้งเดียว)");
});

initAction(); // รัน
initAction(); // ไม่รัน
initAction(); // ไม่รัน
```

### แบบฝึกหัดที่ 7: Chaining
สร้าง chainable block pattern

```objc
@interface NumberTransformer : NSObject

@property (nonatomic, assign) double value;

typedef NumberTransformer *(^TransformChain)(double);

- (TransformChain)add;
- (TransformChain)multiply;
- (TransformChain)subtract;

@end

@implementation NumberTransformer

- (TransformChain)add {
    return ^NumberTransformer *(double n) {
        self.value += n;
        return self;
    };
}

- (TransformChain)multiply {
    return ^NumberTransformer *(double n) {
        self.value *= n;
        return self;
    };
}

- (TransformChain)subtract {
    return ^NumberTransformer *(double n) {
        self.value -= n;
        return self;
    };
}

@end

// ทดสอบ
NumberTransformer *t = [[NumberTransformer alloc] init];
t.value = 10;

// chain: (10 + 5) * 2 - 3 = 27
t.add(5).multiply(2).subtract(3);
NSLog(@"Result: %.0f", t.value); // 27
```

### แบบฝึกหัดที่ 8: Retry Block
เขียน function ที่ retry block จนกว่าจะสำเร็จหรือเกิน limit

```objc
typedef BOOL (^AttemptBlock)(NSInteger attempt, NSError **error);

BOOL retryBlock(NSInteger maxAttempts, AttemptBlock attempt) {
    NSError *error = nil;
    for (NSInteger i = 1; i <= maxAttempts; i++) {
        error = nil;
        if (attempt(i, &error)) {
            NSLog(@"สำเร็จในครั้งที่ %ld", (long)i);
            return YES;
        }
        NSLog(@"ครั้งที่ %ld ล้มเหลว: %@", (long)i, error.localizedDescription);
    }
    return NO;
}

// ทดสอบ
__block NSInteger callCount = 0;
BOOL ok = retryBlock(3, ^BOOL(NSInteger attempt, NSError **err) {
    callCount++;
    if (callCount < 3) {
        if (err) *err = [NSError errorWithDomain:@"test" code:1
                                        userInfo:@{NSLocalizedDescriptionKey: @"ยังไม่สำเร็จ"}];
        return NO;
    }
    return YES;
});
NSLog(@"Result: %@ (%ld attempts)", ok ? @"OK" : @"FAIL", (long)callCount);
```

### แบบฝึกหัดที่ 9: Weak-Strong Dance
แก้ retain cycle ในโค้ดต่อไปนี้

```objc
// โค้ดที่มีปัญหา retain cycle:
@interface ViewController : NSObject
@property (nonatomic, copy) void (^updateHandler)(void);
@property (nonatomic, strong) NSString *title;
@end

@implementation ViewController
- (void)setup {
    // ❌ retain cycle: self -> updateHandler -> self
    self.updateHandler = ^{
        NSLog(@"Title: %@", self.title);
    };
}
@end

// เฉลย: แก้ด้วย __weak/__strong dance
@implementation ViewController
- (void)setup {
    // ✅ ไม่มี retain cycle
    __weak typeof(self) weakSelf = self;
    self.updateHandler = ^{
        __strong typeof(weakSelf) strongSelf = weakSelf;
        if (!strongSelf) return;
        NSLog(@"Title: %@", strongSelf.title);
    };
}
@end
```

### แบบฝึกหัดที่ 10: Custom Sort
เรียงลำดับ array ของ dictionary ด้วย sort block ที่ซับซ้อน

```objc
NSArray *students = @[
    @{@"name": @"Alice",   @"score": @85, @"grade": @"B"},
    @{@"name": @"Bob",     @"score": @92, @"grade": @"A"},
    @{@"name": @"Charlie", @"score": @78, @"grade": @"C"},
    @{@"name": @"Diana",   @"score": @92, @"grade": @"A"},
    @{@"name": @"Eve",     @"score": @85, @"grade": @"B"},
];

// เรียงตามคะแนน (มาก->น้อย) ถ้าคะแนนเท่ากันเรียงตามชื่อ (ก->ฮ)
NSArray *sorted = [students sortedArrayUsingComparator:^NSComparisonResult(id a, id b) {
    NSInteger scoreA = [a[@"score"] integerValue];
    NSInteger scoreB = [b[@"score"] integerValue];
    
    if (scoreB > scoreA) return NSOrderedAscending;
    if (scoreB < scoreA) return NSOrderedDescending;
    
    // คะแนนเท่ากัน เรียงตามชื่อ
    return [a[@"name"] compare:b[@"name"]];
}];

for (NSDictionary *s in sorted) {
    NSLog(@"%-10s: %@ (%@)", [s[@"name"] UTF8String], s[@"score"], s[@"grade"]);
}
```

### แบบฝึกหัดที่ 11: Curry
สร้าง curry function ด้วย blocks

```objc
// Curry: แปลงฟังก์ชัน f(a,b) เป็น f(a)(b)
typedef NSInteger (^IntBinaryOp)(NSInteger, NSInteger);
typedef NSInteger (^IntUnaryOp)(NSInteger);
typedef IntUnaryOp (^CurriedOp)(NSInteger);

CurriedOp curry(IntBinaryOp op) {
    return ^IntUnaryOp(NSInteger a) {
        return ^NSInteger(NSInteger b) {
            return op(a, b);
        };
    };
}

// ทดสอบ
CurriedOp curriedAdd = curry(^NSInteger(NSInteger a, NSInteger b) {
    return a + b;
});

IntUnaryOp addFive = curriedAdd(5);  // partial application
IntUnaryOp addTen  = curriedAdd(10);

NSLog(@"addFive(3) = %ld", (long)addFive(3));  // 8
NSLog(@"addTen(3)  = %ld", (long)addTen(3));   // 13
NSLog(@"addFive(addTen(2)) = %ld", (long)addFive(addTen(2))); // 17
```

### แบบฝึกหัดที่ 12: Event Queue
สร้าง event queue ที่รัน blocks ทีละตัว

```objc
@interface EventQueue : NSObject
- (void)enqueue:(void (^)(void))block;
- (void)processAll;
@end

@implementation EventQueue {
    NSMutableArray<void (^)(void)> *_queue;
}

- (instancetype)init {
    self = [super init];
    if (self) {
        _queue = [NSMutableArray array];
    }
    return self;
}

- (void)enqueue:(void (^)(void))block {
    [_queue addObject:[block copy]];
}

- (void)processAll {
    while (_queue.count > 0) {
        void (^block)(void) = _queue[0];
        [_queue removeObjectAtIndex:0];
        block();
    }
}

@end

// ทดสอบ
EventQueue *queue = [[EventQueue alloc] init];
[queue enqueue:^{ NSLog(@"Event 1: Initialize"); }];
[queue enqueue:^{ NSLog(@"Event 2: Load data"); }];
[queue enqueue:^{ NSLog(@"Event 3: Process"); }];
[queue enqueue:^{ NSLog(@"Event 4: Cleanup"); }];

NSLog(@"Processing events:");
[queue processAll];
```

### แบบฝึกหัดที่ 13: Notification Handler
จำลอง NSNotificationCenter แบบง่ายด้วย blocks

```objc
typedef void (^NotificationHandler)(NSDictionary *userInfo);

@interface SimpleNotificationCenter : NSObject
+ (instancetype)defaultCenter;
- (void)addObserver:(id)observer
              name:(NSString *)name
           handler:(NotificationHandler)handler;
- (void)postNotificationName:(NSString *)name userInfo:(NSDictionary *)userInfo;
- (void)removeObserver:(id)observer;
@end

@implementation SimpleNotificationCenter {
    NSMutableDictionary<NSString *, NSMutableArray *> *_observers;
}

+ (instancetype)defaultCenter {
    static SimpleNotificationCenter *center;
    static dispatch_once_t onceToken;
    dispatch_once(&onceToken, ^{
        center = [[self alloc] init];
    });
    return center;
}

- (instancetype)init {
    self = [super init];
    if (self) {
        _observers = [NSMutableDictionary dictionary];
    }
    return self;
}

- (void)addObserver:(id)observer name:(NSString *)name handler:(NotificationHandler)handler {
    if (!_observers[name]) {
        _observers[name] = [NSMutableArray array];
    }
    [_observers[name] addObject:@{
        @"observer": observer,
        @"handler": [handler copy]
    }];
}

- (void)postNotificationName:(NSString *)name userInfo:(NSDictionary *)userInfo {
    for (NSDictionary *entry in [_observers[name] copy]) {
        NotificationHandler handler = entry[@"handler"];
        handler(userInfo ?: @{});
    }
}

- (void)removeObserver:(id)observer {
    for (NSString *name in [_observers allKeys]) {
        NSMutableArray *obs = _observers[name];
        [obs filterUsingPredicate:[NSPredicate predicateWithBlock:^BOOL(id obj, NSDictionary *b) {
            return obj[@"observer"] != observer;
        }]];
    }
}

@end

// ทดสอบ
SimpleNotificationCenter *nc = [SimpleNotificationCenter defaultCenter];
id observer1 = [[NSObject alloc] init];

[nc addObserver:observer1
          name:@"DataLoaded"
       handler:^(NSDictionary *userInfo) {
    NSLog(@"Observer 1: Data loaded - %@", userInfo[@"data"]);
}];

[nc addObserver:observer1
          name:@"DataLoaded"
       handler:^(NSDictionary *userInfo) {
    NSLog(@"Observer 2: Also got data - count: %@", userInfo[@"count"]);
}];

[nc postNotificationName:@"DataLoaded"
               userInfo:@{@"data": @"Hello", @"count": @42}];
```

### แบบฝึกหัดที่ 14: Lazy Evaluation
ใช้ block สำหรับ lazy evaluation (คำนวณเมื่อจำเป็น)

```objc
// Lazy value - คำนวณเพียงครั้งเดียวเมื่อถูกเรียก
@interface Lazy : NSObject

+ (instancetype)with:(id (^)(void))provider;
- (id)value;

@end

@implementation Lazy {
    id (^_provider)(void);
    id _cachedValue;
    BOOL _computed;
}

+ (instancetype)with:(id (^)(void))provider {
    Lazy *l = [[self alloc] init];
    l->_provider = [provider copy];
    l->_computed = NO;
    return l;
}

- (id)value {
    if (!_computed) {
        _cachedValue = _provider();
        _computed = YES;
        NSLog(@"(Computed!)");
    }
    return _cachedValue;
}

@end

// ทดสอบ
Lazy *expensive = [Lazy with:^id{
    NSLog(@"Running expensive computation...");
    // จำลองการคำนวณที่ใช้เวลา
    NSInteger result = 0;
    for (int i = 0; i < 1000000; i++) result += i;
    return @(result);
}];

NSLog(@"Lazy value created (not yet computed)");
NSLog(@"First access: %@", expensive.value);  // คำนวณ
NSLog(@"Second access: %@", expensive.value); // จาก cache
NSLog(@"Third access: %@", expensive.value);  // จาก cache
```

### แบบฝึกหัดที่ 15: State Machine
สร้าง simple state machine ด้วย blocks

```objc
typedef NS_ENUM(NSInteger, TrafficLightState) {
    TrafficLightRed = 0,
    TrafficLightGreen,
    TrafficLightYellow,
};

@interface TrafficLight : NSObject
@property (nonatomic, assign) TrafficLightState state;
@property (nonatomic, copy) void (^onStateChange)(TrafficLightState newState,
                                                   TrafficLightState oldState);
- (void)next;
- (NSString *)currentStateName;
@end

@implementation TrafficLight

- (instancetype)init {
    self = [super init];
    if (self) {
        _state = TrafficLightRed;
    }
    return self;
}

- (void)next {
    TrafficLightState oldState = _state;
    switch (_state) {
        case TrafficLightRed:    _state = TrafficLightGreen;  break;
        case TrafficLightGreen:  _state = TrafficLightYellow; break;
        case TrafficLightYellow: _state = TrafficLightRed;    break;
    }
    if (self.onStateChange) {
        self.onStateChange(_state, oldState);
    }
}

- (NSString *)currentStateName {
    switch (_state) {
        case TrafficLightRed:    return @"แดง 🔴";
        case TrafficLightGreen:  return @"เขียว 🟢";
        case TrafficLightYellow: return @"เหลือง 🟡";
    }
    return @"ไม่รู้จัก";
}

@end

// ทดสอบ
TrafficLight *light = [[TrafficLight alloc] init];
light.onStateChange = ^(TrafficLightState newState, TrafficLightState oldState) {
    NSLog(@"เปลี่ยนจาก %ld -> %ld", (long)oldState, (long)newState);
};

NSLog(@"เริ่มต้น: %@", [light currentStateName]);
[light next]; NSLog(@"ตอนนี้: %@", [light currentStateName]);
[light next]; NSLog(@"ตอนนี้: %@", [light currentStateName]);
[light next]; NSLog(@"ตอนนี้: %@", [light currentStateName]);
[light next]; NSLog(@"ตอนนี้: %@", [light currentStateName]);
```

---

## สรุป

ในบทนี้เราได้เรียนรู้:

1. **Block Syntax** - `^returnType(params){ body }` และ variations
2. **Capturing Variables** - by value (default) และ by reference (__block)
3. **Block as Parameter** - ส่ง block ให้ function
4. **Block return values** - คืนค่าจาก block
5. **Properties กับ Block** - เก็บ block ใน property (ใช้ copy attribute)
6. **typedef** - ทำ block type อ่านง่ายขึ้น
7. **Memory Management** - retain cycle และ __weak/__strong dance
8. **NSArray Methods** - enumerateObjectsUsingBlock:, sortedArrayUsingComparator:, filteredArrayUsingPredicate:
9. **Patterns** - Builder, Observer, Pipeline, Completion Handler
10. **Recursive Blocks** - ใช้ __block สำหรับ self-reference

ในบทถัดไปเราจะเรียนรู้ **Grand Central Dispatch (GCD)** ที่ใช้ blocks สำหรับ concurrency
