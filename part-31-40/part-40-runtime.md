# Part 40: Objective-C Runtime

## บทนำ

Objective-C Runtime เป็นหนึ่งในส่วนที่ทรงพลังและน่าสนใจที่สุดของภาษา Objective-C มันคือระบบที่ทำงานอยู่เบื้องหลังทุกการทำงานของ Objective-C โดยเฉพาะ message passing, dynamic dispatch, และ introspection

ใน C++ เมื่อคุณเรียก method จะเป็น compile-time binding แต่ใน Objective-C การส่ง message จะถูก resolve ที่ **runtime** ทำให้สามารถทำสิ่งที่น่าอัศจรรย์หลายอย่าง

ในบทนี้เราจะเรียนรู้:
- โครงสร้างของ Objective-C Runtime
- Class และ Method inspection
- Method Swizzling
- Dynamic method resolution
- Message forwarding
- Associated objects
- NSInvocation
- Use cases จริงและ best practices

---

## 40.1 What is the Objective-C Runtime?

### ภาพรวม

Runtime คือ C library ที่ Apple จัดหาให้ (`/usr/lib/libobjc.A.dylib`) ซึ่งทำหน้าที่:

1. **Message Dispatch**: แปล `[obj method]` เป็น `objc_msgSend(obj, @selector(method))`
2. **Dynamic Typing**: ทำให้ `id` ทำงานได้
3. **Introspection**: ตรวจสอบ class, method, property ของ object
4. **Dynamic Features**: Method swizzling, associated objects, ฯลฯ

```
[person greet]
     ↓ (compiler แปลง)
objc_msgSend(person, @selector(greet))
     ↓ (runtime ค้นหา)
Method table ของ Person class
     ↓ (เจอแล้ว)
ดำเนินการ IMP (implementation pointer)
```

### การ Import

```objc
#import <objc/runtime.h>
#import <objc/message.h>
```

### โครงสร้างหลัก

```c
// Class structure (simplified)
typedef struct objc_class {
    Class isa;                  // ชี้ไป metaclass
    Class superclass;           // ชี้ไป superclass
    cache_t cache;              // method cache
    class_data_bits_t bits;     // class data
} *Class;

// Object structure (simplified)
typedef struct objc_object {
    Class isa;    // ชี้ไป class ของ object
} *id;

// Method structure
typedef struct method_t {
    SEL name;       // method name (selector)
    const char *types;  // type encoding
    IMP imp;        // pointer to implementation
} *Method;
```

---

## 40.2 Class Inspection

### ข้อมูล Class พื้นฐาน

```objc
#import <objc/runtime.h>

// ชื่อ class
const char *className = class_getName([NSString class]);
NSLog(@"Class name: %s", className);  // NSString

// Superclass
Class superclass = class_getSuperclass([NSString class]);
NSLog(@"Superclass: %s", class_getName(superclass));  // NSObject

// ตรวจสอบว่าเป็น meta-class หรือไม่
BOOL isMeta = class_isMetaClass([NSString class]);
NSLog(@"Is meta: %@", isMeta ? @"YES" : @"NO");  // NO

// Meta-class
Class metaClass = object_getClass([NSString class]);
NSLog(@"Meta class: %s", class_getName(metaClass));

// Instance size
size_t instanceSize = class_getInstanceSize([NSString class]);
NSLog(@"Instance size: %zu bytes", instanceSize);

// Version
int version = class_getVersion([NSString class]);
NSLog(@"Version: %d", version);
```

### ดู Instance Variables (Ivars)

```objc
@interface MyClass : NSObject {
    NSInteger _count;
    NSString *_name;
}
@end

// ดู ivars ทั้งหมด
unsigned int ivarCount = 0;
Ivar *ivars = class_copyIvarList([MyClass class], &ivarCount);

for (unsigned int i = 0; i < ivarCount; i++) {
    Ivar ivar = ivars[i];
    const char *name = ivar_getName(ivar);
    const char *type = ivar_getTypeEncoding(ivar);
    ptrdiff_t offset = ivar_getOffset(ivar);
    
    NSLog(@"Ivar: %s, type: %s, offset: %td", name, type, offset);
}

free(ivars);  // ต้อง free เสมอ!

// ดู specific ivar
Ivar countIvar = class_getInstanceVariable([MyClass class], "_count");
if (countIvar) {
    NSLog(@"Found _count ivar at offset: %td", ivar_getOffset(countIvar));
}
```

### ดู Properties

```objc
@interface Person : NSObject
@property (nonatomic, copy) NSString *name;
@property (nonatomic, assign) NSInteger age;
@property (nonatomic, strong) NSDate *birthDate;
@end

unsigned int propCount = 0;
objc_property_t *properties = class_copyPropertyList([Person class], &propCount);

for (unsigned int i = 0; i < propCount; i++) {
    objc_property_t prop = properties[i];
    const char *name = property_getName(prop);
    const char *attributes = property_getAttributes(prop);
    
    NSLog(@"Property: %s, attributes: %s", name, attributes);
    // attributes format: T@"NSString",C,N,V_name
    // T = type, C = copy, N = nonatomic, V = ivar name
}

free(properties);
```

### ดู Methods

```objc
unsigned int methodCount = 0;
Method *methods = class_copyMethodList([NSString class], &methodCount);

NSLog(@"NSString methods count: %d", methodCount);

for (unsigned int i = 0; i < methodCount; i++) {
    Method method = methods[i];
    SEL selector = method_getName(method);
    const char *types = method_getTypeEncoding(method);
    
    NSLog(@"Method: %@, types: %s",
          NSStringFromSelector(selector),
          types);
}

free(methods);

// ดู specific method
Method descMethod = class_getInstanceMethod([NSString class], @selector(description));
if (descMethod) {
    NSLog(@"description method exists!");
    unsigned int argCount = method_getNumberOfArguments(descMethod);
    NSLog(@"Arguments: %d", argCount);
}
```

