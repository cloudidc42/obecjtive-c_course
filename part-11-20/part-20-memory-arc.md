# ตอนที่ 20: ARC (Automatic Reference Counting) - การจัดการ Memory อัตโนมัติ

## บทนำ

**Automatic Reference Counting (ARC)** คือระบบที่ compiler แทรก `retain`, `release`, และ `autorelease` ให้อัตโนมัติในเวลา compile ทำให้โปรแกรมเมอร์ไม่ต้องจัดการ memory ด้วยตนเอง

ARC ถูกเปิดตัวใน iOS 5 / Mac OS X 10.7 (Lion) และกลายเป็น default สำหรับ Objective-C และ Swift ใน Xcode

### ARC คืออะไร (ไม่ใช่ Garbage Collection!)

ความเข้าใจผิดที่พบบ่อย: ARC ≠ Garbage Collection (GC)

| | ARC | Garbage Collection |
|--|--|--|
| เวลาทำงาน | Compile time | Runtime |
| ประสิทธิภาพ | Deterministic, เร็ว | Non-deterministic, ช้ากว่า |
| Memory freed | ทันทีเมื่อ count = 0 | ต้องรอ GC cycle |
| Pause | ไม่มี pause | อาจมี pause |
| Platform | iOS, macOS | macOS เท่านั้น (deprecated) |

---

## 20.1 How ARC Works (Compiler Inserts retain/release)

### ARC ทำงานอย่างไร

ARC วิเคราะห์โค้ดในเวลา compile และแทรก retain/release ในตำแหน่งที่เหมาะสม:

```objc
// โค้ดที่เราเขียน (ARC)
- (void)example {
    NSString *str = [[NSString alloc] initWithString:@"Hello"];
    NSLog(@"%@", str);
    // str ออกจาก scope
}

// สิ่งที่ compiler สร้าง (เทียบเท่า)
- (void)example {
    NSString *str = [[NSString alloc] initWithString:@"Hello"];
    NSLog(@"%@", str);
    [str release];  // ← ARC แทรกให้
}
```

```objc
// โค้ดที่ซับซ้อนกว่า (ARC)
- (NSString *)createGreeting:(NSString *)name {
    NSString *greeting = [NSString stringWithFormat:@"Hello, %@!", name];
    return greeting;
}

- (void)useGreeting {
    NSString *g1 = [self createGreeting:@"World"];
    NSString *g2 = g1;  // strong reference
    g1 = nil;           // ลด count ของ original string
    NSLog(@"%@", g2);   // ยังใช้ได้เพราะ g2 ยัง strong
    // ออกจาก scope: g2 ถูก release
}
```

### ข้อห้ามใน ARC

ใน ARC ไม่สามารถเรียกสิ่งเหล่านี้โดยตรง:

```objc
// ❌ ไม่ได้ใน ARC
[obj retain];
[obj release];
[obj autorelease];
[obj retainCount];  // ไม่ควรใช้
NSAllocateObject();
NSDeallocateObject();
```

---

## 20.2 ARC vs Garbage Collection

### ความแตกต่างที่สำคัญ

```
ARC (Automatic Reference Counting):
┌─────────────────────────────────────┐
│  Compile Time Analysis              │
│  Compiler inserts retain/release    │
│  Deterministic deallocation        │
│  No runtime overhead for GC        │
│  Works everywhere: iOS + macOS     │
└─────────────────────────────────────┘

Garbage Collection (Deprecated in macOS 10.8):
┌─────────────────────────────────────┐
│  Runtime Analysis                   │
│  GC thread traces reachability     │
│  Non-deterministic deallocation    │
│  Runtime overhead                  │
│  Only on macOS (removed)           │
└─────────────────────────────────────┘
```

### ทำไม ARC ดีกว่า GC สำหรับ iOS/macOS

1. **Deterministic**: รู้แน่นอนว่า object จะถูก deallocate เมื่อ count = 0
2. **No pause**: ไม่มี GC pause ที่ทำให้ app ค้าง
3. **Lower memory**: ไม่ต้องเก็บ objects ที่รอ GC collect
4. **Faster**: ไม่มี overhead ของ GC thread

---

## 20.3 Strong References (__strong)

### __strong คืออะไร

`__strong` คือ default qualifier สำหรับ object variables ใน ARC มันบอกว่า variable นี้เป็น "เจ้าของ" object:

```objc
// ทั้งสองบรรทัดเหมือนกัน - __strong เป็น default
NSString *str1 = @"Hello";
__strong NSString *str2 = @"Hello";
```

### Strong Reference เพิ่ม Retain Count

```objc
- (void)demonstrateStrong {
    NSObject *obj = [[NSObject alloc] init];  // count = 1
    
    NSObject *ref1 = obj;  // __strong: count = 2
    NSObject *ref2 = obj;  // __strong: count = 3
    
    ref1 = nil;  // count = 2
    ref2 = nil;  // count = 1
    
    // obj ออกจาก scope: count = 0, dealloc
}
```

### Strong Properties

```objc
@interface Library : NSObject

@property (nonatomic, strong) NSString *name;          // เป็นเจ้าของ string
@property (nonatomic, strong) NSMutableArray *books;   // เป็นเจ้าของ array

@end

@implementation Library

- (instancetype)init {
    self = [super init];
    if (self) {
        _name = @"Central Library";
        _books = [[NSMutableArray alloc] init];
    }
    return self;
}

- (void)addBook:(NSString *)title {
    [_books addObject:title];
}

@end
```

### Class-Level Strong References

```objc
@interface DataManager : NSObject

// Strong property เก็บ data
@property (nonatomic, strong) NSMutableDictionary *cache;
@property (nonatomic, strong) NSOperationQueue *operationQueue;

@end

@implementation DataManager

- (instancetype)init {
    self = [super init];
    if (self) {
        _cache = [[NSMutableDictionary alloc] init];
        _operationQueue = [[NSOperationQueue alloc] init];
        _operationQueue.maxConcurrentOperationCount = 4;
    }
    return self;
}

@end
```

---

## 20.4 Weak References (__weak)

### __weak คืออะไร

`__weak` สร้าง reference ที่ไม่เพิ่ม retain count และจะถูก set เป็น `nil` อัตโนมัติเมื่อ object ที่ชี้ไปถูก deallocate:

