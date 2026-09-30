# Part 48: Storyboards และ XIBs

## บทนำ

Storyboard และ XIB (eXternal Interface Builder) เป็นเครื่องมือ visual สำหรับสร้าง UI ใน iOS development โดยใช้ Interface Builder ซึ่งเป็นส่วนหนึ่งของ Xcode

ข้อดีของ Storyboard/XIB:
- เห็น UI ได้ทันทีโดยไม่ต้อง run app
- ลด boilerplate code
- ง่ายสำหรับผู้เริ่มต้น
- เห็น flow ระหว่าง screens

ข้อเสีย:
- Merge conflicts ในทีม
- ยากต่อการ review ใน code review
- บาง customization ต้องทำใน code
- Performance อาจช้ากว่า programmatic

---

## 1. Interface Builder Overview

### 1.1 ส่วนประกอบหลัก

```
Xcode Interface Builder มีส่วนประกอบ:

1. Canvas - พื้นที่วาง UI
2. Document Outline - แสดง hierarchy ของ views
3. Inspector Panel (ขวา):
   - File Inspector - ข้อมูลไฟล์
   - Identity Inspector - Class, Restoration ID
   - Attributes Inspector - คุณสมบัติต่างๆ
   - Size Inspector - ขนาด position constraints
   - Connections Inspector - IBOutlet, IBAction
4. Library - drag components มาวาง
5. Editor Area - ทำงานหลัก
```

### 1.2 การเพิ่ม UI Components

```
1. เปิด Library (+ button หรือ Cmd+Shift+L)
2. ค้นหา component เช่น "Label", "Button", "Text Field"
3. Drag มาวางบน canvas
4. ปรับ position และ size
5. กำหนด constraints สำหรับ Auto Layout
```

---

## 2. Storyboard Basics

### 2.1 สร้าง Storyboard

Storyboard ไฟล์ที่เก็บ View Controllers และ connections ระหว่างกัน

```objc
// โหลด view controller จาก storyboard
UIStoryboard *storyboard = [UIStoryboard storyboardWithName:@"Main" bundle:nil];

// โหลด initial view controller
UIViewController *initialVC = [storyboard instantiateInitialViewController];

// โหลด VC ด้วย identifier (กำหนดใน Identity Inspector > Storyboard ID)
ProfileViewController *profileVC = [storyboard instantiateViewControllerWithIdentifier:@"ProfileViewController"];
```

### 2.2 Initial View Controller

```objc
// AppDelegate.m - ถ้าไม่ใช้ Storyboard entry point
- (BOOL)application:(UIApplication *)application 
didFinishLaunchingWithOptions:(NSDictionary *)launchOptions {
    
    UIStoryboard *storyboard = [UIStoryboard storyboardWithName:@"Main" bundle:nil];
    UIViewController *rootVC = [storyboard instantiateInitialViewController];
    
    self.window = [[UIWindow alloc] initWithFrame:[UIScreen mainScreen].bounds];
    self.window.rootViewController = rootVC;
    [self.window makeKeyAndVisible];
    
    return YES;
}
```

---

## 3. Segues

Segue คือ transition ระหว่าง View Controllers ที่กำหนดใน Storyboard

### 3.1 ประเภท Segues

```
1. Show (Push) - push ใน navigation stack
2. Show Detail - แสดงใน detail column (split view)
3. Present Modally - present เป็น modal
4. Present as Popover - แสดงเป็น popover (iPad)
5. Custom - transition แบบกำหนดเอง
```

### 3.2 การสร้าง Segue ใน Interface Builder

```
1. Ctrl+drag จาก source element ไปยัง destination VC
2. เลือกประเภท segue
3. กำหนด Identifier ใน Attributes Inspector
```

### 3.3 Trigger Segue จาก Code

```objc
// Trigger segue โดยใช้ identifier
[self performSegueWithIdentifier:@"ShowDetail" sender:self];

// ส่ง sender เพื่อใช้ใน prepareForSegue
[self performSegueWithIdentifier:@"ShowProductDetail" sender:self.selectedProduct];
```

### 3.4 prepareForSegue:sender:

```objc
// ส่งข้อมูลไปยัง destination VC
- (void)prepareForSegue:(UIStoryboardSegue *)segue sender:(id)sender {
    
    if ([segue.identifier isEqualToString:@"ShowDetail"]) {
        DetailViewController *detailVC = segue.destinationViewController;
        
        // ส่งข้อมูล
        if ([sender isKindOfClass:[NSIndexPath class]]) {
            NSIndexPath *indexPath = (NSIndexPath *)sender;
            detailVC.item = self.items[indexPath.row];
        }
    }
    
    if ([segue.identifier isEqualToString:@"ShowProfile"]) {
        // ถ้า destination ห่อด้วย nav controller
        UINavigationController *navVC = segue.destinationViewController;
        ProfileViewController *profileVC = navVC.viewControllers.firstObject;
        profileVC.userID = self.currentUser.id;
    }
    
    if ([segue.identifier isEqualToString:@"PresentSettings"]) {
        SettingsViewController *settingsVC = segue.destinationViewController;
        settingsVC.delegate = self;
    }
}

// ยกเลิก segue ได้
- (BOOL)shouldPerformSegueWithIdentifier:(NSString *)identifier sender:(id)sender {
    if ([identifier isEqualToString:@"ShowProfile"] && !self.isLoggedIn) {
        [self showLoginAlert];
        return NO; // ยกเลิก segue
    }
    return YES;
}
```

