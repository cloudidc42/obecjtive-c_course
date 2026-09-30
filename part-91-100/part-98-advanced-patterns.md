# ตอนที่ 98: Advanced Patterns และเทคนิคขั้นสูงใน Objective-C

## บทนำ

บทนี้จะสำรวจ patterns และเทคนิคขั้นสูงที่ใช้ใน production code จริง ตั้งแต่ method swizzling ไปจนถึง reactive programming concepts ที่ implement เองโดยไม่ต้องพึ่ง third-party libraries

---

## ส่วนที่ 1: Method Swizzling ใน Production

### พื้นฐาน Method Swizzling

```objc
#import <objc/runtime.h>

// Method swizzling template ที่ปลอดภัย
@interface NSObject (SafeSwizzling)
+ (BOOL)swizzleInstanceMethod:(SEL)original with:(SEL)swizzled;
+ (BOOL)swizzleClassMethod:(SEL)original with:(SEL)swizzled;
@end

@implementation NSObject (SafeSwizzling)

+ (BOOL)swizzleInstanceMethod:(SEL)original with:(SEL)swizzled {
    Class cls = [self class];
    
    Method originalMethod = class_getInstanceMethod(cls, original);
    Method swizzledMethod = class_getInstanceMethod(cls, swizzled);
    
    if (!originalMethod || !swizzledMethod) {
        NSLog(@"Swizzling failed: method not found");
        return NO;
    }
    
    // ลอง add เพื่อ handle superclass method
    BOOL didAdd = class_addMethod(cls,
                                  original,
                                  method_getImplementation(swizzledMethod),
                                  method_getTypeEncoding(swizzledMethod));
    
    if (didAdd) {
        // original ไม่ได้ implement ใน this class -> add + replace
        class_replaceMethod(cls,
                           swizzled,
                           method_getImplementation(originalMethod),
                           method_getTypeEncoding(originalMethod));
    } else {
        // original มีอยู่แล้ว -> exchange
        method_exchangeImplementations(originalMethod, swizzledMethod);
    }
    
    return YES;
}

+ (BOOL)swizzleClassMethod:(SEL)original with:(SEL)swizzled {
    // Class methods อยู่ใน metaclass
    Class cls = object_getClass((id)self);
    
    Method originalMethod = class_getInstanceMethod(cls, original);
    Method swizzledMethod = class_getInstanceMethod(cls, swizzled);
    
    if (!originalMethod || !swizzledMethod) return NO;
    
    method_exchangeImplementations(originalMethod, swizzledMethod);
    return YES;
}

@end
```

---

### Use Case: Analytics Tracking

```objc
// Track screen views อัตโนมัติ
@implementation UIViewController (Analytics)

+ (void)load {
    static dispatch_once_t onceToken;
    dispatch_once(&onceToken, ^{
        [UIViewController swizzleInstanceMethod:@selector(viewDidAppear:)
                                           with:@selector(analytics_viewDidAppear:)];
        [UIViewController swizzleInstanceMethod:@selector(viewDidDisappear:)
                                           with:@selector(analytics_viewDidDisappear:)];
    });
}

- (void)analytics_viewDidAppear:(BOOL)animated {
    [self analytics_viewDidAppear:animated]; // calls original
    
    NSString *screenName = [self analyticsScreenName];
    if (screenName) {
        [[AnalyticsService shared] trackScreenView:screenName
                                        properties:@{
            @"class": NSStringFromClass([self class]),
            @"timestamp": @([NSDate date].timeIntervalSince1970)
        }];
    }
}

- (void)analytics_viewDidDisappear:(BOOL)animated {
    [self analytics_viewDidDisappear:animated]; // calls original
    
    NSString *screenName = [self analyticsScreenName];
    if (screenName) {
        [[AnalyticsService shared] trackEvent:@"screen_exit"
                                   properties:@{@"screen": screenName}];
    }
}

// Override ใน subclass เพื่อระบุชื่อ screen
- (NSString *)analyticsScreenName {
    return nil; // default: ไม่ track
}

@end

// ใน HomeViewController
@implementation HomeViewController

- (NSString *)analyticsScreenName {
    return @"Home"; // track screen นี้
}

@end
```

---

### Use Case: Crash Prevention

```objc
// ป้องกัน crash จาก unrecognized selector
@implementation NSObject (CrashGuard)

+ (void)load {
    static dispatch_once_t onceToken;
    dispatch_once(&onceToken, ^{
        // Swizzle doesNotRecognizeSelector:
        Method original = class_getInstanceMethod([NSObject class], 
                                                   @selector(doesNotRecognizeSelector:));
        Method guarded = class_getInstanceMethod([NSObject class], 
                                                  @selector(crashGuard_doesNotRecognizeSelector:));
        method_exchangeImplementations(original, guarded);
    });
}

- (void)crashGuard_doesNotRecognizeSelector:(SEL)aSelector {
    // Log แทน crash
    NSString *message = [NSString stringWithFormat:
                         @"[%@ doesNotRecognizeSelector:%@]",
                         NSStringFromClass([self class]),
                         NSStringFromSelector(aSelector)];
    
    NSLog(@"CRASH PREVENTED: %@", message);
    
    // Report to crash analytics
    [[CrashReporter shared] reportNonFatalError:message];
    
    // ไม่ call original -> ไม่ crash
    // (อันตรายใน debug build - ควร crash ใน development)
    
#ifdef DEBUG
    // ใน debug: crash เพื่อ catch bugs ตั้งแต่เนิ่นๆ
    [self crashGuard_doesNotRecognizeSelector:aSelector];
#endif
}

@end
```

---

## ส่วนที่ 2: Associated Objects Patterns

### พื้นฐาน Associated Objects

```objc
#import <objc/runtime.h>

// เพิ่ม "properties" ให้ existing class ผ่าน category
@interface UIView (LoadingIndicator)

@property (nonatomic, strong) UIActivityIndicatorView *loadingIndicator;
@property (nonatomic, assign) BOOL isLoading;

- (void)startLoading;
- (void)stopLoading;

@end

// Keys สำหรับ associated objects
static const void *kLoadingIndicatorKey = &kLoadingIndicatorKey;
static const void *kIsLoadingKey = &kIsLoadingKey;

@implementation UIView (LoadingIndicator)

- (UIActivityIndicatorView *)loadingIndicator {
    UIActivityIndicatorView *indicator = objc_getAssociatedObject(self, kLoadingIndicatorKey);
    
    if (!indicator) {
        // Lazy create
        indicator = [[UIActivityIndicatorView alloc] 
                        initWithActivityIndicatorStyle:UIActivityIndicatorViewStyleMedium];
        indicator.hidesWhenStopped = YES;
        indicator.translatesAutoresizingMaskIntoConstraints = NO;
        
        [self addSubview:indicator];
        [NSLayoutConstraint activateConstraints:@[
            [indicator.centerXAnchor constraintEqualToAnchor:self.centerXAnchor],
            [indicator.centerYAnchor constraintEqualToAnchor:self.centerYAnchor]
        ]];
        
        objc_setAssociatedObject(self, kLoadingIndicatorKey, indicator, 
                                 OBJC_ASSOCIATION_RETAIN_NONATOMIC);
    }
    
    return indicator;
}

- (void)setLoadingIndicator:(UIActivityIndicatorView *)indicator {
    objc_setAssociatedObject(self, kLoadingIndicatorKey, indicator, 
                             OBJC_ASSOCIATION_RETAIN_NONATOMIC);
}

- (BOOL)isLoading {
    return [objc_getAssociatedObject(self, kIsLoadingKey) boolValue];
}

- (void)setIsLoading:(BOOL)isLoading {
    objc_setAssociatedObject(self, kIsLoadingKey, @(isLoading), 
                             OBJC_ASSOCIATION_RETAIN_NONATOMIC);
}

- (void)startLoading {
    self.isLoading = YES;
    [self.loadingIndicator startAnimating];
    self.userInteractionEnabled = NO;
    
    // Dim overlay
    self.alpha = 0.7;
}

- (void)stopLoading {
    self.isLoading = NO;
    [self.loadingIndicator stopAnimating];
    self.userInteractionEnabled = YES;
    self.alpha = 1.0;
}

@end
```

