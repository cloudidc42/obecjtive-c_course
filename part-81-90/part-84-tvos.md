# Part 84: tvOS Development (การพัฒนาแอปสำหรับ Apple TV)

## บทนำ

tvOS คือระบบปฏิบัติการของ Apple TV ซึ่งพัฒนาต่อจาก iOS โดยมีการปรับแต่งสำหรับการใช้งานบนหน้าจอโทรทัศน์และการควบคุมด้วย Siri Remote tvOS มีความแตกต่างจาก iOS ในด้าน Focus Engine, การควบคุม, และ UI patterns ที่เหมาะสมสำหรับการดูจากระยะไกล (10-foot UI)

---

## 1. tvOS App Structure (โครงสร้างแอป tvOS)

### 1.1 ความแตกต่างหลักจาก iOS

- ไม่มี touch screen - ใช้ Siri Remote แทน
- ใช้ Focus Engine แทน tap gestures
- ไม่มี Safari, App Store ที่ผู้ใช้เข้าถึงได้โดยตรง
- Storage จำกัด - ข้อมูลหนักควรใช้ iCloud หรือ network
- ไม่มี persistent local storage หลังลบแอป (On-demand resources)

### 1.2 โครงสร้างพื้นฐาน

```objc
// AppDelegate.h - เหมือน iOS แต่ไม่มี UIWindow ที่จำเป็น
#import <UIKit/UIKit.h>

@interface AppDelegate : UIResponder <UIApplicationDelegate>

@property (strong, nonatomic) UIWindow *window;

@end
```

```objc
// AppDelegate.m
#import "AppDelegate.h"

@implementation AppDelegate

- (BOOL)application:(UIApplication *)application 
    didFinishLaunchingWithOptions:(NSDictionary *)launchOptions {
    
    NSLog(@"tvOS app launched");
    
    // ตั้งค่าเหมือน iOS แต่ระวัง API ที่ไม่รองรับ
    [self setupAppearance];
    
    return YES;
}

- (void)setupAppearance {
    // tvOS navigation bar ปกติจะซ่อน
    // ใช้ custom UI แทน
    
    // ตั้งค่า focus guide appearance
    // Focus item จะมี highlight effect อัตโนมัติ
}

@end
```

---

## 2. Focus Engine (ระบบโฟกัส)

Focus Engine คือหัวใจของ tvOS UI - มันจัดการว่า element ไหนกำลัง "focused" อยู่

### 2.1 หลักการทำงานของ Focus Engine

```
ผู้ใช้กด Swipe บน Remote
        ↓
Focus Engine คำนวณทิศทาง
        ↓
ค้นหา focusable item ที่ใกล้ที่สุดในทิศทางนั้น
        ↓
ย้าย focus ไปยัง item นั้น
        ↓
แสดง focus highlight effect
```

### 2.2 Making Views Focusable

```objc
// ViewController.m
@interface MainViewController : UIViewController

@property (strong, nonatomic) UIButton *playButton;
@property (strong, nonatomic) UIButton *settingsButton;
@property (strong, nonatomic) UICollectionView *contentCollection;

@end

@implementation MainViewController

- (void)viewDidLoad {
    [super viewDidLoad];
    
    [self setupUI];
}

- (void)setupUI {
    self.view.backgroundColor = [UIColor blackColor];
    
    // สร้างปุ่ม - UIButton focusable โดย default บน tvOS
    self.playButton = [UIButton buttonWithType:UIButtonTypeSystem];
    self.playButton.frame = CGRectMake(100, 300, 200, 60);
    [self.playButton setTitle:@"▶ เล่น" forState:UIControlStateNormal];
    [self.playButton setTitleColor:[UIColor whiteColor] forState:UIControlStateNormal];
    self.playButton.titleLabel.font = [UIFont systemFontOfSize:36 weight:UIFontWeightMedium];
    self.playButton.backgroundColor = [UIColor systemBlueColor];
    self.playButton.layer.cornerRadius = 10;
    
    [self.playButton addTarget:self action:@selector(playButtonTapped) 
               forControlEvents:UIControlEventPrimaryActionTriggered];
    
    [self.view addSubview:self.playButton];
    
    // ตั้งค่า focus appearance
    [self setupFocusAppearance];
}

- (void)setupFocusAppearance {
    // กำหนด focus highlight ของปุ่ม
    // tvOS จะ scale up element ที่ focused โดยอัตโนมัติ
    // แต่เราสามารถ customize ได้
}

// Override เพื่อกำหนด focus behavior
- (NSArray<id<UIFocusEnvironment>> *)preferredFocusEnvironments {
    // กำหนด default focus item
    return @[self.playButton];
}

// ตรวจสอบว่าควร update focus หรือไม่
- (BOOL)shouldUpdateFocusInContext:(UIFocusUpdateContext *)context {
    return YES; // allow focus changes
}

// รับการแจ้งเตือนเมื่อ focus เปลี่ยน
- (void)didUpdateFocusInContext:(UIFocusUpdateContext *)context 
        withAnimationCoordinator:(UIFocusAnimationCoordinator *)coordinator {
    
    [super didUpdateFocusInContext:context withAnimationCoordinator:coordinator];
    
    UIView *nextFocused = (UIView *)context.nextFocusedItem;
    UIView *prevFocused = (UIView *)context.previouslyFocusedItem;
    
    NSLog(@"Focus changed from %@ to %@", prevFocused, nextFocused);
    
    [coordinator addCoordinatedAnimations:^{
        // Animate เมื่อ focus เปลี่ยน
        if (nextFocused == self.playButton) {
            self.playButton.transform = CGAffineTransformMakeScale(1.1, 1.1);
            self.playButton.backgroundColor = [UIColor systemOrangeColor];
        } else {
            self.playButton.transform = CGAffineTransformIdentity;
            self.playButton.backgroundColor = [UIColor systemBlueColor];
        }
        
        if (prevFocused == self.playButton) {
            prevFocused.transform = CGAffineTransformIdentity;
        }
    } completion:nil];
}

- (void)playButtonTapped {
    NSLog(@"Play button tapped via remote");
    // เริ่มเล่นวิดีโอ
}

@end
```

### 2.3 Custom Focusable View

