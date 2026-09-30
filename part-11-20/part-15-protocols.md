# ส่วนที่ 15: Protocols ใน Objective-C

## บทนำ

Protocol ใน Objective-C คือ contract หรือข้อตกลงที่กำหนด interface ที่ class ต้องหรืออาจจะ implement โดยที่ protocol ไม่ได้ provide implementation เอง Protocol คล้ายกับ interface ใน Java หรือ C# และเป็นกลไกสำคัญในการทำ delegate pattern, data source pattern, และการออกแบบ loosely-coupled systems

---

## 15.1 Protocol คืออะไร?

### ความหมายและจุดประสงค์

Protocol กำหนดชุดของ method และ property ที่ class สามารถ adopt (นำไปใช้) โดย:
- Class ที่ adopt protocol ต้อง implement method ที่เป็น `@required`
- Class อาจ implement method ที่เป็น `@optional` หรือไม่ก็ได้
- Protocol ไม่ใช่ class และไม่มี implementation ของตัวเอง

```objc
// syntax พื้นฐานของ protocol
@protocol MyProtocol <NSObject>

@required
- (void)requiredMethod;
- (NSString *)anotherRequiredMethod:(NSInteger)value;

@optional
- (void)optionalMethod;
- (BOOL)optionalCheck;

@end
```

### Protocol vs Inheritance

```
Inheritance:
├── ได้รับ implementation จาก parent class
├── สืบทอด state (instance variables)
├── เป็นความสัมพันธ์ "is-a"
└── Objective-C ไม่รองรับ multiple inheritance

Protocol:
├── เป็นแค่ interface (ไม่มี implementation)
├── ไม่สืบทอด state
├── เป็นความสัมพันธ์ "can-do"
└── Class สามารถ conform กับหลาย protocol ได้
```

---

## 15.2 การประกาศ Protocol

### @protocol Syntax

```objc
// ไฟล์: Printable.h
@protocol Printable <NSObject>

@required
- (NSString *)toString;

@optional
- (void)printToConsole;
- (NSData *)printToData;

@end

// ไฟล์: Serializable.h  
@protocol Serializable <NSObject>

@required
- (NSDictionary *)toDictionary;
+ (instancetype)fromDictionary:(NSDictionary *)dict;

@optional
- (NSData *)toJSON;
+ (instancetype)fromJSON:(NSData *)data;

@end
```

### Protocol พร้อม Properties

```objc
@protocol Identifiable <NSObject>

@required
@property (nonatomic, readonly) NSString *uniqueID;
@property (nonatomic, copy) NSString *name;

@optional
@property (nonatomic, copy) NSString *description;

@end

// Class ที่ adopt protocol จะต้อง @synthesize หรือ implement property เหล่านี้
@interface User : NSObject <Identifiable>

@property (nonatomic, readonly) NSString *uniqueID;
@property (nonatomic, copy) NSString *name;

@end

@implementation User

@synthesize uniqueID = _uniqueID;
@synthesize name = _name;

- (instancetype)init {
    self = [super init];
    if (self) {
        _uniqueID = [[NSUUID UUID] UUIDString];
    }
    return self;
}

@end
```

---

## 15.3 @required vs @optional Methods

### @required Methods

Method ที่ต้อง implement เมื่อ class adopt protocol (ค่า default คือ @required)

```objc
@protocol DataProcessor <NSObject>

// ทั้ง 2 method นี้เป็น @required (default)
- (NSData *)processData:(NSData *)input;
- (BOOL)canProcess:(NSData *)data;

@required // ระบุชัดเจน
- (void)reset;

@end
```

### @optional Methods

Method ที่อาจจะ implement หรือไม่ก็ได้

```objc
@protocol AnimationDelegate <NSObject>

@required
- (void)animationDidStart:(id)animation;
- (void)animationDidFinish:(id)animation;

@optional
- (void)animationDidCancel:(id)animation;
- (void)animation:(id)animation didProgress:(double)progress;
- (NSTimeInterval)animationDuration;

@end
```

### การตรวจสอบ Optional Methods ก่อนเรียก

```objc
// เสมอต้องตรวจสอบ optional method ก่อนเรียก!
@interface Animator : NSObject

@property (nonatomic, weak) id<AnimationDelegate> delegate;

- (void)startAnimation;

@end

@implementation Animator

- (void)startAnimation {
    // บอก delegate ว่าเริ่ม animation
    if ([_delegate respondsToSelector:@selector(animationDidStart:)]) {
        [_delegate animationDidStart:self];
    }
    
    // ทำ animation...
    NSLog(@"Animating...");
    
    // ตรวจสอบ optional progress callback
    if ([_delegate respondsToSelector:@selector(animation:didProgress:)]) {
        [_delegate animation:self didProgress:0.5];
    }
    
    // เสร็จสิ้น
    if ([_delegate respondsToSelector:@selector(animationDidFinish:)]) {
        [_delegate animationDidFinish:self];
    }
}

@end
```

---

## 15.4 การ Adopt Protocol

### Syntax การ Adopt

```objc
// การ adopt หนึ่ง protocol
@interface MyClass : NSObject <Protocol1>
// ...
@end

// การ adopt หลาย protocol
@interface MyClass : NSObject <Protocol1, Protocol2, Protocol3>
// ...
@end

// Protocol สามารถ inherit จาก protocol อื่นได้
@protocol DetailedPrintable <Printable>
- (NSString *)toDetailedString;
@end
```

### ตัวอย่างสมบูรณ์

