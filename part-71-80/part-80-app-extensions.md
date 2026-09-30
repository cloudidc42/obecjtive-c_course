# Part 80: App Extensions ใน Objective-C

## บทนำ

**App Extensions** คือ Module ขนาดเล็กที่ขยายฟังก์ชันของแอปหลัก หรือผสานรวมกับระบบ iOS ได้โดยตรง ผู้ใช้สามารถเข้าถึง Extension ได้จากแอปอื่น หรือจากระบบโดยตรง โดยไม่ต้องเปิดแอปหลัก

ประเภทของ App Extensions:
- **Today Widget** - แสดงข้อมูลในหน้า Notification Center
- **Share Extension** - แชร์เนื้อหาจากแอปอื่น
- **Action Extension** - แปลงหรือปรับแต่งเนื้อหา
- **Custom Keyboard** - แทนที่ Keyboard ของ iOS
- **Notification Content** - ปรับแต่ง UI ของ Notification
- **Notification Service** - ปรับแก้ Notification ก่อนแสดง

บทนี้จะครอบคลุม:
- สถาปัตยกรรมของ App Extensions
- Today Extension (Widget)
- Share Extension
- App Groups สำหรับแชร์ข้อมูล
- Custom Keyboard Extension
- Notification Content Extension
- Notification Service Extension
- การสื่อสารระหว่าง App และ Extension

---

# หมวดที่ 1: Extension Architecture

## 80.1 ทำความเข้าใจ Extension Architecture

```
// โครงสร้างของ App Extension:
//
// Host App (แอปที่เรียกใช้ Extension)
//     ↓ (ผ่าน Extension Point)
// Extension Process (แยก Process จาก Host App)
//     ↓ (ผ่าน App Groups / NSExtensionContext)
// Containing App (แอปหลักที่มี Extension อยู่)
//
// ข้อจำกัดสำคัญ:
// - Extension และ App หลักรันใน Process แยกกัน
// - Extension มี Memory Limit ต่ำกว่า App หลัก
// - Extension ไม่สามารถ Access UI ของ App หลักได้โดยตรง
// - ใช้ App Groups สำหรับแชร์ข้อมูล
```

```objc
// ExtensionInfo.m - ข้อมูลพื้นฐานของ Extension
@implementation ExtensionInfo

/*
 ชนิดของ Extension และ NSExtensionPointIdentifier:
 
 Today Extension (Widget):
   com.apple.widget-extension
   
 Share Extension:
   com.apple.share-services
   
 Action Extension:
   com.apple.ui-services
   
 Custom Keyboard:
   com.apple.keyboard-service
   
 Notification Content Extension:
   com.apple.usernotifications.content-extension
   
 Notification Service Extension:
   com.apple.usernotifications.service
   
 iMessage App Extension:
   com.apple.message-payload-provider
   
 Photo Editing Extension:
   com.apple.photo-editing
*/

+ (void)showExtensionLimitations {
    NSLog(@"Extension Limitations:");
    NSLog(@"- ไม่สามารถใช้ openURL: ได้ (ยกเว้นผ่าน NSExtensionContext)");
    NSLog(@"- ไม่สามารถรัน Background Tasks ที่ยาวได้");
    NSLog(@"- Memory Limit: ~60-120 MB (ขึ้นกับ Extension type)");
    NSLog(@"- ไม่สามารถใช้ APIs บางตัว เช่น HealthKit, HomeKit (ต้องขอ Permission)");
    NSLog(@"- ไม่สามารถ Access Camera/Microphone ใน Today Extension");
}

@end
```

## 80.2 Info.plist ของ Extension

```xml
<!-- Extension's Info.plist -->
<key>NSExtension</key>
<dict>
    <!-- Extension Point ที่ต้องการใช้ -->
    <key>NSExtensionPointIdentifier</key>
    <string>com.apple.widget-extension</string>
    
    <!-- Principal Class - Class หลักของ Extension -->
    <key>NSExtensionPrincipalClass</key>
    <string>TodayViewController</string>
    
    <!-- สำหรับ Share/Action Extension -->
    <!-- 
    <key>NSExtensionActivationRule</key>
    <dict>
        <key>NSExtensionActivationSupportsWebURLWithMaxCount</key>
        <integer>1</integer>
        <key>NSExtensionActivationSupportsImageWithMaxCount</key>
        <integer>5</integer>
        <key>NSExtensionActivationSupportsText</key>
        <true/>
    </dict>
    -->
</dict>
```

---

# หมวดที่ 2: Today Extension (Widget)

## 80.3 Today Extension พื้นฐาน

**Today Extension** หรือ Widget แสดงข้อมูลในหน้า Notification Center (Today View) ช่วยให้ผู้ใช้เห็นข้อมูลสำคัญได้โดยไม่ต้องเปิดแอป

```objc
// TodayViewController.h
#import <UIKit/UIKit.h>
#import <NotificationCenter/NotificationCenter.h>

@interface TodayViewController : UIViewController <NCWidgetProviding>

@property (nonatomic, strong) UILabel *titleLabel;
@property (nonatomic, strong) UILabel *subtitleLabel;
@property (nonatomic, strong) UILabel *statsLabel;
@property (nonatomic, strong) UIButton *openAppButton;

@end
```

```objc
// TodayViewController.m
#import "TodayViewController.h"

@interface TodayViewController ()
@property (nonatomic, strong) NSDate *lastUpdated;
@end

@implementation TodayViewController

- (void)viewDidLoad {
    [super viewDidLoad];
    [self setupUI];
    [self loadData];
    
    // กำหนดขนาดของ Widget
    // Compact Mode: ขนาดเล็ก (ประมาณ 110pt)
    // Expanded Mode: ขนาดใหญ่ (กำหนดเองได้)
    self.preferredContentSize = CGSizeMake(0, 110);
}

- (void)setupUI {
    self.view.backgroundColor = [UIColor clearColor];
    
    // Title
    self.titleLabel = [[UILabel alloc] init];
    self.titleLabel.font = [UIFont boldSystemFontOfSize:18];
    self.titleLabel.textColor = [UIColor labelColor];
    self.titleLabel.text = @"สถิติวันนี้";
    self.titleLabel.translatesAutoresizingMaskIntoConstraints = NO;
    [self.view addSubview:self.titleLabel];
    
    // Stats
    self.statsLabel = [[UILabel alloc] init];
    self.statsLabel.font = [UIFont systemFontOfSize:14];
    self.statsLabel.textColor = [UIColor secondaryLabelColor];
    self.statsLabel.numberOfLines = 0;
    self.statsLabel.translatesAutoresizingMaskIntoConstraints = NO;
    [self.view addSubview:self.statsLabel];
    
    // Open App Button
    self.openAppButton = [UIButton buttonWithType:UIButtonTypeSystem];
    [self.openAppButton setTitle:@"เปิดแอป →" forState:UIControlStateNormal];
    self.openAppButton.translatesAutoresizingMaskIntoConstraints = NO;
    [self.openAppButton addTarget:self 
                           action:@selector(openMainApp) 
                 forControlEvents:UIControlEventTouchUpInside];
    [self.view addSubview:self.openAppButton];
    
    // Layout
    [NSLayoutConstraint activateConstraints:@[
        [self.titleLabel.topAnchor constraintEqualToAnchor:self.view.topAnchor constant:12],
        [self.titleLabel.leadingAnchor constraintEqualToAnchor:self.view.leadingAnchor constant:16],
        
        [self.statsLabel.topAnchor constraintEqualToAnchor:self.titleLabel.bottomAnchor constant:8],
        [self.statsLabel.leadingAnchor constraintEqualToAnchor:self.view.leadingAnchor constant:16],
        [self.statsLabel.trailingAnchor constraintEqualToAnchor:self.view.trailingAnchor constant:-16],
        
        [self.openAppButton.trailingAnchor constraintEqualToAnchor:self.view.trailingAnchor constant:-16],
        [self.openAppButton.topAnchor constraintEqualToAnchor:self.view.topAnchor constant:8],
    ]];
}

#pragma mark - NCWidgetProviding

// เรียกเมื่อ Widget ต้องการอัปเดตข้อมูล
- (void)widgetPerformUpdateWithCompletionHandler:(void(^)(NCUpdateResult))completionHandler {
    [self loadData];
    
    // แจ้งผลการอัปเดต
    if ([self hasNewData]) {
        completionHandler(NCUpdateResultNewData);
    } else {
        completionHandler(NCUpdateResultNoData);
    }
}

// กำหนดขนาด Widget ในแต่ละ Mode
- (void)widgetActiveDisplayModeDidChange:(NCWidgetDisplayMode)activeDisplayMode 
                         withMaximumSize:(CGSize)maxSize {
    if (activeDisplayMode == NCWidgetDisplayModeCompact) {
        self.preferredContentSize = CGSizeMake(maxSize.width, 110);
    } else {
        // Expanded mode - แสดงข้อมูลมากขึ้น
        self.preferredContentSize = CGSizeMake(maxSize.width, 220);
        [self showExpandedContent];
    }
}

#pragma mark - Data Loading

- (void)loadData {
    // โหลดข้อมูลจาก App Group Shared Container
    NSUserDefaults *sharedDefaults = [[NSUserDefaults alloc] 
                                      initWithSuiteName:@"group.com.company.myapp"];
    
    NSInteger todayCount = [sharedDefaults integerForKey:@"today_count"];
    NSInteger totalCount = [sharedDefaults integerForKey:@"total_count"];
    NSDate *lastSync = [sharedDefaults objectForKey:@"last_sync_date"];
    
    // อัปเดต UI
    dispatch_async(dispatch_get_main_queue(), ^{
        self.statsLabel.text = [NSString stringWithFormat:
                                @"วันนี้: %ld รายการ\nทั้งหมด: %ld รายการ",
                                todayCount, totalCount];
        
        if (lastSync) {
            NSDateFormatter *formatter = [[NSDateFormatter alloc] init];
            formatter.dateStyle = NSDateFormatterNoStyle;
            formatter.timeStyle = NSDateFormatterShortStyle;
            formatter.doesRelativeDateFormatting = YES;
            
            NSLog(@"Last sync: %@", [formatter stringFromDate:lastSync]);
        }
    });
    
    self.lastUpdated = [NSDate date];
}

- (BOOL)hasNewData {
    // ตรวจสอบว่ามีข้อมูลใหม่หรือไม่
    if (!self.lastUpdated) return YES;
    
    NSTimeInterval elapsed = [[NSDate date] timeIntervalSinceDate:self.lastUpdated];
    return elapsed > 300; // อัปเดตทุก 5 นาที
}

- (void)showExpandedContent {
    // แสดงเนื้อหาเพิ่มเติมในโหมด Expanded
    NSLog(@"แสดง Expanded Content");
}

#pragma mark - Actions

- (void)openMainApp {
    // เปิดแอปหลัก - ใช้ URL Scheme หรือ Universal Link
    NSURL *appURL = [NSURL URLWithString:@"myapp://widget-open?source=widget"];
    
    if (@available(iOS 12.0, *)) {
        // iOS 12+ ใช้ extensionContext
        [self.extensionContext openURL:appURL completionHandler:^(BOOL success) {
            NSLog(@"Open app: %@", success ? @"success" : @"failed");
        }];
    } else {
        // ก่อน iOS 12
        UIResponder *responder = self;
        while ((responder = [responder nextResponder]) != nil) {
            if ([responder respondsToSelector:@selector(openURL:)]) {
                [responder performSelector:@selector(openURL:) withObject:appURL];
                break;
            }
        }
    }
}

@end
```

