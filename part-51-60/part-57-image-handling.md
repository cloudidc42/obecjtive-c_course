# ส่วนที่ 57: การจัดการรูปภาพ (Image Handling) ใน Objective-C

## บทนำ

การจัดการรูปภาพเป็นส่วนสำคัญในการพัฒนาแอปพลิเคชัน iOS การเข้าใจวิธีโหลด แสดงผล แก้ไข และจัดเก็บรูปภาพจะช่วยให้คุณสร้างแอปที่มีประสิทธิภาพและประสบการณ์ผู้ใช้ที่ดีได้

## 1. UIImage - การโหลดรูปภาพ

### 1.1 โหลดจาก Bundle (App Resources)

`UIImage` คือคลาสหลักสำหรับจัดการรูปภาพใน iOS การโหลดรูปภาพจาก bundle เป็นวิธีที่พบบ่อยที่สุด เนื่องจากรูปภาพถูกรวมอยู่ใน app bundle แล้ว

```objc
// โหลดรูปภาพพื้นฐานจาก bundle
UIImage *image = [UIImage imageNamed:@"logo"];

// โหลดพร้อมระบุ bundle และ traitCollection
UIImage *image2 = [UIImage imageNamed:@"logo" 
                             inBundle:[NSBundle mainBundle] 
        compatibleWithTraitCollection:nil];

// ตรวจสอบว่าโหลดสำเร็จหรือไม่
if (image) {
    NSLog(@"โหลดรูปภาพสำเร็จ ขนาด: %.0f x %.0f", image.size.width, image.size.height);
} else {
    NSLog(@"ไม่พบรูปภาพในชื่อนั้น");
}
```

**ข้อดีของ `imageNamed:`**
- มีระบบ cache อัตโนมัติ
- รองรับ @2x, @3x สำหรับ Retina display
- รองรับ Asset Catalog

### 1.2 โหลดจาก File Path

```objc
// โหลดจาก Documents directory
NSString *documentsPath = [NSSearchPathForDirectoriesInDomains(
    NSDocumentDirectory, NSUserDomainMask, YES) firstObject];
NSString *imagePath = [documentsPath stringByAppendingPathComponent:@"photo.jpg"];

UIImage *imageFromFile = [UIImage imageWithContentsOfFile:imagePath];

if (imageFromFile) {
    NSLog(@"โหลดรูปภาพจากไฟล์สำเร็จ");
} else {
    NSLog(@"ไม่พบไฟล์ที่: %@", imagePath);
}

// โหลดจาก Bundle โดยใช้ path โดยตรง
NSString *bundlePath = [[NSBundle mainBundle] pathForResource:@"background" 
                                                       ofType:@"png"];
UIImage *bundleImage = [UIImage imageWithContentsOfFile:bundlePath];
```

### 1.3 โหลดจาก NSData

```objc
// โหลดจาก data (เช่น ดาวน์โหลดจาก network)
NSData *imageData = [NSData dataWithContentsOfFile:imagePath];
UIImage *imageFromData = [UIImage imageWithData:imageData];

// กำหนด scale factor
UIImage *imageWithScale = [UIImage imageWithData:imageData scale:2.0];
```

### 1.4 โหลดจาก URL (Synchronous - ไม่แนะนำ)

```objc
// วิธีนี้ block main thread - ใช้เพื่อการศึกษาเท่านั้น
NSURL *url = [NSURL URLWithString:@"https://example.com/image.jpg"];
NSData *data = [NSData dataWithContentsOfURL:url];
UIImage *urlImage = [UIImage imageWithData:data];
```

---

## 2. UIImageView - การแสดงผลรูปภาพ

### 2.1 สร้างและตั้งค่า UIImageView

```objc
// สร้าง UIImageView แบบ programmatic
UIImageView *imageView = [[UIImageView alloc] initWithFrame:CGRectMake(0, 0, 200, 200)];
imageView.image = [UIImage imageNamed:@"photo"];

// หรือสร้างพร้อมรูปภาพ
UIImageView *imageView2 = [[UIImageView alloc] initWithImage:[UIImage imageNamed:@"photo"]];

// เพิ่มเข้า view hierarchy
[self.view addSubview:imageView];
```

### 2.2 การตั้งค่าขั้นสูง

```objc
UIImageView *imageView = [[UIImageView alloc] initWithFrame:CGRectMake(20, 100, 300, 200)];
imageView.image = [UIImage imageNamed:@"landscape"];

// ตั้งค่า content mode
imageView.contentMode = UIViewContentModeScaleAspectFit;

// ตั้งค่า background color
imageView.backgroundColor = [UIColor lightGrayColor];

// เพิ่ม corner radius
imageView.layer.cornerRadius = 12.0;
imageView.clipsToBounds = YES;

// เพิ่ม border
imageView.layer.borderWidth = 2.0;
imageView.layer.borderColor = [UIColor blueColor].CGColor;

// เปิด user interaction
imageView.userInteractionEnabled = YES;

[self.view addSubview:imageView];
```

---

## 3. Image Scaling Modes (Content Modes)

### 3.1 ประเภทของ Content Mode

```objc
UIImageView *demoView = [[UIImageView alloc] initWithFrame:CGRectMake(0, 0, 200, 200)];

// ---- Scaling Modes ----

// Scale ให้พอดีกับขนาด view โดยรักษา aspect ratio
// รูปภาพจะเห็นทั้งหมด อาจมีพื้นที่ว่างด้านข้าง
demoView.contentMode = UIViewContentModeScaleAspectFit;

// Scale ให้เต็ม view โดยรักษา aspect ratio
// รูปภาพอาจถูกตัดด้านที่เกิน
demoView.contentMode = UIViewContentModeScaleAspectFill;

// Stretch รูปภาพให้เต็ม view (ไม่รักษา aspect ratio)
demoView.contentMode = UIViewContentModeScaleToFill; // default

// ---- Position Modes (ไม่ scale) ----

demoView.contentMode = UIViewContentModeCenter;      // กึ่งกลาง
demoView.contentMode = UIViewContentModeTop;         // บนกึ่งกลาง
demoView.contentMode = UIViewContentModeBottom;      // ล่างกึ่งกลาง
demoView.contentMode = UIViewContentModeLeft;        // ซ้ายกึ่งกลาง
demoView.contentMode = UIViewContentModeRight;       // ขวากึ่งกลาง
demoView.contentMode = UIViewContentModeTopLeft;     // มุมบนซ้าย
demoView.contentMode = UIViewContentModeTopRight;    // มุมบนขวา
demoView.contentMode = UIViewContentModeBottomLeft;  // มุมล่างซ้าย
demoView.contentMode = UIViewContentModeBottomRight; // มุมล่างขวา

// Redraw - วาดรูปใหม่เมื่อ bounds เปลี่ยน
demoView.contentMode = UIViewContentModeRedraw;
```

### 3.2 ตัวอย่างเปรียบเทียบ