```objc
// ไฟล์: Shape.h
@protocol Shape <NSObject>

@required
@property (nonatomic, readonly) NSString *name;
- (double)area;
- (double)perimeter;

@optional
- (void)draw;
- (NSString *)svgPath;

@end

// ไฟล์: ColoredShape.h
@protocol ColoredShape <Shape>

@required
@property (nonatomic, copy) NSString *fillColor;
@property (nonatomic, copy) NSString *strokeColor;

@end

// ไฟล์: Circle.h
@interface Circle : NSObject <ColoredShape>

@property (nonatomic, assign) double radius;
@property (nonatomic, readonly) NSString *name;     // from Shape
@property (nonatomic, copy) NSString *fillColor;    // from ColoredShape
@property (nonatomic, copy) NSString *strokeColor;  // from ColoredShape

- (instancetype)initWithRadius:(double)radius;

@end

// ไฟล์: Circle.m
@implementation Circle

- (instancetype)initWithRadius:(double)radius {
    self = [super init];
    if (self) {
        _radius = radius;
        _fillColor = @"white";
        _strokeColor = @"black";
    }
    return self;
}

- (NSString *)name {
    return @"Circle";
}

- (double)area {
    return M_PI * _radius * _radius;
}

- (double)perimeter {
    return 2 * M_PI * _radius;
}

// Optional method - เลือก implement
- (void)draw {
    NSLog(@"Drawing %@ circle (r=%.2f) fill=%@ stroke=%@",
          self.name, _radius, _fillColor, _strokeColor);
}

- (NSString *)svgPath {
    return [NSString stringWithFormat:@"<circle r=\"%.2f\" fill=\"%@\" stroke=\"%@\"/>",
            _radius, _fillColor, _strokeColor];
}

@end
```

---

## 15.5 การตรวจสอบ Protocol Conformance

### conformsToProtocol:

```objc
// ตรวจสอบว่า class conform to protocol
id obj = [[Circle alloc] init];

if ([obj conformsToProtocol:@protocol(Shape)]) {
    id<Shape> shape = obj;
    NSLog(@"Area: %.2f", [shape area]);
    NSLog(@"Perimeter: %.2f", [shape perimeter]);
}

// ใช้กับ class
BOOL circleConforms = [Circle conformsToProtocol:@protocol(Shape)];
NSLog(@"Circle conforms to Shape: %@", circleConforms ? @"YES" : @"NO");

// ตรวจสอบ optional methods
if ([obj conformsToProtocol:@protocol(Shape)]) {
    id<Shape> shape = obj;
    
    // ตรวจสอบ optional method ก่อนเรียก
    if ([shape respondsToSelector:@selector(draw)]) {
        [shape draw];
    }
    
    if ([shape respondsToSelector:@selector(svgPath)]) {
        NSLog(@"SVG: %@", [shape svgPath]);
    }
}
```

### ใช้ Protocol เป็น Type

```objc
// ใช้ protocol เป็น variable type
id<Shape> myShape = [[Circle alloc] initWithRadius:5.0];
[myShape area]; // Compiler รู้ว่า myShape มี method area

// ใช้ใน array
NSArray<id<Shape>> *shapes = @[
    [[Circle alloc] initWithRadius:5.0],
    [[Rectangle alloc] initWithWidth:4.0 height:6.0],
    [[Triangle alloc] initWithBase:3.0 height:4.0]
];

for (id<Shape> shape in shapes) {
    NSLog(@"%@: area=%.2f", shape.name, [shape area]);
}

// ใช้ใน method parameter
- (double)totalArea:(NSArray<id<Shape>> *)shapes {
    double total = 0;
    for (id<Shape> shape in shapes) {
        total += [shape area];
    }
    return total;
}
```

---

## 15.6 Delegate Pattern

### Delegate Pattern คืออะไร?

Delegate pattern คือ design pattern ที่ object A มอบหมายงานบางส่วนให้ object B ทำ โดย B ต้อง conform to protocol ที่ A กำหนด

```
[Object A] ---(delegate)--> [Object B (ทำหน้าที่แทน A)]
```

### ตัวอย่าง: Download Manager

```objc
// ไฟล์: DownloadManagerDelegate.h

@class DownloadManager;

@protocol DownloadManagerDelegate <NSObject>

@required
// เรียกเมื่อ download เสร็จสิ้น
- (void)downloadManager:(DownloadManager *)manager 
    didFinishDownloadingURL:(NSURL *)url 
                   toPath:(NSString *)localPath;

// เรียกเมื่อ download เกิด error
- (void)downloadManager:(DownloadManager *)manager 
       didFailWithError:(NSError *)error 
                   URL:(NSURL *)url;

@optional
// เรียกเมื่อ download progress เปลี่ยน
- (void)downloadManager:(DownloadManager *)manager 
     didUpdateProgress:(double)progress 
                   URL:(NSURL *)url;

// เรียกก่อนเริ่ม download
- (BOOL)downloadManager:(DownloadManager *)manager 
  shouldDownloadFromURL:(NSURL *)url;

@end
```