```objc
// FocusableCardView.h
@interface FocusableCardView : UIView

@property (strong, nonatomic) UIImage *thumbnailImage;
@property (strong, nonatomic) NSString *title;

@end

// FocusableCardView.m
@interface FocusableCardView ()

@property (strong, nonatomic) UIImageView *imageView;
@property (strong, nonatomic) UILabel *titleLabel;
@property (strong, nonatomic) UIVisualEffectView *blurView;

@end

@implementation FocusableCardView

- (instancetype)initWithFrame:(CGRect)frame {
    self = [super initWithFrame:frame];
    if (self) {
        [self setupView];
    }
    return self;
}

- (void)setupView {
    self.layer.cornerRadius = 12;
    self.clipsToBounds = YES;
    
    // Image view
    self.imageView = [[UIImageView alloc] initWithFrame:self.bounds];
    self.imageView.contentMode = UIViewContentModeScaleAspectFill;
    self.imageView.autoresizingMask = UIViewAutoresizingFlexibleWidth | UIViewAutoresizingFlexibleHeight;
    [self addSubview:self.imageView];
    
    // Gradient overlay
    CAGradientLayer *gradient = [CAGradientLayer layer];
    gradient.frame = CGRectMake(0, self.bounds.size.height * 0.6, 
                                self.bounds.size.width, self.bounds.size.height * 0.4);
    gradient.colors = @[(__bridge id)[UIColor clearColor].CGColor, 
                        (__bridge id)[UIColor blackColor].CGColor];
    [self.layer addSublayer:gradient];
    
    // Title label
    self.titleLabel = [[UILabel alloc] initWithFrame:CGRectMake(16, 
                                                                self.bounds.size.height - 50, 
                                                                self.bounds.size.width - 32, 
                                                                40)];
    self.titleLabel.textColor = [UIColor whiteColor];
    self.titleLabel.font = [UIFont systemFontOfSize:24 weight:UIFontWeightSemibold];
    [self addSubview:self.titleLabel];
}

// ต้องมี canBecomeFocused = YES เพื่อให้ focus ได้
- (BOOL)canBecomeFocused {
    return YES;
}

// Animate เมื่อได้รับ/เสีย focus
- (void)didUpdateFocusInContext:(UIFocusUpdateContext *)context 
        withAnimationCoordinator:(UIFocusAnimationCoordinator *)coordinator {
    
    [super didUpdateFocusInContext:context withAnimationCoordinator:coordinator];
    
    BOOL focused = (context.nextFocusedView == self);
    
    [coordinator addCoordinatedAnimations:^{
        if (focused) {
            // scale up + shadow
            self.transform = CGAffineTransformMakeScale(1.15, 1.15);
            self.layer.shadowColor = [UIColor blackColor].CGColor;
            self.layer.shadowOffset = CGSizeMake(0, 20);
            self.layer.shadowOpacity = 0.5;
            self.layer.shadowRadius = 30;
            
        } else {
            // กลับสู่ normal
            self.transform = CGAffineTransformIdentity;
            self.layer.shadowOpacity = 0;
        }
    } completion:nil];
}

- (void)setThumbnailImage:(UIImage *)thumbnailImage {
    _thumbnailImage = thumbnailImage;
    self.imageView.image = thumbnailImage;
}

- (void)setTitle:(NSString *)title {
    _title = title;
    self.titleLabel.text = title;
}

@end
```

---

## 3. UIFocusEnvironment

UIFocusEnvironment protocol ให้ views/controllers จัดการ focus behavior

```objc
// FocusContainer.h - container ที่จัดการ focus ของ children
@interface FocusContainer : UIView <UIFocusEnvironment>

@end

// FocusContainer.m
@implementation FocusContainer

// กำหนด preferred focus environments
- (NSArray<id<UIFocusEnvironment>> *)preferredFocusEnvironments {
    // คืน array ของ focusable items ตามลำดับ priority
    NSMutableArray *environments = [NSMutableArray array];
    
    for (UIView *subview in self.subviews) {
        if ([subview conformsToProtocol:@protocol(UIFocusEnvironment)]) {
            [environments addObject:subview];
        }
    }
    
    return environments;
}

// ตรวจสอบ focus update
- (BOOL)shouldUpdateFocusInContext:(UIFocusUpdateContext *)context {
    return YES;
}

// Handler เมื่อ focus เปลี่ยน
- (void)didUpdateFocusInContext:(UIFocusUpdateContext *)context 
        withAnimationCoordinator:(UIFocusAnimationCoordinator *)coordinator {
    
    // จัดการ child focus changes
}

// บังคับ focus update
- (void)setNeedsFocusUpdate {
    [super setNeedsFocusUpdate];
}

- (void)updateFocusIfNeeded {
    [super updateFocusIfNeeded];
}

@end
```

### 3.1 UIFocusGuide

Focus Guide ช่วยกำหนด custom focus paths ระหว่าง UI elements

```objc
// ใช้ UIFocusGuide เชื่อม focus ระหว่าง elements ที่ไม่ adjacent
- (void)setupFocusGuides {
    // สร้าง focus guide ระหว่าง leftButton และ rightButton ที่อยู่ห่างกัน
    UIFocusGuide *focusGuide = [[UIFocusGuide alloc] init];
    [self.view addLayoutGuide:focusGuide];
    
    // กำหนด position ของ guide (พื้นที่ว่างระหว่างปุ่ม)
    [focusGuide.leadingAnchor constraintEqualToAnchor:self.leftButton.trailingAnchor].active = YES;
    [focusGuide.trailingAnchor constraintEqualToAnchor:self.rightButton.leadingAnchor].active = YES;
    [focusGuide.topAnchor constraintEqualToAnchor:self.leftButton.topAnchor].active = YES;
    [focusGuide.heightAnchor constraintEqualToAnchor:self.leftButton.heightAnchor].active = YES;
    
    // เมื่อ swipe right เข้า guide area ให้ไปที่ rightButton
    focusGuide.preferredFocusEnvironments = @[self.rightButton];
    
    NSLog(@"Focus guides set up");
}
```

---

## 4. Remote Control Handling (การจัดการ Remote)

### 4.1 Siri Remote (4th generation)

Siri Remote มี:
- Touch surface (Clickpad) - swipe และ click
- Directional buttons
- Menu button
- Home/TV button
- Siri/Search button
- Play/Pause button
- Volume buttons

### 4.2 Basic Gesture Recognition

