# Part 47: Navigation Patterns

## บทนำ

Navigation เป็นส่วนสำคัญที่สุดของ iOS app ที่กำหนดว่าผู้ใช้เดินทางระหว่างหน้าต่างๆ อย่างไร iOS มี navigation patterns หลักหลายอย่าง:

- **UINavigationController** - Stack navigation (push/pop)
- **UITabBarController** - Tab-based navigation
- **UIPageViewController** - Swipe between pages
- **UISplitViewController** - Master/detail (iPad)
- **Modal presentations** - Present over current content

การเลือก pattern ที่เหมาะสมทำให้ app ใช้งานได้ง่ายและสอดคล้องกับ iOS conventions

---

## 1. UINavigationController

### 1.1 พื้นฐาน

UINavigationController จัดการ stack ของ view controllers และแสดง navigation bar ด้านบน

```objc
// AppDelegate.m - ตั้งค่า root navigation controller
- (BOOL)application:(UIApplication *)application 
didFinishLaunchingWithOptions:(NSDictionary *)launchOptions {
    
    self.window = [[UIWindow alloc] initWithFrame:[UIScreen mainScreen].bounds];
    
    HomeViewController *homeVC = [[HomeViewController alloc] init];
    UINavigationController *navController = [[UINavigationController alloc] 
                                            initWithRootViewController:homeVC];
    
    self.window.rootViewController = navController;
    [self.window makeKeyAndVisible];
    
    return YES;
}
```

### 1.2 Stack Operations

```objc
// Push - เพิ่ม VC บน stack
DetailViewController *detailVC = [[DetailViewController alloc] initWithItem:item];
[self.navigationController pushViewController:detailVC animated:YES];

// Pop - ลบ VC ออกจาก stack (กลับหน้าก่อน)
[self.navigationController popViewControllerAnimated:YES];

// Pop ไปหน้าแรก
[self.navigationController popToRootViewControllerAnimated:YES];

// Pop ไปหน้าที่ระบุ
NSArray *stack = self.navigationController.viewControllers;
UIViewController *targetVC = stack[1]; // second VC in stack
[self.navigationController popToViewController:targetVC animated:YES];

// ดู stack ปัจจุบัน
UIViewController *topVC = self.navigationController.topViewController;
UIViewController *visibleVC = self.navigationController.visibleViewController;
NSArray<UIViewController *> *allVCs = self.navigationController.viewControllers;
```

### 1.3 การ Set Stack โดยตรง

```objc
// Replace ทั้ง stack
UIViewController *vc1 = [[ViewController1 alloc] init];
UIViewController *vc2 = [[ViewController2 alloc] init];
UIViewController *vc3 = [[ViewController3 alloc] init];

[self.navigationController setViewControllers:@[vc1, vc2, vc3] animated:YES];
// ผู้ใช้จะเห็น vc3 และมี back stack เป็น vc2 -> vc1
```

---

## 2. Navigation Bar Customization

### 2.1 Title และ Subtitle

```objc
// Title ธรรมดา
self.title = @"หน้าหลัก";

// หรือกำหนดผ่าน navigationItem
self.navigationItem.title = @"หน้าหลัก";

// Title view แบบ custom
UILabel *titleLabel = [[UILabel alloc] init];
titleLabel.text = @"Custom Title";
titleLabel.font = [UIFont boldSystemFontOfSize:17];
titleLabel.textColor = [UIColor whiteColor];
self.navigationItem.titleView = titleLabel;

// Large title (iOS 11+)
self.navigationItem.largeTitleDisplayMode = UINavigationItemLargeTitleDisplayModeAlways;
self.navigationController.navigationBar.prefersLargeTitles = YES;
```

### 2.2 Bar Buttons

```objc
// Right button - ข้อความ
UIBarButtonItem *editBtn = [[UIBarButtonItem alloc] 
                           initWithTitle:@"แก้ไข" 
                                   style:UIBarButtonItemStylePlain 
                                  target:self 
                                  action:@selector(editTapped)];
self.navigationItem.rightBarButtonItem = editBtn;

// Right button - ไอคอน
UIBarButtonItem *addBtn = [[UIBarButtonItem alloc] 
                          initWithBarButtonSystemItem:UIBarButtonSystemItemAdd 
                                              target:self 
                                              action:@selector(addTapped)];

// ปุ่มหลายอัน
self.navigationItem.rightBarButtonItems = @[editBtn, addBtn];

// Left button
self.navigationItem.leftBarButtonItem = [[UIBarButtonItem alloc]
                                        initWithTitle:@"ยกเลิก"
                                                style:UIBarButtonItemStylePlain
                                               target:self
                                               action:@selector(cancelTapped)];

// Custom view button
UIButton *customBtn = [UIButton buttonWithType:UIButtonTypeCustom];
[customBtn setTitle:@"Custom" forState:UIControlStateNormal];
customBtn.frame = CGRectMake(0, 0, 80, 32);
UIBarButtonItem *customBarBtn = [[UIBarButtonItem alloc] initWithCustomView:customBtn];
self.navigationItem.rightBarButtonItem = customBarBtn;
```

