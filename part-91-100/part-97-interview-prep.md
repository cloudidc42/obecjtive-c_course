# ตอนที่ 97: การเตรียมตัวสัมภาษณ์งาน Objective-C และ iOS

## บทนำ

การสัมภาษณ์งานตำแหน่ง iOS Developer นั้นต้องการความรู้ที่หลากหลาย ตั้งแต่พื้นฐาน Objective-C ไปจนถึงสถาปัตยกรรมระบบและการแก้ปัญหาเชิงซับซ้อน บทนี้จะรวบรวมคำถามสัมภาษณ์ที่พบบ่อยกว่า 100 ข้อ พร้อมคำตอบโดยละเอียด

---

## ส่วนที่ 1: คำถามพื้นฐาน Objective-C

### Q1: Objective-C แตกต่างจาก C อย่างไร?

**คำตอบ:**
Objective-C เป็น superset ของ C ที่เพิ่ม object-oriented programming โดยยืมไวยากรณ์มาจาก Smalltalk

```objc
// C แบบปกติ
int add(int a, int b) {
    return a + b;
}

// Objective-C - ส่ง message ไปยัง object
@interface Calculator : NSObject
- (int)addNumber:(int)a toNumber:(int)b;
@end

@implementation Calculator
- (int)addNumber:(int)a toNumber:(int)b {
    return a + b;
}
@end

// การใช้งาน
Calculator *calc = [[Calculator alloc] init];
int result = [calc addNumber:5 toNumber:3];
```

ความแตกต่างหลัก:
1. **Message passing** แทน function calls
2. **Dynamic dispatch** - ตัดสินใจ method ที่จะเรียกตอน runtime
3. **id type** - generic object pointer
4. **nil messaging** - ส่ง message ไปยัง nil ไม่ crash

---

### Q2: อธิบาย @interface และ @implementation

**คำตอบ:**
```objc
// .h file - Declaration (interface)
@interface Person : NSObject

// Properties
@property (nonatomic, strong) NSString *name;
@property (nonatomic, assign) NSInteger age;

// Method declarations
- (instancetype)initWithName:(NSString *)name age:(NSInteger)age;
- (NSString *)introduce;
+ (Person *)unknownPerson; // Class method

@end

// .m file - Definition (implementation)
@implementation Person

- (instancetype)initWithName:(NSString *)name age:(NSInteger)age {
    self = [super init];
    if (self) {
        _name = name;
        _age = age;
    }
    return self;
}

- (NSString *)introduce {
    return [NSString stringWithFormat:@"สวัสดี ฉันชื่อ %@ อายุ %ld ปี",
            self.name, (long)self.age];
}

+ (Person *)unknownPerson {
    return [[self alloc] initWithName:@"ไม่ทราบชื่อ" age:0];
}

@end
```

---

### Q3: ความแตกต่างระหว่าง `+` และ `-` method

**คำตอบ:**
- `-` คือ **instance method** - เรียกใช้กับ object instance
- `+` คือ **class method** - เรียกใช้กับ class โดยตรง

```objc
@interface Dog : NSObject
@property (nonatomic, strong) NSString *name;

// Class method - factory method
+ (Dog *)dogWithName:(NSString *)name;

// Instance method - ทำงานกับ instance
- (void)bark;
- (NSString *)description;
@end

@implementation Dog

+ (Dog *)dogWithName:(NSString *)name {
    Dog *dog = [[self alloc] init];
    dog.name = name;
    return dog;
}

- (void)bark {
    NSLog(@"%@ says: Woof!", self.name);
}

@end

// การใช้งาน
Dog *myDog = [Dog dogWithName:@"Buddy"]; // class method
[myDog bark]; // instance method
```

---

### Q4: อธิบาย `self` และ `super`

**คำตอบ:**
```objc
@interface Animal : NSObject
- (NSString *)sound;
- (void)makeSound;
@end

@implementation Animal
- (NSString *)sound {
    return @"...";
}

- (void)makeSound {
    // self - อ้างถึง current object (dynamic dispatch)
    NSLog(@"Animal makes sound: %@", [self sound]);
}
@end

@interface Cat : Animal
@end

@implementation Cat
- (NSString *)sound {
    return @"Meow";
}

- (void)makeSound {
    // super - เรียก parent class method (static dispatch)
    [super makeSound]; // จะเรียก Cat's sound ผ่าน self ใน Animal
    NSLog(@"Cat specifically says: %@", [self sound]);
}
@end
```

ความแตกต่างสำคัญ:
- `self` ใช้ dynamic dispatch (polymorphism)
- `super` เรียก parent implementation แต่ยังใช้ `self` เป็น receiver

---

### Q5: `nil`, `Nil`, `NULL`, `NSNull` แตกต่างกันอย่างไร?

**คำตอบ:**
```objc
// nil - pointer ที่ชี้ไปยัง Objective-C object ที่เป็น null
NSString *str = nil;
if (str == nil) {
    NSLog(@"str is nil");
}

// Nil - pointer ที่เป็น null สำหรับ Class type
Class cls = Nil;

// NULL - null pointer ในภาษา C
void *ptr = NULL;
char *cString = NULL;

// NSNull - object ที่ใช้แทน nil ใน collections
NSArray *array = @[@"one", [NSNull null], @"three"];
// ไม่สามารถใส่ nil ใน NSArray ได้โดยตรง
for (id obj in array) {
    if ([obj isKindOfClass:[NSNull class]]) {
        NSLog(@"Found null placeholder");
    }
}

// Messaging nil is safe
NSString *nilStr = nil;
NSUInteger length = [nilStr length]; // returns 0, ไม่ crash
id result = [nilStr uppercaseString]; // returns nil
```

---

## ส่วนที่ 2: Memory Management และ ARC

### Q6: อธิบาย Memory Management ใน Objective-C ก่อน ARC

**คำตอบ:**
ก่อน ARC ใช้ Manual Reference Counting (MRC):

```objc
// Manual Reference Counting
- (void)manualMemoryManagement {
    // alloc - retain count = 1
    NSObject *obj = [[NSObject alloc] init];
    
    // retain - retain count + 1 = 2
    [obj retain];
    
    // release - retain count - 1 = 1
    [obj release];
    
    // release - retain count - 1 = 0 -> dealloc
    [obj release];
    
    // autorelease - release เมื่อ autorelease pool drain
    NSObject *autoObj = [[[NSObject alloc] init] autorelease];
}

// Ownership rules (NARC)
// N - New: alloc, new, copy, mutableCopy -> retain count = 1
// A - Acquired: retain -> retain count + 1  
// R - Release: release, autorelease -> retain count - 1
// C - Call dealloc automatically when count = 0
```

---

### Q7: ARC ทำงานอย่างไร?

**คำตอบ:**
ARC (Automatic Reference Counting) คือ compiler feature ที่ insert retain/release calls ให้อัตโนมัติ

```objc
// ARC - compiler จัดการให้
- (void)arcExample {
    // compiler เพิ่ม retain เมื่อกำหนดค่า
    NSString *str = [[NSString alloc] initWithString:@"Hello"];
    
    // compiler เพิ่ม release เมื่อ str ออก scope
} // -> [str release] ถูกเพิ่มโดย compiler

// Ownership qualifiers
__strong // default - ถือ strong reference
__weak   // ไม่ retain, nil เมื่อ object deallocate
__unsafe_unretained // ไม่ retain, ไม่ nil เมื่อ deallocate (อันตราย)
__autoreleasing     // สำหรับ out parameters

// Strong reference
@property (nonatomic, strong) NSString *name;

// Weak reference
@property (nonatomic, weak) id<SomeDelegate> delegate;

// ตัวอย่าง weak
__weak typeof(self) weakSelf = self;
dispatch_async(dispatch_get_main_queue(), ^{
    // ใช้ weakSelf ใน block เพื่อหลีกเลี่ยง retain cycle
    [weakSelf updateUI];
});
```

---

### Q8: Retain Cycle คืออะไร? และป้องกันอย่างไร?

**คำตอบ:**
Retain cycle เกิดเมื่อ 2 objects ถือ strong reference ถึงกัน ทำให้ไม่มีวันถูก deallocate

```objc
// ปัญหา: Retain Cycle
@interface Parent : NSObject
@property (nonatomic, strong) Child *child; // strong
@end

@interface Child : NSObject
@property (nonatomic, strong) Parent *parent; // strong -> CYCLE!
@end

// Parent -> strong -> Child
// Child -> strong -> Parent
// ทั้งคู่ไม่มีวัน dealloc!

// วิธีแก้: ใช้ weak reference
@interface Child : NSObject
@property (nonatomic, weak) Parent *parent; // weak -> แก้แล้ว
@end

// Retain cycle ใน blocks
@interface ViewController : UIViewController
@property (nonatomic, strong) NSTimer *timer;
@end

@implementation ViewController

- (void)viewDidLoad {
    [super viewDidLoad];
    
    // BAD: self ถือ timer, timer ถือ block, block ถือ self
    self.timer = [NSTimer scheduledTimerWithTimeInterval:1.0
                                                  target:self
                                                selector:@selector(update)
                                                userInfo:nil
                                                 repeats:YES];
    
    // GOOD: ใช้ weakSelf
    __weak typeof(self) weakSelf = self;
    self.timer = [NSTimer scheduledTimerWithTimeInterval:1.0
                                                 repeats:YES
                                                   block:^(NSTimer *timer) {
        __strong typeof(weakSelf) strongSelf = weakSelf;
        if (strongSelf) {
            [strongSelf update];
        }
    }];
}

@end
```

---

### Q9: อธิบาย `weak` vs `assign` property

**คำตอบ:**
```objc
@interface MyClass : NSObject

// weak - zeroing weak reference
// เมื่อ object ถูก deallocate, property ถูก set เป็น nil อัตโนมัติ
// ใช้กับ object types เท่านั้น
@property (nonatomic, weak) id<MyDelegate> delegate;
@property (nonatomic, weak) NSObject *weakObject;

// assign - ไม่ retain, ไม่ nil เมื่อ deallocate
// ใช้กับ primitive types
@property (nonatomic, assign) NSInteger count;
@property (nonatomic, assign) BOOL isEnabled;
@property (nonatomic, assign) CGFloat width;

// unsafe_unretained - เหมือน assign แต่สำหรับ objects
// DANGEROUS: อาจเป็น dangling pointer
@property (nonatomic, unsafe_unretained) NSObject *dangerousRef;

@end

// การใช้งาน
MyClass *obj = [[MyClass alloc] init];
NSObject *temp = [[NSObject alloc] init];

obj.weakObject = temp;
obj.dangerousRef = temp;

temp = nil; // temp ถูก dealloc

// weakObject ถูก set เป็น nil อัตโนมัติ ✓
NSLog(@"weak: %@", obj.weakObject); // nil

// dangerousRef ยังชี้ไปที่ memory เดิม - CRASH อาจเกิดขึ้น!
// NSLog(@"unsafe: %@", obj.dangerousRef); // DANGEROUS!
```

---

### Q10: autorelease pool ทำงานอย่างไร?

**คำตอบ:**
```objc
// Autorelease pool จัดการ objects ที่ใช้ autorelease
- (void)autoreleaseExample {
    @autoreleasepool {
        // Objects ที่ถูก autorelease จะถูก release
        // เมื่อ pool ถูก drain (ออกจาก @autoreleasepool block)
        NSString *str = [NSString stringWithFormat:@"Hello %d", 42];
        // str ถูก autorelease อัตโนมัติใน ARC
        NSLog(@"%@", str);
    } // str ถูก release ที่นี่
    
    // ใช้ใน loops เพื่อลด memory pressure
    for (int i = 0; i < 1000000; i++) {
        @autoreleasepool {
            NSString *temp = [NSString stringWithFormat:@"Item %d", i];
            // Process temp
        } // release temp ทุก iteration
    }
}

// Main thread มี autorelease pool ที่ drain ทุก run loop cycle
// Background threads ต้องสร้าง pool เอง
- (void)backgroundThread {
    dispatch_async(dispatch_get_global_queue(0, 0), ^{
        @autoreleasepool {
            // Code here สามารถใช้ autoreleased objects ได้
            NSArray *data = [self processLargeData];
            dispatch_async(dispatch_get_main_queue(), ^{
                [self updateUIWithData:data];
            });
        }
    });
}
```