```objc
// ViewController.m
- (void)viewDidLoad {
    [super viewDidLoad];
    
    [self setupGestureRecognizers];
}

- (void)setupGestureRecognizers {
    // Swipe gestures
    UISwipeGestureRecognizer *swipeRight = 
        [[UISwipeGestureRecognizer alloc] initWithTarget:self 
                                                   action:@selector(handleSwipeRight:)];
    swipeRight.direction = UISwipeGestureRecognizerDirectionRight;
    [self.view addGestureRecognizer:swipeRight];
    
    UISwipeGestureRecognizer *swipeLeft = 
        [[UISwipeGestureRecognizer alloc] initWithTarget:self 
                                                   action:@selector(handleSwipeLeft:)];
    swipeLeft.direction = UISwipeGestureRecognizerDirectionLeft;
    [self.view addGestureRecognizer:swipeLeft];
    
    UISwipeGestureRecognizer *swipeUp = 
        [[UISwipeGestureRecognizer alloc] initWithTarget:self 
                                                   action:@selector(handleSwipeUp:)];
    swipeUp.direction = UISwipeGestureRecognizerDirectionUp;
    [self.view addGestureRecognizer:swipeUp];
    
    UISwipeGestureRecognizer *swipeDown = 
        [[UISwipeGestureRecognizer alloc] initWithTarget:self 
                                                   action:@selector(handleSwipeDown:)];
    swipeDown.direction = UISwipeGestureRecognizerDirectionDown;
    [self.view addGestureRecognizer:swipeDown];
    
    // Tap gesture (click)
    UITapGestureRecognizer *tap = 
        [[UITapGestureRecognizer alloc] initWithTarget:self 
                                                action:@selector(handleTap:)];
    [self.view addGestureRecognizer:tap];
    
    // Long press
    UILongPressGestureRecognizer *longPress = 
        [[UILongPressGestureRecognizer alloc] initWithTarget:self 
                                                       action:@selector(handleLongPress:)];
    longPress.minimumPressDuration = 0.5;
    [self.view addGestureRecognizer:longPress];
    
    // Pan gesture (สำหรับ touch surface)
    UIPanGestureRecognizer *pan = 
        [[UIPanGestureRecognizer alloc] initWithTarget:self 
                                                action:@selector(handlePan:)];
    [self.view addGestureRecognizer:pan];
}

- (void)handleSwipeRight:(UISwipeGestureRecognizer *)gesture {
    NSLog(@"Swiped right");
    [self navigateNext];
}

- (void)handleSwipeLeft:(UISwipeGestureRecognizer *)gesture {
    NSLog(@"Swiped left");
    [self navigatePrevious];
}

- (void)handleSwipeUp:(UISwipeGestureRecognizer *)gesture {
    NSLog(@"Swiped up");
    [self showContextMenu];
}

- (void)handleSwipeDown:(UISwipeGestureRecognizer *)gesture {
    NSLog(@"Swiped down");
    [self dismissContextMenu];
}

- (void)handleTap:(UITapGestureRecognizer *)gesture {
    NSLog(@"Remote clicked");
    UIView *focused = [self.view.window.windowScene.focusSystem.focusedItem isKindOfClass:[UIView class]] 
                       ? (UIView *)self.view.window.windowScene.focusSystem.focusedItem : nil;
    NSLog(@"Focused view: %@", focused);
}

- (void)handleLongPress:(UILongPressGestureRecognizer *)gesture {
    if (gesture.state == UIGestureRecognizerStateBegan) {
        NSLog(@"Long press began");
        [self showContextMenu];
    }
}

- (void)handlePan:(UIPanGestureRecognizer *)gesture {
    CGPoint velocity = [gesture velocityInView:self.view];
    NSLog(@"Pan velocity: %.0f, %.0f", velocity.x, velocity.y);
}
```

### 4.3 Remote Buttons (Press Events)

```objc
// รับ button press events ผ่าน UIResponder
- (void)pressesBegan:(NSSet<UIPress *> *)presses 
           withEvent:(UIPressesEvent *)event {
    
    for (UIPress *press in presses) {
        switch (press.type) {
            case UIPressTypeMenu:
                NSLog(@"Menu button pressed");
                [self handleMenuPress];
                break;
                
            case UIPressTypePlayPause:
                NSLog(@"Play/Pause pressed");
                [self togglePlayPause];
                break;
                
            case UIPressTypeSelect:
                NSLog(@"Select (click) pressed");
                break;
                
            case UIPressTypeUpArrow:
                NSLog(@"Up arrow pressed");
                break;
                
            case UIPressTypeDownArrow:
                NSLog(@"Down arrow pressed");
                break;
                
            case UIPressTypeLeftArrow:
                NSLog(@"Left arrow pressed");
                break;
                
            case UIPressTypeRightArrow:
                NSLog(@"Right arrow pressed");
                break;
                
            default:
                [super pressesBegan:presses withEvent:event];
                break;
        }
    }
}

- (void)pressesEnded:(NSSet<UIPress *> *)presses 
           withEvent:(UIPressesEvent *)event {
    // Handle press end
    [super pressesEnded:presses withEvent:event];
}

- (void)handleMenuPress {
    // Menu = back navigation
    if (self.navigationController.viewControllers.count > 1) {
        [self.navigationController popViewControllerAnimated:YES];
    } else {
        NSLog(@"Already at root");
    }
}

- (void)togglePlayPause {
    if (self.player.rate > 0) {
        [self.player pause];
    } else {
        [self.player play];
    }
}
```

---

## 5. TVMLKit (JavaScript-based tvOS UI)

TVMLKit ช่วยให้สร้าง UI ด้วย XML templates สำหรับ content-heavy apps

### 5.1 Basic TVMLKit Setup

```objc
// AppDelegate.m
#import <TVMLKit/TVMLKit.h>

@interface AppDelegate () <TVApplicationControllerDelegate>

@property (strong, nonatomic) TVApplicationController *appController;

@end

@implementation AppDelegate

- (BOOL)application:(UIApplication *)application 
    didFinishLaunchingWithOptions:(NSDictionary *)launchOptions {
    
    // URL ที่มี JavaScript app
    NSURL *jsURL = [NSURL URLWithString:@"https://example.com/tvml/application.js"];
    
    TVApplicationControllerContext *context = [[TVApplicationControllerContext alloc] init];
    context.javaScriptApplicationURL = jsURL;
    
    self.appController = [[TVApplicationController alloc] initWithContext:context
                                                                   window:self.window
                                                                 delegate:self];
    return YES;
}

#pragma mark - TVApplicationControllerDelegate

- (void)appController:(TVApplicationController *)appController 
    evaluateAppJavaScriptInContext:(JSContext *)jsContext {
    
    // เพิ่ม native functions ที่ JavaScript เรียกได้
    jsContext[@"nativeFunction"] = ^(NSString *message) {
        NSLog(@"JS called native: %@", message);
    };
    
    // ส่ง data ไปยัง JavaScript
    jsContext[@"nativeData"] = @{
        @"userId": @"12345",
        @"userName": @"User",
        @"preferences": @{@"language": @"th"}
    };
}

- (void)appController:(TVApplicationController *)appController 
    didFinishLaunchingWithOptions:(NSDictionary *)options {
    NSLog(@"TVMLKit app launched");
}

- (void)appController:(TVApplicationController *)appController 
    didFailWithError:(NSError *)error {
    NSLog(@"TVMLKit error: %@", error.localizedDescription);
}

@end
```

### 5.2 TVML Document Template