```objc
@interface Person : NSObject
@property (nonatomic, strong) NSString *name;
@end

// ตัวอย่าง weak reference
Person *person = [[Person alloc] init];
person.name = @"สมชาย";

__weak Person *weakPerson = person;  // ไม่เพิ่ม retain count

NSLog(@"Strong: %@", person.name);      // สมชาย
NSLog(@"Weak: %@", weakPerson.name);    // สมชาย - ยังมีอยู่เพราะ person ยัง strong

person = nil;  // strong reference หายไป, count = 0, dealloc

NSLog(@"Weak after dealloc: %@", weakPerson);  // (null) - zeroing weak reference
```

### Weak Properties สำหรับ Delegates

```objc
// NetworkService.h
@protocol NetworkServiceDelegate <NSObject>
- (void)networkService:(id)service didReceiveData:(NSData *)data;
- (void)networkService:(id)service didFailWithError:(NSError *)error;
@end

@interface NetworkService : NSObject

@property (nonatomic, weak) id<NetworkServiceDelegate> delegate;  // ← weak!

- (void)fetchDataFromURL:(NSString *)urlString;

@end
```

```objc
// NetworkService.m
@implementation NetworkService

- (void)fetchDataFromURL:(NSString *)urlString {
    NSURL *url = [NSURL URLWithString:urlString];
    
    NSURLSession *session = [NSURLSession sharedSession];
    NSURLSessionDataTask *task = [session dataTaskWithURL:url
                                        completionHandler:^(NSData *data, 
                                                          NSURLResponse *response, 
                                                          NSError *error) {
        dispatch_async(dispatch_get_main_queue(), ^{
            // ตรวจสอบ weak delegate ก่อนเรียกใช้
            // ถ้า delegate ถูก deallocate, _delegate จะเป็น nil
            if (error) {
                [self->_delegate networkService:self didFailWithError:error];
            } else {
                [self->_delegate networkService:self didReceiveData:data];
            }
        });
    }];
    [task resume];
}

@end
```

### Zeroing Weak References

หนึ่งในฟีเจอร์สำคัญของ ARC weak references คือ **zeroing**: เมื่อ object ที่ถูกชี้ถูก deallocate, weak reference จะถูก set เป็น nil อัตโนมัติ:

```objc
// ในระบบ manual (MRC หรือ __unsafe_unretained)
// ต้องทำเอง:
- (void)dealloc {
    // แจ้ง observers ว่ากำลัง dealloc
    [[NSNotificationCenter defaultCenter] removeObserver:self];
}

// ใน ARC ด้วย __weak
// ไม่ต้องทำอะไร - weak ref จะเป็น nil เองเมื่อ dealloc
```

---

## 20.5 Unsafe Unretained (__unsafe_unretained)

### __unsafe_unretained คืออะไร

`__unsafe_unretained` สร้าง reference ที่ไม่เพิ่ม retain count เหมือน `__weak` แต่ **ไม่ zeroing** เมื่อ object ถูก deallocate:

```objc
Person *person = [[Person alloc] init];
person.name = @"สมชาย";

__unsafe_unretained Person *unsafePerson = person;

person = nil;  // dealloc person

// ❌ DANGEROUS! unsafePerson ยังชี้ไป memory เดิมที่ถูก free แล้ว
// NSLog(@"%@", unsafePerson.name);  // CRASH หรือ garbage
```

### เมื่อไหรควรใช้ __unsafe_unretained

1. **เมื่อต้องทำงานกับ iOS 4 หรือ macOS 10.6** ที่ยังไม่รองรับ __weak
2. **Performance-critical code** ที่ไม่ต้องการ overhead ของ zeroing
3. **Bridging กับ C/C++ code**

```objc
// ใช้กับ Core Foundation หรือ legacy code
@property (nonatomic, unsafe_unretained) id<OldDelegate> legacyDelegate;

// หรือสำหรับ circular weak references ที่ต้องการ performance
__unsafe_unretained MyClass *unsafeRef = self;
// ใช้ระวัง: ต้องแน่ใจว่า self ยังมีอยู่เมื่อใช้ unsafeRef
```

### __weak vs __unsafe_unretained

| Feature | __weak | __unsafe_unretained |
|---------|--------|---------------------|
| Retain count | ไม่เพิ่ม | ไม่เพิ่ม |
| Zeroing | ✅ Auto nil | ❌ Dangling pointer |
| Safety | ✅ Safe | ❌ Unsafe |
| Performance | เล็กน้อยช้ากว่า | เร็วกว่า |
| iOS Support | iOS 5+ | iOS 4+ |

---

## 20.6 Autoreleasing (__autoreleasing)

### __autoreleasing สำหรับ Error Parameters

`__autoreleasing` ใช้บ่อยที่สุดกับ error parameters ใน method ที่รับ `NSError **`:

```objc
// Method ที่รับ NSError**
- (BOOL)loadFileAtPath:(NSString *)path 
                 error:(NSError **)error {
    NSData *data = [NSData dataWithContentsOfFile:path options:0 error:error];
    // error parameter เป็น NSError *__autoreleasing *
    // ถ้า error เกิดขึ้น *error จะถูก set โดย NSData
    return data != nil;
}

// การเรียกใช้
NSError *error = nil;  // __strong โดย default
if (![self loadFileAtPath:@"/path/to/file" error:&error]) {
    NSLog(@"Error: %@", error.localizedDescription);
}
```

```objc
// ARC แปลง error parameter ให้อัตโนมัติ:
// NSError **error  →  NSError *__autoreleasing *error

// เมื่อเราส่ง &error:
// ARC สร้าง temp variable ชั่วคราว:
// NSError *__autoreleasing tmp = error;
// [method error:&tmp];
// error = tmp;
```

### __autoreleasing ใน Bridging

```objc
// บางครั้งต้องระบุ __autoreleasing อย่างชัดเจน
- (void)processWithCompletion:(NSError *__autoreleasing *)outError {
    if (someCondition) {
        if (outError) {
            *outError = [NSError errorWithDomain:@"MyDomain" 
                                           code:100 
                                       userInfo:nil];
        }
    }
}
```

---

## 20.7 Retain Cycles และ Weak References สำหรับ Delegates

### Retain Cycle คืออะไร

```
Strong Reference Cycle (Retain Cycle):
┌──────────┐  strong  ┌──────────┐
│   View   │──────────│Controller│
│Controller│          │          │
└──────────┘          └──────────┘
      ↑    strong         │
      └───────────────────┘
      
ทั้งคู่ retain กัน = ไม่มีใครถูก deallocate
```

### ตัวอย่าง Retain Cycle ที่พบบ่อย