### ดู Protocols

```objc
unsigned int protocolCount = 0;
__unsafe_unretained Protocol **protocols = class_copyProtocolList([NSString class], &protocolCount);

for (unsigned int i = 0; i < protocolCount; i++) {
    Protocol *protocol = protocols[i];
    NSLog(@"Protocol: %s", protocol_getName(protocol));
}

free(protocols);

// ตรวจสอบว่า conform to protocol หรือไม่
BOOL conforms = class_conformsToProtocol([NSArray class], @protocol(NSFastEnumeration));
NSLog(@"NSArray conforms to NSFastEnumeration: %@", conforms ? @"YES" : @"NO");
```

---

## 40.3 Method Inspection

### เข้าถึง Method

```objc
// Instance method
Method method = class_getInstanceMethod([NSString class], @selector(length));
if (method) {
    SEL name = method_getName(method);
    IMP imp = method_getImplementation(method);
    const char *types = method_getTypeEncoding(method);
    
    NSLog(@"Method name: %@", NSStringFromSelector(name));
    NSLog(@"Types: %s", types);
    NSLog(@"IMP: %p", imp);
}

// Class method (+ method)
Method classMethod = class_getClassMethod([NSString class], @selector(string));
if (classMethod) {
    NSLog(@"Found class method +string");
}

// Superclass method
Method superMethod = class_getInstanceMethod([NSMutableString class], @selector(length));
NSLog(@"Same implementation? %@",
      method_getImplementation(method) == method_getImplementation(superMethod) ? @"YES" : @"NO");
```

### Type Encoding

Type encoding เป็น string ที่บอก types ของ arguments และ return value:

```objc
// Common type encodings:
// @ = id (object)
// # = Class
// : = SEL
// c = char, i = int, s = short, l = long, q = long long
// C = unsigned char, I = unsigned int, L = unsigned long, Q = unsigned long long
// f = float, d = double
// B = C++ bool or C99 _Bool
// v = void
// * = char* (C string)
// ^ = pointer to type (e.g., ^i = int*)
// ? = function pointer
// { } = struct

// ตัวอย่าง
Method lengthMethod = class_getInstanceMethod([NSString class], @selector(length));
const char *types = method_getTypeEncoding(lengthMethod);
// "Q16@0:8" = unsigned long long (Q), 16 bytes total, id(@) at offset 0, SEL(:) at offset 8

NSMethodSignature *sig = [NSMethodSignature signatureWithObjCTypes:types];
NSLog(@"Return type: %s", [sig methodReturnType]);
NSLog(@"Number of arguments: %lu", (unsigned long)sig.numberOfArguments);
```

---

## 40.4 Method Swizzling

Method Swizzling คือการสลับ implementation ของสอง methods ณ runtime

### แนวคิด

```
ก่อน Swizzle:           หลัง Swizzle:
method_A → IMP_A         method_A → IMP_B
method_B → IMP_B         method_B → IMP_A

ดังนั้นเมื่อ [obj method_A] → ทำงาน IMP_B
```

### การ Swizzle พื้นฐาน

```objc
#import <objc/runtime.h>

@implementation UIViewController (Tracking)

+ (void)load {
    // +load เรียกครั้งเดียวตอน load class ก่อน main()
    // เหมาะสำหรับ swizzling
    static dispatch_once_t onceToken;
    dispatch_once(&onceToken, ^{
        [self swizzleViewDidAppear];
    });
}

+ (void)swizzleViewDidAppear {
    Class class = [self class];
    
    SEL originalSelector = @selector(viewDidAppear:);
    SEL swizzledSelector = @selector(tracking_viewDidAppear:);
    
    Method originalMethod = class_getInstanceMethod(class, originalSelector);
    Method swizzledMethod = class_getInstanceMethod(class, swizzledSelector);
    
    if (!originalMethod || !swizzledMethod) {
        NSLog(@"Swizzle failed: method not found");
        return;
    }
    
    // ลองเพิ่ม method ก่อน (กรณี method ถูก override ใน subclass)
    BOOL didAddMethod = class_addMethod(class,
                                        originalSelector,
                                        method_getImplementation(swizzledMethod),
                                        method_getTypeEncoding(swizzledMethod));
    
    if (didAddMethod) {
        // เพิ่มสำเร็จ แสดงว่า original method มาจาก superclass
        class_replaceMethod(class,
                            swizzledSelector,
                            method_getImplementation(originalMethod),
                            method_getTypeEncoding(originalMethod));
    } else {
        // ไม่ได้เพิ่ม แสดงว่ามีอยู่แล้ว ให้ swap
        method_exchangeImplementations(originalMethod, swizzledMethod);
    }
    
    NSLog(@"Swizzled viewDidAppear:!");
}

- (void)tracking_viewDidAppear:(BOOL)animated {
    // เรียก original implementation (ตอนนี้ tracking_ ชี้ไป original)
    [self tracking_viewDidAppear:animated];
    
    // เพิ่ม custom behavior
    NSLog(@"[Analytics] ViewController appeared: %@", NSStringFromClass([self class]));
}

@end
```

### Swizzle ตัวอย่างจริง: NSDate Logging

```objc
@implementation NSDate (DebugLogging)

+ (void)load {
    static dispatch_once_t onceToken;
    dispatch_once(&onceToken, ^{
        Method original = class_getClassMethod([self class], @selector(date));
        Method swizzled = class_getClassMethod([self class], @selector(debug_date));
        method_exchangeImplementations(original, swizzled);
    });
}

+ (instancetype)debug_date {
    NSDate *date = [self debug_date];  // เรียก original (เพราะ swap แล้ว)
    NSLog(@"[DEBUG] NSDate created: %@", date);
    return date;
}

@end
```

### Swizzle ที่ปลอดภัยกว่า: Helper Function