```xml
<!-- application.js จะโหลด TVML document นี้ -->
<?xml version="1.0" encoding="UTF-8" ?>
<document>
    <catalogTemplate>
        <banner>
            <background>
                <img src="https://example.com/banner.jpg" 
                     width="1920" height="600" />
            </background>
            <title>My TV App</title>
        </banner>
        
        <list>
            <header>
                <title>แนะนำสำหรับคุณ</title>
            </header>
            
            <section>
                <listItemLockup>
                    <ordinal minLength="1" maxLength="3">1</ordinal>
                    <title>รายการที่ 1</title>
                    <decorationLabel>ใหม่</decorationLabel>
                </listItemLockup>
                
                <listItemLockup>
                    <ordinal minLength="1" maxLength="3">2</ordinal>
                    <title>รายการที่ 2</title>
                </listItemLockup>
            </section>
        </list>
    </catalogTemplate>
</document>
```

---

## 6. Top Shelf Extension (ส่วนขยาย Top Shelf)

Top Shelf แสดงเนื้อหาพิเศษเมื่อผู้ใช้ focus บน app icon ในหน้าแรกของ Apple TV

### 6.1 Static Top Shelf

```objc
// TopShelfProvider.h
#import <TVServices/TVServices.h>

@interface TopShelfProvider : NSObject <TVTopShelfProvider>

@end

// TopShelfProvider.m
#import "TopShelfProvider.h"

@implementation TopShelfProvider

- (TVTopShelfContentStyle)topShelfStyle {
    return TVTopShelfContentStyleSectioned; // หรือ TVTopShelfContentStyleInset
}

- (NSArray<TVTopShelfItem *> *)topShelfItems {
    NSMutableArray *items = [NSMutableArray array];
    
    // สร้าง items สำหรับ Top Shelf
    NSArray *featuredContent = @[
        @{@"id": @"1", @"title": @"ภาพยนตร์ใหม่", @"image": @"movie1.jpg"},
        @{@"id": @"2", @"title": @"ซีรีส์ยอดนิยม", @"image": @"series1.jpg"},
        @{@"id": @"3", @"title": @"สารคดีน่าดู", @"image": @"doc1.jpg"},
    ];
    
    for (NSDictionary *content in featuredContent) {
        TVTopShelfItem *item = [[TVTopShelfItem alloc] initWithIdentifier:content[@"id"]];
        item.title = content[@"title"];
        
        // โหลดรูปภาพ
        NSURL *imageURL = [[NSBundle mainBundle] URLForResource:content[@"image"] 
                                                  withExtension:nil];
        if (imageURL) {
            item.imageURL = imageURL;
        }
        
        item.imageShape = TVTopShelfItemImageShapeHDTV; // 16:9
        
        // กำหนด URL สำหรับ deep link
        item.playURL = [NSURL URLWithString:[NSString stringWithFormat:@"myapp://play/%@", content[@"id"]]];
        item.displayURL = [NSURL URLWithString:[NSString stringWithFormat:@"myapp://show/%@", content[@"id"]]];
        
        [items addObject:item];
    }
    
    return items;
}

@end
```

### 6.2 Sectioned Top Shelf

```objc
- (NSArray<TVTopShelfItem *> *)topShelfItems {
    NSMutableArray *allItems = [NSMutableArray array];
    
    // Section 1: Continue Watching
    TVTopShelfSectionedItem *continueSection = 
        [[TVTopShelfSectionedItem alloc] initWithIdentifier:@"continueWatching"];
    continueSection.title = @"ดูต่อ";
    
    NSMutableArray *continueItems = [NSMutableArray array];
    // เพิ่ม items...
    continueSection.items = continueItems;
    [allItems addObject:continueSection];
    
    // Section 2: New Releases
    TVTopShelfSectionedItem *newSection = 
        [[TVTopShelfSectionedItem alloc] initWithIdentifier:@"newReleases"];
    newSection.title = @"ใหม่ล่าสุด";
    
    NSMutableArray *newItems = [NSMutableArray array];
    // เพิ่ม items...
    newSection.items = newItems;
    [allItems addObject:newSection];
    
    return allItems;
}
```

### 6.3 รับ Deep Link จาก Top Shelf

```objc
// AppDelegate.m
- (BOOL)application:(UIApplication *)application 
    openURL:(NSURL *)url 
    options:(NSDictionary *)options {
    
    NSLog(@"Opened from Top Shelf: %@", url);
    
    // Parse URL
    // myapp://play/123 หรือ myapp://show/123
    NSString *host = url.host; // "play" หรือ "show"
    NSString *path = url.path; // "/123"
    NSString *itemId = [path substringFromIndex:1];
    
    if ([host isEqualToString:@"play"]) {
        [self playContentWithId:itemId];
    } else if ([host isEqualToString:@"show"]) {
        [self showContentWithId:itemId];
    }
    
    return YES;
}
```

---

## 7. TVServices Framework

TVServices รองรับ Top Shelf และ features อื่นๆ ของ tvOS

```objc
// Info.plist ต้องมี
// NSExtension (Dictionary)
//   NSExtensionPointIdentifier: com.apple.tv-top-shelf
//   NSExtensionPrincipalClass: TopShelfProvider

// Refresh Top Shelf ด้วย
#import <TVServices/TVServices.h>

// Notify system ว่า Top Shelf content เปลี่ยนแล้ว
[TVTopShelfContentProvider scheduleTopShelfContentRefresh];
```

---

## 8. Game Controller Support (รองรับ Game Controller)

```objc
// GameControllerManager.h
#import <GameController/GameController.h>

@interface GameControllerManager : NSObject

+ (instancetype)shared;
- (void)startListening;

@property (copy, nonatomic) void (^controllerConnected)(GCController *controller);
@property (copy, nonatomic) void (^controllerDisconnected)(GCController *controller);

@end
```