```objc
// ❌ Retain cycle กับ delegate
@interface NetworkManager : NSObject
@property (nonatomic, strong) id delegate;  // ← STRONG = cycle!
@end

@interface ViewController : UIViewController
@property (nonatomic, strong) NetworkManager *manager;
@end

@implementation ViewController
- (void)viewDidLoad {
    [super viewDidLoad];
    _manager = [[NetworkManager alloc] init];
    _manager.delegate = self;  // VC → manager (strong), manager → VC (strong) = CYCLE!
}
@end
```

```objc
// ✅ ถูกต้อง: Weak delegate
@interface NetworkManager : NSObject
@property (nonatomic, weak) id delegate;  // ← WEAK = no cycle!
@end
```

### Parent-Child Retain Cycle

```objc
// ❌ Retain cycle ระหว่าง parent และ child
@interface Node : NSObject
@property (nonatomic, strong) Node *parent;  // ← STRONG = cycle!
@property (nonatomic, strong) NSArray *children;
@end

// ✅ ถูกต้อง
@interface Node : NSObject
@property (nonatomic, weak) Node *parent;    // ← WEAK = no cycle!
@property (nonatomic, strong) NSArray *children;
@end
```

### NSTimer และ Retain Cycle

```objc
// ❌ Timer retain cycle
@interface ViewController : UIViewController
@property (nonatomic, strong) NSTimer *timer;
@end

@implementation ViewController

- (void)startTimer {
    // NSTimer retain target (self) ด้วย strong reference
    _timer = [NSTimer scheduledTimerWithTimeInterval:1.0
                                              target:self  // ← strong!
                                            selector:@selector(timerFired)
                                            userInfo:nil
                                             repeats:YES];
}

// VC → timer (strong), timer → VC target (strong) = CYCLE!
// เมื่อ VC ถูก dismiss: VC ไม่ถูก dealloc เพราะ timer ยัง hold strong ref

@end
```

```objc
// ✅ ถูกต้อง: ใช้ block-based timer (iOS 10+)
- (void)startTimer {
    __weak typeof(self) weakSelf = self;
    _timer = [NSTimer scheduledTimerWithTimeInterval:1.0
                                             repeats:YES
                                               block:^(NSTimer *timer) {
        __strong typeof(weakSelf) strongSelf = weakSelf;
        if (!strongSelf) {
            [timer invalidate];
            return;
        }
        [strongSelf timerFired];
    }];
}

- (void)dealloc {
    [_timer invalidate];  // สำคัญ: invalidate timer เสมอ
}
```

---

## 20.8 Blocks และ Retain Cycles

### Block Captures Strong References

Block ใน Objective-C capture ตัวแปรที่ใช้ภายใน block โดย default เป็น strong reference:

```objc
// ❌ Retain cycle ใน block
@interface ViewController : UIViewController
@property (nonatomic, copy) void (^completionBlock)(void);
@end

@implementation ViewController

- (void)setupBlock {
    // self ถูก capture ด้วย strong reference
    _completionBlock = ^{
        NSLog(@"Title: %@", self.title);  // block retain self!
    };
    // VC → block (strong), block → VC (strong) = CYCLE!
}

@end
```

### __weak self Pattern

```objc
// ✅ ถูกต้อง: __weak self pattern
- (void)setupBlock {
    __weak typeof(self) weakSelf = self;
    
    _completionBlock = ^{
        NSLog(@"Title: %@", weakSelf.title);  // weak reference ไม่เพิ่ม count
        // ระวัง: weakSelf อาจเป็น nil ถ้า self ถูก deallocate
    };
}
```

### __weak-__strong Dance Pattern

Pattern ที่นิยมคือ weak-strong dance เพื่อให้แน่ใจว่า self ยังมีอยู่ตลอด execution ของ block:

```objc
- (void)loadData {
    __weak typeof(self) weakSelf = self;
    
    [self.networkManager fetchDataWithCompletion:^(NSData *data, NSError *error) {
        // Step 1: Convert weak → strong ชั่วคราว
        __strong typeof(weakSelf) strongSelf = weakSelf;
        
        // Step 2: ตรวจสอบว่ายังมีอยู่
        if (!strongSelf) {
            // self ถูก deallocate ระหว่าง async operation
            return;
        }
        
        // Step 3: ใช้ strongSelf อย่างปลอดภัยใน block
        if (error) {
            [strongSelf handleError:error];
        } else {
            strongSelf.data = data;
            [strongSelf updateUI];
        }
        // strongSelf ถูก release เมื่อออกจาก block scope
    }];
}
```

### ตัวอย่างโลกจริง: Animation Completion

```objc
// ❌ Retain cycle
[UIView animateWithDuration:0.3 animations:^{
    self.view.alpha = 0;
} completion:^(BOOL finished) {
    self.view.removeFromSuperview;  // self ถูก retain ใน block
}];

// ✅ ถูกต้อง
__weak typeof(self) weakSelf = self;
[UIView animateWithDuration:0.3 animations:^{
    weakSelf.view.alpha = 0;
} completion:^(BOOL finished) {
    __strong typeof(weakSelf) strongSelf = weakSelf;
    [strongSelf.view removeFromSuperview];
}];
```

### Block ใน Collection

```objc
// ❌ ปัญหา: block ใน array retain self
@interface TaskManager : NSObject
@property (nonatomic, strong) NSMutableArray *tasks;
@end

@implementation TaskManager

- (void)addTask:(void(^)(void))task {
    [_tasks addObject:task];  // task block อาจ retain self
}

- (void)setupTasks {
    [self addTask:^{
        [self doSomething];  // self ถูก retain ใน block
    }];
    // TaskManager → tasks (strong) → block (strong) → TaskManager = CYCLE!
}
```

```objc
// ✅ ถูกต้อง
- (void)setupTasks {
    __weak typeof(self) weakSelf = self;
    [self addTask:^{
        __strong typeof(weakSelf) strongSelf = weakSelf;
        [strongSelf doSomething];
    }];
}
```

---

## 20.9 Migrating จาก MRC ไป ARC

### ขั้นตอนการ Migrate

**วิธีที่ 1: ใช้ Xcode Migration Tool**

1. Edit → Refactor → Convert to Objective-C ARC
2. Xcode จะวิเคราะห์โค้ดและแนะนำการเปลี่ยนแปลง
3. Review การเปลี่ยนแปลงและยืนยัน

**วิธีที่ 2: Manual Migration**

### การเปลี่ยนแปลงที่จำเป็น

```objc
// MRC code:
@property (nonatomic, retain) NSString *name;

- (void)setName:(NSString *)name {
    if (_name != name) {
        [_name release];
        _name = [name retain];
    }
}

- (void)dealloc {
    [_name release];
    [super dealloc];
}
```