```objc
- (void)demonstrateContentModes {
    NSArray *modes = @[
        @(UIViewContentModeScaleAspectFit),
        @(UIViewContentModeScaleAspectFill),
        @(UIViewContentModeScaleToFill),
        @(UIViewContentModeCenter)
    ];
    
    NSArray *labels = @[@"AspectFit", @"AspectFill", @"ScaleToFill", @"Center"];
    
    for (NSInteger i = 0; i < modes.count; i++) {
        UIImageView *iv = [[UIImageView alloc] initWithFrame:
            CGRectMake(10 + (i % 2) * 180, 100 + (i / 2) * 220, 160, 200)];
        iv.image = [UIImage imageNamed:@"sample"];
        iv.contentMode = [modes[i] integerValue];
        iv.backgroundColor = [UIColor systemGray5Color];
        iv.clipsToBounds = YES;
        
        UILabel *label = [[UILabel alloc] initWithFrame:
            CGRectMake(iv.frame.origin.x, iv.frame.origin.y - 20, 160, 20)];
        label.text = labels[i];
        label.font = [UIFont systemFontOfSize:12];
        label.textAlignment = NSTextAlignmentCenter;
        
        [self.view addSubview:iv];
        [self.view addSubview:label];
    }
}
```

---

## 4. Image Rendering Modes

### 4.1 Template vs Original

```objc
UIImage *originalImage = [UIImage imageNamed:@"icon"];

// Original - แสดงสีตามต้นฉบับ
UIImage *originalMode = [originalImage imageWithRenderingMode:UIImageRenderingModeAlwaysOriginal];

// Template - ใช้สี tintColor แทนสีต้นฉบับ (ดีสำหรับ icon)
UIImage *templateMode = [originalImage imageWithRenderingMode:UIImageRenderingModeAlwaysTemplate];

// Automatic - iOS ตัดสินใจเอง (default)
UIImage *autoMode = [originalImage imageWithRenderingMode:UIImageRenderingModeAutomatic];

// ใช้ template mode กับ UIImageView
UIImageView *iconView = [[UIImageView alloc] initWithImage:templateMode];
iconView.tintColor = [UIColor systemBlueColor]; // เปลี่ยนสี icon ได้ง่าย
```

### 4.2 Dynamic Tint Color

```objc
- (void)setupIconWithTintColor {
    UIImage *settingsIcon = [[UIImage imageNamed:@"gear"] 
                             imageWithRenderingMode:UIImageRenderingModeAlwaysTemplate];
    
    UIImageView *iconView = [[UIImageView alloc] initWithFrame:CGRectMake(50, 200, 44, 44)];
    iconView.image = settingsIcon;
    iconView.tintColor = [UIColor systemPurpleColor];
    
    [self.view addSubview:iconView];
    
    // เปลี่ยน tintColor ด้วย animation
    dispatch_after(dispatch_time(DISPATCH_TIME_NOW, (int64_t)(2.0 * NSEC_PER_SEC)), 
                   dispatch_get_main_queue(), ^{
        [UIView animateWithDuration:0.5 animations:^{
            iconView.tintColor = [UIColor systemOrangeColor];
        }];
    });
}
```

---

## 5. Drawing Images Programmatically

### 5.1 วาดรูปภาพบน CGContext

```objc
- (UIImage *)createCustomImage {
    CGSize size = CGSizeMake(200, 200);
    UIGraphicsBeginImageContextWithOptions(size, NO, 0.0);
    
    CGContextRef context = UIGraphicsGetCurrentContext();
    
    // วาดพื้นหลังสีน้ำเงิน
    CGContextSetFillColorWithColor(context, [UIColor systemBlueColor].CGColor);
    CGContextFillRect(context, CGRectMake(0, 0, 200, 200));
    
    // วาดวงกลมสีขาว
    CGContextSetFillColorWithColor(context, [UIColor whiteColor].CGColor);
    CGContextFillEllipseInRect(context, CGRectMake(50, 50, 100, 100));
    
    // วาดข้อความ
    NSDictionary *attrs = @{
        NSFontAttributeName: [UIFont boldSystemFontOfSize:24],
        NSForegroundColorAttributeName: [UIColor systemBlueColor]
    };
    [@"Hello" drawInRect:CGRectMake(60, 85, 80, 30) withAttributes:attrs];
    
    UIImage *result = UIGraphicsGetImageFromCurrentImageContext();
    UIGraphicsEndImageContext();
    
    return result;
}
```

### 5.2 วาดรูปภาพบนรูปภาพอื่น

```objc
- (UIImage *)addWatermarkToImage:(UIImage *)original watermarkText:(NSString *)text {
    UIGraphicsBeginImageContextWithOptions(original.size, NO, original.scale);
    
    // วาดรูปต้นฉบับ
    [original drawAtPoint:CGPointZero];
    
    // ตั้งค่าข้อความ watermark
    NSDictionary *attrs = @{
        NSFontAttributeName: [UIFont systemFontOfSize:20],
        NSForegroundColorAttributeName: [[UIColor whiteColor] colorWithAlphaComponent:0.7],
        NSBackgroundColorAttributeName: [[UIColor blackColor] colorWithAlphaComponent:0.3]
    };
    
    CGSize textSize = [text sizeWithAttributes:attrs];
    CGRect textRect = CGRectMake(
        original.size.width - textSize.width - 10,
        original.size.height - textSize.height - 10,
        textSize.width,
        textSize.height
    );
    
    [text drawInRect:textRect withAttributes:attrs];
    
    UIImage *watermarked = UIGraphicsGetImageFromCurrentImageContext();
    UIGraphicsEndImageContext();
    
    return watermarked;
}
```

---

## 6. Image Resizing

### 6.1 Resize ด้วย UIGraphics

```objc
- (UIImage *)resizeImage:(UIImage *)image toSize:(CGSize)newSize {
    UIGraphicsBeginImageContextWithOptions(newSize, NO, 0.0);
    [image drawInRect:CGRectMake(0, 0, newSize.width, newSize.height)];
    UIImage *resized = UIGraphicsGetImageFromCurrentImageContext();
    UIGraphicsEndImageContext();
    return resized;
}

// ใช้งาน
UIImage *large = [UIImage imageNamed:@"large_photo"];
UIImage *thumbnail = [self resizeImage:large toSize:CGSizeMake(100, 100)];
```

### 6.2 Resize โดยรักษา Aspect Ratio

```objc
- (UIImage *)resizeImage:(UIImage *)image fitInSize:(CGSize)maxSize {
    CGFloat widthRatio = maxSize.width / image.size.width;
    CGFloat heightRatio = maxSize.height / image.size.height;
    CGFloat ratio = MIN(widthRatio, heightRatio);
    
    CGSize newSize = CGSizeMake(
        image.size.width * ratio,
        image.size.height * ratio
    );
    
    UIGraphicsBeginImageContextWithOptions(newSize, NO, 0.0);
    [image drawInRect:CGRectMake(0, 0, newSize.width, newSize.height)];
    UIImage *resized = UIGraphicsGetImageFromCurrentImageContext();
    UIGraphicsEndImageContext();
    
    return resized;
}
```

### 6.3 Crop รูปภาพ

