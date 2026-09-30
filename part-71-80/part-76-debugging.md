# Part 76: Debugging ใน Objective-C

## บทนำ

Debugging คือทักษะที่สำคัญที่สุดอย่างหนึ่งสำหรับนักพัฒนาซอฟต์แวร์ การหาและแก้ไข bugs อย่างมีประสิทธิภาพสามารถประหยัดเวลาได้มาก Xcode มาพร้อมกับเครื่องมือ debugging ที่ทรงพลัง รวมถึง LLDB debugger, Memory Graph Debugger, และ Sanitizers ต่าง ๆ บทนี้จะครอบคลุมเครื่องมือเหล่านี้อย่างละเอียด

---

## 76.1 Xcode Debugger (LLDB)

LLDB (Low Level Debugger) คือ debugger หลักของ Xcode ซึ่งรองรับการ debug ทั้ง Objective-C และ Swift

### เข้าถึง LLDB

```
วิธีที่ 1: ตั้ง breakpoint แล้ว run แอป
วิธีที่ 2: Debug > Attach to Process
วิธีที่ 3: Debug > Debug Executable
วิธีที่ 4: ใช้ (lldb) prompt ใน Xcode console
```

### Debug Area ใน Xcode

```
Debug Area ประกอบด้วย:
- Variables View (ซ้าย): แสดง variables ใน scope ปัจจุบัน
- Console (ขวา): แสดง output และรับ LLDB commands
- Debug Toolbar: ปุ่ม Continue, Step Over, Step Into, Step Out
```

---

## 76.2 Breakpoints

Breakpoints คือจุดที่โปรแกรมจะหยุดทำงานเพื่อให้เราตรวจสอบ state

### Line Breakpoint

```
วิธีตั้ง Line Breakpoint:
1. คลิกที่ gutter (ซ้ายของ line number)
2. กด ⌘\ เมื่อ cursor อยู่บน line นั้น
```

```objc
// โค้ดตัวอย่างสำหรับ debugging
- (void)processOrder:(Order *)order {
    // ตั้ง breakpoint ที่นี่เพื่อตรวจสอบ order
    NSArray *items = order.items;     // <-- Breakpoint
    double total = 0;
    
    for (OrderItem *item in items) {
        total += item.price * item.quantity;
    }
    
    order.total = total;
}
```

### Symbolic Breakpoint

Symbolic breakpoint หยุดเมื่อ function/method ถูกเรียก โดยไม่ต้องรู้ line number

```
ตั้ง Symbolic Breakpoint:
1. ไปที่ Breakpoint Navigator (⌘7)
2. กดปุ่ม + ด้านล่าง
3. เลือก "Symbolic Breakpoint"
4. ใส่ symbol name

ตัวอย่าง symbols:
- [UIViewController viewDidLoad]     (specific method)
- viewDidLoad                         (any viewDidLoad)
- -[NSArray objectAtIndex:]           (array access)
- UIApplicationMain                   (app entry point)
- objc_msgSend                        (all ObjC message sends)
```

### Conditional Breakpoint

```objc
// หยุดเฉพาะเมื่อเงื่อนไขเป็นจริง

// ตัวอย่าง: หยุดเมื่อ indexPath.row == 5
- (UITableViewCell *)tableView:(UITableView *)tableView 
         cellForRowAtIndexPath:(NSIndexPath *)indexPath {
    // ตั้ง conditional breakpoint: indexPath.row == 5
    
    ProductCell *cell = [tableView dequeueReusableCellWithIdentifier:@"ProductCell" 
                                                        forIndexPath:indexPath];
    [cell configureWithProduct:self.products[indexPath.row]];
    return cell;
}
```

```
ตั้ง Conditional Breakpoint:
1. Right-click บน breakpoint
2. เลือก "Edit Breakpoint"
3. ใส่ Condition: indexPath.row == 5
4. ใส่ Ignore count (ข้าม n ครั้งก่อนหยุด)
```

### Exception Breakpoint

```
ตั้ง Exception Breakpoint เพื่อหยุดเมื่อเกิด exception:
1. Breakpoint Navigator (⌘7)
2. กด + > "Exception Breakpoint"
3. เลือก Exception: Objective-C
4. เลือก Break: On Throw หรือ On Catch
```

```objc
// Exception ที่พบบ่อย

// NSRangeException - array out of bounds
NSArray *array = @[@"a", @"b", @"c"];
// ❌ จะ throw NSRangeException
id item = array[10]; // index 10 ไม่มี

// NSInvalidArgumentException - nil argument
// ❌ ส่ง nil ไปยัง method ที่ไม่รับ nil
[NSString stringWithFormat:nil]; 

// NSUnknownKeyException - KVC
// ❌ Key ไม่มีอยู่จริง
[object valueForKey:@"nonexistentProperty"];
```

### Action Breakpoint

```objc
// Breakpoint ที่ทำ action แทนการหยุด

// Log ค่าตัวแปร โดยไม่ต้องหยุด program
// ตั้ง Action Breakpoint:
// Action: Log Message: "Order total: @total@, items: @items.count@"
// ติ๊ก "Automatically continue after evaluating actions"

- (void)calculateTotal:(Order *)order {
    double total = 0;
    for (OrderItem *item in order.items) {
        total += item.price * item.quantity;  // <-- Action Breakpoint ที่นี่
    }
    order.total = total;
}
```

---

## 76.3 LLDB Commands

LLDB มี command-line interface ที่ทรงพลังมาก

### po (Print Object)

```
(lldb) po <expression>

ตัวอย่าง:
(lldb) po self
(lldb) po self.view
(lldb) po self.dataArray
(lldb) po self.dataArray.count
(lldb) po [self.user name]
(lldb) po self.user.email
```