```objc
// ไฟล์: DownloadManager.h
@interface DownloadManager : NSObject

@property (nonatomic, weak) id<DownloadManagerDelegate> delegate;
@property (nonatomic, readonly) BOOL isDownloading;

- (void)downloadURL:(NSURL *)url toDirectory:(NSString *)directory;
- (void)cancelCurrentDownload;

@end

// ไฟล์: DownloadManager.m
@implementation DownloadManager {
    BOOL _isDownloading;
    NSURL *_currentURL;
}

@synthesize isDownloading = _isDownloading;

- (void)downloadURL:(NSURL *)url toDirectory:(NSString *)directory {
    
    // ถาม delegate ว่าควร download ไหม (optional)
    if ([_delegate respondsToSelector:@selector(downloadManager:shouldDownloadFromURL:)]) {
        BOOL should = [_delegate downloadManager:self shouldDownloadFromURL:url];
        if (!should) {
            NSLog(@"Delegate declined download of %@", url);
            return;
        }
    }
    
    _isDownloading = YES;
    _currentURL = url;
    
    NSLog(@"Starting download: %@", url);
    
    // Simulate download process
    // ในงานจริงจะใช้ NSURLSession
    dispatch_async(dispatch_get_global_queue(DISPATCH_QUEUE_PRIORITY_DEFAULT, 0), ^{
        
        // Simulate progress updates
        for (int i = 1; i <= 10; i++) {
            dispatch_async(dispatch_get_main_queue(), ^{
                if ([self->_delegate respondsToSelector:@selector(downloadManager:didUpdateProgress:URL:)]) {
                    [self->_delegate downloadManager:self 
                                  didUpdateProgress:(double)i / 10.0 
                                               URL:url];
                }
            });
            [NSThread sleepForTimeInterval:0.1];
        }
        
        dispatch_async(dispatch_get_main_queue(), ^{
            self->_isDownloading = NO;
            
            // Simulate success
            NSString *localPath = [directory stringByAppendingPathComponent:[url lastPathComponent]];
            
            // Required method - เรียกเสมอ
            [self->_delegate downloadManager:self 
                  didFinishDownloadingURL:url 
                                 toPath:localPath];
        });
    });
}

- (void)cancelCurrentDownload {
    if (_isDownloading) {
        _isDownloading = NO;
        NSLog(@"Download cancelled");
        
        NSError *error = [NSError errorWithDomain:@"DownloadManagerDomain" 
                                             code:100 
                                         userInfo:@{NSLocalizedDescriptionKey: @"Cancelled by user"}];
        
        if ([_delegate respondsToSelector:@selector(downloadManager:didFailWithError:URL:)]) {
            [_delegate downloadManager:self 
                      didFailWithError:error 
                                   URL:_currentURL];
        }
    }
}

@end
```

```objc
// ไฟล์: ViewController.m (หรือ AppController.m)
@interface AppController : NSObject <DownloadManagerDelegate>

@property (nonatomic, strong) DownloadManager *downloadManager;

- (void)startDownload;

@end

@implementation AppController

- (instancetype)init {
    self = [super init];
    if (self) {
        _downloadManager = [[DownloadManager alloc] init];
        _downloadManager.delegate = self; // ตั้ง delegate
    }
    return self;
}

- (void)startDownload {
    NSURL *url = [NSURL URLWithString:@"https://example.com/file.pdf"];
    [_downloadManager downloadURL:url toDirectory:@"/tmp"];
}

#pragma mark - DownloadManagerDelegate (Required)

- (void)downloadManager:(DownloadManager *)manager 
didFinishDownloadingURL:(NSURL *)url 
               toPath:(NSString *)localPath {
    NSLog(@"✅ Download complete! Saved to: %@", localPath);
    // อัพเดท UI, process file, etc.
}

- (void)downloadManager:(DownloadManager *)manager 
       didFailWithError:(NSError *)error 
                   URL:(NSURL *)url {
    NSLog(@"❌ Download failed: %@", error.localizedDescription);
    // แสดง error alert, retry logic, etc.
}

#pragma mark - DownloadManagerDelegate (Optional)

- (void)downloadManager:(DownloadManager *)manager 
     didUpdateProgress:(double)progress 
                   URL:(NSURL *)url {
    NSLog(@"📥 Progress: %.0f%%", progress * 100);
    // อัพเดท progress bar
}

- (BOOL)downloadManager:(DownloadManager *)manager 
  shouldDownloadFromURL:(NSURL *)url {
    // ตรวจสอบเงื่อนไขก่อน download
    NSLog(@"Checking if should download: %@", url);
    return YES; // อนุญาตให้ download
}

@end
```

---

## 15.7 Data Source Pattern

### Data Source Pattern คืออะไร?

Data source pattern คล้าย delegate แต่เน้นที่การ provide ข้อมูล มากกว่าการ respond to events

```objc
// ตัวอย่าง Table View Data Source (เหมือน UITableViewDataSource)
@class SimpleTableView;

@protocol SimpleTableViewDataSource <NSObject>

@required
// จำนวน rows ทั้งหมด
- (NSInteger)numberOfRowsInTableView:(SimpleTableView *)tableView;

// ข้อมูลสำหรับแต่ละ row
- (NSString *)tableView:(SimpleTableView *)tableView 
         textForRowAtIndex:(NSInteger)index;

@optional
// จำนวน sections
- (NSInteger)numberOfSectionsInTableView:(SimpleTableView *)tableView;

// หัวของแต่ละ section
- (NSString *)tableView:(SimpleTableView *)tableView 
          titleForSection:(NSInteger)section;

// จำนวน rows ใน section
- (NSInteger)tableView:(SimpleTableView *)tableView 
 numberOfRowsInSection:(NSInteger)section;

@end

@protocol SimpleTableViewDelegate <NSObject>

@optional
- (void)tableView:(SimpleTableView *)tableView 
    didSelectRowAtIndex:(NSInteger)index;
- (CGFloat)tableView:(SimpleTableView *)tableView 
      heightForRowAtIndex:(NSInteger)index;

@end
```