### 2.3 Appearance (iOS 15+)

```objc
// กำหนด appearance ของ navigation bar
UINavigationBarAppearance *appearance = [[UINavigationBarAppearance alloc] init];
[appearance configureWithOpaqueBackground];
appearance.backgroundColor = [UIColor systemBlueColor];
appearance.titleTextAttributes = @{
    NSForegroundColorAttributeName: [UIColor whiteColor],
    NSFontAttributeName: [UIFont boldSystemFontOfSize:18]
};
appearance.largeTitleTextAttributes = @{
    NSForegroundColorAttributeName: [UIColor whiteColor],
    NSFontAttributeName: [UIFont boldSystemFontOfSize:34]
};

self.navigationController.navigationBar.standardAppearance = appearance;
self.navigationController.navigationBar.scrollEdgeAppearance = appearance;
self.navigationController.navigationBar.compactAppearance = appearance;

// Tint color (สำหรับ buttons และ icons)
self.navigationController.navigationBar.tintColor = [UIColor whiteColor];
```

### 2.4 Navigation Bar แบบ Transparent

```objc
UINavigationBarAppearance *transparentAppearance = [[UINavigationBarAppearance alloc] init];
[transparentAppearance configureWithTransparentBackground];
transparentAppearance.titleTextAttributes = @{
    NSForegroundColorAttributeName: [UIColor whiteColor]
};

self.navigationController.navigationBar.standardAppearance = transparentAppearance;
self.navigationController.navigationBar.scrollEdgeAppearance = transparentAppearance;
```

---

## 3. Back Button Customization

### 3.1 เปลี่ยน Text

```objc
// ใน view controller ก่อนหน้า (parent)
// เมื่อ push ไปหน้าถัดไป back button จะแสดง title นี้
self.navigationItem.backButtonTitle = @"กลับ";

// หรือใช้ backBarButtonItem
self.navigationItem.backBarButtonItem = [[UIBarButtonItem alloc] 
                                        initWithTitle:@"กลับ" 
                                                style:UIBarButtonItemStylePlain 
                                               target:nil 
                                               action:nil];
```

### 3.2 Custom Back Button

```objc
// ซ่อน default back button
self.navigationItem.hidesBackButton = YES;

// สร้าง custom back button
UIBarButtonItem *backBtn = [[UIBarButtonItem alloc] 
                           initWithImage:[UIImage systemImageNamed:@"chevron.left"]
                                   style:UIBarButtonItemStylePlain
                                  target:self
                                  action:@selector(goBack)];
self.navigationItem.leftBarButtonItem = backBtn;

- (void)goBack {
    [self.navigationController popViewControllerAnimated:YES];
}
```

### 3.3 Back Button Display Mode (iOS 14+)

```objc
// แสดงแค่ chevron ไม่มี title
self.navigationItem.backButtonDisplayMode = UINavigationItemBackButtonDisplayModeMinimal;

// แสดง title เต็ม
self.navigationItem.backButtonDisplayMode = UINavigationItemBackButtonDisplayModeDefault;
```

---

## 4. UINavigationControllerDelegate

```objc
@interface AnimatedNavVC () <UINavigationControllerDelegate>
@end

@implementation AnimatedNavVC

- (void)viewDidLoad {
    [super viewDidLoad];
    self.navigationController.delegate = self;
}

// เมื่อกำลังจะ show VC
- (void)navigationController:(UINavigationController *)navigationController 
      willShowViewController:(UIViewController *)viewController 
                    animated:(BOOL)animated {
    NSLog(@"Will show: %@", viewController);
}

// เมื่อ show VC แล้ว
- (void)navigationController:(UINavigationController *)navigationController 
       didShowViewController:(UIViewController *)viewController 
                    animated:(BOOL)animated {
    NSLog(@"Did show: %@", viewController);
}

// Custom transition animation
- (id<UIViewControllerAnimatedTransitioning>)navigationController:(UINavigationController *)navigationController 
                                  animationControllerForOperation:(UINavigationControllerOperation)operation 
                                               fromViewController:(UIViewController *)fromVC 
                                                 toViewController:(UIViewController *)toVC {
    if (operation == UINavigationControllerOperationPush) {
        return [[SlideInAnimator alloc] init];
    }
    return nil; // nil = default animation
}

@end
```

---

## 5. UITabBarController

### 5.1 การตั้งค่า

