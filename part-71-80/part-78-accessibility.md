# Part 78: iOS Accessibility ใน Objective-C

## บทนำ

Accessibility หรือ "การเข้าถึง" คือการออกแบบแอปให้ผู้ใช้ทุกคนสามารถใช้งานได้ รวมถึงผู้ที่มีความบกพร่องทางการมองเห็น การได้ยิน การเคลื่อนไหว หรือการรับรู้

Apple มีเป้าหมายชัดเจนว่า "Accessibility is a human right" และได้สร้างระบบ Accessibility ที่ครบครันสำหรับ iOS

บทนี้จะครอบคลุม:
- **UIAccessibility Protocol** - พื้นฐานของ Accessibility ใน iOS
- **VoiceOver** - Screen Reader สำหรับผู้บกพร่องทางสายตา
- **Dynamic Type** - การปรับขนาดตัวอักษรอัตโนมัติ
- **UIAccessibilityTraits** - คุณสมบัติพิเศษสำหรับ UI Elements
- **Accessibility Containers** - การจัดกลุ่ม Elements
- **Custom Actions** - การเพิ่ม Actions พิเศษ
- **Switch Control** - สำหรับผู้ใช้ Motor Impairment
- **Color Accessibility** - การออกแบบสีที่เข้าถึงได้

---

# หมวดที่ 1: UIAccessibility Protocol

## 78.1 ทำความเข้าใจ UIAccessibility

**UIAccessibility** เป็น Informal Protocol ที่ NSObject รองรับ (ผ่าน UIAccessibility.h)

Properties หลักที่ต้องรู้:
- `isAccessibilityElement` - กำหนดว่า Element นี้ควรถูก VoiceOver อ่านหรือไม่
- `accessibilityLabel` - ชื่อของ Element ที่ VoiceOver จะอ่าน
- `accessibilityHint` - คำอธิบายว่าจะเกิดอะไรเมื่อกด
- `accessibilityValue` - ค่าปัจจุบันของ Element (เช่น Slider value)
- `accessibilityTraits` - คุณสมบัติพิเศษ (Button, Header, Link, etc.)
- `accessibilityFrame` - พื้นที่ที่ VoiceOver จะ Focus

```objc
// BasicAccessibilityViewController.m
#import <UIKit/UIKit.h>

@interface BasicAccessibilityViewController : UIViewController
@property (nonatomic, strong) UIButton *loginButton;
@property (nonatomic, strong) UILabel *statusLabel;
@property (nonatomic, strong) UITextField *emailField;
@property (nonatomic, strong) UITextField *passwordField;
@property (nonatomic, strong) UISwitch *rememberMeSwitch;
@property (nonatomic, strong) UIImageView *profileImage;
@end

@implementation BasicAccessibilityViewController

- (void)viewDidLoad {
    [super viewDidLoad];
    [self setupUI];
    [self configureAccessibility];
}

- (void)setupUI {
    self.view.backgroundColor = [UIColor systemBackgroundColor];
    
    // Email Field
    self.emailField = [[UITextField alloc] initWithFrame:CGRectMake(20, 100, 320, 44)];
    self.emailField.borderStyle = UITextBorderStyleRoundedRect;
    self.emailField.placeholder = @"อีเมล";
    self.emailField.keyboardType = UIKeyboardTypeEmailAddress;
    [self.view addSubview:self.emailField];
    
    // Password Field
    self.passwordField = [[UITextField alloc] initWithFrame:CGRectMake(20, 160, 320, 44)];
    self.passwordField.borderStyle = UITextBorderStyleRoundedRect;
    self.passwordField.placeholder = @"รหัสผ่าน";
    self.passwordField.secureTextEntry = YES;
    [self.view addSubview:self.passwordField];
    
    // Remember Me Switch
    self.rememberMeSwitch = [[UISwitch alloc] initWithFrame:CGRectMake(20, 230, 51, 31)];
    [self.view addSubview:self.rememberMeSwitch];
    
    // Login Button
    self.loginButton = [UIButton buttonWithType:UIButtonTypeSystem];
    self.loginButton.frame = CGRectMake(20, 280, 320, 50);
    [self.loginButton setTitle:@"เข้าสู่ระบบ" forState:UIControlStateNormal];
    self.loginButton.backgroundColor = [UIColor systemBlueColor];
    [self.loginButton setTitleColor:[UIColor whiteColor] forState:UIControlStateNormal];
    self.loginButton.layer.cornerRadius = 8;
    [self.loginButton addTarget:self action:@selector(loginTapped) forControlEvents:UIControlEventTouchUpInside];
    [self.view addSubview:self.loginButton];
    
    // Status Label
    self.statusLabel = [[UILabel alloc] initWithFrame:CGRectMake(20, 350, 320, 44)];
    self.statusLabel.textAlignment = NSTextAlignmentCenter;
    self.statusLabel.textColor = [UIColor systemGrayColor];
    [self.view addSubview:self.statusLabel];
    
    // Profile Image
    self.profileImage = [[UIImageView alloc] initWithFrame:CGRectMake(140, 20, 80, 80)];
    self.profileImage.backgroundColor = [UIColor systemGray5Color];
    self.profileImage.layer.cornerRadius = 40;
    self.profileImage.clipsToBounds = YES;
    self.profileImage.image = [UIImage systemImageNamed:@"person.circle.fill"];
    [self.view addSubview:self.profileImage];
}

- (void)configureAccessibility {
    // 1. Profile Image
    // UIImageView ไม่ใช่ Accessibility Element โดยค่าเริ่มต้น
    // แต่ถ้ามีข้อมูลสำคัญควรเปิดใช้งาน
    self.profileImage.isAccessibilityElement = YES;
    self.profileImage.accessibilityLabel = @"รูปโปรไฟล์";
    self.profileImage.accessibilityHint = @"แตะสองครั้งเพื่อเปลี่ยนรูปโปรไฟล์";
    self.profileImage.accessibilityTraits = UIAccessibilityTraitButton | UIAccessibilityTraitImage;
    
    // 2. Email TextField
    // UITextField มี accessibilityLabel = placeholder โดยอัตโนมัติ
    // แต่เราสามารถตั้งค่าเองเพื่อให้ชัดเจนขึ้น
    self.emailField.accessibilityLabel = @"ที่อยู่อีเมล";
    self.emailField.accessibilityHint = @"ป้อนอีเมลที่ใช้ในการเข้าสู่ระบบ";
    // ไม่ต้องตั้ง accessibilityValue เพราะ UITextField อัปเดตให้อัตโนมัติ
    
    // 3. Password TextField
    self.passwordField.accessibilityLabel = @"รหัสผ่าน";
    self.passwordField.accessibilityHint = @"ป้อนรหัสผ่านของคุณ อย่างน้อย 8 ตัวอักษร";
    // secureTextEntry = YES จะทำให้ VoiceOver แจ้งว่าเป็น secure field
    
    // 4. Remember Me Switch
    self.rememberMeSwitch.accessibilityLabel = @"จดจำการเข้าสู่ระบบ";
    self.rememberMeSwitch.accessibilityHint = @"เปิดเพื่อให้แอปจดจำการเข้าสู่ระบบในครั้งถัดไป";
    // UISwitch จะอัปเดต accessibilityValue ว่า "เปิด" หรือ "ปิด" อัตโนมัติ
    
    // 5. Login Button
    self.loginButton.accessibilityLabel = @"เข้าสู่ระบบ";
    self.loginButton.accessibilityHint = @"แตะเพื่อเข้าสู่ระบบด้วยอีเมลและรหัสผ่านที่กรอก";
    self.loginButton.accessibilityTraits = UIAccessibilityTraitButton;
    
    // 6. Status Label
    // Label ที่แสดงผลลัพธ์ควรเป็น Accessibility Element
    self.statusLabel.isAccessibilityElement = YES;
    self.statusLabel.accessibilityLabel = @"สถานะการเข้าสู่ระบบ";
    // accessibilityValue จะถูก VoiceOver อ่านแยกจาก Label
}

- (void)loginTapped {
    // อัปเดต Status พร้อมแจ้ง VoiceOver
    self.statusLabel.text = @"กำลังเข้าสู่ระบบ...";
    
    // แจ้ง VoiceOver ให้อ่าน Announcement ทันที
    UIAccessibilityPostNotification(UIAccessibilityAnnouncementNotification, 
                                    @"กำลังตรวจสอบข้อมูลการเข้าสู่ระบบ");
    
    // จำลองการ Login
    dispatch_after(dispatch_time(DISPATCH_TIME_NOW, 2 * NSEC_PER_SEC), 
                   dispatch_get_main_queue(), ^{
        self.statusLabel.text = @"เข้าสู่ระบบสำเร็จ";
        
        // แจ้ง VoiceOver ว่าหน้าจอเปลี่ยนแปลง
        UIAccessibilityPostNotification(UIAccessibilityScreenChangedNotification, 
                                        self.statusLabel);
    });
}

@end
```