---

## ส่วนที่ 3: ARC Internals และ Runtime

### Q11: ARC ต่างจาก Garbage Collection อย่างไร?

**คำตอบ:**
| ประเด็น | ARC | Garbage Collection |
|--------|-----|-------------------|
| เวลาทำงาน | Compile time + runtime | Runtime เท่านั้น |
| ความแน่นอน | Deterministic | Non-deterministic |
| Performance | ดีกว่า | Stop-the-world pauses |
| Control | Manual weak/strong | อัตโนมัติทั้งหมด |
| Memory | ต่ำกว่า | อาจสูงกว่า |

```objc
// ARC - deterministic deallocation
{
    NSObject *obj = [[NSObject alloc] init];
    // obj ถูก dealloc เมื่อออก scope นี้แน่นอน
}

// GC - non-deterministic
// Object ถูกเก็บเมื่อ GC รัน ไม่รู้แน่นอนว่าเมื่อใด
```

---

### Q12: Objective-C Runtime คืออะไร?

**คำตอบ:**
Runtime คือ C library ที่ทำให้ Objective-C ทำงานได้ ประกอบด้วย:

```objc
#import <objc/runtime.h>
#import <objc/message.h>

// 1. Message passing - [obj method] แปลงเป็น objc_msgSend
// [receiver method] -> objc_msgSend(receiver, @selector(method))

id result = objc_msgSend(myObject, @selector(doSomething));

// 2. Introspection
Class cls = [NSObject class];
NSLog(@"Class: %@", NSStringFromClass(cls));
NSLog(@"Superclass: %@", NSStringFromClass([cls superclass]));

// 3. Method manipulation
Method originalMethod = class_getInstanceMethod([MyClass class], 
                                                @selector(original));
Method swizzledMethod = class_getInstanceMethod([MyClass class], 
                                                @selector(swizzled));
method_exchangeImplementations(originalMethod, swizzledMethod);

// 4. Dynamic class creation
Class newClass = objc_allocateClassPair([NSObject class], "DynamicClass", 0);
class_addMethod(newClass, @selector(hello), (IMP)helloImpl, "v@:");
objc_registerClassPair(newClass);

// 5. Associated objects
static void *AssociationKey = &AssociationKey;
objc_setAssociatedObject(obj, AssociationKey, value, 
                         OBJC_ASSOCIATION_RETAIN_NONATOMIC);
id associatedValue = objc_getAssociatedObject(obj, AssociationKey);
```

---

### Q13: Method Swizzling คืออะไร?

**คำตอบ:**
Method Swizzling คือการเปลี่ยน implementation ของ method ตอน runtime

```objc
#import <objc/runtime.h>

@implementation UIViewController (Tracking)

+ (void)load {
    static dispatch_once_t onceToken;
    dispatch_once(&onceToken, ^{
        Class cls = [self class];
        
        SEL originalSEL = @selector(viewWillAppear:);
        SEL swizzledSEL = @selector(tracked_viewWillAppear:);
        
        Method originalMethod = class_getInstanceMethod(cls, originalSEL);
        Method swizzledMethod = class_getInstanceMethod(cls, swizzledSEL);
        
        // ลอง add method ก่อน (กรณี subclass ไม่ override)
        BOOL added = class_addMethod(cls,
                                     originalSEL,
                                     method_getImplementation(swizzledMethod),
                                     method_getTypeEncoding(swizzledMethod));
        if (added) {
            class_replaceMethod(cls,
                               swizzledSEL,
                               method_getImplementation(originalMethod),
                               method_getTypeEncoding(originalMethod));
        } else {
            method_exchangeImplementations(originalMethod, swizzledMethod);
        }
    });
}

- (void)tracked_viewWillAppear:(BOOL)animated {
    // เพิ่ม tracking logic
    NSLog(@"ViewController %@ will appear", NSStringFromClass([self class]));
    
    // เรียก original (ตอนนี้ชื่อ tracked_viewWillAppear:)
    [self tracked_viewWillAppear:animated];
}

@end
```

---

### Q14: `objc_msgSend` ทำงานอย่างไร?

**คำตอบ:**
```
[receiver message:arg] 
    ↓
objc_msgSend(receiver, @selector(message:), arg)
    ↓
1. ดู isa pointer ของ receiver -> ได้ Class
2. ค้น method cache ของ Class
3. ถ้าไม่เจอ ค้น method list ของ Class
4. ถ้าไม่เจอ ขึ้นไป superclass (repeat)
5. ถ้าไม่เจอเลย -> doesNotRecognizeSelector: -> NSException
```

```objc
// Method lookup chain
@interface A : NSObject
- (void)foo;
@end

@interface B : A
// ไม่ override foo
@end

@interface C : B
- (void)foo; // override
@end

C *obj = [[C alloc] init];
[obj foo]; // ค้นหา foo ใน C -> พบ -> เรียก C's foo

B *bObj = [[B alloc] init];
[bObj foo]; // ค้นหาใน B -> ไม่พบ -> ค้นใน A -> พบ -> เรียก A's foo

// Message forwarding (เมื่อไม่พบ method)
@implementation MyClass

// Step 1: resolveInstanceMethod (dynamic method resolution)
+ (BOOL)resolveInstanceMethod:(SEL)sel {
    if (sel == @selector(dynamicMethod)) {
        class_addMethod(self, sel, (IMP)dynamicMethodImpl, "v@:");
        return YES;
    }
    return [super resolveInstanceMethod:sel];
}

// Step 2: forwardingTargetForSelector (fast forwarding)
- (id)forwardingTargetForSelector:(SEL)aSelector {
    if ([self.helper respondsToSelector:aSelector]) {
        return self.helper; // redirect to helper
    }
    return [super forwardingTargetForSelector:aSelector];
}

// Step 3: forwardInvocation (full forwarding)
- (NSMethodSignature *)methodSignatureForSelector:(SEL)aSelector {
    NSMethodSignature *sig = [super methodSignatureForSelector:aSelector];
    if (!sig) {
        // สร้าง signature สำหรับ unknown method
        sig = [NSMethodSignature signatureWithObjCTypes:"v@:"];
    }
    return sig;
}

- (void)forwardInvocation:(NSInvocation *)anInvocation {
    if ([self.proxy respondsToSelector:anInvocation.selector]) {
        [anInvocation invokeWithTarget:self.proxy];
    } else {
        [super forwardInvocation:anInvocation];
    }
}

@end
```

---

## ส่วนที่ 4: Category, Extension, และ Inheritance

### Q15: Category vs Extension แตกต่างกันอย่างไร?

**คำตอบ:**
```objc
// ===== Category =====
// - เพิ่ม method ให้ existing class (รวมถึง system classes)
// - แยก file ได้
// - ไม่สามารถเพิ่ม instance variable ได้ (แต่ใช้ associated objects ได้)
// - ไม่ต้องมี implementation ทั้งหมดใน category

@interface NSString (Utilities)
- (BOOL)isPalindrome;
- (NSString *)reversedString;
- (BOOL)containsEmoji;
@end

@implementation NSString (Utilities)

- (BOOL)isPalindrome {
    NSString *reversed = [[[self componentsSeparatedByCharactersInSet:
                            [NSCharacterSet whitespaceCharacterSet]]
                           componentsJoinedByString:@""] 
                          lowercaseString];
    NSString *original = [self lowercaseString];
    return [original isEqualToString:
            [NSString stringWithString:[[reversed mutableCopy] 
                                        reverseObjects]]];
}

- (NSString *)reversedString {
    NSMutableString *reversed = [NSMutableString string];
    for (NSInteger i = self.length - 1; i >= 0; i--) {
        [reversed appendFormat:@"%C", [self characterAtIndex:i]];
    }
    return [reversed copy];
}

@end

// ===== Extension (Class Extension) =====
// - Anonymous category
// - เพิ่ม private properties และ methods
// - ต้องอยู่ใน .m file ของ class เดิม
// - สามารถ redeclare property เป็น readwrite ได้

// ใน .h - public interface
@interface Person : NSObject
@property (nonatomic, readonly, strong) NSString *name;
- (instancetype)initWithName:(NSString *)name;
@end

// ใน .m - private extension
@interface Person ()
// redeclare เป็น readwrite สำหรับ internal use
@property (nonatomic, readwrite, strong) NSString *name;
// private properties
@property (nonatomic, strong) NSMutableArray *friends;
@property (nonatomic, assign) NSInteger internalScore;
@end

@implementation Person

- (instancetype)initWithName:(NSString *)name {
    self = [super init];
    if (self) {
        _name = name; // สามารถ set ได้เพราะ readwrite ใน extension
        _friends = [NSMutableArray array];
    }
    return self;
}

@end
```

---

### Q16: Category vs Inheritance เมื่อไหรควรใช้อะไร?

**คำตอบ:**
```objc
// ===== Inheritance ใช้เมื่อ =====
// 1. ต้องการ override behavior
// 2. ต้องการเพิ่ม state (instance variables)
// 3. ความสัมพันธ์แบบ "is-a"

@interface EnhancedButton : UIButton
@property (nonatomic, strong) UIColor *highlightColor;
@property (nonatomic, assign) CGFloat cornerRadius;

- (void)setupDefaultAppearance;
@end

@implementation EnhancedButton

- (void)setupDefaultAppearance {
    self.layer.cornerRadius = self.cornerRadius;
    self.backgroundColor = self.highlightColor;
}

- (void)setHighlighted:(BOOL)highlighted {
    [super setHighlighted:highlighted]; // override
    self.backgroundColor = highlighted ? 
        [self.highlightColor colorWithAlphaComponent:0.7] : 
        self.highlightColor;
}

@end

// ===== Category ใช้เมื่อ =====
// 1. เพิ่ม utility methods ให้ existing class
// 2. แยก implementation ออกเป็นหมวดหมู่ (logical grouping)
// 3. ไม่ต้องการ state เพิ่ม

@interface UIColor (AppTheme)
+ (UIColor *)appPrimaryColor;
+ (UIColor *)appSecondaryColor;
+ (UIColor *)appBackgroundColor;
@end

@implementation UIColor (AppTheme)
+ (UIColor *)appPrimaryColor {
    return [UIColor colorWithRed:0.2 green:0.6 blue:1.0 alpha:1.0];
}
@end

@interface NSDate (Formatting)
- (NSString *)formattedDateString;
- (NSString *)timeAgo;
- (BOOL)isToday;
- (BOOL)isYesterday;
@end
```

---

### Q17: Protocol vs Inheritance ต่างกันอย่างไร?

**คำตอบ:**
```objc
// ===== Inheritance =====
// - "is-a" relationship
// - Single inheritance เท่านั้น
// - ได้รับ implementation จาก parent

@interface Animal : NSObject
- (void)breathe;
- (void)eat;
@end

@interface Dog : Animal
- (void)bark;
@end

// ===== Protocol =====
// - "can-do" relationship
// - Multiple protocol adoption
// - ไม่มี implementation (ยกเว้น default implementation)

@protocol Swimmable <NSObject>
@required
- (void)swim;

@optional
- (float)swimSpeed;
@end

@protocol Flyable <NSObject>
- (void)fly;
- (float)maximumAltitude;
@end

// Duck สามารถ swim ได้ แต่ไม่ใช่ Fish
@interface Duck : Animal <Swimmable, Flyable>
@end

@implementation Duck
- (void)swim {
    NSLog(@"Duck is swimming");
}
- (void)fly {
    NSLog(@"Duck is flying");
}
- (float)maximumAltitude {
    return 1000.0f; // feet
}
@end

// การใช้ protocol สำหรับ delegation
@protocol TableViewDataSource <NSObject>
@required
- (NSInteger)numberOfRows;
- (UITableViewCell *)cellForRow:(NSInteger)row;

@optional
- (NSString *)titleForRow:(NSInteger)row;
@end

@interface DataTableView : UIView
@property (nonatomic, weak) id<TableViewDataSource> dataSource;
@end
```

