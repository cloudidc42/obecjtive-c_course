# Part 75: Performance Optimization ใน Objective-C

## บทนำ

Performance เป็นหนึ่งในปัจจัยสำคัญที่สุดที่กำหนดความสำเร็จของแอปบน iOS ผู้ใช้คาดหวังแอปที่ตอบสนองรวดเร็ว ไม่กินแบตเตอรี่มาก และใช้หน่วยความจำอย่างมีประสิทธิภาพ บทนี้จะพาคุณเรียนรู้เครื่องมือและเทคนิคต่าง ๆ เพื่อวิเคราะห์และปรับปรุง performance ของแอป Objective-C

---

## 75.1 Instruments Tool Suite

Instruments คือ performance analysis tool ที่มาพร้อมกับ Xcode ช่วยในการ profile แอปและค้นหา performance bottlenecks

### เข้าถึง Instruments

```
วิธีที่ 1: Xcode > Product > Profile (⌘I)
วิธีที่ 2: Xcode > Open Developer Tool > Instruments
วิธีที่ 3: Terminal: open -a Instruments
```

### เลือก Instrument ที่เหมาะสม

| ปัญหา | Instrument ที่ใช้ |
|-------|-----------------|
| แอปช้า | Time Profiler |
| Memory leak | Leaks |
| Memory ใช้มาก | Allocations |
| CPU ใช้สูง | CPU Profiler |
| GPU ปัญหา | GPU Frame Debugger |
| Network ช้า | Network |
| แบตหมดเร็ว | Energy Log |
| Core Data ช้า | Core Data |

---

## 75.2 Time Profiler

Time Profiler ช่วยระบุว่า code ส่วนไหนใช้ CPU time มากที่สุด

### การใช้ Time Profiler

1. เปิด Instruments
2. เลือก "Time Profiler"
3. เลือก device/simulator และ process
4. กด Record
5. ทำ operations ที่ต้องการ profile
6. กด Stop
7. วิเคราะห์ Call Tree

### อ่านผลลัพธ์ Time Profiler

```
Call Tree แสดง:
- Self (ms): เวลาที่ method นี้ใช้โดยตรง
- Total (ms): เวลาทั้งหมดรวมถึง methods ที่เรียก
- %: เปอร์เซ็นต์ของ total time

ตัวเลือก:
- Invert Call Tree: แสดง hot paths ที่สุด
- Separate by Thread: แยกตาม thread
- Hide System Libraries: ซ่อน system frameworks
```

### ตัวอย่างการ optimize หลังจาก profile

```objc
// Before optimization - ช้ามาก
- (NSArray *)findUsersNearLocation:(CLLocation *)location 
                         inRadius:(double)radiusKm {
    NSMutableArray *nearby = [NSMutableArray array];
    
    for (User *user in self.allUsers) {
        // ❌ สร้าง CLLocation ทุก iteration - expensive!
        CLLocation *userLocation = [[CLLocation alloc] 
                                    initWithLatitude:user.latitude 
                                           longitude:user.longitude];
        
        // ❌ เรียก distanceFromLocation ทุกครั้ง - O(n)
        double distance = [location distanceFromLocation:userLocation] / 1000.0;
        
        if (distance <= radiusKm) {
            [nearby addObject:user];
        }
    }
    
    return [nearby copy];
}

// After optimization - เร็วขึ้นมาก
- (NSArray *)findUsersNearLocation:(CLLocation *)location 
                         inRadius:(double)radiusKm {
    // ✅ คำนวณ bounding box ก่อน เพื่อกรอง candidates
    double latDelta = radiusKm / 111.32; // 1 degree latitude ≈ 111.32 km
    double lonDelta = radiusKm / (111.32 * cos(location.coordinate.latitude * M_PI / 180));
    
    double minLat = location.coordinate.latitude - latDelta;
    double maxLat = location.coordinate.latitude + latDelta;
    double minLon = location.coordinate.longitude - lonDelta;
    double maxLon = location.coordinate.longitude + lonDelta;
    
    NSMutableArray *nearby = [NSMutableArray array];
    
    // ✅ กรอง candidates ด้วย bounding box ก่อน (ถูกกว่า CLLocation)
    NSArray *candidates = [self.allUsers filteredArrayUsingPredicate:
                           [NSPredicate predicateWithFormat:
                            @"latitude >= %@ AND latitude <= %@ AND longitude >= %@ AND longitude <= %@",
                            @(minLat), @(maxLat), @(minLon), @(maxLon)]];
    
    // ✅ คำนวณ exact distance เฉพาะ candidates
    for (User *user in candidates) {
        CLLocation *userLocation = [[CLLocation alloc] initWithLatitude:user.latitude 
                                                              longitude:user.longitude];
        double distance = [location distanceFromLocation:userLocation] / 1000.0;
        
        if (distance <= radiusKm) {
            [nearby addObject:user];
        }
    }
    
    return [nearby copy];
}
```

---

## 75.3 Memory Debugging

### Instruments Allocations

Allocations instrument ช่วยดูว่า memory ถูก allocate ที่ไหนและเท่าไร

```objc
// ตัวอย่างการ optimize memory allocation

// ❌ ไม่ดี: สร้าง objects มากเกินไปใน loop
- (void)processLargeDataset:(NSArray *)data {
    NSMutableArray *results = [NSMutableArray array];
    
    for (NSDictionary *item in data) {
        // สร้าง NSDateFormatter ทุก iteration - expensive!
        NSDateFormatter *formatter = [[NSDateFormatter alloc] init];
        formatter.dateFormat = @"yyyy-MM-dd HH:mm:ss";
        
        NSDate *date = [formatter dateFromString:item[@"date"]];
        NSString *formatted = [formatter stringFromDate:date];
        
        [results addObject:formatted];
    }
}

// ✅ ดี: สร้าง formatter ครั้งเดียว, reuse
- (void)processLargeDataset:(NSArray *)data {
    // สร้างครั้งเดียว ใช้ซ้ำ
    NSDateFormatter *formatter = [[NSDateFormatter alloc] init];
    formatter.dateFormat = @"yyyy-MM-dd HH:mm:ss";
    
    NSMutableArray *results = [NSMutableArray array];
    
    for (NSDictionary *item in data) {
        NSDate *date = [formatter dateFromString:item[@"date"]];
        NSString *formatted = [formatter stringFromDate:date];
        [results addObject:formatted];
    }
}
```

