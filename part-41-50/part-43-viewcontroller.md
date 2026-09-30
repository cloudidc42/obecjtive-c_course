# ตอนที่ 43: UIViewController (View Controller)

## บทนำ

`UIViewController` คือหัวใจสำคัญของการพัฒนาแอป iOS ทำหน้าที่เป็น Controller ใน MVC pattern จัดการความสัมพันธ์ระหว่าง Model และ View ควบคุม lifecycle ของ View และจัดการ navigation ระหว่างหน้าจอต่างๆ

ในตอนนี้เราจะเรียนรู้ทุกแง่มุมของ UIViewController ตั้งแต่ lifecycle ไปจนถึงการส่งข้อมูลระหว่าง View Controllers

---

## 43.1 UIViewController Lifecycle

### ภาพรวม Lifecycle

```
init
  ↓
loadView            (สร้าง View)
  ↓
viewDidLoad         (View โหลดแล้ว - ทำได้ครั้งเดียว)
  ↓
viewWillAppear      (กำลังจะแสดง - ทุกครั้ง)
  ↓
viewDidLayoutSubviews (จัด layout เสร็จ)
  ↓
viewDidAppear       (แสดงแล้ว - ทุกครั้ง)
  ↓
  ↕ (แสดง/ซ่อน ซ้ำๆ)
  ↓
viewWillDisappear   (กำลังจะหายไป)
  ↓
viewDidDisappear    (หายไปแล้ว)
  ↓
deinit/dealloc      (ถูกลบออกจาก memory)
```

### viewDidLoad

```objc
- (void)viewDidLoad {
    [super viewDidLoad]; // ต้องเรียก super เสมอ
    
    // เรียกครั้งเดียวเมื่อ view โหลดเสร็จ
    // ใช้สำหรับ:
    // - ตั้งค่า UI elements
    // - สมัคร notifications
    // - ตั้งค่า delegates
    // - โหลดข้อมูลครั้งแรก
    
    NSLog(@"viewDidLoad: %@", self.title);
    
    self.view.backgroundColor = [UIColor systemBackgroundColor];
    
    // สร้าง UI
    [self setupUI];
    
    // ตั้งค่า Navigation Bar
    self.title = @"หน้าแรก";
    self.navigationItem.rightBarButtonItem = 
        [[UIBarButtonItem alloc] 
            initWithBarButtonSystemItem:UIBarButtonSystemItemAdd
                                 target:self
                                 action:@selector(addButtonTapped)];
    
    // สมัคร Notification
    [[NSNotificationCenter defaultCenter]
        addObserver:self
           selector:@selector(dataDidUpdate:)
               name:@"DataUpdatedNotification"
             object:nil];
    
    // ตรวจสอบ trait (Dark/Light mode)
    if (self.traitCollection.userInterfaceStyle == UIUserInterfaceStyleDark) {
        NSLog(@"Dark Mode");
    }
}
```

### viewWillAppear และ viewDidAppear

```objc
// viewWillAppear: เรียกทุกครั้งก่อน view จะแสดง
// ใช้สำหรับ: refresh ข้อมูล, เริ่ม animations, อัปเดต UI
- (void)viewWillAppear:(BOOL)animated {
    [super viewWillAppear:animated];
    
    NSLog(@"viewWillAppear: %@", self.title);
    
    // รีเฟรชข้อมูล
    [self refreshData];
    
    // เริ่ม Timer
    if (!self.timer) {
        self.timer = [NSTimer scheduledTimerWithTimeInterval:1.0
                                                     target:self
                                                   selector:@selector(timerTick)
                                                   userInfo:nil
                                                    repeats:YES];
    }
    
    // อัปเดต navigation bar
    [self.navigationController setNavigationBarHidden:NO animated:animated];
    
    // Track screen view (Analytics)
    // [Analytics trackScreenView:@"HomeScreen"];
}

// viewDidAppear: เรียกหลัง view แสดงเสร็จแล้ว
// ใช้สำหรับ: เริ่ม animation ที่ต้องการ view บนหน้าจอ
- (void)viewDidAppear:(BOOL)animated {
    [super viewDidAppear:animated];
    
    NSLog(@"viewDidAppear: %@", self.title);
    
    // เริ่ม animation เมื่อ view แสดงแล้ว
    [UIView animateWithDuration:0.5 animations:^{
        self.welcomeLabel.alpha = 1.0;
        self.welcomeLabel.transform = CGAffineTransformIdentity;
    }];
    
    // แสดง Tutorial (ครั้งแรก)
    NSUserDefaults *defaults = [NSUserDefaults standardUserDefaults];
    if (![defaults boolForKey:@"tutorialShown"]) {
        [self showTutorial];
        [defaults setBool:YES forKey:@"tutorialShown"];
    }
    
    // Request permissions
    [self requestPermissionsIfNeeded];
}
```

### viewWillDisappear และ viewDidDisappear

```objc
// viewWillDisappear: กำลังจะหายไป
// ใช้สำหรับ: บันทึกข้อมูล, หยุด tasks
- (void)viewWillDisappear:(BOOL)animated {
    [super viewWillDisappear:animated];
    
    NSLog(@"viewWillDisappear: %@", self.title);
    
    // หยุด Timer
    [self.timer invalidate];
    self.timer = nil;
    
    // บันทึก draft
    [self saveDraft];
    
    // ซ่อน keyboard
    [self.view endEditing:YES];
    
    // ยกเลิก network requests
    [self.currentTask cancel];
}

// viewDidDisappear: หายไปแล้ว
// ใช้สำหรับ: cleanup ที่ต้องทำหลัง view หายไป
- (void)viewDidDisappear:(BOOL)animated {
    [super viewDidDisappear:animated];
    
    NSLog(@"viewDidDisappear: %@", self.title);
    
    // หยุด media playback
    [self.player pause];
}
```

---

## 43.2 loadView

### loadView คืออะไร?

`loadView` เรียกเมื่อ `self.view` ถูกเรียกใช้ครั้งแรกและยังเป็น nil Override เมื่อต้องการสร้าง view เองทั้งหมดโดยไม่ใช้ nib/storyboard