---

## 4. Unwind Segues

Unwind segue ใช้สำหรับกลับไปหน้าก่อนหน้าใน stack และส่งข้อมูลกลับ

### 4.1 สร้าง Unwind Segue

```objc
// ใน destination VC (ที่จะ unwind ไปถึง) ต้องมี IBAction นี้
- (IBAction)unwindToHome:(UIStoryboardSegue *)segue {
    // รับข้อมูลจาก source
    if ([segue.identifier isEqualToString:@"SaveAndReturn"]) {
        EditViewController *editVC = segue.sourceViewController;
        self.item.name = editVC.nameTextField.text;
        [self.tableView reloadData];
    }
}

// หรือ unwind ด้วย completion block
- (IBAction)unwindFromSettings:(UIStoryboardSegue *)segue {
    SettingsViewController *settingsVC = segue.sourceViewController;
    [self applySettings:settingsVC.currentSettings];
}
```

### 4.2 Connect Unwind ใน Interface Builder

```
1. ใน source VC, Ctrl+drag จาก button ไปยัง "Exit" icon บน top bar
2. เลือก unwind action ที่ต้องการ
3. กำหนด Identifier ใน Attributes Inspector
```

### 4.3 Trigger Unwind จาก Code

```objc
// Unwind programmatically
[self performSegueWithIdentifier:@"UnwindToHome" sender:self];
```

---

## 5. IBOutlet

IBOutlet เชื่อม UI element ใน Interface Builder กับ property ใน code

### 5.1 การสร้าง IBOutlet

```objc
// ViewController.h (ถ้า public)
@interface ProfileViewController : UIViewController
// ไม่จำเป็นต้องประกาศ IBOutlets ที่นี่ถ้าเป็น private
@end

// ViewController.m (private IBOutlets)
@interface ProfileViewController ()
// สร้างโดย Ctrl+drag จาก UI element มาใน code
@property (nonatomic, weak) IBOutlet UIImageView *avatarImageView;
@property (nonatomic, weak) IBOutlet UILabel *nameLabel;
@property (nonatomic, weak) IBOutlet UILabel *emailLabel;
@property (nonatomic, weak) IBOutlet UIButton *editButton;
@property (nonatomic, weak) IBOutlet UITableView *postsTableView;
@property (nonatomic, weak) IBOutlet NSLayoutConstraint *headerHeightConstraint;
@end
```

### 5.2 ใช้ IBOutlet

```objc
- (void)viewDidLoad {
    [super viewDidLoad];
    
    // ตรวจสอบว่า IBOutlet connected ก่อนใช้
    NSAssert(self.nameLabel != nil, @"nameLabel IBOutlet not connected!");
    
    // ใช้งาน
    self.nameLabel.text = self.user.name;
    self.emailLabel.text = self.user.email;
    
    // ปรับ constraint
    self.headerHeightConstraint.constant = 200;
}
```

### 5.3 IBOutlet Collections

```objc
// IBOutletCollection - เชื่อม multiple elements เป็น array
@property (nonatomic, strong) IBOutletCollection(UIButton) NSArray<UIButton *> *socialButtons;
@property (nonatomic, strong) IBOutletCollection(UILabel) NSArray<UILabel *> *statLabels;

// ใช้งาน
- (void)viewDidLoad {
    [super viewDidLoad];
    
    // ปิดทุกปุ่ม
    for (UIButton *btn in self.socialButtons) {
        btn.enabled = NO;
    }
}
```

---

## 6. IBAction

IBAction เชื่อม event จาก UI element กับ method ใน code

### 6.1 การสร้าง IBAction

```objc
// Ctrl+drag จาก button มาใน code หรือ:
- (IBAction)loginButtonTapped:(UIButton *)sender {
    NSLog(@"Login tapped!");
    [self performLogin];
}

- (IBAction)textFieldDidChange:(UITextField *)sender {
    self.loginButton.enabled = sender.text.length > 0;
}

// IBAction สำหรับ multiple events
- (IBAction)sliderValueChanged:(UISlider *)sender {
    self.valueLabel.text = [NSString stringWithFormat:@"%.1f", sender.value];
}
```

### 6.2 ประเภท Events ที่ใช้บ่อย