---

### Advanced: Associated Block Handlers

```objc
// เพิ่ม tap handler ให้ UIView ผ่าน associated object
@interface UIView (TapHandler)
- (void)addTapHandlerWithBlock:(void (^)(UIView *))block;
- (void)removeTapHandler;
@end

static const void *kTapHandlerKey = &kTapHandlerKey;

@implementation UIView (TapHandler)

- (void)addTapHandlerWithBlock:(void (^)(UIView *))block {
    // Store block as associated object
    objc_setAssociatedObject(self, kTapHandlerKey, block, 
                             OBJC_ASSOCIATION_COPY_NONATOMIC);
    
    // Remove existing gesture
    for (UIGestureRecognizer *gr in self.gestureRecognizers) {
        if ([gr isKindOfClass:[UITapGestureRecognizer class]]) {
            [self removeGestureRecognizer:gr];
        }
    }
    
    // Add new gesture
    UITapGestureRecognizer *tap = [[UITapGestureRecognizer alloc] 
                                    initWithTarget:self 
                                    action:@selector(handleTap:)];
    [self addGestureRecognizer:tap];
    self.userInteractionEnabled = YES;
}

- (void)handleTap:(UITapGestureRecognizer *)gesture {
    void (^handler)(UIView *) = objc_getAssociatedObject(self, kTapHandlerKey);
    if (handler) {
        handler(self);
    }
}

- (void)removeTapHandler {
    objc_setAssociatedObject(self, kTapHandlerKey, nil, 
                             OBJC_ASSOCIATION_COPY_NONATOMIC);
}

@end

// การใช้งาน
UIView *cardView = [[UIView alloc] initWithFrame:CGRectMake(0, 0, 200, 100)];
[cardView addTapHandlerWithBlock:^(UIView *view) {
    NSLog(@"Card tapped!");
    [UIView animateWithDuration:0.1 animations:^{
        view.transform = CGAffineTransformMakeScale(0.95, 0.95);
    } completion:^(BOOL finished) {
        [UIView animateWithDuration:0.1 animations:^{
            view.transform = CGAffineTransformIdentity;
        }];
    }];
}];
```

---

## ส่วนที่ 3: Proxy Pattern กับ NSProxy

### NSProxy พื้นฐาน

```objc
// NSProxy - abstract class สำหรับ proxy objects
@interface LazyProxy : NSProxy

- (instancetype)initWithClass:(Class)cls;

@end

@implementation LazyProxy {
    Class _proxiedClass;
    id _realObject;
}

- (instancetype)initWithClass:(Class)cls {
    // NSProxy ไม่มี [super init]
    _proxiedClass = cls;
    return self;
}

- (id)realObject {
    if (!_realObject) {
        _realObject = [[_proxiedClass alloc] init];
        NSLog(@"Lazy-created %@", NSStringFromClass(_proxiedClass));
    }
    return _realObject;
}

- (NSMethodSignature *)methodSignatureForSelector:(SEL)sel {
    return [[self realObject] methodSignatureForSelector:sel];
}

- (void)forwardInvocation:(NSInvocation *)invocation {
    invocation.target = [self realObject];
    [invocation invoke];
}

- (BOOL)respondsToSelector:(SEL)aSelector {
    return [[self realObject] respondsToSelector:aSelector];
}

- (BOOL)isKindOfClass:(Class)aClass {
    return [[self realObject] isKindOfClass:aClass];
}

@end

// การใช้งาน
LazyProxy *proxy = [[LazyProxy alloc] initWithClass:[HeavyService class]];
// HeavyService ยังไม่ถูก create

// เมื่อ call method แรก -> HeavyService ถูก create
[proxy performHeavyOperation];
```

---

### Thread-Safe Proxy

```objc
// Thread-safe proxy ที่ serialize operations ผ่าน serial queue
@interface ThreadSafeProxy : NSProxy

+ (instancetype)proxyFor:(id)object;

@end

@implementation ThreadSafeProxy {
    id _object;
    dispatch_queue_t _queue;
}

+ (instancetype)proxyFor:(id)object {
    ThreadSafeProxy *proxy = [self alloc]; // NSProxy ไม่ call init
    proxy->_object = object;
    proxy->_queue = dispatch_queue_create("com.proxy.queue", DISPATCH_QUEUE_SERIAL);
    return proxy;
}

- (NSMethodSignature *)methodSignatureForSelector:(SEL)sel {
    return [_object methodSignatureForSelector:sel];
}

- (void)forwardInvocation:(NSInvocation *)invocation {
    [invocation retainArguments]; // retain args ก่อน dispatch
    
    dispatch_sync(_queue, ^{
        invocation.target = self->_object;
        [invocation invoke];
    });
}

@end

// ใช้งาน
NSMutableArray *array = [NSMutableArray array];
NSMutableArray *safeArray = (NSMutableArray *)[ThreadSafeProxy proxyFor:array];

// Thread-safe access
dispatch_async(queue1, ^{ [safeArray addObject:@"a"]; });
dispatch_async(queue2, ^{ [safeArray addObject:@"b"]; });
```

---

### Logging Proxy

```objc
// Proxy ที่ log ทุก method call
@interface LoggingProxy : NSProxy

+ (instancetype)proxyFor:(id)object logTarget:(NSString *)name;

@end

@implementation LoggingProxy {
    id _object;
    NSString *_name;
    NSMutableArray *_callLog;
}

+ (instancetype)proxyFor:(id)object logTarget:(NSString *)name {
    LoggingProxy *proxy = [self alloc];
    proxy->_object = object;
    proxy->_name = name;
    proxy->_callLog = [NSMutableArray array];
    return proxy;
}

- (NSMethodSignature *)methodSignatureForSelector:(SEL)sel {
    return [_object methodSignatureForSelector:sel];
}

- (void)forwardInvocation:(NSInvocation *)invocation {
    NSString *selName = NSStringFromSelector(invocation.selector);
    NSDate *start = [NSDate date];
    
    invocation.target = _object;
    [invocation invoke];
    
    NSTimeInterval duration = [[NSDate date] timeIntervalSinceDate:start];
    
    NSDictionary *entry = @{
        @"method": selName,
        @"duration": @(duration * 1000), // milliseconds
        @"timestamp": start
    };
    [_callLog addObject:entry];
    
    NSLog(@"[%@] %@ took %.2fms", _name, selName, duration * 1000);
}

- (NSArray *)callLog {
    return [_callLog copy];
}

@end
```

---

## ส่วนที่ 4: Message Forwarding Chains

### Chain of Responsibility