```objc
// AppDelegate.m
- (BOOL)application:(UIApplication *)application 
didFinishLaunchingWithOptions:(NSDictionary *)launchOptions {
    
    self.window = [[UIWindow alloc] initWithFrame:[UIScreen mainScreen].bounds];
    
    // สร้าง tab controllers
    HomeViewController *homeVC = [[HomeViewController alloc] init];
    homeVC.tabBarItem = [[UITabBarItem alloc] initWithTitle:@"หน้าหลัก" 
                                                      image:[UIImage systemImageNamed:@"house"]
                                               selectedImage:[UIImage systemImageNamed:@"house.fill"]];
    
    SearchViewController *searchVC = [[SearchViewController alloc] init];
    searchVC.tabBarItem = [[UITabBarItem alloc] initWithTitle:@"ค้นหา"
                                                        image:[UIImage systemImageNamed:@"magnifyingglass"]
                                                 selectedImage:nil];
    
    ProfileViewController *profileVC = [[ProfileViewController alloc] init];
    profileVC.tabBarItem = [[UITabBarItem alloc] initWithTabBarSystemItem:UITabBarSystemItemMore 
                                                                      tag:2];
    
    // ห่อด้วย navigation controllers
    UINavigationController *homeNav = [[UINavigationController alloc] initWithRootViewController:homeVC];
    UINavigationController *searchNav = [[UINavigationController alloc] initWithRootViewController:searchVC];
    UINavigationController *profileNav = [[UINavigationController alloc] initWithRootViewController:profileVC];
    
    // สร้าง tab bar controller
    UITabBarController *tabBarController = [[UITabBarController alloc] init];
    tabBarController.viewControllers = @[homeNav, searchNav, profileNav];
    tabBarController.selectedIndex = 0; // tab เริ่มต้น
    
    self.window.rootViewController = tabBarController;
    [self.window makeKeyAndVisible];
    
    return YES;
}
```

### 5.2 Badge

```objc
// เพิ่ม badge
self.tabBarItem.badgeValue = @"5";

// ลบ badge
self.tabBarItem.badgeValue = nil;

// Badge สีแดง (default) หรือสีอื่น
self.tabBarItem.badgeColor = [UIColor systemRedColor];

// Badge สำหรับ notification dot (ไม่มีตัวเลข)
self.tabBarItem.badgeValue = @"";
```

### 5.3 Tab Bar Appearance

```objc
// iOS 15+
UITabBarAppearance *appearance = [[UITabBarAppearance alloc] init];
[appearance configureWithOpaqueBackground];
appearance.backgroundColor = [UIColor systemBackgroundColor];

// Item colors
UITabBarItemAppearance *itemAppearance = [[UITabBarItemAppearance alloc] init];
itemAppearance.normal.iconColor = [UIColor systemGrayColor];
itemAppearance.normal.titleTextAttributes = @{
    NSForegroundColorAttributeName: [UIColor systemGrayColor]
};
itemAppearance.selected.iconColor = [UIColor systemBlueColor];
itemAppearance.selected.titleTextAttributes = @{
    NSForegroundColorAttributeName: [UIColor systemBlueColor]
};

appearance.stackedLayoutAppearance = itemAppearance;

UITabBar.appearance.standardAppearance = appearance;
UITabBar.appearance.scrollEdgeAppearance = appearance; // iOS 15+
```

### 5.4 UITabBarControllerDelegate

```objc
@interface MainTabBarVC () <UITabBarControllerDelegate>
@end

@implementation MainTabBarVC

- (void)viewDidLoad {
    [super viewDidLoad];
    self.delegate = self;
}

// ก่อนเปลี่ยน tab (return NO เพื่อยกเลิก)
- (BOOL)tabBarController:(UITabBarController *)tabBarController 
shouldSelectViewController:(UIViewController *)viewController {
    // ตรวจสอบว่า login แล้วหรือยัง
    if (viewController == self.viewControllers[2] && !self.isLoggedIn) {
        [self showLoginVC];
        return NO;
    }
    return YES;
}

// หลังเปลี่ยน tab
- (void)tabBarController:(UITabBarController *)tabBarController 
 didSelectViewController:(UIViewController *)viewController {
    NSLog(@"Selected tab: %ld", (long)tabBarController.selectedIndex);
}

@end
```

---

## 6. Combined Navigation + Tabs

Pattern ที่พบบ่อยที่สุดใน iOS apps