```objc
// SwizzleHelper.h
@interface SwizzleHelper : NSObject

+ (BOOL)swizzleInstanceMethod:(SEL)original
                      inClass:(Class)cls
                   withMethod:(SEL)swizzled;

+ (BOOL)swizzleClassMethod:(SEL)original
                   inClass:(Class)cls
                withMethod:(SEL)swizzled;

@end

@implementation SwizzleHelper

+ (BOOL)swizzleInstanceMethod:(SEL)original
                      inClass:(Class)cls
                   withMethod:(SEL)swizzled {
    Method originalMethod = class_getInstanceMethod(cls, original);
    Method swizzledMethod = class_getInstanceMethod(cls, swizzled);
    
    if (!originalMethod || !swizzledMethod) {
        NSLog(@"[Swizzle] Failed: method not found in %s", class_getName(cls));
        return NO;
    }
    
    BOOL added = class_addMethod(cls,
                                  original,
                                  method_getImplementation(swizzledMethod),
                                  method_getTypeEncoding(swizzledMethod));
    
    if (added) {
        class_replaceMethod(cls,
                            swizzled,
                            method_getImplementation(originalMethod),
                            method_getTypeEncoding(originalMethod));
    } else {
        method_exchangeImplementations(originalMethod, swizzledMethod);
    }
    
    return YES;
}

+ (BOOL)swizzleClassMethod:(SEL)original
                   inClass:(Class)cls
                withMethod:(SEL)swizzled {
    Class metaClass = object_getClass(cls);
    return [self swizzleInstanceMethod:original inClass:metaClass withMethod:swizzled];
}

@end
```

---

## 40.5 Dynamic Method Resolution

### resolveInstanceMethod:

เมื่อ message ส่งไปยัง object และไม่พบ method จะเรียก `resolveInstanceMethod:` ก่อน:

```objc
@interface DynamicClass : NSObject
@end

@implementation DynamicClass

// เรียกเมื่อไม่พบ instance method
+ (BOOL)resolveInstanceMethod:(SEL)sel {
    NSString *selectorName = NSStringFromSelector(sel);
    
    // Handle getters แบบ dynamic
    if ([selectorName hasPrefix:@"get"]) {
        NSLog(@"Dynamically resolving getter: %@", selectorName);
        
        // เพิ่ม method ใหม่
        class_addMethod(self,
                        sel,
                        (IMP)dynamicGetterImplementation,
                        "@@:");
        return YES;
    }
    
    return [super resolveInstanceMethod:sel];
}

// C function ที่ทำหน้าที่เป็น dynamic getter
static id dynamicGetterImplementation(id self, SEL _cmd) {
    NSLog(@"Dynamic getter called: %@", NSStringFromSelector(_cmd));
    return [NSString stringWithFormat:@"Value for %@", NSStringFromSelector(_cmd)];
}

// เรียกเมื่อไม่พบ class method
+ (BOOL)resolveClassMethod:(SEL)sel {
    NSLog(@"Cannot resolve class method: %@", NSStringFromSelector(sel));
    return [super resolveClassMethod:sel];
}

@end

// การใช้งาน
DynamicClass *obj = [[DynamicClass alloc] init];

// #pragma clang diagnostic push
// #pragma clang diagnostic ignored "-Wundeclared-selector"
id result = [obj performSelector:@selector(getName)];
NSLog(@"Result: %@", result);
// Output: Dynamic getter called: getName
//         Result: Value for getName
```

### ตัวอย่างจริง: Dynamic Accessor

```objc
@interface KeyValueStore : NSObject

@property (nonatomic, strong) NSMutableDictionary *store;

@end

@implementation KeyValueStore

- (instancetype)init {
    self = [super init];
    if (self) {
        _store = [NSMutableDictionary dictionary];
    }
    return self;
}

+ (BOOL)resolveInstanceMethod:(SEL)sel {
    NSString *name = NSStringFromSelector(sel);
    
    // Handle setters: setName:
    if ([name hasPrefix:@"set"] && [name hasSuffix:@":"] && name.length > 4) {
        NSString *key = [name substringWithRange:NSMakeRange(3, name.length - 4)];
        key = [NSString stringWithFormat:@"%@%@",
               [[key substringToIndex:1] lowercaseString],
               [key substringFromIndex:1]];
        
        class_addMethod(self, sel, imp_implementationWithBlock(^(id self, id value) {
            KeyValueStore *store = (KeyValueStore *)self;
            store.store[key] = value ?: [NSNull null];
        }), "v@:@");
        
        return YES;
    }
    
    // Handle getters: name
    if (![name hasPrefix:@"set"] && ![name containsString:@":"]) {
        class_addMethod(self, sel, imp_implementationWithBlock(^id(id self) {
            KeyValueStore *store = (KeyValueStore *)self;
            id value = store.store[name];
            return [value isKindOfClass:[NSNull class]] ? nil : value;
        }), "@@:");
        
        return YES;
    }
    
    return [super resolveInstanceMethod:sel];
}

@end

// การใช้งาน
KeyValueStore *kv = [[KeyValueStore alloc] init];
[kv setValue:@"Alice" forKey:@"name"];
[kv setValue:@(30) forKey:@"age"];

id name = [kv valueForKey:@"name"];
id age = [kv valueForKey:@"age"];
NSLog(@"Name: %@, Age: %@", name, age);
```

---

## 40.6 Message Forwarding

เมื่อ `resolveInstanceMethod:` return NO, runtime จะเรียก message forwarding chain:

```
1. resolveInstanceMethod:     → YES? เพิ่ม method แล้วลองใหม่
2. forwardingTargetForSelector: → return proxy object (fast forwarding)
3. methodSignatureForSelector: + forwardInvocation: (full forwarding)
4. doesNotRecognizeSelector: → NSException!
```

### forwardingTargetForSelector: (Fast Forwarding)