```objc
// Middleware chain สำหรับ request processing
@protocol RequestHandler <NSObject>
- (BOOL)handleRequest:(NSMutableDictionary *)request;
@end

@interface RequestChain : NSObject

- (void)addHandler:(id<RequestHandler>)handler;
- (BOOL)processRequest:(NSMutableDictionary *)request;

@end

@implementation RequestChain {
    NSMutableArray<id<RequestHandler>> *_handlers;
}

- (instancetype)init {
    self = [super init];
    if (self) {
        _handlers = [NSMutableArray array];
    }
    return self;
}

- (void)addHandler:(id<RequestHandler>)handler {
    [_handlers addObject:handler];
}

- (BOOL)processRequest:(NSMutableDictionary *)request {
    for (id<RequestHandler> handler in _handlers) {
        if ([handler handleRequest:request]) {
            return YES; // handled
        }
    }
    return NO; // not handled
}

@end

// Handlers
@interface AuthenticationHandler : NSObject <RequestHandler>
@end

@implementation AuthenticationHandler
- (BOOL)handleRequest:(NSMutableDictionary *)request {
    NSString *token = request[@"auth_token"];
    if (!token) {
        request[@"error"] = @"Authentication required";
        return YES; // handled (with error)
    }
    
    // Validate token
    if (![self isValidToken:token]) {
        request[@"error"] = @"Invalid token";
        return YES;
    }
    
    request[@"user"] = [self userForToken:token];
    return NO; // pass to next handler
}
@end

@interface RateLimitHandler : NSObject <RequestHandler>
@property (nonatomic, assign) NSInteger maxRequestsPerMinute;
@end

@implementation RateLimitHandler {
    NSMutableDictionary *_requestCounts;
    NSTimer *_resetTimer;
}

- (BOOL)handleRequest:(NSMutableDictionary *)request {
    NSString *userId = [request[@"user"] valueForKey:@"id"];
    if (!userId) userId = @"anonymous";
    
    NSInteger count = [_requestCounts[userId] integerValue];
    if (count >= self.maxRequestsPerMinute) {
        request[@"error"] = @"Rate limit exceeded";
        return YES; // blocked
    }
    
    _requestCounts[userId] = @(count + 1);
    return NO; // pass through
}

@end

// การใช้งาน
RequestChain *chain = [[RequestChain alloc] init];
[chain addHandler:[[AuthenticationHandler alloc] init]];

RateLimitHandler *rateLimit = [[RateLimitHandler alloc] init];
rateLimit.maxRequestsPerMinute = 60;
[chain addHandler:rateLimit];

[chain addHandler:[[LoggingHandler alloc] init]];
[chain addHandler:[[BusinessLogicHandler alloc] init]];

NSMutableDictionary *request = [@{@"path": @"/api/users", @"auth_token": @"abc123"} mutableCopy];
[chain processRequest:request];
```

---

## ส่วนที่ 5: Dynamic Subclassing

### KVO Implementation Pattern

```objc
// เข้าใจ KVO โดย implement เอง
@interface KVOObserver : NSObject

+ (void)addObserver:(id)observer 
           toObject:(id)object
         forKeyPath:(NSString *)keyPath
            handler:(void (^)(id oldValue, id newValue))handler;

@end

@implementation KVOObserver

+ (void)addObserver:(id)observer 
           toObject:(id)object
         forKeyPath:(NSString *)keyPath
            handler:(void (^)(id, id))handler {
    
    Class originalClass = [object class];
    NSString *dynamicClassName = [NSString stringWithFormat:@"KVOProxy_%@", 
                                  NSStringFromClass(originalClass)];
    
    // สร้าง dynamic subclass ถ้ายังไม่มี
    Class dynamicClass = NSClassFromString(dynamicClassName);
    if (!dynamicClass) {
        dynamicClass = objc_allocateClassPair(originalClass, 
                                              dynamicClassName.UTF8String, 
                                              0);
        
        // Override class เพื่อ hide ว่าเป็น subclass
        IMP classIMP = imp_implementationWithBlock(^Class(id self) {
            return originalClass;
        });
        class_addMethod(dynamicClass, @selector(class), classIMP, "#@:");
        
        objc_registerClassPair(dynamicClass);
    }
    
    // Generate setter name
    NSString *setterName = [NSString stringWithFormat:@"set%@%@:",
                            [[keyPath substringToIndex:1] uppercaseString],
                            [keyPath substringFromIndex:1]];
    SEL setterSEL = NSSelectorFromString(setterName);
    
    // Override setter
    Method originalSetter = class_getInstanceMethod(originalClass, setterSEL);
    if (originalSetter) {
        IMP setterIMP = imp_implementationWithBlock(^(id self, id newValue) {
            id oldValue = [self valueForKey:keyPath];
            
            // Call original setter
            struct objc_super superInfo = { self, originalClass };
            void (*superSetter)(struct objc_super *, SEL, id) = 
                (void *)objc_msgSendSuper;
            superSetter(&superInfo, setterSEL, newValue);
            
            // Notify
            if (handler) handler(oldValue, newValue);
        });
        
        class_addMethod(dynamicClass, setterSEL, setterIMP, 
                       method_getTypeEncoding(originalSetter));
    }
    
    // Change object's class to dynamic subclass
    object_setClass(object, dynamicClass);
}

@end
```

---

## ส่วนที่ 6: Aspect-Oriented Programming ใน ObjC

### AOP ด้วย Message Forwarding

```objc
// Aspect weaving
typedef enum {
    AspectPositionBefore = 0,
    AspectPositionAfter,
    AspectPositionInstead
} AspectPosition;

@interface AspectWeaver : NSObject

+ (void)weaveClass:(Class)cls
          selector:(SEL)selector
          position:(AspectPosition)position
             block:(void (^)(id target, NSArray *args))block;

@end

@implementation AspectWeaver

+ (void)weaveClass:(Class)cls
          selector:(SEL)selector
          position:(AspectPosition)position
             block:(void (^)(id, NSArray *))block {
    
    Method originalMethod = class_getInstanceMethod(cls, selector);
    if (!originalMethod) return;
    
    // ชื่อ aliased selector
    NSString *aliasName = [NSString stringWithFormat:@"__aspect_%@", 
                           NSStringFromSelector(selector)];
    SEL aliasSEL = NSSelectorFromString(aliasName);
    
    // Copy original implementation
    class_addMethod(cls, aliasSEL,
                    method_getImplementation(originalMethod),
                    method_getTypeEncoding(originalMethod));
    
    // Replace with weaved implementation
    IMP weavedIMP = imp_implementationWithBlock(^(id self, ...) {
        // Extract args (simplified - full impl needs NSInvocation)
        NSArray *args = @[]; // simplified
        
        if (position == AspectPositionBefore) {
            block(self, args);
        }
        
        if (position != AspectPositionInstead) {
            // Call original
#pragma clang diagnostic push
#pragma clang diagnostic ignored "-Warc-performSelector-leaks"
            [self performSelector:aliasSEL];
#pragma clang diagnostic pop
        }
        
        if (position == AspectPositionAfter) {
            block(self, args);
        }
        
        if (position == AspectPositionInstead) {
            block(self, args);
        }
    });
    
    class_replaceMethod(cls, selector, weavedIMP, 
                       method_getTypeEncoding(originalMethod));
}

@end

// การใช้งาน
// เพิ่ม logging ให้ทุก save operation
[AspectWeaver weaveClass:[UserRepository class]
               selector:@selector(saveUser:)
               position:AspectPositionBefore
                  block:^(id target, NSArray *args) {
    NSLog(@"About to save user: %@", args.firstObject);
}];

[AspectWeaver weaveClass:[UserRepository class]
               selector:@selector(saveUser:)
               position:AspectPositionAfter
                  block:^(id target, NSArray *args) {
    NSLog(@"User saved successfully");
    [[Analytics shared] trackEvent:@"user_saved"];
}];
```

---

## ส่วนที่ 7: Reactive Programming Concepts

### Observable Pattern (ไม่ใช้ Library)