---

# หมวดที่ 3: Share Extension

## 80.4 Share Extension

**Share Extension** ช่วยให้ผู้ใช้แชร์เนื้อหา (รูปภาพ, ข้อความ, URL) จากแอปอื่นมายังแอปของเรา

```objc
// ShareViewController.h
#import <UIKit/UIKit.h>
#import <Social/Social.h>

@interface ShareViewController : SLComposeServiceViewController

@end
```

```objc
// ShareViewController.m
// แบบ Simple - ใช้ SLComposeServiceViewController (UI สำเร็จรูป)

#import "ShareViewController.h"
#import <MobileCoreServices/MobileCoreServices.h>

@interface ShareViewController ()
@property (nonatomic, copy) NSURL *sharedURL;
@property (nonatomic, strong) UIImage *sharedImage;
@property (nonatomic, copy) NSString *sharedText;
@end

@implementation ShareViewController

- (void)viewDidLoad {
    [super viewDidLoad];
    
    self.title = @"บันทึกไปยัง MyApp";
    self.placeholder = @"เพิ่มหมายเหตุ...";
    
    // โหลดข้อมูลที่แชร์มา
    [self loadSharedData];
}

// กำหนดว่าจะใช้ Share Sheet ได้หรือไม่
// (ตรวจสอบ Input ก่อน)
- (BOOL)isContentValid {
    // ตรวจสอบว่ามีข้อมูลที่ต้องการหรือไม่
    return self.sharedURL != nil || self.sharedImage != nil || self.sharedText.length > 0;
}

// ชื่อ Placeholder ใน Text Field
- (NSString *)placeholder {
    return @"เพิ่มคำอธิบาย...";
}

- (void)loadSharedData {
    NSExtensionItem *extensionItem = self.extensionContext.inputItems.firstObject;
    
    for (NSItemProvider *provider in extensionItem.attachments) {
        
        // ตรวจสอบ URL
        if ([provider hasItemConformingToTypeIdentifier:(NSString *)kUTTypeURL]) {
            [provider loadItemForTypeIdentifier:(NSString *)kUTTypeURL 
                                        options:nil 
                              completionHandler:^(NSURL *url, NSError *error) {
                dispatch_async(dispatch_get_main_queue(), ^{
                    self.sharedURL = url;
                    NSLog(@"Shared URL: %@", url);
                });
            }];
        }
        
        // ตรวจสอบ Image
        if ([provider hasItemConformingToTypeIdentifier:(NSString *)kUTTypeImage]) {
            [provider loadItemForTypeIdentifier:(NSString *)kUTTypeImage 
                                        options:nil 
                              completionHandler:^(UIImage *image, NSError *error) {
                dispatch_async(dispatch_get_main_queue(), ^{
                    self.sharedImage = image;
                    NSLog(@"Shared Image: %@", NSStringFromCGSize(image.size));
                });
            }];
        }
        
        // ตรวจสอบ Text
        if ([provider hasItemConformingToTypeIdentifier:(NSString *)kUTTypeText]) {
            [provider loadItemForTypeIdentifier:(NSString *)kUTTypeText 
                                        options:nil 
                              completionHandler:^(NSString *text, NSError *error) {
                dispatch_async(dispatch_get_main_queue(), ^{
                    self.sharedText = text;
                });
            }];
        }
    }
}

// เรียกเมื่อผู้ใช้กด Post/Save
- (void)didSelectPost {
    NSString *comment = self.contentText;
    
    // บันทึกข้อมูลไปยัง App Group Shared Container
    [self saveSharedContentWithComment:comment];
    
    // ปิด Extension
    [self.extensionContext completeRequestReturningItems:@[] 
                                      completionHandler:nil];
}

- (void)saveSharedContentWithComment:(NSString *)comment {
    NSUserDefaults *sharedDefaults = [[NSUserDefaults alloc] 
                                      initWithSuiteName:@"group.com.company.myapp"];
    
    NSMutableArray *savedItems = [[sharedDefaults arrayForKey:@"saved_shares"] mutableCopy] 
                                 ?: [NSMutableArray array];
    
    NSMutableDictionary *item = [NSMutableDictionary dictionary];
    item[@"timestamp"] = [NSDate date];
    item[@"comment"] = comment ?: @"";
    
    if (self.sharedURL) {
        item[@"url"] = self.sharedURL.absoluteString;
        item[@"type"] = @"url";
    } else if (self.sharedImage) {
        // บันทึกรูปภาพไปยัง Shared Container
        NSString *imageName = [NSString stringWithFormat:@"share_%@.jpg", 
                               [[NSUUID UUID] UUIDString]];
        NSString *imagePath = [self sharedContainerPath:imageName];
        
        NSData *imageData = UIImageJPEGRepresentation(self.sharedImage, 0.8);
        [imageData writeToFile:imagePath atomically:YES];
        
        item[@"image_path"] = imageName;
        item[@"type"] = @"image";
    } else if (self.sharedText) {
        item[@"text"] = self.sharedText;
        item[@"type"] = @"text";
    }
    
    [savedItems addObject:[item copy]];
    [sharedDefaults setObject:[savedItems copy] forKey:@"saved_shares"];
    [sharedDefaults synchronize];
    
    NSLog(@"✅ บันทึก Shared Content สำเร็จ: %@", item[@"type"]);
}

- (NSString *)sharedContainerPath:(NSString *)filename {
    NSURL *containerURL = [[NSFileManager defaultManager] 
                           containerURLForSecurityApplicationGroupIdentifier:@"group.com.company.myapp"];
    return [[containerURL path] stringByAppendingPathComponent:filename];
}

// เรียกเมื่อผู้ใช้กด Cancel
- (void)didSelectCancel {
    [self.extensionContext cancelRequestWithError:
     [NSError errorWithDomain:NSCocoaErrorDomain 
                          code:NSUserCancelledError 
                      userInfo:nil]];
}

// กำหนด Configuration Items (รายการ Options ใน Share Sheet)
- (NSArray *)configurationItems {
    SLComposeSheetConfigurationItem *item = [[SLComposeSheetConfigurationItem alloc] init];
    item.title = @"บันทึกในหมวด";
    item.value = @"ทั่วไป";
    item.tapHandler = ^{
        [self showCategoryPicker];
    };
    
    return @[item];
}

- (void)showCategoryPicker {
    // แสดง Category Picker
    UIAlertController *picker = [UIAlertController 
                                  alertControllerWithTitle:@"เลือกหมวดหมู่"
                                  message:nil
                                  preferredStyle:UIAlertControllerStyleActionSheet];
    
    NSArray *categories = @[@"ทั่วไป", @"งาน", @"ส่วนตัว", @"เรื่องน่าสนใจ"];
    
    for (NSString *category in categories) {
        [picker addAction:[UIAlertAction actionWithTitle:category 
                                                  style:UIAlertActionStyleDefault 
                                                handler:^(UIAlertAction *action) {
            NSLog(@"เลือกหมวด: %@", action.title);
        }]];
    }
    
    [picker addAction:[UIAlertAction actionWithTitle:@"ยกเลิก" 
                                              style:UIAlertActionStyleCancel 
                                            handler:nil]];
    
    [self presentViewController:picker animated:YES completion:nil];
}

@end
```