```objc
@interface Proxy : NSObject
@property (nonatomic, strong) id target;
@end

@implementation Proxy

- (id)forwardingTargetForSelector:(SEL)aSelector {
    // ส่งต่อไป target ถ้าเป็น method ที่ target มี
    if ([self.target respondsToSelector:aSelector]) {
        NSLog(@"Forwarding %@ to target", NSStringFromSelector(aSelector));
        return self.target;
    }
    return [super forwardingTargetForSelector:aSelector];
}

@end

// การใช้งาน
Proxy *proxy = [[Proxy alloc] init];
proxy.target = @"Hello, World!";  // NSString

// proxy ไม่มี method length แต่จะ forward ไปยัง NSString
NSUInteger length = [proxy performSelector:@selector(length)];
NSLog(@"Length via proxy: %lu", (unsigned long)length);
```

### forwardInvocation: (Full Forwarding)

```objc
@interface MessageLogger : NSObject
@property (nonatomic, strong) id realObject;
@end

@implementation MessageLogger

- (NSMethodSignature *)methodSignatureForSelector:(SEL)aSelector {
    // คืน signature ของ realObject
    NSMethodSignature *sig = [self.realObject methodSignatureForSelector:aSelector];
    if (sig) return sig;
    return [super methodSignatureForSelector:aSelector];
}

- (void)forwardInvocation:(NSInvocation *)invocation {
    if ([self.realObject respondsToSelector:invocation.selector]) {
        // Log the call
        NSLog(@"[MessageLogger] Intercepting: %@",
              NSStringFromSelector(invocation.selector));
        
        // Forward to real object
        [invocation invokeWithTarget:self.realObject];
        
        // Log return value
        NSLog(@"[MessageLogger] Call completed");
    } else {
        [super forwardInvocation:invocation];
    }
}

- (BOOL)respondsToSelector:(SEL)aSelector {
    return [self.realObject respondsToSelector:aSelector] ||
           [super respondsToSelector:aSelector];
}

@end

// การใช้งาน
MessageLogger *logger = [[MessageLogger alloc] init];
logger.realObject = [[NSMutableArray alloc] init];

[logger performSelector:@selector(addObject:) withObject:@"Item 1"];
[logger performSelector:@selector(addObject:) withObject:@"Item 2"];
NSUInteger count = [[logger performSelector:@selector(count)] unsignedIntegerValue];
NSLog(@"Count: %lu", (unsigned long)count);
```

---

## 40.7 Associated Objects

Associated objects ช่วยให้เราเพิ่ม properties ให้ class ผ่าน Category โดยไม่ต้อง subclass:

```objc
#import <objc/runtime.h>

// ปัญหา: Categories ไม่สามารถ synthesize properties ได้
@interface UIView (BorderStyle)
@property (nonatomic, assign) CGFloat borderWidth;
@property (nonatomic, strong) UIColor *borderColor;
@end

@implementation UIView (BorderStyle)

static const char kBorderWidthKey = '\0';
static const char kBorderColorKey = '\0';

// Setter
- (void)setBorderWidth:(CGFloat)borderWidth {
    // บันทึกค่าโดยผูกกับ self
    objc_setAssociatedObject(self,
                              &kBorderWidthKey,
                              @(borderWidth),
                              OBJC_ASSOCIATION_RETAIN_NONATOMIC);
    self.layer.borderWidth = borderWidth;
}

// Getter
- (CGFloat)borderWidth {
    NSNumber *value = objc_getAssociatedObject(self, &kBorderWidthKey);
    return value ? [value floatValue] : 0.0;
}

- (void)setBorderColor:(UIColor *)borderColor {
    objc_setAssociatedObject(self,
                              &kBorderColorKey,
                              borderColor,
                              OBJC_ASSOCIATION_RETAIN_NONATOMIC);
    self.layer.borderColor = borderColor.CGColor;
}

- (UIColor *)borderColor {
    return objc_getAssociatedObject(self, &kBorderColorKey);
}

@end
```

### OBJC_ASSOCIATION Policies

```objc
// Policies (คล้าย property attributes):

// OBJC_ASSOCIATION_ASSIGN
// - เหมือน assign (weak ที่ไม่ zeroing)
// - ระวัง dangling pointer!
objc_setAssociatedObject(self, key, value, OBJC_ASSOCIATION_ASSIGN);

// OBJC_ASSOCIATION_RETAIN_NONATOMIC
// - เหมือน strong, nonatomic
// - ใช้บ่อยที่สุด
objc_setAssociatedObject(self, key, value, OBJC_ASSOCIATION_RETAIN_NONATOMIC);

// OBJC_ASSOCIATION_COPY_NONATOMIC
// - เหมือน copy, nonatomic
// - ใช้กับ NSString, NSArray
objc_setAssociatedObject(self, key, value, OBJC_ASSOCIATION_COPY_NONATOMIC);

// OBJC_ASSOCIATION_RETAIN
// - เหมือน strong, atomic
objc_setAssociatedObject(self, key, value, OBJC_ASSOCIATION_RETAIN);

// OBJC_ASSOCIATION_COPY
// - เหมือน copy, atomic
objc_setAssociatedObject(self, key, value, OBJC_ASSOCIATION_COPY);

// ลบ associated object
objc_setAssociatedObject(self, key, nil, OBJC_ASSOCIATION_ASSIGN);

// ลบ associated objects ทั้งหมด
objc_removeAssociatedObjects(self);
```

### ตัวอย่างการใช้จริง: UIButton Action Block