```objc
// UIButton
- (IBAction)buttonTapped:(id)sender;            // Touch Up Inside

// UITextField  
- (IBAction)textChanged:(UITextField *)sender;    // Editing Changed
- (IBAction)editingDone:(UITextField *)sender;   // Did End on Exit

// UISlider
- (IBAction)sliderChanged:(UISlider *)sender;     // Value Changed

// UISwitch
- (IBAction)switchChanged:(UISwitch *)sender;    // Value Changed

// UISegmentedControl
- (IBAction)segmentChanged:(UISegmentedControl *)sender; // Value Changed

// UIDatePicker
- (IBAction)dateChanged:(UIDatePicker *)sender; // Value Changed
```

### 6.3 Sender Patterns

```objc
// ใช้ tag เพื่อแยก buttons หลายอัน
- (IBAction)categoryButtonTapped:(UIButton *)sender {
    NSInteger tag = sender.tag;
    // Tag กำหนดใน Interface Builder > Attributes Inspector > Tag
    
    switch (tag) {
        case 1: [self showCategory:@"Food"]; break;
        case 2: [self showCategory:@"Tech"]; break;
        case 3: [self showCategory:@"Travel"]; break;
    }
}

// ใช้ IBOutletCollection แทน
- (IBAction)filterButtonTapped:(UIButton *)sender {
    // Deselect others
    for (UIButton *btn in self.filterButtons) {
        btn.selected = NO;
    }
    // Select tapped
    sender.selected = YES;
    
    NSUInteger index = [self.filterButtons indexOfObject:sender];
    [self applyFilter:index];
}
```

---

## 7. NIB/XIB Files

XIB (XML Interface Builder) เป็นไฟล์สำหรับสร้าง views แบบ standalone ไม่ใช่ flow ของ VCs

### 7.1 สร้าง XIB

```
1. File > New > File
2. เลือก View (ใน User Interface section)
3. ตั้งชื่อไฟล์ เช่น ProfileHeaderView.xib
```

### 7.2 โหลด View จาก NIB

```objc
// วิธีที่ 1: โหลดด้วย UINib
UINib *nib = [UINib nibWithNibName:@"ProfileHeaderView" bundle:nil];
NSArray *objects = [nib instantiateWithOwner:self options:nil];
ProfileHeaderView *headerView = objects.firstObject;

// วิธีที่ 2: โหลดด้วย NSBundle
NSArray *views = [[NSBundle mainBundle] loadNibNamed:@"ProfileHeaderView" 
                                               owner:self 
                                             options:nil];
ProfileHeaderView *headerView = views.firstObject;
```

### 7.3 Custom View ใน XIB

```objc
// ProfileHeaderView.h
#import <UIKit/UIKit.h>

@interface ProfileHeaderView : UIView

@property (nonatomic, weak) IBOutlet UIImageView *avatarImageView;
@property (nonatomic, weak) IBOutlet UILabel *nameLabel;
@property (nonatomic, weak) IBOutlet UILabel *bioLabel;
@property (nonatomic, weak) IBOutlet UIButton *followButton;

- (void)configureWithUser:(User *)user;

@end
```

```objc
// ProfileHeaderView.m
#import "ProfileHeaderView.h"

@implementation ProfileHeaderView

// Class method สำหรับสร้าง instance จาก XIB
+ (instancetype)headerView {
    NSArray *views = [[NSBundle mainBundle] loadNibNamed:@"ProfileHeaderView" 
                                                   owner:nil 
                                                 options:nil];
    return views.firstObject;
}

// ถ้า view ถูกสร้างจาก XIB awakeFromNib จะถูกเรียก
- (void)awakeFromNib {
    [super awakeFromNib];
    
    // Setup ที่ต้องทำหลัง load จาก nib
    self.avatarImageView.layer.cornerRadius = 40;
    self.avatarImageView.clipsToBounds = YES;
    
    self.followButton.layer.cornerRadius = 8;
    self.followButton.layer.borderWidth = 1;
    self.followButton.layer.borderColor = [UIColor systemBlueColor].CGColor;
}

- (void)configureWithUser:(User *)user {
    self.nameLabel.text = user.name;
    self.bioLabel.text = user.bio;
    // Load avatar image...
}

- (IBAction)followButtonTapped:(UIButton *)sender {
    // Handle follow
}

@end
```

### 7.4 ใช้ XIB View ใน Code

```objc
// ใน ViewController
- (void)viewDidLoad {
    [super viewDidLoad];
    
    ProfileHeaderView *headerView = [ProfileHeaderView headerView];
    headerView.frame = CGRectMake(0, 0, self.view.bounds.size.width, 180);
    [headerView configureWithUser:self.currentUser];
    
    self.tableView.tableHeaderView = headerView;
}
```

### 7.5 NIB สำหรับ UITableViewCell

```objc
// ContactCell.xib - สร้าง XIB สำหรับ cell

// ContactCell.h
@interface ContactCell : UITableViewCell
@property (nonatomic, weak) IBOutlet UIImageView *avatarImageView;
@property (nonatomic, weak) IBOutlet UILabel *nameLabel;
@property (nonatomic, weak) IBOutlet UILabel *phoneLabel;
- (void)configureWithContact:(Contact *)contact;
@end

// Register ใน ViewController
UINib *cellNib = [UINib nibWithNibName:@"ContactCell" bundle:nil];
[self.tableView registerNib:cellNib forCellReuseIdentifier:@"ContactCell"];
```