## 78.2 accessibilityLabel vs accessibilityHint vs accessibilityValue

```objc
// AccessibilityPropertiesDemo.m
@implementation AccessibilityPropertiesDemo

- (void)demonstrateProperties {
    // accessibilityLabel: ชื่อสั้นๆ ที่บอก Element ว่าคืออะไร
    // VoiceOver อ่าน: "เพิ่มในรายการโปรด"
    UIButton *favoriteButton = [UIButton buttonWithType:UIButtonTypeSystem];
    favoriteButton.accessibilityLabel = @"เพิ่มในรายการโปรด";
    
    // accessibilityHint: คำอธิบายผลที่จะเกิดขึ้น ใช้กริยา
    // VoiceOver อ่าน: "แตะสองครั้งเพื่อเพิ่มรายการนี้ในรายการโปรดของคุณ"
    favoriteButton.accessibilityHint = @"เพิ่มรายการนี้ในรายการโปรดของคุณ";
    
    // accessibilityValue: ค่าปัจจุบัน
    // ใช้สำหรับ Slider, Progress Bar, Toggle เป็นต้น
    UISlider *volumeSlider = [[UISlider alloc] init];
    volumeSlider.accessibilityLabel = @"ระดับเสียง";
    // accessibilityValue จะถูกอัปเดตเมื่อ Slider เปลี่ยน
    // คุณสามารถ Override ได้:
    // volumeSlider.accessibilityValue = [NSString stringWithFormat:@"%.0f เปอร์เซ็นต์", volumeSlider.value * 100];
    
    // ตัวอย่าง Custom Slider
    CustomVolumeSlider *customSlider = [[CustomVolumeSlider alloc] init];
    customSlider.accessibilityLabel = @"ระดับเสียงเพลง";
    customSlider.accessibilityHint = @"เลื่อนขึ้นลงเพื่อปรับระดับเสียง";
    
    // ตัวอย่าง Progress Bar
    UIProgressView *progressView = [[UIProgressView alloc] init];
    progressView.isAccessibilityElement = YES;
    progressView.accessibilityLabel = @"ความคืบหน้าการดาวน์โหลด";
    progressView.accessibilityValue = [NSString stringWithFormat:@"%.0f เปอร์เซ็นต์", progressView.progress * 100];
    
    // ตัวอย่าง Image ที่มีความหมาย
    UIImageView *ratingImage = [[UIImageView alloc] init];
    ratingImage.isAccessibilityElement = YES;
    ratingImage.accessibilityLabel = @"คะแนน 4.5 จาก 5 ดาว";
    // ไม่ต้องใช้ accessibilityHint ถ้าไม่มี Action
    
    // ตัวอย่าง Decorative Image (ไม่ต้องการ Accessibility)
    UIImageView *decorativeImage = [[UIImageView alloc] init];
    decorativeImage.isAccessibilityElement = NO; // ซ่อนจาก VoiceOver
}

@end
```

---

# หมวดที่ 2: VoiceOver Support

## 78.3 การทดสอบกับ VoiceOver

```objc
// VoiceOverSupportViewController.m
@interface VoiceOverSupportViewController : UIViewController

// Notification Observers สำหรับ VoiceOver State
@end

@implementation VoiceOverSupportViewController

- (void)viewDidLoad {
    [super viewDidLoad];
    [self setupVoiceOverNotifications];
    [self adaptUIForVoiceOver];
}

- (void)setupVoiceOverNotifications {
    // ตรวจสอบเมื่อ VoiceOver เปลี่ยนสถานะ
    [[NSNotificationCenter defaultCenter] addObserver:self
                                             selector:@selector(voiceOverStatusDidChange:)
                                                 name:UIAccessibilityVoiceOverStatusDidChangeNotification
                                               object:nil];
    
    // ตรวจสอบเมื่อมีการ Focus เปลี่ยน
    [[NSNotificationCenter defaultCenter] addObserver:self
                                             selector:@selector(accessibilityElementFocused:)
                                                 name:UIAccessibilityElementFocusedNotification
                                               object:nil];
}

- (void)voiceOverStatusDidChange:(NSNotification *)notification {
    BOOL isVoiceOverRunning = UIAccessibilityIsVoiceOverRunning();
    NSLog(@"VoiceOver: %@", isVoiceOverRunning ? @"เปิดใช้งาน" : @"ปิดใช้งาน");
    
    // ปรับ UI ตาม VoiceOver state
    [self adaptUIForVoiceOver];
}

- (void)adaptUIForVoiceOver {
    BOOL voiceOverOn = UIAccessibilityIsVoiceOverRunning();
    
    if (voiceOverOn) {
        // เพิ่ม Target ขนาดของ Button สำหรับ VoiceOver
        // (VoiceOver ใช้ Swipe เพื่อ Navigate ไม่ใช่ Touch)
        
        // ซ่อน Decorative Elements
        // self.decorativeStar.isAccessibilityElement = NO;
        
        // เพิ่มคำอธิบายให้ชัดเจนขึ้น
        NSLog(@"ปรับ UI สำหรับ VoiceOver");
    }
}

- (void)accessibilityElementFocused:(NSNotification *)notification {
    // ทราบว่า Element ไหนกำลัง Focus อยู่
    id element = notification.userInfo[UIAccessibilityFocusedElementKey];
    NSLog(@"VoiceOver Focus: %@", [element accessibilityLabel]);
}

// VoiceOver Announcements
- (void)announceToVoiceOver:(NSString *)message {
    // ประกาศข้อความผ่าน VoiceOver
    UIAccessibilityPostNotification(UIAccessibilityAnnouncementNotification, message);
}

// Focus Management
- (void)moveFocusToElement:(id)element {
    // ย้าย VoiceOver Focus ไปยัง Element ที่ต้องการ
    UIAccessibilityPostNotification(UIAccessibilityLayoutChangedNotification, element);
}

- (void)notifyScreenChanged:(id)focusElement {
    // แจ้งว่าหน้าจอเปลี่ยนแปลงทั้งหมด
    UIAccessibilityPostNotification(UIAccessibilityScreenChangedNotification, focusElement);
}

- (void)dealloc {
    [[NSNotificationCenter defaultCenter] removeObserver:self];
}

@end
```

## 78.4 Custom VoiceOver Navigation Order

```objc
// CustomNavigationOrderView.m
// ปรับลำดับการ Navigate ของ VoiceOver

@interface CustomNavigationOrderView : UIView
@end

@implementation CustomNavigationOrderView {
    UILabel *_titleLabel;
    UILabel *_priceLabel;
    UIButton *_addToCartButton;
    UIImageView *_productImage;
    UILabel *_descriptionLabel;
}

- (instancetype)initWithFrame:(CGRect)frame {
    self = [super initWithFrame:frame];
    if (self) {
        [self setupSubviews];
        [self setupAccessibility];
    }
    return self;
}

- (void)setupSubviews {
    // สร้าง UI elements
    _productImage = [[UIImageView alloc] initWithFrame:CGRectMake(0, 0, 150, 150)];
    _productImage.backgroundColor = [UIColor systemGray5Color];
    [self addSubview:_productImage];
    
    _titleLabel = [[UILabel alloc] initWithFrame:CGRectMake(160, 10, 200, 30)];
    _titleLabel.text = @"MacBook Pro 16 นิ้ว";
    _titleLabel.font = [UIFont boldSystemFontOfSize:18];
    [self addSubview:_titleLabel];
    
    _priceLabel = [[UILabel alloc] initWithFrame:CGRectMake(160, 50, 200, 25)];
    _priceLabel.text = @"฿89,900";
    _priceLabel.textColor = [UIColor systemRedColor];
    [self addSubview:_priceLabel];
    
    _descriptionLabel = [[UILabel alloc] initWithFrame:CGRectMake(160, 85, 200, 50)];
    _descriptionLabel.text = @"M3 Pro, RAM 18GB, SSD 512GB";
    _descriptionLabel.numberOfLines = 2;
    _descriptionLabel.textColor = [UIColor secondaryLabelColor];
    [self addSubview:_descriptionLabel];
    
    _addToCartButton = [UIButton buttonWithType:UIButtonTypeSystem];
    _addToCartButton.frame = CGRectMake(160, 145, 200, 44);
    [_addToCartButton setTitle:@"เพิ่มลงตะกร้า" forState:UIControlStateNormal];
    _addToCartButton.backgroundColor = [UIColor systemBlueColor];
    [_addToCartButton setTitleColor:[UIColor whiteColor] forState:UIControlStateNormal];
    _addToCartButton.layer.cornerRadius = 8;
    [self addSubview:_addToCartButton];
}

- (void)setupAccessibility {
    // กำหนดลำดับ Navigation ของ VoiceOver
    // โดยปกติ VoiceOver จะอ่านตามลำดับ Layout (ซ้ายบนไปขวาล่าง)
    // แต่เราต้องการให้อ่าน: ชื่อสินค้า -> ราคา -> รายละเอียด -> รูป -> ปุ่ม
    
    self.accessibilityElements = @[
        _titleLabel,
        _priceLabel,
        _descriptionLabel,
        _productImage,
        _addToCartButton
    ];
    
    // ตั้งค่า Accessibility ของแต่ละ Element
    _productImage.isAccessibilityElement = YES;
    _productImage.accessibilityLabel = @"รูปสินค้า MacBook Pro";
    _productImage.accessibilityTraits = UIAccessibilityTraitImage;
    
    _titleLabel.isAccessibilityElement = YES;
    _titleLabel.accessibilityLabel = @"MacBook Pro 16 นิ้ว";
    _titleLabel.accessibilityTraits = UIAccessibilityTraitHeader;
    
    _priceLabel.isAccessibilityElement = YES;
    _priceLabel.accessibilityLabel = @"ราคา 89,900 บาท";
    
    _descriptionLabel.isAccessibilityElement = YES;
    _descriptionLabel.accessibilityLabel = @"M3 Pro, RAM 18 Gigabyte, SSD 512 Gigabyte";
    
    _addToCartButton.accessibilityLabel = @"เพิ่มลงตะกร้า";
    _addToCartButton.accessibilityHint = @"เพิ่ม MacBook Pro นี้ลงในตะกร้าสินค้าของคุณ";
    _addToCartButton.accessibilityTraits = UIAccessibilityTraitButton;
}

@end
```