```objc
// UIButton+Block.h
@interface UIButton (Block)

- (void)setTapBlock:(void(^)(UIButton *button))block;
- (void(^)(UIButton *button))tapBlock;

@end

// UIButton+Block.m
static const char kTapBlockKey = '\0';

@implementation UIButton (Block)

- (void)setTapBlock:(void(^)(UIButton *button))block {
    objc_setAssociatedObject(self,
                              &kTapBlockKey,
                              block,
                              OBJC_ASSOCIATION_COPY_NONATOMIC);
    
    if (block) {
        [self addTarget:self
                 action:@selector(buttonTapped:)
       forControlEvents:UIControlEventTouchUpInside];
    } else {
        [self removeTarget:self
                    action:@selector(buttonTapped:)
          forControlEvents:UIControlEventTouchUpInside];
    }
}

- (void(^)(UIButton *))tapBlock {
    return objc_getAssociatedObject(self, &kTapBlockKey);
}

- (void)buttonTapped:(UIButton *)button {
    void(^block)(UIButton *) = [self tapBlock];
    if (block) {
        block(button);
    }
}

@end

// การใช้งาน
UIButton *button = [UIButton buttonWithType:UIButtonTypeSystem];
[button setTitle:@"กด!" forState:UIControlStateNormal];

[button setTapBlock:^(UIButton *btn) {
    NSLog(@"Button tapped! Title: %@", [btn titleForState:UIControlStateNormal]);
}];
```

---

## 40.8 NSInvocation

NSInvocation เป็น wrapper ของ message ที่ส่งไปยัง object ช่วยให้เราสามารถ:
- เก็บ method call ไว้ใช้ทีหลัง
- เรียก method ที่รู้จาก type ณ runtime
- ส่งต่อ calls

```objc
// สร้าง NSInvocation
NSString *string = @"Hello, World!";
SEL selector = @selector(substringFromIndex:);

NSMethodSignature *sig = [NSString instanceMethodSignatureForSelector:selector];
NSInvocation *invocation = [NSInvocation invocationWithMethodSignature:sig];

invocation.target = string;
invocation.selector = selector;

// ตั้ง arguments (index 0 = self, 1 = _cmd, 2+ = actual args)
NSUInteger index = 7;
[invocation setArgument:&index atIndex:2];

// เรียก method
[invocation invoke];

// รับ return value
NSString *result = nil;
[invocation getReturnValue:&result];
NSLog(@"Result: %@", result);  // "World!"
```

### เก็บ Invocation สำหรับ Deferred Execution

```objc
@interface DeferredCallQueue : NSObject

- (void)deferInvocation:(NSInvocation *)invocation afterDelay:(NSTimeInterval)delay;
- (void)executeAll;

@end

@implementation DeferredCallQueue {
    NSMutableArray *_pendingCalls;
}

- (instancetype)init {
    self = [super init];
    if (self) {
        _pendingCalls = [NSMutableArray array];
    }
    return self;
}

- (void)deferInvocation:(NSInvocation *)invocation afterDelay:(NSTimeInterval)delay {
    // Retain arguments ไม่ให้ถูก deallocate
    [invocation retainArguments];
    
    NSDictionary *item = @{
        @"invocation": invocation,
        @"delay": @(delay),
        @"scheduledAt": [NSDate date]
    };
    [_pendingCalls addObject:item];
}

- (void)executeAll {
    NSArray *calls = [_pendingCalls copy];
    [_pendingCalls removeAllObjects];
    
    for (NSDictionary *item in calls) {
        NSInvocation *inv = item[@"invocation"];
        NSTimeInterval delay = [item[@"delay"] doubleValue];
        
        dispatch_after(dispatch_time(DISPATCH_TIME_NOW, (int64_t)(delay * NSEC_PER_SEC)),
                        dispatch_get_main_queue(), ^{
            [inv invoke];
        });
    }
}

@end
```

### NSInvocation สำหรับ Undo/Redo

```objc
@interface UndoableAction : NSObject

@property (nonatomic, strong) NSInvocation *doAction;
@property (nonatomic, strong) NSInvocation *undoAction;
@property (nonatomic, copy) NSString *name;

@end

@interface ActionManager : NSObject

- (void)performAction:(UndoableAction *)action;
- (void)undo;
- (void)redo;

@end

@implementation ActionManager {
    NSMutableArray *_undoStack;
    NSMutableArray *_redoStack;
}

- (instancetype)init {
    self = [super init];
    if (self) {
        _undoStack = [NSMutableArray array];
        _redoStack = [NSMutableArray array];
    }
    return self;
}

- (void)performAction:(UndoableAction *)action {
    [action.doAction retainArguments];
    [action.undoAction retainArguments];
    [action.doAction invoke];
    
    [_undoStack addObject:action];
    [_redoStack removeAllObjects];
    
    NSLog(@"Did: %@", action.name);
}

- (void)undo {
    if (_undoStack.count == 0) return;
    
    UndoableAction *action = _undoStack.lastObject;
    [_undoStack removeLastObject];
    [action.undoAction invoke];
    [_redoStack addObject:action];
    
    NSLog(@"Undid: %@", action.name);
}

- (void)redo {
    if (_redoStack.count == 0) return;
    
    UndoableAction *action = _redoStack.lastObject;
    [_redoStack removeLastObject];
    [action.doAction invoke];
    [_undoStack addObject:action];
    
    NSLog(@"Redid: %@", action.name);
}

@end
```

---

## 40.9 Runtime Class Creation

สามารถสร้าง class ใหม่ ณ runtime ได้:

```objc
// สร้าง class ใหม่
Class DynamicAnimal = objc_allocateClassPair([NSObject class], "DynamicAnimal", 0);

// เพิ่ม ivar
class_addIvar(DynamicAnimal, "_name", sizeof(id), log2(sizeof(id)), "@");

// เพิ่ม method
IMP nameGetter = imp_implementationWithBlock(^NSString *(id self) {
    Ivar nameIvar = class_getInstanceVariable([self class], "_name");
    return object_getIvar(self, nameIvar);
});
class_addMethod(DynamicAnimal, @selector(name), nameGetter, "@@:");

IMP nameSetter = imp_implementationWithBlock(^void(id self, NSString *name) {
    Ivar nameIvar = class_getInstanceVariable([self class], "_name");
    object_setIvar(self, nameIvar, name);
});
class_addMethod(DynamicAnimal, @selector(setName:), nameSetter, "v@:@");

IMP speakIMP = imp_implementationWithBlock(^void(id self) {
    NSLog(@"I am a dynamic animal named: %@", [self performSelector:@selector(name)]);
});
class_addMethod(DynamicAnimal, @selector(speak), speakIMP, "v@:");

// Register class
objc_registerClassPair(DynamicAnimal);

// ใช้งาน
id animal = [[DynamicAnimal alloc] init];
[animal performSelector:@selector(setName:) withObject:@"Buddy"];
[animal performSelector:@selector(speak)];
// Output: I am a dynamic animal named: Buddy

// Cleanup (ถ้าไม่ใช้แล้ว)
// objc_disposeClassPair(DynamicAnimal);  // แต่ต้องไม่มี instance อยู่!
```

---

## 40.10 Practical Use Cases

### 1. AOP (Aspect-Oriented Programming)

```objc
// ใช้ method swizzling สำหรับ logging/analytics
@implementation NSURLSession (Analytics)

+ (void)load {
    static dispatch_once_t onceToken;
    dispatch_once(&onceToken, ^{
        [SwizzleHelper swizzleInstanceMethod:@selector(dataTaskWithURL:completionHandler:)
                                     inClass:[self class]
                                  withMethod:@selector(analytics_dataTaskWithURL:completionHandler:)];
    });
}

- (NSURLSessionDataTask *)analytics_dataTaskWithURL:(NSURL *)url
                                  completionHandler:(void(^)(NSData *, NSURLResponse *, NSError *))handler {
    NSDate *startTime = [NSDate date];
    NSLog(@"[Analytics] Request started: %@", url.absoluteString);
    
    return [self analytics_dataTaskWithURL:url completionHandler:^(NSData *data, NSURLResponse *response, NSError *error) {
        NSTimeInterval duration = -[startTime timeIntervalSinceNow];
        NSHTTPURLResponse *httpResponse = (NSHTTPURLResponse *)response;
        
        NSLog(@"[Analytics] Request completed: %@ status: %ld, time: %.2fs",
              url.absoluteString,
              (long)httpResponse.statusCode,
              duration);
        
        if (handler) handler(data, response, error);
    }];
}

@end
```

### 2. Automatic JSON Mapping ด้วย Runtime

```objc
@interface AutoJSONMapper : NSObject

+ (id)mapDictionary:(NSDictionary *)dict toClass:(Class)cls;

@end

@implementation AutoJSONMapper

+ (id)mapDictionary:(NSDictionary *)dict toClass:(Class)cls {
    id object = [[cls alloc] init];
    
    unsigned int propCount = 0;
    objc_property_t *properties = class_copyPropertyList(cls, &propCount);
    
    for (unsigned int i = 0; i < propCount; i++) {
        objc_property_t prop = properties[i];
        NSString *propName = [NSString stringWithUTF8String:property_getName(prop)];
        
        id value = dict[propName];
        if (value && value != [NSNull null]) {
            // ตรวจสอบ type ของ property
            NSString *attributes = [NSString stringWithUTF8String:property_getAttributes(prop)];
            
            // Parse type from attributes (T@"ClassName"...)
            if ([attributes hasPrefix:@"T@\"NSString\""]) {
                [object setValue:[value description] forKey:propName];
            } else if ([attributes hasPrefix:@"T@\"NSNumber\""] ||
                       [attributes hasPrefix:@"Ti"] ||
                       [attributes hasPrefix:@"Tq"] ||
                       [attributes hasPrefix:@"Td"]) {
                [object setValue:@([value doubleValue]) forKey:propName];
            } else if ([attributes hasPrefix:@"TB"]) {
                [object setValue:@([value boolValue]) forKey:propName];
            } else {
                // fallback: ลองตั้งค่าตรงๆ
                @try {
                    [object setValue:value forKey:propName];
                } @catch (NSException *e) {
                    NSLog(@"Cannot set %@: %@", propName, e);
                }
            }
        }
    }
    
    free(properties);
    return object;
}

@end
```

### 3. Debugging Tool ด้วย Runtime

```objc
@interface RuntimeDebugger : NSObject

+ (void)printClassHierarchy:(Class)cls;
+ (void)printAllMethodsOfClass:(Class)cls;
+ (void)printAllPropertiesOfClass:(Class)cls;
+ (void)printObjectIvars:(id)object;

@end

@implementation RuntimeDebugger

+ (void)printClassHierarchy:(Class)cls {
    NSMutableArray *hierarchy = [NSMutableArray array];
    Class current = cls;
    
    while (current) {
        [hierarchy addObject:[NSString stringWithUTF8String:class_getName(current)]];
        current = class_getSuperclass(current);
    }
    
    NSLog(@"Class hierarchy for %s:", class_getName(cls));
    for (NSInteger i = 0; i < hierarchy.count; i++) {
        NSString *indent = [@"" stringByPaddingToLength:i * 2 withString:@" " startingAtIndex:0];
        NSLog(@"%@%@", indent, hierarchy[i]);
    }
}

+ (void)printAllMethodsOfClass:(Class)cls {
    NSLog(@"\n=== Methods of %s ===", class_getName(cls));
    
    unsigned int count = 0;
    Method *methods = class_copyMethodList(cls, &count);
    
    NSMutableArray *methodNames = [NSMutableArray array];
    for (unsigned int i = 0; i < count; i++) {
        [methodNames addObject:NSStringFromSelector(method_getName(methods[i]))];
    }
    
    [methodNames sortUsingSelector:@selector(compare:)];
    for (NSString *name in methodNames) {
        NSLog(@"  - %@", name);
    }
    
    free(methods);
    NSLog(@"Total: %d methods", count);
}

+ (void)printAllPropertiesOfClass:(Class)cls {
    NSLog(@"\n=== Properties of %s ===", class_getName(cls));
    
    unsigned int count = 0;
    objc_property_t *props = class_copyPropertyList(cls, &count);
    
    for (unsigned int i = 0; i < count; i++) {
        const char *name = property_getName(props[i]);
        const char *attrs = property_getAttributes(props[i]);
        NSLog(@"  %s (%s)", name, attrs);
    }
    
    free(props);
}

+ (void)printObjectIvars:(id)object {
    NSLog(@"\n=== Ivars of %s instance ===", class_getName([object class]));
    
    unsigned int count = 0;
    Ivar *ivars = class_copyIvarList([object class], &count);
    
    for (unsigned int i = 0; i < count; i++) {
        const char *name = ivar_getName(ivars[i]);
        id value = object_getIvar(object, ivars[i]);
        NSLog(@"  %s = %@", name, value);
    }
    
    free(ivars);
}

@end

// การใช้งาน
[RuntimeDebugger printClassHierarchy:[UIButton class]];
[RuntimeDebugger printAllPropertiesOfClass:[UIView class]];
```