```objc
// MainTabBarController.m

@implementation MainTabBarController

- (void)viewDidLoad {
    [super viewDidLoad];
    [self setupTabs];
}

- (void)setupTabs {
    // Tab 1: Feed
    FeedViewController *feedVC = [[FeedViewController alloc] init];
    feedVC.title = @"Feed";
    UINavigationController *feedNav = [[UINavigationController alloc] 
                                       initWithRootViewController:feedVC];
    feedNav.tabBarItem = [[UITabBarItem alloc] 
                         initWithTitle:@"Feed" 
                                 image:[UIImage systemImageNamed:@"rectangle.stack"]
                          selectedImage:[UIImage systemImageNamed:@"rectangle.stack.fill"]];
    
    // Tab 2: Explore
    ExploreViewController *exploreVC = [[ExploreViewController alloc] init];
    exploreVC.title = @"Explore";
    UINavigationController *exploreNav = [[UINavigationController alloc] 
                                          initWithRootViewController:exploreVC];
    exploreNav.tabBarItem = [[UITabBarItem alloc] 
                            initWithTitle:@"Explore" 
                                    image:[UIImage systemImageNamed:@"safari"]
                             selectedImage:[UIImage systemImageNamed:@"safari.fill"]];
    
    // Tab 3: Notifications
    NotificationsViewController *notifVC = [[NotificationsViewController alloc] init];
    notifVC.title = @"Notifications";
    UINavigationController *notifNav = [[UINavigationController alloc] 
                                        initWithRootViewController:notifVC];
    notifNav.tabBarItem = [[UITabBarItem alloc] 
                          initWithTitle:@"Notifications" 
                                  image:[UIImage systemImageNamed:@"bell"]
                           selectedImage:[UIImage systemImageNamed:@"bell.fill"]];
    
    // Tab 4: Profile
    ProfileViewController *profileVC = [[ProfileViewController alloc] init];
    profileVC.title = @"Profile";
    UINavigationController *profileNav = [[UINavigationController alloc] 
                                          initWithRootViewController:profileVC];
    profileNav.tabBarItem = [[UITabBarItem alloc] 
                            initWithTitle:@"Profile" 
                                    image:[UIImage systemImageNamed:@"person"]
                             selectedImage:[UIImage systemImageNamed:@"person.fill"]];
    
    self.viewControllers = @[feedNav, exploreNav, notifNav, profileNav];
}

// ไปที่ tab โดยใช้ index
- (void)switchToTab:(NSInteger)index {
    self.selectedIndex = index;
}

// ไปที่ tab + push VC
- (void)navigateToFeedAndShowDetail:(id)item {
    self.selectedIndex = 0;
    UINavigationController *feedNav = self.viewControllers[0];
    DetailViewController *detailVC = [[DetailViewController alloc] initWithItem:item];
    [feedNav pushViewController:detailVC animated:NO]; // NO เพราะเพิ่งเปลี่ยน tab
}

@end
```

---

## 7. UIPageViewController

### 7.1 การตั้งค่า

```objc
// OnboardingViewController.m

@interface OnboardingViewController () <UIPageViewControllerDataSource, UIPageViewControllerDelegate>
@property (nonatomic, strong) UIPageViewController *pageViewController;
@property (nonatomic, strong) NSArray<UIViewController *> *pages;
@property (nonatomic, strong) UIPageControl *pageControl;
@end

@implementation OnboardingViewController

- (void)viewDidLoad {
    [super viewDidLoad];
    
    [self setupPages];
    [self setupPageViewController];
    [self setupPageControl];
}

- (void)setupPages {
    OnboardPage1VC *page1 = [[OnboardPage1VC alloc] init];
    OnboardPage2VC *page2 = [[OnboardPage2VC alloc] init];
    OnboardPage3VC *page3 = [[OnboardPage3VC alloc] init];
    self.pages = @[page1, page2, page3];
}

- (void)setupPageViewController {
    self.pageViewController = [[UIPageViewController alloc] 
        initWithTransitionStyle:UIPageViewControllerTransitionStyleScroll
                  navigationOrientation:UIPageViewControllerNavigationOrientationHorizontal
                                options:@{
                                    UIPageViewControllerOptionSpineLocationKey: @(UIPageViewControllerSpineLocationMin),
                                    UIPageViewControllerOptionInterPageSpacingKey: @(20)
                                }];
    
    self.pageViewController.dataSource = self;
    self.pageViewController.delegate = self;
    
    // กำหนดหน้าแรก
    [self.pageViewController setViewControllers:@[self.pages[0]] 
                                      direction:UIPageViewControllerNavigationDirectionForward 
                                       animated:NO 
                                     completion:nil];
    
    // เพิ่มเป็น child VC
    [self addChildViewController:self.pageViewController];
    self.pageViewController.view.frame = self.view.bounds;
    [self.view addSubview:self.pageViewController.view];
    [self.pageViewController didMoveToParentViewController:self];
}

- (void)setupPageControl {
    self.pageControl = [[UIPageControl alloc] init];
    self.pageControl.translatesAutoresizingMaskIntoConstraints = NO;
    self.pageControl.numberOfPages = self.pages.count;
    self.pageControl.currentPage = 0;
    self.pageControl.pageIndicatorTintColor = [UIColor systemGray4Color];
    self.pageControl.currentPageIndicatorTintColor = [UIColor systemBlueColor];
    [self.view addSubview:self.pageControl];
    
    [NSLayoutConstraint activateConstraints:@[
        [self.pageControl.centerXAnchor constraintEqualToAnchor:self.view.centerXAnchor],
        [self.pageControl.bottomAnchor constraintEqualToAnchor:self.view.safeAreaLayoutGuide.bottomAnchor constant:-20],
    ]];
}

// DataSource
- (UIViewController *)pageViewController:(UIPageViewController *)pageViewController 
       viewControllerBeforeViewController:(UIViewController *)viewController {
    NSInteger index = [self.pages indexOfObject:viewController];
    if (index == 0 || index == NSNotFound) return nil;
    return self.pages[index - 1];
}

- (UIViewController *)pageViewController:(UIPageViewController *)pageViewController 
        viewControllerAfterViewController:(UIViewController *)viewController {
    NSInteger index = [self.pages indexOfObject:viewController];
    if (index == self.pages.count - 1 || index == NSNotFound) return nil;
    return self.pages[index + 1];
}

// Delegate
- (void)pageViewController:(UIPageViewController *)pageViewController 
        didFinishAnimating:(BOOL)finished 
   previousViewControllers:(NSArray<UIViewController *> *)previousViewControllers 
       transitionCompleted:(BOOL)completed {
    if (completed) {
        UIViewController *currentVC = pageViewController.viewControllers.firstObject;
        self.pageControl.currentPage = [self.pages indexOfObject:currentVC];
    }
}

// Navigate programmatically
- (void)goToPage:(NSInteger)index {
    UIViewController *targetVC = self.pages[index];
    UIViewController *currentVC = self.pageViewController.viewControllers.firstObject;
    NSInteger currentIndex = [self.pages indexOfObject:currentVC];
    
    UIPageViewControllerNavigationDirection direction = 
        index > currentIndex ? UIPageViewControllerNavigationDirectionForward 
                             : UIPageViewControllerNavigationDirectionReverse;
    
    [self.pageViewController setViewControllers:@[targetVC]
                                       direction:direction
                                        animated:YES
                                      completion:nil];
    self.pageControl.currentPage = index;
}

@end
```