```objc
// SimpleTableView.h
@interface SimpleTableView : NSObject

@property (nonatomic, weak) id<SimpleTableViewDataSource> dataSource;
@property (nonatomic, weak) id<SimpleTableViewDelegate> delegate;

- (void)reloadData;
- (void)selectRowAtIndex:(NSInteger)index;

@end

// SimpleTableView.m
@implementation SimpleTableView

- (void)reloadData {
    if (!_dataSource) {
        NSLog(@"Warning: No data source set");
        return;
    }
    
    NSInteger sections = 1;
    if ([_dataSource respondsToSelector:@selector(numberOfSectionsInTableView:)]) {
        sections = [_dataSource numberOfSectionsInTableView:self];
    }
    
    NSLog(@"\n=== Table View ===");
    
    for (NSInteger s = 0; s < sections; s++) {
        // แสดง section header
        if ([_dataSource respondsToSelector:@selector(tableView:titleForSection:)]) {
            NSString *title = [_dataSource tableView:self titleForSection:s];
            if (title) NSLog(@"[Section %ld: %@]", (long)s, title);
        }
        
        NSInteger rows = [_dataSource numberOfRowsInTableView:self];
        if ([_dataSource respondsToSelector:@selector(tableView:numberOfRowsInSection:)]) {
            rows = [_dataSource tableView:self numberOfRowsInSection:s];
        }
        
        for (NSInteger r = 0; r < rows; r++) {
            NSString *text = [_dataSource tableView:self textForRowAtIndex:r];
            NSLog(@"  [%ld] %@", (long)r, text);
        }
    }
    
    NSLog(@"=================\n");
}

- (void)selectRowAtIndex:(NSInteger)index {
    if ([_delegate respondsToSelector:@selector(tableView:didSelectRowAtIndex:)]) {
        [_delegate tableView:self didSelectRowAtIndex:index];
    }
}

@end
```

```objc
// ContactsViewController.m
@interface ContactsViewController : NSObject <SimpleTableViewDataSource, SimpleTableViewDelegate>

@property (nonatomic, strong) NSArray *contacts;
@property (nonatomic, strong) SimpleTableView *tableView;

@end

@implementation ContactsViewController

- (instancetype)init {
    self = [super init];
    if (self) {
        _contacts = @[
            @{@"name": @"Alice", @"phone": @"081-111-1111"},
            @{@"name": @"Bob", @"phone": @"082-222-2222"},
            @{@"name": @"Charlie", @"phone": @"083-333-3333"},
            @{@"name": @"Diana", @"phone": @"084-444-4444"},
        ];
        
        _tableView = [[SimpleTableView alloc] init];
        _tableView.dataSource = self;
        _tableView.delegate = self;
        
        [_tableView reloadData];
    }
    return self;
}

#pragma mark - SimpleTableViewDataSource

- (NSInteger)numberOfRowsInTableView:(SimpleTableView *)tableView {
    return [_contacts count];
}

- (NSString *)tableView:(SimpleTableView *)tableView textForRowAtIndex:(NSInteger)index {
    NSDictionary *contact = _contacts[index];
    return [NSString stringWithFormat:@"%@ - %@", contact[@"name"], contact[@"phone"]];
}

- (NSString *)tableView:(SimpleTableView *)tableView titleForSection:(NSInteger)section {
    return @"Contacts";
}

#pragma mark - SimpleTableViewDelegate

- (void)tableView:(SimpleTableView *)tableView didSelectRowAtIndex:(NSInteger)index {
    NSDictionary *contact = _contacts[index];
    NSLog(@"Selected: %@", contact[@"name"]);
}

@end
```

---

## 15.8 Protocol Inheritance

### Protocol สามารถ inherit จาก Protocol อื่น

```objc
// Base protocol
@protocol Identifiable <NSObject>
@required
@property (nonatomic, readonly) NSString *identifier;
@end

// Protocol ที่ inherit จาก Identifiable
@protocol Nameable <Identifiable>
@required
@property (nonatomic, copy) NSString *name;
@end

// Protocol ที่ inherit จาก Nameable
@protocol Describable <Nameable>
@required
- (NSString *)describe;

@optional
- (NSAttributedString *)attributedDescription;
@end

// Class ที่ adopt Describable ต้อง implement ทั้ง 3 protocols
@interface Product : NSObject <Describable>

@property (nonatomic, readonly) NSString *identifier;
@property (nonatomic, copy) NSString *name;
@property (nonatomic, assign) double price;

@end

@implementation Product

- (instancetype)initWithName:(NSString *)name price:(double)price {
    self = [super init];
    if (self) {
        _identifier = [[NSUUID UUID] UUIDString];
        _name = [name copy];
        _price = price;
    }
    return self;
}

// จาก Describable
- (NSString *)describe {
    return [NSString stringWithFormat:@"Product: %@ (ID: %@, Price: ฿%.2f)",
            _name, [_identifier substringToIndex:8], _price];
}

@end

// ตรวจสอบ protocol conformance
Product *p = [[Product alloc] initWithName:@"iPhone" price:35000.0];

NSLog(@"Conforms to Identifiable: %@",
      [p conformsToProtocol:@protocol(Identifiable)] ? @"YES" : @"NO"); // YES
NSLog(@"Conforms to Nameable: %@",
      [p conformsToProtocol:@protocol(Nameable)] ? @"YES" : @"NO");     // YES
NSLog(@"Conforms to Describable: %@",
      [p conformsToProtocol:@protocol(Describable)] ? @"YES" : @"NO");  // YES
NSLog(@"%@", [p describe]);
```

---

## 15.9 Protocols vs Inheritance vs Categories

### ตารางเปรียบเทียบ