---

## 40.11 Risks and Best Practices

### ข้อควรระวัง

```objc
// ❌ อันตราย: Swizzle โดยไม่ใช้ dispatch_once
// อาจถูกเรียกหลายครั้งและ swizzle กลับ
+ (void)load {
    // ❌
    Method a = class_getInstanceMethod(self, @selector(methodA));
    Method b = class_getInstanceMethod(self, @selector(methodB));
    method_exchangeImplementations(a, b);
}

// ✅ ถูกต้อง
+ (void)load {
    static dispatch_once_t onceToken;
    dispatch_once(&onceToken, ^{
        // swizzle ที่นี่
    });
}

// ❌ Swizzle method ใน +initialize อาจเกิดปัญหา
+ (void)initialize {
    // ❌ อาจถูกเรียกหลายครั้งสำหรับ subclasses
}

// ❌ ไม่เรียก original implementation ใน swizzled method
- (void)swizzled_viewDidAppear:(BOOL)animated {
    // ❌ ลืมเรียก original!
    NSLog(@"Custom code");
}

// ✅ ต้องเรียก original เสมอ (ยกเว้นตั้งใจ override)
- (void)swizzled_viewDidAppear:(BOOL)animated {
    [self swizzled_viewDidAppear:animated];  // เรียก original (ตอนนี้ชี้ไป original)
    NSLog(@"Custom code");
}
```

### Best Practices

```objc
// 1. ใช้ unique prefix สำหรับ swizzled methods
- (void)myapp_viewDidAppear:(BOOL)animated {
    // ชื่อ prefix ป้องกัน conflict กับ library อื่น
}

// 2. ตรวจสอบผลลัพธ์
BOOL success = class_addMethod(cls, sel, imp, types);
if (!success) {
    NSLog(@"Failed to add method");
}

// 3. สร้าง instance ใหม่ของ NSFileManager สำหรับ delegate
NSFileManager *fm = [[NSFileManager alloc] init];  // ไม่ใช้ defaultManager สำหรับ delegate

// 4. Swizzle ใน +load ไม่ใช่ +initialize
// +load เรียกครั้งเดียวต่อ class ก่อน main()
// +initialize อาจเรียกหลายครั้ง

// 5. อย่า swizzle ใน production ถ้าไม่จำเป็น
// ใช้สำหรับ debugging, testing, analytics เท่านั้น

// 6. Document ทุก swizzle อย่างชัดเจน
// สร้าง SWIZZLING.md หรือ comment ชัดเจน

// 7. ระวัง thread safety
// objc_msgSend เป็น thread-safe แต่ swizzling ไม่ใช่
// ทำ swizzling ก่อน multi-threading เริ่ม (ใน +load)
```

### เมื่อใดควรใช้ Runtime

```objc
// ✅ ใช้ Runtime เมื่อ:
// 1. Method swizzling สำหรับ debugging/analytics/testing
// 2. Associated objects สำหรับ extend existing classes
// 3. Introspection สำหรับ frameworks/tools
// 4. Dynamic dispatch สำหรับ plugin architectures
// 5. NSInvocation สำหรับ deferred execution หรือ undo/redo

// ❌ อย่าใช้ Runtime เมื่อ:
// 1. มีวิธีปกติที่ทำได้ง่ายกว่า
// 2. ใน production code ที่ไม่จำเป็น
// 3. Swizzle Apple's private methods
// 4. เป็น quick fix สำหรับ bugs ที่ควรแก้อย่างอื่น
```

---

## 40.12 Complete Example: Simple AOP Framework