## 80.5 Custom Share Extension (ไม่ใช้ SLComposeServiceViewController)

```objc
// CustomShareViewController.h
#import <UIKit/UIKit.h>

@interface CustomShareViewController : UIViewController

@end
```

```objc
// CustomShareViewController.m - Custom UI สำหรับ Share Extension
@interface CustomShareViewController ()
@property (nonatomic, strong) UINavigationBar *navBar;
@property (nonatomic, strong) UIScrollView *contentScrollView;
@property (nonatomic, strong) UIImageView *previewImageView;
@property (nonatomic, strong) UITextView *notesTextView;
@property (nonatomic, strong) UIButton *saveButton;
@end

@implementation CustomShareViewController

- (void)viewDidLoad {
    [super viewDidLoad];
    [self setupCustomUI];
    [self loadInputData];
}

- (void)setupCustomUI {
    self.view.backgroundColor = [UIColor systemBackgroundColor];
    
    // Navigation Bar
    self.navBar = [[UINavigationBar alloc] init];
    self.navBar.translatesAutoresizingMaskIntoConstraints = NO;
    
    UINavigationItem *navItem = [[UINavigationItem alloc] initWithTitle:@"บันทึกลิงก์"];
    navItem.leftBarButtonItem = [[UIBarButtonItem alloc] 
                                  initWithBarButtonSystemItem:UIBarButtonSystemItemCancel 
                                  target:self 
                                  action:@selector(cancelShare)];
    navItem.rightBarButtonItem = [[UIBarButtonItem alloc] 
                                   initWithTitle:@"บันทึก" 
                                   style:UIBarButtonItemStyleDone 
                                   target:self 
                                   action:@selector(saveShare)];
    [self.navBar setItems:@[navItem]];
    [self.view addSubview:self.navBar];
    
    // Preview Image
    self.previewImageView = [[UIImageView alloc] init];
    self.previewImageView.contentMode = UIViewContentModeScaleAspectFit;
    self.previewImageView.backgroundColor = [UIColor systemGray6Color];
    self.previewImageView.layer.cornerRadius = 8;
    self.previewImageView.clipsToBounds = YES;
    self.previewImageView.translatesAutoresizingMaskIntoConstraints = NO;
    [self.view addSubview:self.previewImageView];
    
    // Notes Text View
    self.notesTextView = [[UITextView alloc] init];
    self.notesTextView.font = [UIFont preferredFontForTextStyle:UIFontTextStyleBody];
    self.notesTextView.layer.borderColor = [UIColor systemGray4Color].CGColor;
    self.notesTextView.layer.borderWidth = 1.0;
    self.notesTextView.layer.cornerRadius = 8;
    self.notesTextView.translatesAutoresizingMaskIntoConstraints = NO;
    [self.view addSubview:self.notesTextView];
    
    [NSLayoutConstraint activateConstraints:@[
        [self.navBar.topAnchor constraintEqualToAnchor:self.view.safeAreaLayoutGuide.topAnchor],
        [self.navBar.leadingAnchor constraintEqualToAnchor:self.view.leadingAnchor],
        [self.navBar.trailingAnchor constraintEqualToAnchor:self.view.trailingAnchor],
        
        [self.previewImageView.topAnchor constraintEqualToAnchor:self.navBar.bottomAnchor constant:16],
        [self.previewImageView.leadingAnchor constraintEqualToAnchor:self.view.leadingAnchor constant:16],
        [self.previewImageView.trailingAnchor constraintEqualToAnchor:self.view.trailingAnchor constant:-16],
        [self.previewImageView.heightAnchor constraintEqualToConstant:150],
        
        [self.notesTextView.topAnchor constraintEqualToAnchor:self.previewImageView.bottomAnchor constant:16],
        [self.notesTextView.leadingAnchor constraintEqualToAnchor:self.view.leadingAnchor constant:16],
        [self.notesTextView.trailingAnchor constraintEqualToAnchor:self.view.trailingAnchor constant:-16],
        [self.notesTextView.heightAnchor constraintEqualToConstant:100],
    ]];
}

- (void)loadInputData {
    NSExtensionItem *item = self.extensionContext.inputItems.firstObject;
    NSItemProvider *provider = item.attachments.firstObject;
    
    if ([provider hasItemConformingToTypeIdentifier:(NSString *)kUTTypeURL]) {
        [provider loadItemForTypeIdentifier:(NSString *)kUTTypeURL 
                                    options:nil 
                          completionHandler:^(NSURL *url, NSError *error) {
            dispatch_async(dispatch_get_main_queue(), ^{
                NSLog(@"URL to share: %@", url.absoluteString);
                // โหลด Preview ของ URL
                [self loadPreviewForURL:url];
            });
        }];
    }
}

- (void)loadPreviewForURL:(NSURL *)url {
    // สร้าง Preview ง่ายๆ
    UILabel *urlLabel = [[UILabel alloc] initWithFrame:self.previewImageView.bounds];
    urlLabel.text = url.host;
    urlLabel.textAlignment = NSTextAlignmentCenter;
    urlLabel.font = [UIFont boldSystemFontOfSize:14];
    [self.previewImageView addSubview:urlLabel];
}

- (void)cancelShare {
    [self.extensionContext cancelRequestWithError:
     [NSError errorWithDomain:NSCocoaErrorDomain 
                          code:NSUserCancelledError 
                      userInfo:nil]];
}

- (void)saveShare {
    // บันทึกข้อมูล
    NSLog(@"บันทึก Share...");
    
    // ปิด Extension
    [self.extensionContext completeRequestReturningItems:@[] 
                                      completionHandler:nil];
}

@end
```

---

# หมวดที่ 4: App Groups และ Shared Data

## 80.6 App Groups Configuration

**App Groups** ช่วยให้แอปหลักและ Extension แชร์ข้อมูลกันได้ผ่าน Shared Container

```objc
// AppGroupsManager.h
// จัดการการแชร์ข้อมูลระหว่าง App และ Extension

#import <Foundation/Foundation.h>

static NSString *const kAppGroupIdentifier = @"group.com.company.myapp";

@interface AppGroupsManager : NSObject

+ (instancetype)sharedManager;

// NSUserDefaults ที่แชร์
@property (nonatomic, readonly, strong) NSUserDefaults *sharedDefaults;

// Path ของ Shared Container
@property (nonatomic, readonly, copy) NSString *sharedContainerPath;

// CRUD สำหรับข้อมูลที่แชร์
- (void)setValue:(id)value forKey:(NSString *)key;
- (id)valueForKey:(NSString *)key;
- (void)removeValueForKey:(NSString *)key;

// การจัดการไฟล์ใน Shared Container
- (BOOL)saveData:(NSData *)data withFilename:(NSString *)filename;
- (NSData *)loadDataWithFilename:(NSString *)filename;
- (BOOL)deleteFileWithFilename:(NSString *)filename;
- (NSArray<NSString *> *)listFiles;

@end
```

```objc
// AppGroupsManager.m
#import "AppGroupsManager.h"

@interface AppGroupsManager ()
@property (nonatomic, strong) NSUserDefaults *_sharedDefaults;
@property (nonatomic, copy) NSString *_sharedContainerPath;
@end

@implementation AppGroupsManager

+ (instancetype)sharedManager {
    static AppGroupsManager *instance = nil;
    static dispatch_once_t onceToken;
    dispatch_once(&onceToken, ^{
        instance = [[self alloc] init];
    });
    return instance;
}

- (instancetype)init {
    self = [super init];
    if (self) {
        // สร้าง NSUserDefaults ที่แชร์กัน
        __sharedDefaults = [[NSUserDefaults alloc] initWithSuiteName:kAppGroupIdentifier];
        
        if (!__sharedDefaults) {
            NSLog(@"❌ ไม่สามารถสร้าง Shared NSUserDefaults ได้ - ตรวจสอบ App Group configuration");
        }
        
        // Path ของ Shared Container
        NSURL *containerURL = [[NSFileManager defaultManager] 
                               containerURLForSecurityApplicationGroupIdentifier:kAppGroupIdentifier];
        __sharedContainerPath = containerURL.path;
        
        if (!__sharedContainerPath) {
            NSLog(@"❌ ไม่พบ Shared Container - ตรวจสอบ App Group Entitlement");
        }
    }
    return self;
}

- (NSUserDefaults *)sharedDefaults {
    return __sharedDefaults;
}

- (NSString *)sharedContainerPath {
    return __sharedContainerPath;
}

#pragma mark - Key-Value Storage

- (void)setValue:(id)value forKey:(NSString *)key {
    [self.sharedDefaults setObject:value forKey:key];
    [self.sharedDefaults synchronize];
}

- (id)valueForKey:(NSString *)key {
    return [self.sharedDefaults objectForKey:key];
}

- (void)removeValueForKey:(NSString *)key {
    [self.sharedDefaults removeObjectForKey:key];
    [self.sharedDefaults synchronize];
}

#pragma mark - File Operations

- (BOOL)saveData:(NSData *)data withFilename:(NSString *)filename {
    NSString *filePath = [self.sharedContainerPath stringByAppendingPathComponent:filename];
    return [data writeToFile:filePath atomically:YES];
}

- (NSData *)loadDataWithFilename:(NSString *)filename {
    NSString *filePath = [self.sharedContainerPath stringByAppendingPathComponent:filename];
    return [NSData dataWithContentsOfFile:filePath];
}

- (BOOL)deleteFileWithFilename:(NSString *)filename {
    NSString *filePath = [self.sharedContainerPath stringByAppendingPathComponent:filename];
    NSError *error = nil;
    BOOL success = [[NSFileManager defaultManager] removeItemAtPath:filePath error:&error];
    if (!success) {
        NSLog(@"❌ ลบไฟล์ไม่สำเร็จ: %@", error.localizedDescription);
    }
    return success;
}

- (NSArray<NSString *> *)listFiles {
    NSError *error = nil;
    NSArray *files = [[NSFileManager defaultManager] contentsOfDirectoryAtPath:self.sharedContainerPath 
                                                                         error:&error];
    return files ?: @[];
}

@end
```