### Memory Pressure Handling

```objc
@interface DataCache : NSObject

@property (nonatomic, strong) NSMutableDictionary *cache;

@end

@implementation DataCache

- (instancetype)init {
    self = [super init];
    if (self) {
        _cache = [NSMutableDictionary dictionary];
        
        // ตอบสนองต่อ memory warnings
        [[NSNotificationCenter defaultCenter] 
            addObserver:self
               selector:@selector(handleMemoryWarning:)
                   name:UIApplicationDidReceiveMemoryWarningNotification
                 object:nil];
    }
    return self;
}

- (void)handleMemoryWarning:(NSNotification *)notification {
    // ล้าง cache เมื่อ memory ต่ำ
    [self.cache removeAllObjects];
    NSLog(@"Cache cleared due to memory warning");
}

- (void)dealloc {
    [[NSNotificationCenter defaultCenter] removeObserver:self];
}

@end

// ใน ViewController
- (void)didReceiveMemoryWarning {
    [super didReceiveMemoryWarning];
    
    // ล้าง resources ที่ recreate ได้
    self.thumbnailCache = nil;
    self.cachedData = nil;
    
    // ล้าง views ที่ไม่แสดงอยู่
    if (!self.isViewLoaded || !self.view.window) {
        self.view = nil; // ระวัง! ทำเฉพาะเมื่อ view ไม่แสดง
    }
}
```

---

## 75.4 Leaks Instrument

Leaks ช่วยค้นหา memory leaks ในแอป

### ประเภทของ Memory Leaks

```objc
// Leak ชนิดที่ 1: Retain Cycle (ปัญหาหลักใน ARC)

// ❌ Retain cycle ระหว่าง parent และ child
@interface Parent : NSObject
@property (nonatomic, strong) Child *child; // Strong reference
@end

@interface Child : NSObject
@property (nonatomic, strong) Parent *parent; // Strong reference - CYCLE!
@end

// ✅ แก้ด้วย weak reference
@interface Child : NSObject
@property (nonatomic, weak) Parent *parent; // Weak reference - no cycle
@end

// Leak ชนิดที่ 2: Block Retain Cycle

// ❌ ปัญหา: self retain block, block retain self
@implementation ViewController

- (void)viewDidLoad {
    [super viewDidLoad];
    
    // self retain dataLoader (strong property)
    // block capture self strongly (retain cycle!)
    self.dataLoader.onComplete = ^(NSArray *data) {
        [self displayData:data]; // self captured strongly
    };
}

@end

// ✅ แก้ด้วย weak-strong dance
- (void)viewDidLoad {
    [super viewDidLoad];
    
    __weak typeof(self) weakSelf = self;
    self.dataLoader.onComplete = ^(NSArray *data) {
        __strong typeof(weakSelf) strongSelf = weakSelf;
        if (strongSelf) {
            [strongSelf displayData:data];
        }
    };
}

// Leak ชนิดที่ 3: NSTimer
// ❌ Timer retain self strongly
@implementation AnimationController

- (void)startAnimation {
    // Timer retain self, self retain timer - CYCLE!
    self.timer = [NSTimer scheduledTimerWithTimeInterval:0.1
                                                  target:self
                                                selector:@selector(updateAnimation)
                                                userInfo:nil
                                                 repeats:YES];
}

// ✅ ใช้ block-based timer (iOS 10+)
- (void)startAnimation {
    __weak typeof(self) weakSelf = self;
    self.timer = [NSTimer scheduledTimerWithTimeInterval:0.1
                                                 repeats:YES
                                                   block:^(NSTimer *timer) {
        __strong typeof(weakSelf) strongSelf = weakSelf;
        [strongSelf updateAnimation];
    }];
}

@end
```

### ตรวจสอบ Retain Cycle ด้วย Debug Memory Graph

1. Run แอปใน Xcode
2. กดปุ่ม "Debug Memory Graph" ใน debug bar
3. ดู cycles ที่เส้นสีแดง/ส้ม

```objc
// เพิ่ม description สำหรับ debugging
@implementation User

- (NSString *)debugDescription {
    return [NSString stringWithFormat:@"<User: %p | ID: %@ | Name: %@>", 
            self, self.userId, self.name];
}

@end
```

---

## 75.5 CPU Profiling

### ลด CPU Usage

```objc
// ❌ การคำนวณที่ใช้ CPU มากใน main thread
- (void)viewDidLoad {
    [super viewDidLoad];
    
    // ❌ ทำบน main thread - ทำให้ UI หยุด!
    NSArray *processedData = [self processLargeData:self.rawData];
    [self.tableView reloadData];
}

// ✅ ทำงาน heavy บน background thread
- (void)viewDidLoad {
    [super viewDidLoad];
    
    // ✅ ทำ processing บน background
    dispatch_async(dispatch_get_global_queue(DISPATCH_QUEUE_PRIORITY_DEFAULT, 0), ^{
        NSArray *processedData = [self processLargeData:self.rawData];
        
        // ✅ update UI บน main thread เท่านั้น
        dispatch_async(dispatch_get_main_queue(), ^{
            self.tableData = processedData;
            [self.tableView reloadData];
        });
    });
}

// Pattern สำหรับ CPU-intensive tasks
- (void)performExpensiveOperation:(void(^)(NSArray *results))completion {
    dispatch_queue_t bgQueue = dispatch_queue_create("com.app.processing", 
                                                      DISPATCH_QUEUE_CONCURRENT);
    
    dispatch_async(bgQueue, ^{
        // CPU-intensive work
        NSMutableArray *results = [NSMutableArray array];
        
        for (NSInteger i = 0; i < 1000000; i++) {
            // Heavy computation
            [results addObject:@(i * i)];
        }
        
        dispatch_async(dispatch_get_main_queue(), ^{
            completion([results copy]);
        });
    });
}
```

### NSOperation สำหรับ Complex CPU Tasks