---

## 8. UISplitViewController (iPad)

### 8.1 การตั้งค่า

```objc
// AppDelegate.m
- (BOOL)application:(UIApplication *)application 
didFinishLaunchingWithOptions:(NSDictionary *)launchOptions {
    
    self.window = [[UIWindow alloc] initWithFrame:[UIScreen mainScreen].bounds];
    
    // Master (sidebar)
    ContactsListViewController *masterVC = [[ContactsListViewController alloc] init];
    UINavigationController *masterNav = [[UINavigationController alloc] 
                                        initWithRootViewController:masterVC];
    
    // Detail (main content)
    ContactDetailViewController *detailVC = [[ContactDetailViewController alloc] init];
    UINavigationController *detailNav = [[UINavigationController alloc] 
                                        initWithRootViewController:detailVC];
    
    // Split view
    UISplitViewController *splitVC = [[UISplitViewController alloc] init];
    splitVC.viewControllers = @[masterNav, detailNav];
    splitVC.delegate = self;
    splitVC.preferredDisplayMode = UISplitViewControllerDisplayModeOneBesideSecondary;
    splitVC.primaryColumnMinimumWidth = 250;
    splitVC.primaryColumnMaximumWidth = 350;
    
    self.window.rootViewController = splitVC;
    [self.window makeKeyAndVisible];
    
    return YES;
}
```

### 8.2 การ Navigate จาก Master ไป Detail

```objc
// ContactsListViewController.m
- (void)tableView:(UITableView *)tableView didSelectRowAtIndexPath:(NSIndexPath *)indexPath {
    Contact *contact = self.contacts[indexPath.row];
    
    // สร้าง detail VC
    ContactDetailViewController *detailVC = [[ContactDetailViewController alloc] 
                                            initWithContact:contact];
    
    // แสดงใน detail column
    UINavigationController *detailNav = [[UINavigationController alloc] 
                                        initWithRootViewController:detailVC];
    
    // ใช้ showDetailViewController สำหรับ split view
    [self showDetailViewController:detailNav sender:self];
}
```

### 8.3 Triple Column Layout (iOS 14+)

```objc
// Modern split view controller
UISplitViewController *splitVC = [[UISplitViewController alloc] initWithStyle:UISplitViewControllerStyleTripleColumn];

// Columns
[splitVC setViewController:sidebarVC forColumn:UISplitViewControllerColumnPrimary];
[splitVC setViewController:masterVC forColumn:UISplitViewControllerColumnSupplementary];
[splitVC setViewController:detailVC forColumn:UISplitViewControllerColumnSecondary];
```

---

## 9. Modal Presentations

### 9.1 Present Modal

```objc
// Present แบบ default (sheet - iOS 13+)
SettingsViewController *settingsVC = [[SettingsViewController alloc] init];
UINavigationController *settingsNav = [[UINavigationController alloc] 
                                       initWithRootViewController:settingsVC];

[self presentViewController:settingsNav animated:YES completion:nil];

// Dismiss
[self dismissViewControllerAnimated:YES completion:nil];

// หรือจาก parent
[self.presentingViewController dismissViewControllerAnimated:YES completion:nil];
```

### 9.2 Presentation Styles