---

## ส่วนที่ 5: Properties และ KVC/KVO

### Q18: อธิบาย Property Attributes

**คำตอบ:**
```objc
@interface MyObject : NSObject

// Atomicity
@property (atomic, strong) NSString *atomicString;      // thread-safe (default)
@property (nonatomic, strong) NSString *nonatomicString; // faster, not thread-safe

// Memory management
@property (nonatomic, strong) NSObject *strongProp;   // retain reference
@property (nonatomic, weak) NSObject *weakProp;       // zeroing weak
@property (nonatomic, copy) NSString *copiedString;   // copy on set
@property (nonatomic, assign) NSInteger intProp;      // primitive, no memory mgmt

// Access control
@property (nonatomic, readonly) NSString *readOnlyProp;   // no setter generated
@property (nonatomic, readwrite) NSString *readWriteProp; // default

// Nullability (Swift interop)
@property (nonatomic, strong, nullable) NSString *maybeNil;
@property (nonatomic, strong, nonnull) NSString *neverNil;

@end

// ทำไมต้องใช้ copy กับ NSString?
@interface Config : NSObject
// BAD: ถ้าใช้ strong กับ NSString
@property (nonatomic, strong) NSString *badName;

// GOOD: ใช้ copy เพื่อป้องกัน NSMutableString
@property (nonatomic, copy) NSString *goodName;
@end

@implementation Config
- (void)demonstration {
    NSMutableString *mutable = [NSMutableString stringWithString:@"John"];
    
    self.badName = mutable;  // strong reference to mutable
    self.goodName = mutable; // copy -> immutable copy
    
    [mutable appendString:@" Doe"];
    
    NSLog(@"bad: %@", self.badName);   // "John Doe" - ค่าเปลี่ยน!
    NSLog(@"good: %@", self.goodName); // "John" - ค่าไม่เปลี่ยน
}
@end
```

---

### Q19: KVC (Key-Value Coding) คืออะไร?

**คำตอบ:**
```objc
// KVC ให้เข้าถึง properties ผ่าน string keys
@interface Person : NSObject
@property (nonatomic, strong) NSString *name;
@property (nonatomic, assign) NSInteger age;
@property (nonatomic, strong) Address *address;
@end

@interface Address : NSObject
@property (nonatomic, strong) NSString *city;
@property (nonatomic, strong) NSString *country;
@end

// การใช้งาน KVC
Person *person = [[Person alloc] init];

// setValue:forKey: (setName:)
[person setValue:@"สมชาย" forKey:@"name"];
[person setValue:@(30) forKey:@"age"];

// valueForKey: (name)
NSString *name = [person valueForKey:@"name"];
NSNumber *age = [person valueForKey:@"age"];

// Key path - เข้าถึง nested properties
[person setValue:@"Bangkok" forKeyPath:@"address.city"];
NSString *city = [person valueForKeyPath:@"address.city"];

// KVC กับ Arrays
NSArray *people = @[person1, person2, person3];

// Collection operators
NSNumber *avgAge = [people valueForKeyPath:@"@avg.age"];
NSNumber *maxAge = [people valueForKeyPath:@"@max.age"];
NSNumber *minAge = [people valueForKeyPath:@"@min.age"];
NSNumber *count = [people valueForKeyPath:@"@count"];
NSArray *names = [people valueForKeyPath:@"@distinctUnionOfObjects.name"];

// Custom KVC validation
- (BOOL)validateAge:(id *)ioValue error:(NSError **)outError {
    NSNumber *age = *ioValue;
    if ([age integerValue] < 0 || [age integerValue] > 150) {
        if (outError) {
            *outError = [NSError errorWithDomain:@"PersonDomain"
                                           code:1
                                       userInfo:@{NSLocalizedDescriptionKey: 
                                                  @"อายุต้องอยู่ระหว่าง 0-150"}];
        }
        return NO;
    }
    return YES;
}
```

---

### Q20: KVO (Key-Value Observing) คืออะไรและทำงานอย่างไร?

**คำตอบ:**
```objc
// KVO ให้ observe การเปลี่ยนแปลงของ property
@interface StockTracker : NSObject
@property (nonatomic, assign) double price;
@property (nonatomic, strong) NSString *symbol;
@end

@interface PriceMonitor : NSObject
@property (nonatomic, strong) StockTracker *stock;
@end

@implementation PriceMonitor

- (void)startMonitoring:(StockTracker *)stock {
    self.stock = stock;
    
    // เพิ่ม observer
    [stock addObserver:self
            forKeyPath:@"price"
               options:NSKeyValueObservingOptionNew | 
                       NSKeyValueObservingOptionOld
               context:NULL];
}

// รับ notification เมื่อ value เปลี่ยน
- (void)observeValueForKeyPath:(NSString *)keyPath
                      ofObject:(id)object
                        change:(NSDictionary *)change
                       context:(void *)context {
    if ([keyPath isEqualToString:@"price"]) {
        double oldPrice = [change[NSKeyValueChangeOldKey] doubleValue];
        double newPrice = [change[NSKeyValueChangeNewKey] doubleValue];
        
        NSLog(@"ราคาเปลี่ยนจาก %.2f เป็น %.2f", oldPrice, newPrice);
        
        if (newPrice > oldPrice) {
            NSLog(@"ราคาเพิ่มขึ้น %.2f%%", 
                  ((newPrice - oldPrice) / oldPrice) * 100);
        }
    }
}

- (void)stopMonitoring {
    // ต้อง remove observer เสมอ!
    [self.stock removeObserver:self forKeyPath:@"price"];
}

- (void)dealloc {
    [self stopMonitoring]; // ทำใน dealloc เพื่อความปลอดภัย
}

@end

// KVO ทำงานอย่างไรภายใน
// Runtime สร้าง subclass ชื่อ NSKVONotifying_StockTracker
// Override setter:
- (void)setPrice:(double)price {
    [self willChangeValueForKey:@"price"];
    _price = price; // actual set
    [self didChangeValueForKey:@"price"];
}
```

---

## ส่วนที่ 6: GCD และ Concurrency

### Q21: อธิบาย GCD (Grand Central Dispatch)

**คำตอบ:**
```objc
// GCD จัดการ thread pool ให้อัตโนมัติ

// Serial Queue - งานทำทีละงานตามลำดับ
dispatch_queue_t serialQueue = dispatch_queue_create("com.app.serial", 
                                                      DISPATCH_QUEUE_SERIAL);

// Concurrent Queue - งานทำพร้อมกันได้
dispatch_queue_t concurrentQueue = dispatch_queue_create("com.app.concurrent", 
                                                          DISPATCH_QUEUE_CONCURRENT);

// Global Queue - system-provided concurrent queues
dispatch_queue_t highPriority = dispatch_get_global_queue(DISPATCH_QUEUE_PRIORITY_HIGH, 0);
dispatch_queue_t defaultQueue = dispatch_get_global_queue(DISPATCH_QUEUE_PRIORITY_DEFAULT, 0);
dispatch_queue_t lowPriority = dispatch_get_global_queue(DISPATCH_QUEUE_PRIORITY_LOW, 0);
dispatch_queue_t background = dispatch_get_global_queue(DISPATCH_QUEUE_PRIORITY_BACKGROUND, 0);

// Main Queue - serial, runs on main thread
dispatch_queue_t mainQueue = dispatch_get_main_queue();

// async - ไม่รอ
dispatch_async(dispatch_get_global_queue(0, 0), ^{
    // Background work
    NSData *data = [NSData dataWithContentsOfURL:url];
    
    dispatch_async(dispatch_get_main_queue(), ^{
        // Update UI on main thread
        self.imageView.image = [UIImage imageWithData:data];
    });
});

// sync - รอจนเสร็จ (อาจทำให้ deadlock ถ้าใช้ผิด!)
dispatch_sync(serialQueue, ^{
    // This blocks current thread until done
    [self.database save];
});

// group - รอหลาย tasks
dispatch_group_t group = dispatch_group_create();

dispatch_group_async(group, concurrentQueue, ^{
    [self fetchUserData];
});

dispatch_group_async(group, concurrentQueue, ^{
    [self fetchProductData];
});

dispatch_group_notify(group, dispatch_get_main_queue(), ^{
    // เรียกเมื่อทั้งสอง tasks เสร็จ
    [self updateUI];
});

// semaphore - ควบคุม concurrent access
dispatch_semaphore_t semaphore = dispatch_semaphore_create(3); // max 3 concurrent

for (NSURL *url in urls) {
    dispatch_async(concurrentQueue, ^{
        dispatch_semaphore_wait(semaphore, DISPATCH_TIME_FOREVER);
        [self downloadFromURL:url completion:^{
            dispatch_semaphore_signal(semaphore);
        }];
    });
}
```

---

### Q22: NSOperationQueue vs GCD

**คำตอบ:**
```objc
// GCD - ต่ำระดับกว่า, performance ดี
dispatch_async(queue, ^{
    // work
});

// NSOperationQueue - สูงระดับกว่า, features มากกว่า
NSOperationQueue *queue = [[NSOperationQueue alloc] init];
queue.maxConcurrentOperationCount = 3;
queue.qualityOfService = NSQualityOfServiceUtility;

// NSOperation - สามารถ cancel, pause, monitor ได้
NSBlockOperation *op = [NSBlockOperation blockOperationWithBlock:^{
    [self doWork];
}];

// Dependencies
NSOperation *download = [NSBlockOperation blockOperationWithBlock:^{
    [self downloadData];
}];

NSOperation *process = [NSBlockOperation blockOperationWithBlock:^{
    [self processData];
}];

// process รอ download เสร็จก่อน
[process addDependency:download];

[queue addOperation:download];
[queue addOperation:process];

// Custom NSOperation
@interface DataFetchOperation : NSOperation

@property (nonatomic, strong) NSURL *url;
@property (nonatomic, strong) NSData *result;
@property (nonatomic, copy) void (^completion)(NSData *data, NSError *error);

@end

@implementation DataFetchOperation

- (void)main {
    if (self.isCancelled) return;
    
    NSError *error = nil;
    NSData *data = [NSData dataWithContentsOfURL:self.url
                                         options:0
                                           error:&error];
    
    if (self.isCancelled) return;
    
    self.result = data;
    
    if (self.completion) {
        dispatch_async(dispatch_get_main_queue(), ^{
            self.completion(data, error);
        });
    }
}

@end
```

---

### Q23: Thread Safety และ Race Condition