---

## 8. Auto Layout ใน Interface Builder

### 8.1 การเพิ่ม Constraints

```
1. Pin button (ด้านล่างขวา) - กำหนดระยะห่างจากขอบ
2. Align button - จัดตำแหน่ง (center, leading edge ฯลฯ)
3. Ctrl+drag ระหว่าง views
4. Resolve Auto Layout Issues - แก้ปัญหา
```

### 8.2 IBOutlet สำหรับ Constraint

```objc
@interface AnimatingViewController ()
@property (nonatomic, weak) IBOutlet NSLayoutConstraint *topConstraint;
@property (nonatomic, weak) IBOutlet NSLayoutConstraint *heightConstraint;
@end

@implementation AnimatingViewController

- (void)animateHeader {
    [UIView animateWithDuration:0.3 animations:^{
        self.heightConstraint.constant = 100; // ลดความสูง
        self.topConstraint.constant = 0;
        [self.view layoutIfNeeded]; // สำคัญมาก!
    }];
}

@end
```

### 8.3 Size Classes

```
Size Classes ใน Interface Builder:
- Compact Width (wC) - iPhone portrait
- Regular Width (wR) - iPad หรือ iPhone landscape
- Compact Height (hC) - iPhone landscape
- Regular Height (hR) - iPad

สามารถกำหนด constraints ต่างกันตาม size class
```

---

## 9. Storyboard References

Storyboard Reference ช่วยแบ่ง storyboard ใหญ่เป็นไฟล์เล็กๆ

### 9.1 สร้าง Reference

```
1. ใน storyboard หลัก เพิ่ม Storyboard Reference object
2. กำหนด Referenced ID (storyboard ชื่ออะไร)
3. กำหนด Referenced ID = Storyboard ID ของ VC ใน storyboard นั้น
4. Ctrl+drag segue มาที่ reference
```

### 9.2 โหลด Storyboard Reference

```objc
// โหลด VC จาก storyboard อื่น
UIStoryboard *settingsStoryboard = [UIStoryboard storyboardWithName:@"Settings" bundle:nil];
SettingsViewController *settingsVC = [settingsStoryboard instantiateInitialViewController];
[self.navigationController pushViewController:settingsVC animated:YES];
```

---

## 10. Custom Segues

### 10.1 สร้าง Custom Segue

```objc
// SlideUpSegue.h
@interface SlideUpSegue : UIStoryboardSegue
@end

// SlideUpSegue.m
#import "SlideUpSegue.h"

@implementation SlideUpSegue

- (void)perform {
    UIViewController *sourceVC = self.sourceViewController;
    UIViewController *destVC = self.destinationViewController;
    UIView *destView = destVC.view;
    
    // เตรียม position เริ่มต้น (อยู่ด้านล่างจอ)
    CGRect screenBounds = [UIScreen mainScreen].bounds;
    destView.frame = CGRectOffset(screenBounds, 0, screenBounds.size.height);
    
    // เพิ่ม dest view
    UIWindow *window = sourceVC.view.window;
    [window addSubview:destView];
    
    // Animation
    [UIView animateWithDuration:0.4 
                          delay:0 
         usingSpringWithDamping:0.8 
          initialSpringVelocity:0.5 
                        options:UIViewAnimationOptionCurveEaseOut 
                     animations:^{
        destView.frame = screenBounds;
    } completion:^(BOOL finished) {
        [sourceVC presentViewController:destVC animated:NO completion:nil];
    }];
}

@end
```

### 10.2 Custom Unwind Segue

```objc
// SlideDownUnwindSegue.m
@implementation SlideDownUnwindSegue

- (void)perform {
    UIViewController *sourceVC = self.sourceViewController;
    UIViewController *destVC = self.destinationViewController;
    
    [UIView animateWithDuration:0.3 animations:^{
        CGRect frame = sourceVC.view.frame;
        frame.origin.y = [UIScreen mainScreen].bounds.size.height;
        sourceVC.view.frame = frame;
    } completion:^(BOOL finished) {
        [destVC dismissViewControllerAnimated:NO completion:nil];
    }];
}

@end
```

---

## 11. ตัวอย่างสมบูรณ์: Login Flow

### 11.1 Storyboard Structure

```
Main Storyboard:
├── Initial VC: MainTabBarController
└── Login Storyboard Reference

Login Storyboard:
├── Initial VC: LoginViewController
│   └── [Show] → SignUpViewController
│   └── [Unwind] ← MainTabBarController.unwindFromLogin
└── SignUpViewController
    └── [Unwind] ← MainTabBarController.unwindFromLogin
```

### 11.2 LoginViewController