```objc
// ARC code: (หลัง migrate)
@property (nonatomic, strong) NSString *name;

// setter ถูกลบออก - ARC จัดการให้
// dealloc ถูกลบออก (หรือใส่แค่ cleanup code)
```

### Checklist สำหรับ Migration

```objc
// 1. ลบ retain/release/autorelease calls
[obj retain];     // ← ลบ
[obj release];    // ← ลบ
[obj autorelease]; // ← ลบ

// 2. เปลี่ยน property attributes
// retain → strong
// assign (สำหรับ objects) → weak

// 3. ลบ dealloc ที่มีแค่ release
// (หรือเก็บไว้ถ้ามี cleanup อื่น เช่น removeObserver)

// 4. เปลี่ยน @property ใน protocol
// 5. แก้ไข Toll-Free Bridging ด้วย __bridge

// 6. File ที่ไม่ต้องการ ARC: เพิ่ม -fno-objc-arc flag
```

### Mixed Mode: ไฟล์บางไฟล์ใช้ MRC

```
// ใน Xcode:
// Build Phases → Compile Sources
// เพิ่ม -fno-objc-arc flag ให้ไฟล์ที่ต้องการ MRC

MyLegacyFile.m   -fno-objc-arc
```

```objc
// หรือ per-file ใน source:
// ไม่ได้รองรับ per-file flag ใน source code
// ต้องทำผ่าน Xcode build settings
```

---

## 20.10 ARC กับ Core Foundation (CF Bridging)

### ปัญหา: ARC ไม่จัดการ CF Objects

ARC จัดการแค่ Objective-C objects ส่วน Core Foundation objects (CFStringRef, CFArrayRef, etc.) ต้องจัดการ memory เอง:

```objc
// CF Object: ต้อง release เอง
CFStringRef cfStr = CFStringCreateWithCString(NULL, "Hello", kCFStringEncodingUTF8);
NSLog(@"%@", (__bridge NSString *)cfStr);
CFRelease(cfStr);  // ← ต้องเรียก CFRelease เอง
```

### __bridge: ไม่โอน Ownership

```objc
// __bridge: แปลง type โดยไม่โอน ownership
CFStringRef cfStr = CFStringCreateWithCString(NULL, "Hello", kCFStringEncodingUTF8);
NSString *nsStr = (__bridge NSString *)cfStr;
// nsStr ไม่ own object - cfStr ยังต้อง CFRelease
NSLog(@"%@", nsStr);
CFRelease(cfStr);  // ← เรายังต้องจัดการ CF side
```

### CFBridgingRetain: โอน Ownership จาก ObjC ไป CF

```objc
// ส่ง NSString ไปยัง CF API และโอน ownership
NSString *nsStr = @"Hello, World!";
CFStringRef cfStr = CFBridgingRetain(nsStr);
// ตอนนี้ CF side เป็นเจ้าของ object
// เมื่อเสร็จต้อง CFRelease
// nsStr สามารถเป็น nil ได้

// ทำงานกับ CF string...

CFRelease(cfStr);  // ← โอน ownership ไป CF แล้ว ต้อง CFRelease
```

### CFBridgingRelease: โอน Ownership จาก CF ไป ARC

```objc
// รับ CF object มาและโอน ownership ให้ ARC
CFStringRef cfStr = CFStringCreateWithCString(NULL, "Hello", kCFStringEncodingUTF8);
// cfStr มี +1 retain count จาก Create

NSString *nsStr = CFBridgingRelease(cfStr);
// ตอนนี้ ARC เป็นเจ้าของ object
// ไม่ต้อง CFRelease แล้ว
// cfStr ไม่ควรใช้ต่อ

NSLog(@"%@", nsStr);
// ARC จัดการ release ให้เมื่อ nsStr ออกจาก scope
```

### __bridge_transfer (เหมือน CFBridgingRelease)

```objc
// CFStringCreateWithCString มี +1 count
CFStringRef cfStr = CFStringCreateWithCString(NULL, "Hello", kCFStringEncodingUTF8);

// __bridge_transfer: โอน ownership ไป ARC
NSString *nsStr = (__bridge_transfer NSString *)cfStr;
// ARC รับ ownership, ไม่ต้อง CFRelease

NSLog(@"%@", nsStr);
```

### __bridge_retained (เหมือน CFBridgingRetain)

```objc
NSString *nsStr = @"Hello";

// __bridge_retained: ARC โอน ownership ไป CF
CFStringRef cfStr = (__bridge_retained CFStringRef)nsStr;
// CF side เป็นเจ้าของ, ต้อง CFRelease เอง

// ทำงานกับ CF string...
CFRelease(cfStr);  // ← จำเป็น!
```

### ตัวอย่างโลกจริง: Keychain Access

```objc
// การ query Keychain ใช้ CF types
- (NSString *)getPasswordForAccount:(NSString *)account {
    NSDictionary *query = @{
        (__bridge NSString *)kSecClass: (__bridge NSString *)kSecClassGenericPassword,
        (__bridge NSString *)kSecAttrAccount: account,
        (__bridge NSString *)kSecReturnData: @YES,
        (__bridge NSString *)kSecMatchLimit: (__bridge NSString *)kSecMatchLimitOne
    };
    
    CFTypeRef result = NULL;
    OSStatus status = SecItemCopyMatching((__bridge CFDictionaryRef)query, &result);
    
    if (status == errSecSuccess) {
        // result เป็น CF object ที่ต้อง release
        NSData *data = (__bridge_transfer NSData *)result;
        // __bridge_transfer: ARC รับ ownership, ไม่ต้อง CFRelease
        return [[NSString alloc] initWithData:data encoding:NSUTF8StringEncoding];
    }
    
    return nil;
}
```

---

## 20.11 @autoreleasepool ใน Loops

### ทำไมต้องใช้ @autoreleasepool ใน Loops

แม้ใน ARC objects ที่ได้จาก convenience constructors ยังคงเป็น autorelease จนกระทั่ง pool drain:

```objc
// ❌ ปัญหา: memory สะสมจำนวนมาก
- (void)processLargeDataset {
    for (int i = 0; i < 1000000; i++) {
        // autoreleased objects สะสมจนกว่า pool จะ drain (run loop)
        NSString *str = [NSString stringWithFormat:@"Processing item %d", i];
        NSData *data = [str dataUsingEncoding:NSUTF8StringEncoding];
        // ทำอะไรกับ data...
    }
    // memory usage สูงมากระหว่าง loop
}
```