---

# หมวดที่ 3: Dynamic Type

## 78.5 การรองรับ Dynamic Type

**Dynamic Type** ช่วยให้ผู้ใช้ปรับขนาดตัวอักษรตามที่ต้องการในการตั้งค่าระบบ

```objc
// DynamicTypeViewController.m
#import <UIKit/UIKit.h>

@interface DynamicTypeViewController : UIViewController
@property (nonatomic, strong) UILabel *titleLabel;
@property (nonatomic, strong) UILabel *bodyLabel;
@property (nonatomic, strong) UILabel *captionLabel;
@property (nonatomic, strong) UIButton *actionButton;
@end

@implementation DynamicTypeViewController

- (void)viewDidLoad {
    [super viewDidLoad];
    [self setupDynamicTypeUI];
    [self observeContentSizeCategory];
}

- (void)setupDynamicTypeUI {
    self.view.backgroundColor = [UIColor systemBackgroundColor];
    
    // ✅ ใช้ preferredFontForTextStyle สำหรับ Dynamic Type
    // แทน [UIFont systemFontOfSize:17]
    
    // Title - ใช้ UIFontTextStyleTitle1
    self.titleLabel = [[UILabel alloc] init];
    self.titleLabel.font = [UIFont preferredFontForTextStyle:UIFontTextStyleTitle1];
    self.titleLabel.adjustsFontForContentSizeCategory = YES; // สำคัญมาก!
    self.titleLabel.text = @"หัวข้อหลัก";
    self.titleLabel.numberOfLines = 0;
    self.titleLabel.translatesAutoresizingMaskIntoConstraints = NO;
    [self.view addSubview:self.titleLabel];
    
    // Body - ใช้ UIFontTextStyleBody
    self.bodyLabel = [[UILabel alloc] init];
    self.bodyLabel.font = [UIFont preferredFontForTextStyle:UIFontTextStyleBody];
    self.bodyLabel.adjustsFontForContentSizeCategory = YES;
    self.bodyLabel.text = @"เนื้อหาหลักของแอปพลิเคชัน ควรใช้ขนาดตัวอักษรที่อ่านง่ายและปรับตามการตั้งค่าผู้ใช้ได้";
    self.bodyLabel.numberOfLines = 0;
    self.bodyLabel.translatesAutoresizingMaskIntoConstraints = NO;
    [self.view addSubview:self.bodyLabel];
    
    // Caption - ใช้ UIFontTextStyleCaption1
    self.captionLabel = [[UILabel alloc] init];
    self.captionLabel.font = [UIFont preferredFontForTextStyle:UIFontTextStyleCaption1];
    self.captionLabel.adjustsFontForContentSizeCategory = YES;
    self.captionLabel.text = @"หมายเหตุ: ข้อมูลนี้อาจมีการเปลี่ยนแปลง";
    self.captionLabel.textColor = [UIColor secondaryLabelColor];
    self.captionLabel.numberOfLines = 0;
    self.captionLabel.translatesAutoresizingMaskIntoConstraints = NO;
    [self.view addSubview:self.captionLabel];
    
    // Button - ใช้ Custom Font พร้อม Dynamic Type
    self.actionButton = [UIButton buttonWithType:UIButtonTypeSystem];
    UIFont *buttonFont = [UIFont preferredFontForTextStyle:UIFontTextStyleHeadline];
    self.actionButton.titleLabel.font = buttonFont;
    self.actionButton.titleLabel.adjustsFontForContentSizeCategory = YES;
    [self.actionButton setTitle:@"ดำเนินการต่อ" forState:UIControlStateNormal];
    self.actionButton.translatesAutoresizingMaskIntoConstraints = NO;
    [self.view addSubview:self.actionButton];
    
    // Auto Layout
    [NSLayoutConstraint activateConstraints:@[
        [self.titleLabel.topAnchor constraintEqualToAnchor:self.view.safeAreaLayoutGuide.topAnchor constant:20],
        [self.titleLabel.leadingAnchor constraintEqualToAnchor:self.view.leadingAnchor constant:20],
        [self.titleLabel.trailingAnchor constraintEqualToAnchor:self.view.trailingAnchor constant:-20],
        
        [self.bodyLabel.topAnchor constraintEqualToAnchor:self.titleLabel.bottomAnchor constant:16],
        [self.bodyLabel.leadingAnchor constraintEqualToAnchor:self.view.leadingAnchor constant:20],
        [self.bodyLabel.trailingAnchor constraintEqualToAnchor:self.view.trailingAnchor constant:-20],
        
        [self.captionLabel.topAnchor constraintEqualToAnchor:self.bodyLabel.bottomAnchor constant:12],
        [self.captionLabel.leadingAnchor constraintEqualToAnchor:self.view.leadingAnchor constant:20],
        [self.captionLabel.trailingAnchor constraintEqualToAnchor:self.view.trailingAnchor constant:-20],
        
        [self.actionButton.topAnchor constraintEqualToAnchor:self.captionLabel.bottomAnchor constant:24],
        [self.actionButton.centerXAnchor constraintEqualToAnchor:self.view.centerXAnchor],
    ]];
}

- (void)observeContentSizeCategory {
    // ตรวจสอบเมื่อผู้ใช้เปลี่ยนขนาดตัวอักษร
    [[NSNotificationCenter defaultCenter] addObserver:self
                                             selector:@selector(contentSizeCategoryDidChange:)
                                                 name:UIContentSizeCategoryDidChangeNotification
                                               object:nil];
}

- (void)contentSizeCategoryDidChange:(NSNotification *)notification {
    UIContentSizeCategory category = [UIApplication sharedApplication].preferredContentSizeCategory;
    NSLog(@"Content Size Category เปลี่ยนเป็น: %@", category);
    
    // อัปเดต UI ที่จำเป็น (ส่วนใหญ่จะอัปเดตเองถ้าตั้ง adjustsFontForContentSizeCategory = YES)
    [self.view setNeedsLayout];
}

- (void)dealloc {
    [[NSNotificationCenter defaultCenter] removeObserver:self];
}

@end
```

## 78.6 UIFontMetrics สำหรับ Custom Fonts

```objc
// CustomFontDynamicType.m
@implementation CustomFontDynamicType

- (UIFont *)scaledFontForStyle:(UIFontTextStyle)style customFont:(UIFont *)customFont {
    // ใช้ UIFontMetrics เพื่อ Scale Custom Font ตาม Dynamic Type
    UIFontMetrics *fontMetrics = [UIFontMetrics metricsForTextStyle:style];
    return [fontMetrics scaledFontForFont:customFont];
}

- (void)demonstrateCustomFontScaling {
    // Base font ขนาดปกติ
    UIFont *baseFont = [UIFont fontWithName:@"Georgia" size:17.0];
    
    // Scale ตาม Body Text Style
    UIFontMetrics *metrics = [UIFontMetrics metricsForTextStyle:UIFontTextStyleBody];
    UIFont *scaledFont = [metrics scaledFontForFont:baseFont];
    
    // ใช้กับ Label
    UILabel *label = [[UILabel alloc] init];
    label.font = scaledFont;
    label.adjustsFontForContentSizeCategory = YES;
    label.text = @"ข้อความที่ใช้ Custom Font พร้อม Dynamic Type";
    
    NSLog(@"Base font size: %.1f", baseFont.pointSize);
    NSLog(@"Scaled font size: %.1f", scaledFont.pointSize);
}

// ตรวจสอบ Accessibility Size Category
- (BOOL)isAccessibilitySize {
    UIContentSizeCategory category = [UIApplication sharedApplication].preferredContentSizeCategory;
    return [UIContentSizeCategoryIsAccessibilityCategory(category) boolValue];
}

// ปรับ Layout เมื่อใช้ Accessibility Size ขนาดใหญ่
- (void)updateLayoutForAccessibilityIfNeeded:(UIStackView *)stackView {
    if ([self isAccessibilitySize]) {
        // เปลี่ยนจาก Horizontal เป็น Vertical Layout
        stackView.axis = UILayoutConstraintAxisVertical;
        NSLog(@"เปลี่ยนเป็น Vertical layout สำหรับ Accessibility");
    } else {
        stackView.axis = UILayoutConstraintAxisHorizontal;
    }
}

@end
```