**คำตอบ:**
```objc
// Race Condition - หลาย threads อ่าน/เขียน data พร้อมกัน

// ปัญหา
@interface Counter : NSObject
@property (nonatomic, assign) NSInteger count;
@end

// UNSAFE - race condition
- (void)increment {
    self.count++; // read-increment-write ไม่ atomic!
}

// วิธีแก้ 1: @synchronized
- (void)safeIncrement {
    @synchronized(self) {
        self.count++;
    }
}

// วิธีแก้ 2: Serial queue
@interface SafeCounter : NSObject {
    dispatch_queue_t _queue;
    NSInteger _count;
}
@end

@implementation SafeCounter

- (instancetype)init {
    self = [super init];
    if (self) {
        _queue = dispatch_queue_create("com.app.counter", DISPATCH_QUEUE_SERIAL);
    }
    return self;
}

- (void)increment {
    dispatch_async(_queue, ^{
        self->_count++;
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

// วิธีแก้ 3: Barrier (สำหรับ readers-writers)
@interface ThreadSafeCache : NSObject {
    dispatch_queue_t _queue;
    NSMutableDictionary *_cache;
}
@end

@implementation ThreadSafeCache

- (instancetype)init {
    self = [super init];
    if (self) {
        _queue = dispatch_queue_create("com.app.cache", DISPATCH_QUEUE_CONCURRENT);
        _cache = [NSMutableDictionary dictionary];
    }
    return self;
}

// Multiple readers allowed
- (id)objectForKey:(id)key {
    __block id result;
    dispatch_sync(_queue, ^{
        result = self->_cache[key];
    });
    return result;
}

// Only one writer at a time, no readers during write
- (void)setObject:(id)obj forKey:(id)key {
    dispatch_barrier_async(_queue, ^{
        self->_cache[key] = obj;
    });
}

@end
```

---

## ส่วนที่ 7: iOS Architecture

### Q24: MVC, MVP, MVVM ต่างกันอย่างไร?

**คำตอบ:**

**MVC (Model-View-Controller)**
```objc
// Model
@interface UserModel : NSObject
@property (nonatomic, strong) NSString *name;
@property (nonatomic, strong) NSString *email;
@end

// View - UIView subclass หรือ .xib/.storyboard
@interface UserView : UIView
@property (nonatomic, strong) UILabel *nameLabel;
@property (nonatomic, strong) UILabel *emailLabel;
@end

// Controller - ผูก Model กับ View
@interface UserViewController : UIViewController
@property (nonatomic, strong) UserModel *user;
@property (nonatomic, strong) UserView *userView;
@end

@implementation UserViewController

- (void)viewDidLoad {
    [super viewDidLoad];
    [self updateView];
}

- (void)updateView {
    self.userView.nameLabel.text = self.user.name;
    self.userView.emailLabel.text = self.user.email;
}

@end
```

**MVVM (Model-View-ViewModel)**
```objc
// Model
@interface User : NSObject
@property (nonatomic, strong) NSString *firstName;
@property (nonatomic, strong) NSString *lastName;
@property (nonatomic, strong) NSString *email;
@end

// ViewModel - ไม่รู้จัก View
@interface UserViewModel : NSObject

@property (nonatomic, readonly) NSString *displayName;
@property (nonatomic, readonly) NSString *email;
@property (nonatomic, readonly) BOOL isEmailValid;

- (instancetype)initWithUser:(User *)user;
- (void)updateUser:(User *)user;

@end

@implementation UserViewModel {
    User *_user;
}

- (instancetype)initWithUser:(User *)user {
    self = [super init];
    if (self) {
        _user = user;
    }
    return self;
}

- (NSString *)displayName {
    return [NSString stringWithFormat:@"%@ %@", _user.firstName, _user.lastName];
}

- (BOOL)isEmailValid {
    NSString *pattern = @"[A-Z0-9a-z._%+-]+@[A-Za-z0-9.-]+\\.[A-Za-z]{2,}";
    NSPredicate *pred = [NSPredicate predicateWithFormat:@"SELF MATCHES %@", pattern];
    return [pred evaluateWithObject:_user.email];
}

@end

// View/ViewController - bind กับ ViewModel
@interface UserViewController : UIViewController
@property (nonatomic, strong) UserViewModel *viewModel;
@end

@implementation UserViewController

- (void)viewDidLoad {
    [super viewDidLoad];
    [self bindViewModel];
}

- (void)bindViewModel {
    self.nameLabel.text = self.viewModel.displayName;
    self.emailLabel.text = self.viewModel.email;
    self.emailLabel.textColor = self.viewModel.isEmailValid ? 
        [UIColor greenColor] : [UIColor redColor];
}

@end
```

---

### Q25: Delegate Pattern ทำงานอย่างไร?

**คำตอบ:**
```objc
// Protocol definition
@protocol DownloadManagerDelegate <NSObject>

@required
- (void)downloadManager:(DownloadManager *)manager 
       didFinishWithData:(NSData *)data;
- (void)downloadManager:(DownloadManager *)manager 
       didFailWithError:(NSError *)error;

@optional
- (void)downloadManager:(DownloadManager *)manager 
    didUpdateProgress:(float)progress;

@end

// Manager class
@interface DownloadManager : NSObject
@property (nonatomic, weak) id<DownloadManagerDelegate> delegate;
- (void)downloadFromURL:(NSURL *)url;
@end

@implementation DownloadManager

- (void)downloadFromURL:(NSURL *)url {
    NSURLSession *session = [NSURLSession sharedSession];
    NSURLSessionDataTask *task = [session dataTaskWithURL:url
                                       completionHandler:^(NSData *data, 
                                                          NSURLResponse *response, 
                                                          NSError *error) {
        dispatch_async(dispatch_get_main_queue(), ^{
            if (error) {
                if ([self.delegate respondsToSelector:
                     @selector(downloadManager:didFailWithError:)]) {
                    [self.delegate downloadManager:self didFailWithError:error];
                }
            } else {
                [self.delegate downloadManager:self didFinishWithData:data];
            }
        });
    }];
    [task resume];
}

@end

// ViewController implementing delegate
@interface MainViewController : UIViewController <DownloadManagerDelegate>
@end

@implementation MainViewController

- (void)viewDidLoad {
    [super viewDidLoad];
    
    DownloadManager *manager = [[DownloadManager alloc] init];
    manager.delegate = self;
    [manager downloadFromURL:[NSURL URLWithString:@"https://example.com/data"]];
}

- (void)downloadManager:(DownloadManager *)manager didFinishWithData:(NSData *)data {
    // Process data
    NSLog(@"Downloaded %lu bytes", (unsigned long)data.length);
}

- (void)downloadManager:(DownloadManager *)manager didFailWithError:(NSError *)error {
    [self showAlertWithMessage:error.localizedDescription];
}

@end
```

---

## ส่วนที่ 8: Data Structures ใน Objective-C

### Q26: NSArray vs NSMutableArray ต่างกันอย่างไร?

**คำตอบ:**
```objc
// NSArray - immutable
NSArray *immutable = @[@"apple", @"banana", @"cherry"];
// immutable[0] = @"mango"; // ERROR: ไม่ได้!

// การเข้าถึง
NSString *first = immutable[0];
NSString *last = [immutable lastObject];
NSUInteger count = immutable.count;

// NSMutableArray - mutable
NSMutableArray *mutable = [NSMutableArray arrayWithArray:immutable];
[mutable addObject:@"date"];
[mutable insertObject:@"elderberry" atIndex:2];
[mutable removeObjectAtIndex:1];
[mutable removeLastObject];
[mutable replaceObjectAtIndex:0 withObject:@"mango"];

// Sorting
NSArray *sorted = [immutable sortedArrayUsingSelector:@selector(compare:)];

NSArray *customSorted = [mutable sortedArrayUsingComparator:
    ^NSComparisonResult(NSString *a, NSString *b) {
        return [a compare:b options:NSCaseInsensitiveSearch];
    }];

// Filtering
NSArray *longNames = [immutable filteredArrayUsingPredicate:
    [NSPredicate predicateWithBlock:^BOOL(NSString *name, NSDictionary *bindings) {
        return name.length > 5;
    }]];

// Map (valueForKey - limited)
NSArray *upperNames = [immutable valueForKey:@"uppercaseString"];

// Performance tips
// - NSArray ดีกว่า NSMutableArray สำหรับ read-only
// - ใช้ [mutableArray copy] เพื่อ immutable snapshot
// - Fast enumeration เร็วกว่า for loop ด้วย index
```

---

### Q27: NSDictionary ทำงานอย่างไร?

**คำตอบ:**
```objc
// NSDictionary - key-value pairs, immutable
NSDictionary *dict = @{
    @"name": @"John",
    @"age": @30,
    @"city": @"Bangkok"
};

// การเข้าถึง
NSString *name = dict[@"name"];  // subscript syntax
NSNumber *age = [dict objectForKey:@"age"];

// ตรวจสอบ key
if (dict[@"email"]) {
    NSLog(@"has email");
}

// iterate
[dict enumerateKeysAndObjectsUsingBlock:^(id key, id obj, BOOL *stop) {
    NSLog(@"%@: %@", key, obj);
    if ([key isEqualToString:@"name"]) {
        *stop = YES; // stop early
    }
}];

// NSMutableDictionary
NSMutableDictionary *mutable = [NSMutableDictionary dictionaryWithDictionary:dict];
mutable[@"email"] = @"john@example.com";
[mutable removeObjectForKey:@"city"];

// Nested dictionaries
NSDictionary *nested = @{
    @"user": @{
        @"name": @"Jane",
        @"address": @{
            @"street": @"123 Main St",
            @"city": @"Bangkok"
        }
    }
};

// KVC key path
NSString *city = [nested valueForKeyPath:@"user.address.city"];
NSLog(@"City: %@", city); // "Bangkok"
```

---

### Q28: Algorithms ใน Objective-C - Binary Search

**คำตอบ:**
```objc
// Binary Search
@interface Algorithms : NSObject

+ (NSInteger)binarySearch:(NSArray *)sortedArray target:(id)target;
+ (NSArray *)mergeSort:(NSArray *)array;
+ (NSArray *)quickSort:(NSArray *)array;

@end

@implementation Algorithms

+ (NSInteger)binarySearch:(NSArray *)sortedArray target:(id)target {
    NSInteger left = 0;
    NSInteger right = sortedArray.count - 1;
    
    while (left <= right) {
        NSInteger mid = left + (right - left) / 2;
        NSComparisonResult result = [sortedArray[mid] compare:target];
        
        if (result == NSOrderedSame) {
            return mid; // found
        } else if (result == NSOrderedAscending) {
            left = mid + 1; // target อยู่ขวา
        } else {
            right = mid - 1; // target อยู่ซ้าย
        }
    }
    
    return NSNotFound;
}

+ (NSArray *)mergeSort:(NSArray *)array {
    if (array.count <= 1) return array;
    
    NSInteger mid = array.count / 2;
    NSArray *left = [self mergeSort:[array subarrayWithRange:NSMakeRange(0, mid)]];
    NSArray *right = [self mergeSort:[array subarrayWithRange:NSMakeRange(mid, array.count - mid)]];
    
    return [self merge:left with:right];
}

+ (NSArray *)merge:(NSArray *)left with:(NSArray *)right {
    NSMutableArray *result = [NSMutableArray array];
    NSInteger i = 0, j = 0;
    
    while (i < left.count && j < right.count) {
        if ([left[i] compare:right[j]] != NSOrderedDescending) {
            [result addObject:left[i++]];
        } else {
            [result addObject:right[j++]];
        }
    }
    
    while (i < left.count) [result addObject:left[i++]];
    while (j < right.count) [result addObject:right[j++]];
    
    return [result copy];
}

@end

// Stack implementation
@interface Stack : NSObject

- (void)push:(id)object;
- (id)pop;
- (id)peek;
- (BOOL)isEmpty;
- (NSUInteger)size;

@end

@implementation Stack {
    NSMutableArray *_storage;
}

- (instancetype)init {
    self = [super init];
    if (self) {
        _storage = [NSMutableArray array];
    }
    return self;
}

- (void)push:(id)object {
    [_storage addObject:object];
}

- (id)pop {
    if (self.isEmpty) return nil;
    id obj = _storage.lastObject;
    [_storage removeLastObject];
    return obj;
}

- (id)peek {
    return _storage.lastObject;
}

- (BOOL)isEmpty {
    return _storage.count == 0;
}

- (NSUInteger)size {
    return _storage.count;
}

@end
```

---

## ส่วนที่ 9: System Design Questions