```objc
// ตัวอย่างผลลัพธ์ po
(lldb) po self.user
<User: 0x600000234b20>
  userId = "user-123"
  name = "John Doe"
  email = "john@example.com"
  createdAt = 2024-01-01 00:00:00 +0000

(lldb) po self.products
<__NSArrayM 0x600000123456>
  [0] = <Product: ...> (iPhone 15)
  [1] = <Product: ...> (AirPods Pro)
  [2] = <Product: ...> (MacBook Air)
```

### p (Print)

```
(lldb) p <expression>

p แสดงผล raw value รวมถึง type information
ใช้สำหรับ primitive types หรือเมื่อต้องการ type info

ตัวอย่าง:
(lldb) p self.count
(NSUInteger) $0 = 42

(lldb) p total
(double) $1 = 99.99

(lldb) p (int)[array count]
(int) $2 = 5
```

### bt (Backtrace)

```
(lldb) bt

แสดง call stack ทั้งหมด ช่วยเข้าใจว่า crash เกิดที่ไหน

ผลลัพธ์:
* thread #1, queue = 'com.apple.main-thread'
  * frame #0: 0x00000001 MyApp`-[OrderService processOrder:] + 45
    frame #1: 0x00000002 MyApp`-[CheckoutViewController checkout:] + 123
    frame #2: 0x00000003 MyApp`-[UIButton sendAction:to:from:forEvent:] + 96
    frame #3: 0x00000004 UIKitCore`...

// ดู full backtrace ของ thread ทั้งหมด
(lldb) bt all
```

### frame commands

```
// ดู frame ปัจจุบัน
(lldb) frame info

// ไปยัง frame ที่ต้องการ
(lldb) frame select 2

// ดู variables ใน frame
(lldb) frame variable
(lldb) frame variable order
(lldb) frame variable -r order  // recursive display

// ดู source code ที่ frame นั้น
(lldb) frame select 1
(lldb) list
```

### thread commands

```
// ดู threads ทั้งหมด
(lldb) thread list

// เลือก thread
(lldb) thread select 2

// ดู thread info
(lldb) thread info

// Continue เฉพาะ thread นี้
(lldb) thread continue

// Backtrace ของ thread ที่เลือก
(lldb) thread backtrace
```

### expr (Expression)

```
// รันโค้ดระหว่าง debugging
(lldb) expr self.userName = @"Debug User"
(lldb) expr [self.tableView reloadData]
(lldb) expr (void)NSLog(@"Debug value: %@", self.value)

// สร้าง object ใหม่
(lldb) expr NSString *$myStr = @"Test String"
(lldb) po $myStr
```

### Commands อื่น ๆ ที่มีประโยชน์

```
// Continue execution
(lldb) c
(lldb) continue

// Step over (ข้าม function call)
(lldb) n
(lldb) next

// Step into (เข้าไปใน function)
(lldb) s
(lldb) step

// Step out (ออกจาก function ปัจจุบัน)
(lldb) finish

// ดู memory ที่ address
(lldb) memory read 0x600000234b20
(lldb) x/16xb 0x600000234b20

// ดู registers
(lldb) register read
(lldb) register read rax rip

// หา object ใน memory
(lldb) po [UIApp windows]
```

---

## 76.4 Debug View Hierarchy

View Hierarchy Debugger ช่วยตรวจสอบ UI ที่ runtime

### เปิด View Hierarchy Debugger

```
วิธีที่ 1: Debug > View Debugging > Capture View Hierarchy
วิธีที่ 2: กดปุ่ม Debug View Hierarchy ใน debug toolbar (icon กล่อง 3 มิติ)
วิธีที่ 3: ผ่าน LLDB: (lldb) expr @import UIKit; [UIApp windows]
```

### วิเคราะห์ View Hierarchy

```objc
// ดู view hierarchy ผ่าน code
- (void)debugPrintViewHierarchy:(UIView *)view indent:(NSInteger)indent {
    NSMutableString *indentStr = [NSMutableString string];
    for (int i = 0; i < indent; i++) {
        [indentStr appendString:@"  "];
    }
    
    NSLog(@"%@%@ frame=%@ hidden=%d alpha=%.2f", 
          indentStr,
          NSStringFromClass([view class]),
          NSStringFromCGRect(view.frame),
          view.isHidden,
          view.alpha);
    
    for (UIView *subview in view.subviews) {
        [self debugPrintViewHierarchy:subview indent:indent + 1];
    }
}

// ใช้งาน
- (void)viewDidLoad {
    [super viewDidLoad];
    [self debugPrintViewHierarchy:self.view indent:0];
}
```

### Debug Constraint Issues

```objc
// เพิ่ม identifier ให้ constraints สำหรับ debugging
NSLayoutConstraint *topConstraint = [view.topAnchor constraintEqualToAnchor:superview.topAnchor constant:20];
topConstraint.identifier = @"myView.topConstraint";
[topConstraint setActive:YES];

// Debug constraint conflicts
// ใน console จะเห็น:
// Unable to simultaneously satisfy constraints.
//   Probably at least one of the constraints in the following list is one you don't want.
//   Try this: (1) look at each constraint and try to figure out which you don't expect;
//   (2) find the code that added the unwanted constraint or constraints and fix it.
// (
//     "<NSLayoutConstraint:0x... myView.topConstraint>",
//     "<NSLayoutConstraint:0x... UIView:0x...top == UIView:0x...bottom + 20>",
// )
```

---

## 76.5 Memory Graph Debugger

Memory Graph Debugger แสดง object graph ใน memory ณ เวลาหนึ่ง

### เปิด Memory Graph Debugger

```
Debug > Memory Graph Debugger
หรือกดปุ่ม Memory Graph ใน debug toolbar (icon graph)
```

### วิเคราะห์ Memory Graph

```objc
// ตัวอย่าง retain cycle ที่ Memory Graph จะแสดง
@interface ViewController : UIViewController