---

# หมวดที่ 4: UIAccessibilityTraits

## 78.7 ประเภทของ Accessibility Traits

```objc
// AccessibilityTraitsDemo.m
@implementation AccessibilityTraitsDemo

- (void)demonstrateTraits {
    // 1. UIAccessibilityTraitButton
    // ใช้กับ: ปุ่ม, ส่วนที่กดได้
    UIView *customButton = [[UIView alloc] init];
    customButton.isAccessibilityElement = YES;
    customButton.accessibilityLabel = @"ดาวน์โหลด";
    customButton.accessibilityTraits = UIAccessibilityTraitButton;
    // VoiceOver จะอ่าน: "ดาวน์โหลด, ปุ่ม"
    
    // 2. UIAccessibilityTraitHeader
    // ใช้กับ: หัวข้อหน้า, Section Header
    UILabel *sectionHeader = [[UILabel alloc] init];
    sectionHeader.text = @"สินค้าแนะนำ";
    sectionHeader.isAccessibilityElement = YES;
    sectionHeader.accessibilityTraits = UIAccessibilityTraitHeader;
    // VoiceOver จะอ่าน: "สินค้าแนะนำ, หัวเรื่อง"
    
    // 3. UIAccessibilityTraitLink
    // ใช้กับ: ลิงก์ที่เปิดเว็บไซต์
    UILabel *linkLabel = [[UILabel alloc] init];
    linkLabel.text = @"เยี่ยมชมเว็บไซต์";
    linkLabel.isAccessibilityElement = YES;
    linkLabel.accessibilityTraits = UIAccessibilityTraitLink;
    // VoiceOver จะอ่าน: "เยี่ยมชมเว็บไซต์, ลิงก์"
    
    // 4. UIAccessibilityTraitImage
    // ใช้กับ: รูปภาพ
    UIImageView *photo = [[UIImageView alloc] init];
    photo.isAccessibilityElement = YES;
    photo.accessibilityLabel = @"ภาพถ่ายพระอาทิตย์ตกที่ทะเล";
    photo.accessibilityTraits = UIAccessibilityTraitImage;
    
    // 5. UIAccessibilityTraitSelected
    // ใช้กับ: รายการที่ถูกเลือก
    UICollectionViewCell *selectedCell = [[UICollectionViewCell alloc] init];
    selectedCell.isAccessibilityElement = YES;
    selectedCell.accessibilityLabel = @"หมวดหมู่ อิเล็กทรอนิกส์";
    selectedCell.accessibilityTraits = UIAccessibilityTraitButton | UIAccessibilityTraitSelected;
    // VoiceOver จะอ่าน: "อิเล็กทรอนิกส์, เลือกแล้ว, ปุ่ม"
    
    // 6. UIAccessibilityTraitNotEnabled
    // ใช้กับ: ปุ่มที่ disable
    UIButton *disabledButton = [UIButton buttonWithType:UIButtonTypeSystem];
    disabledButton.enabled = NO;
    disabledButton.accessibilityLabel = @"ส่งคำสั่งซื้อ";
    disabledButton.accessibilityTraits = UIAccessibilityTraitButton | UIAccessibilityTraitNotEnabled;
    // VoiceOver จะอ่าน: "ส่งคำสั่งซื้อ, ไม่พร้อมใช้งาน, ปุ่ม"
    
    // 7. UIAccessibilityTraitUpdatesFrequently
    // ใช้กับ: Timer, Counter, Live Updates
    UILabel *timerLabel = [[UILabel alloc] init];
    timerLabel.isAccessibilityElement = YES;
    timerLabel.accessibilityLabel = @"เวลาที่เหลือ";
    timerLabel.accessibilityTraits = UIAccessibilityTraitUpdatesFrequently;
    // VoiceOver จะอ่านทุกครั้งที่ค่าเปลี่ยน
    
    // 8. UIAccessibilityTraitAllowsDirectInteraction
    // ใช้กับ: Piano Keys, Drawing Canvas, Games
    // ผู้ใช้ VoiceOver สามารถ interact โดยตรงโดยไม่ผ่าน VoiceOver gestures
    UIView *pianoView = [[UIView alloc] init];
    pianoView.isAccessibilityElement = YES;
    pianoView.accessibilityLabel = @"เปียโน Virtual";
    pianoView.accessibilityTraits = UIAccessibilityTraitAllowsDirectInteraction;
    
    // 9. UIAccessibilityTraitPlaysSound
    // ใช้กับ: ปุ่มที่เล่นเสียง
    UIButton *soundButton = [UIButton buttonWithType:UIButtonTypeSystem];
    soundButton.accessibilityLabel = @"ตัวอย่างเสียงเพลง";
    soundButton.accessibilityTraits = UIAccessibilityTraitButton | UIAccessibilityTraitPlaysSound;
    
    // 10. Combine Multiple Traits
    UIButton *specialButton = [UIButton buttonWithType:UIButtonTypeSystem];
    specialButton.accessibilityLabel = @"Facebook";
    specialButton.accessibilityTraits = UIAccessibilityTraitButton | UIAccessibilityTraitLink;
}

@end
```

---

# หมวดที่ 5: Accessibility Containers & Custom Actions

## 78.8 Accessibility Containers