```objc
// Override loadView เพื่อสร้าง custom view
- (void)loadView {
    // สร้าง custom root view
    MyCustomView *customView = [[MyCustomView alloc] init];
    self.view = customView;
    
    // หมายเหตุ: ห้ามเรียก [super loadView] เมื่อ override นี้
    // (ถ้าเรียก super มันจะโหลด nib/storyboard แทน)
}

// ตัวอย่างการสร้าง Custom Root View
@interface LoginView : UIView

@property (nonatomic, strong) UITextField *emailField;
@property (nonatomic, strong) UITextField *passwordField;
@property (nonatomic, strong) UIButton *loginButton;

@end

@implementation LoginView

- (instancetype)init {
    self = [super init];
    if (self) {
        self.backgroundColor = [UIColor systemBackgroundColor];
        [self setupSubviews];
        [self setupConstraints];
    }
    return self;
}

- (void)setupSubviews {
    self.emailField = [[UITextField alloc] init];
    self.emailField.translatesAutoresizingMaskIntoConstraints = NO;
    self.emailField.placeholder = @"Email";
    self.emailField.borderStyle = UITextBorderStyleRoundedRect;
    [self addSubview:self.emailField];
    
    self.passwordField = [[UITextField alloc] init];
    self.passwordField.translatesAutoresizingMaskIntoConstraints = NO;
    self.passwordField.placeholder = @"Password";
    self.passwordField.secureTextEntry = YES;
    self.passwordField.borderStyle = UITextBorderStyleRoundedRect;
    [self addSubview:self.passwordField];
    
    self.loginButton = [UIButton buttonWithType:UIButtonTypeSystem];
    self.loginButton.translatesAutoresizingMaskIntoConstraints = NO;
    [self.loginButton setTitle:@"Login" forState:UIControlStateNormal];
    self.loginButton.backgroundColor = [UIColor systemBlueColor];
    [self.loginButton setTitleColor:[UIColor whiteColor] 
                           forState:UIControlStateNormal];
    self.loginButton.layer.cornerRadius = 8;
    [self addSubview:self.loginButton];
}

- (void)setupConstraints {
    UILayoutGuide *safeArea = self.safeAreaLayoutGuide;
    [NSLayoutConstraint activateConstraints:@[
        [self.emailField.topAnchor constraintEqualToAnchor:safeArea.topAnchor constant:100],
        [self.emailField.leadingAnchor constraintEqualToAnchor:self.leadingAnchor constant:24],
        [self.emailField.trailingAnchor constraintEqualToAnchor:self.trailingAnchor constant:-24],
        [self.emailField.heightAnchor constraintEqualToConstant:44],
        
        [self.passwordField.topAnchor constraintEqualToAnchor:self.emailField.bottomAnchor constant:12],
        [self.passwordField.leadingAnchor constraintEqualToAnchor:self.emailField.leadingAnchor],
        [self.passwordField.trailingAnchor constraintEqualToAnchor:self.emailField.trailingAnchor],
        [self.passwordField.heightAnchor constraintEqualToConstant:44],
        
        [self.loginButton.topAnchor constraintEqualToAnchor:self.passwordField.bottomAnchor constant:24],
        [self.loginButton.leadingAnchor constraintEqualToAnchor:self.emailField.leadingAnchor],
        [self.loginButton.trailingAnchor constraintEqualToAnchor:self.emailField.trailingAnchor],
        [self.loginButton.heightAnchor constraintEqualToConstant:50],
    ]];
}

@end

// LoginViewController
@implementation LoginViewController

- (void)loadView {
    LoginView *loginView = [[LoginView alloc] init];
    self.view = loginView;
}

- (LoginView *)loginView {
    return (LoginView *)self.view;
}

- (void)viewDidLoad {
    [super viewDidLoad];
    
    // ตั้งค่า actions บน custom view
    [self.loginView.loginButton addTarget:self
                                   action:@selector(loginTapped)
                         forControlEvents:UIControlEventTouchUpInside];
    
    self.loginView.emailField.delegate = self;
    self.loginView.passwordField.delegate = self;
}

@end
```

---

## 43.3 viewDidLayoutSubviews

### viewDidLayoutSubviews คืออะไร?

เรียกหลัง view จัดการ layout ของ subviews เสร็จ (หลัง Auto Layout ทำงาน) เรียกบ่อยกว่า viewDidLoad เพราะจะเรียกทุกครั้งที่ layout เปลี่ยน

```objc
- (void)viewDidLayoutSubviews {
    [super viewDidLayoutSubviews];
    
    // ใช้เมื่อต้องการ frame ที่แน่นอน (หลัง Auto Layout)
    
    // ทำ circle จาก frame ที่รู้แน่นอนแล้ว
    self.avatarImageView.layer.cornerRadius = 
        self.avatarImageView.frame.size.width / 2;
    
    // Gradient layer ที่ต้องรู้ขนาด
    self.gradientLayer.frame = self.gradientView.bounds;
    
    // ปรับ shadow path
    UIBezierPath *shadowPath = [UIBezierPath 
        bezierPathWithRoundedRect:self.cardView.bounds 
                    cornerRadius:12];
    self.cardView.layer.shadowPath = shadowPath.CGPath;
    
    NSLog(@"viewDidLayoutSubviews - cardView frame: %@", 
          NSStringFromCGRect(self.cardView.frame));
}

// ตัวอย่างการใช้ Gradient Layer
- (void)setupGradient {
    CAGradientLayer *gradient = [CAGradientLayer layer];
    gradient.colors = @[
        (id)[UIColor systemBlueColor].CGColor,
        (id)[UIColor systemPurpleColor].CGColor
    ];
    gradient.startPoint = CGPointMake(0, 0);
    gradient.endPoint = CGPointMake(1, 1);
    
    self.gradientLayer = gradient;
    [self.headerView.layer insertSublayer:gradient atIndex:0];
    // Frame จะถูกตั้งใน viewDidLayoutSubviews
}
```

---

## 43.4 Memory Warnings

### การจัดการ Memory Warning

iOS จะส่ง memory warning เมื่อหน่วยความจำเริ่มจะไม่พอ แอปควร release ข้อมูลที่ไม่จำเป็น

```objc
// Override didReceiveMemoryWarning
- (void)didReceiveMemoryWarning {
    [super didReceiveMemoryWarning];
    
    NSLog(@"Memory Warning! ล้าง cache...");
    
    // ล้าง cache ต่างๆ
    [self.imageCache removeAllObjects];
    [self.dataCache removeAllObjects];
    
    // ล้าง offline data (สามารถ load ใหม่ได้)
    self.cachedResults = nil;
    
    // หยุด background tasks ที่ไม่จำเป็น
    [self.backgroundQueue cancelAllOperations];
}

// สมัครรับ notification
- (void)viewDidLoad {
    [super viewDidLoad];
    
    [[NSNotificationCenter defaultCenter]
        addObserver:self
           selector:@selector(handleMemoryWarning)
               name:UIApplicationDidReceiveMemoryWarningNotification
             object:nil];
}

- (void)handleMemoryWarning {
    NSLog(@"Received memory warning via notification");
    [self clearCaches];
}

// ล้าง notification เมื่อ deallocate
- (void)dealloc {
    [[NSNotificationCenter defaultCenter] removeObserver:self];
    NSLog(@"ViewController deallocated: %@", self.title);
}
```