```
Feature          | Protocol | Inheritance | Category
-----------------+----------+-------------+---------
Implementation   | ไม่มี    | มี          | มี
Multiple         | ได้       | ไม่ได้      | ได้หลาย
State            | ไม่มี     | มี (สืบทอด) | ไม่ได้เพิ่ม
Required/Optional| ได้       | ไม่มี       | ไม่มี
Type Checking    | Compile   | Compile     | Compile
Use Case         | Contract  | Sharing imp | Extend class
```

### เมื่อไหรควรใช้อะไร

```objc
// ใช้ Protocol เมื่อ:
// 1. ต้องการกำหนด contract โดยไม่ผูกติดกับ class hierarchy
// 2. ต้องการ multiple "inheritance" (adopt หลาย protocol)
// 3. ใช้กับ delegate/data source pattern
// 4. ต้องการ loose coupling

@protocol Drawable <NSObject>
- (void)draw;
@end

// ทั้ง UIView, CALayer, ภาพ SVG สามารถ conform to Drawable
// โดยไม่ต้องมี common superclass

// ใช้ Inheritance เมื่อ:
// 1. มี shared implementation ที่ต้องการแชร์
// 2. ความสัมพันธ์ "is-a" ชัดเจน
// 3. ต้องการ override behavior

@interface Vehicle : NSObject
- (void)startEngine; // shared implementation
@end

@interface Car : Vehicle
// Override หรือ extend
@end

// ใช้ Category เมื่อ:
// 1. ต้องการเพิ่ม method ให้ existing class
// 2. ไม่ต้องการ subclass
// 3. Organize code into logical groups

@interface NSString (Validation)
- (BOOL)isValidEmail;
- (BOOL)isValidPhone;
@end
```

---

## 15.10 Custom Delegate Pattern แบบ Step-by-Step

### ตัวอย่าง: Form Validator

```objc
// Step 1: ประกาศ Protocol
// ไฟล์: FormValidatorDelegate.h

@class FormValidator;

@protocol FormValidatorDelegate <NSObject>

@required
// เรียกเมื่อ validation สำเร็จ
- (void)formValidator:(FormValidator *)validator 
    didValidateWithSuccess:(NSDictionary *)formData;

// เรียกเมื่อ validation ล้มเหลว
- (void)formValidator:(FormValidator *)validator 
      didFailWithErrors:(NSArray<NSString *> *)errors;

@optional
// เรียกเมื่อ field เดี่ยวๆ valid/invalid
- (void)formValidator:(FormValidator *)validator 
         field:(NSString *)fieldName 
   validationResult:(BOOL)isValid 
           message:(NSString *)message;

@end
```

```objc
// Step 2: สร้าง Class ที่มี Delegate
// ไฟล์: FormValidator.h

@interface FormValidator : NSObject

@property (nonatomic, weak) id<FormValidatorDelegate> delegate;
@property (nonatomic, strong) NSDictionary *validationRules;

- (instancetype)initWithRules:(NSDictionary *)rules;
- (void)validateForm:(NSDictionary *)formData;

@end

// ไฟล์: FormValidator.m
@implementation FormValidator

- (instancetype)initWithRules:(NSDictionary *)rules {
    self = [super init];
    if (self) {
        _validationRules = [rules copy];
    }
    return self;
}

- (void)validateForm:(NSDictionary *)formData {
    NSMutableArray *errors = [NSMutableArray array];
    
    [_validationRules enumerateKeysAndObjectsUsingBlock:^(NSString *field, 
                                                           NSDictionary *rules, 
                                                           BOOL *stop) {
        id value = formData[field];
        NSString *fieldError = nil;
        
        // ตรวจสอบ required
        if ([rules[@"required"] boolValue]) {
            if (!value || (([value isKindOfClass:[NSString class]]) && 
                          [(NSString *)value length] == 0)) {
                fieldError = [NSString stringWithFormat:@"%@ is required", field];
            }
        }
        
        // ตรวจสอบ minLength
        if (!fieldError && rules[@"minLength"] && [value isKindOfClass:[NSString class]]) {
            NSInteger minLen = [rules[@"minLength"] integerValue];
            if ([(NSString *)value length] < minLen) {
                fieldError = [NSString stringWithFormat:@"%@ must be at least %ld characters", 
                              field, (long)minLen];
            }
        }
        
        // ตรวจสอบ pattern (email, phone)
        if (!fieldError && rules[@"pattern"]) {
            NSString *pattern = rules[@"pattern"];
            NSPredicate *pred = [NSPredicate predicateWithFormat:@"SELF MATCHES %@", pattern];
            if (![pred evaluateWithObject:value]) {
                fieldError = [NSString stringWithFormat:@"%@ format is invalid", field];
            }
        }
        
        BOOL isValid = (fieldError == nil);
        
        // แจ้ง delegate เกี่ยวกับ field นี้ (optional)
        if ([self->_delegate respondsToSelector:@selector(formValidator:field:validationResult:message:)]) {
            [self->_delegate formValidator:self 
                               field:field 
                     validationResult:isValid 
                             message:fieldError ?: @"Valid"];
        }
        
        if (fieldError) {
            [errors addObject:fieldError];
        }
    }];
    
    // แจ้งผล (required)
    if ([errors count] == 0) {
        [_delegate formValidator:self didValidateWithSuccess:formData];
    } else {
        [_delegate formValidator:self didFailWithErrors:[errors copy]];
    }
}

@end
```