```objc
// LoginViewController.h
@interface LoginViewController : UIViewController
@end

// LoginViewController.m
@interface LoginViewController ()
@property (nonatomic, weak) IBOutlet UITextField *emailTextField;
@property (nonatomic, weak) IBOutlet UITextField *passwordTextField;
@property (nonatomic, weak) IBOutlet UIButton *loginButton;
@property (nonatomic, weak) IBOutlet UIActivityIndicatorView *loadingIndicator;
@property (nonatomic, weak) IBOutlet UILabel *errorLabel;
@end

@implementation LoginViewController

- (void)viewDidLoad {
    [super viewDidLoad];
    [self setupUI];
}

- (void)setupUI {
    self.title = @"เข้าสู่ระบบ";
    
    // Style buttons
    self.loginButton.layer.cornerRadius = 12;
    self.loginButton.enabled = NO;
    
    // Hide error label
    self.errorLabel.hidden = YES;
    self.loadingIndicator.hidesWhenStopped = YES;
    
    // Text field delegates
    self.emailTextField.delegate = self;
    self.passwordTextField.delegate = self;
}

- (IBAction)emailChanged:(UITextField *)sender {
    [self validateInputs];
}

- (IBAction)passwordChanged:(UITextField *)sender {
    [self validateInputs];
}

- (void)validateInputs {
    BOOL hasEmail = self.emailTextField.text.length > 0;
    BOOL hasPassword = self.passwordTextField.text.length >= 6;
    self.loginButton.enabled = hasEmail && hasPassword;
    self.loginButton.alpha = (hasEmail && hasPassword) ? 1.0 : 0.5;
}

- (IBAction)loginButtonTapped:(UIButton *)sender {
    [self.view endEditing:YES];
    
    NSString *email = self.emailTextField.text;
    NSString *password = self.passwordTextField.text;
    
    [self setLoading:YES];
    
    [[AuthService shared] loginWithEmail:email 
                                password:password 
                              completion:^(User *user, NSError *error) {
        dispatch_async(dispatch_get_main_queue(), ^{
            [self setLoading:NO];
            
            if (error) {
                [self showError:error.localizedDescription];
            } else {
                [self performSegueWithIdentifier:@"LoginSuccess" sender:user];
            }
        });
    }];
}

- (IBAction)signUpButtonTapped:(UIButton *)sender {
    [self performSegueWithIdentifier:@"ShowSignUp" sender:nil];
}

- (IBAction)forgotPasswordTapped:(UIButton *)sender {
    UIAlertController *alert = [UIAlertController 
        alertControllerWithTitle:@"ลืมรหัสผ่าน"
                         message:@"กรอก Email เพื่อรับลิงก์ reset"
                  preferredStyle:UIAlertControllerStyleAlert];
    
    [alert addTextFieldWithConfigurationHandler:^(UITextField *textField) {
        textField.placeholder = @"Email";
        textField.keyboardType = UIKeyboardTypeEmailAddress;
        textField.text = self.emailTextField.text;
    }];
    
    [alert addAction:[UIAlertAction actionWithTitle:@"ส่ง" 
                                             style:UIAlertActionStyleDefault 
                                           handler:^(UIAlertAction *action) {
        NSString *email = alert.textFields.firstObject.text;
        [[AuthService shared] resetPasswordForEmail:email completion:^(NSError *error) {
            if (!error) {
                [self showMessage:@"ส่ง Email แล้ว กรุณาตรวจสอบกล่องจดหมาย"];
            }
        }];
    }]];
    
    [alert addAction:[UIAlertAction actionWithTitle:@"ยกเลิก" 
                                             style:UIAlertActionStyleCancel 
                                           handler:nil]];
    
    [self presentViewController:alert animated:YES completion:nil];
}

- (void)setLoading:(BOOL)loading {
    if (loading) {
        [self.loadingIndicator startAnimating];
        self.loginButton.enabled = NO;
    } else {
        [self.loadingIndicator stopAnimating];
        [self validateInputs];
    }
}

- (void)showError:(NSString *)message {
    self.errorLabel.text = message;
    self.errorLabel.hidden = NO;
    
    // Shake animation
    CAKeyframeAnimation *shake = [CAKeyframeAnimation animationWithKeyPath:@"transform.translation.x"];
    shake.timingFunction = [CAMediaTimingFunction functionWithName:kCAMediaTimingFunctionLinear];
    shake.duration = 0.4;
    shake.values = @[@(-10), @(10), @(-8), @(8), @(-5), @(5), @(0)];
    [self.emailTextField.layer addAnimation:shake forKey:@"shake"];
}

- (void)showMessage:(NSString *)message {
    UIAlertController *alert = [UIAlertController alertControllerWithTitle:nil
                                                                   message:message
                                                            preferredStyle:UIAlertControllerStyleAlert];
    [alert addAction:[UIAlertAction actionWithTitle:@"OK" style:UIAlertActionStyleDefault handler:nil]];
    [self presentViewController:alert animated:YES completion:nil];
}

- (void)prepareForSegue:(UIStoryboardSegue *)segue sender:(id)sender {
    if ([segue.identifier isEqualToString:@"LoginSuccess"]) {
        // ไม่ต้องส่งข้อมูลเพราะ unwind จะ dismiss login
    }
}

// UITextFieldDelegate
- (BOOL)textFieldShouldReturn:(UITextField *)textField {
    if (textField == self.emailTextField) {
        [self.passwordTextField becomeFirstResponder];
    } else {
        [textField resignFirstResponder];
        if (self.loginButton.enabled) {
            [self loginButtonTapped:self.loginButton];
        }
    }
    return YES;
}

@end
```