```objc
ModalVC *modalVC = [[ModalVC alloc] init];
UINavigationController *nav = [[UINavigationController alloc] initWithRootViewController:modalVC];

// Full screen
nav.modalPresentationStyle = UIModalPresentationFullScreen;

// Form sheet (iPad center, iPhone sheet)
nav.modalPresentationStyle = UIModalPresentationFormSheet;

// Page sheet (iOS 13+ ค่า default)
nav.modalPresentationStyle = UIModalPresentationPageSheet;

// Over full screen (เห็น VC ด้านหลัง)
nav.modalPresentationStyle = UIModalPresentationOverFullScreen;

// Current context
nav.modalPresentationStyle = UIModalPresentationCurrentContext;

[self presentViewController:nav animated:YES completion:nil];
```

### 9.3 Sheet Presentation (iOS 15+)

```objc
ModalVC *modalVC = [[ModalVC alloc] init];

// Sheet detent
UISheetPresentationController *sheetController = modalVC.sheetPresentationController;
if (sheetController) {
    // กำหนดขนาด sheet
    sheetController.detents = @[
        UISheetPresentationControllerDetent.mediumDetent,   // ~50% screen
        UISheetPresentationControllerDetent.largeDetent     // full screen
    ];
    
    // เริ่มที่ medium
    sheetController.selectedDetentIdentifier = UISheetPresentationControllerDetentIdentifierMedium;
    
    // แสดง grabber
    sheetController.prefersGrabberVisible = YES;
    
    // ไม่ dim background (สำหรับ medium detent)
    sheetController.largestUndimmedDetentIdentifier = UISheetPresentationControllerDetentIdentifierMedium;
    
    // Corner radius
    sheetController.preferredCornerRadius = 20;
    
    // ไม่อนุญาต swipe to dismiss
    sheetController.prefersScrollingExpandsWhenScrolledToEdge = NO;
}

[self presentViewController:modalVC animated:YES completion:nil];
```

### 9.4 Dismissal Callbacks

```objc
// ใน presenting VC
[self presentViewController:modalVC animated:YES completion:^{
    NSLog(@"Modal presented");
}];

[self dismissViewControllerAnimated:YES completion:^{
    NSLog(@"Modal dismissed");
    [self refreshData]; // อัปเดตข้อมูลหลัง dismiss
}];
```

### 9.5 Passing Data ไป Modal

```objc
// Delegation pattern
@protocol ModalDelegate <NSObject>
- (void)modalDidFinishWithResult:(id)result;
@end

@interface ModalVC : UIViewController
@property (nonatomic, weak) id<ModalDelegate> delegate;
@end

// ใน Modal VC
- (void)saveButtonTapped {
    [self.delegate modalDidFinishWithResult:self.result];
    [self dismissViewControllerAnimated:YES completion:nil];
}

// ใน presenting VC
ModalVC *modalVC = [[ModalVC alloc] init];
modalVC.delegate = self;
[self presentViewController:modalVC animated:YES completion:nil];

// Implement delegate
- (void)modalDidFinishWithResult:(id)result {
    // ใช้ result
}
```

---

## 10. Custom Transitions

### 10.1 Animated Transitioning

```objc
// FadeAnimator.h
@interface FadeAnimator : NSObject <UIViewControllerAnimatedTransitioning>
@property (nonatomic, assign) BOOL isPresenting;
@end

// FadeAnimator.m
@implementation FadeAnimator

- (NSTimeInterval)transitionDuration:(id<UIViewControllerContextTransitioning>)transitionContext {
    return 0.3;
}

- (void)animateTransition:(id<UIViewControllerContextTransitioning>)transitionContext {
    UIViewController *toVC = [transitionContext viewControllerForKey:UITransitionContextToViewControllerKey];
    UIViewController *fromVC = [transitionContext viewControllerForKey:UITransitionContextFromViewControllerKey];
    UIView *containerView = [transitionContext containerView];
    
    if (self.isPresenting) {
        toVC.view.alpha = 0;
        [containerView addSubview:toVC.view];
        
        [UIView animateWithDuration:[self transitionDuration:transitionContext] 
                         animations:^{
            toVC.view.alpha = 1;
        } completion:^(BOOL finished) {
            [transitionContext completeTransition:!transitionContext.transitionWasCancelled];
        }];
    } else {
        [UIView animateWithDuration:[self transitionDuration:transitionContext] 
                         animations:^{
            fromVC.view.alpha = 0;
        } completion:^(BOOL finished) {
            [fromVC.view removeFromSuperview];
            [transitionContext completeTransition:!transitionContext.transitionWasCancelled];
        }];
    }
}

@end
```

### 10.2 Transitioning Delegate