```objc
// AccessibilityContainerDemo.m
// สำหรับกลุ่ม Elements ที่ต้องการให้ VoiceOver อ่านเป็นกลุ่มเดียว

@interface ProductCardView : UIView
@property (nonatomic, strong) UIImageView *productImage;
@property (nonatomic, strong) UILabel *nameLabel;
@property (nonatomic, strong) UILabel *priceLabel;
@property (nonatomic, strong) UILabel *ratingLabel;
@property (nonatomic, strong) UIButton *buyButton;
@end

@implementation ProductCardView

- (instancetype)initWithFrame:(CGRect)frame {
    self = [super initWithFrame:frame];
    if (self) {
        [self setupUI];
        [self setupAccessibilityAsContainer];
    }
    return self;
}

- (void)setupUI {
    self.backgroundColor = [UIColor secondarySystemBackgroundColor];
    self.layer.cornerRadius = 12;
    
    self.productImage = [[UIImageView alloc] initWithFrame:CGRectMake(0, 0, self.bounds.size.width, 150)];
    self.productImage.contentMode = UIViewContentModeScaleAspectFill;
    self.productImage.clipsToBounds = YES;
    [self addSubview:self.productImage];
    
    self.nameLabel = [[UILabel alloc] initWithFrame:CGRectMake(12, 158, self.bounds.size.width - 24, 24)];
    self.nameLabel.font = [UIFont preferredFontForTextStyle:UIFontTextStyleHeadline];
    self.nameLabel.adjustsFontForContentSizeCategory = YES;
    [self addSubview:self.nameLabel];
    
    self.priceLabel = [[UILabel alloc] initWithFrame:CGRectMake(12, 188, 120, 20)];
    self.priceLabel.font = [UIFont preferredFontForTextStyle:UIFontTextStyleSubheadline];
    self.priceLabel.adjustsFontForContentSizeCategory = YES;
    self.priceLabel.textColor = [UIColor systemRedColor];
    [self addSubview:self.priceLabel];
    
    self.ratingLabel = [[UILabel alloc] initWithFrame:CGRectMake(self.bounds.size.width - 80, 188, 68, 20)];
    self.ratingLabel.font = [UIFont preferredFontForTextStyle:UIFontTextStyleSubheadline];
    self.ratingLabel.adjustsFontForContentSizeCategory = YES;
    [self addSubview:self.ratingLabel];
    
    self.buyButton = [UIButton buttonWithType:UIButtonTypeSystem];
    self.buyButton.frame = CGRectMake(12, 220, self.bounds.size.width - 24, 40);
    [self.buyButton setTitle:@"ซื้อเลย" forState:UIControlStateNormal];
    [self addSubview:self.buyButton];
}

- (void)setupAccessibilityAsContainer {
    // วิธีที่ 1: ทำให้ View ทั้งหมดเป็น Accessibility Element เดียว
    // เหมาะกับ Card ที่ข้อมูลทุกอย่างสัมพันธ์กัน
    
    // ตั้งค่า Card ให้เป็น Accessibility Element
    self.isAccessibilityElement = YES;
    
    // สร้าง Label รวมจากทุก Elements
    // VoiceOver จะอ่านทั้งหมดในครั้งเดียว
    [self updateAccessibilityLabel];
    
    self.accessibilityTraits = UIAccessibilityTraitButton;
    self.accessibilityHint = @"แตะสองครั้งเพื่อดูรายละเอียดสินค้า";
    
    // ซ่อน sub-elements จาก VoiceOver
    self.productImage.isAccessibilityElement = NO;
    self.nameLabel.isAccessibilityElement = NO;
    self.priceLabel.isAccessibilityElement = NO;
    self.ratingLabel.isAccessibilityElement = NO;
    self.buyButton.isAccessibilityElement = NO;
    
    // เพิ่ม Custom Actions แทนปุ่ม
    [self setupCustomActions];
}

- (void)updateAccessibilityLabel {
    NSString *name = self.nameLabel.text ?: @"";
    NSString *price = self.priceLabel.text ?: @"";
    NSString *rating = self.ratingLabel.text ?: @"";
    
    self.accessibilityLabel = [NSString stringWithFormat:
                                @"%@, ราคา %@, คะแนน %@",
                                name, price, rating];
}

- (void)setupCustomActions {
    // เพิ่ม Custom Actions สำหรับ VoiceOver
    UIAccessibilityCustomAction *buyAction = [[UIAccessibilityCustomAction alloc] 
        initWithName:@"ซื้อสินค้า"
              target:self
            selector:@selector(handleBuyAction)];
    
    UIAccessibilityCustomAction *addToFavAction = [[UIAccessibilityCustomAction alloc]
        initWithName:@"เพิ่มในรายการโปรด"
              target:self
            selector:@selector(handleAddToFavAction)];
    
    UIAccessibilityCustomAction *shareAction = [[UIAccessibilityCustomAction alloc]
        initWithName:@"แชร์สินค้า"
              target:self
            selector:@selector(handleShareAction)];
    
    self.accessibilityCustomActions = @[buyAction, addToFavAction, shareAction];
}

- (BOOL)handleBuyAction {
    NSLog(@"ซื้อสินค้า: %@", self.nameLabel.text);
    UIAccessibilityPostNotification(UIAccessibilityAnnouncementNotification, 
                                    @"เพิ่มลงตะกร้าสินค้าแล้ว");
    return YES;
}

- (BOOL)handleAddToFavAction {
    NSLog(@"เพิ่มในรายการโปรด: %@", self.nameLabel.text);
    UIAccessibilityPostNotification(UIAccessibilityAnnouncementNotification, 
                                    @"เพิ่มในรายการโปรดแล้ว");
    return YES;
}

- (BOOL)handleShareAction {
    NSLog(@"แชร์: %@", self.nameLabel.text);
    return YES;
}

// วิธีที่ 2: ใช้ Accessibility Container (ไม่ทำให้ View เป็น Single Element)
- (void)setupAsAccessibilityContainerMode2 {
    self.isAccessibilityElement = NO; // View หลักไม่ใช่ Accessibility Element
    
    // กำหนดลำดับของ sub-elements
    self.accessibilityElements = @[
        self.productImage,
        self.nameLabel,
        self.priceLabel,
        self.ratingLabel,
        self.buyButton
    ];
    
    // ตั้งค่าแต่ละ Element
    self.productImage.isAccessibilityElement = YES;
    self.productImage.accessibilityLabel = @"รูปสินค้า";
    self.productImage.accessibilityTraits = UIAccessibilityTraitImage;
    
    self.nameLabel.isAccessibilityElement = YES;
    self.nameLabel.accessibilityTraits = UIAccessibilityTraitHeader;
    
    self.priceLabel.isAccessibilityElement = YES;
    
    self.ratingLabel.isAccessibilityElement = YES;
    
    self.buyButton.accessibilityLabel = @"ซื้อเลย";
    self.buyButton.accessibilityTraits = UIAccessibilityTraitButton;
}

@end
```

## 78.9 Custom Actions สำหรับ Table View Cell

```objc
// AccessibleTableViewCell.m
@interface AccessibleTableViewCell : UITableViewCell
@property (nonatomic, copy) NSString *itemName;
@property (nonatomic, copy) void(^deleteHandler)(void);
@property (nonatomic, copy) void(^favoriteHandler)(void);
@property (nonatomic, copy) void(^shareHandler)(void);
@end

@implementation AccessibleTableViewCell

- (void)configureWithItem:(NSString *)item {
    self.itemName = item;
    self.textLabel.text = item;
    
    // เพิ่ม Custom Accessibility Actions
    // แทนที่จะต้อง Swipe เพื่อ Delete
    [self setupAccessibilityActions];
}

- (void)setupAccessibilityActions {
    self.isAccessibilityElement = YES;
    self.accessibilityLabel = self.itemName;
    self.accessibilityTraits = UIAccessibilityTraitButton;
    
    UIAccessibilityCustomAction *deleteAction = [[UIAccessibilityCustomAction alloc]
        initWithName:@"ลบ"
              target:self
            selector:@selector(performDelete)];
    
    UIAccessibilityCustomAction *favoriteAction = [[UIAccessibilityCustomAction alloc]
        initWithName:@"เพิ่มในรายการโปรด"
              target:self
            selector:@selector(performFavorite)];
    
    UIAccessibilityCustomAction *shareAction = [[UIAccessibilityCustomAction alloc]
        initWithName:@"แชร์"
              target:self
            selector:@selector(performShare)];
    
    self.accessibilityCustomActions = @[deleteAction, favoriteAction, shareAction];
}

- (BOOL)performDelete {
    if (self.deleteHandler) {
        self.deleteHandler();
    }
    UIAccessibilityPostNotification(UIAccessibilityAnnouncementNotification,
                                    [NSString stringWithFormat:@"ลบ %@ แล้ว", self.itemName]);
    return YES;
}

- (BOOL)performFavorite {
    if (self.favoriteHandler) {
        self.favoriteHandler();
    }
    UIAccessibilityPostNotification(UIAccessibilityAnnouncementNotification,
                                    [NSString stringWithFormat:@"เพิ่ม %@ ในรายการโปรดแล้ว", self.itemName]);
    return YES;
}

- (BOOL)performShare {
    if (self.shareHandler) {
        self.shareHandler();
    }
    return YES;
}

@end
```

---

# หมวดที่ 6: Custom Accessible Controls

## 78.10 สร้าง Custom Accessible Slider

```objc
// AccessibleRatingControl.h
// Custom Star Rating Control ที่รองรับ VoiceOver ครบถ้วน

@interface AccessibleRatingControl : UIControl

@property (nonatomic, assign) NSInteger maxRating;      // จำนวนดาวสูงสุด
@property (nonatomic, assign) NSInteger currentRating;  // คะแนนปัจจุบัน

@end
```