### 11.3 MainTabBarController รับ Unwind

```objc
// MainTabBarController.m

@implementation MainTabBarController

// ต้องมี IBAction นี้เพื่อรับ unwind จาก Login
- (IBAction)unwindFromLogin:(UIStoryboardSegue *)segue {
    // Login สำเร็จ - ไม่ต้องทำอะไร tab bar จะแสดงเอง
    NSLog(@"Logged in successfully");
    
    if ([segue.sourceViewController isKindOfClass:[LoginViewController class]]) {
        // อัปเดต UI ถ้าจำเป็น
        [[self viewControllers] enumerateObjectsUsingBlock:^(UIViewController *vc, NSUInteger idx, BOOL *stop) {
            // Reload data ใน each tab
        }];
    }
}

@end
```

---

## 12. Storyboard vs Programmatic UI

### 12.1 เปรียบเทียบ

| | Storyboard/XIB | Programmatic |
|---|---|---|
| Visual feedback | ทันที | ต้อง run |
| ความเร็ว development | เร็วกว่าสำหรับ simple UI | ช้ากว่าแต่ flexible กว่า |
| Code review | ยาก (XML) | ง่าย |
| Merge conflicts | บ่อย | น้อย |
| Reuse | ยาก | ง่าย |
| Dynamic UI | จำกัด | ไม่จำกัด |
| Performance | เล็กน้อยช้ากว่า | เร็วกว่า |

### 12.2 Hybrid Approach

```objc
// แนวทางที่นิยม: ใช้ Storyboard สำหรับ flow
// แต่สร้าง views ที่ซับซ้อนด้วย code

// ViewController สร้างจาก Storyboard
@interface ProductViewController : UIViewController
// IBOutlets สำหรับ simple elements
@property (nonatomic, weak) IBOutlet UIScrollView *scrollView;
@property (nonatomic, weak) IBOutlet UIView *containerView;
@end

@implementation ProductViewController

- (void)viewDidLoad {
    [super viewDidLoad];
    
    // สร้าง complex custom views ด้วย code
    [self setupImageCarousel];
    [self setupRatingView];
    [self setupReviewsSection];
}

- (void)setupImageCarousel {
    // Custom view ที่ยากทำใน IB
    ImageCarouselView *carousel = [[ImageCarouselView alloc] init];
    carousel.translatesAutoresizingMaskIntoConstraints = NO;
    [self.containerView addSubview:carousel];
    // Add constraints...
}

@end
```

---

## 13. Dynamic Type และ Accessibility ใน IB

### 13.1 Dynamic Type

```objc
// ใน Interface Builder
// Font > System Font หรือเลือก Text Style เช่น Title, Headline, Body, Caption
// ติ๊ก "Automatically Adjusts Font"

// ใน Code (ถ้า programmatic)
self.titleLabel.font = [UIFont preferredFontForTextStyle:UIFontTextStyleTitle1];
self.titleLabel.adjustsFontForContentSizeCategory = YES;
```

### 13.2 Accessibility

```objc
// กำหนดใน Attributes Inspector > Accessibility
// หรือใน code:

self.avatarImageView.isAccessibilityElement = YES;
self.avatarImageView.accessibilityLabel = @"รูปโปรไฟล์";
self.avatarImageView.accessibilityHint = @"แตะเพื่อเปลี่ยนรูป";
self.avatarImageView.accessibilityTraits = UIAccessibilityTraitImage | UIAccessibilityTraitButton;
```

---

## 14. Preview ใน Interface Builder

### 14.1 Device Preview

```
1. Show Assistant Editor
2. เลือก Preview
3. เลือก device ที่ต้องการ preview
4. เพิ่มหลาย device ได้พร้อมกัน
```

### 14.2 Live Preview สำหรับ Custom Views