## 80.7 การสื่อสารข้อมูลระหว่าง App และ Extension

```objc
// DataSyncManager.m
// จัดการการ Sync ข้อมูลระหว่าง App หลักและ Extensions

@interface DataSyncManager : NSObject

// Keys สำหรับ Shared Data
extern NSString *const kSharedKeyLastSyncDate;
extern NSString *const kSharedKeyTodayStats;
extern NSString *const kSharedKeyPendingShares;
extern NSString *const kSharedKeyUserPreferences;

+ (instancetype)sharedManager;

// App -> Extension: อัปเดตข้อมูลสำหรับ Widget
- (void)updateWidgetData:(NSDictionary *)data;

// Extension -> App: ดึงข้อมูลที่ Extension บันทึกไว้
- (NSArray *)fetchPendingSharesFromExtension;

// ทำ Pending Shares
- (void)processPendingShares;

@end

NSString *const kSharedKeyLastSyncDate = @"last_sync_date";
NSString *const kSharedKeyTodayStats = @"today_stats";
NSString *const kSharedKeyPendingShares = @"pending_shares";
NSString *const kSharedKeyUserPreferences = @"user_preferences";

@implementation DataSyncManager

+ (instancetype)sharedManager {
    static DataSyncManager *instance = nil;
    static dispatch_once_t onceToken;
    dispatch_once(&onceToken, ^{
        instance = [[self alloc] init];
    });
    return instance;
}

// App หลักอัปเดตข้อมูลสำหรับ Widget แสดง
- (void)updateWidgetData:(NSDictionary *)data {
    AppGroupsManager *groups = [AppGroupsManager sharedManager];
    
    [groups setValue:data forKey:kSharedKeyTodayStats];
    [groups setValue:[NSDate date] forKey:kSharedKeyLastSyncDate];
    
    NSLog(@"✅ อัปเดต Widget Data: %@", data);
    
    // Request ให้ Widget Reload
    // (ใน iOS 14+ ใช้ WidgetCenter)
    // [WidgetCenter.shared reloadTimelines(ofKind: "MyWidget")]
}

// ดึง Shares ที่ Extension บันทึกไว้ แล้วประมวลผลใน App หลัก
- (NSArray *)fetchPendingSharesFromExtension {
    AppGroupsManager *groups = [AppGroupsManager sharedManager];
    NSArray *shares = [groups valueForKey:kSharedKeyPendingShares];
    return shares ?: @[];
}

- (void)processPendingShares {
    NSArray *pendingShares = [self fetchPendingSharesFromExtension];
    
    if (pendingShares.count == 0) {
        NSLog(@"ไม่มี Pending Shares");
        return;
    }
    
    NSLog(@"กำลังประมวลผล %lu Pending Shares...", (unsigned long)pendingShares.count);
    
    for (NSDictionary *shareItem in pendingShares) {
        NSString *type = shareItem[@"type"];
        
        if ([type isEqualToString:@"url"]) {
            [self processURLShare:shareItem[@"url"] comment:shareItem[@"comment"]];
        } else if ([type isEqualToString:@"image"]) {
            [self processImageShare:shareItem[@"image_path"] comment:shareItem[@"comment"]];
        } else if ([type isEqualToString:@"text"]) {
            [self processTextShare:shareItem[@"text"] comment:shareItem[@"comment"]];
        }
    }
    
    // ล้าง Pending Shares หลังจากประมวลผลแล้ว
    [[AppGroupsManager sharedManager] removeValueForKey:kSharedKeyPendingShares];
    NSLog(@"✅ ประมวลผล Pending Shares เสร็จสิ้น");
}

- (void)processURLShare:(NSString *)urlString comment:(NSString *)comment {
    NSLog(@"ประมวลผล URL Share: %@", urlString);
    // TODO: บันทึก URL ลง Database
}

- (void)processImageShare:(NSString *)imagePath comment:(NSString *)comment {
    NSLog(@"ประมวลผล Image Share: %@", imagePath);
    // TODO: ย้ายรูปไปยัง App Storage
}

- (void)processTextShare:(NSString *)text comment:(NSString *)comment {
    NSLog(@"ประมวลผล Text Share: %@", text);
    // TODO: บันทึก Text Note
}

@end
```

---

# หมวดที่ 5: Custom Keyboard Extension

## 80.8 Custom Keyboard Extension

**Custom Keyboard Extension** ช่วยให้ผู้ใช้ใช้ Keyboard ที่แอปสร้างขึ้นเองในทุกแอป

```objc
// KeyboardViewController.h
#import <UIKit/UIKit.h>

@interface KeyboardViewController : UIInputViewController

@end
```

```objc
// KeyboardViewController.m
#import "KeyboardViewController.h"

@interface KeyboardViewController ()
@property (nonatomic, strong) UIButton *nextKeyboardButton;
@property (nonatomic, strong) UIView *keyboardView;
@property (nonatomic, strong) NSArray<NSArray<NSString *> *> *keyRows;
@end

@implementation KeyboardViewController

- (void)viewDidLoad {
    [super viewDidLoad];
    [self setupKeyboard];
}

- (void)setupKeyboard {
    // แถวของ Keys
    self.keyRows = @[
        @[@"ๆ", @"ไ", @"ำ", @"พ", @"ะ", @"ั", @"ี", @"ร", @"น", @"ย"],
        @[@"ฟ", @"ห", @"ก", @"ด", @"เ", @"้", @"่", @"า", @"ส", @"ว"],
        @[@"ผ", @"ป", @"แ", @"อ", @"ิ", @"ื", @"ท", @"ม", @"ใ", @"ฝ"],
    ];
    
    self.view.backgroundColor = [UIColor systemGroupedBackgroundColor];
    
    // สร้าง Keyboard View
    [self buildKeyboardLayout];
    
    // ปุ่มเปลี่ยน Keyboard
    self.nextKeyboardButton = [UIButton buttonWithType:UIButtonTypeSystem];
    [self.nextKeyboardButton setTitle:@"🌐" forState:UIControlStateNormal];
    self.nextKeyboardButton.titleLabel.font = [UIFont systemFontOfSize:20];
    [self.nextKeyboardButton addTarget:self 
                                action:@selector(handleInputModeListFromView:withEvent:) 
                      forControlEvents:UIControlEventAllTouchEvents];
    [self.view addSubview:self.nextKeyboardButton];
}

- (void)buildKeyboardLayout {
    CGFloat keyWidth = (UIScreen.mainScreen.bounds.width - 20) / 10 - 4;
    CGFloat keyHeight = 44.0;
    CGFloat verticalPadding = 8.0;
    
    for (NSInteger row = 0; row < self.keyRows.count; row++) {
        NSArray<NSString *> *rowKeys = self.keyRows[row];
        CGFloat totalWidth = rowKeys.count * (keyWidth + 4);
        CGFloat startX = (UIScreen.mainScreen.bounds.width - totalWidth) / 2;
        
        for (NSInteger col = 0; col < rowKeys.count; col++) {
            NSString *keyTitle = rowKeys[col];
            
            UIButton *keyButton = [UIButton buttonWithType:UIButtonTypeSystem];
            keyButton.frame = CGRectMake(
                startX + col * (keyWidth + 4),
                verticalPadding + row * (keyHeight + 8),
                keyWidth,
                keyHeight
            );
            
            [keyButton setTitle:keyTitle forState:UIControlStateNormal];
            keyButton.titleLabel.font = [UIFont systemFontOfSize:16];
            keyButton.backgroundColor = [UIColor systemBackgroundColor];
            keyButton.layer.cornerRadius = 5;
            keyButton.layer.shadowColor = [UIColor blackColor].CGColor;
            keyButton.layer.shadowOffset = CGSizeMake(0, 1);
            keyButton.layer.shadowOpacity = 0.3;
            keyButton.layer.shadowRadius = 0;
            
            [keyButton addTarget:self 
                          action:@selector(keyTapped:) 
                forControlEvents:UIControlEventTouchUpInside];
            
            [self.view addSubview:keyButton];
        }
    }
}

- (void)keyTapped:(UIButton *)sender {
    NSString *key = [sender titleForState:UIControlStateNormal];
    
    // แทรกข้อความเข้า Text Input
    [self.textDocumentProxy insertText:key];
    
    // Play Sound / Haptic
    [self playKeyTapFeedback];
}

- (void)playKeyTapFeedback {
    // Haptic Feedback
    UIImpactFeedbackGenerator *generator = [[UIImpactFeedbackGenerator alloc] 
                                             initWithStyle:UIImpactFeedbackStyleLight];
    [generator impactOccurred];
}

// Backspace
- (void)backspaceTapped {
    [self.textDocumentProxy deleteBackward];
}

// Return/Enter
- (void)returnTapped {
    [self.textDocumentProxy insertText:@"\n"];
}

// Space
- (void)spaceTapped {
    [self.textDocumentProxy insertText:@" "];
}

// ดูเนื้อหาใน Text Field ปัจจุบัน
- (void)checkCurrentInput {
    NSString *beforeCursor = self.textDocumentProxy.documentContextBeforeInput;
    NSString *afterCursor = self.textDocumentProxy.documentContextAfterInput;
    
    NSLog(@"Before cursor: %@", beforeCursor);
    NSLog(@"After cursor: %@", afterCursor);
    
    // ตรวจสอบ Keyboard Type
    UIKeyboardType keyboardType = self.textDocumentProxy.keyboardType;
    UIReturnKeyType returnKeyType = self.textDocumentProxy.returnKeyType;
    
    NSLog(@"Keyboard Type: %ld", (long)keyboardType);
    NSLog(@"Return Key Type: %ld", (long)returnKeyType);
}

- (void)viewWillLayoutSubviews {
    [super viewWillLayoutSubviews];
    
    // ตำแหน่งปุ่มเปลี่ยน Keyboard
    self.nextKeyboardButton.frame = CGRectMake(4, self.view.bounds.size.height - 44, 50, 40);
}

@end
```