```objc
// AccessibleRatingControl.m
#import "AccessibleRatingControl.h"

@interface AccessibleRatingControl ()
@property (nonatomic, strong) NSMutableArray<UIImageView *> *starViews;
@end

@implementation AccessibleRatingControl

- (instancetype)initWithFrame:(CGRect)frame {
    self = [super initWithFrame:frame];
    if (self) {
        _maxRating = 5;
        _currentRating = 0;
        [self setupStars];
        [self setupAccessibility];
    }
    return self;
}

- (void)setupStars {
    self.starViews = [NSMutableArray array];
    CGFloat starSize = 44.0; // Apple แนะนำ minimum touch target 44x44
    CGFloat gap = 8.0;
    
    for (NSInteger i = 0; i < self.maxRating; i++) {
        UIImageView *star = [[UIImageView alloc] initWithFrame:CGRectMake(i * (starSize + gap), 0, starSize, starSize)];
        star.contentMode = UIViewContentModeScaleAspectFit;
        star.image = [UIImage systemImageNamed:@"star"];
        star.tintColor = [UIColor systemGrayColor];
        star.userInteractionEnabled = YES;
        
        // ไม่ทำให้ Star แต่ละดวงเป็น Accessibility Element
        // เราจะจัดการ Accessibility ที่ Control ระดับบนแทน
        star.isAccessibilityElement = NO;
        
        [self addSubview:star];
        [self.starViews addObject:star];
    }
    
    self.frame = CGRectMake(self.frame.origin.x, self.frame.origin.y, 
                            self.maxRating * (starSize + gap) - gap, starSize);
}

- (void)setupAccessibility {
    // Control ทั้งหมดเป็น Accessibility Element เดียว
    self.isAccessibilityElement = YES;
    self.accessibilityLabel = @"คะแนนดาว";
    self.accessibilityHint = @"ปัดขึ้นหรือลงเพื่อเปลี่ยนคะแนน";
    
    // ใช้ Adjustable Trait เพื่อรองรับ Swipe Up/Down ใน VoiceOver
    self.accessibilityTraits = UIAccessibilityTraitAdjustable;
    
    [self updateAccessibilityValue];
}

- (void)updateAccessibilityValue {
    self.accessibilityValue = [NSString stringWithFormat:@"%ld จาก %ld ดาว", 
                                (long)self.currentRating, (long)self.maxRating];
}

// VoiceOver เรียก accessibilityIncrement เมื่อผู้ใช้ Swipe Up
- (void)accessibilityIncrement {
    if (self.currentRating < self.maxRating) {
        self.currentRating++;
        [self updateStarViews];
        [self updateAccessibilityValue];
        [self sendActionsForControlEvents:UIControlEventValueChanged];
        
        // แจ้ง VoiceOver ว่าค่าเปลี่ยน
        UIAccessibilityPostNotification(UIAccessibilityAnnouncementNotification, 
                                        self.accessibilityValue);
    }
}

// VoiceOver เรียก accessibilityDecrement เมื่อผู้ใช้ Swipe Down
- (void)accessibilityDecrement {
    if (self.currentRating > 0) {
        self.currentRating--;
        [self updateStarViews];
        [self updateAccessibilityValue];
        [self sendActionsForControlEvents:UIControlEventValueChanged];
        
        UIAccessibilityPostNotification(UIAccessibilityAnnouncementNotification, 
                                        self.accessibilityValue);
    }
}

- (void)setCurrentRating:(NSInteger)currentRating {
    _currentRating = MAX(0, MIN(currentRating, self.maxRating));
    [self updateStarViews];
    [self updateAccessibilityValue];
}

- (void)updateStarViews {
    for (NSInteger i = 0; i < self.starViews.count; i++) {
        UIImageView *star = self.starViews[i];
        if (i < self.currentRating) {
            star.image = [UIImage systemImageNamed:@"star.fill"];
            star.tintColor = [UIColor systemYellowColor];
        } else {
            star.image = [UIImage systemImageNamed:@"star"];
            star.tintColor = [UIColor systemGrayColor];
        }
    }
}

// Touch Handling
- (void)touchesBegan:(NSSet<UITouch *> *)touches withEvent:(UIEvent *)event {
    [self handleTouches:touches];
}

- (void)touchesMoved:(NSSet<UITouch *> *)touches withEvent:(UIEvent *)event {
    [self handleTouches:touches];
}

- (void)handleTouches:(NSSet<UITouch *> *)touches {
    UITouch *touch = touches.anyObject;
    CGPoint point = [touch locationInView:self];
    
    CGFloat starWidth = 44.0 + 8.0;
    NSInteger newRating = (NSInteger)(point.x / starWidth) + 1;
    newRating = MAX(0, MIN(newRating, self.maxRating));
    
    if (newRating != self.currentRating) {
        self.currentRating = newRating;
        [self sendActionsForControlEvents:UIControlEventValueChanged];
    }
}

@end
```

---

# หมวดที่ 7: Switch Control Support

## 78.11 การรองรับ Switch Control

```objc
// SwitchControlViewController.m
// Switch Control ช่วยผู้ใช้ที่มี Motor Impairment ควบคุม iPhone ด้วย External Switch

@implementation SwitchControlViewController

- (void)viewDidLoad {
    [super viewDidLoad];
    [self setupUIForSwitchControl];
    [self observeSwitchControlStatus];
}

- (void)setupUIForSwitchControl {
    // Switch Control ต้องการ:
    // 1. Accessibility Elements ที่ชัดเจน
    // 2. การจัดกลุ่มที่เหมาะสม
    // 3. Custom Actions
    
    // สร้าง Form
    UIScrollView *scrollView = [[UIScrollView alloc] initWithFrame:self.view.bounds];
    [self.view addSubview:scrollView];
    
    // Section 1: Profile Info
    UIView *profileSection = [[UIView alloc] initWithFrame:CGRectMake(20, 20, self.view.bounds.size.width - 40, 120)];
    profileSection.backgroundColor = [UIColor secondarySystemBackgroundColor];
    profileSection.layer.cornerRadius = 12;
    [scrollView addSubview:profileSection];
    
    // กำหนด Accessibility Container
    profileSection.isAccessibilityElement = NO;
    profileSection.accessibilityElements = [self createProfileElements:profileSection];
    
    // Switch Control Scanning Groups
    [self configureScanningGroups];
}

- (NSArray *)createProfileElements:(UIView *)container {
    // สร้าง Elements ที่ Switch Control จะ Scan ตามลำดับ
    UILabel *nameLabel = [[UILabel alloc] initWithFrame:CGRectMake(12, 12, 200, 30)];
    nameLabel.text = @"สมชาย ใจดี";
    nameLabel.isAccessibilityElement = YES;
    nameLabel.accessibilityLabel = @"ชื่อ: สมชาย ใจดี";
    [container addSubview:nameLabel];
    
    UILabel *emailLabel = [[UILabel alloc] initWithFrame:CGRectMake(12, 50, 250, 25)];
    emailLabel.text = @"somchai@example.com";
    emailLabel.isAccessibilityElement = YES;
    emailLabel.accessibilityLabel = @"อีเมล: somchai@example.com";
    [container addSubview:emailLabel];
    
    UIButton *editButton = [UIButton buttonWithType:UIButtonTypeSystem];
    editButton.frame = CGRectMake(container.bounds.size.width - 80, 45, 68, 36);
    [editButton setTitle:@"แก้ไข" forState:UIControlStateNormal];
    editButton.isAccessibilityElement = YES;
    editButton.accessibilityLabel = @"แก้ไขโปรไฟล์";
    editButton.accessibilityTraits = UIAccessibilityTraitButton;
    [container addSubview:editButton];
    
    return @[nameLabel, emailLabel, editButton];
}

- (void)configureScanningGroups {
    // Switch Control จะ Scan ตามลำดับ accessibilityElements
    // จัดกลุ่มให้สมเหตุสมผล เพื่อลดจำนวน Switch Click
    
    NSLog(@"Switch Control Scanning groups configured");
}

- (void)observeSwitchControlStatus {
    [[NSNotificationCenter defaultCenter] addObserver:self
                                             selector:@selector(switchControlStatusChanged:)
                                                 name:UIAccessibilitySwitchControlStatusDidChangeNotification
                                               object:nil];
}

- (void)switchControlStatusChanged:(NSNotification *)notification {
    BOOL isActive = UIAccessibilityIsSwitchControlRunning();
    NSLog(@"Switch Control: %@", isActive ? @"เปิดใช้งาน" : @"ปิดใช้งาน");
    
    if (isActive) {
        // ปรับ UI สำหรับ Switch Control
        // - เพิ่มขนาด Touch Target
        // - ลดความซับซ้อนของ Layout
        [self simplifyUIForSwitchControl];
    }
}

- (void)simplifyUIForSwitchControl {
    // รวม Elements ที่เกี่ยวข้องเข้าด้วยกัน
    // ลดจำนวน Scanning Steps
    NSLog(@"ปรับ UI สำหรับ Switch Control");
}

@end
```

---

# หมวดที่ 8: Color Accessibility

## 78.12 Color Contrast และ Visual Accessibility