```objc
// Custom operation สำหรับ image processing
@interface ImageProcessingOperation : NSOperation

@property (nonatomic, strong) UIImage *inputImage;
@property (nonatomic, strong) UIImage *outputImage;
@property (nonatomic, copy) void(^completionHandler)(UIImage *);

@end

@implementation ImageProcessingOperation

- (void)main {
    if (self.cancelled) return;
    
    // Heavy image processing
    CIImage *ciImage = [CIImage imageWithCGImage:self.inputImage.CGImage];
    CIFilter *filter = [CIFilter filterWithName:@"CIColorControls"];
    [filter setValue:ciImage forKey:kCIInputImageKey];
    [filter setValue:@(1.5) forKey:kCIInputSaturationKey];
    [filter setValue:@(0.5) forKey:kCIInputBrightnessKey];
    
    if (self.cancelled) return;
    
    CIContext *context = [CIContext contextWithOptions:nil];
    CGImageRef outputCGImage = [context createCGImage:filter.outputImage 
                                             fromRect:filter.outputImage.extent];
    
    self.outputImage = [UIImage imageWithCGImage:outputCGImage];
    CGImageRelease(outputCGImage);
    
    if (!self.cancelled && self.completionHandler) {
        dispatch_async(dispatch_get_main_queue(), ^{
            self.completionHandler(self.outputImage);
        });
    }
}

@end

// ใช้งาน NSOperationQueue
NSOperationQueue *processingQueue = [[NSOperationQueue alloc] init];
processingQueue.maxConcurrentOperationCount = 2;
processingQueue.qualityOfService = NSQualityOfServiceUserInitiated;

ImageProcessingOperation *op = [[ImageProcessingOperation alloc] init];
op.inputImage = rawImage;
op.completionHandler = ^(UIImage *processedImage) {
    self.imageView.image = processedImage;
};

[processingQueue addOperation:op];
```

---

## 75.6 GPU Profiling

### Xcode GPU Frame Debugger

```
เปิด GPU Frame Debugger:
Debug > Capture GPU Frame (ขณะ running)

หรือกดปุ่ม camera icon ใน debug bar
```

### ลด Drawing Overhead

```objc
// ❌ ไม่ดี: วาด shadow ในทุก cell - ช้ามาก
@implementation ProductCell

- (void)setupUI {
    // Shadow บน main layer - GPU expensive!
    self.productImageView.layer.shadowColor = [UIColor blackColor].CGColor;
    self.productImageView.layer.shadowOpacity = 0.5;
    self.productImageView.layer.shadowOffset = CGSizeMake(0, 2);
    self.productImageView.layer.shadowRadius = 4;
    // ไม่มี shadowPath - iOS ต้องคำนวณทุกครั้ง!
}

@end

// ✅ ดี: กำหนด shadowPath เพื่อ cache shape
@implementation ProductCell

- (void)layoutSubviews {
    [super layoutSubviews];
    
    self.productImageView.layer.shadowColor = [UIColor blackColor].CGColor;
    self.productImageView.layer.shadowOpacity = 0.5;
    self.productImageView.layer.shadowOffset = CGSizeMake(0, 2);
    self.productImageView.layer.shadowRadius = 4;
    
    // ✅ กำหนด shadowPath เพื่อหลีกเลี่ยง offscreen rendering
    self.productImageView.layer.shadowPath = 
        [UIBezierPath bezierPathWithRect:self.productImageView.bounds].CGPath;
    
    // ✅ shouldRasterize สำหรับ static content ที่ไม่เปลี่ยน
    self.productImageView.layer.shouldRasterize = YES;
    self.productImageView.layer.rasterizationScale = [UIScreen mainScreen].scale;
}

@end

// ✅ ดียิ่งกว่า: วาด shadow เป็น pre-rendered image
@interface ShadowView : UIView
@end

@implementation ShadowView

- (void)drawRect:(CGRect)rect {
    CGContextRef context = UIGraphicsGetCurrentContext();
    
    // วาด shadow ลง context โดยตรง
    CGContextSetShadowWithColor(context, 
                                CGSizeMake(0, 2), 
                                4, 
                                [UIColor colorWithWhite:0 alpha:0.3].CGColor);
    
    // วาด content
    [[UIColor whiteColor] setFill];
    UIBezierPath *path = [UIBezierPath bezierPathWithRoundedRect:rect cornerRadius:8];
    [path fill];
}

@end
```

### Offscreen Rendering

```objc
// ❌ Offscreen rendering triggers - ช้ามาก
view.layer.cornerRadius = 10;
view.layer.masksToBounds = YES; // Offscreen rendering!

view.layer.mask = someLayer; // Offscreen rendering!

view.layer.allowsEdgeAntialiasing = YES; // อาจ trigger offscreen rendering

// ✅ หลีกเลี่ยง offscreen rendering
// วิธีที่ 1: ใช้ CALayer.cornerRadius โดยตรง (iOS 13+)
view.layer.cornerRadius = 10;
view.layer.cornerCurve = kCACornerCurveContinuous;
// ไม่ต้อง masksToBounds ถ้าไม่มี content เกิน bounds

// วิธีที่ 2: วาด rounded corners ใน drawRect:
- (void)drawRect:(CGRect)rect {
    UIBezierPath *path = [UIBezierPath bezierPathWithRoundedRect:rect cornerRadius:10];
    [path addClip];
    
    // Draw content
    [self.image drawInRect:rect];
}

// วิธีที่ 3: ใช้ image mask แทน layer mask
- (UIImage *)roundedImageFromImage:(UIImage *)image radius:(CGFloat)radius {
    CGRect rect = CGRectMake(0, 0, image.size.width, image.size.height);
    
    UIGraphicsBeginImageContextWithOptions(rect.size, NO, 0);
    UIBezierPath *path = [UIBezierPath bezierPathWithRoundedRect:rect cornerRadius:radius];
    [path addClip];
    [image drawInRect:rect];
    UIImage *roundedImage = UIGraphicsGetImageFromCurrentImageContext();
    UIGraphicsEndImageContext();
    
    return roundedImage;
}
```

---

## 75.7 Network Profiling

### Network Instrument