```objc
// GameControllerManager.m
@interface GameControllerManager ()

@property (strong, nonatomic) NSMutableArray *connectedControllers;

@end

@implementation GameControllerManager

+ (instancetype)shared {
    static GameControllerManager *instance = nil;
    static dispatch_once_t onceToken;
    dispatch_once(&onceToken, ^{
        instance = [[self alloc] init];
    });
    return instance;
}

- (instancetype)init {
    self = [super init];
    if (self) {
        _connectedControllers = [NSMutableArray array];
    }
    return self;
}

- (void)startListening {
    // ฟัง notification เมื่อ controller เชื่อมต่อ/ตัดการเชื่อมต่อ
    [[NSNotificationCenter defaultCenter] 
     addObserver:self
        selector:@selector(controllerConnected:)
            name:GCControllerDidConnectNotification
          object:nil];
    
    [[NSNotificationCenter defaultCenter] 
     addObserver:self
        selector:@selector(controllerDisconnected:)
            name:GCControllerDidDisconnectNotification
          object:nil];
    
    // ตรวจสอบ controllers ที่เชื่อมต่ออยู่แล้ว
    NSArray *existing = [GCController controllers];
    for (GCController *controller in existing) {
        [self setupController:controller];
    }
    
    NSLog(@"Controller listening started. Connected: %lu", (unsigned long)existing.count);
}

- (void)controllerConnected:(NSNotification *)notification {
    GCController *controller = notification.object;
    NSLog(@"Controller connected: %@", controller.vendorName);
    
    [self.connectedControllers addObject:controller];
    [self setupController:controller];
    
    if (self.controllerConnected) {
        self.controllerConnected(controller);
    }
}

- (void)controllerDisconnected:(NSNotification *)notification {
    GCController *controller = notification.object;
    NSLog(@"Controller disconnected: %@", controller.vendorName);
    
    [self.connectedControllers removeObject:controller];
    
    if (self.controllerDisconnected) {
        self.controllerDisconnected(controller);
    }
}

- (void)setupController:(GCController *)controller {
    // ตั้งค่า input handlers
    
    // Extended Gamepad (Xbox/PlayStation controller)
    GCExtendedGamepad *extGamepad = controller.extendedGamepad;
    if (extGamepad) {
        [self setupExtendedGamepad:extGamepad];
        return;
    }
    
    // Micro Gamepad (Siri Remote เป็น gamepad)
    GCMicroGamepad *microGamepad = controller.microGamepad;
    if (microGamepad) {
        [self setupMicroGamepad:microGamepad];
        return;
    }
}

- (void)setupExtendedGamepad:(GCExtendedGamepad *)gamepad {
    // Button A (select)
    gamepad.buttonA.pressedChangedHandler = ^(GCControllerButtonInput *button, 
                                               float value, BOOL pressed) {
        if (pressed) {
            NSLog(@"Button A pressed");
            [[NSNotificationCenter defaultCenter] 
             postNotificationName:@"ControllerButtonA" object:nil];
        }
    };
    
    // Button B (back)
    gamepad.buttonB.pressedChangedHandler = ^(GCControllerButtonInput *button, 
                                               float value, BOOL pressed) {
        if (pressed) {
            NSLog(@"Button B pressed (back)");
        }
    };
    
    // D-Pad
    gamepad.dpad.valueChangedHandler = ^(GCControllerDirectionPad *dpad, 
                                          float xValue, float yValue) {
        NSLog(@"D-pad: x=%.2f, y=%.2f", xValue, yValue);
    };
    
    // Left Stick
    gamepad.leftThumbstick.valueChangedHandler = ^(GCControllerDirectionPad *stick, 
                                                    float xValue, float yValue) {
        NSLog(@"Left stick: x=%.2f, y=%.2f", xValue, yValue);
    };
    
    // Right Stick
    gamepad.rightThumbstick.valueChangedHandler = ^(GCControllerDirectionPad *stick, 
                                                     float xValue, float yValue) {
        NSLog(@"Right stick: x=%.2f, y=%.2f", xValue, yValue);
    };
    
    // Triggers
    gamepad.leftTrigger.valueChangedHandler = ^(GCControllerButtonInput *trigger, 
                                                 float value, BOOL pressed) {
        NSLog(@"Left trigger: %.2f", value);
    };
    
    gamepad.rightTrigger.valueChangedHandler = ^(GCControllerButtonInput *trigger, 
                                                  float value, BOOL pressed) {
        NSLog(@"Right trigger: %.2f", value);
    };
    
    NSLog(@"Extended gamepad set up: %@", gamepad.controller.vendorName);
}

- (void)setupMicroGamepad:(GCMicroGamepad *)gamepad {
    // สำหรับ Siri Remote ที่ใช้เป็น gamepad
    gamepad.buttonA.pressedChangedHandler = ^(GCControllerButtonInput *button, 
                                               float value, BOOL pressed) {
        if (pressed) {
            NSLog(@"Micro gamepad A pressed");
        }
    };
    
    gamepad.dpad.valueChangedHandler = ^(GCControllerDirectionPad *dpad, 
                                          float xValue, float yValue) {
        NSLog(@"Micro gamepad dpad: x=%.2f, y=%.2f", xValue, yValue);
    };
}

@end
```

### 8.1 ใช้ Game Controller ในเกม

```objc
// GameViewController.m
@interface GameViewController ()

@property (strong, nonatomic) SKScene *gameScene;
@property (strong, nonatomic) GCController *primaryController;

@end

@implementation GameViewController

- (void)viewDidLoad {
    [super viewDidLoad];
    
    [self setupGameController];
}

- (void)setupGameController {
    GameControllerManager *manager = [GameControllerManager shared];
    
    manager.controllerConnected = ^(GCController *controller) {
        if (!self.primaryController) {
            self.primaryController = controller;
            [self configurePlayerWithController:controller];
        }
    };
    
    manager.controllerDisconnected = ^(GCController *controller) {
        if (self.primaryController == controller) {
            self.primaryController = nil;
            // หยุดเกมหรือแสดง "กรุณาเชื่อมต่อ controller"
            [self pauseGame];
        }
    };
    
    [manager startListening];
}

- (void)configurePlayerWithController:(GCController *)controller {
    GCExtendedGamepad *gamepad = controller.extendedGamepad;
    if (!gamepad) return;
    
    // Poll input ทุก frame ใน game loop
    // หรือใช้ valueChangedHandler
    
    gamepad.leftThumbstick.valueChangedHandler = ^(GCControllerDirectionPad *stick, 
                                                    float xValue, float yValue) {
        // เคลื่อนที่ตัวละคร
        [self movePlayerX:xValue y:yValue];
    };
    
    gamepad.buttonA.pressedChangedHandler = ^(GCControllerButtonInput *button, 
                                               float value, BOOL pressed) {
        if (pressed) {
            [self playerJump];
        }
    };
    
    gamepad.buttonB.pressedChangedHandler = ^(GCControllerButtonInput *button, 
                                               float value, BOOL pressed) {
        if (pressed) {
            [self playerAttack];
        }
    };
    
    gamepad.rightTrigger.valueChangedHandler = ^(GCControllerButtonInput *trigger, 
                                                  float value, BOOL pressed) {
        [self updatePlayerSpeed:value]; // 0.0 - 1.0
    };
}

- (void)movePlayerX:(float)x y:(float)y {
    // อัพเดทตำแหน่งตัวละคร
}

- (void)playerJump {
    NSLog(@"Player jump");
}

- (void)playerAttack {
    NSLog(@"Player attack");
}

- (void)updatePlayerSpeed:(float)speed {
    NSLog(@"Player speed: %.2f", speed);
}

- (void)pauseGame {
    NSLog(@"Game paused");
}

@end
```