```objc
// AspectPoint.h - สร้าง mini AOP framework
typedef NS_ENUM(NSInteger, AspectPointPosition) {
    AspectPointPositionBefore,
    AspectPointPositionAfter,
    AspectPointPositionInstead
};

typedef void(^AspectBlock)(id target, NSArray *arguments);

@interface AspectPoint : NSObject

+ (BOOL)hookClass:(Class)cls
         selector:(SEL)sel
         position:(AspectPointPosition)position
            block:(AspectBlock)block;

@end

@implementation AspectPoint

static const char kAspectBlocksKey = '\0';

+ (BOOL)hookClass:(Class)cls
         selector:(SEL)sel
         position:(AspectPointPosition)position
            block:(AspectBlock)block {
    
    Method method = class_getInstanceMethod(cls, sel);
    if (!method) return NO;
    
    // ชื่อ swizzled selector
    NSString *swizzledName = [NSString stringWithFormat:@"aspect_%@",
                               NSStringFromSelector(sel)];
    SEL swizzledSel = NSSelectorFromString(swizzledName);
    
    // เก็บ blocks ไว้ใน associated object
    NSMutableDictionary *allBlocks = objc_getAssociatedObject(cls, &kAspectBlocksKey);
    if (!allBlocks) {
        allBlocks = [NSMutableDictionary dictionary];
        objc_setAssociatedObject(cls,
                                  &kAspectBlocksKey,
                                  allBlocks,
                                  OBJC_ASSOCIATION_RETAIN_NONATOMIC);
    }
    
    NSString *key = [NSString stringWithFormat:@"%@_%d",
                     NSStringFromSelector(sel), (int)position];
    allBlocks[key] = [block copy];
    
    // Swizzle
    IMP newIMP = imp_implementationWithBlock(^(id self, ...) {
        NSMutableDictionary *blocks = objc_getAssociatedObject([self class], &kAspectBlocksKey);
        
        NSString *beforeKey = [NSString stringWithFormat:@"%@_%d",
                                NSStringFromSelector(sel), (int)AspectPointPositionBefore];
        NSString *afterKey = [NSString stringWithFormat:@"%@_%d",
                               NSStringFromSelector(sel), (int)AspectPointPositionAfter];
        NSString *insteadKey = [NSString stringWithFormat:@"%@_%d",
                                 NSStringFromSelector(sel), (int)AspectPointPositionInstead];
        
        // Before block
        AspectBlock beforeBlock = blocks[beforeKey];
        if (beforeBlock) {
            beforeBlock(self, @[]);
        }
        
        // Instead or original
        AspectBlock insteadBlock = blocks[insteadKey];
        if (!insteadBlock) {
            // เรียก original
            [self performSelector:swizzledSel];
        } else {
            insteadBlock(self, @[]);
        }
        
        // After block
        AspectBlock afterBlock = blocks[afterKey];
        if (afterBlock) {
            afterBlock(self, @[]);
        }
    });
    
    class_addMethod(cls, swizzledSel,
                    method_getImplementation(method),
                    method_getTypeEncoding(method));
    
    class_replaceMethod(cls, sel, newIMP, method_getTypeEncoding(method));
    
    return YES;
}

@end

// การใช้งาน
[AspectPoint hookClass:[UIViewController class]
              selector:@selector(viewDidLoad)
              position:AspectPointPositionAfter
                 block:^(id target, NSArray *arguments) {
    NSLog(@"[AOP] viewDidLoad called on: %@",
          NSStringFromClass([target class]));
}];
```

---

## แบบฝึกหัด (Practice Exercises)

### Exercise 1: Class Inspector
สร้าง `ClassInspector` ที่แสดง:
- Method list ของ class และ superclasses
- Property list พร้อม attributes
- Protocol list
- Ivar list พร้อม offset

### Exercise 2: Method Profiler
สร้าง method profiler โดยใช้ swizzling ที่:
- วัดเวลาที่ใช้ใน method
- นับจำนวนครั้งที่เรียก
- สร้าง report ได้

### Exercise 3: KVO Implementation
สร้าง manual KVO implementation ด้วย runtime ที่:
- Monitor property changes
- แจ้งเมื่อค่าเปลี่ยน
- Unsubscribe ได้

### Exercise 4: Auto Layout ด้วย Runtime
สร้าง category บน UIView ที่ใช้ associated objects เก็บ:
- Custom constraints
- Layout priorities
- Padding/margin values

### Exercise 5: Method Caching
สร้าง system ที่ cache return values ของ methods โดย:
- Swizzle method เพื่อ intercept calls
- Cache results ด้วย arguments เป็น key
- Invalidate cache เมื่อ properties เปลี่ยน

### Exercise 6: Dynamic Delegate
สร้าง `DynamicDelegate` ที่:
- Implement ทุก optional methods ของ protocol แบบ dynamic
- Return sensible defaults
- Log calls ที่ไม่มี real implementation

### Exercise 7: Object Comparison
สร้าง function ที่ใช้ runtime เปรียบเทียบ 2 objects:
- ตรวจสอบ class เหมือนกันหรือไม่
- เปรียบเทียบทุก property value
- สร้าง diff report

### Exercise 8: Method Injection
สร้าง system ที่:
- อ่าน method implementations จาก class หนึ่ง
- Inject เข้าไปใน class อื่น
- Handle conflicts

### Exercise 9: Automatic toString
สร้าง base class หรือ category ที่:
- Override `description` โดยอัตโนมัติ
- แสดง ivar values ทั้งหมด
- Exclude certain ivars ด้วย annotation

### Exercise 10: Runtime-based Test Framework
สร้าง mini unit test framework ที่:
- Find methods ที่ขึ้นต้นด้วย "test"
- Run แต่ละ method และจับ exceptions
- Report pass/fail

---

## สรุป

Objective-C Runtime เป็น powerful mechanism เบื้องหลัง:

1. **Class Inspection**: ดู methods, properties, protocols ณ runtime
2. **Method Swizzling**: สลับ implementations สำหรับ AOP patterns
3. **Dynamic Resolution**: เพิ่ม methods ณ runtime ตามความต้องการ
4. **Message Forwarding**: Proxy patterns และ delegation ขั้นสูง
5. **Associated Objects**: เพิ่ม properties ให้ existing classes
6. **NSInvocation**: Deferred execution และ undo/redo

**Rules:**
- ใช้ `dispatch_once` เสมอสำหรับ swizzling
- เรียก original implementation เมื่อ swizzle (ยกเว้นตั้งใจ)
- ระวัง thread safety
- Document ทุกการใช้ runtime
- หลีกเลี่ยงการ swizzle Apple's private methods

นี่คือจุดสิ้นสุดของ Part 31-40 ของคอร์ส Objective-C คุณได้เรียนรู้ตั้งแต่ Delegate Pattern, File System, Serialization จนถึง Runtime ซึ่งเป็น features ขั้นสูงที่ทำให้ Objective-C เป็นภาษาที่ทรงพลัง