```objc
// เพิ่ม network debugging
@interface NetworkLogger : NSObject

+ (void)logRequest:(NSURLRequest *)request;
+ (void)logResponse:(NSURLResponse *)response data:(NSData *)data time:(NSTimeInterval)time;

@end

@implementation NetworkLogger

+ (void)logRequest:(NSURLRequest *)request {
    NSLog(@"[Network] → %@ %@", request.HTTPMethod, request.URL);
    if (request.HTTPBody) {
        NSLog(@"[Network] Body: %@", [[NSString alloc] initWithData:request.HTTPBody 
                                                           encoding:NSUTF8StringEncoding]);
    }
}

+ (void)logResponse:(NSURLResponse *)response data:(NSData *)data time:(NSTimeInterval)time {
    NSHTTPURLResponse *httpResponse = (NSHTTPURLResponse *)response;
    NSLog(@"[Network] ← %ld %@ (%.0fms, %@ bytes)", 
          (long)httpResponse.statusCode,
          response.URL,
          time * 1000,
          @(data.length));
}

@end

// Intercepting requests ด้วย URLProtocol
@interface NetworkMonitorProtocol : NSURLProtocol
@end

@implementation NetworkMonitorProtocol

+ (BOOL)canInitWithRequest:(NSURLRequest *)request {
    return [NSURLProtocol propertyForKey:@"Handled" inRequest:request] == nil;
}

+ (NSURLRequest *)canonicalRequestForRequest:(NSURLRequest *)request {
    return request;
}

- (void)startLoading {
    NSMutableURLRequest *newRequest = [self.request mutableCopy];
    [NSURLProtocol setProperty:@YES forKey:@"Handled" inRequest:newRequest];
    
    NSDate *startTime = [NSDate date];
    [NetworkLogger logRequest:newRequest];
    
    NSURLSession *session = [NSURLSession sessionWithConfiguration:[NSURLSessionConfiguration defaultSessionConfiguration]
                                                          delegate:nil
                                                     delegateQueue:nil];
    
    [[session dataTaskWithRequest:newRequest completionHandler:^(NSData *data, NSURLResponse *response, NSError *error) {
        NSTimeInterval elapsed = [[NSDate date] timeIntervalSinceDate:startTime];
        [NetworkLogger logResponse:response data:data time:elapsed];
        
        if (error) {
            [self.client URLProtocol:self didFailWithError:error];
        } else {
            [self.client URLProtocol:self didReceiveResponse:response 
                  cacheStoragePolicy:NSURLCacheStorageAllowed];
            [self.client URLProtocol:self didLoadData:data];
            [self.client URLProtocolDidFinishLoading:self];
        }
    }] resume];
}

@end
```

### Network Optimization

```objc
// Image caching ลด network requests
@interface ImageCache : NSObject

+ (instancetype)sharedCache;
- (void)imageForURL:(NSURL *)url completion:(void(^)(UIImage *image))completion;

@end

@implementation ImageCache {
    NSCache *_memoryCache;
    NSString *_diskCachePath;
    dispatch_queue_t _diskQueue;
}

+ (instancetype)sharedCache {
    static ImageCache *instance;
    static dispatch_once_t onceToken;
    dispatch_once(&onceToken, ^{
        instance = [[ImageCache alloc] init];
    });
    return instance;
}

- (instancetype)init {
    self = [super init];
    if (self) {
        _memoryCache = [[NSCache alloc] init];
        _memoryCache.countLimit = 100;
        _memoryCache.totalCostLimit = 50 * 1024 * 1024; // 50MB
        
        NSString *cacheDir = [NSSearchPathForDirectoriesInDomains(NSCachesDirectory, NSUserDomainMask, YES) firstObject];
        _diskCachePath = [cacheDir stringByAppendingPathComponent:@"ImageCache"];
        [[NSFileManager defaultManager] createDirectoryAtPath:_diskCachePath
                                  withIntermediateDirectories:YES
                                                   attributes:nil
                                                        error:nil];
        
        _diskQueue = dispatch_queue_create("com.app.imagecache.disk", DISPATCH_QUEUE_SERIAL);
    }
    return self;
}

- (void)imageForURL:(NSURL *)url completion:(void(^)(UIImage *image))completion {
    NSString *key = [url absoluteString];
    
    // 1. ตรวจสอบ memory cache
    UIImage *cached = [_memoryCache objectForKey:key];
    if (cached) {
        dispatch_async(dispatch_get_main_queue(), ^{
            completion(cached);
        });
        return;
    }
    
    // 2. ตรวจสอบ disk cache
    dispatch_async(_diskQueue, ^{
        NSString *diskPath = [self diskPathForKey:key];
        UIImage *diskImage = [UIImage imageWithContentsOfFile:diskPath];
        
        if (diskImage) {
            // บันทึกใน memory cache
            [self->_memoryCache setObject:diskImage forKey:key];
            dispatch_async(dispatch_get_main_queue(), ^{
                completion(diskImage);
            });
            return;
        }
        
        // 3. ดาวน์โหลดจาก network
        [[[NSURLSession sharedSession] dataTaskWithURL:url 
                                     completionHandler:^(NSData *data, NSURLResponse *response, NSError *error) {
            if (error || !data) {
                dispatch_async(dispatch_get_main_queue(), ^{
                    completion(nil);
                });
                return;
            }
            
            UIImage *image = [UIImage imageWithData:data];
            if (!image) {
                dispatch_async(dispatch_get_main_queue(), ^{
                    completion(nil);
                });
                return;
            }
            
            // บันทึก memory cache
            [self->_memoryCache setObject:image forKey:key];
            
            // บันทึก disk cache
            dispatch_async(self->_diskQueue, ^{
                [data writeToFile:diskPath atomically:YES];
            });
            
            dispatch_async(dispatch_get_main_queue(), ^{
                completion(image);
            });
        }] resume];
    });
}

- (NSString *)diskPathForKey:(NSString *)key {
    // Hash key เพื่อใช้เป็น filename
    NSUInteger hash = key.hash;
    NSString *filename = [NSString stringWithFormat:@"%lu.cache", (unsigned long)hash];
    return [_diskCachePath stringByAppendingPathComponent:filename];
}

@end
```

---

## 75.8 NSCache

NSCache เป็น collection ที่ออกแบบมาสำหรับ caching objects

### ใช้งาน NSCache