---

## 9. AVKit สำหรับ tvOS

AVKit บน tvOS มี features พิเศษสำหรับ TV experience

### 9.1 Basic Video Playback

```objc
// VideoPlayerViewController.h
#import <AVKit/AVKit.h>

@interface VideoPlayerViewController : AVPlayerViewController

@end
```

```objc
// VideoPlayerViewController.m
#import "VideoPlayerViewController.h"

@interface VideoPlayerViewController ()

@property (strong, nonatomic) AVPlayer *videoPlayer;

@end

@implementation VideoPlayerViewController

- (void)viewDidLoad {
    [super viewDidLoad];
    
    // ตั้งค่า player
    NSURL *videoURL = [NSURL URLWithString:@"https://example.com/video.m3u8"];
    self.videoPlayer = [AVPlayer playerWithURL:videoURL];
    self.player = self.videoPlayer;
    
    // ตั้งค่า info panel (แสดงระหว่างเล่น)
    [self setupInfoPanel];
    
    // ตั้งค่า skip intros/credits
    [self setupSkipBehavior];
}

- (void)viewDidAppear:(BOOL)animated {
    [super viewDidAppear:animated];
    [self.videoPlayer play];
}

- (void)setupInfoPanel {
    // กำหนดข้อมูลที่แสดงใน info panel
    AVPlayerItem *item = self.videoPlayer.currentItem;
    if (!item) return;
    
    // External metadata
    NSMutableArray *metadata = [NSMutableArray array];
    
    // Title
    AVMutableMetadataItem *titleItem = [[AVMutableMetadataItem alloc] init];
    titleItem.identifier = AVMetadataCommonIdentifierTitle;
    titleItem.value = @"ชื่อภาพยนตร์";
    titleItem.extendedLanguageTag = @"und";
    [metadata addObject:titleItem];
    
    // Description
    AVMutableMetadataItem *descItem = [[AVMutableMetadataItem alloc] init];
    descItem.identifier = AVMetadataCommonIdentifierDescription;
    descItem.value = @"คำอธิบายของภาพยนตร์หรือซีรีส์นี้";
    descItem.extendedLanguageTag = @"und";
    [metadata addObject:descItem];
    
    // Artwork
    AVMutableMetadataItem *artworkItem = [[AVMutableMetadataItem alloc] init];
    artworkItem.identifier = AVMetadataCommonIdentifierArtwork;
    artworkItem.value = UIImagePNGRepresentation([UIImage imageNamed:@"thumbnail"]);
    artworkItem.dataType = (__bridge NSString *)kCMMetadataBaseDataType_RawData;
    [metadata addObject:artworkItem];
    
    item.externalMetadata = metadata;
}

- (void)setupSkipBehavior {
    // tvOS 15+ Skip Intro/Credits
    if (@available(tvOS 15.0, *)) {
        AVPlayerItem *item = self.videoPlayer.currentItem;
        
        // กำหนดช่วง intro ที่จะ skip ได้
        CMTimeRange introRange = CMTimeRangeMake(
            CMTimeMake(0, 1), 
            CMTimeMake(90, 1)  // 90 วินาทีแรก
        );
        
        // สร้าง navigation marker group
        AVNavigationMarkersGroup *markerGroup = 
            [[AVNavigationMarkersGroup alloc] initWithTitle:@"Chapters"
                                           timedNavigationMarkers:@[]];
        
        // เพิ่ม intro skip
        // item.interstitialTimeRanges = ...; // ขึ้นกับ API ที่ใช้งาน
    }
}

// Custom overlay controls
- (void)setupCustomControls {
    // เพิ่ม custom overlay บน player
    UIView *overlayView = [[UIView alloc] initWithFrame:self.view.bounds];
    overlayView.backgroundColor = [UIColor clearColor];
    
    // Skip button
    UIButton *skipButton = [UIButton buttonWithType:UIButtonTypeSystem];
    skipButton.frame = CGRectMake(self.view.bounds.size.width - 200, 100, 180, 50);
    [skipButton setTitle:@"ข้าม intro" forState:UIControlStateNormal];
    skipButton.titleLabel.font = [UIFont systemFontOfSize:28];
    skipButton.backgroundColor = [[UIColor blackColor] colorWithAlphaComponent:0.7];
    skipButton.layer.cornerRadius = 8;
    [skipButton setTitleColor:[UIColor whiteColor] forState:UIControlStateNormal];
    
    [skipButton addTarget:self 
                   action:@selector(skipIntro) 
         forControlEvents:UIControlEventPrimaryActionTriggered];
    
    [overlayView addSubview:skipButton];
    
    // เพิ่ม overlay ผ่าน contentOverlayView
    [self.contentOverlayView addSubview:overlayView];
}

- (void)skipIntro {
    // ข้ามไปที่นาทีที่ 2 (ตัวอย่าง)
    CMTime skipTime = CMTimeMake(120, 1); // 2 นาที
    [self.videoPlayer seekToTime:skipTime];
}

@end
```

### 9.2 Custom Transport Controls

```objc
// Custom player สำหรับ UI ที่ต้องการความยืดหยุ่น
@interface CustomPlayerViewController : UIViewController

@end

@implementation CustomPlayerViewController

- (void)viewDidLoad {
    [super viewDidLoad];
    
    NSURL *videoURL = [NSURL URLWithString:@"https://example.com/video.mp4"];
    AVPlayer *player = [AVPlayer playerWithURL:videoURL];
    
    AVPlayerLayer *playerLayer = [AVPlayerLayer playerLayerWithPlayer:player];
    playerLayer.frame = self.view.bounds;
    playerLayer.videoGravity = AVLayerVideoGravityResizeAspect;
    [self.view.layer addSublayer:playerLayer];
    
    [player play];
    
    // สร้าง custom controls
    [self setupCustomTransportControls:player];
}

- (void)setupCustomTransportControls:(AVPlayer *)player {
    // Progress bar
    UISlider *progressBar = [[UISlider alloc] initWithFrame:CGRectMake(100, 
                                                                        self.view.bounds.size.height - 100, 
                                                                        self.view.bounds.size.width - 200, 
                                                                        40)];
    progressBar.minimumTrackTintColor = [UIColor whiteColor];
    progressBar.maximumTrackTintColor = [[UIColor whiteColor] colorWithAlphaComponent:0.3];
    [self.view addSubview:progressBar];
    
    // อัพเดท progress
    CMTime interval = CMTimeMake(1, 10); // 0.1 วินาที
    [player addPeriodicTimeObserverForInterval:interval
                                         queue:nil
                                    usingBlock:^(CMTime time) {
        float duration = CMTimeGetSeconds(player.currentItem.duration);
        float current = CMTimeGetSeconds(time);
        
        dispatch_async(dispatch_get_main_queue(), ^{
            progressBar.value = duration > 0 ? current / duration : 0;
        });
    }];
    
    // Seek เมื่อ slider เปลี่ยน
    [progressBar addTarget:self 
                    action:@selector(sliderDidChange:) 
          forControlEvents:UIControlEventValueChanged];
}

- (void)sliderDidChange:(UISlider *)slider {
    AVPlayer *player = /* reference to player */nil;
    float duration = CMTimeGetSeconds(player.currentItem.duration);
    float seekTime = duration * slider.value;
    
    [player seekToTime:CMTimeMakeWithSeconds(seekTime, NSEC_PER_SEC)
         toleranceBefore:kCMTimeZero
          toleranceAfter:kCMTimeZero];
}

@end
```