### การป้องกัน Memory Leaks

```objc
// Retain Cycle ที่พบบ่อย
@interface MyViewController ()
@property (nonatomic, strong) NSTimer *timer;
@property (nonatomic, strong) dispatch_block_t completionBlock;
@end

@implementation MyViewController

// ❌ Retain Cycle: timer retain self, self retain timer
- (void)startTimerBad {
    self.timer = [NSTimer scheduledTimerWithTimeInterval:1.0
                                                  target:self  // strong reference!
                                                selector:@selector(tick)
                                                userInfo:nil
                                                 repeats:YES];
}

// ✅ ไม่มี Retain Cycle: ใช้ weak reference
- (void)startTimerGood {
    __weak typeof(self) weakSelf = self;
    self.timer = [NSTimer scheduledTimerWithTimeInterval:1.0
                                                repeats:YES
                                                  block:^(NSTimer *timer) {
        __strong typeof(weakSelf) strongSelf = weakSelf;
        if (!strongSelf) {
            [timer invalidate];
            return;
        }
        [strongSelf tick];
    }];
}

// ❌ Retain Cycle ใน Block
- (void)setupBlockBad {
    self.completionBlock = ^{
        [self doSomething];  // self retain block, block retain self
    };
}

// ✅ ถูกต้อง: ใช้ weakSelf ใน block
- (void)setupBlockGood {
    __weak typeof(self) weakSelf = self;
    self.completionBlock = ^{
        __strong typeof(weakSelf) strongSelf = weakSelf;
        [strongSelf doSomething];
    };
}

- (void)dealloc {
    [self.timer invalidate];
    self.timer = nil;
    [[NSNotificationCenter defaultCenter] removeObserver:self];
}

@end
```

---

## 43.5 UINavigationController

### UINavigationController คืออะไร?

`UINavigationController` จัดการ stack ของ view controllers แบบ push/pop ทำให้ผู้ใช้ navigate ไปข้างหน้าและกลับมาได้

```objc
// สร้าง Navigation Controller
UIViewController *rootVC = [[HomeViewController alloc] init];
UINavigationController *navController = 
    [[UINavigationController alloc] initWithRootViewController:rootVC];

// ตั้งเป็น root ของ window
self.window.rootViewController = navController;

// Push ไปหน้าใหม่
DetailViewController *detailVC = [[DetailViewController alloc] init];
detailVC.title = @"รายละเอียด";
[self.navigationController pushViewController:detailVC animated:YES];

// Pop กลับ
[self.navigationController popViewControllerAnimated:YES];

// Pop กลับไป root
[self.navigationController popToRootViewControllerAnimated:YES];

// Pop กลับไป VC เฉพาะ
NSArray *viewControllers = self.navigationController.viewControllers;
for (UIViewController *vc in viewControllers) {
    if ([vc isKindOfClass:[ListViewController class]]) {
        [self.navigationController popToViewController:vc animated:YES];
        break;
    }
}

// ตั้งค่า Navigation Bar
self.navigationController.navigationBar.prefersLargeTitles = YES;
self.navigationItem.largeTitleDisplayMode = UINavigationItemLargeTitleDisplayModeAutomatic;

// Navigation Bar Appearance (iOS 15+)
if (@available(iOS 15.0, *)) {
    UINavigationBarAppearance *appearance = [[UINavigationBarAppearance alloc] init];
    [appearance configureWithOpaqueBackground];
    appearance.backgroundColor = [UIColor systemBlueColor];
    appearance.titleTextAttributes = @{
        NSForegroundColorAttributeName: [UIColor whiteColor]
    };
    appearance.largeTitleTextAttributes = @{
        NSForegroundColorAttributeName: [UIColor whiteColor],
        NSFontAttributeName: [UIFont boldSystemFontOfSize:34]
    };
    
    self.navigationController.navigationBar.standardAppearance = appearance;
    self.navigationController.navigationBar.scrollEdgeAppearance = appearance;
    self.navigationController.navigationBar.compactAppearance = appearance;
}

// Bar Button Items
// ซ้าย
UIBarButtonItem *backButton = [[UIBarButtonItem alloc]
    initWithTitle:@"กลับ" 
            style:UIBarButtonItemStylePlain 
           target:self 
           action:@selector(backTapped)];
self.navigationItem.leftBarButtonItem = backButton;

// ขวา
UIBarButtonItem *addButton = [[UIBarButtonItem alloc]
    initWithBarButtonSystemItem:UIBarButtonSystemItemAdd
                         target:self
                         action:@selector(addTapped)];
UIBarButtonItem *editButton = [[UIBarButtonItem alloc]
    initWithBarButtonSystemItem:UIBarButtonSystemItemEdit
                         target:self
                         action:@selector(editTapped)];

self.navigationItem.rightBarButtonItems = @[addButton, editButton];

// Custom View ใน Navigation Bar
UIButton *customButton = [UIButton buttonWithType:UIButtonTypeCustom];
[customButton setImage:[UIImage systemImageNamed:@"bell.fill"] 
              forState:UIControlStateNormal];
customButton.frame = CGRectMake(0, 0, 44, 44);
UIBarButtonItem *customItem = [[UIBarButtonItem alloc] initWithCustomView:customButton];
self.navigationItem.rightBarButtonItem = customItem;

// ซ่อน/แสดง Navigation Bar
[self.navigationController setNavigationBarHidden:YES animated:YES];
```

### UINavigationControllerDelegate

```objc
// รู้เมื่อ push/pop
@interface HomeViewController () <UINavigationControllerDelegate>
@end

- (void)viewDidLoad {
    [super viewDidLoad];
    self.navigationController.delegate = self;
}

- (void)navigationController:(UINavigationController *)navigationController
       willShowViewController:(UIViewController *)viewController
                     animated:(BOOL)animated {
    NSLog(@"กำลังแสดง: %@", viewController.title);
}

- (void)navigationController:(UINavigationController *)navigationController
        didShowViewController:(UIViewController *)viewController
                     animated:(BOOL)animated {
    NSLog(@"แสดงแล้ว: %@", viewController.title);
}
```

---

## 43.6 UITabBarController

### UITabBarController คืออะไร?

`UITabBarController` แสดงแถบ tabs ด้านล่าง ให้ผู้ใช้สลับระหว่าง view controllers ได้