```objc
- (UIImage *)cropImage:(UIImage *)image toRect:(CGRect)rect {
    CGFloat scale = image.scale;
    CGRect scaledRect = CGRectMake(
        rect.origin.x * scale,
        rect.origin.y * scale,
        rect.size.width * scale,
        rect.size.height * scale
    );
    
    CGImageRef croppedRef = CGImageCreateWithImageInRect(image.CGImage, scaledRect);
    UIImage *cropped = [UIImage imageWithCGImage:croppedRef 
                                           scale:scale 
                                     orientation:image.imageOrientation];
    CGImageRelease(croppedRef);
    
    return cropped;
}

// ตัดรูปเป็นวงกลม
- (UIImage *)circularCropImage:(UIImage *)image {
    CGFloat size = MIN(image.size.width, image.size.height);
    CGRect rect = CGRectMake(0, 0, size, size);
    
    UIGraphicsBeginImageContextWithOptions(rect.size, NO, image.scale);
    CGContextRef ctx = UIGraphicsGetCurrentContext();
    
    // สร้าง clipping path วงกลม
    CGContextAddEllipseInRect(ctx, rect);
    CGContextClip(ctx);
    
    // วาดรูปภาพ
    [image drawInRect:rect];
    
    UIImage *circular = UIGraphicsGetImageFromCurrentImageContext();
    UIGraphicsEndImageContext();
    
    return circular;
}
```

---

## 7. Image Caching ด้วย NSCache

### 7.1 สร้าง Image Cache

```objc
// ImageCache.h
@interface ImageCache : NSObject

@property (nonatomic, strong) NSCache *cache;

+ (instancetype)sharedCache;
- (UIImage *)imageForKey:(NSString *)key;
- (void)setImage:(UIImage *)image forKey:(NSString *)key;
- (void)removeImageForKey:(NSString *)key;
- (void)clearCache;

@end

// ImageCache.m
@implementation ImageCache

+ (instancetype)sharedCache {
    static ImageCache *instance = nil;
    static dispatch_once_t onceToken;
    dispatch_once(&onceToken, ^{
        instance = [[ImageCache alloc] init];
    });
    return instance;
}

- (instancetype)init {
    self = [super init];
    if (self) {
        _cache = [[NSCache alloc] init];
        _cache.countLimit = 100;         // สูงสุด 100 รูป
        _cache.totalCostLimit = 50 * 1024 * 1024; // 50 MB
        _cache.name = @"com.app.ImageCache";
    }
    return self;
}

- (UIImage *)imageForKey:(NSString *)key {
    return [self.cache objectForKey:key];
}

- (void)setImage:(UIImage *)image forKey:(NSString *)key {
    if (!image || !key) return;
    
    // คำนวณขนาดของรูปภาพเป็น cost
    NSUInteger cost = (NSUInteger)(image.size.width * image.size.height * image.scale * 4);
    [self.cache setObject:image forKey:key cost:cost];
}

- (void)removeImageForKey:(NSString *)key {
    [self.cache removeObjectForKey:key];
}

- (void)clearCache {
    [self.cache removeAllObjects];
}

@end
```

### 7.2 ใช้งาน Cache ร่วมกับ Download

```objc
- (void)loadImageWithURL:(NSURL *)url 
              completion:(void(^)(UIImage *image, NSError *error))completion {
    
    NSString *cacheKey = url.absoluteString;
    
    // ตรวจสอบ cache ก่อน
    UIImage *cached = [[ImageCache sharedCache] imageForKey:cacheKey];
    if (cached) {
        if (completion) {
            dispatch_async(dispatch_get_main_queue(), ^{
                completion(cached, nil);
            });
        }
        return;
    }
    
    // ดาวน์โหลดจาก network
    NSURLSessionDataTask *task = [[NSURLSession sharedSession] 
        dataTaskWithURL:url 
      completionHandler:^(NSData *data, NSURLResponse *response, NSError *error) {
        
        if (error) {
            dispatch_async(dispatch_get_main_queue(), ^{
                if (completion) completion(nil, error);
            });
            return;
        }
        
        UIImage *image = [UIImage imageWithData:data];
        if (image) {
            // บันทึกลง cache
            [[ImageCache sharedCache] setImage:image forKey:cacheKey];
        }
        
        dispatch_async(dispatch_get_main_queue(), ^{
            if (completion) completion(image, nil);
        });
    }];
    
    [task resume];
}
```

---

## 8. Async Image Loading Pattern

### 8.1 AsyncImageLoader

```objc
// AsyncImageLoader.h
@interface AsyncImageLoader : NSObject

+ (instancetype)sharedLoader;

- (NSURLSessionDataTask *)loadImageFromURL:(NSURL *)url
                                completion:(void(^)(UIImage *image))completion;

- (void)cancelLoadForURL:(NSURL *)url;
- (void)cancelAllLoads;

@end

// AsyncImageLoader.m
@interface AsyncImageLoader ()
@property (nonatomic, strong) NSCache *imageCache;
@property (nonatomic, strong) NSMutableDictionary<NSString *, NSURLSessionDataTask *> *activeTasks;
@property (nonatomic, strong) dispatch_queue_t processingQueue;
@end

@implementation AsyncImageLoader

+ (instancetype)sharedLoader {
    static AsyncImageLoader *instance = nil;
    static dispatch_once_t onceToken;
    dispatch_once(&onceToken, ^{
        instance = [[AsyncImageLoader alloc] init];
    });
    return instance;
}

- (instancetype)init {
    self = [super init];
    if (self) {
        _imageCache = [[NSCache alloc] init];
        _imageCache.countLimit = 50;
        _activeTasks = [NSMutableDictionary dictionary];
        _processingQueue = dispatch_queue_create("com.app.imageProcessing", 
                                                  DISPATCH_QUEUE_CONCURRENT);
    }
    return self;
}

- (NSURLSessionDataTask *)loadImageFromURL:(NSURL *)url
                                completion:(void(^)(UIImage *image))completion {
    if (!url) {
        if (completion) completion(nil);
        return nil;
    }
    
    NSString *key = url.absoluteString;
    
    // ตรวจสอบ cache
    UIImage *cached = [self.imageCache objectForKey:key];
    if (cached) {
        dispatch_async(dispatch_get_main_queue(), ^{
            if (completion) completion(cached);
        });
        return nil;
    }
    
    // สร้าง task ใหม่
    NSURLSessionDataTask *task = [[NSURLSession sharedSession] 
        dataTaskWithURL:url 
      completionHandler:^(NSData *data, NSURLResponse *response, NSError *error) {
        
        [self.activeTasks removeObjectForKey:key];
        
        if (!data || error) {
            dispatch_async(dispatch_get_main_queue(), ^{
                if (completion) completion(nil);
            });
            return;
        }
        
        // ประมวลผลรูปภาพบน background queue
        dispatch_async(self.processingQueue, ^{
            UIImage *image = [UIImage imageWithData:data];
            if (image) {
                [self.imageCache setObject:image forKey:key];
            }
            
            dispatch_async(dispatch_get_main_queue(), ^{
                if (completion) completion(image);
            });
        });
    }];
    
    self.activeTasks[key] = task;
    [task resume];
    
    return task;
}

- (void)cancelLoadForURL:(NSURL *)url {
    NSString *key = url.absoluteString;
    NSURLSessionDataTask *task = self.activeTasks[key];
    [task cancel];
    [self.activeTasks removeObjectForKey:key];
}

- (void)cancelAllLoads {
    for (NSURLSessionDataTask *task in self.activeTasks.allValues) {
        [task cancel];
    }
    [self.activeTasks removeAllObjects];
}

@end
```