```objc
// Step 3: ใช้ Delegate
// ไฟล์: RegistrationController.m

@interface RegistrationController : NSObject <FormValidatorDelegate>

@property (nonatomic, strong) FormValidator *validator;

- (void)submitRegistrationForm:(NSDictionary *)formData;

@end

@implementation RegistrationController

- (instancetype)init {
    self = [super init];
    if (self) {
        // กำหนด validation rules
        NSDictionary *rules = @{
            @"username": @{
                @"required": @YES,
                @"minLength": @4
            },
            @"email": @{
                @"required": @YES,
                @"pattern": @"[A-Za-z0-9._%+-]+@[A-Za-z0-9.-]+\\.[A-Za-z]{2,}"
            },
            @"password": @{
                @"required": @YES,
                @"minLength": @8
            }
        };
        
        _validator = [[FormValidator alloc] initWithRules:rules];
        _validator.delegate = self;
    }
    return self;
}

- (void)submitRegistrationForm:(NSDictionary *)formData {
    NSLog(@"Submitting registration...");
    [_validator validateForm:formData];
}

#pragma mark - FormValidatorDelegate

- (void)formValidator:(FormValidator *)validator 
    didValidateWithSuccess:(NSDictionary *)formData {
    NSLog(@"✅ Registration successful!");
    NSLog(@"   Username: %@", formData[@"username"]);
    NSLog(@"   Email: %@", formData[@"email"]);
    // ดำเนินการสมัครสมาชิก
}

- (void)formValidator:(FormValidator *)validator 
      didFailWithErrors:(NSArray<NSString *> *)errors {
    NSLog(@"❌ Registration failed with %lu errors:", (unsigned long)[errors count]);
    for (NSString *error in errors) {
        NSLog(@"   • %@", error);
    }
}

- (void)formValidator:(FormValidator *)validator 
         field:(NSString *)fieldName 
   validationResult:(BOOL)isValid 
           message:(NSString *)message {
    NSLog(@"  Field '%@': %@ (%@)", fieldName, isValid ? @"✅" : @"❌", message);
}

@end

// main.m
int main(int argc, const char * argv[]) {
    @autoreleasepool {
        RegistrationController *ctrl = [[RegistrationController alloc] init];
        
        // ทดสอบ valid form
        NSLog(@"\n=== Valid Form ===");
        [ctrl submitRegistrationForm:@{
            @"username": @"johndoe",
            @"email": @"john@example.com",
            @"password": @"securepass123"
        }];
        
        // ทดสอบ invalid form
        NSLog(@"\n=== Invalid Form ===");
        [ctrl submitRegistrationForm:@{
            @"username": @"jo",         // too short
            @"email": @"not-an-email",  // invalid format
            @"password": @""            // empty
        }];
    }
    return 0;
}
```

---

## 15.11 Protocol กับ Properties

### ประกาศ Properties ใน Protocol

```objc
@protocol Configurable <NSObject>

@required
// Property ที่ต้อง implement
@property (nonatomic, copy) NSString *configKey;
@property (nonatomic, readonly) NSDictionary *defaultValues;

@optional
// Property ที่อาจ implement
@property (nonatomic, strong) NSDictionary *currentValues;

// Method ที่ใช้กับ configuration
- (void)configure:(NSDictionary *)settings;
- (id)valueForKey:(NSString *)key;

@end

// Implement
@interface DatabaseConfig : NSObject <Configurable>

@property (nonatomic, copy) NSString *configKey;
@property (nonatomic, readonly) NSDictionary *defaultValues;
@property (nonatomic, strong) NSDictionary *currentValues;

@end

@implementation DatabaseConfig {
    NSMutableDictionary *_currentValues;
}

- (instancetype)init {
    self = [super init];
    if (self) {
        _configKey = @"database";
        _defaultValues = @{
            @"host": @"localhost",
            @"port": @5432,
            @"name": @"mydb"
        };
        _currentValues = [_defaultValues mutableCopy];
    }
    return self;
}

- (void)configure:(NSDictionary *)settings {
    [_currentValues addEntriesFromDictionary:settings];
}

- (id)valueForKey:(NSString *)key {
    return _currentValues[key] ?: _defaultValues[key];
}

@end
```

---

## 15.12 Real Examples: UITableViewDelegate Pattern

### จำลอง UITableViewDelegate และ UITableViewDataSource