```objc
// สร้าง TabBarController
UITabBarController *tabBarController = [[UITabBarController alloc] init];

// Tab 1: Home
HomeViewController *homeVC = [[HomeViewController alloc] init];
homeVC.title = @"หน้าแรก";
homeVC.tabBarItem = [[UITabBarItem alloc] 
    initWithTitle:@"หน้าแรก"
            image:[UIImage systemImageNamed:@"house"]
              tag:0];

UINavigationController *homeNav = 
    [[UINavigationController alloc] initWithRootViewController:homeVC];

// Tab 2: Search
SearchViewController *searchVC = [[SearchViewController alloc] init];
searchVC.tabBarItem = [[UITabBarItem alloc]
    initWithTabBarSystemItem:UITabBarSystemItemSearch
                         tag:1];

// Tab 3: Profile
ProfileViewController *profileVC = [[ProfileViewController alloc] init];
profileVC.tabBarItem = [[UITabBarItem alloc]
    initWithTitle:@"โปรไฟล์"
            image:[UIImage systemImageNamed:@"person"]
    selectedImage:[UIImage systemImageNamed:@"person.fill"]];

UINavigationController *profileNav = 
    [[UINavigationController alloc] initWithRootViewController:profileVC];

// ตั้งค่า View Controllers
tabBarController.viewControllers = @[homeNav, searchVC, profileNav];

// เลือก tab เริ่มต้น
tabBarController.selectedIndex = 0;

// Badge
homeVC.tabBarItem.badgeValue = @"3";
homeVC.tabBarItem.badgeColor = [UIColor systemRedColor];

// TabBar Appearance (iOS 15+)
if (@available(iOS 15.0, *)) {
    UITabBarAppearance *appearance = [[UITabBarAppearance alloc] init];
    [appearance configureWithDefaultBackground];
    
    UITabBarItemAppearance *itemAppearance = [[UITabBarItemAppearance alloc] init];
    itemAppearance.selected.iconColor = [UIColor systemBlueColor];
    itemAppearance.selected.titleTextAttributes = @{
        NSForegroundColorAttributeName: [UIColor systemBlueColor]
    };
    
    appearance.stackedLayoutAppearance = itemAppearance;
    appearance.inlineLayoutAppearance = itemAppearance;
    appearance.compactInlineLayoutAppearance = itemAppearance;
    
    tabBarController.tabBar.standardAppearance = appearance;
    tabBarController.tabBar.scrollEdgeAppearance = appearance;
}

// UITabBarControllerDelegate
tabBarController.delegate = self;

- (void)tabBarController:(UITabBarController *)tabBarController
 didSelectViewController:(UIViewController *)viewController {
    NSLog(@"เลือก tab: %@", viewController.title);
}

- (BOOL)tabBarController:(UITabBarController *)tabBarController
    shouldSelectViewController:(UIViewController *)viewController {
    // ตรวจสอบก่อนสลับ (เช่น ต้อง login ก่อน)
    if ([viewController isKindOfClass:[ProfileViewController class]]) {
        if (!self.isLoggedIn) {
            [self showLoginScreen];
            return NO;
        }
    }
    return YES;
}
```

---

## 43.7 Presenting View Controllers

### Modal Presentation

```objc
// Present Modal View Controller
ModalViewController *modalVC = [[ModalViewController alloc] init];

// Presentation Style
modalVC.modalPresentationStyle = UIModalPresentationPageSheet;  // Default iOS 13+
modalVC.modalPresentationStyle = UIModalPresentationFullScreen; // เต็มจอ
modalVC.modalPresentationStyle = UIModalPresentationFormSheet;  // iPad form
modalVC.modalPresentationStyle = UIModalPresentationOverFullScreen; // ทับ
modalVC.modalPresentationStyle = UIModalPresentationPopover;    // Popover

// Transition Style
modalVC.modalTransitionStyle = UIModalTransitionStyleCoverVertical;  // Default (ขึ้นจากล่าง)
modalVC.modalTransitionStyle = UIModalTransitionStyleCrossDissolve;  // Fade
modalVC.modalTransitionStyle = UIModalTransitionStyleFlipHorizontal; // Flip
modalVC.modalTransitionStyle = UIModalTransitionStylePartialCurl;    // Page curl

// Present
[self presentViewController:modalVC animated:YES completion:^{
    NSLog(@"แสดง modal เสร็จแล้ว");
}];

// Present with Navigation Controller
UINavigationController *navController = 
    [[UINavigationController alloc] initWithRootViewController:modalVC];
[self presentViewController:navController animated:YES completion:nil];

// Sheet Presentation (iOS 15+)
if (@available(iOS 15.0, *)) {
    UISheetPresentationController *sheet = modalVC.sheetPresentationController;
    sheet.detents = @[
        [UISheetPresentationControllerDetent mediumDetent],
        [UISheetPresentationControllerDetent largeDetent]
    ];
    sheet.prefersGrabberVisible = YES;
    sheet.preferredCornerRadius = 20;
    sheet.largestUndimmedDetentIdentifier = 
        UISheetPresentationControllerDetentIdentifierMedium;
    
    [self presentViewController:modalVC animated:YES completion:nil];
}
```

### Dismiss

```objc
// Dismiss จาก VC ที่ถูก present (child)
- (void)dismissSelf {
    [self dismissViewControllerAnimated:YES completion:^{
        NSLog(@"ปิด modal แล้ว");
    }];
}

// Dismiss จาก Presenter (parent)
- (void)dismissModal {
    [self dismissViewControllerAnimated:YES completion:nil];
}

// Dismiss all modals
UIViewController *rootVC = self.view.window.rootViewController;
while (rootVC.presentedViewController) {
    rootVC = rootVC.presentedViewController;
}
[rootVC dismissViewControllerAnimated:YES completion:nil];

// ตรวจสอบว่ามี presented VC
if (self.presentedViewController) {
    [self dismissViewControllerAnimated:YES completion:nil];
}
```

### Popover (iPad)

```objc
InfoViewController *infoVC = [[InfoViewController alloc] init];
infoVC.modalPresentationStyle = UIModalPresentationPopover;

UIPopoverPresentationController *popover = infoVC.popoverPresentationController;
popover.sourceView = sender;  // ปุ่มที่กด
popover.sourceRect = sender.bounds;
popover.permittedArrowDirections = UIPopoverArrowDirectionUp;
popover.delegate = self;

[self presentViewController:infoVC animated:YES completion:nil];

// UIPopoverPresentationControllerDelegate
- (UIModalPresentationStyle)adaptivePresentationStyleForPresentationController:
    (UIPresentationController *)controller {
    return UIModalPresentationNone; // แสดงเป็น popover บน iPhone ด้วย
}
```