### Q29: ออกแบบ Cache System

**คำตอบ:**
```objc
// LRU Cache
@interface LRUCache : NSObject

- (instancetype)initWithCapacity:(NSInteger)capacity;
- (id)objectForKey:(id)key;
- (void)setObject:(id)object forKey:(id)key;

@end

@interface CacheNode : NSObject
@property (nonatomic, strong) id key;
@property (nonatomic, strong) id value;
@property (nonatomic, weak) CacheNode *prev;
@property (nonatomic, strong) CacheNode *next;
@end

@implementation CacheNode
@end

@implementation LRUCache {
    NSInteger _capacity;
    NSMutableDictionary *_map;
    CacheNode *_head; // dummy head
    CacheNode *_tail; // dummy tail
}

- (instancetype)initWithCapacity:(NSInteger)capacity {
    self = [super init];
    if (self) {
        _capacity = capacity;
        _map = [NSMutableDictionary dictionary];
        
        // dummy nodes
        _head = [[CacheNode alloc] init];
        _tail = [[CacheNode alloc] init];
        _head.next = _tail;
        _tail.prev = _head;
    }
    return self;
}

- (id)objectForKey:(id)key {
    CacheNode *node = _map[key];
    if (!node) return nil;
    
    // Move to front (most recently used)
    [self removeNode:node];
    [self addToFront:node];
    
    return node.value;
}

- (void)setObject:(id)object forKey:(id)key {
    CacheNode *existing = _map[key];
    
    if (existing) {
        existing.value = object;
        [self removeNode:existing];
        [self addToFront:existing];
    } else {
        if (_map.count >= _capacity) {
            // Remove LRU (tail.prev)
            CacheNode *lru = _tail.prev;
            [self removeNode:lru];
            [_map removeObjectForKey:lru.key];
        }
        
        CacheNode *newNode = [[CacheNode alloc] init];
        newNode.key = key;
        newNode.value = object;
        [self addToFront:newNode];
        _map[key] = newNode;
    }
}

- (void)removeNode:(CacheNode *)node {
    node.prev.next = node.next;
    node.next.prev = node.prev;
}

- (void)addToFront:(CacheNode *)node {
    node.next = _head.next;
    node.prev = _head;
    _head.next.prev = node;
    _head.next = node;
}

@end
```

---

### Q30: ออกแบบ Notification Center

**คำตอบ:**
```objc
// Simple notification center
@interface SimpleNotificationCenter : NSObject

+ (instancetype)defaultCenter;

- (void)addObserver:(id)observer
           selector:(SEL)selector
               name:(NSString *)name;

- (void)postNotificationName:(NSString *)name
                      object:(id)object
                    userInfo:(NSDictionary *)userInfo;

- (void)removeObserver:(id)observer;

@end

@implementation SimpleNotificationCenter {
    NSMutableDictionary<NSString *, NSMutableArray *> *_observers;
    dispatch_queue_t _queue;
}

+ (instancetype)defaultCenter {
    static SimpleNotificationCenter *center = nil;
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
        _queue = dispatch_queue_create("com.notification.center", 
                                       DISPATCH_QUEUE_CONCURRENT);
    }
    return self;
}

- (void)addObserver:(id)observer selector:(SEL)selector name:(NSString *)name {
    dispatch_barrier_async(_queue, ^{
        if (!self->_observers[name]) {
            self->_observers[name] = [NSMutableArray array];
        }
        
        NSDictionary *entry = @{
            @"observer": observer,
            @"selector": NSStringFromSelector(selector)
        };
        [self->_observers[name] addObject:entry];
    });
}

- (void)postNotificationName:(NSString *)name object:(id)object userInfo:(NSDictionary *)userInfo {
    dispatch_sync(_queue, ^{
        NSArray *entries = [self->_observers[name] copy];
        
        dispatch_async(dispatch_get_main_queue(), ^{
            for (NSDictionary *entry in entries) {
                id observer = entry[@"observer"];
                SEL selector = NSSelectorFromString(entry[@"selector"]);
                
                if ([observer respondsToSelector:selector]) {
                    NSNotification *notification = [NSNotification 
                        notificationWithName:name 
                        object:object 
                        userInfo:userInfo];
#pragma clang diagnostic push
#pragma clang diagnostic ignored "-Warc-performSelector-leaks"
                    [observer performSelector:selector withObject:notification];
#pragma clang diagnostic pop
                }
            }
        });
    });
}

@end
```

---

## ส่วนที่ 10: Coding Challenges

### Q31: Reverse a Linked List

```objc
@interface ListNode : NSObject
@property (nonatomic, assign) NSInteger val;
@property (nonatomic, strong) ListNode *next;
@end

@implementation ListNode
@end

// Iterative approach
ListNode* reverseList(ListNode *head) {
    ListNode *prev = nil;
    ListNode *current = head;
    
    while (current != nil) {
        ListNode *nextNode = current.next;
        current.next = prev;
        prev = current;
        current = nextNode;
    }
    
    return prev;
}

// Recursive approach
ListNode* reverseListRecursive(ListNode *head) {
    if (head == nil || head.next == nil) return head;
    
    ListNode *newHead = reverseListRecursive(head.next);
    head.next.next = head;
    head.next = nil;
    
    return newHead;
}
```

---

### Q32: Valid Parentheses

```objc
BOOL isValidParentheses(NSString *s) {
    NSMutableArray *stack = [NSMutableArray array];
    NSDictionary *matching = @{@")": @"(", @"]": @"[", @"}": @"{"};
    
    for (NSUInteger i = 0; i < s.length; i++) {
        NSString *ch = [s substringWithRange:NSMakeRange(i, 1)];
        
        if ([ch isEqualToString:@"("] || 
            [ch isEqualToString:@"["] || 
            [ch isEqualToString:@"{"]) {
            [stack addObject:ch];
        } else {
            if (stack.count == 0 || ![stack.lastObject isEqualToString:matching[ch]]) {
                return NO;
            }
            [stack removeLastObject];
        }
    }
    
    return stack.count == 0;
}
```

---

### Q33: Two Sum

```objc
NSArray* twoSum(NSArray *nums, NSInteger target) {
    NSMutableDictionary *seen = [NSMutableDictionary dictionary];
    
    for (NSInteger i = 0; i < nums.count; i++) {
        NSInteger num = [nums[i] integerValue];
        NSInteger complement = target - num;
        
        NSNumber *complementIdx = seen[@(complement)];
        if (complementIdx) {
            return @[complementIdx, @(i)];
        }
        
        seen[@(num)] = @(i);
    }
    
    return nil; // ไม่พบ
}
```

---

### Q34: Fibonacci Sequence

```objc
// Recursive - O(2^n)
NSInteger fibonacci(NSInteger n) {
    if (n <= 1) return n;
    return fibonacci(n - 1) + fibonacci(n - 2);
}

// Dynamic Programming - O(n)
NSInteger fibDP(NSInteger n) {
    if (n <= 1) return n;
    
    NSMutableArray *dp = [NSMutableArray arrayWithCapacity:n + 1];
    dp[0] = @0;
    dp[1] = @1;
    
    for (NSInteger i = 2; i <= n; i++) {
        dp[i] = @([dp[i-1] integerValue] + [dp[i-2] integerValue]);
    }
    
    return [dp[n] integerValue];
}

// Space optimized - O(1)
NSInteger fibOptimized(NSInteger n) {
    if (n <= 1) return n;
    
    NSInteger a = 0, b = 1;
    for (NSInteger i = 2; i <= n; i++) {
        NSInteger c = a + b;
        a = b;
        b = c;
    }
    return b;
}
```

---

## ส่วนที่ 11: More iOS-Specific Questions

### Q35: UIViewController Lifecycle

**คำตอบ:**
```objc
@implementation MyViewController

// 1. loadView - สร้าง view hierarchy
- (void)loadView {
    // เรียกเมื่อ self.view ถูก access แต่ยังไม่ถูกตั้งค่า
    // ถ้า override นี้ ต้องตั้งค่า self.view เอง
    self.view = [[UIView alloc] initWithFrame:UIScreen.mainScreen.bounds];
}

// 2. viewDidLoad - view ถูกโหลดแล้ว (เรียกครั้งเดียว)
- (void)viewDidLoad {
    [super viewDidLoad];
    // Setup UI, data, etc.
    NSLog(@"viewDidLoad"); 
}

// 3. viewWillAppear - กำลังจะแสดง (เรียกทุกครั้งที่จะแสดง)
- (void)viewWillAppear:(BOOL)animated {
    [super viewWillAppear:animated];
    NSLog(@"viewWillAppear");
    [self refreshData];
}

// 4. viewDidAppear - แสดงแล้ว
- (void)viewDidAppear:(BOOL)animated {
    [super viewDidAppear:animated];
    NSLog(@"viewDidAppear");
    [self startAnimations];
}

// 5. viewWillDisappear - กำลังจะหายไป
- (void)viewWillDisappear:(BOOL)animated {
    [super viewWillDisappear:animated];
    NSLog(@"viewWillDisappear");
    [self saveState];
}

// 6. viewDidDisappear - หายไปแล้ว
- (void)viewDidDisappear:(BOOL)animated {
    [super viewDidDisappear:animated];
    NSLog(@"viewDidDisappear");
    [self stopExpensiveOperations];
}

// 7. viewWillLayoutSubviews / viewDidLayoutSubviews
- (void)viewWillLayoutSubviews {
    [super viewWillLayoutSubviews];
    // ก่อน layout
}

- (void)viewDidLayoutSubviews {
    [super viewDidLayoutSubviews];
    // หลัง layout - ขนาด view ถูกต้องแล้ว
    self.roundButton.layer.cornerRadius = self.roundButton.bounds.height / 2;
}

// Memory warning
- (void)didReceiveMemoryWarning {
    [super didReceiveMemoryWarning];
    // ปล่อย cached data
    [self.imageCache removeAllObjects];
}

@end
```

---

### Q36: App Lifecycle

**คำตอบ:**
```objc
@implementation AppDelegate

// แอปเพิ่งเปิด
- (BOOL)application:(UIApplication *)application 
    didFinishLaunchingWithOptions:(NSDictionary *)launchOptions {
    // Setup หลัก
    [self setupDatabase];
    [self setupPushNotifications];
    return YES;
}

// กำลังจะไป background
- (void)applicationWillResignActive:(UIApplication *)application {
    // Pause operations (incoming call, etc.)
    [self pauseGame];
}

// ไป background แล้ว
- (void)applicationDidEnterBackground:(UIApplication *)application {
    // Save state, stop location updates, etc.
    [self saveApplicationState];
    [self stopLocationUpdates];
    
    // Request background time
    __block UIBackgroundTaskIdentifier bgTask = [application beginBackgroundTaskWithName:@"SaveData" 
        expirationHandler:^{
            [application endBackgroundTask:bgTask];
            bgTask = UIBackgroundTaskInvalid;
        }];
    
    dispatch_async(dispatch_get_global_queue(0, 0), ^{
        [self saveAllData];
        [application endBackgroundTask:bgTask];
        bgTask = UIBackgroundTaskInvalid;
    });
}

// กำลังจะกลับมา foreground
- (void)applicationWillEnterForeground:(UIApplication *)application {
    // Refresh data
}

// กลับมา active แล้ว
- (void)applicationDidBecomeActive:(UIApplication *)application {
    // Resume operations
    [self resumeGame];
    [self refreshContent];
}

// แอปกำลังจะปิด
- (void)applicationWillTerminate:(UIApplication *)application {
    // Final cleanup
    [self finalSave];
}

@end
```

---

### Q37: Core Data Basics