```objc
// ColorAccessibilityManager.m
@interface ColorAccessibilityManager : NSObject

+ (CGFloat)contrastRatioBetweenForeground:(UIColor *)foreground 
                               background:(UIColor *)background;

+ (BOOL)meetsWCAGAAForForeground:(UIColor *)foreground 
                      background:(UIColor *)background;

+ (BOOL)meetsWCAGAAAForForeground:(UIColor *)foreground 
                       background:(UIColor *)background;

+ (UIColor *)accessibleTextColorOnBackground:(UIColor *)backgroundColor;

@end

@implementation ColorAccessibilityManager

// คำนวณ Relative Luminance ตาม WCAG
+ (CGFloat)relativeLuminanceForColor:(UIColor *)color {
    CGFloat red, green, blue, alpha;
    [color getRed:&red green:&green blue:&blue alpha:&alpha];
    
    // sRGB to linear
    CGFloat r = (red <= 0.03928) ? (red / 12.92) : pow((red + 0.055) / 1.055, 2.4);
    CGFloat g = (green <= 0.03928) ? (green / 12.92) : pow((green + 0.055) / 1.055, 2.4);
    CGFloat b = (blue <= 0.03928) ? (blue / 12.92) : pow((blue + 0.055) / 1.055, 2.4);
    
    return 0.2126 * r + 0.7152 * g + 0.0722 * b;
}

+ (CGFloat)contrastRatioBetweenForeground:(UIColor *)foreground 
                               background:(UIColor *)background {
    
    CGFloat l1 = [self relativeLuminanceForColor:foreground];
    CGFloat l2 = [self relativeLuminanceForColor:background];
    
    CGFloat lighter = MAX(l1, l2);
    CGFloat darker = MIN(l1, l2);
    
    return (lighter + 0.05) / (darker + 0.05);
}

// WCAG AA: Contrast Ratio >= 4.5:1 สำหรับ Normal Text
//          Contrast Ratio >= 3:1  สำหรับ Large Text (18pt+ หรือ 14pt Bold+)
+ (BOOL)meetsWCAGAAForForeground:(UIColor *)foreground 
                      background:(UIColor *)background {
    CGFloat ratio = [self contrastRatioBetweenForeground:foreground background:background];
    return ratio >= 4.5;
}

// WCAG AAA: Contrast Ratio >= 7:1 สำหรับ Normal Text
+ (BOOL)meetsWCAGAAAForForeground:(UIColor *)foreground 
                       background:(UIColor *)background {
    CGFloat ratio = [self contrastRatioBetweenForeground:foreground background:background];
    return ratio >= 7.0;
}

+ (UIColor *)accessibleTextColorOnBackground:(UIColor *)backgroundColor {
    CGFloat whiteContrast = [self contrastRatioBetweenForeground:[UIColor whiteColor] 
                                                      background:backgroundColor];
    CGFloat blackContrast = [self contrastRatioBetweenForeground:[UIColor blackColor] 
                                                      background:backgroundColor];
    
    return (whiteContrast > blackContrast) ? [UIColor whiteColor] : [UIColor blackColor];
}

+ (void)auditColorsInView:(UIView *)view {
    // ตรวจสอบ Contrast Ratio ของ UI Elements ทั้งหมด
    [self recurseSubviewsOf:view];
}

+ (void)recurseSubviewsOf:(UIView *)view {
    for (UIView *subview in view.subviews) {
        if ([subview isKindOfClass:[UILabel class]]) {
            UILabel *label = (UILabel *)subview;
            UIColor *bgColor = subview.backgroundColor ?: subview.superview.backgroundColor ?: [UIColor whiteColor];
            
            CGFloat ratio = [self contrastRatioBetweenForeground:label.textColor 
                                                      background:bgColor];
            
            if (ratio < 4.5) {
                NSLog(@"⚠️ Low Contrast: Label '%@', Ratio: %.1f:1 (ต้องการ >= 4.5:1)", 
                      label.text, ratio);
            }
        }
        [self recurseSubviewsOf:subview];
    }
}

@end
```

## 78.13 Differentiate Without Color

```objc
// AccessibleChartView.m
// Chart ที่ใช้ทั้งสีและรูปแบบ เพื่อรองรับผู้ที่บอดสี

@interface AccessibleChartView : UIView

@property (nonatomic, strong) NSArray<NSDictionary *> *dataPoints;

@end

@implementation AccessibleChartView

- (void)drawRect:(CGRect)rect {
    CGContextRef context = UIGraphicsGetCurrentContext();
    
    // ตรวจสอบว่าควรใช้ Differentiated Colors หรือไม่
    BOOL differentiateColors = UIAccessibilityShouldDifferentiateWithoutColor();
    BOOL reduceMotion = UIAccessibilityIsReduceMotionEnabled();
    BOOL increaseContrast = UIAccessibilityShouldDifferentiateWithoutColor();
    
    // สีสำหรับ Data Series ที่รองรับผู้บอดสี
    NSArray *accessibleColors = @[
        [UIColor systemBlueColor],    // ปกติ: น้ำเงิน
        [UIColor systemOrangeColor],  // ปกติ: ส้ม
        [UIColor systemGreenColor],   // ปกติ: เขียว
        [UIColor systemRedColor],     // ปกติ: แดง
    ];
    
    // Pattern/Line Style สำหรับผู้บอดสี
    NSArray *lineStyles = @[@"solid", @"dashed", @"dotted", @"dash-dot"];
    NSArray *markerShapes = @[@"circle", @"square", @"triangle", @"diamond"];
    
    for (NSInteger i = 0; i < self.dataPoints.count; i++) {
        UIColor *color = accessibleColors[i % accessibleColors.count];
        
        // ใช้ทั้งสีและรูปแบบ เพื่อแยกแยะ Data Series
        CGContextSetStrokeColorWithColor(context, color.CGColor);
        
        if (differentiateColors) {
            // เพิ่ม Pattern สำหรับผู้บอดสี
            NSString *lineStyle = lineStyles[i % lineStyles.count];
            if ([lineStyle isEqualToString:@"dashed"]) {
                CGFloat dashPattern[] = {8, 4};
                CGContextSetLineDash(context, 0, dashPattern, 2);
            } else if ([lineStyle isEqualToString:@"dotted"]) {
                CGFloat dotPattern[] = {2, 4};
                CGContextSetLineDash(context, 0, dotPattern, 2);
            }
        }
        
        // วาด Line...
    }
}

@end
```

---

# หมวดที่ 9: Testing with Accessibility Inspector

## 78.14 การใช้ Accessibility Inspector

```objc
// AccessibilityTestingHelper.m
// Helper สำหรับทดสอบ Accessibility ใน Unit Tests

#import <XCTest/XCTest.h>

@interface AccessibilityTestCase : XCTestCase
@end

@implementation AccessibilityTestCase

- (void)testLoginButtonAccessibility {
    // สร้าง ViewController
    LoginViewController *vc = [[LoginViewController alloc] init];
    [vc loadViewIfNeeded];
    
    UIButton *loginButton = vc.loginButton;
    
    // ตรวจสอบว่าเป็น Accessibility Element
    XCTAssertTrue(loginButton.isAccessibilityElement, 
                  @"Login button ต้องเป็น Accessibility Element");
    
    // ตรวจสอบ Label
    XCTAssertNotNil(loginButton.accessibilityLabel, 
                    @"Login button ต้องมี accessibilityLabel");
    XCTAssertGreaterThan(loginButton.accessibilityLabel.length, 0,
                         @"accessibilityLabel ต้องไม่ว่าง");
    
    // ตรวจสอบ Trait
    XCTAssertTrue(loginButton.accessibilityTraits & UIAccessibilityTraitButton,
                  @"Login button ต้องมี Button trait");
    
    NSLog(@"✅ Login button Accessibility ผ่านการทดสอบ");
}

- (void)testTextFieldAccessibility {
    LoginViewController *vc = [[LoginViewController alloc] init];
    [vc loadViewIfNeeded];
    
    UITextField *emailField = vc.emailField;
    
    XCTAssertTrue(emailField.isAccessibilityElement);
    XCTAssertNotNil(emailField.accessibilityLabel);
    
    // ตรวจสอบว่าไม่มี Generic label เช่น "text field"
    NSString *label = emailField.accessibilityLabel.lowercaseString;
    XCTAssertFalse([label isEqualToString:@"text field"],
                   @"accessibilityLabel ต้องไม่ใช่ Generic 'text field'");
    
    NSLog(@"✅ Email field Accessibility ผ่านการทดสอบ");
}

- (void)testColorContrast {
    UILabel *label = [[UILabel alloc] init];
    label.textColor = [UIColor systemGrayColor];
    label.backgroundColor = [UIColor whiteColor];
    
    CGFloat ratio = [ColorAccessibilityManager contrastRatioBetweenForeground:label.textColor 
                                                                    background:label.backgroundColor];
    
    XCTAssertGreaterThanOrEqual(ratio, 4.5,
                                @"Text contrast ratio ต้องไม่ต่ำกว่า 4.5:1 (WCAG AA)");
    
    NSLog(@"Contrast ratio: %.2f:1", ratio);
}

- (void)testDynamicTypeFontAdjustment {
    UILabel *label = [[UILabel alloc] init];
    label.font = [UIFont preferredFontForTextStyle:UIFontTextStyleBody];
    label.adjustsFontForContentSizeCategory = YES;
    
    XCTAssertTrue(label.adjustsFontForContentSizeCategory,
                  @"Label ต้องตั้ง adjustsFontForContentSizeCategory = YES");
}

@end
```

---

# แบบฝึกหัดท้ายบท

## แบบฝึกหัดที่ 1: Accessible Form Builder

สร้าง Form ที่รองรับ Accessibility ครบถ้วนประกอบด้วย:
- Text Fields พร้อม Label และ Hint
- Toggle Switches
- Date Picker
- Submit Button
- Validation Messages