```objc
// ✅ ถูกต้อง: ใช้ @autoreleasepool ใน loop
- (void)processLargeDataset {
    for (int i = 0; i < 1000000; i++) {
        @autoreleasepool {
            NSString *str = [NSString stringWithFormat:@"Processing item %d", i];
            NSData *data = [str dataUsingEncoding:NSUTF8StringEncoding];
            // ทำอะไรกับ data...
        }
        // str และ data ถูก release ทุก iteration
    }
    // memory usage ต่ำมากตลอด loop
}
```

### Background Thread ด้วย @autoreleasepool

```objc
// Background thread ต้องสร้าง autorelease pool เอง
- (void)performBackgroundWork {
    dispatch_async(dispatch_get_global_queue(DISPATCH_QUEUE_PRIORITY_DEFAULT, 0), ^{
        @autoreleasepool {
            // งานที่ทำใน background
            for (int i = 0; i < 100; i++) {
                @autoreleasepool {
                    // ทำงาน...
                    NSString *result = [self processItem:i];
                    NSLog(@"Result: %@", result);
                }
            }
        }
    });
}
```

### NSOperation กับ @autoreleasepool

```objc
// Custom NSOperation ต้องสร้าง pool เองถ้า non-main-thread
@interface DataProcessingOperation : NSOperation
@end

@implementation DataProcessingOperation

- (void)main {
    if (self.isCancelled) return;
    
    @autoreleasepool {
        // process data...
        for (NSInteger i = 0; i < 10000; i++) {
            if (self.isCancelled) break;
            
            @autoreleasepool {
                // ทำงาน intensive ที่สร้าง autoreleased objects เยอะ
                [self processChunk:i];
            }
        }
    }
}

@end
```

---

## 20.12 Zeroing Weak References

### การทำงานของ Zeroing Weak References

เมื่อ object ถูก deallocate ใน ARC, runtime จะ:
1. Set weak references ทั้งหมดที่ชี้มายัง object นั้นให้เป็น nil
2. ทำก่อน dealloc method ทำงาน

```objc
@interface Animal : NSObject
@property (nonatomic, copy) NSString *name;
@end

// ตัวอย่าง zeroing
Animal *dog = [[Animal alloc] init];
dog.name = @"Rex";

__weak Animal *weakDog = dog;
NSLog(@"Before: %@", weakDog.name);  // Rex

dog = nil;  // retain count = 0, dealloc ทำงาน
            // ก่อน dealloc: runtime set weakDog = nil

NSLog(@"After: %@", weakDog);  // (null) - zeroed!
NSLog(@"Name: %@", weakDog.name);  // (null) - safe! ไม่ crash
```

### Implementing Custom Zeroing-like Behavior

```objc
// สำหรับ case ที่ต้องการ notification เมื่อ weak ref เป็น nil
@interface WeakReference : NSObject

@property (nonatomic, weak) id object;
@property (nonatomic, copy) void (^onDealloc)(void);

+ (instancetype)weakReferenceWithObject:(id)object 
                               onDealloc:(void(^)(void))block;

@end

@implementation WeakReference

+ (instancetype)weakReferenceWithObject:(id)object onDealloc:(void(^)(void))block {
    WeakReference *ref = [[WeakReference alloc] init];
    ref.object = object;
    ref.onDealloc = block;
    
    // ใช้ associated objects เพื่อ observe deallocation
    // (เทคนิคขั้นสูง)
    return ref;
}

@end
```

### Weak Reference Arrays และ Collections

```objc
// NSArray เก็บ strong references เสมอ
// ถ้าต้องการ weak references ต้องสร้าง wrapper

@interface WeakObjectWrapper : NSObject
@property (nonatomic, weak) id object;
+ (instancetype)wrapperWithObject:(id)object;
@end

@implementation WeakObjectWrapper
+ (instancetype)wrapperWithObject:(id)object {
    WeakObjectWrapper *w = [[WeakObjectWrapper alloc] init];
    w.object = object;
    return w;
}
@end

// การใช้งาน
NSMutableArray *weakArray = [NSMutableArray array];
for (NSObject *obj in objects) {
    [weakArray addObject:[WeakObjectWrapper wrapperWithObject:obj]];
}

// ดึงค่า
for (WeakObjectWrapper *wrapper in weakArray) {
    if (wrapper.object) {  // ตรวจสอบว่ายังมีอยู่
        // ใช้ wrapper.object
    }
}
```

### NSHashTable กับ NSMapTable สำหรับ Weak Collections

```objc
// NSHashTable รองรับ weak references
NSHashTable *weakSet = [NSHashTable weakObjectsHashTable];

MyObject *obj1 = [[MyObject alloc] init];
MyObject *obj2 = [[MyObject alloc] init];

[weakSet addObject:obj1];
[weakSet addObject:obj2];

NSLog(@"Count: %lu", (unsigned long)weakSet.count);  // 2

obj1 = nil;  // dealloc obj1
// NSHashTable จะ clean up entry ของ obj1 อัตโนมัติ (lazily)

NSLog(@"Count after: %lu", (unsigned long)weakSet.count);  // อาจยังเป็น 2 จนกว่าจะ access
NSLog(@"Count (cleaned): %lu", (unsigned long)[[weakSet allObjects] count]);  // 1
```

```objc
// NSMapTable รองรับ weak keys หรือ weak values
NSMapTable *weakValueMap = [NSMapTable strongToWeakObjectsMapTable];
// key เป็น strong, value เป็น weak

NSMapTable *weakKeyMap = [NSMapTable weakToStrongObjectsMapTable];
// key เป็น weak, value เป็น strong
```

---

## 20.13 Instruments Leaks Tool

### เปิดใช้งาน Instruments

Instruments เป็น tool ใน Xcode สำหรับ profiling app รวมถึงการตรวจหา memory leaks:

**วิธีเปิด:**
1. Xcode → Product → Profile (⌘I)
2. เลือก "Leaks" template
3. กด Record

### ตีความผลลัพธ์ Leaks

```
Leaks instrument แสดง:
- Memory usage graph
- Leaked objects list
- Call stack ของ allocation
- Retained but not released objects
```

### Instruments Allocations

```
Allocations instrument:
- All allocations (live + dead)
- Generation tracking
- Call trees สำหรับ allocation
- MemGraph สำหรับ visualization
```

### การใช้ Debug Memory Graph

```
Xcode Debug Memory Graph (Shift+Ctrl+Command+M):
- แสดง object graph ใน memory
- ช่วย identify retain cycles
- แสดง reference paths ไปยัง objects
```

### ตัวอย่างการ Debug ด้วย Instruments