@property (nonatomic, strong) DataLoader *loader;

@end

@implementation ViewController

- (void)viewDidLoad {
    self.loader = [[DataLoader alloc] init];
    
    // ❌ Retain cycle:
    // ViewController → loader (strong)
    // loader.completionBlock → ViewController (strong capture)
    self.loader.onComplete = ^(NSArray *data) {
        [self.tableView reloadData]; // self retained strongly
    };
}

@end
```

### ใช้ Memory Graph กับ NSZombie

```
เปิด NSZombie:
Edit Scheme > Run > Diagnostics > Zombie Objects (ติ๊ก)

NSZombie ช่วยตรวจสอบ:
- การ message object ที่ถูก deallocate แล้ว
- Over-release bugs (ใน MRC)
```

```objc
// ตัวอย่าง use-after-free bug ที่ NSZombie จะจับได้
@implementation BadCode

- (void)demonstrateBug {
    NSObject *obj = [[NSObject alloc] init];
    
    // ใน MRC: 
    // [obj release]; // Deallocate
    // [obj description]; // ❌ Use after free - NSZombie จะแสดง error
    
    // ใน ARC:
    // __unsafe_unretained NSObject *unsafeRef = obj;
    // obj = nil; // Released
    // [unsafeRef description]; // ❌ Use after free
}

@end
```

---

## 76.6 NSZombie

NSZombie เป็นเทคนิคสำหรับตรวจจับ message-to-deallocated-object bugs

### วิธีทำงานของ NSZombie

```
ปกติ: Object released → memory freed
NSZombie: Object released → converted to zombie → ไม่ free memory
                          → ถ้ามีการ message → แสดง error + crash

NSZombie error message:
*** -[NSString description]: message sent to deallocated instance 0x600000234b20
```

### เปิดใช้งาน NSZombie

```
วิธีที่ 1: Xcode Scheme
Product > Scheme > Edit Scheme > Run > Diagnostics > Zombie Objects

วิธีที่ 2: Environment Variable (ใน Scheme)
NSZombieEnabled = YES

วิธีที่ 3: LLDB
(lldb) e (void)[[NSUserDefaults standardUserDefaults] setBool:YES forKey:@"NSZombiesEnabled"]
```

### ตัวอย่างการใช้ NSZombie

```objc
// ตัวอย่างปัญหาที่ NSZombie ช่วยได้
@implementation NotificationManager

- (void)startListening {
    // ❌ Observer อาจถูก deallocate ก่อนที่จะ removeObserver
    [[NSNotificationCenter defaultCenter] addObserver:self
                                             selector:@selector(handleNotification:)
                                                 name:@"DataUpdated"
                                               object:nil];
}

// ❌ ถ้า ViewController deallocate โดยไม่เรียก stopListening
// NSNotificationCenter จะ message zombie object

// ✅ แก้ไข
- (void)dealloc {
    [[NSNotificationCenter defaultCenter] removeObserver:self];
}

@end
```

---

## 76.7 Address Sanitizer (ASan)

Address Sanitizer ตรวจจับ memory errors ต่าง ๆ

### ประเภท Memory Errors ที่ ASan ตรวจจับ

```
1. Use after free
2. Heap buffer overflow
3. Stack buffer overflow
4. Use after scope
5. Use after return
6. Double free
7. Invalid free
```

### เปิดใช้งาน ASan

```
Product > Scheme > Edit Scheme > Run > Diagnostics > Address Sanitizer (ติ๊ก)

หรือ Build Settings:
CLANG_ADDRESS_SANITIZER = YES
```

### ตัวอย่าง ASan ในการทำงาน

```objc
// ASan จะตรวจจับ bugs เหล่านี้

// 1. Buffer overflow
void demonstrateBufferOverflow() {
    char buffer[10];
    // ❌ ASan จะจับ: heap-buffer-overflow
    // write 11 bytes to 10-byte buffer
    strcpy(buffer, "HelloWorld!"); // 11 chars + null = 12 bytes
}

// 2. Use after free
void demonstrateUseAfterFree() {
    int *ptr = (int *)malloc(sizeof(int));
    *ptr = 42;
    free(ptr);
    // ❌ ASan จะจับ: heap-use-after-free
    int value = *ptr; // Accessing freed memory
}

// 3. Objective-C specific
- (void)demonstrateObjCBug {
    NSMutableArray *array = @[@"a", @"b", @"c"].mutableCopy;
    
    // ❌ Mutation ระหว่าง fast enumeration
    // ASan + runtime check จะจับ: mutation during enumeration
    for (NSString *item in array) {
        [array removeObject:item]; // ❌ Modifying while iterating
    }
}
```

### ASan Error Report

```
ERROR: AddressSanitizer: heap-use-after-free on address 0x602000004290

READ of size 8 at 0x602000004290 thread T0
    #0 0x1049d234 in -[BadClass badMethod] BadClass.m:45
    #1 0x1049c123 in -[MainViewController viewDidLoad] MainViewController.m:89

0x602000004290 was freed at:
    #0 0x7fff53a23456 in free libsystem_malloc.dylib
    #1 0x1049d100 in -[BadClass releaseObject] BadClass.m:30
```

---

## 76.8 Thread Sanitizer (TSan)

Thread Sanitizer ตรวจจับ race conditions และ thread safety issues

### เปิดใช้งาน TSan

```
Product > Scheme > Edit Scheme > Run > Diagnostics > Thread Sanitizer (ติ๊ก)