```objc
// แนวทางคำตอบ
@interface AccessibleFormViewController : UIViewController

@property (nonatomic, strong) UITextField *nameField;
@property (nonatomic, strong) UITextField *emailField;
@property (nonatomic, strong) UISwitch *newsletterSwitch;
@property (nonatomic, strong) UIButton *submitButton;
@property (nonatomic, strong) UILabel *errorLabel;

@end

@implementation AccessibleFormViewController

- (void)viewDidLoad {
    [super viewDidLoad];
    [self setupForm];
    [self configureAccessibility];
}

- (void)configureAccessibility {
    // Name Field
    self.nameField.accessibilityLabel = @"ชื่อ-นามสกุล";
    self.nameField.accessibilityHint = @"ป้อนชื่อเต็มของคุณ";
    
    // Email Field
    self.emailField.accessibilityLabel = @"ที่อยู่อีเมล";
    self.emailField.accessibilityHint = @"ป้อนอีเมลที่ใช้งานอยู่";
    
    // Newsletter Switch
    self.newsletterSwitch.accessibilityLabel = @"รับข่าวสารและโปรโมชัน";
    self.newsletterSwitch.accessibilityHint = @"เปิดเพื่อรับอีเมลข่าวสารและโปรโมชัน";
    
    // Submit Button
    self.submitButton.accessibilityLabel = @"ส่งฟอร์ม";
    self.submitButton.accessibilityHint = @"ส่งข้อมูลที่กรอกทั้งหมด";
    
    // Error Label (ซ่อนเริ่มต้น)
    self.errorLabel.isAccessibilityElement = YES;
    self.errorLabel.accessibilityLabel = @"ข้อความแสดงข้อผิดพลาด";
}

- (void)showValidationError:(NSString *)message forField:(UITextField *)field {
    self.errorLabel.text = message;
    self.errorLabel.hidden = NO;
    
    // แจ้ง VoiceOver ทันที
    UIAccessibilityPostNotification(UIAccessibilityAnnouncementNotification, 
                                    [NSString stringWithFormat:@"ข้อผิดพลาด: %@", message]);
    
    // ย้าย Focus ไปยัง Field ที่มีปัญหา
    UIAccessibilityPostNotification(UIAccessibilityLayoutChangedNotification, field);
    
    // ทำให้ Field ดูเหมือนมีปัญหา
    field.layer.borderColor = [UIColor systemRedColor].CGColor;
    field.layer.borderWidth = 2.0;
    field.accessibilityValue = [NSString stringWithFormat:@"ข้อผิดพลาด: %@", message];
}

- (void)clearValidationErrorForField:(UITextField *)field {
    field.layer.borderWidth = 0;
    field.accessibilityValue = nil;
    
    if (self.nameField.layer.borderWidth == 0 && self.emailField.layer.borderWidth == 0) {
        self.errorLabel.hidden = YES;
    }
}

- (void)setupForm {
    self.nameField = [[UITextField alloc] init];
    self.emailField = [[UITextField alloc] init];
    self.newsletterSwitch = [[UISwitch alloc] init];
    self.submitButton = [UIButton buttonWithType:UIButtonTypeSystem];
    self.errorLabel = [[UILabel alloc] init];
    
    // Layout...
    NSLog(@"Form setup complete");
}

@end
```

## แบบฝึกหัดที่ 2: Accessible Music Player

สร้าง Music Player UI ที่:
- VoiceOver อ่านชื่อเพลงและศิลปิน
- รองรับ Custom Actions (Play, Next, Previous, Add to Playlist)
- Progress Bar แสดงสถานะพร้อม Accessibility Value
- Volume Slider ที่เป็น Adjustable Trait

```objc
@interface AccessibleMusicPlayerView : UIView

@property (nonatomic, copy) NSString *songTitle;
@property (nonatomic, copy) NSString *artistName;
@property (nonatomic, assign) float progress; // 0.0 - 1.0
@property (nonatomic, assign) float volume;   // 0.0 - 1.0
@property (nonatomic, assign) BOOL isPlaying;

@end

@implementation AccessibleMusicPlayerView {
    UILabel *_titleLabel;
    UILabel *_artistLabel;
    UIProgressView *_progressView;
    UISlider *_volumeSlider;
    UIButton *_playPauseButton;
    UIButton *_nextButton;
    UIButton *_prevButton;
}

- (void)configureAccessibility {
    // Player View เป็น Container
    self.isAccessibilityElement = NO;
    
    // Title + Artist รวมเป็น Element เดียว
    _titleLabel.isAccessibilityElement = YES;
    _titleLabel.accessibilityLabel = [NSString stringWithFormat:@"%@, โดย %@", 
                                       self.songTitle, self.artistName];
    _titleLabel.accessibilityTraits = UIAccessibilityTraitHeader;
    _artistLabel.isAccessibilityElement = NO;
    
    // Progress Bar
    _progressView.isAccessibilityElement = YES;
    _progressView.accessibilityLabel = @"ความคืบหน้าการเล่น";
    
    NSInteger minutes = (NSInteger)(self.progress * 240) / 60;
    NSInteger seconds = (NSInteger)(self.progress * 240) % 60;
    _progressView.accessibilityValue = [NSString stringWithFormat:@"%ld นาที %ld วินาที จาก 4 นาที", 
                                         (long)minutes, (long)seconds];
    
    // Play/Pause Button
    _playPauseButton.accessibilityLabel = self.isPlaying ? @"หยุดชั่วคราว" : @"เล่น";
    _playPauseButton.accessibilityHint = self.isPlaying ? @"หยุดเพลงชั่วคราว" : @"เล่นเพลงต่อ";
    
    // Next/Previous
    _nextButton.accessibilityLabel = @"เพลงถัดไป";
    _prevButton.accessibilityLabel = @"เพลงก่อนหน้า";
    
    // Volume - Adjustable
    _volumeSlider.accessibilityLabel = @"ระดับเสียง";
    _volumeSlider.accessibilityTraits = UIAccessibilityTraitAdjustable;
    _volumeSlider.accessibilityValue = [NSString stringWithFormat:@"%.0f เปอร์เซ็นต์", self.volume * 100];
    
    // Custom Actions
    UIAccessibilityCustomAction *addToPlaylistAction = [[UIAccessibilityCustomAction alloc]
        initWithName:@"เพิ่มใน Playlist"
              target:self
            selector:@selector(addToPlaylist)];
    
    UIAccessibilityCustomAction *shareAction = [[UIAccessibilityCustomAction alloc]
        initWithName:@"แชร์เพลง"
              target:self
            selector:@selector(shareCurrentSong)];
    
    _titleLabel.accessibilityCustomActions = @[addToPlaylistAction, shareAction];
}

- (BOOL)addToPlaylist {
    NSLog(@"เพิ่ม %@ ใน Playlist", self.songTitle);
    UIAccessibilityPostNotification(UIAccessibilityAnnouncementNotification, 
                                    @"เพิ่มเพลงใน Playlist แล้ว");
    return YES;
}

- (BOOL)shareCurrentSong {
    NSLog(@"แชร์เพลง: %@", self.songTitle);
    return YES;
}

@end
```

## แบบฝึกหัดที่ 3: Color Accessibility Checker

```objc
@interface ColorAccessibilityCheckerViewController : UIViewController
@end

@implementation ColorAccessibilityCheckerViewController

- (void)viewDidLoad {
    [super viewDidLoad];
    [self runColorAudit];
}

- (void)runColorAudit {
    // ตรวจสอบ Contrast ของ UI ทั้งหมดในแอป
    [ColorAccessibilityManager auditColorsInView:self.view];
    
    // ตรวจสอบ Color Blind Mode
    if (UIAccessibilityShouldDifferentiateWithoutColor()) {
        NSLog(@"⚠️ ผู้ใช้เปิด Differentiate Without Color - ต้องใช้รูปแบบแทนสี");
    }
    
    // คำนวณ Contrast สำหรับ Custom Elements
    UIColor *textColor = [UIColor colorWithRed:0.4 green:0.4 blue:0.4 alpha:1.0];
    UIColor *bgColor = [UIColor whiteColor];
    
    CGFloat ratio = [ColorAccessibilityManager contrastRatioBetweenForeground:textColor 
                                                                   background:bgColor];
    NSLog(@"Contrast Ratio: %.2f:1 (%@)", ratio, 
          (ratio >= 4.5) ? @"✅ WCAG AA" : @"❌ ไม่ผ่าน WCAG AA");
}

@end
```

---

## สรุปบทที่ 78

| ฟีเจอร์ | สิ่งสำคัญที่ต้องทำ |
|---------|-------------------|
| accessibilityLabel | สั้น กระชับ บอกว่า Element คืออะไร |
| accessibilityHint | บอกผลที่จะเกิดขึ้น ใช้กริยา |
| accessibilityValue | ค่าปัจจุบัน อัปเดตเมื่อค่าเปลี่ยน |
| Dynamic Type | ตั้ง adjustsFontForContentSizeCategory = YES |
| VoiceOver Order | ใช้ accessibilityElements กำหนดลำดับ |
| Custom Actions | เพิ่ม Actions ที่ VoiceOver เข้าถึงได้ |
| Color Contrast | ≥ 4.5:1 สำหรับ Normal Text (WCAG AA) |
| Adjustable Trait | Override increment/decrement สำหรับ Slider |

> **คำแนะนำ**: ทดสอบกับ VoiceOver จริงเสมอ ไม่ใช่แค่ตรวจสอบโค้ด การใช้ VoiceOver เองจะทำให้เข้าใจ User Experience ของผู้บกพร่องทางสายตาได้ดีขึ้นมาก