```objc
// Observable
@interface Observable<T> : NSObject

+ (instancetype)create:(void (^)(void (^next)(T), void (^error)(NSError *), void (^complete)(void)))subscribe;

- (id)subscribe:(void (^)(T value))next
          error:(void (^)(NSError *))error
       complete:(void (^)(void))complete;

- (Observable *)map:(id (^)(T value))transform;
- (Observable *)filter:(BOOL (^)(T value))predicate;
- (Observable *)take:(NSInteger)count;

@end

@interface Subscription : NSObject
- (void)dispose;
@property (nonatomic, assign, getter=isDisposed) BOOL disposed;
@end

@implementation Subscription {
    dispatch_block_t _disposer;
}

- (instancetype)initWithDisposer:(dispatch_block_t)disposer {
    self = [super init];
    if (self) _disposer = disposer;
    return self;
}

- (void)dispose {
    _disposed = YES;
    if (_disposer) _disposer();
}

@end

@implementation Observable {
    void (^_subscribeBlock)(void (^)(id), void (^)(NSError *), void (^)(void));
}

+ (instancetype)create:(void (^)(void (^)(id), void (^)(NSError *), void (^)(void)))subscribe {
    Observable *obs = [[self alloc] init];
    obs->_subscribeBlock = subscribe;
    return obs;
}

- (id)subscribe:(void (^)(id))next 
          error:(void (^)(NSError *))error 
       complete:(void (^)(void))complete {
    
    __block BOOL disposed = NO;
    
    void (^safeNext)(id) = ^(id value) {
        if (!disposed && next) next(value);
    };
    
    void (^safeError)(NSError *) = ^(NSError *err) {
        if (!disposed && error) error(err);
    };
    
    void (^safeComplete)(void) = ^{
        if (!disposed && complete) complete();
    };
    
    _subscribeBlock(safeNext, safeError, safeComplete);
    
    return [[Subscription alloc] initWithDisposer:^{
        disposed = YES;
    }];
}

- (Observable *)map:(id (^)(id))transform {
    return [Observable create:^(void (^next)(id), void (^error)(NSError *), void (^complete)(void)) {
        [self subscribe:^(id value) {
            next(transform(value));
        } error:error complete:complete];
    }];
}

- (Observable *)filter:(BOOL (^)(id))predicate {
    return [Observable create:^(void (^next)(id), void (^error)(NSError *), void (^complete)(void)) {
        [self subscribe:^(id value) {
            if (predicate(value)) {
                next(value);
            }
        } error:error complete:complete];
    }];
}

- (Observable *)take:(NSInteger)count {
    __block NSInteger remaining = count;
    
    return [Observable create:^(void (^next)(id), void (^error)(NSError *), void (^complete)(void)) {
        __block id sub;
        sub = [self subscribe:^(id value) {
            if (remaining > 0) {
                next(value);
                remaining--;
                if (remaining == 0) {
                    complete();
                    [sub dispose];
                }
            }
        } error:error complete:complete];
    }];
}

@end

// Subject - ทั้ง Observable และ Observer
@interface Subject : Observable

- (void)next:(id)value;
- (void)error:(NSError *)error;
- (void)complete;

@end

@implementation Subject {
    NSMutableArray *_observers; // [(next, error, complete)]
    BOOL _completed;
    NSError *_error;
}

- (instancetype)init {
    self = [super init];
    if (self) {
        _observers = [NSMutableArray array];
    }
    return self;
}

- (id)subscribe:(void (^)(id))next 
          error:(void (^)(NSError *))error 
       complete:(void (^)(void))complete {
    
    NSDictionary *observer = @{
        @"next": next ?: ^(id v){},
        @"error": error ?: ^(NSError *e){},
        @"complete": complete ?: ^{}
    };
    
    [_observers addObject:observer];
    
    // Replay terminal state
    if (_completed) complete();
    if (_error) error(_error);
    
    NSUInteger idx = _observers.count - 1;
    return [[Subscription alloc] initWithDisposer:^{
        if (idx < self->_observers.count) {
            [self->_observers removeObjectAtIndex:idx];
        }
    }];
}

- (void)next:(id)value {
    if (_completed || _error) return;
    NSArray *observers = [_observers copy];
    for (NSDictionary *obs in observers) {
        ((void (^)(id))obs[@"next"])(value);
    }
}

- (void)error:(NSError *)error {
    _error = error;
    for (NSDictionary *obs in _observers) {
        ((void (^)(NSError *))obs[@"error"])(error);
    }
    [_observers removeAllObjects];
}

- (void)complete {
    _completed = YES;
    for (NSDictionary *obs in _observers) {
        ((void (^)(void))obs[@"complete"])();
    }
    [_observers removeAllObjects];
}

@end

// การใช้งาน
Subject *searchSubject = [[Subject alloc] init];

// Chain operations
id subscription = [[[searchSubject
    filter:^BOOL(NSString *text) {
        return text.length >= 3; // debounce short queries
    }]
    map:^id(NSString *text) {
        return [text lowercaseString];
    }]
    subscribe:^(NSString *query) {
        NSLog(@"Searching for: %@", query);
        [self performSearch:query];
    } error:^(NSError *error) {
        NSLog(@"Error: %@", error);
    } complete:^{
        NSLog(@"Search complete");
    }];

// Trigger search
[searchSubject next:@"a"];   // filtered out (length < 3)
[searchSubject next:@"iOS"]; // triggers search
[searchSubject next:@"Objective-C"]; // triggers search
```

---

## ส่วนที่ 8: Promise/Future Pattern