```objc
// ModalWithCustomTransition.m
@interface CustomModalVC () <UIViewControllerTransitioningDelegate>
@end

@implementation CustomModalVC

- (instancetype)init {
    self = [super init];
    if (self) {
        self.modalPresentationStyle = UIModalPresentationCustom;
        self.transitioningDelegate = self;
    }
    return self;
}

// Presentation animator
- (id<UIViewControllerAnimatedTransitioning>)animationControllerForPresentedController:(UIViewController *)presented
                                                                 presentingController:(UIViewController *)presenting 
                                                                     sourceController:(UIViewController *)source {
    FadeAnimator *animator = [[FadeAnimator alloc] init];
    animator.isPresenting = YES;
    return animator;
}

// Dismissal animator
- (id<UIViewControllerAnimatedTransitioning>)animationControllerForDismissedController:(UIViewController *)dismissed {
    FadeAnimator *animator = [[FadeAnimator alloc] init];
    animator.isPresenting = NO;
    return animator;
}

@end
```

### 10.3 Interactive Transition

```objc
// SlideDownInteraction.h
@interface SlideDownInteraction : UIPercentDrivenInteractiveTransition
- (void)wireToViewController:(UIViewController *)viewController;
@end

// SlideDownInteraction.m
@interface SlideDownInteraction ()
@property (nonatomic, assign) BOOL shouldComplete;
@property (nonatomic, weak) UIViewController *presentedVC;
@end

@implementation SlideDownInteraction

- (void)wireToViewController:(UIViewController *)viewController {
    self.presentedVC = viewController;
    UIPanGestureRecognizer *pan = [[UIPanGestureRecognizer alloc] 
                                  initWithTarget:self 
                                          action:@selector(handlePan:)];
    [viewController.view addGestureRecognizer:pan];
}

- (void)handlePan:(UIPanGestureRecognizer *)gesture {
    CGPoint translation = [gesture translationInView:gesture.view.superview];
    
    switch (gesture.state) {
        case UIGestureRecognizerStateBegan:
            self.shouldComplete = NO;
            [self.presentedVC dismissViewControllerAnimated:YES completion:nil];
            break;
            
        case UIGestureRecognizerStateChanged: {
            CGFloat progress = MAX(0, MIN(1, translation.y / gesture.view.bounds.size.height));
            self.shouldComplete = progress > 0.5;
            [self updateInteractiveTransition:progress];
            break;
        }
            
        case UIGestureRecognizerStateEnded:
        case UIGestureRecognizerStateCancelled:
            if (self.shouldComplete) {
                [self finishInteractiveTransition];
            } else {
                [self cancelInteractiveTransition];
            }
            break;
            
        default:
            break;
    }
}

@end
```

---

## 11. Deep Linking

### 11.1 URL Scheme

```objc
// AppDelegate.m
- (BOOL)application:(UIApplication *)application 
            openURL:(NSURL *)url 
            options:(NSDictionary<UIApplicationOpenURLOptionsKey, id> *)options {
    
    // myapp://profile/123
    // myapp://feed
    // myapp://settings/notifications
    
    return [self handleURL:url];
}

- (BOOL)handleURL:(NSURL *)url {
    if (![url.scheme isEqualToString:@"myapp"]) return NO;
    
    NSString *host = url.host; // path component after scheme
    NSArray *pathComponents = url.pathComponents;
    
    UITabBarController *tabBar = (UITabBarController *)self.window.rootViewController;
    
    if ([host isEqualToString:@"profile"]) {
        NSString *userID = pathComponents.count > 1 ? pathComponents[1] : nil;
        [self navigateToProfile:userID tabBar:tabBar];
        return YES;
    }
    
    if ([host isEqualToString:@"feed"]) {
        tabBar.selectedIndex = 0;
        return YES;
    }
    
    if ([host isEqualToString:@"settings"]) {
        [self navigateToSettings:pathComponents tabBar:tabBar];
        return YES;
    }
    
    return NO;
}

- (void)navigateToProfile:(NSString *)userID tabBar:(UITabBarController *)tabBar {
    // ไปที่ profile tab
    tabBar.selectedIndex = 3;
    
    UINavigationController *profileNav = tabBar.viewControllers[3];
    [profileNav popToRootViewControllerAnimated:NO];
    
    if (userID) {
        ProfileViewController *profileVC = [[ProfileViewController alloc] initWithUserID:userID];
        [profileNav pushViewController:profileVC animated:NO];
    }
}
```

### 11.2 Universal Links / NSUserActivity

```objc
// AppDelegate.m
- (BOOL)application:(UIApplication *)application 
continueUserActivity:(NSUserActivity *)userActivity 
 restorationHandler:(void (^)(NSArray<id<UIUserActivityRestoring>> *))restorationHandler {
    
    if ([userActivity.activityType isEqualToString:NSUserActivityTypeBrowsingWeb]) {
        NSURL *url = userActivity.webpageURL;
        // https://myapp.com/products/123
        
        return [self handleUniversalLinkURL:url];
    }
    
    return NO;
}

- (BOOL)handleUniversalLinkURL:(NSURL *)url {
    NSURLComponents *components = [NSURLComponents componentsWithURL:url resolvingAgainstBaseURL:YES];
    NSArray *pathComponents = components.path.pathComponents;
    
    // /products/123 -> ["", "products", "123"]
    if (pathComponents.count >= 3 && [pathComponents[1] isEqualToString:@"products"]) {
        NSString *productID = pathComponents[2];
        [self navigateToProduct:productID];
        return YES;
    }
    
    return NO;
}
```