---

## 43.8 Segues (แนวคิด)

### Segue คืออะไร?

Segue เป็น connection ระหว่าง scenes ใน Storyboard ใช้สำหรับ navigate ระหว่าง view controllers

### ประเภท Segue

```
Show (Push)           - Push ใน navigation stack
Show Detail           - สำหรับ split view
Present Modally       - Modal presentation
Present As Popover    - Popover บน iPad
Custom               - Custom transition
```

### การทำงานกับ Segue ใน Code

```objc
// Trigger segue จาก code
[self performSegueWithIdentifier:@"ShowDetail" sender:self];
[self performSegueWithIdentifier:@"ShowDetail" sender:selectedItem];

// เตรียมข้อมูลก่อน segue
- (void)prepareForSegue:(UIStoryboardSegue *)segue sender:(id)sender {
    if ([segue.identifier isEqualToString:@"ShowDetail"]) {
        DetailViewController *destVC = segue.destinationViewController;
        
        // ถ้า destination อยู่ใน Navigation Controller
        if ([destVC isKindOfClass:[UINavigationController class]]) {
            UINavigationController *navVC = (UINavigationController *)destVC;
            destVC = navVC.topViewController;
        }
        
        // ส่งข้อมูล
        if ([destVC isKindOfClass:[DetailViewController class]]) {
            DetailViewController *detail = (DetailViewController *)destVC;
            detail.item = self.selectedItem;
            detail.delegate = self;
        }
    }
}

// ตรวจสอบก่อน perform segue
- (BOOL)shouldPerformSegueWithIdentifier:(NSString *)identifier sender:(id)sender {
    if ([identifier isEqualToString:@"ShowDetail"]) {
        if (!self.selectedItem) {
            // ไม่มีข้อมูล ไม่ไป
            return NO;
        }
    }
    return YES;
}

// Unwind Segue - กลับไปยัง VC ที่ผ่านมา
// ใน destination VC ต้องมี method นี้
- (IBAction)unwindToHome:(UIStoryboardSegue *)segue {
    // รับข้อมูลจาก source
    UIViewController *sourceVC = segue.sourceViewController;
    if ([sourceVC isKindOfClass:[CreateItemViewController class]]) {
        CreateItemViewController *createVC = (CreateItemViewController *)sourceVC;
        [self addItem:createVC.createdItem];
    }
}
```

---

## 43.9 Passing Data Between View Controllers

### วิธีที่ 1: Properties (Forward)

```objc
// ส่งข้อมูลไปข้างหน้า
DetailViewController *detailVC = [[DetailViewController alloc] init];

// ตั้งค่า properties ก่อน push/present
detailVC.item = self.selectedItem;
detailVC.userId = self.currentUserId;
detailVC.delegate = self;

[self.navigationController pushViewController:detailVC animated:YES];
```

### วิธีที่ 2: Delegation (Backward)

```objc
// Protocol สำหรับส่งข้อมูลกลับ
// CreateItemViewController.h

@protocol CreateItemDelegate <NSObject>
- (void)createItemViewController:(CreateItemViewController *)vc
                     didCreateItem:(Item *)item;
- (void)createItemViewControllerDidCancel:(CreateItemViewController *)vc;
@end

@interface CreateItemViewController : UIViewController
@property (nonatomic, weak) id<CreateItemDelegate> delegate;
@end

// CreateItemViewController.m
- (void)saveButtonTapped {
    Item *newItem = [self createItem];
    [self.delegate createItemViewController:self didCreateItem:newItem];
}

- (void)cancelButtonTapped {
    [self.delegate createItemViewControllerDidCancel:self];
}

// HomeViewController.m
- (void)showCreateItemScreen {
    CreateItemViewController *createVC = [[CreateItemViewController alloc] init];
    createVC.delegate = self;  // ตั้งตัวเองเป็น delegate
    
    UINavigationController *navVC = [[UINavigationController alloc] 
        initWithRootViewController:createVC];
    [self presentViewController:navVC animated:YES completion:nil];
}

- (void)createItemViewController:(CreateItemViewController *)vc 
                    didCreateItem:(Item *)item {
    [self dismissViewControllerAnimated:YES completion:nil];
    [self.items addObject:item];
    [self.tableView reloadData];
}

- (void)createItemViewControllerDidCancel:(CreateItemViewController *)vc {
    [self dismissViewControllerAnimated:YES completion:nil];
}
```

### วิธีที่ 3: Notification Center

```objc
// ส่ง notification
[[NSNotificationCenter defaultCenter] 
    postNotificationName:@"ItemCreatedNotification"
                  object:self
                userInfo:@{@"item": newItem}];

// รับ notification
[[NSNotificationCenter defaultCenter]
    addObserver:self
       selector:@selector(itemCreated:)
           name:@"ItemCreatedNotification"
         object:nil];

- (void)itemCreated:(NSNotification *)notification {
    Item *item = notification.userInfo[@"item"];
    [self.items addObject:item];
    [self.tableView reloadData];
}

// ลืม removeObserver เมื่อไม่ใช้แล้ว!
- (void)dealloc {
    [[NSNotificationCenter defaultCenter] removeObserver:self];
}
```

### วิธีที่ 4: Shared Model/Singleton

```objc
// DataManager.h - Singleton
@interface DataManager : NSObject

@property (nonatomic, strong) NSMutableArray *items;
@property (nonatomic, strong) User *currentUser;

+ (instancetype)sharedManager;

@end

// DataManager.m
@implementation DataManager

+ (instancetype)sharedManager {
    static DataManager *manager = nil;
    static dispatch_once_t onceToken;
    dispatch_once(&onceToken, ^{
        manager = [[DataManager alloc] init];
    });
    return manager;
}

@end

// ใช้จาก ViewController ใดก็ได้
DataManager *manager = [DataManager sharedManager];
manager.items = @[item1, item2];
NSLog(@"User: %@", manager.currentUser.name);
```

### วิธีที่ 5: Completion Blocks

```objc
// ViewController ที่ส่ง completion block
@interface ColorPickerViewController : UIViewController

@property (nonatomic, copy) void (^completionHandler)(UIColor *selectedColor);

@end

// แสดง ColorPicker
ColorPickerViewController *picker = [[ColorPickerViewController alloc] init];
picker.completionHandler = ^(UIColor *color) {
    self.view.backgroundColor = color;
};
[self presentViewController:picker animated:YES completion:nil];

// ใน ColorPickerViewController
- (void)colorSelected:(UIColor *)color {
    if (self.completionHandler) {
        self.completionHandler(color);
    }
    [self dismissViewControllerAnimated:YES completion:nil];
}
```