```objc
// ก่อน: มี leak
@interface DataCache : NSObject

@property (nonatomic, strong) NSMutableDictionary *cache;
@property (nonatomic, strong) DataCache *parentCache;  // ← strong ทำให้ cycle ถ้า parent เก็บ child ด้วย

@end

// หลัง Debug พบ cycle ด้วย Instruments:
@interface DataCache : NSObject

@property (nonatomic, strong) NSMutableDictionary *cache;
@property (nonatomic, weak) DataCache *parentCache;  // ← เปลี่ยนเป็น weak

@end
```

### การวิเคราะห์ Call Stack ใน Leaks

```
เมื่อพบ leak:
1. คลิก leaked object ใน list
2. ดู "Responsible Frame" - ตำแหน่งที่ allocate
3. ดู call stack
4. หา ตำแหน่ง ที่ควร release แต่ไม่ release
```

---

## 20.14 ตัวอย่าง Complete ด้วย ARC

### Full Example: Message Queue System

```objc
// Message.h
@interface Message : NSObject

@property (nonatomic, copy) NSString *content;
@property (nonatomic, copy) NSString *sender;
@property (nonatomic, strong) NSDate *timestamp;
@property (nonatomic, assign) BOOL isRead;

+ (instancetype)messageWithContent:(NSString *)content from:(NSString *)sender;

@end
```

```objc
// Message.m
@implementation Message

+ (instancetype)messageWithContent:(NSString *)content from:(NSString *)sender {
    Message *msg = [[Message alloc] init];
    msg.content = content;
    msg.sender = sender;
    msg.timestamp = [NSDate date];
    msg.isRead = NO;
    return msg;  // ARC จัดการ retain/release
}

- (NSString *)description {
    NSDateFormatter *formatter = [[NSDateFormatter alloc] init];
    formatter.dateStyle = NSDateFormatterShortStyle;
    formatter.timeStyle = NSDateFormatterShortStyle;
    return [NSString stringWithFormat:@"[%@] %@: %@", 
            [formatter stringFromDate:_timestamp], _sender, _content];
}

@end
```

```objc
// MessageQueue.h
@protocol MessageQueueDelegate <NSObject>
@optional
- (void)messageQueue:(id)queue didReceiveMessage:(Message *)message;
- (void)messageQueue:(id)queue didReadMessage:(Message *)message;
- (void)messageQueueDidEmpty:(id)queue;
@end

@interface MessageQueue : NSObject

@property (nonatomic, weak) id<MessageQueueDelegate> delegate;  // ← weak!
@property (nonatomic, readonly) NSUInteger count;
@property (nonatomic, readonly) NSUInteger unreadCount;

- (void)enqueue:(Message *)message;
- (Message *)dequeue;
- (void)markAllAsRead;
- (NSArray<Message *> *)unreadMessages;

@end
```

```objc
// MessageQueue.m
@interface MessageQueue ()
@property (nonatomic, strong) NSMutableArray<Message *> *messages;
@property (nonatomic, strong) dispatch_queue_t accessQueue;
@end

@implementation MessageQueue

- (instancetype)init {
    self = [super init];
    if (self) {
        _messages = [[NSMutableArray alloc] init];
        _accessQueue = dispatch_queue_create("com.queue.messages", 
                                            DISPATCH_QUEUE_CONCURRENT);
    }
    return self;
}

- (NSUInteger)count {
    __block NSUInteger count;
    dispatch_sync(_accessQueue, ^{
        count = self->_messages.count;
    });
    return count;
}

- (NSUInteger)unreadCount {
    __block NSUInteger count = 0;
    dispatch_sync(_accessQueue, ^{
        for (Message *msg in self->_messages) {
            if (!msg.isRead) count++;
        }
    });
    return count;
}

- (void)enqueue:(Message *)message {
    dispatch_barrier_async(_accessQueue, ^{
        [self->_messages addObject:message];
    });
    
    // Notify delegate on main thread
    dispatch_async(dispatch_get_main_queue(), ^{
        if ([self->_delegate respondsToSelector:@selector(messageQueue:didReceiveMessage:)]) {
            [self->_delegate messageQueue:self didReceiveMessage:message];
        }
    });
}

- (Message *)dequeue {
    __block Message *message = nil;
    dispatch_barrier_sync(_accessQueue, ^{
        if (self->_messages.count > 0) {
            message = self->_messages.firstObject;
            [self->_messages removeObjectAtIndex:0];
        }
    });
    
    if (_messages.count == 0) {
        dispatch_async(dispatch_get_main_queue(), ^{
            if ([self->_delegate respondsToSelector:@selector(messageQueueDidEmpty:)]) {
                [self->_delegate messageQueueDidEmpty:self];
            }
        });
    }
    
    return message;
}

- (void)markAllAsRead {
    dispatch_barrier_async(_accessQueue, ^{
        for (Message *msg in self->_messages) {
            msg.isRead = YES;
        }
    });
}

- (NSArray<Message *> *)unreadMessages {
    __block NSArray *unread;
    dispatch_sync(_accessQueue, ^{
        NSPredicate *pred = [NSPredicate predicateWithFormat:@"isRead == NO"];
        unread = [self->_messages filteredArrayUsingPredicate:pred];
    });
    return unread;
}

@end
```

```objc
// ChatRoom.h
@interface ChatRoom : NSObject <MessageQueueDelegate>

@property (nonatomic, copy) NSString *roomName;
@property (nonatomic, strong) MessageQueue *messageQueue;
@property (nonatomic, copy) void (^onNewMessage)(Message *message);  // block property

+ (instancetype)chatRoomWithName:(NSString *)name;
- (void)sendMessage:(NSString *)text from:(NSString *)sender;

@end
```

```objc
// ChatRoom.m
@implementation ChatRoom

+ (instancetype)chatRoomWithName:(NSString *)name {
    ChatRoom *room = [[ChatRoom alloc] init];
    room.roomName = name;
    room.messageQueue = [[MessageQueue alloc] init];
    room.messageQueue.delegate = room;
    return room;
}

- (void)sendMessage:(NSString *)text from:(NSString *)sender {
    Message *message = [Message messageWithContent:text from:sender];
    [_messageQueue enqueue:message];
}

// Implement delegate
- (void)messageQueue:(id)queue didReceiveMessage:(Message *)message {
    NSLog(@"[%@] New message: %@", _roomName, message.description);
    
    // Call block if set (weak-strong dance ไม่จำเป็นที่นี่เพราะ block เป็น property)
    if (_onNewMessage) {
        _onNewMessage(message);
    }
}

- (void)messageQueueDidEmpty:(id)queue {
    NSLog(@"[%@] No more messages", _roomName);
}

@end
```