หมายเหตุ: ไม่สามารถใช้ ASan และ TSan พร้อมกันได้
```

### ตัวอย่าง Race Conditions

```objc
// ❌ Data race - TSan จะตรวจจับ
@interface UnsafeCounter : NSObject
@property (nonatomic, assign) NSInteger count;
@end

@implementation UnsafeCounter

- (void)increment {
    // ❌ Data race: หลาย threads access count พร้อมกัน
    self.count++; // Not atomic!
}

@end

// การใช้งานที่ก่อ race condition
UnsafeCounter *counter = [[UnsafeCounter alloc] init];

dispatch_queue_t queue = dispatch_queue_create("test", DISPATCH_QUEUE_CONCURRENT);
for (int i = 0; i < 1000; i++) {
    dispatch_async(queue, ^{
        [counter increment]; // ❌ Race condition!
    });
}
```

### แก้ไข Race Conditions

```objc
// วิธีที่ 1: Serial Queue
@interface ThreadSafeCounter : NSObject

@property (nonatomic, assign) NSInteger count;

@end

@implementation ThreadSafeCounter {
    dispatch_queue_t _queue;
}

- (instancetype)init {
    self = [super init];
    if (self) {
        _queue = dispatch_queue_create("com.app.counter", DISPATCH_QUEUE_SERIAL);
    }
    return self;
}

- (void)increment {
    dispatch_async(_queue, ^{
        self.count++;
    });
}

- (NSInteger)count {
    __block NSInteger count;
    dispatch_sync(_queue, ^{
        count = self->_count;
    });
    return count;
}

@end

// วิธีที่ 2: Atomic operations
#import <stdatomic.h>

@interface AtomicCounter : NSObject
@end

@implementation AtomicCounter {
    atomic_long _count;
}

- (void)increment {
    atomic_fetch_add(&_count, 1);
}

- (long)count {
    return atomic_load(&_count);
}

@end

// วิธีที่ 3: OSSpinLock / os_unfair_lock (iOS 10+)
#import <os/lock.h>

@interface LockedResource : NSObject
@end

@implementation LockedResource {
    os_unfair_lock _lock;
    NSMutableDictionary *_data;
}

- (instancetype)init {
    self = [super init];
    if (self) {
        _lock = OS_UNFAIR_LOCK_INIT;
        _data = [NSMutableDictionary dictionary];
    }
    return self;
}

- (void)setValue:(id)value forKey:(NSString *)key {
    os_unfair_lock_lock(&_lock);
    _data[key] = value;
    os_unfair_lock_unlock(&_lock);
}

- (id)valueForKey:(NSString *)key {
    os_unfair_lock_lock(&_lock);
    id value = _data[key];
    os_unfair_lock_unlock(&_lock);
    return value;
}

@end
```

### TSan Error Report

```
WARNING: ThreadSanitizer: data race (pid=12345)
  Write of size 8 at 0x7fc1234 by thread T2:
    #0 -[UnsafeCounter setCount:] UnsafeCounter.m:15
    #1 -[UnsafeCounter increment] UnsafeCounter.m:20
    #2 __30-[Tests testConcurrent]_block_invoke Tests.m:45

  Previous read of size 8 at 0x7fc1234 by thread T3:
    #0 -[UnsafeCounter count] UnsafeCounter.m:12
    #1 __30-[Tests testConcurrent]_block_invoke Tests.m:47
```

---

## 76.9 Undefined Behavior Sanitizer (UBSan)

UBSan ตรวจจับ undefined behavior ใน C/C++ code

### เปิดใช้งาน UBSan

```
Build Settings > CLANG_UNDEFINED_BEHAVIOR_SANITIZER = YES

หรือใน Xcode:
Product > Scheme > Edit Scheme > Run > Diagnostics > Undefined Behavior Sanitizer
```

### ตัวอย่าง Undefined Behavior

```objc
// UBSan จะจับ behaviors เหล่านี้

// 1. Integer overflow
void integerOverflow() {
    int maxInt = INT_MAX; // 2147483647
    // ❌ Undefined behavior: signed integer overflow
    int overflow = maxInt + 1;
}

// 2. Null pointer dereference
void nullDereference() {
    int *ptr = NULL;
    // ❌ Undefined behavior
    int value = *ptr;
}

// 3. Division by zero
void divByZero(int divisor) {
    int result = 42 / divisor; // ❌ ถ้า divisor == 0
}

// 4. Array out of bounds (C arrays)
void arrayBounds() {
    int arr[5] = {1, 2, 3, 4, 5};
    // ❌ Undefined behavior: access beyond bounds
    int value = arr[10];
}

// 5. Invalid shift operations
void invalidShift() {
    int x = 1;
    // ❌ Shift amount ≥ bit width
    int result = x << 32; // undefined for 32-bit int
}
```

---

## 76.10 Logging Strategies

การ logging ที่ดีช่วยให้ debug ได้ง่ายขึ้น

### NSLog

```objc
// NSLog พื้นฐาน
NSLog(@"Simple message");
NSLog(@"Value: %@", someObject);
NSLog(@"Number: %ld", (long)count);
NSLog(@"Float: %.2f", price);

// NSLog format specifiers
NSLog(@"%@ = NSObject (or nil)");
NSLog(@"%d = int");
NSLog(@"%ld = NSInteger (long)");
NSLog(@"%lu = NSUInteger (unsigned long)");
NSLog(@"%f = double/float");
NSLog(@"%.2f = float with 2 decimal places");
NSLog(@"%@ = NSString");
NSLog(@"%p = pointer address");
NSLog(@"%zu = size_t");