```objc
@interface DataManager : NSObject

@property (nonatomic, strong) NSCache *dataCache;

@end

@implementation DataManager

- (instancetype)init {
    self = [super init];
    if (self) {
        _dataCache = [[NSCache alloc] init];
        
        // กำหนด limits
        _dataCache.countLimit = 200;          // จำนวน objects สูงสุด
        _dataCache.totalCostLimit = 10 * 1024 * 1024; // 10MB
        
        // Delegate สำหรับรับ notification เมื่อ object ถูก evict
        _dataCache.delegate = self;
        _dataCache.name = @"DataManager.dataCache";
    }
    return self;
}

- (void)cacheData:(NSData *)data 
           forKey:(NSString *)key {
    // Cost = ขนาดข้อมูลใน bytes
    [_dataCache setObject:data 
                   forKey:key 
                     cost:data.length];
}

- (NSData *)cachedDataForKey:(NSString *)key {
    return [_dataCache objectForKey:key];
}

// NSCacheDelegate method
- (void)cache:(NSCache *)cache willEvictObject:(id)obj {
    NSLog(@"Cache evicting object, size: %lu bytes", 
          (unsigned long)[(NSData *)obj length]);
}

@end

// Typed Cache Wrapper
@interface TypedCache<KeyType, ObjectType> : NSObject

- (void)setObject:(ObjectType)object forKey:(KeyType)key;
- (void)setObject:(ObjectType)object forKey:(KeyType)key cost:(NSUInteger)cost;
- (ObjectType)objectForKey:(KeyType)key;
- (void)removeObjectForKey:(KeyType)key;
- (void)removeAllObjects;

@property (nonatomic, assign) NSUInteger countLimit;
@property (nonatomic, assign) NSUInteger totalCostLimit;

@end

@implementation TypedCache {
    NSCache *_cache;
}

- (instancetype)init {
    self = [super init];
    if (self) {
        _cache = [[NSCache alloc] init];
    }
    return self;
}

- (void)setObject:(id)object forKey:(id)key {
    [_cache setObject:object forKey:key];
}

- (id)objectForKey:(id)key {
    return [_cache objectForKey:key];
}

// ... ฯลฯ

@end
```

---

## 75.9 Lazy Loading

Lazy loading ช่วยลด startup time และ memory usage

```objc
@interface UserProfile : NSObject

@property (nonatomic, copy) NSString *userId;
@property (nonatomic, copy) NSString *name;
@property (nonatomic, copy) NSString *email;

// Lazy-loaded expensive properties
@property (nonatomic, readonly) UIImage *avatar;
@property (nonatomic, readonly) NSArray<Post *> *posts;
@property (nonatomic, readonly) NSArray<Friend *> *friends;

@end

@implementation UserProfile {
    UIImage *_avatar;
    NSArray<Post *> *_posts;
    NSArray<Friend *> *_friends;
}

// ✅ Lazy load avatar
- (UIImage *)avatar {
    if (!_avatar) {
        // โหลด avatar เมื่อต้องการเท่านั้น
        NSURL *avatarURL = [NSURL URLWithString:self.avatarURLString];
        NSData *imageData = [NSData dataWithContentsOfURL:avatarURL];
        _avatar = [UIImage imageWithData:imageData];
    }
    return _avatar;
}

// ✅ Lazy load posts
- (NSArray<Post *> *)posts {
    if (!_posts) {
        _posts = [PostRepository postsForUserId:self.userId];
    }
    return _posts;
}

- (void)clearCachedData {
    _avatar = nil;
    _posts = nil;
    _friends = nil;
}

@end

// Lazy initialization ใน ViewController
@interface ProductListViewController : UIViewController

@end

@implementation ProductListViewController {
    NSArray *_products;
    UISearchController *_searchController;
}

// ✅ Lazy load search controller
- (UISearchController *)searchController {
    if (!_searchController) {
        _searchController = [[UISearchController alloc] initWithSearchResultsController:nil];
        _searchController.searchResultsUpdater = self;
        _searchController.obscuresBackgroundDuringPresentation = NO;
    }
    return _searchController;
}

- (void)viewDidLoad {
    [super viewDidLoad];
    
    // ✅ ไม่ต้อง load search controller ถ้าผู้ใช้ไม่ได้ใช้ search
    // searchController จะถูกสร้างเมื่อ user tap search
}

- (IBAction)searchButtonTapped:(id)sender {
    // สร้าง searchController ตอนนี้เท่านั้น
    self.navigationItem.searchController = self.searchController;
    [self.searchController.searchBar becomeFirstResponder];
}

@end
```

---

## 75.10 Table View and Collection View Optimization

Table/Collection View เป็น component ที่ใช้บ่อยที่สุดและมักเป็น bottleneck

### Cell Reuse

```objc
@implementation ProductListViewController

- (void)viewDidLoad {
    [super viewDidLoad];
    
    // ✅ ลงทะเบียน cell class
    [self.tableView registerClass:[ProductCell class] 
              forCellReuseIdentifier:@"ProductCell"];
    
    // หรือ register nib
    UINib *nib = [UINib nibWithNibName:@"ProductCell" bundle:nil];
    [self.tableView registerNib:nib forCellReuseIdentifier:@"ProductCell"];
}

- (UITableViewCell *)tableView:(UITableView *)tableView 
         cellForRowAtIndexPath:(NSIndexPath *)indexPath {
    // ✅ ใช้ dequeueReusableCellWithIdentifier เสมอ
    ProductCell *cell = [tableView dequeueReusableCellWithIdentifier:@"ProductCell" 
                                                        forIndexPath:indexPath];
    
    Product *product = self.products[indexPath.row];
    [cell configureWithProduct:product];
    
    return cell;
}

@end
```

### Prefetch Data