**คำตอบ:**
```objc
// Core Data Stack
@interface CoreDataManager : NSObject

@property (readonly) NSManagedObjectContext *mainContext;
@property (readonly) NSManagedObjectContext *backgroundContext;

+ (instancetype)sharedManager;
- (void)saveContext;

@end

@implementation CoreDataManager {
    NSPersistentContainer *_container;
}

+ (instancetype)sharedManager {
    static CoreDataManager *manager = nil;
    static dispatch_once_t token;
    dispatch_once(&token, ^{
        manager = [[CoreDataManager alloc] init];
    });
    return manager;
}

- (instancetype)init {
    self = [super init];
    if (self) {
        _container = [[NSPersistentContainer alloc] initWithName:@"MyApp"];
        [_container loadPersistentStoresWithCompletionHandler:
            ^(NSPersistentStoreDescription *desc, NSError *error) {
            if (error) {
                NSLog(@"Core Data error: %@", error);
            }
        }];
        _container.viewContext.automaticallyMergesChangesFromParent = YES;
    }
    return self;
}

- (NSManagedObjectContext *)mainContext {
    return _container.viewContext;
}

- (NSManagedObjectContext *)backgroundContext {
    return [_container newBackgroundContext];
}

- (void)saveContext {
    NSManagedObjectContext *ctx = self.mainContext;
    if (ctx.hasChanges) {
        NSError *error = nil;
        if (![ctx save:&error]) {
            NSLog(@"Save error: %@", error);
        }
    }
}

// CRUD Operations
- (void)createUserWithName:(NSString *)name email:(NSString *)email {
    NSManagedObjectContext *ctx = self.backgroundContext;
    [ctx performBlock:^{
        NSManagedObject *user = [NSEntityDescription insertNewObjectForEntityForName:@"User"
                                                              inManagedObjectContext:ctx];
        [user setValue:name forKey:@"name"];
        [user setValue:email forKey:@"email"];
        [user setValue:[NSDate date] forKey:@"createdAt"];
        
        NSError *error = nil;
        [ctx save:&error];
    }];
}

- (NSArray *)fetchUsersWithName:(NSString *)name {
    NSFetchRequest *request = [NSFetchRequest fetchRequestWithEntityName:@"User"];
    request.predicate = [NSPredicate predicateWithFormat:@"name CONTAINS[cd] %@", name];
    request.sortDescriptors = @[[NSSortDescriptor sortDescriptorWithKey:@"name" ascending:YES]];
    
    NSError *error = nil;
    NSArray *results = [self.mainContext executeFetchRequest:request error:&error];
    return results ?: @[];
}

@end
```

---

## ส่วนที่ 12: คำถามเพิ่มเติมและ Tips

### Q38: Blocks และ Closures

```objc
// Block syntax
typedef void (^CompletionBlock)(BOOL success, NSError *error);
typedef NSString *(^TransformBlock)(NSString *input);

@interface NetworkManager : NSObject
- (void)fetchDataWithCompletion:(CompletionBlock)completion;
@end

@implementation NetworkManager

- (void)fetchDataWithCompletion:(CompletionBlock)completion {
    dispatch_async(dispatch_get_global_queue(0, 0), ^{
        // Simulate network
        BOOL success = YES;
        NSError *error = nil;
        
        dispatch_async(dispatch_get_main_queue(), ^{
            if (completion) {
                completion(success, error);
            }
        });
    });
}

@end

// Block as property
@interface Button : NSObject
@property (nonatomic, copy) void (^tapHandler)(void);
@end

// Block variable capture
- (void)blockCapture {
    NSInteger counter = 0;
    
    void (^normalBlock)(void) = ^{
        // Captures counter by value - cannot modify
        NSLog(@"counter: %ld", (long)counter);
    };
    
    __block NSInteger modifiableCounter = 0;
    void (^modifyBlock)(void) = ^{
        // __block allows modification
        modifiableCounter++;
        NSLog(@"counter: %ld", (long)modifiableCounter);
    };
    
    modifyBlock(); // counter = 1
    modifyBlock(); // counter = 2
}
```

---

### Q39: Error Handling

```objc
// NSError pattern
- (BOOL)parseJSON:(NSData *)data 
           result:(NSDictionary **)result 
            error:(NSError **)error {
    NSError *jsonError = nil;
    id parsed = [NSJSONSerialization JSONObjectWithData:data
                                               options:0
                                                 error:&jsonError];
    
    if (jsonError) {
        if (error) {
            *error = [NSError errorWithDomain:@"ParserDomain"
                                         code:100
                                     userInfo:@{
                NSLocalizedDescriptionKey: @"JSON parsing failed",
                NSUnderlyingErrorKey: jsonError
            }];
        }
        return NO;
    }
    
    if (![parsed isKindOfClass:[NSDictionary class]]) {
        if (error) {
            *error = [NSError errorWithDomain:@"ParserDomain"
                                         code:101
                                     userInfo:@{
                NSLocalizedDescriptionKey: @"Expected dictionary"
            }];
        }
        return NO;
    }
    
    if (result) {
        *result = (NSDictionary *)parsed;
    }
    return YES;
}

// การใช้งาน
NSError *error = nil;
NSDictionary *result = nil;
if ([parser parseJSON:data result:&result error:&error]) {
    // Success
} else {
    NSLog(@"Error: %@", error.localizedDescription);
    NSLog(@"Underlying: %@", error.userInfo[NSUnderlyingErrorKey]);
}
```

---

### Q40: Singleton Pattern

```objc
// Thread-safe Singleton
@implementation DatabaseManager

+ (instancetype)sharedManager {
    static DatabaseManager *instance = nil;
    static dispatch_once_t onceToken;
    dispatch_once(&onceToken, ^{
        instance = [[self alloc] initPrivate];
    });
    return instance;
}

// ป้องกัน direct init
- (instancetype)init {
    [NSException raise:NSInternalInconsistencyException
                format:@"Use +sharedManager instead"];
    return nil;
}

- (instancetype)initPrivate {
    self = [super init];
    if (self) {
        // Initialize
    }
    return self;
}

// ป้องกัน copy
- (instancetype)copyWithZone:(NSZone *)zone {
    return self;
}

@end
```

---

## ส่วนที่ 13: คำถามระดับ Senior

### Q41: ออกแบบ Image Loading Library

```objc
@interface ImageLoader : NSObject

+ (instancetype)sharedLoader;

- (void)loadImageFromURL:(NSURL *)url
            placeholder:(UIImage *)placeholder
             completion:(void (^)(UIImage *image, NSError *error))completion;

- (void)cancelLoadForURL:(NSURL *)url;
- (void)clearCache;

@end

@implementation ImageLoader {
    NSCache *_memoryCache;
    NSOperationQueue *_downloadQueue;
    NSMutableDictionary<NSString *, NSOperation *> *_activeOperations;
    dispatch_queue_t _lockQueue;
}

- (instancetype)init {
    self = [super init];
    if (self) {
        _memoryCache = [[NSCache alloc] init];
        _memoryCache.countLimit = 100;
        _memoryCache.totalCostLimit = 50 * 1024 * 1024; // 50MB
        
        _downloadQueue = [[NSOperationQueue alloc] init];
        _downloadQueue.maxConcurrentOperationCount = 4;
        
        _activeOperations = [NSMutableDictionary dictionary];
        _lockQueue = dispatch_queue_create("com.imageloader.lock", 
                                           DISPATCH_QUEUE_SERIAL);
        
        // Listen for memory warnings
        [[NSNotificationCenter defaultCenter] 
            addObserver:self
               selector:@selector(handleMemoryWarning)
                   name:UIApplicationDidReceiveMemoryWarningNotification
                 object:nil];
    }
    return self;
}

- (void)loadImageFromURL:(NSURL *)url
            placeholder:(UIImage *)placeholder
             completion:(void (^)(UIImage *, NSError *))completion {
    NSString *key = url.absoluteString;
    
    // Check memory cache
    UIImage *cached = [_memoryCache objectForKey:key];
    if (cached) {
        dispatch_async(dispatch_get_main_queue(), ^{
            completion(cached, nil);
        });
        return;
    }
    
    // Check disk cache
    [self checkDiskCacheForKey:key completion:^(UIImage *diskImage) {
        if (diskImage) {
            [self->_memoryCache setObject:diskImage forKey:key];
            dispatch_async(dispatch_get_main_queue(), ^{
                completion(diskImage, nil);
            });
            return;
        }
        
        // Download
        NSOperation *op = [NSBlockOperation blockOperationWithBlock:^{
            NSError *error = nil;
            NSData *data = [NSData dataWithContentsOfURL:url options:0 error:&error];
            UIImage *image = data ? [UIImage imageWithData:data] : nil;
            
            if (image) {
                [self->_memoryCache setObject:image forKey:key];
                [self saveToDisk:data forKey:key];
            }
            
            dispatch_async(dispatch_get_main_queue(), ^{
                completion(image, error);
            });
        }];
        
        dispatch_async(self->_lockQueue, ^{
            self->_activeOperations[key] = op;
        });
        
        [self->_downloadQueue addOperation:op];
    }];
}

- (void)handleMemoryWarning {
    [_memoryCache removeAllObjects];
}

@end
```

---

### Q42-50: Quick Q&A

**Q42: isEqual: vs == สำหรับ NSString**
```objc
NSString *a = @"hello";
NSString *b = @"hello";
NSString *c = [NSString stringWithFormat:@"%@", @"hello"];

a == b;          // YES (string literals ใช้ interning)
a == c;          // อาจเป็น NO (different object)
[a isEqual:b];   // YES (content comparison)
[a isEqual:c];   // YES
```

**Q43: `copy` vs `mutableCopy`**
```objc
NSArray *original = @[@1, @2, @3];
NSArray *shallowCopy = [original copy];         // immutable copy
NSMutableArray *mutableCopy = [original mutableCopy]; // mutable copy
// Both are shallow - elements are NOT copied
```

**Q44: `@selector` vs `NSSelectorFromString`**
```objc
SEL sel1 = @selector(methodName:);        // compile-time check
SEL sel2 = NSSelectorFromString(@"methodName:"); // runtime, no compile check
```

**Q45: Toll-Free Bridging**
```objc
// Core Foundation <-> Objective-C
NSString *nsStr = @"Hello";
CFStringRef cfStr = (__bridge CFStringRef)nsStr;     // no transfer
CFStringRef cfStr2 = (__bridge_retained CFStringRef)nsStr; // transfer to CF (must CFRelease)
NSString *nsStr2 = (__bridge_transfer NSString *)cfStr2;   // transfer to ObjC (ARC takes over)
```

**Q46: `instancetype` vs `id`**
```objc
// id - any object type, no type checking
+ (id)create;

// instancetype - actual class type, enables type checking
+ (instancetype)create; // compiler knows return type
// [MyClass create].specificMethod; // OK with instancetype
```

**Q47: `__kindof`**
```objc
// __kindof - แสดงว่า return type เป็น class หรือ subclass
- (__kindof UIView *)viewForRow:(NSInteger)row;
// ไม่ต้อง cast: UIButton *btn = [self viewForRow:0];
```

**Q48: Designated Initializer**
```objc
@implementation MyClass

// Designated initializer - เต็มที่สุด
- (instancetype)initWithX:(CGFloat)x y:(CGFloat)y {
    self = [super init]; // เรียก super's designated init
    if (self) {
        _x = x;
        _y = y;
    }
    return self;
}

// Convenience initializer - เรียก designated init
- (instancetype)init {
    return [self initWithX:0 y:0]; // เรียก self's designated init
}

- (instancetype)initWithX:(CGFloat)x {
    return [self initWithX:x y:0];
}

@end
```

**Q49: `@dynamic` vs `@synthesize`**
```objc
@interface MyClass : NSObject
@property (nonatomic, strong) NSString *name;
@property (nonatomic, assign) NSInteger age;
@end

@implementation MyClass

// @synthesize - สร้าง getter/setter + ivar อัตโนมัติ (default ใน modern ObjC)
@synthesize name = _name; // _name คือ ivar backing store

// @dynamic - บอก compiler ว่า getter/setter จะ provide ตอน runtime
// ใช้ใน CoreData NSManagedObject
@dynamic age; // compiler ไม่ generate, แต่ไม่ warning

@end
```