---

## 12. Navigation State Restoration

```objc
// AppDelegate.m
- (void)application:(UIApplication *)application 
willFinishLaunchingWithOptions:(NSDictionary *)launchOptions {
    // Enable state restoration
    application.delegate = self;
}

// ViewController.m
- (void)viewDidLoad {
    [super viewDidLoad];
    // กำหนด restoration identifier
    self.restorationIdentifier = @"HomeViewController";
    self.navigationController.restorationIdentifier = @"MainNavigationController";
}

// Encode state
- (void)encodeRestorableStateWithCoder:(NSCoder *)coder {
    [super encodeRestorableStateWithCoder:coder];
    [coder encodeObject:self.selectedItem forKey:@"selectedItem"];
}

// Decode state
- (void)decodeRestorableStateWithCoder:(NSCoder *)coder {
    [super decodeRestorableStateWithCoder:coder];
    self.selectedItem = [coder decodeObjectForKey:@"selectedItem"];
}
```

---

## 13. Coordinator Pattern

Pattern ที่นิยมสำหรับจัดการ navigation

```objc
// Coordinator.h
@protocol Coordinator <NSObject>
@property (nonatomic, strong) NSMutableArray *childCoordinators;
@property (nonatomic, strong) UINavigationController *navigationController;
- (void)start;
@end

// AppCoordinator.h
@interface AppCoordinator : NSObject <Coordinator>
- (instancetype)initWithNavigationController:(UINavigationController *)navController;
@end

// AppCoordinator.m
@implementation AppCoordinator

- (instancetype)initWithNavigationController:(UINavigationController *)navController {
    self = [super init];
    if (self) {
        _navigationController = navController;
        _childCoordinators = [NSMutableArray array];
    }
    return self;
}

- (void)start {
    [self showHome];
}

- (void)showHome {
    HomeCoordinator *homeCoord = [[HomeCoordinator alloc] 
                                 initWithNavigationController:self.navigationController];
    homeCoord.parentCoordinator = self;
    [self.childCoordinators addObject:homeCoord];
    [homeCoord start];
}

- (void)showLogin {
    LoginCoordinator *loginCoord = [[LoginCoordinator alloc] 
                                   initWithNavigationController:self.navigationController];
    loginCoord.parentCoordinator = self;
    [self.childCoordinators addObject:loginCoord];
    [loginCoord start];
}

- (void)childCoordinatorDidFinish:(id<Coordinator>)coordinator {
    [self.childCoordinators removeObject:coordinator];
}

@end
```

---

## 14. แบบฝึกหัด (Practice Exercises)

### Exercise 1: Master-Detail App
สร้าง app ที่ทำงานทั้ง iPhone (push navigation) และ iPad (split view):
- News list + News detail
- Responsive layout

### Exercise 2: Onboarding Flow
สร้าง onboarding 3 หน้าด้วย UIPageViewController:
- Swipe ระหว่างหน้า
- Page control
- Skip button
- Get Started button ในหน้าสุดท้าย

### Exercise 3: Custom Sheet
สร้าง bottom sheet ที่:
- สามารถ drag ขึ้น/ลงได้
- มี 3 ขนาด (quarter/half/full)
- Dimmed background
- Tap outside to dismiss

### Exercise 4: Tab Bar with Badge
สร้าง app ที่มี tab bar:
- Tab 1: List ที่แสดง badge count
- Tab 2: Add item form
- เมื่อ add item ให้อัปเดต badge ใน Tab 1

### Exercise 5: Deep Link Handler
สร้าง URL scheme handler ที่รองรับ:
- `myapp://home`
- `myapp://profile/{id}`
- `myapp://product/{id}?color=red`
- Navigate ไปหน้าที่ถูกต้องพร้อมส่งข้อมูล

---

## สรุป

ในบทนี้เราได้เรียนรู้:

1. **UINavigationController** - Stack-based navigation, push/pop
2. **Navigation Bar** - Customization ทั้ง appearance และ buttons
3. **Back Button** - การ customize back button
4. **UITabBarController** - Tab-based navigation
5. **Tab Bar** - Badges, appearance
6. **UIPageViewController** - Swipe between pages
7. **UISplitViewController** - iPad master/detail
8. **Modal** - Presentation styles, sheets
9. **Custom Transitions** - Animated/interactive
10. **Deep Linking** - URL schemes, Universal Links
11. **Coordinator Pattern** - การจัดการ navigation logic

การออกแบบ navigation ที่ดีทำให้ app ใช้งานได้ง่ายและสอดคล้องกับ iOS conventions ควรเลือก pattern ที่เหมาะสมกับ content และ use case ของ app