```objc
@interface ProductListViewController () <UITableViewDataSourcePrefetching>
@end

@implementation ProductListViewController

- (void)viewDidLoad {
    [super viewDidLoad];
    self.tableView.prefetchDataSource = self;
}

// เรียกก่อนที่ cells จะแสดง
- (void)tableView:(UITableView *)tableView 
   prefetchRowsAtIndexPaths:(NSArray<NSIndexPath *> *)indexPaths {
    for (NSIndexPath *indexPath in indexPaths) {
        Product *product = self.products[indexPath.row];
        
        // Pre-load images
        if (!product.isImageLoaded) {
            [[ImageCache sharedCache] preloadImageForURL:product.imageURL];
        }
        
        // Pre-fetch related data
        if (!product.isDetailLoaded) {
            [[ProductAPI shared] prefetchDetailsForProduct:product.productId];
        }
    }
}

// ยกเลิก prefetch เมื่อ cells ไม่จำเป็นอีกต่อไป
- (void)tableView:(UITableView *)tableView 
   cancelPrefetchingForRowsAtIndexPaths:(NSArray<NSIndexPath *> *)indexPaths {
    for (NSIndexPath *indexPath in indexPaths) {
        Product *product = self.products[indexPath.row];
        [[ImageCache sharedCache] cancelPreloadForURL:product.imageURL];
    }
}

@end
```

### เพิ่ม Performance ด้วย Pre-calculated Heights

```objc
@interface ProductListViewController ()

@property (nonatomic, strong) NSMutableDictionary<NSIndexPath *, NSNumber *> *heightCache;

@end

@implementation ProductListViewController

- (instancetype)init {
    self = [super init];
    if (self) {
        _heightCache = [NSMutableDictionary dictionary];
    }
    return self;
}

- (CGFloat)tableView:(UITableView *)tableView 
   heightForRowAtIndexPath:(NSIndexPath *)indexPath {
    // ✅ ใช้ cached height ถ้ามี
    NSNumber *cachedHeight = self.heightCache[indexPath];
    if (cachedHeight) {
        return cachedHeight.floatValue;
    }
    
    // คำนวณ height
    Product *product = self.products[indexPath.row];
    CGFloat height = [self calculateHeightForProduct:product width:tableView.bounds.size.width];
    
    // Cache ผลลัพธ์
    self.heightCache[indexPath] = @(height);
    
    return height;
}

- (CGFloat)calculateHeightForProduct:(Product *)product width:(CGFloat)width {
    // คำนวณ dynamic height
    CGFloat baseHeight = 80.0;
    
    // เพิ่ม height สำหรับ description ถ้ายาว
    if (product.description.length > 100) {
        NSString *desc = product.description;
        CGSize maxSize = CGSizeMake(width - 32, CGFLOAT_MAX);
        CGRect rect = [desc boundingRectWithSize:maxSize
                                         options:NSStringDrawingUsesLineFragmentOrigin
                                      attributes:@{NSFontAttributeName: [UIFont systemFontOfSize:14]}
                                         context:nil];
        baseHeight += rect.size.height + 8;
    }
    
    return baseHeight;
}

// ล้าง height cache เมื่อ data เปลี่ยน
- (void)reloadProducts {
    [self.heightCache removeAllObjects];
    self.products = [ProductRepository loadProducts];
    [self.tableView reloadData];
}

@end
```

---

## 75.11 Image Optimization

### ลด Image Memory Footprint

```objc
@implementation ImageOptimizer

// ✅ Decode image บน background thread เพื่อหลีกเลี่ยง main thread stutter
+ (void)decodeImageAsync:(UIImage *)image 
              completion:(void(^)(UIImage *decodedImage))completion {
    dispatch_async(dispatch_get_global_queue(DISPATCH_QUEUE_PRIORITY_DEFAULT, 0), ^{
        // Force decode image data
        UIGraphicsBeginImageContextWithOptions(image.size, NO, image.scale);
        [image drawAtPoint:CGPointZero];
        UIImage *decodedImage = UIGraphicsGetImageFromCurrentImageContext();
        UIGraphicsEndImageContext();
        
        dispatch_async(dispatch_get_main_queue(), ^{
            completion(decodedImage);
        });
    });
}

// ✅ Scale image ให้พอดีกับ size ที่แสดง
+ (UIImage *)scaledImage:(UIImage *)image toSize:(CGSize)targetSize {
    if (CGSizeEqualToSize(image.size, targetSize)) {
        return image;
    }
    
    UIGraphicsBeginImageContextWithOptions(targetSize, NO, 0.0);
    [image drawInRect:CGRectMake(0, 0, targetSize.width, targetSize.height)];
    UIImage *scaledImage = UIGraphicsGetImageFromCurrentImageContext();
    UIGraphicsEndImageContext();
    
    return scaledImage;
}

// ✅ Thumbnail จาก large image
+ (UIImage *)thumbnailFromImage:(UIImage *)image size:(CGSize)thumbnailSize {
    // คำนวณ aspect-fit size
    CGFloat widthRatio = thumbnailSize.width / image.size.width;
    CGFloat heightRatio = thumbnailSize.height / image.size.height;
    CGFloat ratio = MIN(widthRatio, heightRatio);
    
    CGSize scaledSize = CGSizeMake(image.size.width * ratio, image.size.height * ratio);
    
    return [self scaledImage:image toSize:scaledSize];
}

@end

// ใช้ ImageIO สำหรับ large images (ประหยัด memory มากกว่า)
+ (UIImage *)thumbnailFromFileURL:(NSURL *)fileURL size:(CGSize)thumbnailSize {
    NSDictionary *options = @{
        (NSString *)kCGImageSourceCreateThumbnailFromImageIfAbsent: @YES,
        (NSString *)kCGImageSourceThumbnailMaxPixelSize: @(MAX(thumbnailSize.width, thumbnailSize.height)),
        (NSString *)kCGImageSourceCreateThumbnailWithTransform: @YES,
    };
    
    CGImageSourceRef source = CGImageSourceCreateWithURL((__bridge CFURLRef)fileURL, nil);
    if (!source) return nil;
    
    CGImageRef thumbnailCGImage = CGImageSourceCreateThumbnailAtIndex(source, 0, (__bridge CFDictionaryRef)options);
    CFRelease(source);
    
    if (!thumbnailCGImage) return nil;
    
    UIImage *thumbnail = [UIImage imageWithCGImage:thumbnailCGImage];
    CGImageRelease(thumbnailCGImage);
    
    return thumbnail;
}
```

---

## 75.12 Background Processing