```objc
// การใช้งาน
int main(int argc, char *argv[]) {
    @autoreleasepool {
        ChatRoom *room = [ChatRoom chatRoomWithName:@"General"];
        
        // ตั้ง block handler (ระวัง retain cycle ถ้า block capture room ด้วย strong)
        __weak ChatRoom *weakRoom = room;
        room.onNewMessage = ^(Message *message) {
            __strong ChatRoom *strongRoom = weakRoom;
            NSLog(@"Handler for room '%@': %@", strongRoom.roomName, message.content);
        };
        
        // ส่งข้อความ
        [room sendMessage:@"สวัสดีทุกคน!" from:@"Admin"];
        [room sendMessage:@"มีใครอยู่บ้าง?" from:@"User1"];
        [room sendMessage:@"ผมอยู่ครับ" from:@"User2"];
        
        NSLog(@"Unread: %lu", (unsigned long)room.messageQueue.unreadCount);
        
        [room.messageQueue markAllAsRead];
        NSLog(@"After mark read: %lu", (unsigned long)room.messageQueue.unreadCount);
        
        // room, messageQueue จะถูก deallocate อัตโนมัติเมื่อออกจาก scope
    }
    
    return 0;
}
```

---

## แบบฝึกหัด (Practice Exercises)

### ข้อ 1: Identifyและแก้ไข Retain Cycles

วิเคราะห์และแก้ไข retain cycles ในโค้ดต่อไปนี้:

```objc
// ปัญหาข้อ 1:
@interface ViewController : UIViewController
@property (nonatomic, strong) NSTimer *updateTimer;
@property (nonatomic, copy) NSString *statusText;
@end

@implementation ViewController
- (void)viewDidLoad {
    [super viewDidLoad];
    self.updateTimer = [NSTimer scheduledTimerWithTimeInterval:1.0
                                                        target:self
                                                      selector:@selector(updateStatus)
                                                      userInfo:nil
                                                       repeats:YES];
}
- (void)updateStatus {
    self.statusText = [NSString stringWithFormat:@"Updated: %@", [NSDate date]];
}
@end
```

```objc
// เฉลย:
@implementation ViewController

- (void)viewDidLoad {
    [super viewDidLoad];
    
    __weak typeof(self) weakSelf = self;
    // ใช้ block-based timer ที่ไม่ retain target
    self.updateTimer = [NSTimer scheduledTimerWithTimeInterval:1.0
                                                       repeats:YES
                                                         block:^(NSTimer *timer) {
        __strong typeof(weakSelf) strongSelf = weakSelf;
        if (!strongSelf) {
            [timer invalidate];
            return;
        }
        [strongSelf updateStatus];
    }];
}

- (void)updateStatus {
    self.statusText = [NSString stringWithFormat:@"Updated: %@", [NSDate date]];
}

- (void)dealloc {
    [self.updateTimer invalidate];
    self.updateTimer = nil;
}

@end
```

### ข้อ 2: Core Foundation Bridging

```objc
// เฉลย: แปลงระหว่าง CF และ NS types ด้วย proper bridging

// 1. สร้าง CF String และแปลงไป NS
CFStringRef cfName = CFStringCreateWithCString(NULL, "สวัสดี", kCFStringEncodingUTF8);
NSString *nsName = CFBridgingRelease(cfName);  // Transfer ownership to ARC
NSLog(@"%@", nsName);
// ไม่ต้อง CFRelease(cfName)

// 2. ส่ง NS String ไปยัง CF API
NSString *greeting = @"Hello";
CFStringRef cfGreeting = CFBridgingRetain(greeting);  // Transfer ownership to CF
// ทำงานกับ CF string...
CFRelease(cfGreeting);  // ต้อง CFRelease

// 3. Toll-free bridging (NS และ CF คือ type เดียวกัน)
NSArray *array = @[@"a", @"b", @"c"];
CFArrayRef cfArray = (__bridge CFArrayRef)array;  // No transfer, just cast
CFIndex count = CFArrayGetCount(cfArray);
NSLog(@"Count: %ld", count);
// ไม่ต้อง CFRelease - ARC ยัง own array
```

### ข้อ 3: Weak-Strong Dance ใน Nested Blocks

```objc
// เฉลย: Proper weak-strong dance ใน async code
@interface DataFetcher : NSObject
@property (nonatomic, strong) NSURLSession *session;
@property (nonatomic, copy) void(^completionHandler)(NSArray *results);
@end

@implementation DataFetcher

- (void)fetchWithCompletion:(void(^)(NSArray *))completion {
    self.completionHandler = completion;
    
    __weak typeof(self) weakSelf = self;
    
    NSURL *url = [NSURL URLWithString:@"https://api.example.com/data"];
    NSURLSessionDataTask *task = [self.session dataTaskWithURL:url
                                            completionHandler:^(NSData *data, 
                                                               NSURLResponse *response, 
                                                               NSError *error) {
        // Inner block (network callback)
        __strong typeof(weakSelf) strongSelf = weakSelf;
        if (!strongSelf) return;
        
        // Parse data
        NSArray *results = [NSJSONSerialization JSONObjectWithData:data options:0 error:nil];
        
        dispatch_async(dispatch_get_main_queue(), ^{
            // Another nested block (main queue callback)
            // strongSelf ถูก retain ใน outer block scope
            // แต่ต้องตรวจสอบอีกครั้งถ้า async
            if (strongSelf.completionHandler) {
                strongSelf.completionHandler(results);
            }
        });
    }];
    [task resume];
}

@end
```

### ข้อ 4: Observer Pattern ด้วย ARC