---

## 10. แบบฝึกหัด (Practice Exercises)

### Exercise 1: Movie Catalog App

```objc
// MovieCatalogViewController.m
@interface MovieCatalogViewController : UIViewController

@end

@implementation MovieCatalogViewController {
    UICollectionView *_collectionView;
    NSArray *_movies;
}

- (void)viewDidLoad {
    [super viewDidLoad];
    
    _movies = @[
        @{@"id": @"1", @"title": @"ภาพยนตร์ไทย 1", @"year": @"2024", @"rating": @"8.5"},
        @{@"id": @"2", @"title": @"แอ็คชั่น 1", @"year": @"2024", @"rating": @"7.8"},
        @{@"id": @"3", @"title": @"ตลก 1", @"year": @"2023", @"rating": @"8.0"},
        @{@"id": @"4", @"title": @"ดราม่า 1", @"year": @"2024", @"rating": @"9.1"},
    ];
    
    [self setupCollectionView];
}

- (void)setupCollectionView {
    // tvOS layout - ใหญ่กว่า iOS
    UICollectionViewFlowLayout *layout = [[UICollectionViewFlowLayout alloc] init];
    layout.itemSize = CGSizeMake(380, 220);
    layout.minimumInteritemSpacing = 30;
    layout.minimumLineSpacing = 50;
    layout.sectionInset = UIEdgeInsetsMake(60, 90, 60, 90);
    layout.scrollDirection = UICollectionViewScrollDirectionHorizontal;
    
    _collectionView = [[UICollectionView alloc] initWithFrame:self.view.bounds 
                                         collectionViewLayout:layout];
    _collectionView.backgroundColor = [UIColor clearColor];
    _collectionView.dataSource = self;
    _collectionView.delegate = self;
    
    [_collectionView registerClass:[MovieCell class] forCellWithReuseIdentifier:@"MovieCell"];
    
    [self.view addSubview:_collectionView];
}

- (NSInteger)collectionView:(UICollectionView *)collectionView 
     numberOfItemsInSection:(NSInteger)section {
    return _movies.count;
}

- (UICollectionViewCell *)collectionView:(UICollectionView *)collectionView 
                  cellForItemAtIndexPath:(NSIndexPath *)indexPath {
    MovieCell *cell = [collectionView dequeueReusableCellWithReuseIdentifier:@"MovieCell" 
                                                                  forIndexPath:indexPath];
    [cell configureWithMovie:_movies[indexPath.item]];
    return cell;
}

- (void)collectionView:(UICollectionView *)collectionView 
    didSelectItemAtIndexPath:(NSIndexPath *)indexPath {
    NSDictionary *movie = _movies[indexPath.item];
    NSLog(@"Selected: %@", movie[@"title"]);
    
    // เล่นวิดีโอ
    NSURL *videoURL = [NSURL URLWithString:[NSString stringWithFormat:@"https://example.com/movie/%@.m3u8", movie[@"id"]]];
    
    VideoPlayerViewController *playerVC = [[VideoPlayerViewController alloc] init];
    playerVC.videoURL = videoURL;
    playerVC.movieInfo = movie;
    
    [self presentViewController:playerVC animated:YES completion:nil];
}

@end
```

### Exercise 2: Music Player for tvOS

```objc
// MusicPlayerViewController.m
@interface MusicPlayerViewController : UIViewController

@end

@implementation MusicPlayerViewController

- (void)viewDidLoad {
    [super viewDidLoad];
    self.view.backgroundColor = [UIColor blackColor];
    
    [self setupUI];
    [self setupMusicPlayer];
    [self setupGameControllerSupport];
}

- (void)setupUI {
    // Large album art
    UIImageView *albumArt = [[UIImageView alloc] initWithFrame:CGRectMake(200, 100, 600, 600)];
    albumArt.image = [UIImage imageNamed:@"album"];
    albumArt.contentMode = UIViewContentModeScaleAspectFit;
    albumArt.layer.cornerRadius = 20;
    albumArt.clipsToBounds = YES;
    [self.view addSubview:albumArt];
    
    // Song info
    UILabel *titleLabel = [[UILabel alloc] initWithFrame:CGRectMake(900, 200, 900, 80)];
    titleLabel.text = @"ชื่อเพลง";
    titleLabel.font = [UIFont systemFontOfSize:60 weight:UIFontWeightBold];
    titleLabel.textColor = [UIColor whiteColor];
    [self.view addSubview:titleLabel];
    
    UILabel *artistLabel = [[UILabel alloc] initWithFrame:CGRectMake(900, 300, 900, 60)];
    artistLabel.text = @"ชื่อศิลปิน";
    artistLabel.font = [UIFont systemFontOfSize:44 weight:UIFontWeightRegular];
    artistLabel.textColor = [[UIColor whiteColor] colorWithAlphaComponent:0.7];
    [self.view addSubview:artistLabel];
    
    // Progress bar
    UIProgressView *progressView = [[UIProgressView alloc] initWithProgressViewStyle:UIProgressViewStyleDefault];
    progressView.frame = CGRectMake(900, 500, 900, 8);
    progressView.progressTintColor = [UIColor whiteColor];
    progressView.trackTintColor = [[UIColor whiteColor] colorWithAlphaComponent:0.3];
    [self.view addSubview:progressView];
    
    // Control buttons - ต้อง focusable
    NSArray *buttonTitles = @[@"⏮", @"⏪", @"⏯", @"⏩", @"⏭"];
    for (NSInteger i = 0; i < buttonTitles.count; i++) {
        UIButton *btn = [UIButton buttonWithType:UIButtonTypeSystem];
        btn.frame = CGRectMake(900 + i * 180, 600, 160, 80);
        [btn setTitle:buttonTitles[i] forState:UIControlStateNormal];
        btn.titleLabel.font = [UIFont systemFontOfSize:48];
        [btn setTitleColor:[UIColor whiteColor] forState:UIControlStateNormal];
        btn.tag = i;
        
        [btn addTarget:self action:@selector(controlButtonTapped:) 
      forControlEvents:UIControlEventPrimaryActionTriggered];
        
        [self.view addSubview:btn];
    }
}

- (void)setupMusicPlayer {
    // ตั้งค่า AVPlayer สำหรับเพลง
}

- (void)setupGameControllerSupport {
    GCController *controller = [GCController controllers].firstObject;
    if (!controller) return;
    
    GCExtendedGamepad *gamepad = controller.extendedGamepad;
    if (!gamepad) return;
    
    // Left/Right ข้ามเพลง
    gamepad.leftShoulder.pressedChangedHandler = ^(GCControllerButtonInput *button, 
                                                     float value, BOOL pressed) {
        if (pressed) [self playPrevious];
    };
    
    gamepad.rightShoulder.pressedChangedHandler = ^(GCControllerButtonInput *button, 
                                                     float value, BOOL pressed) {
        if (pressed) [self playNext];
    };
    
    // A = Play/Pause
    gamepad.buttonA.pressedChangedHandler = ^(GCControllerButtonInput *button, 
                                               float value, BOOL pressed) {
        if (pressed) [self togglePlayPause];
    };
}

- (void)controlButtonTapped:(UIButton *)button {
    switch (button.tag) {
        case 0: [self playPrevious]; break;
        case 1: [self seekBackward]; break;
        case 2: [self togglePlayPause]; break;
        case 3: [self seekForward]; break;
        case 4: [self playNext]; break;
    }
}

- (void)playPrevious { NSLog(@"Play previous"); }
- (void)playNext { NSLog(@"Play next"); }
- (void)seekBackward { NSLog(@"Seek -10s"); }
- (void)seekForward { NSLog(@"Seek +10s"); }
- (void)togglePlayPause { NSLog(@"Toggle play/pause"); }

@end
```