```objc
// Promise implementation
typedef NS_ENUM(NSInteger, PromiseState) {
    PromiseStatePending,
    PromiseStateFulfilled,
    PromiseStateRejected
};

@interface Promise : NSObject

+ (instancetype)promiseWithBlock:(void (^)(void (^resolve)(id), void (^reject)(NSError *)))block;

- (Promise *)then:(id (^)(id value))onFulfilled;
- (Promise *)catch:(void (^)(NSError *error))onRejected;
- (Promise *)finally:(void (^)(void))handler;

+ (Promise *)all:(NSArray<Promise *> *)promises;
+ (Promise *)race:(NSArray<Promise *> *)promises;

@end

@implementation Promise {
    PromiseState _state;
    id _value;
    NSError *_error;
    
    NSMutableArray *_onFulfilledCallbacks;
    NSMutableArray *_onRejectedCallbacks;
    
    dispatch_queue_t _queue;
}

+ (instancetype)promiseWithBlock:(void (^)(void (^)(id), void (^)(NSError *)))block {
    Promise *promise = [[self alloc] init];
    
    void (^resolve)(id) = ^(id value) {
        dispatch_async(promise->_queue, ^{
            if (promise->_state != PromiseStatePending) return;
            
            promise->_state = PromiseStateFulfilled;
            promise->_value = value;
            
            NSArray *callbacks = [promise->_onFulfilledCallbacks copy];
            for (void (^cb)(id) in callbacks) {
                cb(value);
            }
        });
    };
    
    void (^reject)(NSError *) = ^(NSError *error) {
        dispatch_async(promise->_queue, ^{
            if (promise->_state != PromiseStatePending) return;
            
            promise->_state = PromiseStateRejected;
            promise->_error = error;
            
            NSArray *callbacks = [promise->_onRejectedCallbacks copy];
            for (void (^cb)(NSError *) in callbacks) {
                cb(error);
            }
        });
    };
    
    dispatch_async(dispatch_get_global_queue(0, 0), ^{
        block(resolve, reject);
    });
    
    return promise;
}

- (instancetype)init {
    self = [super init];
    if (self) {
        _state = PromiseStatePending;
        _onFulfilledCallbacks = [NSMutableArray array];
        _onRejectedCallbacks = [NSMutableArray array];
        _queue = dispatch_queue_create("com.promise.queue", DISPATCH_QUEUE_SERIAL);
    }
    return self;
}

- (Promise *)then:(id (^)(id))onFulfilled {
    return [Promise promiseWithBlock:^(void (^resolve)(id), void (^reject)(NSError *)) {
        dispatch_async(self->_queue, ^{
            if (self->_state == PromiseStateFulfilled) {
                id result = onFulfilled(self->_value);
                if ([result isKindOfClass:[Promise class]]) {
                    [(Promise *)result then:^id(id v) { resolve(v); return nil; }];
                } else {
                    resolve(result);
                }
            } else if (self->_state == PromiseStateRejected) {
                reject(self->_error);
            } else {
                [self->_onFulfilledCallbacks addObject:^(id value) {
                    id result = onFulfilled(value);
                    resolve(result);
                }];
                [self->_onRejectedCallbacks addObject:^(NSError *error) {
                    reject(error);
                }];
            }
        });
    }];
}

- (Promise *)catch:(void (^)(NSError *))onRejected {
    return [Promise promiseWithBlock:^(void (^resolve)(id), void (^reject)(NSError *)) {
        dispatch_async(self->_queue, ^{
            if (self->_state == PromiseStateRejected) {
                onRejected(self->_error);
                resolve(nil);
            } else if (self->_state == PromiseStateFulfilled) {
                resolve(self->_value);
            } else {
                [self->_onRejectedCallbacks addObject:^(NSError *error) {
                    onRejected(error);
                    resolve(nil);
                }];
                [self->_onFulfilledCallbacks addObject:^(id value) {
                    resolve(value);
                }];
            }
        });
    }];
}

+ (Promise *)all:(NSArray<Promise *> *)promises {
    return [Promise promiseWithBlock:^(void (^resolve)(id), void (^reject)(NSError *)) {
        NSMutableArray *results = [NSMutableArray arrayWithCapacity:promises.count];
        __block NSInteger remaining = promises.count;
        
        for (NSUInteger i = 0; i < promises.count; i++) {
            results[i] = [NSNull null]; // placeholder
            [promises[i] then:^id(id value) {
                results[i] = value ?: [NSNull null];
                remaining--;
                if (remaining == 0) resolve(results);
                return nil;
            }];
            [promises[i] catch:^(NSError *error) {
                reject(error);
            }];
        }
    }];
}

@end

// การใช้งาน
Promise *fetchUser = [Promise promiseWithBlock:^(void (^resolve)(id), void (^reject)(NSError *)) {
    [APIClient fetchUserWithCompletion:^(NSDictionary *user, NSError *error) {
        if (error) reject(error);
        else resolve(user);
    }];
}];

Promise *fetchPosts = [Promise promiseWithBlock:^(void (^resolve)(id), void (^reject)(NSError *)) {
    [APIClient fetchPostsWithCompletion:^(NSArray *posts, NSError *error) {
        if (error) reject(error);
        else resolve(posts);
    }];
}];

// รอทั้งสองเสร็จ
[[Promise all:@[fetchUser, fetchPosts]] then:^id(NSArray *results) {
    NSDictionary *user = results[0];
    NSArray *posts = results[1];
    NSLog(@"User: %@, Posts: %lu", user[@"name"], (unsigned long)posts.count);
    return nil;
}];

// Chain
[[[fetchUser then:^id(NSDictionary *user) {
    NSLog(@"Got user: %@", user[@"name"]);
    return [self fetchPostsForUser:user[@"id"]];
}] then:^id(NSArray *posts) {
    NSLog(@"Got %lu posts", (unsigned long)posts.count);
    return [self enrichPosts:posts];
}] catch:^(NSError *error) {
    NSLog(@"Error: %@", error.localizedDescription);
}];
```

---

## ส่วนที่ 9: Event Sourcing Pattern

```objc
// Event Sourcing - บันทึก events แทน state
@interface DomainEvent : NSObject

@property (nonatomic, readonly) NSString *eventId;
@property (nonatomic, readonly) NSString *eventType;
@property (nonatomic, readonly) NSDate *occurredAt;
@property (nonatomic, readonly) NSDictionary *payload;

- (instancetype)initWithType:(NSString *)type payload:(NSDictionary *)payload;

@end

@implementation DomainEvent

- (instancetype)initWithType:(NSString *)type payload:(NSDictionary *)payload {
    self = [super init];
    if (self) {
        _eventId = [[NSUUID UUID] UUIDString];
        _eventType = type;
        _occurredAt = [NSDate date];
        _payload = [payload copy];
    }
    return self;
}

@end

// Account aggregate ที่ใช้ Event Sourcing
@interface BankAccount : NSObject

@property (nonatomic, readonly) NSString *accountId;
@property (nonatomic, readonly) double balance;
@property (nonatomic, readonly) BOOL isOpen;
@property (nonatomic, readonly) NSArray<DomainEvent *> *uncommittedEvents;

- (instancetype)initWithId:(NSString *)accountId ownerName:(NSString *)name;
+ (instancetype)replayFromEvents:(NSArray<DomainEvent *> *)events;

- (BOOL)deposit:(double)amount error:(NSError **)error;
- (BOOL)withdraw:(double)amount error:(NSError **)error;
- (void)close;

@end

@implementation BankAccount {
    NSMutableArray<DomainEvent *> *_uncommittedEvents;
    NSMutableArray<DomainEvent *> *_committedEvents;
}

- (instancetype)initWithId:(NSString *)accountId ownerName:(NSString *)name {
    self = [super init];
    if (self) {
        _uncommittedEvents = [NSMutableArray array];
        _committedEvents = [NSMutableArray array];
        _accountId = accountId;
        
        // Raise creation event
        [self raiseEvent:[[DomainEvent alloc] initWithType:@"AccountOpened"
                                                   payload:@{
            @"accountId": accountId,
            @"ownerName": name
        }]];
    }
    return self;
}

+ (instancetype)replayFromEvents:(NSArray<DomainEvent *> *)events {
    BankAccount *account = [[self alloc] initEmpty];
    for (DomainEvent *event in events) {
        [account applyEvent:event isNew:NO];
    }
    return account;
}

- (void)raiseEvent:(DomainEvent *)event {
    [_uncommittedEvents addObject:event];
    [self applyEvent:event isNew:YES];
}

- (void)applyEvent:(DomainEvent *)event isNew:(BOOL)isNew {
    NSString *type = event.eventType;
    
    if ([type isEqualToString:@"AccountOpened"]) {
        _accountId = event.payload[@"accountId"];
        _balance = 0;
        _isOpen = YES;
    } else if ([type isEqualToString:@"MoneyDeposited"]) {
        _balance += [event.payload[@"amount"] doubleValue];
    } else if ([type isEqualToString:@"MoneyWithdrawn"]) {
        _balance -= [event.payload[@"amount"] doubleValue];
    } else if ([type isEqualToString:@"AccountClosed"]) {
        _isOpen = NO;
    }
    
    if (!isNew) {
        [_committedEvents addObject:event];
    }
}

- (BOOL)deposit:(double)amount error:(NSError **)error {
    if (!_isOpen) {
        if (error) *error = [NSError errorWithDomain:@"BankError" code:1
                                           userInfo:@{NSLocalizedDescriptionKey: @"Account is closed"}];
        return NO;
    }
    if (amount <= 0) {
        if (error) *error = [NSError errorWithDomain:@"BankError" code:2
                                           userInfo:@{NSLocalizedDescriptionKey: @"Amount must be positive"}];
        return NO;
    }
    
    [self raiseEvent:[[DomainEvent alloc] initWithType:@"MoneyDeposited"
                                               payload:@{@"amount": @(amount)}]];
    return YES;
}

- (BOOL)withdraw:(double)amount error:(NSError **)error {
    if (!_isOpen) {
        if (error) *error = [NSError errorWithDomain:@"BankError" code:1 userInfo:nil];
        return NO;
    }
    if (amount > _balance) {
        if (error) *error = [NSError errorWithDomain:@"BankError" code:3
                                           userInfo:@{NSLocalizedDescriptionKey: @"Insufficient funds"}];
        return NO;
    }
    
    [self raiseEvent:[[DomainEvent alloc] initWithType:@"MoneyWithdrawn"
                                               payload:@{@"amount": @(amount)}]];
    return YES;
}

- (NSArray<DomainEvent *> *)uncommittedEvents {
    return [_uncommittedEvents copy];
}

@end

// Event Store
@interface EventStore : NSObject

- (void)saveEvents:(NSArray<DomainEvent *> *)events 
        forAggregate:(NSString *)aggregateId;

- (NSArray<DomainEvent *> *)loadEventsForAggregate:(NSString *)aggregateId;

@end
```