**Q50: Fast Enumeration vs Block Enumeration**
```objc
NSArray *array = @[@1, @2, @3, @4, @5];

// Fast enumeration - เร็วที่สุด, ไม่สามารถ modify collection
for (NSNumber *num in array) {
    NSLog(@"%@", num);
}

// Block enumeration - ใช้ได้กับ concurrent, สามารถ stop
[array enumerateObjectsUsingBlock:^(NSNumber *num, NSUInteger idx, BOOL *stop) {
    NSLog(@"%ld: %@", (long)idx, num);
    if ([num integerValue] == 3) *stop = YES;
}];

// Concurrent enumeration
[array enumerateObjectsWithOptions:NSEnumerationConcurrent 
                        usingBlock:^(NSNumber *num, NSUInteger idx, BOOL *stop) {
    // Thread-safe operations only
}];
```

---

## ส่วนที่ 14: คำถามสัมภาษณ์ระดับ Advanced (Q51-100)

### Q51: Objective-C Generics

```objc
// Lightweight generics (ObjC 2015+)
@interface TypedArray<ObjectType> : NSObject

- (void)addObject:(ObjectType)object;
- (ObjectType)objectAtIndex:(NSUInteger)index;

@end

// ตัวอย่างการใช้
NSArray<NSString *> *strings = @[@"hello", @"world"];
NSString *first = strings[0]; // no cast needed

NSMutableArray<UIView *> *views = [NSMutableArray array];
[views addObject:[[UIView alloc] init]];
```

---

### Q52: NSHashTable และ NSMapTable

```objc
// NSSet ใช้ strong references
// NSHashTable ให้ custom memory semantics

// Weak references set
NSHashTable *weakSet = [NSHashTable weakObjectsHashTable];
// Objects ถูก remove อัตโนมัติเมื่อ deallocate

// NSMapTable - เหมือน NSDictionary แต่ flexible
NSMapTable *weakToStrong = [NSMapTable weakToStrongObjectsMapTable];
// Keys are weak, values are strong

NSMapTable *strongToWeak = [NSMapTable strongToWeakObjectsMapTable];
// Keys are strong, values are weak

id key = [[NSObject alloc] init];
NSMapTable *map = [NSMapTable strongToWeakObjectsMapTable];
map[key] = someObject; // someObject เป็น weak
```

---

### Q53: NSPointerArray

```objc
// เหมือน NSArray แต่ support nil และ weak refs
NSPointerArray *weakArray = [NSPointerArray weakObjectsPointerArray];

NSObject *obj = [[NSObject alloc] init];
[weakArray addPointer:(__bridge void *)obj];

NSLog(@"count: %lu", weakArray.count); // 1

obj = nil; // obj deallocates
[weakArray compact]; // remove nil pointers
NSLog(@"count: %lu", weakArray.count); // 0
```

---

### Q54: Objective-C Exception Handling

```objc
@try {
    NSArray *arr = @[@1, @2];
    id obj = arr[10]; // NSRangeException
} @catch (NSException *exception) {
    NSLog(@"Exception: %@", exception.name);
    NSLog(@"Reason: %@", exception.reason);
    NSLog(@"UserInfo: %@", exception.userInfo);
} @catch (NSException *specific) {
    // catch specific type
} @finally {
    // Always runs
    [self cleanup];
}

// Throwing
- (void)validateAge:(NSInteger)age {
    if (age < 0) {
        [NSException raise:NSInvalidArgumentException
                    format:@"Age cannot be negative: %ld", (long)age];
    }
}

// NOTE: ObjC exceptions ไม่ unwind ARC properly
// ใช้ NSError pattern แทนสำหรับ expected errors
```

---

### Q55: Message Forwarding Deep Dive

```objc
// Complete message forwarding chain
@implementation ForwardingProxy

// Step 1: resolveInstanceMethod - dynamic method resolution
+ (BOOL)resolveInstanceMethod:(SEL)sel {
    // สามารถ add method ตอน runtime
    if (sel == @selector(dynamicMethod)) {
        class_addMethod(self, sel, (IMP)dynamicMethodIMP, "v@:");
        return YES;
    }
    return NO;
}

void dynamicMethodIMP(id self, SEL _cmd) {
    NSLog(@"Dynamic method called");
}

// Step 2: forwardingTargetForSelector - fast path
- (id)forwardingTargetForSelector:(SEL)aSelector {
    if ([self.realObject respondsToSelector:aSelector]) {
        return self.realObject; // redirect
    }
    return nil;
}

// Step 3: full forwarding
- (NSMethodSignature *)methodSignatureForSelector:(SEL)aSelector {
    return [self.realObject methodSignatureForSelector:aSelector];
}

- (void)forwardInvocation:(NSInvocation *)anInvocation {
    if ([self.realObject respondsToSelector:anInvocation.selector]) {
        [anInvocation invokeWithTarget:self.realObject];
    } else {
        [super forwardInvocation:anInvocation]; // -> doesNotRecognizeSelector
    }
}

@end
```

---

### Q56-60: Network และ Security

**Q56: URLSession best practices**
```objc
// Background URLSession
NSURLSessionConfiguration *config = 
    [NSURLSessionConfiguration backgroundSessionConfigurationWithIdentifier:
     @"com.app.background"];
config.timeoutIntervalForRequest = 30;
config.timeoutIntervalForResource = 300;
config.waitsForConnectivity = YES;

NSURLSession *session = [NSURLSession sessionWithConfiguration:config
                                                      delegate:self
                                                 delegateQueue:nil];

// Certificate pinning
- (void)URLSession:(NSURLSession *)session 
    didReceiveChallenge:(NSURLAuthenticationChallenge *)challenge
     completionHandler:(void (^)(NSURLSessionAuthChallengeDisposition, 
                                 NSURLCredential *))completionHandler {
    
    if ([challenge.protectionSpace.authenticationMethod 
         isEqualToString:NSURLAuthenticationMethodServerTrust]) {
        
        SecTrustRef serverTrust = challenge.protectionSpace.serverTrust;
        
        // Pin certificate
        SecCertificateRef certificate = SecTrustGetCertificateAtIndex(serverTrust, 0);
        NSData *serverCertData = (__bridge_transfer NSData *)
            SecCertificateCopyData(certificate);
        NSData *localCertData = [NSData dataWithContentsOfFile:
            [[NSBundle mainBundle] pathForResource:@"cert" ofType:@"der"]];
        
        if ([serverCertData isEqualToData:localCertData]) {
            completionHandler(NSURLSessionAuthChallengeUseCredential,
                            [NSURLCredential credentialForTrust:serverTrust]);
        } else {
            completionHandler(NSURLSessionAuthChallengeCancelAuthenticationChallenge, nil);
        }
    }
}
```

**Q57: Keychain usage**
```objc
// Store in Keychain
- (BOOL)storeToken:(NSString *)token forService:(NSString *)service {
    NSData *tokenData = [token dataUsingEncoding:NSUTF8StringEncoding];
    
    NSDictionary *query = @{
        (__bridge id)kSecClass: (__bridge id)kSecClassGenericPassword,
        (__bridge id)kSecAttrService: service,
        (__bridge id)kSecValueData: tokenData,
        (__bridge id)kSecAttrAccessible: 
            (__bridge id)kSecAttrAccessibleWhenUnlockedThisDeviceOnly
    };
    
    SecItemDelete((__bridge CFDictionaryRef)query); // Remove existing
    OSStatus status = SecItemAdd((__bridge CFDictionaryRef)query, nil);
    
    return status == errSecSuccess;
}

// Retrieve from Keychain
- (NSString *)tokenForService:(NSString *)service {
    NSDictionary *query = @{
        (__bridge id)kSecClass: (__bridge id)kSecClassGenericPassword,
        (__bridge id)kSecAttrService: service,
        (__bridge id)kSecReturnData: @YES,
        (__bridge id)kSecMatchLimit: (__bridge id)kSecMatchLimitOne
    };
    
    CFDataRef result = nil;
    OSStatus status = SecItemCopyMatching((__bridge CFDictionaryRef)query, 
                                         (CFTypeRef *)&result);
    
    if (status == errSecSuccess && result) {
        NSData *data = (__bridge_transfer NSData *)result;
        return [[NSString alloc] initWithData:data encoding:NSUTF8StringEncoding];
    }
    
    return nil;
}
```

---

### Q61-70: Performance Optimization

**Q61: Instruments profiling**
- Time Profiler: ค้นหา slow code
- Allocations: memory usage, leaks
- Leaks: memory leaks
- Core Data: fetch performance
- Network: URLSession activity

**Q62: Common performance issues**
```objc
// BAD: Layout ใน UITableViewCell
- (UITableViewCell *)tableView:(UITableView *)tv cellForRowAtIndexPath:(NSIndexPath *)ip {
    UITableViewCell *cell = [tv dequeueReusableCellWithIdentifier:@"Cell"];
    
    // BAD: ทำ heavy work บน main thread
    NSData *imageData = [NSData dataWithContentsOfURL:imageURL]; // blocking!
    cell.imageView.image = [UIImage imageWithData:imageData];
    
    return cell;
}

// GOOD: Async image loading
- (UITableViewCell *)tableView:(UITableView *)tv cellForRowAtIndexPath:(NSIndexPath *)ip {
    MyCell *cell = [tv dequeueReusableCellWithIdentifier:@"Cell" forIndexPath:ip];
    
    Model *model = self.data[ip.row];
    cell.titleLabel.text = model.title;
    cell.tag = ip.row; // prevent stale update
    
    dispatch_async(dispatch_get_global_queue(0, 0), ^{
        UIImage *image = [UIImage imageWithData:[NSData dataWithContentsOfURL:model.imageURL]];
        dispatch_async(dispatch_get_main_queue(), ^{
            if (cell.tag == ip.row) { // still valid cell?
                cell.thumbnailImageView.image = image;
            }
        });
    });
    
    return cell;
}
```

**Q63: View rendering optimization**
```objc
// Rasterize layer สำหรับ complex static views
view.layer.shouldRasterize = YES;
view.layer.rasterizationScale = [UIScreen mainScreen].scale;

// Opaque views เร็วกว่า transparent
view.opaque = YES;
view.backgroundColor = [UIColor whiteColor]; // ไม่ใช่ clearColor

// Avoid offscreen rendering
view.layer.cornerRadius = 5;
view.layer.masksToBounds = YES; // กระตุ้น offscreen rendering!
// แทนที่ด้วย maskedCorners หรือ CAShapeLayer
```

---

### Q71-80: Testing

**Q71: Unit Testing ใน Objective-C**
```objc
#import <XCTest/XCTest.h>
#import "Calculator.h"

@interface CalculatorTests : XCTestCase

@property (nonatomic, strong) Calculator *calculator;

@end

@implementation CalculatorTests

- (void)setUp {
    [super setUp];
    self.calculator = [[Calculator alloc] init];
}

- (void)tearDown {
    self.calculator = nil;
    [super tearDown];
}

- (void)testAddition {
    NSInteger result = [self.calculator add:2 to:3];
    XCTAssertEqual(result, 5, @"2 + 3 should equal 5");
}

- (void)testDivisionByZero {
    XCTAssertThrows([self.calculator divide:10 by:0], @"Should throw on divide by zero");
}

- (void)testAsyncOperation {
    XCTestExpectation *expectation = [self expectationWithDescription:@"Fetch data"];
    
    [self.calculator fetchResultWithCompletion:^(NSInteger result, NSError *error) {
        XCTAssertNil(error);
        XCTAssertEqual(result, 42);
        [expectation fulfill];
    }];
    
    [self waitForExpectationsWithTimeout:5.0 handler:nil];
}

@end
```