// NSLog กับ struct
CGRect frame = CGRectMake(0, 0, 100, 200);
NSLog(@"Frame: %@", NSStringFromCGRect(frame));
NSLog(@"Size: %@", NSStringFromCGSize(frame.size));
NSLog(@"Point: %@", NSStringFromCGPoint(frame.origin));
```

### Custom Logger

```objc
// Log levels
typedef NS_ENUM(NSInteger, LogLevel) {
    LogLevelVerbose = 0,
    LogLevelDebug,
    LogLevelInfo,
    LogLevelWarning,
    LogLevelError,
    LogLevelFatal
};

// Logger implementation
@interface AppLogger : NSObject

+ (instancetype)sharedLogger;

@property (nonatomic, assign) LogLevel minimumLevel;
@property (nonatomic, assign) BOOL includeTimestamp;
@property (nonatomic, assign) BOOL includeFileInfo;
@property (nonatomic, strong) NSMutableArray<NSString *> *logHistory;

- (void)log:(NSString *)message 
      level:(LogLevel)level 
       file:(const char *)file 
       line:(int)line 
   function:(const char *)function;

@end

@implementation AppLogger

+ (instancetype)sharedLogger {
    static AppLogger *instance;
    static dispatch_once_t onceToken;
    dispatch_once(&onceToken, ^{
        instance = [[AppLogger alloc] init];
    });
    return instance;
}

- (instancetype)init {
    self = [super init];
    if (self) {
        _minimumLevel = LogLevelDebug;
        _includeTimestamp = YES;
        _includeFileInfo = YES;
        _logHistory = [NSMutableArray array];
    }
    return self;
}

- (void)log:(NSString *)message 
      level:(LogLevel)level 
       file:(const char *)file 
       line:(int)line 
   function:(const char *)function {
    
    if (level < self.minimumLevel) return;
    
    NSMutableString *logMessage = [NSMutableString string];
    
    // Level prefix
    NSString *levelStr;
    switch (level) {
        case LogLevelVerbose: levelStr = @"[V]"; break;
        case LogLevelDebug:   levelStr = @"[D]"; break;
        case LogLevelInfo:    levelStr = @"[I]"; break;
        case LogLevelWarning: levelStr = @"[W]"; break;
        case LogLevelError:   levelStr = @"[E]"; break;
        case LogLevelFatal:   levelStr = @"[F]"; break;
    }
    
    // Timestamp
    if (self.includeTimestamp) {
        NSDateFormatter *formatter = [[NSDateFormatter alloc] init];
        formatter.dateFormat = @"HH:mm:ss.SSS";
        [logMessage appendFormat:@"%@ ", [formatter stringFromDate:[NSDate date]]];
    }
    
    [logMessage appendFormat:@"%@ ", levelStr];
    
    // File info
    if (self.includeFileInfo) {
        NSString *fileName = [[NSString stringWithUTF8String:file] lastPathComponent];
        [logMessage appendFormat:@"(%@:%d) ", fileName, line];
    }
    
    [logMessage appendString:message];
    
    NSLog(@"%@", logMessage);
    
    // Store in history
    if (self.logHistory.count < 1000) {
        [self.logHistory addObject:[logMessage copy]];
    }
}

@end