---

# หมวดที่ 6: Notification Content Extension

## 80.9 Notification Content Extension

**Notification Content Extension** ช่วยให้ปรับแต่ง UI ของ Notification ที่ขยายออกมาได้

```objc
// NotificationViewController.h
#import <UIKit/UIKit.h>
#import <UserNotifications/UserNotifications.h>
#import <UserNotificationsUI/UserNotificationsUI.h>

@interface NotificationViewController : UIViewController <UNNotificationContentExtension>

@end
```

```objc
// NotificationViewController.m
#import "NotificationViewController.h"

@interface NotificationViewController ()
@property (nonatomic, strong) UILabel *titleLabel;
@property (nonatomic, strong) UILabel *bodyLabel;
@property (nonatomic, strong) UIImageView *heroImageView;
@property (nonatomic, strong) UIProgressView *progressView;
@property (nonatomic, strong) UILabel *progressLabel;
@end

@implementation NotificationViewController

- (void)viewDidLoad {
    [super viewDidLoad];
    [self setupCustomNotificationUI];
}

- (void)setupCustomNotificationUI {
    self.view.backgroundColor = [UIColor systemBackgroundColor];
    
    // Hero Image
    self.heroImageView = [[UIImageView alloc] init];
    self.heroImageView.contentMode = UIViewContentModeScaleAspectFill;
    self.heroImageView.clipsToBounds = YES;
    self.heroImageView.translatesAutoresizingMaskIntoConstraints = NO;
    [self.view addSubview:self.heroImageView];
    
    // Title
    self.titleLabel = [[UILabel alloc] init];
    self.titleLabel.font = [UIFont boldSystemFontOfSize:18];
    self.titleLabel.translatesAutoresizingMaskIntoConstraints = NO;
    [self.view addSubview:self.titleLabel];
    
    // Body
    self.bodyLabel = [[UILabel alloc] init];
    self.bodyLabel.font = [UIFont systemFontOfSize:14];
    self.bodyLabel.textColor = [UIColor secondaryLabelColor];
    self.bodyLabel.numberOfLines = 0;
    self.bodyLabel.translatesAutoresizingMaskIntoConstraints = NO;
    [self.view addSubview:self.bodyLabel];
    
    // Progress Bar (สำหรับ Download Notification)
    self.progressView = [[UIProgressView alloc] initWithProgressViewStyle:UIProgressViewStyleDefault];
    self.progressView.translatesAutoresizingMaskIntoConstraints = NO;
    [self.view addSubview:self.progressView];
    
    self.progressLabel = [[UILabel alloc] init];
    self.progressLabel.font = [UIFont systemFontOfSize:12];
    self.progressLabel.textColor = [UIColor secondaryLabelColor];
    self.progressLabel.translatesAutoresizingMaskIntoConstraints = NO;
    [self.view addSubview:self.progressLabel];
    
    // Layout
    [NSLayoutConstraint activateConstraints:@[
        [self.heroImageView.topAnchor constraintEqualToAnchor:self.view.topAnchor],
        [self.heroImageView.leadingAnchor constraintEqualToAnchor:self.view.leadingAnchor],
        [self.heroImageView.trailingAnchor constraintEqualToAnchor:self.view.trailingAnchor],
        [self.heroImageView.heightAnchor constraintEqualToConstant:180],
        
        [self.titleLabel.topAnchor constraintEqualToAnchor:self.heroImageView.bottomAnchor constant:12],
        [self.titleLabel.leadingAnchor constraintEqualToAnchor:self.view.leadingAnchor constant:16],
        [self.titleLabel.trailingAnchor constraintEqualToAnchor:self.view.trailingAnchor constant:-16],
        
        [self.bodyLabel.topAnchor constraintEqualToAnchor:self.titleLabel.bottomAnchor constant:8],
        [self.bodyLabel.leadingAnchor constraintEqualToAnchor:self.view.leadingAnchor constant:16],
        [self.bodyLabel.trailingAnchor constraintEqualToAnchor:self.view.trailingAnchor constant:-16],
        
        [self.progressView.topAnchor constraintEqualToAnchor:self.bodyLabel.bottomAnchor constant:16],
        [self.progressView.leadingAnchor constraintEqualToAnchor:self.view.leadingAnchor constant:16],
        [self.progressView.trailingAnchor constraintEqualToAnchor:self.view.trailingAnchor constant:-16],
        
        [self.progressLabel.topAnchor constraintEqualToAnchor:self.progressView.bottomAnchor constant:4],
        [self.progressLabel.leadingAnchor constraintEqualToAnchor:self.view.leadingAnchor constant:16],
        [self.progressLabel.bottomAnchor constraintEqualToAnchor:self.view.bottomAnchor constant:-12],
    ]];
}

#pragma mark - UNNotificationContentExtension

// เรียกเมื่อ Notification ถูกขยาย
- (void)didReceiveNotification:(UNNotification *)notification {
    UNNotificationContent *content = notification.request.content;
    
    self.titleLabel.text = content.title;
    self.bodyLabel.text = content.body;
    
    // ดึงข้อมูลจาก userInfo
    NSDictionary *userInfo = content.userInfo;
    
    NSString *type = userInfo[@"type"];
    
    if ([type isEqualToString:@"download"]) {
        [self setupDownloadNotification:userInfo];
    } else if ([type isEqualToString:@"image"]) {
        [self setupImageNotification:content.attachments];
    }
    
    // กำหนดขนาด Notification View
    // preferredContentSize จะกำหนดความสูง
}

- (void)setupDownloadNotification:(NSDictionary *)userInfo {
    float progress = [userInfo[@"progress"] floatValue];
    NSString *filename = userInfo[@"filename"];
    
    self.progressView.progress = progress;
    self.progressLabel.text = [NSString stringWithFormat:@"กำลังดาวน์โหลด %@ - %.0f%%", 
                                filename, progress * 100];
    
    // ซ่อน Hero Image
    self.heroImageView.hidden = YES;
}

- (void)setupImageNotification:(NSArray<UNNotificationAttachment *> *)attachments {
    UNNotificationAttachment *attachment = attachments.firstObject;
    
    if (attachment && [attachment.URL startAccessingSecurityScopedResource]) {
        NSData *imageData = [NSData dataWithContentsOfURL:attachment.URL];
        self.heroImageView.image = [UIImage imageWithData:imageData];
        [attachment.URL stopAccessingSecurityScopedResource];
    }
}

// เรียกเมื่อผู้ใช้กดปุ่ม Action
- (void)didReceiveNotificationResponse:(UNNotificationResponse *)response 
                     completionHandler:(void(^)(UNNotificationContentExtensionResponseOption))completion {
    
    NSString *actionIdentifier = response.actionIdentifier;
    
    if ([actionIdentifier isEqualToString:@"ACCEPT_ACTION"]) {
        NSLog(@"ผู้ใช้กด Accept");
        // อัปเดต UI ก่อน Dismiss
        self.titleLabel.text = @"ยืนยันแล้ว ✅";
        
        // Dismiss หลังจาก Delay สั้นๆ
        completion(UNNotificationContentExtensionResponseOptionDismissAndForwardAction);
        
    } else if ([actionIdentifier isEqualToString:@"DECLINE_ACTION"]) {
        NSLog(@"ผู้ใช้กด Decline");
        completion(UNNotificationContentExtensionResponseOptionDismiss);
        
    } else {
        completion(UNNotificationContentExtensionResponseOptionDoNotDismiss);
    }
}

@end
```