---

## ส่วนที่ 10: CQRS Pattern

```objc
// CQRS - Command Query Responsibility Segregation
// แยก read (Query) และ write (Command) operations

// Commands
@interface CreateUserCommand : NSObject
@property (nonatomic, strong) NSString *name;
@property (nonatomic, strong) NSString *email;
@property (nonatomic, strong) NSString *password;
@end

@interface UpdateEmailCommand : NSObject
@property (nonatomic, strong) NSString *userId;
@property (nonatomic, strong) NSString *newEmail;
@end

// Command Handler
@protocol CommandHandler <NSObject>
- (BOOL)execute:(id)command error:(NSError **)error;
@end

@interface CreateUserCommandHandler : NSObject <CommandHandler>
@end

@implementation CreateUserCommandHandler

- (BOOL)execute:(CreateUserCommand *)command error:(NSError **)error {
    // Validate
    if (command.name.length == 0) {
        if (error) *error = [NSError errorWithDomain:@"UserDomain" code:1
                                           userInfo:@{NSLocalizedDescriptionKey: @"Name required"}];
        return NO;
    }
    
    // Check duplicate email
    if ([self emailExists:command.email]) {
        if (error) *error = [NSError errorWithDomain:@"UserDomain" code:2
                                           userInfo:@{NSLocalizedDescriptionKey: @"Email already exists"}];
        return NO;
    }
    
    // Create user in write model
    User *user = [User createWithName:command.name
                                email:command.email
                             password:[self hashPassword:command.password]];
    
    // Publish event for read model update
    [[EventBus shared] publish:@"UserCreated" data:@{
        @"userId": user.id,
        @"name": user.name,
        @"email": user.email
    }];
    
    return YES;
}

@end

// Queries
@interface GetUserByIdQuery : NSObject
@property (nonatomic, strong) NSString *userId;
@end

@interface UserListQuery : NSObject
@property (nonatomic, assign) NSInteger page;
@property (nonatomic, assign) NSInteger pageSize;
@property (nonatomic, strong) NSString *searchTerm;
@end

// Query Handler - อ่านจาก read model (อาจเป็น denormalized view)
@interface UserQueryHandler : NSObject

- (NSDictionary *)executeGetUser:(GetUserByIdQuery *)query;
- (NSDictionary *)executeList:(UserListQuery *)query;

@end

@implementation UserQueryHandler

- (NSDictionary *)executeGetUser:(GetUserByIdQuery *)query {
    // อ่านจาก read-optimized store
    return [self.readDatabase userById:query.userId];
}

- (NSDictionary *)executeList:(UserListQuery *)query {
    // อ่านจาก pre-computed view
    return [self.readDatabase usersPage:query.page
                               pageSize:query.pageSize
                                 search:query.searchTerm];
}

@end

// Command Bus
@interface CommandBus : NSObject

+ (instancetype)shared;

- (void)registerHandler:(id<CommandHandler>)handler 
            forCommand:(Class)commandClass;

- (BOOL)dispatch:(id)command error:(NSError **)error;

@end

@implementation CommandBus {
    NSMutableDictionary<NSString *, id<CommandHandler>> *_handlers;
}

+ (instancetype)shared {
    static CommandBus *bus = nil;
    static dispatch_once_t onceToken;
    dispatch_once(&onceToken, ^{
        bus = [[self alloc] init];
    });
    return bus;
}

- (void)registerHandler:(id<CommandHandler>)handler forCommand:(Class)cls {
    _handlers[NSStringFromClass(cls)] = handler;
}

- (BOOL)dispatch:(id)command error:(NSError **)error {
    id<CommandHandler> handler = _handlers[NSStringFromClass([command class])];
    if (!handler) {
        if (error) *error = [NSError errorWithDomain:@"CommandBus" code:404
                                           userInfo:@{NSLocalizedDescriptionKey: @"No handler found"}];
        return NO;
    }
    return [handler execute:command error:error];
}

@end
```

---

## ส่วนที่ 11: Circuit Breaker Pattern

```objc
// Circuit Breaker - ป้องกัน cascade failures
typedef NS_ENUM(NSInteger, CircuitState) {
    CircuitStateClosed,   // ปกติ - request ผ่านได้
    CircuitStateOpen,     // เปิด - block requests
    CircuitStateHalfOpen  // ทดสอบว่า service recover แล้วหรือยัง
};

@interface CircuitBreaker : NSObject

- (instancetype)initWithFailureThreshold:(NSInteger)threshold
                              timeout:(NSTimeInterval)timeout
                         successThreshold:(NSInteger)successThreshold;

- (void)executeRequest:(void (^)(void))request 
              fallback:(void (^)(NSError *))fallback;

@property (nonatomic, readonly) CircuitState state;
@property (nonatomic, readonly) NSInteger failureCount;

@end

@implementation CircuitBreaker {
    NSInteger _failureThreshold;
    NSTimeInterval _timeout;
    NSInteger _successThreshold;
    
    CircuitState _state;
    NSInteger _failureCount;
    NSInteger _successCount;
    NSDate *_lastFailureTime;
    
    dispatch_queue_t _queue;
}

- (instancetype)initWithFailureThreshold:(NSInteger)threshold
                                 timeout:(NSTimeInterval)timeout
                        successThreshold:(NSInteger)successThreshold {
    self = [super init];
    if (self) {
        _failureThreshold = threshold;
        _timeout = timeout;
        _successThreshold = successThreshold;
        _state = CircuitStateClosed;
        _queue = dispatch_queue_create("com.circuit.breaker", DISPATCH_QUEUE_SERIAL);
    }
    return self;
}

- (void)executeRequest:(void (^)(void))request fallback:(void (^)(NSError *))fallback {
    dispatch_async(_queue, ^{
        CircuitState currentState = [self currentState];
        
        if (currentState == CircuitStateOpen) {
            NSError *error = [NSError errorWithDomain:@"CircuitBreaker"
                                                 code:503
                                             userInfo:@{
                NSLocalizedDescriptionKey: @"Circuit breaker is open"
            }];
            dispatch_async(dispatch_get_main_queue(), ^{
                if (fallback) fallback(error);
            });
            return;
        }
        
        // Try request
        @try {
            dispatch_sync(dispatch_get_main_queue(), ^{
                request();
            });
            [self recordSuccess];
        } @catch (NSException *exception) {
            [self recordFailure];
            NSError *error = [NSError errorWithDomain:@"CircuitBreaker"
                                                 code:500
                                             userInfo:@{
                NSLocalizedDescriptionKey: exception.reason
            }];
            dispatch_async(dispatch_get_main_queue(), ^{
                if (fallback) fallback(error);
            });
        }
    });
}

- (CircuitState)currentState {
    if (_state == CircuitStateOpen) {
        // ตรวจสอบว่าหมดเวลา timeout แล้วหรือยัง
        if ([[NSDate date] timeIntervalSinceDate:_lastFailureTime] > _timeout) {
            _state = CircuitStateHalfOpen;
            _successCount = 0;
            NSLog(@"Circuit breaker: HALF-OPEN - testing recovery");
        }
    }
    return _state;
}

- (void)recordSuccess {
    _failureCount = 0;
    
    if (_state == CircuitStateHalfOpen) {
        _successCount++;
        if (_successCount >= _successThreshold) {
            _state = CircuitStateClosed;
            NSLog(@"Circuit breaker: CLOSED - service recovered");
        }
    }
}

- (void)recordFailure {
    _failureCount++;
    _lastFailureTime = [NSDate date];
    
    if (_state == CircuitStateHalfOpen || _failureCount >= _failureThreshold) {
        _state = CircuitStateOpen;
        NSLog(@"Circuit breaker: OPEN - failures: %ld", (long)_failureCount);
    }
}

@end

// การใช้งาน
CircuitBreaker *breaker = [[CircuitBreaker alloc] 
                            initWithFailureThreshold:5
                                            timeout:60.0
                                   successThreshold:3];

[breaker executeRequest:^{
    // Call external service
    [ExternalAPI callService];
} fallback:^(NSError *error) {
    // Use cached data or show error
    NSLog(@"Using fallback: %@", error.localizedDescription);
    [self showCachedData];
}];
```