---

## 43.10 Custom View Controller Transitions

### UIViewControllerTransitioningDelegate

```objc
// Custom Transition Animator
// SlideAnimator.h

#import <UIKit/UIKit.h>

typedef NS_ENUM(NSInteger, SlideDirection) {
    SlideDirectionLeft,
    SlideDirectionRight,
    SlideDirectionUp,
    SlideDirectionDown
};

@interface SlideAnimator : NSObject <UIViewControllerAnimatedTransitioning>

@property (nonatomic, assign) SlideDirection direction;
@property (nonatomic, assign) BOOL isPresenting;
@property (nonatomic, assign) NSTimeInterval duration;

@end
```

```objc
// SlideAnimator.m

@implementation SlideAnimator

- (NSTimeInterval)transitionDuration:(id<UIViewControllerContextTransitioning>)transitionContext {
    return self.duration > 0 ? self.duration : 0.3;
}

- (void)animateTransition:(id<UIViewControllerContextTransitioning>)transitionContext {
    UIView *containerView = transitionContext.containerView;
    UIViewController *fromVC = [transitionContext viewControllerForKey:UITransitionContextFromViewControllerKey];
    UIViewController *toVC = [transitionContext viewControllerForKey:UITransitionContextToViewControllerKey];
    
    CGRect finalFrame = [transitionContext finalFrameForViewController:toVC];
    CGRect initialFrame = [self initialFrameForFrame:finalFrame];
    
    if (self.isPresenting) {
        toVC.view.frame = initialFrame;
        [containerView addSubview:toVC.view];
        
        [UIView animateWithDuration:[self transitionDuration:transitionContext]
                              delay:0
             usingSpringWithDamping:0.85
              initialSpringVelocity:0.5
                            options:UIViewAnimationOptionCurveEaseOut
                         animations:^{
            toVC.view.frame = finalFrame;
        } completion:^(BOOL finished) {
            [transitionContext completeTransition:!transitionContext.transitionWasCancelled];
        }];
    } else {
        // Dismissal
        CGRect dismissFrame = [self dismissFrameForFrame:fromVC.view.frame];
        
        [UIView animateWithDuration:[self transitionDuration:transitionContext]
                         animations:^{
            fromVC.view.frame = dismissFrame;
        } completion:^(BOOL finished) {
            [fromVC.view removeFromSuperview];
            [transitionContext completeTransition:!transitionContext.transitionWasCancelled];
        }];
    }
}

- (CGRect)initialFrameForFrame:(CGRect)frame {
    switch (self.direction) {
        case SlideDirectionLeft:
            return CGRectOffset(frame, -frame.size.width, 0);
        case SlideDirectionRight:
            return CGRectOffset(frame, frame.size.width, 0);
        case SlideDirectionUp:
            return CGRectOffset(frame, 0, -frame.size.height);
        case SlideDirectionDown:
        default:
            return CGRectOffset(frame, 0, frame.size.height);
    }
}

- (CGRect)dismissFrameForFrame:(CGRect)frame {
    switch (self.direction) {
        case SlideDirectionLeft:
            return CGRectOffset(frame, frame.size.width, 0);
        case SlideDirectionRight:
            return CGRectOffset(frame, -frame.size.width, 0);
        case SlideDirectionUp:
            return CGRectOffset(frame, 0, frame.size.height);
        case SlideDirectionDown:
        default:
            return CGRectOffset(frame, 0, frame.size.height);
    }
}

@end
```

### ใช้ Custom Transition

```objc
// CustomTransitionViewController.h
@interface CustomTransitionViewController : UIViewController <UIViewControllerTransitioningDelegate>
@end

// CustomTransitionViewController.m
@interface CustomTransitionViewController ()
@property (nonatomic, strong) SlideAnimator *animator;
@end

@implementation CustomTransitionViewController

- (void)viewDidLoad {
    [super viewDidLoad];
    self.animator = [[SlideAnimator alloc] init];
    self.animator.direction = SlideDirectionUp;
}

- (void)showDetailVC {
    DetailViewController *detailVC = [[DetailViewController alloc] init];
    detailVC.transitioningDelegate = self;
    detailVC.modalPresentationStyle = UIModalPresentationCustom;
    
    [self presentViewController:detailVC animated:YES completion:nil];
}

// UIViewControllerTransitioningDelegate
- (id<UIViewControllerAnimatedTransitioning>)animationControllerForPresentedController:
    (UIViewController *)presented 
                                presentingController:(UIViewController *)presenting 
                                    sourceController:(UIViewController *)source {
    self.animator.isPresenting = YES;
    return self.animator;
}

- (id<UIViewControllerAnimatedTransitioning>)animationControllerForDismissedController:
    (UIViewController *)dismissed {
    self.animator.isPresenting = NO;
    return self.animator;
}

@end
```

### Interactive Transition (Pan Gesture)

```objc
// Interactive Dismiss ด้วย Pan Gesture
@interface PanDismissViewController () <UIViewControllerTransitioningDelegate>
@property (nonatomic, strong) UIPercentDrivenInteractiveTransition *interactiveTransition;
@property (nonatomic, assign) BOOL isInteractive;
@end

@implementation PanDismissViewController

- (void)viewDidLoad {
    [super viewDidLoad];
    
    // เพิ่ม Pan Gesture
    UIPanGestureRecognizer *panGesture = [[UIPanGestureRecognizer alloc]
        initWithTarget:self action:@selector(handlePan:)];
    [self.view addGestureRecognizer:panGesture];
}

- (void)handlePan:(UIPanGestureRecognizer *)gesture {
    CGFloat translation = [gesture translationInView:self.view].y;
    CGFloat progress = MAX(0, MIN(1, translation / self.view.bounds.size.height));
    
    switch (gesture.state) {
        case UIGestureRecognizerStateBegan:
            self.isInteractive = YES;
            [self dismissViewControllerAnimated:YES completion:nil];
            break;
            
        case UIGestureRecognizerStateChanged:
            [self.interactiveTransition updateInteractiveTransition:progress];
            break;
            
        case UIGestureRecognizerStateEnded:
        case UIGestureRecognizerStateCancelled: {
            BOOL shouldFinish = progress > 0.5;
            if (shouldFinish) {
                [self.interactiveTransition finishInteractiveTransition];
            } else {
                [self.interactiveTransition cancelInteractiveTransition];
            }
            self.isInteractive = NO;
            break;
        }
        default: break;
    }
}

- (id<UIViewControllerInteractiveTransitioning>)interactionControllerForDismissal:
    (id<UIViewControllerAnimatedTransitioning>)animator {
    
    if (self.isInteractive) {
        self.interactiveTransition = [[UIPercentDrivenInteractiveTransition alloc] init];
        return self.interactiveTransition;
    }
    return nil;
}

@end
```