### 8.2 UIImageView Category สำหรับ Async Loading

```objc
// UIImageView+AsyncLoad.h
@interface UIImageView (AsyncLoad)

- (void)setImageWithURL:(NSURL *)url 
            placeholder:(UIImage *)placeholder;

- (void)setImageWithURL:(NSURL *)url 
            placeholder:(UIImage *)placeholder
             completion:(void(^)(UIImage *image, NSError *error))completion;

- (void)cancelImageLoad;

@end

// UIImageView+AsyncLoad.m
#import <objc/runtime.h>

static const void *kURLKey = &kURLKey;

@implementation UIImageView (AsyncLoad)

- (void)setImageWithURL:(NSURL *)url placeholder:(UIImage *)placeholder {
    [self setImageWithURL:url placeholder:placeholder completion:nil];
}

- (void)setImageWithURL:(NSURL *)url 
            placeholder:(UIImage *)placeholder
             completion:(void(^)(UIImage *image, NSError *error))completion {
    
    // ยกเลิก request เก่า
    [self cancelImageLoad];
    
    // แสดง placeholder ก่อน
    if (placeholder) {
        self.image = placeholder;
    }
    
    // จำ URL ปัจจุบัน
    objc_setAssociatedObject(self, kURLKey, url, OBJC_ASSOCIATION_RETAIN_NONATOMIC);
    
    __weak typeof(self) weakSelf = self;
    [[AsyncImageLoader sharedLoader] loadImageFromURL:url completion:^(UIImage *image) {
        __strong typeof(weakSelf) strongSelf = weakSelf;
        
        // ตรวจสอบว่า URL ยังตรงกันอยู่
        NSURL *currentURL = objc_getAssociatedObject(strongSelf, kURLKey);
        if (![currentURL.absoluteString isEqualToString:url.absoluteString]) {
            return;
        }
        
        if (image) {
            [UIView transitionWithView:strongSelf 
                              duration:0.3 
                               options:UIViewAnimationOptionTransitionCrossDissolve 
                            animations:^{
                strongSelf.image = image;
            } completion:nil];
        }
        
        if (completion) {
            completion(image, nil);
        }
    }];
}

- (void)cancelImageLoad {
    NSURL *url = objc_getAssociatedObject(self, kURLKey);
    if (url) {
        [[AsyncImageLoader sharedLoader] cancelLoadForURL:url];
    }
}

@end
```

---

## 9. UIGraphicsImageRenderer

`UIGraphicsImageRenderer` เป็น API สมัยใหม่ที่แนะนำตั้งแต่ iOS 10 ซึ่งมีประสิทธิภาพดีกว่า `UIGraphicsBeginImageContext`

### 9.1 การใช้งานพื้นฐาน

```objc
- (UIImage *)createImageWithRenderer {
    UIGraphicsImageRenderer *renderer = [[UIGraphicsImageRenderer alloc] 
                                          initWithSize:CGSizeMake(200, 200)];
    
    UIImage *image = [renderer imageWithActions:^(UIGraphicsImageRendererContext *rendererContext) {
        CGContextRef context = rendererContext.CGContext;
        
        // วาดพื้นหลัง gradient
        CGColorSpaceRef colorSpace = CGColorSpaceCreateDeviceRGB();
        CGFloat colors[] = {
            0.2, 0.6, 1.0, 1.0, // น้ำเงิน
            0.0, 0.3, 0.8, 1.0  // น้ำเงินเข้ม
        };
        CGGradientRef gradient = CGGradientCreateWithColorComponents(
            colorSpace, colors, NULL, 2);
        
        CGContextDrawLinearGradient(context, gradient, 
                                    CGPointMake(0, 0), CGPointMake(0, 200), 0);
        
        CGGradientRelease(gradient);
        CGColorSpaceRelease(colorSpace);
        
        // วาดดาว
        [[UIColor whiteColor] setFill];
        UIBezierPath *starPath = [self createStarPathWithCenter:CGPointMake(100, 100) 
                                                  outerRadius:60 
                                                  innerRadius:25 
                                                   numPoints:5];
        [starPath fill];
    }];
    
    return image;
}

- (UIBezierPath *)createStarPathWithCenter:(CGPoint)center 
                               outerRadius:(CGFloat)outer
                               innerRadius:(CGFloat)inner
                                numPoints:(NSInteger)num {
    UIBezierPath *path = [UIBezierPath bezierPath];
    CGFloat angleOffset = -M_PI / 2.0;
    CGFloat step = M_PI * 2.0 / num;
    
    for (NSInteger i = 0; i < num; i++) {
        CGFloat outerAngle = i * step + angleOffset;
        CGFloat innerAngle = outerAngle + step / 2.0;
        
        CGPoint outerPoint = CGPointMake(
            center.x + outer * cos(outerAngle),
            center.y + outer * sin(outerAngle)
        );
        CGPoint innerPoint = CGPointMake(
            center.x + inner * cos(innerAngle),
            center.y + inner * sin(innerAngle)
        );
        
        if (i == 0) {
            [path moveToPoint:outerPoint];
        } else {
            [path addLineToPoint:outerPoint];
        }
        [path addLineToPoint:innerPoint];
    }
    [path closePath];
    
    return path;
}
```

### 9.2 สร้าง PDF ด้วย UIGraphicsImageRenderer

```objc
- (NSData *)createPDFData {
    UIGraphicsPDFRendererFormat *format = [UIGraphicsPDFRendererFormat defaultFormat];
    format.documentInfo = @{
        (NSString *)kCGPDFContextTitle: @"My Document",
        (NSString *)kCGPDFContextAuthor: @"App Name"
    };
    
    CGRect pageRect = CGRectMake(0, 0, 595, 842); // A4
    UIGraphicsPDFRenderer *renderer = [[UIGraphicsPDFRenderer alloc] 
                                        initWithBounds:pageRect 
                                               format:format];
    
    NSData *pdfData = [renderer PDFDataWithActions:^(UIGraphicsPDFRendererContext *rendererContext) {
        [rendererContext beginPage];
        
        // หัวเรื่อง
        NSDictionary *titleAttrs = @{
            NSFontAttributeName: [UIFont boldSystemFontOfSize:24],
            NSForegroundColorAttributeName: [UIColor blackColor]
        };
        [@"รายงานประจำเดือน" drawAtPoint:CGPointMake(50, 50) withAttributes:titleAttrs];
        
        // เส้น separator
        CGContextRef ctx = rendererContext.CGContext;
        CGContextSetStrokeColorWithColor(ctx, [UIColor grayColor].CGColor);
        CGContextSetLineWidth(ctx, 1.0);
        CGContextMoveToPoint(ctx, 50, 80);
        CGContextAddLineToPoint(ctx, 545, 80);
        CGContextStrokePath(ctx);
        
        // เนื้อหา
        NSDictionary *bodyAttrs = @{
            NSFontAttributeName: [UIFont systemFontOfSize:14],
            NSForegroundColorAttributeName: [UIColor darkGrayColor]
        };
        [@"เนื้อหาของรายงาน..." drawAtPoint:CGPointMake(50, 100) withAttributes:bodyAttrs];
    }];
    
    return pdfData;
}
```