### Exercise 3: tvOS Settings Screen

```objc
// SettingsViewController.m - ตัวอย่าง Settings UI สำหรับ tvOS
@interface SettingsViewController : UITableViewController

@end

@implementation SettingsViewController

- (void)viewDidLoad {
    [super viewDidLoad];
    self.title = @"การตั้งค่า";
    
    [self.tableView registerClass:[UITableViewCell class] forCellReuseIdentifier:@"Cell"];
}

- (NSInteger)numberOfSectionsInTableView:(UITableView *)tableView {
    return 3;
}

- (NSString *)tableView:(UITableView *)tableView titleForHeaderInSection:(NSInteger)section {
    switch (section) {
        case 0: return @"บัญชี";
        case 1: return @"การแสดงผล";
        case 2: return @"เสียง";
        default: return nil;
    }
}

- (NSInteger)tableView:(UITableView *)tableView numberOfRowsInSection:(NSInteger)section {
    switch (section) {
        case 0: return 2;
        case 1: return 3;
        case 2: return 2;
        default: return 0;
    }
}

- (UITableViewCell *)tableView:(UITableView *)tableView 
         cellForRowAtIndexPath:(NSIndexPath *)indexPath {
    
    UITableViewCell *cell = [tableView dequeueReusableCellWithIdentifier:@"Cell" 
                                                            forIndexPath:indexPath];
    
    cell.textLabel.font = [UIFont systemFontOfSize:36];
    cell.detailTextLabel.font = [UIFont systemFontOfSize:30];
    cell.accessoryType = UITableViewCellAccessoryDisclosureIndicator;
    
    // กำหนดเนื้อหา
    NSDictionary *settings = @{
        @"0-0": @{@"title": @"เข้าสู่ระบบ", @"detail": @"user@example.com"},
        @"0-1": @{@"title": @"โปรไฟล์", @"detail": @""},
        @"1-0": @{@"title": @"ความละเอียด", @"detail": @"4K HDR"},
        @"1-1": @{@"title": @"ขนาดตัวอักษร", @"detail": @"ปกติ"},
        @"1-2": @{@"title": @"คำบรรยาย", @"detail": @"ไทย"},
        @"2-0": @{@"title": @"ภาษาเสียง", @"detail": @"ไทย"},
        @"2-1": @{@"title": @"ระดับเสียง", @"detail": @""},
    };
    
    NSString *key = [NSString stringWithFormat:@"%ld-%ld", 
                     (long)indexPath.section, (long)indexPath.row];
    NSDictionary *info = settings[key];
    
    cell.textLabel.text = info[@"title"];
    cell.detailTextLabel.text = info[@"detail"];
    
    return cell;
}

- (void)tableView:(UITableView *)tableView 
    didSelectRowAtIndexPath:(NSIndexPath *)indexPath {
    
    NSLog(@"Settings item selected: %ld-%ld", (long)indexPath.section, (long)indexPath.row);
    
    // Navigate to detail settings
}

@end
```

---

## สรุป (Summary)

ในบทนี้เราได้เรียนรู้:

1. **tvOS App Structure** - ความแตกต่างจาก iOS และโครงสร้างพื้นฐาน
2. **Focus Engine** - ระบบ focus ที่เป็นหัวใจของ tvOS UI
3. **UIFocusEnvironment** - Protocol สำหรับจัดการ focus behavior
4. **Remote Control** - การจัดการ Siri Remote ผ่าน gestures และ press events
5. **TVMLKit** - สร้าง UI ด้วย XML templates
6. **Top Shelf Extension** - แสดงเนื้อหาบน Home screen
7. **TVServices** - Framework สำหรับ tvOS-specific features
8. **Game Controller** - รองรับ MFi controllers และ Siri Remote เป็น gamepad
9. **AVKit for tvOS** - Video playback ที่ optimize สำหรับ TV

### Best Practices สำหรับ tvOS

- ออกแบบสำหรับ "10-foot UI" - ตัวอักษรและปุ่มต้องใหญ่พอที่จะเห็นจากระยะ 3 เมตร
- Focus path ต้องเป็นธรรมชาติและ predictable
- ใช้ Focus Engine อย่างถูกต้อง อย่า override behavior โดยไม่จำเป็น
- รองรับทั้ง Siri Remote และ Game Controllers
- Content ควร load เร็ว เพราะผู้ใช้นั่งดูโทรทัศน์ (ไม่มีความอดทนนาน)
- ใช้ AVKit สำหรับ video playback เพื่อได้ features ที่ครบถ้วน
- ทดสอบบน Apple TV จริงเสมอ เพราะ simulator มีข้อจำกัดมาก

---

*ต่อไป: Part 85 - (หัวข้อถัดไปในคอร์ส)*