---

## 43.11 Child View Controllers

### การใช้ Container View Controllers

```objc
// Parent VC เพิ่ม Child VC
- (void)addChildViewController:(UIViewController *)childVC 
                        toView:(UIView *)containerView {
    
    // 1. แจ้ง child ว่ากำลังจะถูกเพิ่ม
    [self addChildViewController:childVC];
    
    // 2. เพิ่ม child view
    childVC.view.frame = containerView.bounds;
    childVC.view.autoresizingMask = UIViewAutoresizingFlexibleWidth | 
                                   UIViewAutoresizingFlexibleHeight;
    [containerView addSubview:childVC.view];
    
    // 3. แจ้ง child ว่าเพิ่มแล้ว
    [childVC didMoveToParentViewController:self];
}

// ลบ Child VC
- (void)removeChildViewController:(UIViewController *)childVC {
    // 1. แจ้ง child ว่ากำลังจะถูกลบ
    [childVC willMoveToParentViewController:nil];
    
    // 2. ลบ view
    [childVC.view removeFromSuperview];
    
    // 3. ลบออกจาก parent
    [childVC removeFromParentViewController];
}

// ตัวอย่าง Tab-like Container
@implementation ContainerViewController

- (void)viewDidLoad {
    [super viewDidLoad];
    
    // เพิ่ม Tab Buttons
    [self setupTabButtons];
    
    // แสดง Tab แรก
    [self showTab:0];
}

- (void)showTab:(NSInteger)index {
    // ลบ child VC เดิม
    for (UIViewController *child in self.childViewControllers) {
        [self removeChildViewController:child];
    }
    
    // สร้าง VC สำหรับ tab ที่เลือก
    UIViewController *newVC = self.viewControllers[index];
    [self addChildViewController:newVC toView:self.contentView];
}

@end
```

---

## 43.12 View Controller Containment Patterns

### PageViewController

```objc
// UIPageViewController
UIPageViewController *pageVC = [[UIPageViewController alloc]
    initWithTransitionStyle:UIPageViewControllerTransitionStyleScroll
      navigationOrientation:UIPageViewControllerNavigationOrientationHorizontal
                    options:@{UIPageViewControllerOptionInterPageSpacingKey: @20}];

pageVC.dataSource = self;
pageVC.delegate = self;

// ตั้งค่า initial page
UIViewController *firstPage = self.pages[0];
[pageVC setViewControllers:@[firstPage]
                 direction:UIPageViewControllerNavigationDirectionForward
                  animated:NO
                completion:nil];

[self addChildViewController:pageVC];
pageVC.view.frame = self.pageContainerView.bounds;
[self.pageContainerView addSubview:pageVC.view];
[pageVC didMoveToParentViewController:self];

// UIPageViewControllerDataSource
- (UIViewController *)pageViewController:(UIPageViewController *)pageVC
    viewControllerBeforeViewController:(UIViewController *)vc {
    
    NSInteger index = [self.pages indexOfObject:vc];
    if (index == 0 || index == NSNotFound) return nil;
    return self.pages[index - 1];
}

- (UIViewController *)pageViewController:(UIPageViewController *)pageVC
     viewControllerAfterViewController:(UIViewController *)vc {
    
    NSInteger index = [self.pages indexOfObject:vc];
    if (index == self.pages.count - 1 || index == NSNotFound) return nil;
    return self.pages[index + 1];
}

// Page Indicator
- (NSInteger)presentationCountForPageViewController:(UIPageViewController *)pageVC {
    return self.pages.count;
}

- (NSInteger)presentationIndexForPageViewController:(UIPageViewController *)pageVC {
    return [self.pages indexOfObject:pageVC.viewControllers.firstObject];
}
```

---

## 43.13 แบบฝึกหัด

### แบบฝึกหัดที่ 1: Login Flow

```objc
/*
สร้าง Login Flow:
1. LoginViewController - มี email/password fields, login button
2. HomeViewController - หน้าหลังจาก login สำเร็จ
3. ProfileViewController - แสดงข้อมูล user

Requirements:
- Login สำเร็จ: push ไป HomeViewController (ด้วย UINavigationController)
- HomeVC มีปุ่ม Profile: present ProfileViewController เป็น modal
- ProfileVC สามารถ dismiss ตัวเอง
- ส่งข้อมูล username จาก Login ไปแสดงที่ Home และ Profile
- มีปุ่ม Logout ที่ pop กลับไป Login (clearBackStack)
*/

// HomeViewController.m
@implementation HomeViewController

- (void)logoutButtonTapped {
    // Pop ไป Login (root)
    [self.navigationController popToRootViewControllerAnimated:YES];
    
    // หรือ: สร้าง navigation controller ใหม่
    UIWindow *window = self.view.window;
    LoginViewController *loginVC = [[LoginViewController alloc] init];
    UINavigationController *navVC = [[UINavigationController alloc] 
        initWithRootViewController:loginVC];
    
    // Animate transition
    [UIView transitionWithView:window
                      duration:0.3
                       options:UIViewAnimationOptionTransitionCrossDissolve
                    animations:^{
        window.rootViewController = navVC;
    } completion:nil];
}

@end
```

### แบบฝึกหัดที่ 2: Master-Detail

```objc
/*
สร้าง Master-Detail pattern:
1. ListViewController (Master) - UITableView แสดงรายการ
2. DetailViewController (Detail) - แสดงรายละเอียด

Requirements:
- เลือกรายการใน List → push Detail
- Detail แสดงข้อมูลจาก List
- Detail มีปุ่ม Edit → present EditViewController เป็น modal
- EditVC ส่งข้อมูลที่แก้ไขกลับ DetailVC ผ่าน Delegation
- DetailVC อัปเดตข้อมูลและ reload
*/
```

### ตัวอย่าง Complete ViewController