**Q72: Mock objects**
```objc
// Protocol-based mocking
@protocol NetworkServiceProtocol <NSObject>
- (void)fetchDataWithCompletion:(void (^)(NSData *, NSError *))completion;
@end

@interface MockNetworkService : NSObject <NetworkServiceProtocol>
@property (nonatomic, strong) NSData *mockedData;
@property (nonatomic, strong) NSError *mockedError;
@end

@implementation MockNetworkService

- (void)fetchDataWithCompletion:(void (^)(NSData *, NSError *))completion {
    if (completion) {
        completion(self.mockedData, self.mockedError);
    }
}

@end

// ใน test
- (void)testWithMock {
    MockNetworkService *mock = [[MockNetworkService alloc] init];
    mock.mockedData = [@"test data" dataUsingEncoding:NSUTF8StringEncoding];
    
    MyViewModel *vm = [[MyViewModel alloc] initWithNetworkService:mock];
    [vm loadData];
    
    XCTAssertEqualObjects(vm.displayText, @"test data");
}
```

---

### Q81-90: Design Patterns

**Q81: Observer Pattern**
```objc
// Notification-based observer
@implementation EventBus

+ (void)subscribe:(id)observer 
          toEvent:(NSString *)event 
         callback:(void (^)(NSDictionary *))callback {
    
    [[NSNotificationCenter defaultCenter] 
        addObserverForName:event
                   object:nil
                    queue:[NSOperationQueue mainQueue]
               usingBlock:^(NSNotification *note) {
        if (callback) callback(note.userInfo);
    }];
}

+ (void)publish:(NSString *)event userInfo:(NSDictionary *)info {
    [[NSNotificationCenter defaultCenter] 
        postNotificationName:event
                      object:nil
                    userInfo:info];
}

@end
```

**Q82: Strategy Pattern**
```objc
@protocol SortStrategy <NSObject>
- (NSArray *)sortArray:(NSArray *)array;
@end

@interface BubbleSortStrategy : NSObject <SortStrategy>
@end

@interface QuickSortStrategy : NSObject <SortStrategy>
@end

@interface Sorter : NSObject
@property (nonatomic, strong) id<SortStrategy> strategy;
- (NSArray *)sort:(NSArray *)array;
@end

@implementation Sorter
- (NSArray *)sort:(NSArray *)array {
    return [self.strategy sortArray:array];
}
@end
```

**Q83: Builder Pattern**
```objc
@interface AlertBuilder : NSObject

- (AlertBuilder *)withTitle:(NSString *)title;
- (AlertBuilder *)withMessage:(NSString *)message;
- (AlertBuilder *)addAction:(NSString *)title style:(UIAlertActionStyle)style handler:(void (^)(void))handler;
- (UIAlertController *)build;

@end

@implementation AlertBuilder {
    NSString *_title, *_message;
    NSMutableArray *_actions;
}

- (instancetype)init {
    self = [super init];
    if (self) { _actions = [NSMutableArray array]; }
    return self;
}

- (AlertBuilder *)withTitle:(NSString *)title {
    _title = title;
    return self;
}

- (AlertBuilder *)withMessage:(NSString *)message {
    _message = message;
    return self;
}

- (AlertBuilder *)addAction:(NSString *)title style:(UIAlertActionStyle)style handler:(void(^)(void))handler {
    UIAlertAction *action = [UIAlertAction actionWithTitle:title 
                                                     style:style 
                                                   handler:^(UIAlertAction *a) { 
        if (handler) handler(); 
    }];
    [_actions addObject:action];
    return self;
}

- (UIAlertController *)build {
    UIAlertController *alert = [UIAlertController alertControllerWithTitle:_title
                                                                   message:_message
                                                            preferredStyle:UIAlertControllerStyleAlert];
    for (UIAlertAction *action in _actions) {
        [alert addAction:action];
    }
    return alert;
}

@end

// Usage
UIAlertController *alert = [[[[[AlertBuilder new]
    withTitle:@"ยืนยัน"]
    withMessage:@"ต้องการลบข้อมูลนี้หรือไม่?"]
    addAction:@"ลบ" style:UIAlertActionStyleDestructive handler:^{
        [self deleteItem];
    }]
    addAction:@"ยกเลิก" style:UIAlertActionStyleCancel handler:nil]
    build];
```

---

### Q91-100: Final Questions

**Q91: Instruments - Detecting Memory Leaks**
1. Product -> Profile -> Leaks
2. ดู Allocation graph
3. ค้นหา objects ที่ยังอยู่หลัง view controller dismiss
4. ตรวจสอบ retain cycle ด้วย Memory Graph Debugger

**Q92: App Store Submission Requirements**
- Privacy policy
- App Transport Security (ATS)
- 64-bit support
- No private API usage
- Screenshots สำหรับทุก device size
- Privacy usage descriptions (NSCameraUsageDescription, etc.)

**Q93: Code Review Checklist**
```
✓ No retain cycles
✓ Thread-safe for shared resources
✓ Error handling
✓ Unit tests
✓ No force unwrapping (id)
✓ Consistent naming conventions
✓ No magic numbers
✓ Commented complex logic
✓ Memory efficient
✓ No deprecated APIs
```

**Q94: Debugging Tools**
```objc
// LLDB commands
// po object          - print object description
// p variable         - print value
// bt                 - backtrace
// frame variable     - local variables
// expr (expression)  - evaluate expression

// Runtime debugging
extern void _objc_autoreleasePoolPrint(void);
_objc_autoreleasePoolPrint(); // show autorelease pool contents

// Zombie objects detection
// Enable NSZombieEnabled in scheme
// Detects message to deallocated objects
```

**Q95: Continuous Integration**
```yaml
# Fastlane example
lane :test do
  scan(
    scheme: "MyApp",
    devices: ["iPhone 15"],
    clean: true,
    code_coverage: true
  )
end

lane :deploy do
  increment_build_number
  gym(
    scheme: "MyApp",
    configuration: "Release",
    export_method: "app-store"
  )
  deliver(
    submit_for_review: false,
    automatic_release: false
  )
end
```

**Q96: Swift Interoperability**
```objc
// ObjC class ที่ Swift ใช้ได้
NS_SWIFT_NAME(Person) // rename for Swift
@interface ObjCPerson : NSObject

@property (nonatomic, strong, nonnull) NSString *name;
@property (nonatomic, assign) NSInteger age;

NS_SWIFT_NAME(init(name:age:))
- (instancetype)initWithName:(nonnull NSString *)name 
                         age:(NSInteger)age NS_DESIGNATED_INITIALIZER;

// Enumeration สำหรับ Swift
typedef NS_ENUM(NSInteger, Direction) {
    DirectionNorth NS_SWIFT_NAME(north),
    DirectionSouth NS_SWIFT_NAME(south),
    DirectionEast NS_SWIFT_NAME(east),
    DirectionWest NS_SWIFT_NAME(west)
};

// Option set
typedef NS_OPTIONS(NSUInteger, Permission) {
    PermissionNone   = 0,
    PermissionRead   = 1 << 0,
    PermissionWrite  = 1 << 1,
    PermissionDelete = 1 << 2
};

@end
```

**Q97: Responsive UI Best Practices**
```objc
// เสมอ update UI บน main thread
dispatch_async(dispatch_get_main_queue(), ^{
    self.label.text = result;
    [self.tableView reloadData];
    [self.activityIndicator stopAnimating];
});

// Background tasks
- (void)loadData {
    [self.activityIndicator startAnimating];
    
    dispatch_async(dispatch_get_global_queue(QOS_CLASS_USER_INITIATED, 0), ^{
        NSData *data = [self fetchFromNetwork]; // blocking
        NSArray *parsed = [self parseData:data];
        
        dispatch_async(dispatch_get_main_queue(), ^{
            self.items = parsed;
            [self.tableView reloadData];
            [self.activityIndicator stopAnimating];
        });
    });
}
```

**Q98: Memory Profiling**
```objc
// ดู memory usage
- (void)logMemoryUsage {
    struct task_basic_info info;
    mach_msg_type_number_t size = sizeof(info);
    kern_return_t kerr = task_info(mach_task_self(), 
                                    TASK_BASIC_INFO, 
                                    (task_info_t)&info, 
                                    &size);
    if (kerr == KERN_SUCCESS) {
        NSLog(@"Memory in use: %lu MB", 
              (unsigned long)info.resident_size / 1024 / 1024);
    }
}
```

**Q99: Auto Layout Programmatic**
```objc
// NSLayoutConstraint
- (void)setupConstraints {
    UIView *subview = [[UIView alloc] init];
    subview.translatesAutoresizingMaskIntoConstraints = NO;
    [self.view addSubview:subview];
    
    [NSLayoutConstraint activateConstraints:@[
        [subview.topAnchor constraintEqualToAnchor:self.view.safeAreaLayoutGuide.topAnchor constant:16],
        [subview.leadingAnchor constraintEqualToAnchor:self.view.leadingAnchor constant:16],
        [subview.trailingAnchor constraintEqualToAnchor:self.view.trailingAnchor constant:-16],
        [subview.heightAnchor constraintEqualToConstant:200]
    ]];
}
```

**Q100: Final Tips for iOS Interview**

1. **ทบทวน fundamentals**: Memory management, ARC, blocks, protocols
2. **รู้จัก Apple frameworks**: UIKit, Foundation, Core Data, GCD
3. **Data structures & Algorithms**: Array, Dictionary, LinkedList, Tree, Graph, Sorting, Searching
4. **Design patterns**: MVC/MVVM, Singleton, Observer, Delegate, Factory
5. **Concurrency**: GCD, NSOperation, Thread safety
6. **Testing**: XCTest, TDD, Mocking
7. **App lifecycle**: Background fetch, push notifications, deep linking
8. **Performance**: Profiling, optimization, lazy loading
9. **Security**: Keychain, certificate pinning, obfuscation
10. **Swift knowledge**: Interop, migration strategies
11. **WWDC sessions**: ติดตาม updates
12. **Side projects**: แสดงใน portfolio
13. **Communication**: อธิบาย tradeoffs ได้
14. **Ask questions**: ถามเกี่ยวกับ tech stack, team

---

## Tips การสัมภาษณ์

### การตอบคำถาม Technical
1. **STAR Method**: Situation, Task, Action, Result
2. **Think aloud**: อธิบายกระบวนการคิด
3. **Ask clarifying questions**: ถามก่อนเขียน code
4. **Consider edge cases**: nil, empty, large inputs
5. **Time/Space complexity**: อธิบาย Big-O

### Whiteboard Coding
```
1. Understand the problem
2. Work through examples
3. Identify edge cases
4. Design algorithm  
5. Write code step by step
6. Test with examples
7. Optimize if needed
```

### การเตรียมตัว
- **LeetCode**: practice easy-medium problems
- **System design**: ฝึก design large systems
- **อ่าน Apple docs**: รู้จัก APIs ที่ใช้บ่อย
- **สร้าง sample projects**: แสดงใน GitHub
- **บล็อก/community**: เข้าร่วม iOS community

---

## สรุป

การสัมภาษณ์งาน iOS Developer ต้องการทั้ง:
- **Technical depth**: รู้ลึกใน Objective-C/Swift, iOS frameworks
- **Problem solving**: แก้ปัญหาเชิงอัลกอริทึม
- **System design**: ออกแบบระบบขนาดใหญ่
- **Soft skills**: สื่อสาร, teamwork, learning mindset

จงฝึกฝนสม่ำเสมอ ทบทวน fundamentals และสร้าง projects จริงๆ เพื่อแสดงความสามารถ

---

*จบตอนที่ 97 - การเตรียมตัวสัมภาษณ์งาน Objective-C และ iOS*