```objc
// เพิ่ม IB_DESIGNABLE ให้ custom view เพื่อให้เห็นใน IB
IB_DESIGNABLE
@interface GradientButton : UIButton

@property (nonatomic, strong) IBInspectable UIColor *startColor;
@property (nonatomic, strong) IBInspectable UIColor *endColor;
@property (nonatomic, assign) IBInspectable CGFloat cornerRadius;

@end

// GradientButton.m
IB_DESIGNABLE
@implementation GradientButton

- (void)prepareForInterfaceBuilder {
    [super prepareForInterfaceBuilder];
    // ตั้งค่าเริ่มต้นสำหรับแสดงใน IB
    if (!self.startColor) self.startColor = [UIColor systemBlueColor];
    if (!self.endColor) self.endColor = [UIColor systemPurpleColor];
    [self setupGradient];
}

- (void)layoutSubviews {
    [super layoutSubviews];
    [self setupGradient];
}

- (void)setupGradient {
    // ลบ gradient เก่า
    [self.layer.sublayers enumerateObjectsUsingBlock:^(CALayer *layer, NSUInteger idx, BOOL *stop) {
        if ([layer isKindOfClass:[CAGradientLayer class]]) {
            [layer removeFromSuperlayer];
        }
    }];
    
    CAGradientLayer *gradient = [CAGradientLayer layer];
    gradient.frame = self.bounds;
    gradient.colors = @[
        (__bridge id)self.startColor.CGColor,
        (__bridge id)self.endColor.CGColor
    ];
    gradient.startPoint = CGPointMake(0, 0.5);
    gradient.endPoint = CGPointMake(1, 0.5);
    gradient.cornerRadius = self.cornerRadius;
    
    [self.layer insertSublayer:gradient atIndex:0];
    self.layer.cornerRadius = self.cornerRadius;
    self.layer.masksToBounds = YES;
}

@end
```

---

## 15. Localization ใน Interface Builder

### 15.1 Base Localization

```
1. Project settings > Info > Localizations
2. เพิ่มภาษา เช่น Thai
3. ใน Storyboard/XIB file inspector > Localize
4. เลือก "Localizable Strings" เพื่อสร้าง strings file

Main.strings (Thai):
"homeTitle.text" = "หน้าหลัก";
"loginButton.normalTitle" = "เข้าสู่ระบบ";
```

### 15.2 ใน Code

```objc
// แทนที่จะ hardcode:
self.titleLabel.text = NSLocalizedString(@"welcome_title", @"Welcome screen title");

// Localizable.strings (Base/English):
"welcome_title" = "Welcome";

// Localizable.strings (Thai):
"welcome_title" = "ยินดีต้อนรับ";
```

---

## 16. Traits และ Adaptive UI

### 16.1 Trait-based Constraints

```objc
// Override ใน ViewController เพื่อตอบสนองต่อ trait changes
- (void)traitCollectionDidChange:(UITraitCollection *)previousTraitCollection {
    [super traitCollectionDidChange:previousTraitCollection];
    
    if (self.traitCollection.horizontalSizeClass == UIUserInterfaceSizeClassRegular) {
        // iPad landscape หรือ iPhone Plus landscape
        self.columnsLayout.itemsPerRow = 3;
    } else {
        // iPhone portrait
        self.columnsLayout.itemsPerRow = 2;
    }
}
```

### 16.2 Varied Constraints ใน IB

```
ใน Size Inspector ของ constraint:
- กด "+" ถัดจาก Constant
- เลือก Size Class เช่น "wCompact hAny"
- กำหนด constant ต่างกันสำหรับ size class นั้น
```

---

## 17. แบบฝึกหัด (Practice Exercises)

### Exercise 1: Registration Form
สร้าง registration form ใน Storyboard ที่มี:
- Name, Email, Password, Confirm Password fields
- Date of Birth picker
- Terms & Conditions switch
- Register button (disabled จนกว่าจะกรอกครบ)
- IBOutlets และ IBActions ครบถ้วน

```objc
// Template
@interface RegisterViewController : UIViewController

@property (nonatomic, weak) IBOutlet UITextField *nameTextField;
@property (nonatomic, weak) IBOutlet UITextField *emailTextField;
@property (nonatomic, weak) IBOutlet UITextField *passwordTextField;
@property (nonatomic, weak) IBOutlet UITextField *confirmTextField;
@property (nonatomic, weak) IBOutlet UIDatePicker *dobPicker;
@property (nonatomic, weak) IBOutlet UISwitch *termsSwitch;
@property (nonatomic, weak) IBOutlet UIButton *registerButton;

- (IBAction)fieldChanged:(UITextField *)sender;
- (IBAction)termsChanged:(UISwitch *)sender;
- (IBAction)registerTapped:(UIButton *)sender;

@end
```

### Exercise 2: Settings Screen ด้วย XIB
สร้าง settings cell ด้วย XIB ที่มี:
- Icon image view
- Title label
- Subtitle label
- Toggle switch หรือ chevron
- Custom IBInspectable properties

### Exercise 3: Onboarding Flow ด้วย Storyboard
สร้าง onboarding 4 หน้าใน Storyboard:
- ใช้ Container View + Page View Controller
- แต่ละหน้ามี image, title, description
- Skip button และ Next/Get Started button
- Page control ที่ sync กับ swipe

### Exercise 4: Custom Segue
สร้าง custom segue:
- Zoom in animation เมื่อ push
- Zoom out เมื่อ pop
- ใช้กับ table view ที่ tap ไปหน้า detail