```objc
// CompleteExampleViewController.m

#import "CompleteExampleViewController.h"
#import "ItemDetailViewController.h"

@interface CompleteExampleViewController () <UITableViewDelegate, 
                                              UITableViewDataSource,
                                              ItemDetailDelegate>

@property (nonatomic, strong) UITableView *tableView;
@property (nonatomic, strong) NSMutableArray *items;

@end

@implementation CompleteExampleViewController

#pragma mark - Lifecycle

- (void)viewDidLoad {
    [super viewDidLoad];
    self.title = @"รายการ";
    self.view.backgroundColor = [UIColor systemBackgroundColor];
    
    // ข้อมูลตัวอย่าง
    self.items = [NSMutableArray arrayWithArray:@[
        @{@"name": @"รายการที่ 1", @"detail": @"รายละเอียด 1"},
        @{@"name": @"รายการที่ 2", @"detail": @"รายละเอียด 2"},
        @{@"name": @"รายการที่ 3", @"detail": @"รายละเอียด 3"}
    ]];
    
    [self setupTableView];
    [self setupNavigationBar];
}

- (void)viewWillAppear:(BOOL)animated {
    [super viewWillAppear:animated];
    [self.tableView deselectRowAtIndexPath:self.tableView.indexPathForSelectedRow 
                                 animated:animated];
}

#pragma mark - Setup

- (void)setupTableView {
    self.tableView = [[UITableView alloc] initWithFrame:self.view.bounds 
                                                  style:UITableViewStyleInsetGrouped];
    self.tableView.autoresizingMask = UIViewAutoresizingFlexibleWidth | 
                                     UIViewAutoresizingFlexibleHeight;
    self.tableView.delegate = self;
    self.tableView.dataSource = self;
    [self.tableView registerClass:[UITableViewCell class] 
           forCellReuseIdentifier:@"Cell"];
    [self.view addSubview:self.tableView];
}

- (void)setupNavigationBar {
    UIBarButtonItem *addButton = [[UIBarButtonItem alloc]
        initWithBarButtonSystemItem:UIBarButtonSystemItemAdd
                             target:self
                             action:@selector(addItem)];
    self.navigationItem.rightBarButtonItem = addButton;
    self.navigationItem.leftBarButtonItem = self.editButtonItem;
}

#pragma mark - UITableViewDataSource

- (NSInteger)tableView:(UITableView *)tableView numberOfRowsInSection:(NSInteger)section {
    return self.items.count;
}

- (UITableViewCell *)tableView:(UITableView *)tableView 
         cellForRowAtIndexPath:(NSIndexPath *)indexPath {
    
    UITableViewCell *cell = [tableView 
        dequeueReusableCellWithIdentifier:@"Cell" 
                             forIndexPath:indexPath];
    
    NSDictionary *item = self.items[indexPath.row];
    cell.textLabel.text = item[@"name"];
    cell.detailTextLabel.text = item[@"detail"];
    cell.accessoryType = UITableViewCellAccessoryDisclosureIndicator;
    
    return cell;
}

- (void)tableView:(UITableView *)tableView 
    commitEditingStyle:(UITableViewCellEditingStyle)editingStyle 
     forRowAtIndexPath:(NSIndexPath *)indexPath {
    
    if (editingStyle == UITableViewCellEditingStyleDelete) {
        [self.items removeObjectAtIndex:indexPath.row];
        [tableView deleteRowsAtIndexPaths:@[indexPath]
                         withRowAnimation:UITableViewRowAnimationFade];
    }
}

#pragma mark - UITableViewDelegate

- (void)tableView:(UITableView *)tableView 
    didSelectRowAtIndexPath:(NSIndexPath *)indexPath {
    
    NSDictionary *item = self.items[indexPath.row];
    
    ItemDetailViewController *detailVC = [[ItemDetailViewController alloc] init];
    detailVC.item = item;
    detailVC.delegate = self;
    detailVC.itemIndex = indexPath.row;
    
    [self.navigationController pushViewController:detailVC animated:YES];
}

#pragma mark - Actions

- (void)addItem {
    UIAlertController *alert = [UIAlertController 
        alertControllerWithTitle:@"เพิ่มรายการ"
                         message:nil
                  preferredStyle:UIAlertControllerStyleAlert];
    
    [alert addTextFieldWithConfigurationHandler:^(UITextField *field) {
        field.placeholder = @"ชื่อรายการ";
    }];
    
    UIAlertAction *addAction = [UIAlertAction 
        actionWithTitle:@"เพิ่ม"
                  style:UIAlertActionStyleDefault
                handler:^(UIAlertAction *action) {
        
        NSString *name = alert.textFields.firstObject.text;
        if (name.length > 0) {
            NSDictionary *newItem = @{@"name": name, @"detail": @"ใหม่"};
            [self.items addObject:newItem];
            
            NSIndexPath *indexPath = [NSIndexPath 
                indexPathForRow:self.items.count - 1 
                      inSection:0];
            [self.tableView insertRowsAtIndexPaths:@[indexPath]
                                  withRowAnimation:UITableViewRowAnimationAutomatic];
        }
    }];
    
    UIAlertAction *cancelAction = [UIAlertAction 
        actionWithTitle:@"ยกเลิก"
                  style:UIAlertActionStyleCancel
                handler:nil];
    
    [alert addAction:addAction];
    [alert addAction:cancelAction];
    [self presentViewController:alert animated:YES completion:nil];
}

#pragma mark - ItemDetailDelegate

- (void)itemDetailViewController:(ItemDetailViewController *)vc
                    didUpdateItem:(NSDictionary *)item
                          atIndex:(NSInteger)index {
    
    self.items[index] = item;
    NSIndexPath *indexPath = [NSIndexPath indexPathForRow:index inSection:0];
    [self.tableView reloadRowsAtIndexPaths:@[indexPath] 
                          withRowAnimation:UITableViewRowAnimationAutomatic];
}

@end
```

---

## 43.14 สรุป

ในตอนนี้เราได้เรียนรู้:

1. **UIViewController Lifecycle** - viewDidLoad, viewWillAppear, viewDidAppear, viewWillDisappear, viewDidDisappear
2. **loadView** - สร้าง view เองทั้งหมด
3. **viewDidLayoutSubviews** - ใช้เมื่อต้องการ frame ที่แน่นอน
4. **Memory Warnings** - จัดการหน่วยความจำ
5. **UINavigationController** - Navigation stack
6. **UITabBarController** - Tab-based navigation
7. **Presenting VCs** - Modal, Popover, Sheet
8. **Dismiss** - ปิด modal VC
9. **Segues** - Interface Builder transitions
10. **Passing Data** - Properties, Delegation, Notification, Singleton, Blocks
11. **Custom Transitions** - Animated transitions
12. **Child VCs** - Container view controllers

---

*ตอนที่ 43 จบแล้ว - ไปต่อที่ตอนที่ 44: UIView Deep Dive*