---

## ส่วนที่ 12: Advanced Block Patterns

### Currying ใน Objective-C

```objc
// Currying - แปลง multi-arg function เป็น chain of single-arg functions

// Function pointer แบบ curried
typedef id (^CurriedBlock)(id);

CurriedBlock add(NSInteger a) {
    return ^id(NSNumber *b) {
        return @(a + [b integerValue]);
    };
}

// การใช้งาน
CurriedBlock add5 = add(5);
id result1 = add5(@3);  // 8
id result2 = add5(@10); // 15

// Curried function สำหรับ NSArray operations
typedef NSArray *(^ArrayTransform)(NSArray *);

ArrayTransform map(id (^f)(id)) {
    return ^NSArray *(NSArray *array) {
        NSMutableArray *result = [NSMutableArray arrayWithCapacity:array.count];
        for (id item in array) {
            [result addObject:f(item)];
        }
        return [result copy];
    };
}

ArrayTransform filter(BOOL (^predicate)(id)) {
    return ^NSArray *(NSArray *array) {
        NSMutableArray *result = [NSMutableArray array];
        for (id item in array) {
            if (predicate(item)) [result addObject:item];
        }
        return [result copy];
    };
}

// Function composition
ArrayTransform compose(ArrayTransform f, ArrayTransform g) {
    return ^NSArray *(NSArray *array) {
        return f(g(array)); // f(g(x))
    };
}

// การใช้งาน
NSArray *numbers = @[@1, @2, @3, @4, @5, @6, @7, @8, @9, @10];

ArrayTransform doubleAll = map(^id(NSNumber *n) {
    return @([n integerValue] * 2);
});

ArrayTransform filterEvens = filter(^BOOL(NSNumber *n) {
    return [n integerValue] % 2 == 0;
});

// Compose: กรอง evens แล้วคูณ 2
ArrayTransform doubleEvens = compose(doubleAll, filterEvens);

NSArray *result = doubleEvens(numbers);
NSLog(@"Result: %@", result); // [4, 8, 12, 16, 20]
```

---

### Memoization

```objc
// Cache function results
typedef id (^MemoizableBlock)(id);

MemoizableBlock memoize(MemoizableBlock block) {
    NSMutableDictionary *cache = [NSMutableDictionary dictionary];
    
    return ^id(id arg) {
        id cached = cache[arg];
        if (cached) {
            NSLog(@"Cache hit for: %@", arg);
            return cached;
        }
        
        id result = block(arg);
        cache[arg] = result;
        return result;
    };
}

// การใช้งาน
__block MemoizableBlock memoFib;
memoFib = memoize(^id(NSNumber *n) {
    NSInteger num = [n integerValue];
    if (num <= 1) return n;
    return @([memoFib(@(num - 1)) integerValue] + 
             [memoFib(@(num - 2)) integerValue]);
});

NSLog(@"%@", memoFib(@40)); // เร็วมากเพราะ memoized
```

---

### Throttle และ Debounce

```objc
// Throttle - จำกัดความถี่การ call
typedef void (^ThrottledBlock)(void);

ThrottledBlock throttle(void (^block)(void), NSTimeInterval interval) {
    __block BOOL canFire = YES;
    
    return ^{
        if (!canFire) return;
        canFire = NO;
        block();
        dispatch_after(dispatch_time(DISPATCH_TIME_NOW, (int64_t)(interval * NSEC_PER_SEC)),
                      dispatch_get_main_queue(), ^{
            canFire = YES;
        });
    };
}

// Debounce - รอจน stop calling แล้วจึง execute
ThrottledBlock debounce(void (^block)(void), NSTimeInterval delay) {
    __block dispatch_block_t pendingBlock = nil;
    
    return ^{
        if (pendingBlock) {
            dispatch_block_cancel(pendingBlock);
        }
        
        pendingBlock = dispatch_block_create(0, ^{
            block();
            pendingBlock = nil;
        });
        
        dispatch_after(dispatch_time(DISPATCH_TIME_NOW, (int64_t)(delay * NSEC_PER_SEC)),
                      dispatch_get_main_queue(), pendingBlock);
    };
}

// การใช้งาน
ThrottledBlock throttledSearch = throttle(^{
    [self performSearch];
}, 0.5);

ThrottledBlock debouncedSave = debounce(^{
    [self saveDocument];
}, 1.0);

// ใน text field delegate
- (void)textFieldDidChange {
    throttledSearch(); // fire ทุก 0.5 วินาที
    debouncedSave();   // save เมื่อหยุดพิมพ์ 1 วินาที
}
```

---

## ส่วนที่ 13: Reactive Extensions Manual Implementation