```objc
// ตัวอย่างที่ใกล้เคียงกับ iOS UIKit

@class ListView;

// Data Source Protocol
@protocol ListViewDataSource <NSObject>

@required
- (NSInteger)listView:(ListView *)listView numberOfItemsInSection:(NSInteger)section;
- (NSString *)listView:(ListView *)listView cellTextForIndexPath:(NSIndexPath *)indexPath;

@optional
- (NSInteger)numberOfSectionsInListView:(ListView *)listView;
- (NSString *)listView:(ListView *)listView titleForHeaderInSection:(NSInteger)section;
- (BOOL)listView:(ListView *)listView canEditItemAtIndexPath:(NSIndexPath *)indexPath;

@end

// Delegate Protocol
@protocol ListViewDelegate <NSObject>

@optional
- (void)listView:(ListView *)listView didSelectItemAtIndexPath:(NSIndexPath *)indexPath;
- (void)listView:(ListView *)listView didDeselectItemAtIndexPath:(NSIndexPath *)indexPath;
- (CGFloat)listView:(ListView *)listView heightForRowAtIndexPath:(NSIndexPath *)indexPath;
- (void)listView:(ListView *)listView willDisplayCell:(NSString *)cell 
    forItemAtIndexPath:(NSIndexPath *)indexPath;
- (void)listView:(ListView *)listView commitEditingStyle:(NSInteger)style 
    forItemAtIndexPath:(NSIndexPath *)indexPath;

@end

// ListView Implementation
@interface ListView : NSObject

@property (nonatomic, weak) id<ListViewDataSource> dataSource;
@property (nonatomic, weak) id<ListViewDelegate> delegate;
@property (nonatomic, strong) NSIndexPath *selectedIndexPath;

- (void)reloadData;
- (void)selectItemAtIndexPath:(NSIndexPath *)indexPath;
- (void)deselectCurrentItem;

@end

@implementation ListView

- (void)reloadData {
    if (!_dataSource) {
        NSLog(@"ListView: No data source!");
        return;
    }
    
    NSInteger sections = 1;
    if ([_dataSource respondsToSelector:@selector(numberOfSectionsInListView:)]) {
        sections = [_dataSource numberOfSectionsInListView:self];
    }
    
    NSLog(@"\n╔══════════════════════╗");
    NSLog(@"║    LIST VIEW         ║");
    NSLog(@"╠══════════════════════╣");
    
    for (NSInteger s = 0; s < sections; s++) {
        if ([_dataSource respondsToSelector:@selector(listView:titleForHeaderInSection:)]) {
            NSString *header = [_dataSource listView:self titleForHeaderInSection:s];
            if (header) {
                NSLog(@"║ ▸ Section: %-11@ ║", header);
                NSLog(@"╠══════════════════════╣");
            }
        }
        
        NSIndexPath *path = [NSIndexPath indexPathForRow:0 inSection:s];
        NSInteger count = [_dataSource listView:self numberOfItemsInSection:s];
        
        for (NSInteger r = 0; r < count; r++) {
            NSIndexPath *indexPath = [NSIndexPath indexPathForRow:r inSection:s];
            NSString *text = [_dataSource listView:self cellTextForIndexPath:indexPath];
            
            // แจ้ง delegate ก่อน display
            if ([_delegate respondsToSelector:@selector(listView:willDisplayCell:forItemAtIndexPath:)]) {
                [_delegate listView:self willDisplayCell:text forItemAtIndexPath:indexPath];
            }
            
            NSString *selectedMark = ([_selectedIndexPath isEqual:indexPath]) ? @"►" : @" ";
            NSLog(@"║ %@ %-19@ ║", selectedMark, text);
        }
    }
    
    NSLog(@"╚══════════════════════╝");
}

- (void)selectItemAtIndexPath:(NSIndexPath *)indexPath {
    NSIndexPath *previous = _selectedIndexPath;
    _selectedIndexPath = indexPath;
    
    if (previous && [_delegate respondsToSelector:@selector(listView:didDeselectItemAtIndexPath:)]) {
        [_delegate listView:self didDeselectItemAtIndexPath:previous];
    }
    
    if ([_delegate respondsToSelector:@selector(listView:didSelectItemAtIndexPath:)]) {
        [_delegate listView:self didSelectItemAtIndexPath:indexPath];
    }
}

- (void)deselectCurrentItem {
    if (_selectedIndexPath) {
        if ([_delegate respondsToSelector:@selector(listView:didDeselectItemAtIndexPath:)]) {
            [_delegate listView:self didDeselectItemAtIndexPath:_selectedIndexPath];
        }
        _selectedIndexPath = nil;
    }
}

@end

// Controller ที่ implement ทั้ง DataSource และ Delegate
@interface ProductListController : NSObject <ListViewDataSource, ListViewDelegate>

@property (nonatomic, strong) NSArray<NSDictionary *> *products;
@property (nonatomic, strong) ListView *listView;

@end

@implementation ProductListController

- (instancetype)init {
    self = [super init];
    if (self) {
        _products = @[
            @{@"name": @"iPhone 15", @"price": @35000, @"category": @"Phones"},
            @{@"name": @"MacBook Pro", @"price": @85000, @"category": @"Computers"},
            @{@"name": @"iPad Air", @"price": @25000, @"category": @"Tablets"},
            @{@"name": @"AirPods Pro", @"price": @9000, @"category": @"Audio"},
        ];
        
        _listView = [[ListView alloc] init];
        _listView.dataSource = self;
        _listView.delegate = self;
        
        [_listView reloadData];
        
        // จำลองการ select
        [_listView selectItemAtIndexPath:[NSIndexPath indexPathForRow:1 inSection:0]];
        [_listView reloadData];
    }
    return self;
}

#pragma mark - ListViewDataSource

- (NSInteger)numberOfSectionsInListView:(ListView *)listView {
    return 1;
}

- (NSInteger)listView:(ListView *)listView numberOfItemsInSection:(NSInteger)section {
    return [_products count];
}

- (NSString *)listView:(ListView *)listView cellTextForIndexPath:(NSIndexPath *)indexPath {
    NSDictionary *product = _products[indexPath.row];
    return [NSString stringWithFormat:@"%@ ฿%@", product[@"name"], product[@"price"]];
}

- (NSString *)listView:(ListView *)listView titleForHeaderInSection:(NSInteger)section {
    return @"Products";
}

#pragma mark - ListViewDelegate

- (void)listView:(ListView *)listView didSelectItemAtIndexPath:(NSIndexPath *)indexPath {
    NSDictionary *product = _products[indexPath.row];
    NSLog(@"Selected: %@", product[@"name"]);
}

- (void)listView:(ListView *)listView didDeselectItemAtIndexPath:(NSIndexPath *)indexPath {
    NSDictionary *product = _products[indexPath.row];
    NSLog(@"Deselected: %@", product[@"name"]);
}

@end
```

---

## แบบฝึกหัด

### ระดับเริ่มต้น

**แบบฝึกหัดที่ 1**: Logger Protocol
```objc
// สร้าง protocol Loggable
@protocol Loggable <NSObject>
@required
- (void)logInfo:(NSString *)message;
- (void)logError:(NSString *)message;
@optional
- (void)logDebug:(NSString *)message;
- (void)logWarning:(NSString *)message;
@end

// สร้าง implementations:
// - ConsoleLogger (log ไปที่ console)
// - FileLogger (log ไปที่ file - simulate ด้วย array)
// - MultiLogger (log ไปหลายที่พร้อมกัน)
```