---

# หมวดที่ 7: Notification Service Extension

## 80.10 Notification Service Extension

**Notification Service Extension** ทำงานก่อนที่ Notification จะแสดง ช่วยให้:
- ถอดรหัส Payload ที่เข้ารหัส (End-to-End Encryption)
- ดาวน์โหลด Attachment เพิ่มเติม
- แก้ไขเนื้อหา Notification

```objc
// NotificationService.h
#import <UserNotifications/UserNotifications.h>

@interface NotificationService : UNNotificationServiceExtension

@end
```

```objc
// NotificationService.m
#import "NotificationService.h"

@interface NotificationService ()
@property (nonatomic, strong) void(^contentHandler)(UNNotificationContent *);
@property (nonatomic, strong) UNMutableNotificationContent *bestAttemptContent;
@end

@implementation NotificationService

// เรียกเมื่อได้รับ Remote Notification
- (void)didReceiveNotificationRequest:(UNNotificationRequest *)request 
                   withContentHandler:(void(^)(UNNotificationContent *))contentHandler {
    
    self.contentHandler = contentHandler;
    
    // สร้าง Mutable Content ให้แก้ไขได้
    self.bestAttemptContent = [request.content mutableCopy];
    
    NSDictionary *userInfo = request.content.userInfo;
    
    // 1. ถอดรหัส Encrypted Payload
    if (userInfo[@"encrypted_message"]) {
        [self decryptAndUpdateContent:userInfo];
        return;
    }
    
    // 2. ดาวน์โหลด Image Attachment
    NSString *imageURLString = userInfo[@"image_url"];
    if (imageURLString) {
        [self downloadImageAndAttach:imageURLString];
        return;
    }
    
    // 3. ไม่มีการปรับแก้ - ส่ง Content ต้นฉบับ
    contentHandler(self.bestAttemptContent);
}

#pragma mark - End-to-End Encryption

- (void)decryptAndUpdateContent:(NSDictionary *)userInfo {
    NSString *encryptedMessage = userInfo[@"encrypted_message"];
    NSString *senderId = userInfo[@"sender_id"];
    
    // ดึง Decryption Key จาก Keychain
    // (Keychain แชร์ได้ผ่าน App Group)
    NSUserDefaults *sharedDefaults = [[NSUserDefaults alloc] 
                                      initWithSuiteName:@"group.com.company.myapp"];
    NSData *encryptionKey = [sharedDefaults dataForKey:@"message_encryption_key"];
    
    if (!encryptionKey) {
        // ไม่มี Key - แสดง Generic Message
        self.bestAttemptContent.title = @"ข้อความใหม่";
        self.bestAttemptContent.body = @"คุณมีข้อความใหม่ที่เข้ารหัส";
        self.contentHandler(self.bestAttemptContent);
        return;
    }
    
    // ถอดรหัส (ตัวอย่างแบบ Simplified)
    dispatch_async(dispatch_get_global_queue(DISPATCH_QUEUE_PRIORITY_DEFAULT, 0), ^{
        NSString *decryptedMessage = [self decryptMessage:encryptedMessage 
                                                  withKey:encryptionKey];
        
        dispatch_async(dispatch_get_main_queue(), ^{
            if (decryptedMessage) {
                self.bestAttemptContent.body = decryptedMessage;
                self.bestAttemptContent.title = [NSString stringWithFormat:@"ข้อความจาก %@", senderId];
            } else {
                self.bestAttemptContent.body = @"ไม่สามารถถอดรหัสข้อความได้";
            }
            
            self.contentHandler(self.bestAttemptContent);
        });
    });
}

- (NSString *)decryptMessage:(NSString *)encryptedBase64 withKey:(NSData *)key {
    // ถอดรหัสด้วย AES-256
    NSData *encryptedData = [[NSData alloc] initWithBase64EncodedString:encryptedBase64 
                                                                options:0];
    if (!encryptedData || encryptedData.length < 16) return nil;
    
    // แยก IV (16 bytes แรก)
    NSData *iv = [encryptedData subdataWithRange:NSMakeRange(0, 16)];
    NSData *ciphertext = [encryptedData subdataWithRange:NSMakeRange(16, encryptedData.length - 16)];
    
    // Decrypt
    // (ใช้ CryptoManager จาก Part 77)
    // NSData *decryptedData = [[CryptoManager sharedManager] decryptData:ciphertext withKey:key iv:iv error:nil];
    // return [[NSString alloc] initWithData:decryptedData encoding:NSUTF8StringEncoding];
    
    return @"Decrypted message placeholder";
}

#pragma mark - Image Attachment

- (void)downloadImageAndAttach:(NSString *)imageURLString {
    NSURL *imageURL = [NSURL URLWithString:imageURLString];
    
    NSURLSessionDownloadTask *downloadTask = [[NSURLSession sharedSession] 
        downloadTaskWithURL:imageURL 
          completionHandler:^(NSURL *location, NSURLResponse *response, NSError *error) {
        
        if (error) {
            NSLog(@"❌ ดาวน์โหลดรูปภาพไม่สำเร็จ: %@", error.localizedDescription);
            self.contentHandler(self.bestAttemptContent);
            return;
        }
        
        // ย้ายไฟล์ไปยัง Temp Directory
        NSString *tempFile = [NSTemporaryDirectory() 
                              stringByAppendingPathComponent:@"notification_image.jpg"];
        NSURL *tempURL = [NSURL fileURLWithPath:tempFile];
        
        [[NSFileManager defaultManager] moveItemAtURL:location toURL:tempURL error:nil];
        
        // สร้าง Attachment
        NSError *attachmentError = nil;
        UNNotificationAttachment *attachment = [UNNotificationAttachment 
            attachmentWithIdentifier:@"notification_image"
            URL:tempURL
            options:nil
            error:&attachmentError];
        
        if (attachment) {
            self.bestAttemptContent.attachments = @[attachment];
            NSLog(@"✅ แนบรูปภาพสำเร็จ");
        } else {
            NSLog(@"❌ สร้าง Attachment ไม่สำเร็จ: %@", attachmentError.localizedDescription);
        }
        
        self.contentHandler(self.bestAttemptContent);
    }];
    
    [downloadTask resume];
}

// เรียกเมื่อ Service Extension ทำงานนานเกินไป
// ต้องส่ง Content กลับ iOS ภายในเวลาที่กำหนด
- (void)serviceExtensionTimeWillExpire {
    // ส่ง Best Attempt Content ก่อน Expire
    self.contentHandler(self.bestAttemptContent);
}

@end
```

---

# หมวดที่ 8: Extension Lifecycle

## 80.11 Memory Limits และ Lifecycle

```objc
// ExtensionLifecycleManager.m

@implementation ExtensionLifecycleManager

/*
 Memory Limits ของ Extensions (โดยประมาณ):
 
 Today Widget:          16 MB
 Share Extension:       60 MB
 Action Extension:      60 MB
 Custom Keyboard:       50 MB
 Notification Content:  24 MB
 Notification Service:  24 MB
 
 เมื่อ Memory เกิน Limit:
 - Extension จะถูก Terminate ทันที
 - ไม่มี Crash Report เหมือน App
 - ผู้ใช้จะเห็น UI ค้างหรือ Error
*/

// Monitor Memory ใน Extension
- (void)setupMemoryWarningHandler {
    [[NSNotificationCenter defaultCenter] 
     addObserver:self 
     selector:@selector(handleMemoryWarning:) 
     name:UIApplicationDidReceiveMemoryWarningNotification 
     object:nil];
}

- (void)handleMemoryWarning:(NSNotification *)notification {
    NSLog(@"⚠️ Memory Warning ใน Extension - ล้างข้อมูลที่ไม่จำเป็น");
    
    // ล้าง Cache
    [[NSURLCache sharedURLCache] removeAllCachedResponses];
    
    // ล้าง Image Cache
    // [[SDImageCache sharedImageCache] clearMemory];
    
    // ล้างข้อมูลชั่วคราว
    [self clearTemporaryData];
}

- (void)clearTemporaryData {
    // ล้าง Temporary Files
    NSString *tempDir = NSTemporaryDirectory();
    NSArray *tempFiles = [[NSFileManager defaultManager] 
                          contentsOfDirectoryAtPath:tempDir error:nil];
    
    for (NSString *file in tempFiles) {
        NSString *filePath = [tempDir stringByAppendingPathComponent:file];
        [[NSFileManager defaultManager] removeItemAtPath:filePath error:nil];
    }
    
    NSLog(@"✅ ล้าง Temporary Data สำเร็จ");
}

// ตรวจสอบ Memory Usage ปัจจุบัน
- (long long)currentMemoryUsageBytes {
    struct task_basic_info info;
    mach_msg_type_number_t size = sizeof(info);
    kern_return_t kerr = task_info(mach_task_self(), TASK_BASIC_INFO, (task_info_t)&info, &size);
    
    if (kerr == KERN_SUCCESS) {
        return info.resident_size;
    }
    return -1;
}

@end
```