---

## 10. Image from Core Graphics Context

### 10.1 สร้าง Bitmap Context

```objc
- (UIImage *)createBitmapImage {
    int width = 256;
    int height = 256;
    
    // สร้าง bitmap data
    unsigned char *rawData = (unsigned char *)calloc(height * width * 4, sizeof(unsigned char));
    
    CGColorSpaceRef colorSpace = CGColorSpaceCreateDeviceRGB();
    CGContextRef context = CGBitmapContextCreate(
        rawData, width, height, 8,
        width * 4, colorSpace,
        kCGImageAlphaPremultipliedLast | kCGBitmapByteOrder32Big
    );
    
    // วาดกรอบ
    CGContextSetStrokeColorWithColor(context, [UIColor systemRedColor].CGColor);
    CGContextSetLineWidth(context, 3.0);
    CGContextStrokeRect(context, CGRectMake(10, 10, width - 20, height - 20));
    
    // วาดเส้นทแยงมุม
    CGContextSetStrokeColorWithColor(context, [UIColor systemBlueColor].CGColor);
    CGContextMoveToPoint(context, 10, 10);
    CGContextAddLineToPoint(context, width - 10, height - 10);
    CGContextStrokePath(context);
    
    // สร้าง CGImage แล้วแปลงเป็น UIImage
    CGImageRef cgImage = CGBitmapContextCreateImage(context);
    UIImage *image = [UIImage imageWithCGImage:cgImage];
    
    // Cleanup
    CGImageRelease(cgImage);
    CGContextRelease(context);
    CGColorSpaceRelease(colorSpace);
    free(rawData);
    
    return image;
}
```

### 10.2 Pixel Manipulation

```objc
- (UIImage *)applyGrayscaleFilter:(UIImage *)image {
    CGImageRef imageRef = image.CGImage;
    NSUInteger width = CGImageGetWidth(imageRef);
    NSUInteger height = CGImageGetHeight(imageRef);
    
    unsigned char *rawData = (unsigned char *)calloc(height * width * 4, sizeof(unsigned char));
    
    CGColorSpaceRef colorSpace = CGColorSpaceCreateDeviceRGB();
    CGContextRef context = CGBitmapContextCreate(
        rawData, width, height, 8, width * 4,
        colorSpace,
        kCGImageAlphaPremultipliedLast | kCGBitmapByteOrder32Big
    );
    
    CGContextDrawImage(context, CGRectMake(0, 0, width, height), imageRef);
    
    // แปลงทุก pixel เป็น grayscale
    for (NSUInteger y = 0; y < height; y++) {
        for (NSUInteger x = 0; x < width; x++) {
            NSUInteger offset = (y * width + x) * 4;
            
            unsigned char r = rawData[offset];
            unsigned char g = rawData[offset + 1];
            unsigned char b = rawData[offset + 2];
            
            // สูตร grayscale
            unsigned char gray = (unsigned char)(0.299 * r + 0.587 * g + 0.114 * b);
            
            rawData[offset] = gray;
            rawData[offset + 1] = gray;
            rawData[offset + 2] = gray;
            // alpha ไม่เปลี่ยน
        }
    }
    
    CGImageRef newImageRef = CGBitmapContextCreateImage(context);
    UIImage *result = [UIImage imageWithCGImage:newImageRef];
    
    CGImageRelease(newImageRef);
    CGContextRelease(context);
    CGColorSpaceRelease(colorSpace);
    free(rawData);
    
    return result;
}
```

---

## 11. JPEG/PNG Data Conversion

### 11.1 UIImage เป็น NSData

```objc
UIImage *image = [UIImage imageNamed:@"photo"];

// แปลงเป็น JPEG (quality: 0.0 - 1.0)
NSData *jpegData = UIImageJPEGRepresentation(image, 0.8);
NSLog(@"JPEG ขนาด: %lu bytes", (unsigned long)jpegData.length);

// แปลงเป็น PNG (lossless)
NSData *pngData = UIImagePNGRepresentation(image);
NSLog(@"PNG ขนาด: %lu bytes", (unsigned long)pngData.length);

// บันทึกลง Documents
NSString *docPath = [NSSearchPathForDirectoriesInDomains(
    NSDocumentDirectory, NSUserDomainMask, YES) firstObject];

NSString *jpegPath = [docPath stringByAppendingPathComponent:@"photo.jpg"];
[jpegData writeToFile:jpegPath atomically:YES];

NSString *pngPath = [docPath stringByAppendingPathComponent:@"photo.png"];
[pngData writeToFile:pngPath atomically:YES];
```

### 11.2 เปรียบเทียบขนาดไฟล์

```objc
- (void)compareImageFormats:(UIImage *)image {
    NSArray *qualities = @[@(0.1), @(0.5), @(0.8), @(1.0)];
    
    for (NSNumber *quality in qualities) {
        NSData *jpegData = UIImageJPEGRepresentation(image, [quality floatValue]);
        NSLog(@"JPEG quality %.1f: %lu KB", 
              [quality floatValue], 
              (unsigned long)jpegData.length / 1024);
    }
    
    NSData *pngData = UIImagePNGRepresentation(image);
    NSLog(@"PNG: %lu KB", (unsigned long)pngData.length / 1024);
}
```

---

## 12. Photo Library Access (PHPhotoLibrary)

### 12.1 ขอสิทธิ์การเข้าถึง

ก่อนใช้งานต้องเพิ่ม Privacy keys ใน Info.plist:
- `NSPhotoLibraryUsageDescription`
- `NSPhotoLibraryAddUsageDescription`

```objc
#import <Photos/Photos.h>

- (void)requestPhotoLibraryPermission {
    PHAuthorizationStatus status = [PHPhotoLibrary authorizationStatus];
    
    switch (status) {
        case PHAuthorizationStatusAuthorized:
            NSLog(@"มีสิทธิ์แล้ว");
            [self accessPhotoLibrary];
            break;
            
        case PHAuthorizationStatusNotDetermined:
            [PHPhotoLibrary requestAuthorization:^(PHAuthorizationStatus newStatus) {
                dispatch_async(dispatch_get_main_queue(), ^{
                    if (newStatus == PHAuthorizationStatusAuthorized) {
                        [self accessPhotoLibrary];
                    } else {
                        [self showPermissionDeniedAlert];
                    }
                });
            }];
            break;
            
        case PHAuthorizationStatusDenied:
        case PHAuthorizationStatusRestricted:
            [self showPermissionDeniedAlert];
            break;
            
        default:
            break;
    }
}
```

### 12.2 อ่านรูปภาพจาก Photo Library