// Convenience macros
#define LOG_VERBOSE(msg, ...) \
    [[AppLogger sharedLogger] log:[NSString stringWithFormat:msg, ##__VA_ARGS__] \
                            level:LogLevelVerbose \
                             file:__FILE__ \
                             line:__LINE__ \
                         function:__FUNCTION__]

#define LOG_DEBUG(msg, ...) \
    [[AppLogger sharedLogger] log:[NSString stringWithFormat:msg, ##__VA_ARGS__] \
                            level:LogLevelDebug \
                             file:__FILE__ \
                             line:__LINE__ \
                         function:__FUNCTION__]

#define LOG_INFO(msg, ...) \
    [[AppLogger sharedLogger] log:[NSString stringWithFormat:msg, ##__VA_ARGS__] \
                            level:LogLevelInfo \
                             file:__FILE__ \
                             line:__LINE__ \
                         function:__FUNCTION__]

#define LOG_ERROR(msg, ...) \
    [[AppLogger sharedLogger] log:[NSString stringWithFormat:msg, ##__VA_ARGS__] \
                            level:LogLevelError \
                             file:__FILE__ \
                             line:__LINE__ \
                         function:__FUNCTION__]

// การใช้งาน
- (void)processPayment:(Payment *)payment {
    LOG_INFO(@"Processing payment: %@", payment.transactionId);
    
    if (!payment.isValid) {
        LOG_WARNING(@"Invalid payment: %@", payment.transactionId);
        return;
    }
    
    PaymentResult *result = [self.gateway charge:payment];
    
    if (result.success) {
        LOG_INFO(@"Payment successful: %@", payment.transactionId);
    } else {
        LOG_ERROR(@"Payment failed: %@ - %@", payment.transactionId, result.errorMessage);
    }
}
```

### os_log (Unified Logging - iOS 10+)

```objc
#import <os/log.h>

// สร้าง logger categories
static os_log_t networkLog;
static os_log_t databaseLog;
static os_log_t uiLog;

+ (void)initialize {
    networkLog = os_log_create("com.myapp", "network");
    databaseLog = os_log_create("com.myapp", "database");
    uiLog = os_log_create("com.myapp", "ui");
}

// Log ด้วย os_log
- (void)fetchDataFromAPI {
    os_log(networkLog, "Fetching data from API");
    os_log_info(networkLog, "Request URL: %{public}@", self.apiURL);
    
    // Private data (ไม่แสดงใน Console ถ้าไม่ได้ enable)
    os_log_debug(networkLog, "Auth token: %{private}@", self.authToken);
    
    // Log fault (serious errors)
    if (!self.apiURL) {
        os_log_fault(networkLog, "API URL is nil - cannot make request");
        return;
    }
}

// ดู logs ใน Console.app
// Filter: process:MyApp subsystem:com.myapp category:network
```

---

## 76.11 Crash Logs และ Symbolication

### อ่าน Crash Log

```
Crash Log มีโครงสร้างดังนี้:

1. Header: ข้อมูล device, OS, app version
2. Exception Information: ประเภทของ crash
3. Thread Backtraces: stack trace ของ thread ต่าง ๆ
4. Binary Images: frameworks ที่ loaded
```

```
ตัวอย่าง Crash Log:

Exception Type: SIGABRT
Exception Codes: 0x0000000000000000
Termination Reason: abort() called

Thread 0 Crashed:
0  libsystem_kernel.dylib         0x1bc1231 __pthread_kill + 8
1  libsystem_pthread.dylib        0x1c3450f pthread_kill + 272
2  libsystem_c.dylib              0x1ba54f3 abort + 120
3  MyApp                          0x10048f2 -[UserService validateUser:] + 234
4  MyApp                          0x1004234 -[CheckoutViewController placeOrder:] + 156
```

### Symbolication

Symbolication แปลง memory addresses ให้เป็น function names และ line numbers

```
วิธีที่ 1: Xcode Organizer
Window > Organizer > Crashes
Xcode ทำ symbolication อัตโนมัติ

วิธีที่ 2: atos command
atos -arch arm64 -o MyApp.dSYM/Contents/Resources/DWARF/MyApp -l 0x100000000 0x10048f2

วิธีที่ 3: symbolicatecrash (Xcode tool)
```

### เก็บ dSYM สำหรับ Symbolication

```
dSYM (Debug Symbol) ต้องเก็บให้ตรงกับ app binary

การตั้งค่า:
Build Settings > Debug Information Format = DWARF with dSYM File
Build Settings > Strip Debug Symbols = NO (สำหรับ Debug)

Bitcode ถ้า enable: Apple generate dSYM ให้ใน App Store Connect
```

### Firebase Crashlytics Integration

```objc
// ใช้ Crashlytics สำหรับ crash reporting
#import <Firebase/Firebase.h>

// ส่ง custom crash info
- (void)userDidTapCheckout {
    // Log user action ก่อน crash
    [[FIRCrashlytics crashlytics] log:@"User initiated checkout"];
    [[FIRCrashlytics crashlytics] setUserID:self.currentUser.userId];
    
    [self performCheckout];
}

// Non-fatal error
- (void)handleNetworkError:(NSError *)error {
    [[FIRCrashlytics crashlytics] recordError:error];
    [self showErrorAlert:error];
}

// Force crash สำหรับ testing
- (void)testCrashReporting {
    [[FIRCrashlytics crashlytics] log:@"Testing crash reporting"];
    [FIRCrashlytics.crashlytics crash];
}
```

---

## 76.12 Advanced Debugging Techniques

### Watchpoints

```
Watchpoints หยุด execution เมื่อตัวแปรถูกเปลี่ยนค่า

ตั้ง watchpoint:
1. ระหว่าง debugging, คลิกขวาที่ variable ใน Variables View
2. เลือก "Watch <variable>"
หรือ LLDB:
(lldb) watchpoint set variable self->_count
(lldb) watchpoint set expression -w read_write -- &self->_count
```

### LLDB Python Scripting

```python
# Custom LLDB command ด้วย Python
import lldb

def print_view_hierarchy(debugger, command, result, internal_dict):
    """Print UIView hierarchy"""
    target = debugger.GetSelectedTarget()
    process = target.GetProcess()
    thread = process.GetSelectedThread()
    frame = thread.GetSelectedFrame()
    
    # Run expression
    options = lldb.SBExpressionOptions()
    options.SetLanguage(lldb.eLanguageTypeObjC)
    
    value = frame.EvaluateExpression(
        '(NSString *)[[UIApplication sharedApplication].windows[0] recursiveDescription]',
        options
    )
    print(value.GetObjectDescription())

# Load script ใน LLDB
# (lldb) command script import ~/lldb_scripts/view_hierarchy.py
# (lldb) command script add -f view_hierarchy.print_view_hierarchy vh
# (lldb) vh
```

### Debug Build Configurations

```objc
// ใช้ preprocessor macros สำหรับ debug code
#ifdef DEBUG

// Debug-only code
#define DebugLog(fmt, ...) NSLog((@"%s [Line %d] " fmt), __PRETTY_FUNCTION__, __LINE__, ##__VA_ARGS__)

// Assertions ที่ active เฉพาะ debug
#define DebugAssert(condition, message) \
    do { \
        if (!(condition)) { \
            NSLog(@"ASSERTION FAILED: %@", message); \
            NSLog(@"In %s line %d", __PRETTY_FUNCTION__, __LINE__); \
            abort(); \
        } \
    } while(0)

#else
#define DebugLog(...)
#define DebugAssert(condition, message)
#endif

// การใช้งาน
- (void)processData:(NSArray *)data {
    DebugAssert(data != nil, @"data must not be nil");
    DebugAssert(data.count > 0, @"data must not be empty");
    
    DebugLog(@"Processing %lu items", (unsigned long)data.count);
    
    for (id item in data) {
        // process...
    }
}
```

### Debugging Network Issues

```objc
// Charles Proxy Integration
// เพิ่ม proxy settings สำหรับ debug
#ifdef DEBUG

+ (NSURLSessionConfiguration *)debugConfiguration {
    NSURLSessionConfiguration *config = [NSURLSessionConfiguration defaultSessionConfiguration];
    
    // Route through Charles Proxy
    config.connectionProxyDictionary = @{
        (NSString *)kCFNetworkProxiesHTTPEnable: @YES,
        (NSString *)kCFNetworkProxiesHTTPProxy: @"localhost",
        (NSString *)kCFNetworkProxiesHTTPPort: @8888,
        (NSString *)kCFNetworkProxiesHTTPSEnable: @YES,
        (NSString *)kCFNetworkProxiesHTTPSProxy: @"localhost",
        (NSString *)kCFNetworkProxiesHTTPSPort: @8888,
    };
    
    return config;
}

#endif
```

---

## 76.13 Debugging Common Issues

### NSArray Index Out of Bounds

```objc
// ❌ Crash: NSRangeException
- (void)showItem:(NSInteger)index {
    // ไม่ตรวจสอบ bounds
    Product *product = self.products[index]; // Crash ถ้า index out of range
    [self displayProduct:product];
}

// ✅ ปลอดภัย
- (void)showItem:(NSInteger)index {
    if (index < 0 || index >= self.products.count) {
        NSLog(@"Invalid index: %ld, count: %lu", (long)index, (unsigned long)self.products.count);
        return;
    }
    
    Product *product = self.products[index];
    [self displayProduct:product];
}

// ✅ ใช้ safe accessor category
@implementation NSArray (SafeAccess)

- (id)safeObjectAtIndex:(NSUInteger)index {
    if (index >= self.count) {
        NSLog(@"Warning: index %lu out of bounds (count: %lu)", 
              (unsigned long)index, (unsigned long)self.count);
        return nil;
    }
    return [self objectAtIndex:index];
}

@end
```

### NSDictionary Nil Key

```objc
// ❌ Crash: NSInvalidArgumentException (nil key)
- (void)addToCache:(id)object forKey:(NSString *)key {
    self.cache[key] = object; // Crash ถ้า key เป็น nil
}

// ✅ ตรวจสอบ nil ก่อน
- (void)addToCache:(id)object forKey:(NSString *)key {
    if (!key) {
        NSLog(@"Warning: Attempting to add object with nil key");
        return;
    }
    if (!object) {
        [self.cache removeObjectForKey:key];
        return;
    }
    self.cache[key] = object;
}
```

### KVO Crash

```objc
// ❌ Crash: ไม่ removeObserver ก่อน dealloc
@implementation SomeViewController

- (void)viewDidLoad {
    [self.model addObserver:self 
                forKeyPath:@"value" 
                   options:NSKeyValueObservingOptionNew 
                   context:nil];
}

// ❌ ลืม removeObserver
// - (void)dealloc ไม่มี removeObserver

@end

// ✅ เพิ่ม removeObserver ใน dealloc
- (void)dealloc {
    [self.model removeObserver:self forKeyPath:@"value"];
}

// ✅ ดีกว่า: ใช้ KVO helper
@interface KVOHelper : NSObject

- (instancetype)observeObject:(NSObject *)object 
                      keyPath:(NSString *)keyPath 
                       change:(void(^)(id newValue))changeBlock;

@end

@implementation KVOHelper {
    NSObject *_object;
    NSString *_keyPath;
    void(^_changeBlock)(id);
}

- (instancetype)observeObject:(NSObject *)object 
                      keyPath:(NSString *)keyPath 
                       change:(void(^)(id newValue))changeBlock {
    self = [super init];
    if (self) {
        _object = object;
        _keyPath = keyPath;
        _changeBlock = changeBlock;
        [object addObserver:self forKeyPath:keyPath options:NSKeyValueObservingOptionNew context:nil];
    }
    return self;
}

- (void)observeValueForKeyPath:(NSString *)keyPath ofObject:(id)object change:(NSDictionary *)change context:(void *)context {
    if (_changeBlock) {
        _changeBlock(change[NSKeyValueChangeNewKey]);
    }
}

- (void)dealloc {
    [_object removeObserver:self forKeyPath:_keyPath];
}

@end

// การใช้งาน - KVOHelper จัดการ lifecycle อัตโนมัติ
@interface MyViewController : UIViewController
@property (nonatomic, strong) KVOHelper *kvoHelper;
@end

@implementation MyViewController

- (void)viewDidLoad {
    self.kvoHelper = [[KVOHelper alloc] observeObject:self.model
                                              keyPath:@"value"
                                               change:^(id newValue) {
        [self updateUI:newValue];
    }];
    // เมื่อ kvoHelper deallocated จะ removeObserver อัตโนมัติ
}

@end
```

---

## 76.14 Debugging Tips and Best Practices

### 1. ใช้ NSParameterAssert และ NSAssert

```objc
- (void)saveUser:(User *)user toDatabase:(id<DatabaseProtocol>)database {
    // Assert parameters ที่ต้อง non-nil
    NSParameterAssert(user != nil);
    NSParameterAssert(database != nil);
    NSParameterAssert(user.userId.length > 0);
    
    [database save:user];
}

// NSAssert ใน logic
- (double)calculateDiscount:(double)price forUser:(User *)user {
    NSAssert(price >= 0, @"Price cannot be negative");
    NSAssert(user.membershipLevel >= 0 && user.membershipLevel <= 5, 
             @"Invalid membership level: %ld", (long)user.membershipLevel);
    
    // Calculate...
    return price * 0.1;
}
```

### 2. Override description สำหรับ better debugging

```objc
@implementation Order

- (NSString *)description {
    return [NSString stringWithFormat:
            @"<Order: orderId=%@, total=%.2f, items=%lu, status=%@>",
            self.orderId,
            self.total,
            (unsigned long)self.items.count,
            self.statusString];
}

- (NSString *)debugDescription {
    return [NSString stringWithFormat:
            @"<Order: %p\n"
            @"  orderId: %@\n"
            @"  total: %.2f\n"
            @"  items: %@\n"
            @"  status: %@\n"
            @"  createdAt: %@>",
            self,
            self.orderId,
            self.total,
            self.items,
            self.statusString,
            self.createdAt];
}

@end
```

### 3. ใช้ breakpoint actions สำหรับ logging

```
ตั้ง breakpoint action:
1. Right-click breakpoint > Edit Breakpoint
2. Add Action > Log Message
3. ใส่: "User: @(self.user)@, Count: @(self.count)@"
4. ติ๊ก "Automatically continue after evaluating actions"

ผลลัพธ์: Log โดยไม่หยุด program - useful สำหรับ tracing
```

### 4. Debug ด้วย print statements ที่ถูกวิธี

```objc
// ❌ ไม่ดี: ลืม remove ก่อน production
- (void)processItem:(id)item {
    NSLog(@"DEBUG: processing item: %@", item);
    // ...
}

// ✅ ดี: ใช้ conditional compile
- (void)processItem:(id)item {
    #ifdef DEBUG
    NSLog(@"Processing item: %@", item);
    #endif
    // ...
}

// ✅ ดีที่สุด: ใช้ custom logger ที่ปิดได้
- (void)processItem:(id)item {
    AppLog(@"Processing item: %@", item); // Custom macro ที่ใช้ os_log
    // ...
}
```

---

## 76.15 แบบฝึกหัด

### แบบฝึกหัดที่ 1: ตั้ง Breakpoints หลายชนิด

1. ตั้ง line breakpoint ที่ method `processPayment:`
2. ตั้ง conditional breakpoint ที่หยุดเมื่อ `amount > 1000`
3. ตั้ง exception breakpoint สำหรับ Objective-C exceptions
4. ใช้ action breakpoint เพื่อ log ค่าตัวแปรโดยไม่หยุด

### แบบฝึกหัดที่ 2: LLDB Practice

ระหว่าง debugging ลอง:
```
(lldb) po self
(lldb) po self.view.subviews
(lldb) p self.count
(lldb) bt
(lldb) frame select 2
(lldb) expr self.title = @"Debug Mode"
```

### แบบฝึกหัดที่ 3: หา Memory Leaks

```objc
// หาและแก้ leaks ในโค้ดนี้
@interface LeakyClass : NSObject
@property (nonatomic, strong) NSTimer *timer;
@property (nonatomic, copy) void(^callback)(void);
@end

@implementation LeakyClass

- (void)start {
    self.timer = [NSTimer scheduledTimerWithTimeInterval:1.0 
                                                  target:self  
                                                selector:@selector(tick)
                                                userInfo:nil 
                                                 repeats:YES];
    
    self.callback = ^{
        [self doSomething]; // Retain cycle?
    };
}

- (void)tick {
    self.callback();
}

- (void)doSomething { }

@end
```

### แบบฝึกหัดที่ 4: Thread Safety

```objc
// ทำให้ code นี้ thread-safe
@interface SharedCache : NSObject

@property (nonatomic, strong) NSMutableDictionary *storage;

- (void)setValue:(id)value forKey:(NSString *)key;
- (id)valueForKey:(NSString *)key;
- (void)removeAll;

@end

@implementation SharedCache

- (void)setValue:(id)value forKey:(NSString *)key {
    // TODO: ทำให้ thread-safe
    self.storage[key] = value;
}

// TODO: implement thread-safe methods

@end
```

### แบบฝึกหัดที่ 5: Crash Reproduction

สร้าง test case ที่:
1. จำลอง crash จาก crash log ที่ให้มา
2. แก้ไข bug
3. เขียน test เพื่อ prevent regression

---

## สรุป

Debugging เป็นทักษะที่พัฒนาได้จากการฝึกฝน เครื่องมือใน Xcode มีความสามารถสูงมาก:

### เครื่องมือหลัก:
- **LLDB**: debugger ที่ทรงพลัง รองรับ po, p, bt, expr
- **Breakpoints**: line, symbolic, conditional, exception, action
- **View Hierarchy**: debug UI ที่ runtime
- **Memory Graph**: ดู object relationships และหา retain cycles
- **NSZombie**: จับ message-to-deallocated-object

### Sanitizers:
- **Address Sanitizer (ASan)**: memory errors - buffer overflow, use-after-free
- **Thread Sanitizer (TSan)**: race conditions
- **Undefined Behavior Sanitizer (UBSan)**: undefined behavior ใน C/C++

### Best Practices:
- ใช้ assertions เพื่อตรวจสอบ preconditions
- override description/debugDescription สำหรับ readable debug output
- ใช้ conditional compilation สำหรับ debug-only code
- เก็บ dSYM ไฟล์เพื่อ symbolicate crash logs
- ใช้ crash reporting tools เช่น Firebase Crashlytics

การ debug ที่ดีไม่ใช่แค่การใช้เครื่องมือ แต่ต้องมีกระบวนการคิดที่เป็นระบบ:
1. **Reproduce**: ทำให้ bug เกิดซ้ำได้อย่างสม่ำเสมอ
2. **Isolate**: จำกัดขอบเขตของปัญหาให้แคบที่สุด
3. **Identify**: หาสาเหตุที่แท้จริง
4. **Fix**: แก้ไขอย่างถูกวิธี
5. **Verify**: ยืนยันว่า fix ถูกต้องและไม่ก่อ regression

---

*จบ Part 76 - Debugging ใน Objective-C*