```objc
// BehaviorSubject - เหมือน Subject แต่ emit current value ให้ subscriber ใหม่
@interface BehaviorSubject : Subject

- (instancetype)initWithValue:(id)initialValue;
@property (nonatomic, readonly) id currentValue;

@end

@implementation BehaviorSubject {
    id _currentValue;
}

- (instancetype)initWithValue:(id)initialValue {
    self = [super init];
    if (self) {
        _currentValue = initialValue;
    }
    return self;
}

- (id)subscribe:(void (^)(id))next error:(void (^)(NSError *))error complete:(void (^)(void))complete {
    // Emit current value immediately
    if (next && _currentValue) {
        next(_currentValue);
    }
    return [super subscribe:next error:error complete:complete];
}

- (void)next:(id)value {
    _currentValue = value;
    [super next:value];
}

@end

// ReplaySubject - replay N last values
@interface ReplaySubject : Subject

- (instancetype)initWithBufferSize:(NSInteger)bufferSize;

@end

@implementation ReplaySubject {
    NSInteger _bufferSize;
    NSMutableArray *_buffer;
}

- (instancetype)initWithBufferSize:(NSInteger)bufferSize {
    self = [super init];
    if (self) {
        _bufferSize = bufferSize;
        _buffer = [NSMutableArray array];
    }
    return self;
}

- (id)subscribe:(void (^)(id))next error:(void (^)(NSError *))error complete:(void (^)(void))complete {
    // Replay buffered values
    NSArray *buffered = [_buffer copy];
    for (id value in buffered) {
        if (next) next(value);
    }
    return [super subscribe:next error:error complete:complete];
}

- (void)next:(id)value {
    [_buffer addObject:value];
    if (_buffer.count > _bufferSize) {
        [_buffer removeObjectAtIndex:0];
    }
    [super next:value];
}

@end

// Operators เพิ่มเติม
@implementation Observable (Operators)

// Combine latest values from multiple observables
+ (Observable *)combineLatest:(NSArray<Observable *> *)observables {
    return [Observable create:^(void (^next)(id), void (^error)(NSError *), void (^complete)(void)) {
        NSMutableArray *latestValues = [NSMutableArray arrayWithCapacity:observables.count];
        NSMutableSet *received = [NSMutableSet set];
        
        for (NSUInteger i = 0; i < observables.count; i++) {
            latestValues[i] = [NSNull null];
            NSUInteger idx = i;
            
            [observables[i] subscribe:^(id value) {
                latestValues[idx] = value ?: [NSNull null];
                [received addObject:@(idx)];
                
                if (received.count == observables.count) {
                    next([latestValues copy]);
                }
            } error:error complete:nil];
        }
    }];
}

// Merge multiple observables
+ (Observable *)merge:(NSArray<Observable *> *)observables {
    return [Observable create:^(void (^next)(id), void (^error)(NSError *), void (^complete)(void)) {
        __block NSInteger remaining = observables.count;
        
        for (Observable *obs in observables) {
            [obs subscribe:^(id value) {
                next(value);
            } error:error complete:^{
                remaining--;
                if (remaining == 0) complete();
            }];
        }
    }];
}

// Zip - pair values from multiple observables
+ (Observable *)zip:(NSArray<Observable *> *)observables {
    return [Observable create:^(void (^next)(id), void (^error)(NSError *), void (^complete)(void)) {
        NSMutableArray *queues = [NSMutableArray array];
        for (NSUInteger i = 0; i < observables.count; i++) {
            queues[i] = [NSMutableArray array];
        }
        
        for (NSUInteger i = 0; i < observables.count; i++) {
            NSUInteger idx = i;
            [observables[i] subscribe:^(id value) {
                [queues[idx] addObject:value ?: [NSNull null]];
                
                // Check if all queues have values
                BOOL allHaveValues = YES;
                for (NSMutableArray *q in queues) {
                    if (q.count == 0) { allHaveValues = NO; break; }
                }
                
                if (allHaveValues) {
                    NSMutableArray *combined = [NSMutableArray array];
                    for (NSMutableArray *q in queues) {
                        [combined addObject:q.firstObject];
                        [q removeObjectAtIndex:0];
                    }
                    next([combined copy]);
                }
            } error:error complete:nil];
        }
    }];
}

// switchMap - cancel previous inner observable
- (Observable *)switchMap:(Observable *(^)(id))project {
    return [Observable create:^(void (^next)(id), void (^error)(NSError *), void (^complete)(void)) {
        __block id currentSubscription = nil;
        
        [self subscribe:^(id value) {
            // Cancel previous
            if (currentSubscription) [currentSubscription dispose];
            
            Observable *inner = project(value);
            currentSubscription = [inner subscribe:next error:error complete:nil];
        } error:error complete:complete];
    }];
}

@end
```

---

## ส่วนที่ 14: Advanced Memory Management Patterns

```objc
// Object Pool - reuse expensive objects
@interface ObjectPool<T : NSObject *> : NSObject

- (instancetype)initWithFactory:(T (^)(void))factory capacity:(NSInteger)capacity;
- (T)acquireObject;
- (void)releaseObject:(T)object;

@end

@implementation ObjectPool {
    NSMutableArray *_availableObjects;
    NSMutableSet *_inUseObjects;
    id (^_factory)(void);
    NSInteger _capacity;
    dispatch_semaphore_t _semaphore;
    dispatch_queue_t _queue;
}

- (instancetype)initWithFactory:(id (^)(void))factory capacity:(NSInteger)capacity {
    self = [super init];
    if (self) {
        _factory = [factory copy];
        _capacity = capacity;
        _availableObjects = [NSMutableArray array];
        _inUseObjects = [NSMutableSet set];
        _semaphore = dispatch_semaphore_create(capacity);
        _queue = dispatch_queue_create("com.pool.queue", DISPATCH_QUEUE_SERIAL);
        
        // Pre-create objects
        for (NSInteger i = 0; i < capacity; i++) {
            [_availableObjects addObject:_factory()];
        }
    }
    return self;
}

- (id)acquireObject {
    dispatch_semaphore_wait(_semaphore, DISPATCH_TIME_FOREVER);
    
    __block id obj;
    dispatch_sync(_queue, ^{
        obj = self->_availableObjects.lastObject;
        [self->_availableObjects removeLastObject];
        [self->_inUseObjects addObject:obj];
    });
    
    return obj;
}

- (void)releaseObject:(id)object {
    dispatch_async(_queue, ^{
        [self->_inUseObjects removeObject:object];
        [self->_availableObjects addObject:object];
        dispatch_semaphore_signal(self->_semaphore);
    });
}

@end

// การใช้งาน
ObjectPool *connectionPool = [[ObjectPool alloc]
                               initWithFactory:^id{
    return [[DatabaseConnection alloc] init];
} capacity:10];

dispatch_async(queue, ^{
    DatabaseConnection *conn = [connectionPool acquireObject];
    
    @try {
        [conn executeQuery:@"SELECT * FROM users"];
    } @finally {
        [connectionPool releaseObject:conn]; // คืน pool เสมอ
    }
});
```

---

## แบบฝึกหัด

### แบบฝึกหัดที่ 1: Implement Task Queue
```objc
// สร้าง task queue ที่:
// 1. รอง concurrent tasks จำกัด N tasks
// 2. มี priority (high, normal, low)
// 3. สามารถ cancel task ได้
// 4. มี callback เมื่อ queue empty

@interface PriorityTaskQueue : NSObject

- (instancetype)initWithMaxConcurrency:(NSInteger)maxConcurrency;

- (NSString *)addTask:(void (^)(void (^completion)(void)))task 
             priority:(NSInteger)priority; // returns task ID

- (void)cancelTask:(NSString *)taskId;
- (void)setQueueEmptyHandler:(void (^)(void))handler;

@end
```

### แบบฝึกหัดที่ 2: Implement Simple DI Container
```objc
// สร้าง Dependency Injection Container ที่:
// 1. ลงทะเบียน services ด้วย protocol
// 2. Resolve dependencies อัตโนมัติ
// 3. Support singleton และ transient lifetime
// 4. Detect circular dependencies

@interface DIContainer : NSObject

- (void)registerSingleton:(Class)cls forProtocol:(Protocol *)protocol;
- (void)registerTransient:(Class)cls forProtocol:(Protocol *)protocol;
- (id)resolve:(Protocol *)protocol;

@end
```

### แบบฝึกหัดที่ 3: Implement Undo/Redo System
```objc
// สร้าง Undo/Redo system ที่:
// 1. Record commands
// 2. Undo last command
// 3. Redo undone command
// 4. Clear history

@protocol Command <NSObject>
- (void)execute;
- (void)undo;
@optional
- (NSString *)description;
@end

@interface UndoRedoManager : NSObject
- (void)executeCommand:(id<Command>)command;
- (BOOL)undo;
- (BOOL)redo;
- (void)clearHistory;
@property (nonatomic, readonly) BOOL canUndo;
@property (nonatomic, readonly) BOOL canRedo;
@end
```

---

## สรุป

บทนี้ครอบคลุม advanced patterns ที่ช่วยให้โค้ด Objective-C มีคุณภาพสูง:

1. **Method Swizzling** - เปลี่ยน behavior ตอน runtime อย่างปลอดภัย
2. **Associated Objects** - เพิ่ม state ให้ existing classes ผ่าน categories
3. **NSProxy** - สร้าง proxy objects ที่ยืดหยุ่น
4. **Message Forwarding** - chain of responsibility pattern
5. **Dynamic Subclassing** - สร้าง classes ตอน runtime
6. **AOP** - cross-cutting concerns อย่าง logging, security
7. **Reactive Programming** - Observable pattern ด้วยมือ
8. **Promises** - จัดการ async code อย่างสวยงาม
9. **Event Sourcing** - บันทึก events แทน state
10. **Circuit Breaker** - ป้องกัน cascade failures
11. **Currying** - functional programming ใน ObjC

---

*จบตอนที่ 98 - Advanced Patterns และเทคนิคขั้นสูงใน Objective-C*