```objc
- (void)accessPhotoLibrary {
    PHFetchOptions *options = [[PHFetchOptions alloc] init];
    options.sortDescriptors = @[[NSSortDescriptor sortDescriptorWithKey:@"creationDate" 
                                                              ascending:NO]];
    options.predicate = [NSPredicate predicateWithFormat:@"mediaType == %d", 
                         PHAssetMediaTypeImage];
    
    PHFetchResult *result = [PHAsset fetchAssetsWithOptions:options];
    NSLog(@"พบรูปภาพทั้งหมด: %lu รูป", (unsigned long)result.count);
    
    // โหลดรูปแรก
    if (result.count > 0) {
        PHAsset *asset = result.firstObject;
        
        PHImageRequestOptions *requestOptions = [[PHImageRequestOptions alloc] init];
        requestOptions.deliveryMode = PHImageRequestOptionsDeliveryModeHighQualityFormat;
        requestOptions.synchronous = NO;
        
        [[PHImageManager defaultManager] requestImageForAsset:asset
                                                   targetSize:CGSizeMake(300, 300)
                                                  contentMode:PHImageContentModeAspectFit
                                                      options:requestOptions
                                                resultHandler:^(UIImage *result, NSDictionary *info) {
            dispatch_async(dispatch_get_main_queue(), ^{
                self.imageView.image = result;
            });
        }];
    }
}
```

### 12.3 บันทึกรูปลง Photo Library

```objc
- (void)saveImageToLibrary:(UIImage *)image {
    [[PHPhotoLibrary sharedPhotoLibrary] performChanges:^{
        PHAssetChangeRequest *request = [PHAssetChangeRequest creationRequestForAssetFromImage:image];
        request.creationDate = [NSDate date];
        
    } completionHandler:^(BOOL success, NSError *error) {
        dispatch_async(dispatch_get_main_queue(), ^{
            if (success) {
                UIAlertController *alert = [UIAlertController 
                    alertControllerWithTitle:@"สำเร็จ"
                                     message:@"บันทึกรูปภาพแล้ว"
                              preferredStyle:UIAlertControllerStyleAlert];
                [alert addAction:[UIAlertAction actionWithTitle:@"ตกลง" 
                                                          style:UIAlertActionStyleDefault 
                                                        handler:nil]];
                [self presentViewController:alert animated:YES completion:nil];
            } else {
                NSLog(@"เกิดข้อผิดพลาด: %@", error.localizedDescription);
            }
        });
    }];
}
```

---

## 13. Camera Access (AVFoundation)

### 13.1 ตรวจสอบกล้อง

ต้องเพิ่ม Privacy keys ใน Info.plist:
- `NSCameraUsageDescription`

```objc
#import <AVFoundation/AVFoundation.h>

- (void)checkCameraAuthorization {
    AVAuthorizationStatus status = [AVCaptureDevice authorizationStatusForMediaType:AVMediaTypeVideo];
    
    switch (status) {
        case AVAuthorizationStatusAuthorized:
            NSLog(@"มีสิทธิ์ใช้กล้องแล้ว");
            break;
            
        case AVAuthorizationStatusNotDetermined:
            [AVCaptureDevice requestAccessForMediaType:AVMediaTypeVideo 
                                     completionHandler:^(BOOL granted) {
                NSLog(@"สิทธิ์กล้อง: %@", granted ? @"ได้รับ" : @"ปฏิเสธ");
            }];
            break;
            
        case AVAuthorizationStatusDenied:
            NSLog(@"ผู้ใช้ปฏิเสธสิทธิ์กล้อง");
            break;
            
        case AVAuthorizationStatusRestricted:
            NSLog(@"กล้องถูกจำกัดการใช้งาน");
            break;
    }
}
```

### 13.2 AVCaptureSession พื้นฐาน

```objc
@interface CameraViewController ()
@property (nonatomic, strong) AVCaptureSession *session;
@property (nonatomic, strong) AVCapturePhotoOutput *photoOutput;
@property (nonatomic, strong) AVCaptureVideoPreviewLayer *previewLayer;
@end

@implementation CameraViewController

- (void)setupCamera {
    self.session = [[AVCaptureSession alloc] init];
    self.session.sessionPreset = AVCaptureSessionPresetPhoto;
    
    // เพิ่ม input (กล้องหลัง)
    AVCaptureDevice *device = [AVCaptureDevice defaultDeviceWithMediaType:AVMediaTypeVideo];
    NSError *error = nil;
    AVCaptureDeviceInput *input = [AVCaptureDeviceInput deviceInputWithDevice:device 
                                                                         error:&error];
    if (error) {
        NSLog(@"เกิดข้อผิดพลาด: %@", error.localizedDescription);
        return;
    }
    
    if ([self.session canAddInput:input]) {
        [self.session addInput:input];
    }
    
    // เพิ่ม output
    self.photoOutput = [[AVCapturePhotoOutput alloc] init];
    if ([self.session canAddOutput:self.photoOutput]) {
        [self.session addOutput:self.photoOutput];
    }
    
    // เพิ่ม preview layer
    self.previewLayer = [AVCaptureVideoPreviewLayer layerWithSession:self.session];
    self.previewLayer.videoGravity = AVLayerVideoGravityResizeAspectFill;
    self.previewLayer.frame = self.view.bounds;
    [self.view.layer insertSublayer:self.previewLayer atIndex:0];
    
    // เริ่ม session บน background thread
    dispatch_async(dispatch_get_global_queue(DISPATCH_QUEUE_PRIORITY_HIGH, 0), ^{
        [self.session startRunning];
    });
}

- (void)capturePhoto {
    AVCapturePhotoSettings *settings = [AVCapturePhotoSettings photoSettings];
    settings.flashMode = AVCaptureFlashModeAuto;
    [self.photoOutput capturePhotoWithSettings:settings delegate:self];
}

#pragma mark - AVCapturePhotoCaptureDelegate

- (void)captureOutput:(AVCapturePhotoOutput *)output 
didFinishProcessingPhoto:(AVCapturePhoto *)photo 
                error:(NSError *)error {
    
    if (error) {
        NSLog(@"เกิดข้อผิดพลาดในการถ่ายภาพ: %@", error.localizedDescription);
        return;
    }
    
    NSData *imageData = [photo fileDataRepresentation];
    UIImage *capturedImage = [UIImage imageWithData:imageData];
    
    dispatch_async(dispatch_get_main_queue(), ^{
        // ใช้งานรูปภาพ
        self.imageView.image = capturedImage;
    });
}

@end
```

---

## 14. UIImagePickerController

### 14.1 เปิด Photo Library ด้วย UIImagePickerController