## 80.12 การสื่อสารระหว่าง Extension และ App หลัก

```objc
// AppCommunicationManager.m
// วิธีการสื่อสารระหว่าง Extension และ App หลัก

@implementation AppCommunicationManager

/*
 วิธีสื่อสาร:
 
 1. NSUserDefaults with App Groups (Simple Data)
    - ใช้สำหรับ: Settings, Counters, Small Data
    - Extension -> App: อ่าน defaults เมื่อ App เปิด
    - App -> Extension: เขียน defaults แล้ว request reload
 
 2. Shared File in App Group Container (Larger Data)
    - ใช้สำหรับ: Images, Documents, Database
    - ระวัง: Read/Write Conflicts
 
 3. URL Scheme (One-way: Extension -> App)
    - extensionContext openURL: ใน Extension
    - App รับ URL ใน application:openURL:options:
 
 4. Darwin Notifications (Real-time)
    - สำหรับแจ้ง App ว่ามีข้อมูลใหม่จาก Extension
 
 5. Core Data with App Groups
    - Shared Core Data Store
    - ต้องระวัง Concurrency
*/

// Darwin Notification สำหรับแจ้ง App
- (void)notifyAppDataChanged {
    // Extension ส่ง Notification
    CFStringRef notificationName = CFSTR("com.company.myapp.extensionDataChanged");
    CFNotificationCenterPostNotification(
        CFNotificationCenterGetDarwinNotifyCenter(),
        notificationName,
        NULL,
        NULL,
        YES
    );
}

// App รับ Darwin Notification
- (void)observeExtensionChanges {
    CFStringRef notificationName = CFSTR("com.company.myapp.extensionDataChanged");
    
    CFNotificationCenterAddObserver(
        CFNotificationCenterGetDarwinNotifyCenter(),
        (__bridge const void *)(self),
        &extensionDataChangedCallback,
        notificationName,
        NULL,
        CFNotificationSuspensionBehaviorDeliverImmediately
    );
}

static void extensionDataChangedCallback(CFNotificationCenterRef center, 
                                          void *observer, 
                                          CFStringRef name, 
                                          const void *object, 
                                          CFDictionaryRef userInfo) {
    NSLog(@"📨 App ได้รับ Darwin Notification จาก Extension");
    
    // ประมวลผลข้อมูลจาก Extension
    [[DataSyncManager sharedManager] processPendingShares];
}

@end
```

---

# แบบฝึกหัดท้ายบท

## แบบฝึกหัดที่ 1: Todo Widget

สร้าง Today Widget ที่แสดงรายการ Todo ที่ยังไม่เสร็จ

```objc
// TodoWidgetViewController.m
@interface TodoWidgetViewController : UIViewController <NCWidgetProviding>

@property (nonatomic, strong) UITableView *todoTableView;
@property (nonatomic, strong) NSArray<NSDictionary *> *pendingTodos;

@end

@implementation TodoWidgetViewController

- (void)viewDidLoad {
    [super viewDidLoad];
    [self setupUI];
    [self loadTodos];
    
    // รองรับ Expanded Mode
    self.extensionContext.widgetLargestAvailableDisplayMode = NCWidgetDisplayModeExpanded;
}

- (void)setupUI {
    self.view.backgroundColor = [UIColor clearColor];
    
    self.todoTableView = [[UITableView alloc] init];
    self.todoTableView.backgroundColor = [UIColor clearColor];
    self.todoTableView.separatorStyle = UITableViewCellSeparatorStyleNone;
    self.todoTableView.delegate = self;
    self.todoTableView.dataSource = self;
    self.todoTableView.translatesAutoresizingMaskIntoConstraints = NO;
    [self.view addSubview:self.todoTableView];
    
    [NSLayoutConstraint activateConstraints:@[
        [self.todoTableView.topAnchor constraintEqualToAnchor:self.view.topAnchor],
        [self.todoTableView.leadingAnchor constraintEqualToAnchor:self.view.leadingAnchor],
        [self.todoTableView.trailingAnchor constraintEqualToAnchor:self.view.trailingAnchor],
        [self.todoTableView.bottomAnchor constraintEqualToAnchor:self.view.bottomAnchor],
    ]];
}

- (void)loadTodos {
    // โหลดจาก App Group
    NSUserDefaults *shared = [[NSUserDefaults alloc] 
                               initWithSuiteName:@"group.com.company.todoapp"];
    NSArray *todos = [shared arrayForKey:@"todos"];
    
    // กรองเฉพาะที่ยังไม่เสร็จ
    self.pendingTodos = [todos filteredArrayUsingPredicate:
                         [NSPredicate predicateWithFormat:@"completed == NO"]];
    
    [self.todoTableView reloadData];
    
    // ปรับขนาดตามจำนวน Todo
    NSInteger displayCount = MIN(self.pendingTodos.count, 3);
    self.preferredContentSize = CGSizeMake(0, MAX(110, displayCount * 44 + 20));
}

- (void)widgetPerformUpdateWithCompletionHandler:(void(^)(NCUpdateResult))completionHandler {
    NSInteger oldCount = self.pendingTodos.count;
    [self loadTodos];
    
    if (self.pendingTodos.count != oldCount) {
        completionHandler(NCUpdateResultNewData);
    } else {
        completionHandler(NCUpdateResultNoData);
    }
}

- (void)widgetActiveDisplayModeDidChange:(NCWidgetDisplayMode)activeDisplayMode 
                         withMaximumSize:(CGSize)maxSize {
    if (activeDisplayMode == NCWidgetDisplayModeCompact) {
        self.preferredContentSize = CGSizeMake(maxSize.width, 110);
    } else {
        NSInteger count = MIN(self.pendingTodos.count, 8);
        self.preferredContentSize = CGSizeMake(maxSize.width, MAX(220, count * 44 + 20));
    }
}

// TableView DataSource
- (NSInteger)tableView:(UITableView *)tableView numberOfRowsInSection:(NSInteger)section {
    return MIN(self.pendingTodos.count, 
               self.extensionContext.widgetActiveDisplayMode == NCWidgetDisplayModeCompact ? 2 : 8);
}

- (UITableViewCell *)tableView:(UITableView *)tableView cellForRowAtIndexPath:(NSIndexPath *)indexPath {
    UITableViewCell *cell = [tableView dequeueReusableCellWithIdentifier:@"TodoCell"];
    if (!cell) {
        cell = [[UITableViewCell alloc] initWithStyle:UITableViewCellStyleDefault 
                                      reuseIdentifier:@"TodoCell"];
        cell.backgroundColor = [UIColor clearColor];
    }
    
    NSDictionary *todo = self.pendingTodos[indexPath.row];
    cell.textLabel.text = [NSString stringWithFormat:@"• %@", todo[@"title"]];
    cell.textLabel.font = [UIFont preferredFontForTextStyle:UIFontTextStyleSubheadline];
    
    return cell;
}

- (void)tableView:(UITableView *)tableView didSelectRowAtIndexPath:(NSIndexPath *)indexPath {
    [tableView deselectRowAtIndexPath:indexPath animated:YES];
    
    // เปิดแอปไปยัง Todo นั้น
    NSDictionary *todo = self.pendingTodos[indexPath.row];
    NSString *todoID = todo[@"id"];
    
    NSURL *deepLinkURL = [NSURL URLWithString:[NSString stringWithFormat:@"todoapp://todo/%@", todoID]];
    [self.extensionContext openURL:deepLinkURL completionHandler:nil];
}

@end
```

## แบบฝึกหัดที่ 2: Image Share Extension

สร้าง Share Extension ที่รับรูปภาพและเพิ่ม Filter