```objc
@interface BackgroundTaskManager : NSObject

+ (instancetype)sharedManager;
- (UIBackgroundTaskIdentifier)beginBackgroundTask:(void(^)(void))expirationHandler;
- (void)endBackgroundTask:(UIBackgroundTaskIdentifier)taskId;

@end

@implementation BackgroundTaskManager

+ (instancetype)sharedManager {
    static BackgroundTaskManager *instance;
    static dispatch_once_t onceToken;
    dispatch_once(&onceToken, ^{
        instance = [[BackgroundTaskManager alloc] init];
    });
    return instance;
}

- (UIBackgroundTaskIdentifier)beginBackgroundTask:(void(^)(void))expirationHandler {
    return [[UIApplication sharedApplication] beginBackgroundTaskWithExpirationHandler:^{
        if (expirationHandler) {
            expirationHandler();
        }
    }];
}

- (void)endBackgroundTask:(UIBackgroundTaskIdentifier)taskId {
    if (taskId != UIBackgroundTaskInvalid) {
        [[UIApplication sharedApplication] endBackgroundTask:taskId];
    }
}

@end

// การใช้งาน
@implementation SyncService

- (void)startSync {
    __block UIBackgroundTaskIdentifier taskId = 
        [[BackgroundTaskManager sharedManager] beginBackgroundTask:^{
            // Expiration handler - ยกเลิก sync อย่างสวยงาม
            [self cancelSync];
            [[BackgroundTaskManager sharedManager] endBackgroundTask:taskId];
            taskId = UIBackgroundTaskInvalid;
        }];
    
    dispatch_async(dispatch_get_global_queue(DISPATCH_QUEUE_PRIORITY_DEFAULT, 0), ^{
        [self performSync];
        
        dispatch_async(dispatch_get_main_queue(), ^{
            [[BackgroundTaskManager sharedManager] endBackgroundTask:taskId];
            taskId = UIBackgroundTaskInvalid;
        });
    });
}

- (void)performSync {
    // Sync operations
    NSArray *pendingItems = [self.database fetchPendingSyncItems];
    
    for (SyncItem *item in pendingItems) {
        [self syncItem:item];
        
        if (self.isCancelled) {
            break;
        }
    }
}

@end
```

---

## 75.13 Auto Layout Performance

Auto Layout อาจเป็น bottleneck ถ้าใช้ไม่ถูกวิธี

```objc
@implementation OptimizedViewController

- (void)viewDidLoad {
    [super viewDidLoad];
    
    // ✅ ลด constraint updates ด้วย batch update
    [NSLayoutConstraint activateConstraints:@[
        [self.imageView.topAnchor constraintEqualToAnchor:self.view.topAnchor constant:20],
        [self.imageView.leadingAnchor constraintEqualToAnchor:self.view.leadingAnchor constant:16],
        [self.imageView.widthAnchor constraintEqualToConstant:100],
        [self.imageView.heightAnchor constraintEqualToConstant:100],
        
        [self.titleLabel.topAnchor constraintEqualToAnchor:self.imageView.topAnchor],
        [self.titleLabel.leadingAnchor constraintEqualToAnchor:self.imageView.trailingAnchor constant:12],
        [self.titleLabel.trailingAnchor constraintEqualToAnchor:self.view.trailingAnchor constant:-16],
    ]];
}

// ✅ Cache constraint references สำหรับการอัพเดต
@interface DynamicView : UIView

@property (nonatomic, strong) NSLayoutConstraint *heightConstraint;

@end

@implementation DynamicView

- (void)setupConstraints {
    self.heightConstraint = [self.heightAnchor constraintEqualToConstant:100];
    [NSLayoutConstraint activateConstraints:@[self.heightConstraint, ...]];
}

- (void)updateHeight:(CGFloat)newHeight {
    // ✅ อัพเดต constant แทนการ add/remove constraints
    self.heightConstraint.constant = newHeight;
    
    // ✅ Animate constraint change
    [UIView animateWithDuration:0.3 animations:^{
        [self layoutIfNeeded];
    }];
}

@end
```

---

## 75.14 Allocations Instrument

```objc
// วิเคราะห์ memory allocations

// ❌ String ที่สร้างมากเกินไป
- (NSString *)buildComplexString:(NSArray *)items {
    NSString *result = @"";
    for (NSString *item in items) {
        // ❌ สร้าง NSString ใหม่ทุก iteration
        result = [result stringByAppendingString:item];
        result = [result stringByAppendingString:@", "];
    }
    return result;
}

// ✅ ใช้ NSMutableString
- (NSString *)buildComplexString:(NSArray *)items {
    NSMutableString *result = [NSMutableString string];
    for (NSString *item in items) {
        [result appendString:item];
        [result appendString:@", "];
    }
    // ลบ trailing comma
    if (result.length >= 2) {
        [result deleteCharactersInRange:NSMakeRange(result.length - 2, 2)];
    }
    return [result copy];
}

// ✅ ดีที่สุด: ใช้ componentsJoinedByString:
- (NSString *)buildComplexString:(NSArray *)items {
    return [items componentsJoinedByString:@", "];
}
```

---

## 75.15 Energy Log