```objc
#import <MobileCoreServices/MobileCoreServices.h>

- (void)openImagePicker {
    if (![UIImagePickerController isSourceTypeAvailable:UIImagePickerControllerSourceTypePhotoLibrary]) {
        NSLog(@"Photo Library ไม่พร้อมใช้งาน");
        return;
    }
    
    UIImagePickerController *picker = [[UIImagePickerController alloc] init];
    picker.sourceType = UIImagePickerControllerSourceTypePhotoLibrary;
    picker.mediaTypes = @[(NSString *)kUTTypeImage];
    picker.allowsEditing = YES;  // อนุญาตให้ crop
    picker.delegate = self;
    
    [self presentViewController:picker animated:YES completion:nil];
}

- (void)openCamera {
    if (![UIImagePickerController isSourceTypeAvailable:UIImagePickerControllerSourceTypeCamera]) {
        NSLog(@"กล้องไม่พร้อมใช้งาน");
        return;
    }
    
    UIImagePickerController *picker = [[UIImagePickerController alloc] init];
    picker.sourceType = UIImagePickerControllerSourceTypeCamera;
    picker.cameraDevice = UIImagePickerControllerCameraDeviceRear;
    picker.cameraCaptureMode = UIImagePickerControllerCameraCaptureModePhoto;
    picker.allowsEditing = NO;
    picker.delegate = self;
    
    [self presentViewController:picker animated:YES completion:nil];
}
```

### 14.2 UIImagePickerControllerDelegate

```objc
#pragma mark - UIImagePickerControllerDelegate

- (void)imagePickerController:(UIImagePickerController *)picker 
didFinishPickingMediaWithInfo:(NSDictionary<UIImagePickerControllerInfoKey, id> *)info {
    
    UIImage *selectedImage = nil;
    
    // ถ้าอนุญาตให้ edit จะได้ edited image
    if (picker.allowsEditing) {
        selectedImage = info[UIImagePickerControllerEditedImage];
    } else {
        selectedImage = info[UIImagePickerControllerOriginalImage];
    }
    
    // ดู URL ของรูปต้นฉบับ
    NSURL *imageURL = info[UIImagePickerControllerImageURL];
    
    // ดู PHAsset (iOS 11+)
    PHAsset *asset = info[UIImagePickerControllerPHAsset];
    
    if (selectedImage) {
        self.imageView.image = selectedImage;
        NSLog(@"รูปขนาด: %.0f x %.0f", 
              selectedImage.size.width, 
              selectedImage.size.height);
    }
    
    [picker dismissViewControllerAnimated:YES completion:nil];
}

- (void)imagePickerControllerDidCancel:(UIImagePickerController *)picker {
    [picker dismissViewControllerAnimated:YES completion:nil];
}
```

### 14.3 Action Sheet สำหรับเลือกแหล่งที่มา

```objc
- (void)showImageSourceSelector {
    UIAlertController *alert = [UIAlertController 
        alertControllerWithTitle:@"เลือกรูปภาพ"
                         message:nil
                  preferredStyle:UIAlertControllerStyleActionSheet];
    
    // ถ่ายภาพจากกล้อง
    if ([UIImagePickerController isSourceTypeAvailable:UIImagePickerControllerSourceTypeCamera]) {
        UIAlertAction *cameraAction = [UIAlertAction 
            actionWithTitle:@"ถ่ายภาพ" 
                      style:UIAlertActionStyleDefault 
                    handler:^(UIAlertAction *action) {
            [self openCamera];
        }];
        [alert addAction:cameraAction];
    }
    
    // เลือกจาก Photo Library
    UIAlertAction *libraryAction = [UIAlertAction 
        actionWithTitle:@"เลือกจากคลังภาพ" 
                  style:UIAlertActionStyleDefault 
                handler:^(UIAlertAction *action) {
        [self openImagePicker];
    }];
    [alert addAction:libraryAction];
    
    // ยกเลิก
    UIAlertAction *cancel = [UIAlertAction 
        actionWithTitle:@"ยกเลิก" 
                  style:UIAlertActionStyleCancel 
                handler:nil];
    [alert addAction:cancel];
    
    [self presentViewController:alert animated:YES completion:nil];
}
```

---

## 15. การจัดการ Image Orientation

```objc
// แก้ไข orientation ของรูปภาพ
- (UIImage *)fixImageOrientation:(UIImage *)image {
    if (image.imageOrientation == UIImageOrientationUp) {
        return image;
    }
    
    UIGraphicsBeginImageContextWithOptions(image.size, NO, image.scale);
    [image drawInRect:CGRectMake(0, 0, image.size.width, image.size.height)];
    UIImage *normalizedImage = UIGraphicsGetImageFromCurrentImageContext();
    UIGraphicsEndImageContext();
    
    return normalizedImage;
}
```

---

## 16. Memory Management สำหรับ Images

```objc
// จัดการหน่วยความจำอย่างถูกต้อง
- (void)loadLargeImage {
    // ใช้ autoreleasepool เพื่อปลดปล่อยหน่วยความจำเร็วขึ้น
    @autoreleasepool {
        UIImage *largeImage = [UIImage imageWithContentsOfFile:self.imagePath];
        UIImage *resized = [self resizeImage:largeImage toSize:CGSizeMake(200, 200)];
        
        dispatch_async(dispatch_get_main_queue(), ^{
            self.imageView.image = resized;
        });
    }
    // largeImage ถูกปลดปล่อยหลังจาก autoreleasepool
}

// ฟัง memory warning
- (void)didReceiveMemoryWarning {
    [super didReceiveMemoryWarning];
    
    // ล้าง cache
    [[ImageCache sharedCache] clearCache];
    
    // ปลดรูปภาพที่ไม่ได้แสดงอยู่
    if (!self.view.window) {
        self.imageView.image = nil;
    }
}
```

---

## แบบฝึกหัด (Practice Exercises)

### แบบฝึกหัดที่ 1: Image Gallery
สร้าง UICollectionView ที่แสดงรูปภาพจาก Photo Library โดย:
- ขอสิทธิ์เข้าถึง Photo Library
- โหลดรูปแบบ async
- แสดง thumbnail ขนาด 100x100
- Tap รูปเพื่อดูขนาดเต็ม