```objc
// เฉลย: Thread-safe observer pattern
@interface EventSystem : NSObject

+ (instancetype)shared;

- (NSString *)subscribe:(NSString *)eventName 
                handler:(void(^)(id eventData))handler;
- (void)unsubscribe:(NSString *)token;
- (void)emit:(NSString *)eventName data:(id)data;

@end

@implementation EventSystem {
    NSMutableDictionary<NSString *, NSMutableDictionary<NSString *, id> *> *_subscriptions;
    dispatch_queue_t _queue;
    NSInteger _tokenCounter;
}

+ (instancetype)shared {
    static EventSystem *instance;
    static dispatch_once_t onceToken;
    dispatch_once(&onceToken, ^{
        instance = [[EventSystem alloc] init];
    });
    return instance;
}

- (instancetype)init {
    self = [super init];
    if (self) {
        _subscriptions = [NSMutableDictionary dictionary];
        _queue = dispatch_queue_create("com.events.queue", DISPATCH_QUEUE_CONCURRENT);
        _tokenCounter = 0;
    }
    return self;
}

- (NSString *)subscribe:(NSString *)eventName handler:(void(^)(id))handler {
    NSString *token = [NSString stringWithFormat:@"token_%ld", (long)++_tokenCounter];
    
    dispatch_barrier_async(_queue, ^{
        if (!self->_subscriptions[eventName]) {
            self->_subscriptions[eventName] = [NSMutableDictionary dictionary];
        }
        self->_subscriptions[eventName][token] = [handler copy];
    });
    
    return token;
}

- (void)unsubscribe:(NSString *)token {
    dispatch_barrier_async(_queue, ^{
        [self->_subscriptions enumerateKeysAndObjectsUsingBlock:^(NSString *event, 
                                                                    NSMutableDictionary *handlers,
                                                                    BOOL *stop) {
            [handlers removeObjectForKey:token];
        }];
    });
}

- (void)emit:(NSString *)eventName data:(id)data {
    dispatch_sync(_queue, ^{
        NSDictionary *handlers = [self->_subscriptions[eventName] copy];
        dispatch_async(dispatch_get_main_queue(), ^{
            for (void(^handler)(id) in handlers.allValues) {
                handler(data);
            }
        });
    });
}

@end
```

### ข้อ 5 - 10: แบบฝึกหัดเพิ่มเติม

**ข้อ 5**: สร้าง `ImageCache` ด้วย `NSCache` (ที่มี weak semantics สำหรับ values) และ proper memory handling ใน ARC

```objc
// เฉลยบางส่วน
@interface ImageCache : NSObject

+ (instancetype)shared;
- (void)setImage:(UIImage *)image forKey:(NSString *)key;
- (UIImage *)imageForKey:(NSString *)key;

@end

@implementation ImageCache {
    NSCache *_cache;  // NSCache ใช้ strong references แต่ evict เมื่อ memory pressure
}

+ (instancetype)shared {
    static ImageCache *instance;
    static dispatch_once_t token;
    dispatch_once(&token, ^{ instance = [[ImageCache alloc] init]; });
    return instance;
}

- (instancetype)init {
    self = [super init];
    if (self) {
        _cache = [[NSCache alloc] init];
        _cache.countLimit = 100;
        _cache.totalCostLimit = 50 * 1024 * 1024;  // 50MB
        
        [[NSNotificationCenter defaultCenter] 
            addObserver:self
               selector:@selector(memoryWarning)
                   name:UIApplicationDidReceiveMemoryWarningNotification
                 object:nil];
    }
    return self;
}

- (void)memoryWarning {
    [_cache removeAllObjects];
}

- (void)dealloc {
    [[NSNotificationCenter defaultCenter] removeObserver:self];
}

@end
```

**ข้อ 6**: แก้ไข memory issues ในโค้ดต่อไปนี้โดยใช้ ARC concepts:

```objc
// มี problems ดังนี้ - หาและแก้ไข:

// Problem 1: Retain cycle ใน notification
[[NSNotificationCenter defaultCenter] 
    addObserverForName:@"DataReady" 
               object:nil 
                queue:nil 
           usingBlock:^(NSNotification *n) {
    [self handleNotification:n];  // self ถูก capture
}];

// Problem 2: Timer ที่ไม่ invalidate
self.refreshTimer = [NSTimer scheduledTimerWithTimeInterval:30.0
                                                     target:self
                                                   selector:@selector(refresh)
                                                   userInfo:nil
                                                    repeats:YES];

// Problem 3: Block ที่ capture strong ของตัวเอง
self.networkManager.completionBlock = ^{
    [self.tableView reloadData];
    self.loadingView.hidden = YES;
};
```

**ข้อ 7**: สร้าง `Promise`-like class ด้วย ARC ที่รองรับ chaining และ error handling

**ข้อ 8**: สร้าง `AutoreleasingProperty` class ที่แสดงการทำงานของ autorelease ผ่าน Custom getter/setter

**ข้อ 9**: สร้าง Thread-safe `ReactiveProperty` class ที่สามารถ observe การเปลี่ยนแปลงได้ ด้วย weak references ไปยัง observers

**ข้อ 10**: Profile app ที่มี memory leak โดยใช้ Instruments และเขียน report สรุปปัญหาที่พบ

---

## สรุปเปรียบเทียบ MRC vs ARC

| Feature | MRC | ARC |
|---------|-----|-----|
| retain/release | เขียนเอง | Compiler แทรกให้ |
| @autoreleasepool | ต้องใช้เอง | ใช้ในลูป/thread |
| Property attributes | `retain`, `assign` | `strong`, `weak` |
| dealloc | ต้อง release ใน dealloc | แค่ cleanup, ไม่ release |
| Convenience constructor | ต้อง autorelease เอง | อัตโนมัติ |
| retain cycle | จัดการเอง | จัดการเอง (เหมือนกัน!) |
| CF objects | จัดการเอง | จัดการเอง + bridging |
| Debugging | NSZombie, Instruments | Instruments, Debug Memory Graph |

---

## สรุป

ในบทนี้เราได้เรียนรู้ ARC อย่างครบถ้วน:

1. **How ARC Works** - Compiler วิเคราะห์และแทรก retain/release ให้อัตโนมัติ
2. **ARC vs GC** - ความแตกต่างและทำไม ARC ดีกว่า GC
3. **__strong** - Strong reference เป็นเจ้าของ object
4. **__weak** - Weak reference ที่ zeroing เมื่อ object dealloc
5. **__unsafe_unretained** - Weak ที่ไม่ zeroing สำหรับ legacy code
6. **__autoreleasing** - สำหรับ NSError** parameters
7. **Retain Cycles** - เข้าใจและป้องกันด้วย weak references
8. **Block Retain Cycles** - __weak self / __strong self pattern
9. **Migration จาก MRC** - ขั้นตอนการ migrate โค้ด
10. **CF Bridging** - __bridge, CFBridgingRetain, CFBridgingRelease
11. **@autoreleasepool in Loops** - ลด memory usage ในลูป
12. **Zeroing Weak References** - Runtime behavior ของ weak refs
13. **Instruments** - ใช้ Leaks tool และ Debug Memory Graph

การเข้าใจทั้ง MRC และ ARC ช่วยให้เป็น Objective-C developer ที่มีความสามารถสูง สามารถเขียนโค้ดที่มีประสิทธิภาพ ปลอดภัยจาก memory leaks และ debug ปัญหา memory ได้อย่างมืออาชีพ