### Exercise 5: IB_DESIGNABLE Card View
สร้าง card view ที่:
- IBInspectable: cornerRadius, shadowRadius, shadowOpacity, cardColor
- เห็นผลทันทีใน Interface Builder
- มี elevation effect
- Reusable ได้ทุก screen

```objc
IB_DESIGNABLE
@interface CardView : UIView
@property (nonatomic, assign) IBInspectable CGFloat cornerRadius;
@property (nonatomic, assign) IBInspectable CGFloat shadowRadius;
@property (nonatomic, assign) IBInspectable CGFloat shadowOpacity;
@property (nonatomic, strong) IBInspectable UIColor *shadowColor;
@end
```

---

## 18. Best Practices

### 18.1 Organization

```
ควรแบ่ง storyboard ตาม feature:
- Main.storyboard - Tab bar, initial flow
- Auth.storyboard - Login, Register, Reset password
- Settings.storyboard - Settings screens
- Onboarding.storyboard - First launch screens
```

### 18.2 Avoid Massive Storyboards

```objc
// ใช้ Storyboard References แบ่ง storyboard ใหญ่
// แต่ละ feature เป็น storyboard ของตัวเอง
// ทำให้ทำงานเป็นทีมได้ง่ายขึ้น (ลด merge conflicts)
```

### 18.3 Reuse Components

```objc
// สร้าง XIB สำหรับ components ที่ใช้บ่อย
// เช่น EmptyStateView, LoadingView, ErrorView
// แทนที่จะ duplicate ใน Storyboard

@interface EmptyStateView : UIView
+ (instancetype)emptyStateWithTitle:(NSString *)title 
                            message:(NSString *)message 
                              image:(UIImage *)image 
                         buttonText:(NSString *)buttonText 
                             action:(void(^)(void))action;
@end
```

### 18.4 Programmatic Constraints vs IB

```objc
// IBOutlet สำหรับ constraints ที่ต้องแก้ใน code
@property (nonatomic, weak) IBOutlet NSLayoutConstraint *bottomConstraint;

// เพิ่ม keyboard avoidance
- (void)keyboardWillShow:(NSNotification *)notification {
    CGFloat keyboardHeight = [notification.userInfo[UIKeyboardFrameEndUserInfoKey] CGRectValue].size.height;
    self.bottomConstraint.constant = keyboardHeight + 20;
    [UIView animateWithDuration:0.3 animations:^{
        [self.view layoutIfNeeded];
    }];
}
```

---

## 19. Migration: Storyboard ไป Programmatic

หากต้องการย้ายจาก Storyboard ไป Programmatic UI:

```objc
// 1. สร้าง VC แบบ programmatic
@implementation ProductViewController

- (instancetype)initWithProduct:(Product *)product {
    self = [super init]; // ไม่ใช้ initWithNibName:bundle:
    if (self) {
        _product = product;
    }
    return self;
}

- (void)loadView {
    // สร้าง view hierarchy เอง
    UIView *rootView = [[UIView alloc] init];
    rootView.backgroundColor = [UIColor systemBackgroundColor];
    
    self.titleLabel = [[UILabel alloc] init];
    // ... setup UI
    
    [rootView addSubview:self.titleLabel];
    self.view = rootView;
}

- (void)viewDidLoad {
    [super viewDidLoad];
    [self setupConstraints];
    [self configureWithProduct:self.product];
}

@end

// 2. ลบ Storyboard ID และ segue
// 3. ใน caller สร้าง VC โดยตรง
ProductViewController *productVC = [[ProductViewController alloc] initWithProduct:product];
[self.navigationController pushViewController:productVC animated:YES];
```

---

## สรุป

ในบทนี้เราได้เรียนรู้:

1. **Interface Builder** - ส่วนประกอบและการใช้งาน
2. **Storyboard Basics** - การสร้างและจัดการ
3. **Segues** - push, modal, custom transitions
4. **Unwind Segues** - กลับหน้าพร้อมส่งข้อมูล
5. **prepareForSegue:sender:** - ส่งข้อมูลระหว่าง VCs
6. **IBOutlet** - เชื่อม UI elements กับ code
7. **IBAction** - จัดการ events
8. **NIB/XIB** - standalone views
9. **Auto Layout ใน IB** - constraints, size classes
10. **Storyboard References** - แบ่ง storyboard ใหญ่
11. **IB_DESIGNABLE** - preview custom views ใน IB
12. **Best Practices** - การจัดระเบียบและ reuse

ทั้ง Storyboard และ Programmatic UI มีข้อดีและข้อเสียของตัวเอง การเลือกใช้ขึ้นอยู่กับ:
- ขนาดทีม (ทีมใหญ่ = programmatic ดีกว่า)
- ความซับซ้อนของ UI
- ความต้องการ reuse
- ความถนัดของ developer

ในปัจจุบัน trend เอนไปทาง programmatic UI มากขึ้น โดยเฉพาะกับการมาของ SwiftUI แต่ Storyboard/XIB ยังคงมีประโยชน์สำหรับ rapid prototyping และ teams ที่ไม่ต้องการ merge conflicts บ่อย