```objc
// โครงสร้างพื้นฐาน
@interface GalleryViewController : UIViewController <UICollectionViewDataSource, 
                                                       UICollectionViewDelegate>
@property (nonatomic, strong) UICollectionView *collectionView;
@property (nonatomic, strong) PHFetchResult *assets;
@end

@implementation GalleryViewController

- (void)viewDidLoad {
    [super viewDidLoad];
    [self setupCollectionView];
    [self requestPhotoAccess];
}

- (void)setupCollectionView {
    UICollectionViewFlowLayout *layout = [[UICollectionViewFlowLayout alloc] init];
    layout.itemSize = CGSizeMake(100, 100);
    layout.minimumInteritemSpacing = 2;
    layout.minimumLineSpacing = 2;
    
    self.collectionView = [[UICollectionView alloc] initWithFrame:self.view.bounds 
                                             collectionViewLayout:layout];
    self.collectionView.dataSource = self;
    self.collectionView.delegate = self;
    [self.collectionView registerClass:[UICollectionViewCell class] 
            forCellWithReuseIdentifier:@"cell"];
    [self.view addSubview:self.collectionView];
}

- (void)requestPhotoAccess {
    [PHPhotoLibrary requestAuthorization:^(PHAuthorizationStatus status) {
        if (status == PHAuthorizationStatusAuthorized) {
            PHFetchOptions *options = [[PHFetchOptions alloc] init];
            options.sortDescriptors = @[[NSSortDescriptor 
                sortDescriptorWithKey:@"creationDate" ascending:NO]];
            
            self.assets = [PHAsset fetchAssetsWithMediaType:PHAssetMediaTypeImage 
                                                    options:options];
            dispatch_async(dispatch_get_main_queue(), ^{
                [self.collectionView reloadData];
            });
        }
    }];
}

- (NSInteger)collectionView:(UICollectionView *)collectionView 
     numberOfItemsInSection:(NSInteger)section {
    return self.assets.count;
}

- (UICollectionViewCell *)collectionView:(UICollectionView *)collectionView 
                  cellForItemAtIndexPath:(NSIndexPath *)indexPath {
    UICollectionViewCell *cell = [collectionView 
        dequeueReusableCellWithReuseIdentifier:@"cell" forIndexPath:indexPath];
    
    // สร้าง imageView ถ้ายังไม่มี
    UIImageView *imageView = [cell.contentView viewWithTag:100];
    if (!imageView) {
        imageView = [[UIImageView alloc] initWithFrame:cell.contentView.bounds];
        imageView.contentMode = UIViewContentModeScaleAspectFill;
        imageView.clipsToBounds = YES;
        imageView.tag = 100;
        [cell.contentView addSubview:imageView];
    }
    
    PHAsset *asset = self.assets[indexPath.item];
    PHImageRequestOptions *options = [[PHImageRequestOptions alloc] init];
    options.deliveryMode = PHImageRequestOptionsDeliveryModeOpportunistic;
    
    [[PHImageManager defaultManager] requestImageForAsset:asset
                                               targetSize:CGSizeMake(100, 100)
                                              contentMode:PHImageContentModeAspectFill
                                                  options:options
                                            resultHandler:^(UIImage *result, NSDictionary *info) {
        dispatch_async(dispatch_get_main_queue(), ^{
            if ([collectionView indexPathForCell:cell] == indexPath) {
                imageView.image = result;
            }
        });
    }];
    
    return cell;
}

@end
```

### แบบฝึกหัดที่ 2: Image Filter App
```objc
// สร้าง app ที่สามารถ apply filter ต่างๆ บนรูปภาพ
// ให้ implement ฟิลเตอร์ต่อไปนี้:
// 1. Grayscale
// 2. Sepia tone
// 3. Blur (ใช้ Core Image)
// 4. Brightness/Contrast adjustment

#import <CoreImage/CoreImage.h>

- (UIImage *)applyFilterType:(NSString *)filterType toImage:(UIImage *)image {
    CIImage *ciImage = [CIImage imageWithCGImage:image.CGImage];
    CIFilter *filter = nil;
    
    if ([filterType isEqualToString:@"grayscale"]) {
        filter = [CIFilter filterWithName:@"CIColorMonochrome"];
        [filter setValue:ciImage forKey:kCIInputImageKey];
        [filter setValue:[CIColor colorWithRed:0.7 green:0.7 blue:0.7] 
                  forKey:kCIInputColorKey];
        [filter setValue:@1.0 forKey:kCIInputIntensityKey];
        
    } else if ([filterType isEqualToString:@"sepia"]) {
        filter = [CIFilter filterWithName:@"CISepiaTone"];
        [filter setValue:ciImage forKey:kCIInputImageKey];
        [filter setValue:@0.8 forKey:kCIInputIntensityKey];
        
    } else if ([filterType isEqualToString:@"blur"]) {
        filter = [CIFilter filterWithName:@"CIGaussianBlur"];
        [filter setValue:ciImage forKey:kCIInputImageKey];
        [filter setValue:@10.0 forKey:kCIInputRadiusKey];
    }
    
    if (!filter) return image;
    
    CIImage *outputImage = [filter outputImage];
    CIContext *context = [CIContext context];
    CGImageRef cgImage = [context createCGImage:outputImage fromRect:ciImage.extent];
    UIImage *result = [UIImage imageWithCGImage:cgImage];
    CGImageRelease(cgImage);
    
    return result;
}
```

### แบบฝึกหัดที่ 3: Custom UIImageView ที่รองรับ Zoom
```objc
@interface ZoomableImageView : UIScrollView <UIScrollViewDelegate>
@property (nonatomic, strong) UIImageView *imageView;
- (instancetype)initWithImage:(UIImage *)image;
@end

@implementation ZoomableImageView

- (instancetype)initWithImage:(UIImage *)image {
    self = [super initWithFrame:CGRectZero];
    if (self) {
        self.minimumZoomScale = 1.0;
        self.maximumZoomScale = 4.0;
        self.delegate = self;
        self.showsHorizontalScrollIndicator = NO;
        self.showsVerticalScrollIndicator = NO;
        
        self.imageView = [[UIImageView alloc] initWithImage:image];
        self.imageView.contentMode = UIViewContentModeScaleAspectFit;
        [self addSubview:self.imageView];
        
        // Double tap to zoom
        UITapGestureRecognizer *doubleTap = [[UITapGestureRecognizer alloc] 
            initWithTarget:self action:@selector(handleDoubleTap:)];
        doubleTap.numberOfTapsRequired = 2;
        [self addGestureRecognizer:doubleTap];
    }
    return self;
}

- (UIView *)viewForZoomingInScrollView:(UIScrollView *)scrollView {
    return self.imageView;
}

- (void)handleDoubleTap:(UITapGestureRecognizer *)gesture {
    if (self.zoomScale > 1.0) {
        [self setZoomScale:1.0 animated:YES];
    } else {
        CGPoint tapPoint = [gesture locationInView:self.imageView];
        CGRect zoomRect = CGRectMake(tapPoint.x - 50, tapPoint.y - 50, 100, 100);
        [self zoomToRect:zoomRect animated:YES];
    }
}

@end
```

---

## สรุป

ในบทนี้เราได้เรียนรู้การจัดการรูปภาพใน iOS อย่างครอบคลุม:

1. **UIImage loading** - การโหลดรูปภาพจากแหล่งต่างๆ
2. **UIImageView** - การแสดงผลและตั้งค่ารูปภาพ  
3. **Content Modes** - การควบคุมการ scale รูปภาพ
4. **Rendering Modes** - Original vs Template
5. **Drawing** - การวาดรูปภาพด้วย code
6. **Resizing/Cropping** - การปรับขนาดและตัดรูป
7. **NSCache** - การ cache รูปภาพอย่างมีประสิทธิภาพ
8. **Async Loading** - การโหลดรูปแบบ asynchronous
9. **UIGraphicsImageRenderer** - API สมัยใหม่สำหรับวาดรูป
10. **Bitmap Context** - การจัดการ pixel โดยตรง
11. **Data Conversion** - JPEG/PNG
12. **PHPhotoLibrary** - การเข้าถึง Photo Library
13. **Camera** - การใช้กล้องถ่ายภาพ
14. **UIImagePickerController** - UI สำหรับเลือก/ถ่ายรูป

ในบทต่อไปจะเรียนรู้เรื่อง Core Graphics ซึ่งให้ความสามารถในการวาดกราฟิกขั้นสูง