```objc
// ImageShareViewController.m
@interface ImageShareViewController : UIViewController

@property (nonatomic, strong) UIImageView *originalImageView;
@property (nonatomic, strong) UIImageView *filteredImageView;
@property (nonatomic, strong) UIScrollView *filterScrollView;
@property (nonatomic, strong) UIImage *originalImage;

@end

@implementation ImageShareViewController

- (void)viewDidLoad {
    [super viewDidLoad];
    [self setupUI];
    [self loadSharedImage];
}

- (void)loadSharedImage {
    NSExtensionItem *item = self.extensionContext.inputItems.firstObject;
    
    for (NSItemProvider *provider in item.attachments) {
        if ([provider hasItemConformingToTypeIdentifier:(NSString *)kUTTypeImage]) {
            [provider loadItemForTypeIdentifier:(NSString *)kUTTypeImage 
                                        options:nil 
                              completionHandler:^(UIImage *image, NSError *error) {
                dispatch_async(dispatch_get_main_queue(), ^{
                    self.originalImage = image;
                    self.originalImageView.image = image;
                    self.filteredImageView.image = image;
                });
            }];
            break;
        }
    }
}

- (void)applyFilter:(CIFilter *)filter {
    CIImage *ciImage = [CIImage imageWithCGImage:self.originalImage.CGImage];
    [filter setValue:ciImage forKey:kCIInputImageKey];
    
    CIImage *outputImage = filter.outputImage;
    CIContext *context = [CIContext contextWithOptions:nil];
    CGImageRef cgImage = [context createCGImage:outputImage fromRect:outputImage.extent];
    
    UIImage *filteredImage = [UIImage imageWithCGImage:cgImage];
    CGImageRelease(cgImage);
    
    self.filteredImageView.image = filteredImage;
}

- (void)saveFilteredImage {
    UIImage *imageToSave = self.filteredImageView.image;
    
    // บันทึกไปยัง App Group
    AppGroupsManager *groups = [AppGroupsManager sharedManager];
    NSData *imageData = UIImageJPEGRepresentation(imageToSave, 0.9);
    NSString *filename = [NSString stringWithFormat:@"edited_%@.jpg", 
                          [[NSUUID UUID] UUIDString]];
    
    [groups saveData:imageData withFilename:filename];
    
    // อัปเดต Pending Items
    NSMutableArray *pending = [NSMutableArray arrayWithArray:
                               [[groups valueForKey:@"pending_images"] ?: @[]]];
    [pending addObject:@{@"filename": filename, @"date": [NSDate date]}];
    [groups setValue:[pending copy] forKey:@"pending_images"];
    
    // ปิด Extension
    [self.extensionContext completeRequestReturningItems:@[] completionHandler:nil];
}

- (void)setupUI {
    self.view.backgroundColor = [UIColor systemBackgroundColor];
    NSLog(@"Image Share UI setup complete");
    
    self.originalImageView = [[UIImageView alloc] init];
    self.filteredImageView = [[UIImageView alloc] init];
    self.filterScrollView = [[UIScrollView alloc] init];
    
    self.originalImageView.contentMode = UIViewContentModeScaleAspectFit;
    self.filteredImageView.contentMode = UIViewContentModeScaleAspectFit;
}

@end
```

## แบบฝึกหัดที่ 3: Secure Notification Service

สร้าง Notification Service Extension ที่:
- ถอดรหัส Encrypted Messages
- ดาวน์โหลด Attachments
- ป้องกัน Message Preview ใน Notification

```objc
// SecureNotificationService.m
@interface SecureNotificationService : UNNotificationServiceExtension
@property (nonatomic, strong) void(^contentHandler)(UNNotificationContent *);
@property (nonatomic, strong) UNMutableNotificationContent *bestAttemptContent;
@end

@implementation SecureNotificationService

- (void)didReceiveNotificationRequest:(UNNotificationRequest *)request 
                   withContentHandler:(void(^)(UNNotificationContent *))contentHandler {
    self.contentHandler = contentHandler;
    self.bestAttemptContent = [request.content mutableCopy];
    
    NSDictionary *userInfo = request.content.userInfo;
    
    // 1. Obfuscate ข้อมูลใน Notification ก่อน (ป้องกัน Preview Leak)
    self.bestAttemptContent.title = @"ข้อความใหม่";
    self.bestAttemptContent.subtitle = @"";
    self.bestAttemptContent.body = @"แตะเพื่ออ่าน";
    
    // 2. ถอดรหัสถ้ามี Encrypted Content
    NSString *encrypted = userInfo[@"enc"];
    if (encrypted) {
        NSString *decrypted = [self decryptPayload:encrypted];
        if (decrypted) {
            NSDictionary *message = [NSJSONSerialization JSONObjectWithData:
                                     [decrypted dataUsingEncoding:NSUTF8StringEncoding]
                                     options:0 error:nil];
            
            if (message) {
                self.bestAttemptContent.title = message[@"sender"] ?: @"ข้อความใหม่";
                self.bestAttemptContent.body = message[@"preview"] ?: @"แตะเพื่ออ่าน";
                
                // บันทึกข้อความเต็มใน App Group
                [self storeDecryptedMessage:message withID:userInfo[@"msg_id"]];
            }
        }
    }
    
    // 3. ดาวน์โหลด Attachment ถ้ามี
    NSString *attachmentURL = userInfo[@"media_url"];
    if (attachmentURL) {
        [self downloadAttachment:attachmentURL completion:^(UNNotificationAttachment *attachment) {
            if (attachment) {
                self.bestAttemptContent.attachments = @[attachment];
            }
            contentHandler(self.bestAttemptContent);
        }];
    } else {
        contentHandler(self.bestAttemptContent);
    }
}

- (NSString *)decryptPayload:(NSString *)encrypted {
    // ดึง Key จาก App Group Keychain
    NSUserDefaults *shared = [[NSUserDefaults alloc] 
                               initWithSuiteName:@"group.com.company.myapp"];
    NSData *keyData = [shared dataForKey:@"decryption_key"];
    
    if (!keyData) {
        NSLog(@"❌ ไม่พบ Decryption Key");
        return nil;
    }
    
    // TODO: ถอดรหัสด้วย CryptoManager
    return @"Decrypted placeholder";
}

- (void)storeDecryptedMessage:(NSDictionary *)message withID:(NSString *)messageID {
    NSUserDefaults *shared = [[NSUserDefaults alloc] 
                               initWithSuiteName:@"group.com.company.myapp"];
    
    NSMutableDictionary *messages = [[shared dictionaryForKey:@"cached_messages"] mutableCopy] 
                                    ?: [NSMutableDictionary dictionary];
    messages[messageID] = message;
    [shared setObject:[messages copy] forKey:@"cached_messages"];
    [shared synchronize];
}

- (void)downloadAttachment:(NSString *)urlString 
                completion:(void(^)(UNNotificationAttachment *))completion {
    
    NSURL *url = [NSURL URLWithString:urlString];
    NSURLSessionDownloadTask *task = [[NSURLSession sharedSession] 
        downloadTaskWithURL:url 
          completionHandler:^(NSURL *location, NSURLResponse *response, NSError *error) {
        
        if (error || !location) {
            completion(nil);
            return;
        }
        
        NSString *filename = [NSString stringWithFormat:@"%@.%@", 
                              [[NSUUID UUID] UUIDString],
                              url.pathExtension ?: @"jpg"];
        NSURL *destURL = [NSURL fileURLWithPath:[NSTemporaryDirectory() 
                                                 stringByAppendingPathComponent:filename]];
        
        [[NSFileManager defaultManager] moveItemAtURL:location toURL:destURL error:nil];
        
        NSError *attachError = nil;
        UNNotificationAttachment *attachment = [UNNotificationAttachment 
            attachmentWithIdentifier:@"media"
            URL:destURL
            options:nil
            error:&attachError];
        
        completion(attachment);
    }];
    
    [task resume];
}

- (void)serviceExtensionTimeWillExpire {
    self.contentHandler(self.bestAttemptContent);
}

@end
```

---

## สรุปบทที่ 80

| Extension Type | Use Case | Memory Limit |
|---------------|----------|-------------|
| Today Widget | แสดงข้อมูลสรุป | ~16 MB |
| Share Extension | รับเนื้อหาจากแอปอื่น | ~60 MB |
| Action Extension | แปลงเนื้อหา | ~60 MB |
| Custom Keyboard | Keyboard พิเศษ | ~50 MB |
| Notification Content | Custom Notification UI | ~24 MB |
| Notification Service | ปรับ Notification ก่อนแสดง | ~24 MB |

| การแชร์ข้อมูล | เหมาะกับ |
|--------------|---------|
| NSUserDefaults (App Group) | ข้อมูลขนาดเล็ก, Settings |
| Shared File Container | รูปภาพ, ไฟล์ขนาดใหญ่ |
| Darwin Notifications | Real-time notifications |
| URL Scheme | เปิด App จาก Extension |
| Core Data (Shared) | Database ที่ซับซ้อน |

> **คำแนะนำสำคัญ**:
> 1. Extension มี Memory Limit ต่ำ - หลีกเลี่ยง Image Loading ขนาดใหญ่
> 2. ใช้ App Groups สำหรับข้อมูลทั้งหมด - Extension และ App ต้องใช้ Shared Container
> 3. ทดสอบ Extension แยกจาก App - ตั้ง Scheme สำหรับ Extension ใน Xcode
> 4. Extension ถูก Terminate เมื่อ Memory เกิน - ต้องจัดการ Memory อย่างระมัดระวัง
> 5. Notification Service มีเวลาจำกัด - ต้องเรียก contentHandler ก่อน Expire

```objc
// Checklist สำหรับ App Extension:
// □ เพิ่ม App Groups Entitlement ทั้ง App และ Extension
// □ ใช้ kAppGroupIdentifier เดียวกันทั้งคู่
// □ ทดสอบ Memory Usage ด้วย Xcode Memory Gauge
// □ Handle Extension Termination อย่างถูกต้อง
// □ ตั้งค่า NSExtensionActivationRule ใน Info.plist อย่างเหมาะสม
// □ ทดสอบ Extension ในทุก iOS version ที่รองรับ
```