**แบบฝึกหัดที่ 2**: Sortable Protocol
```objc
// สร้าง protocol Sortable
@protocol Sortable <NSObject>
@required
- (NSComparisonResult)compareWith:(id<Sortable>)other;
- (id)sortKey;
@end

// Implement ให้กับ Product class
// เขียน sort function ที่รับ NSArray<id<Sortable>>
```

**แบบฝึกหัดที่ 3**: Custom Alert Delegate
```objc
// สร้าง AlertView class พร้อม delegate
// ที่มี:
// - alertViewDidConfirm:
// - alertViewDidCancel:
// - alertView:didSelectButtonAtIndex:
```

### ระดับกลาง

**แบบฝึกหัดที่ 4**: Network Request Delegate
```objc
@protocol NetworkDelegate <NSObject>
@required
- (void)networkRequest:(id)request didSucceedWithData:(NSData *)data;
- (void)networkRequest:(id)request didFailWithError:(NSError *)error;
@optional
- (void)networkRequest:(id)request didUpdateProgress:(double)progress;
- (NSDictionary *)headersForNetworkRequest:(id)request;
@end

// สร้าง NetworkManager ที่ใช้ delegate
// สร้าง APIController ที่ implement delegate
```

**แบบฝึกหัดที่ 5**: Shopping Cart Data Source
```objc
// สร้าง ShoppingCart ที่ใช้ data source pattern
// Data source provide: items, quantities, prices
// Delegate handle: item selected, quantity changed, checkout

// ใช้ pattern ที่คล้ายกับ UITableView
```

**แบบฝึกหัดที่ 6**: Serializable Protocol
```objc
@protocol JSONSerializable <NSObject>
@required
- (NSDictionary *)toJSON;
+ (instancetype)fromJSON:(NSDictionary *)json;
@optional
- (BOOL)validateJSON:(NSDictionary *)json;
@end

// Implement ให้กับ classes: User, Product, Order
// สร้าง serialization helper ที่ทำงานกับ id<JSONSerializable>
```

### ระดับสูง

**แบบฝึกหัดที่ 7**: Event System
```objc
@protocol EventListener <NSObject>
@required
- (void)handleEvent:(NSDictionary *)event;
@optional
- (NSArray<NSString *> *)subscribedEventTypes;
- (BOOL)shouldHandleEvent:(NSDictionary *)event;
@end

// สร้าง EventBus ที่รับ listener หลายตัว
// Dispatch events ไปยัง listener ที่ subscribe
// รองรับ wildcard subscriptions
```

**แบบฝึกหัดที่ 8**: Plugin Protocol
```objc
@protocol AppPlugin <NSObject>
@required
@property (nonatomic, readonly) NSString *pluginIdentifier;
@property (nonatomic, readonly) NSString *pluginName;
- (void)pluginDidLoad;
- (void)pluginWillUnload;
- (void)execute;
@optional
@property (nonatomic, readonly) NSArray<NSString *> *requiredPermissions;
- (NSDictionary *)pluginInfo;
@end

// สร้าง PluginManager ที่ register/unregister plugins
// Execute plugins ตาม order ที่กำหนด
```

**แบบฝึกหัดที่ 9**: Validation Chain
```objc
@protocol Validator <NSObject>
@required
- (BOOL)validate:(id)value error:(NSError **)error;
@property (nonatomic, readonly) NSString *validatorName;
@optional
- (NSString *)describeRule;
@end

// สร้าง Validator types:
// RequiredValidator, MinLengthValidator, MaxLengthValidator
// EmailValidator, RangeValidator, PatternValidator

// สร้าง ValidationChain ที่รัน validators ต่อเนื่อง
```

**แบบฝึกหัดที่ 10**: State Machine Delegate
```objc
@protocol StateMachineDelegate <NSObject>
@required
- (void)stateMachine:(id)machine didTransitionFromState:(NSString *)from toState:(NSString *)to;
@optional
- (BOOL)stateMachine:(id)machine shouldTransitionFromState:(NSString *)from toState:(NSString *)to;
- (void)stateMachineDidEnterFinalState:(id)machine;
@end

// สร้าง StateMachine generic class
// ใช้ delegate เพื่อควบคุมและ monitor transitions
// ตัวอย่าง: Order state machine (pending → processing → shipped → delivered)
```

---

## สรุปบทเรียน

ในบทนี้เราได้เรียนรู้:
- **Protocol คืออะไร**: contract ที่กำหนด interface โดยไม่มี implementation
- **@required vs @optional**: การกำหนดว่า method ใดต้อง implement
- **Protocol Adoption**: วิธีที่ class adopt protocol และ implement methods
- **conformsToProtocol:**: การตรวจสอบว่า object conform to protocol
- **Delegate Pattern**: รูปแบบ callback ที่ใช้กันอย่างแพร่หลายใน iOS
- **Data Source Pattern**: การ provide ข้อมูลผ่าน protocol
- **Protocol Inheritance**: Protocol สามารถ inherit จาก protocol อื่น
- **Protocols vs Inheritance**: เมื่อไหรควรใช้อะไร

> **Best Practice**: ตั้งชื่อ delegate methods ให้ส่ง sender เสมอเป็น argument แรก เช่น `downloadManager:didFinish:` แทนที่จะเป็น `downloadDidFinish:` เพื่อให้ delegate รู้ว่า event มาจาก object ใด