```objc
// ลด energy usage
@implementation EfficientLocationManager

- (void)startTracking {
    self.locationManager = [[CLLocationManager alloc] init];
    self.locationManager.delegate = self;
    
    // ✅ ใช้ accuracy ที่เหมาะสม - ไม่ต้องใช้ kCLLocationAccuracyBest เสมอ
    self.locationManager.desiredAccuracy = kCLLocationAccuracyHundredMeters;
    
    // ✅ กำหนด distance filter เพื่อลด updates
    self.locationManager.distanceFilter = 50; // Update เมื่อเดิน 50 เมตรเท่านั้น
    
    [self.locationManager startUpdatingLocation];
}

- (void)pauseTracking {
    // ✅ หยุด location updates เมื่อไม่ต้องการ
    [self.locationManager stopUpdatingLocation];
}

- (void)locationManager:(CLLocationManager *)manager 
     didUpdateLocations:(NSArray<CLLocation *> *)locations {
    CLLocation *location = locations.lastObject;
    
    // Process location...
    
    // ✅ หยุด updates หลังจากได้ location ที่ดีพอ (ถ้าต้องการ one-time)
    if (location.horizontalAccuracy < 100) {
        [manager stopUpdatingLocation];
    }
}

@end

// Batch network requests เพื่อลด radio wake-ups
@interface RequestBatcher : NSObject

- (void)addRequest:(NSURLRequest *)request 
        completion:(void(^)(NSData *data, NSError *error))completion;
- (void)flush;

@end

@implementation RequestBatcher {
    NSMutableArray *_pendingRequests;
    NSTimer *_flushTimer;
}

- (void)addRequest:(NSURLRequest *)request 
        completion:(void(^)(NSData *, NSError *))completion {
    [_pendingRequests addObject:@{@"request": request, @"completion": completion}];
    
    // ✅ รวม requests ให้ flush พร้อมกัน ลด radio wake-ups
    if (!_flushTimer) {
        _flushTimer = [NSTimer scheduledTimerWithTimeInterval:1.0
                                                       target:self
                                                     selector:@selector(flush)
                                                     userInfo:nil
                                                      repeats:NO];
    }
}

- (void)flush {
    [_flushTimer invalidate];
    _flushTimer = nil;
    
    NSArray *requests = [_pendingRequests copy];
    [_pendingRequests removeAllObjects];
    
    // Execute all pending requests
    for (NSDictionary *item in requests) {
        NSURLRequest *request = item[@"request"];
        void(^completion)(NSData *, NSError *) = item[@"completion"];
        
        [[[NSURLSession sharedSession] dataTaskWithRequest:request 
                                        completionHandler:^(NSData *data, NSURLResponse *response, NSError *error) {
            completion(data, error);
        }] resume];
    }
}

@end
```

---

## 75.16 Performance Tips สรุป

```objc
// 1. ✅ Reuse objects แทนการสร้างใหม่
static NSDateFormatter *sharedFormatter;
+ (void)initialize {
    sharedFormatter = [[NSDateFormatter alloc] init];
    sharedFormatter.dateFormat = @"yyyy-MM-dd";
}

// 2. ✅ ใช้ dispatch_once สำหรับ singletons
+ (instancetype)sharedInstance {
    static id instance;
    static dispatch_once_t token;
    dispatch_once(&token, ^{ instance = [[self alloc] init]; });
    return instance;
}

// 3. ✅ Avoid unnecessary object creation
// ❌ Bad
NSArray *result = [array filteredArrayUsingPredicate:pred];
if (result.count > 0) { ... }

// ✅ Good
BOOL exists = [array indexOfObjectPassingTest:^BOOL(id obj, NSUInteger idx, BOOL *stop) {
    return [pred evaluateWithObject:obj];
}] != NSNotFound;

// 4. ✅ String formatting
// ❌ Bad (multiple allocations)
NSString *s = [@"Hello " stringByAppendingString:name];
s = [s stringByAppendingString:@"!"];

// ✅ Good
NSString *s = [NSString stringWithFormat:@"Hello %@!", name];

// 5. ✅ Fast enumeration แทน index-based
// ❌ Slower
for (NSInteger i = 0; i < array.count; i++) {
    id item = array[i];
}

// ✅ Faster
for (id item in array) { ... }

// 6. ✅ Block enumeration สำหรับ concurrent processing
[array enumerateObjectsWithOptions:NSEnumerationConcurrent 
                        usingBlock:^(id obj, NSUInteger idx, BOOL *stop) {
    // Process in parallel
}];
```

---

## 75.17 แบบฝึกหัด

### แบบฝึกหัดที่ 1: Profile และ Optimize

ใช้ Time Profiler วิเคราะห์แอปที่ให้มาและหา 3 bottlenecks หลัก จากนั้น optimize โดยวัดผลก่อนและหลัง

### แบบฝึกหัดที่ 2: แก้ Memory Leak

```objc
// หา และแก้ memory leaks ในโค้ดนี้
@interface DataViewController : UIViewController
@property (nonatomic, strong) NSTimer *refreshTimer;
@property (nonatomic, strong) DataLoader *dataLoader;
@end

@implementation DataViewController

- (void)viewDidLoad {
    self.dataLoader = [[DataLoader alloc] init];
    
    // BUG: Retain cycle ที่ไหน?
    self.refreshTimer = [NSTimer scheduledTimerWithTimeInterval:5.0
                                                         target:self
                                                       selector:@selector(refresh)
                                                       userInfo:nil
                                                        repeats:YES];
    
    self.dataLoader.onDataLoaded = ^(NSArray *data) {
        [self updateUI:data]; // BUG: Retain cycle?
    };
}

@end
```

### แบบฝึกหัดที่ 3: Table View Optimization

Optimize table view ที่มี 10,000 cells ให้ scroll ราบเรียบ:
- ใช้ cell reuse
- Pre-calculate heights
- Lazy load images
- Prefetch data

### แบบฝึกหัดที่ 4: Image Processing

สร้าง image processing pipeline ที่:
- ประมวลผลบน background thread
- Cache ผลลัพธ์ใน NSCache
- ยกเลิกได้เมื่อ cell ถูก reuse
- แสดง placeholder ระหว่างโหลด

---

## สรุป

Performance Optimization เป็นกระบวนการต่อเนื่องที่ต้องทำอย่างมีระเบียบ:

1. **วัดก่อน optimize**: ใช้ Instruments หา actual bottlenecks
2. **หา low-hanging fruits**: memory leaks, main thread blocking
3. **ปรับปรุงเป็นส่วน ๆ**: optimize ทีละ component
4. **วัดผลหลัง optimize**: ยืนยันว่าดีขึ้นจริง
5. **Testing**: ตรวจสอบว่า optimization ไม่ทำให้ behavior เปลี่ยน

Key principles:
- **Lazy loading**: โหลดเมื่อต้องการ
- **Caching**: เก็บผลลัพธ์ที่ expensive
- **Background processing**: ทำงาน heavy บน background thread
- **Reuse objects**: หลีกเลี่ยงการสร้าง objects บ่อย ๆ
- **Reduce drawing**: ลด offscreen rendering และ overdraw

ในบทต่อไป (Part 76) เราจะเรียนรู้เรื่อง Debugging อย่างละเอียด ซึ่งเป็นทักษะสำคัญในการค้นหาและแก้ปัญหา

---

*จบ Part 75 - Performance Optimization*
