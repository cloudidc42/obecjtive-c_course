# ตอนที่ 42: UIKit Basics (พื้นฐาน UIKit)

## บทนำ

UIKit เป็น framework หลักสำหรับพัฒนา UI บน iOS มาตั้งแต่เริ่มต้น (แม้ปัจจุบันจะมี SwiftUI ด้วย) UIKit มี classes ครอบคลุมทุกอย่างตั้งแต่ View, Controller, Animation ไปจนถึง Touch handling และ Drawing

ในตอนนี้เราจะเรียนรู้พื้นฐาน UIKit ทั้งหมดที่จำเป็นสำหรับการพัฒนาแอป iOS

---

## 42.1 UIKit Framework Overview

### UIKit ประกอบด้วยอะไรบ้าง?

```
UIKit Framework
├── User Interface
│   ├── Views and Controls    (UIView, UIButton, UILabel...)
│   ├── View Controllers      (UIViewController, UINavigationController...)
│   ├── Windows and Screens   (UIWindow, UIScreen)
│   └── Animations            (UIViewPropertyAnimator...)
├── User Interactions
│   ├── Touches and Presses   (UITouch, UIPress...)
│   ├── Gestures              (UIGestureRecognizer...)
│   └── Keyboard and Menus    (UITextInput...)
├── Text Display and Fonts    (UIFont, NSAttributedString...)
├── Images and PDFs           (UIImage, UIGraphicsImageRenderer...)
├── Drawing                   (UIBezierPath, Core Graphics...)
└── Printing                  (UIPrintInteractionController...)
```

### Import UIKit

```objc
// นำเข้า UIKit framework
#import <UIKit/UIKit.h>

// หรือในหลายกรณีจะถูก import ผ่าน Prefix Header ของโปรเจกต์
// ใน MyApp-Prefix.pch:
// #import <UIKit/UIKit.h>
```

---

## 42.2 UIView Hierarchy (ลำดับชั้นของ View)

### UIView คืออะไร?

`UIView` คือ base class ของทุก UI element ใน UIKit ทุก view มี:
- **Frame/Bounds**: ขนาดและตำแหน่ง
- **Superview/Subviews**: ความสัมพันธ์แบบ tree
- **Layer**: Core Animation layer สำหรับ rendering

### View Hierarchy

```
UIWindow
└── UIView (Root View of ViewController)
    ├── UILabel
    ├── UIButton
    ├── UIView (Container)
    │   ├── UIImageView
    │   └── UILabel
    └── UIScrollView
        └── UIView (Content View)
            ├── UITextField
            └── UITextView
```

### การสร้างและเพิ่ม View

```objc
// สร้าง UIView
UIView *containerView = [[UIView alloc] initWithFrame:
    CGRectMake(20, 100, 300, 200)];
containerView.backgroundColor = [UIColor systemGray6Color];
containerView.layer.cornerRadius = 12;

// เพิ่ม View เข้าไปใน hierarchy
[self.view addSubview:containerView];

// เพิ่ม subview ข้างใน
UILabel *label = [[UILabel alloc] initWithFrame:
    CGRectMake(10, 10, 280, 30)];
label.text = @"ข้อความใน Container";
[containerView addSubview:label];

// ลำดับการวาด (Z-order)
[self.view sendSubviewToBack:containerView];    // ส่งไป background
[self.view bringSubviewToFront:containerView]; // นำมา foreground

// สลับลำดับ
[self.view exchangeSubviewAtIndex:0 withSubviewAtIndex:1];

// ลบ view ออกจาก parent
[label removeFromSuperview];

// ตรวจสอบ hierarchy
NSLog(@"Superview: %@", label.superview);
NSLog(@"Subviews: %@", containerView.subviews);
NSLog(@"Window: %@", label.window);

// ดึง View จาก Tag
label.tag = 42;
UILabel *foundLabel = (UILabel *)[self.view viewWithTag:42];
```

### View Properties พื้นฐาน

```objc
UIView *view = [[UIView alloc] init];

// Frame: position ใน coordinate space ของ superview
view.frame = CGRectMake(10, 20, 100, 50);
// x=10, y=20 จาก superview origin
// width=100, height=50

// Bounds: ขนาดใน coordinate space ของตัวเอง
NSLog(@"Bounds: %@", NSStringFromCGRect(view.bounds));
// bounds.origin = (0,0) เสมอ (ปกติ)
// bounds.size = view.frame.size

// Center: จุดกึ่งกลาง ใน coordinate space ของ superview
view.center = CGPointMake(100, 200);

// Alpha: ความโปร่งใส (0.0 = transparent, 1.0 = opaque)
view.alpha = 0.8;

// Hidden: ซ่อน/แสดง (ยังคงพื้นที่)
view.hidden = NO;

// Clip to bounds: ตัด subviews ที่เกินขอบ
view.clipsToBounds = YES;

// User Interaction
view.userInteractionEnabled = YES;

// Background Color
view.backgroundColor = [UIColor systemBlueColor];

// Tint Color (สีธีม)
view.tintColor = [UIColor systemOrangeColor];

// Transform (rotate, scale, translate)
view.transform = CGAffineTransformMakeRotation(M_PI / 4); // 45 องศา
view.transform = CGAffineTransformMakeScale(1.5, 1.5);    // ขยาย 1.5x
view.transform = CGAffineTransformMakeTranslation(10, 0); // เลื่อน

// Reset transform
view.transform = CGAffineTransformIdentity;
```

---

## 42.3 UIWindow

### UIWindow คืออะไร?

`UIWindow` คือ root container สำหรับ views ทั้งหมด มักมีแค่ 1 window ในแอป

```objc
// สร้าง UIWindow (ทำใน SceneDelegate หรือ AppDelegate)
UIWindow *window = [[UIWindow alloc] initWithWindowScene:windowScene];

// ตั้งค่า Root View Controller
UIViewController *rootVC = [[UIViewController alloc] init];
window.rootViewController = rootVC;

// แสดง window และทำให้เป็น key window
[window makeKeyAndVisible];

// เข้าถึง Key Window
UIWindow *keyWindow = nil;
for (UIScene *scene in [UIApplication sharedApplication].connectedScenes) {
    if ([scene isKindOfClass:[UIWindowScene class]]) {
        UIWindowScene *windowScene = (UIWindowScene *)scene;
        for (UIWindow *w in windowScene.windows) {
            if (w.isKeyWindow) {
                keyWindow = w;
                break;
            }
        }
    }
}

// ดึง Root ViewController
UIViewController *rootVC2 = keyWindow.rootViewController;

// Coordinate conversion
CGPoint pointInWindow = [someView convertPoint:CGPointMake(10, 10) 
                                        toView:nil]; // nil = window

// Window Level
window.windowLevel = UIWindowLevelNormal;     // ปกติ
window.windowLevel = UIWindowLevelAlert;      // บน Alert
window.windowLevel = UIWindowLevelStatusBar;  // ระดับ Status Bar
```

---

## 42.4 Common UI Controls

### UILabel

```objc
// สร้าง UILabel
UILabel *label = [[UILabel alloc] initWithFrame:
    CGRectMake(20, 100, 300, 44)];

// ข้อความ
label.text = @"สวัสดีชาวโลก!";

// Font
label.font = [UIFont systemFontOfSize:18];
label.font = [UIFont boldSystemFontOfSize:20];
label.font = [UIFont italicSystemFontOfSize:16];

// Custom Font
UIFont *customFont = [UIFont fontWithName:@"Helvetica-Bold" size:18];
if (customFont) {
    label.font = customFont;
}

// สี
label.textColor = [UIColor labelColor];
label.shadowColor = [UIColor grayColor];
label.shadowOffset = CGSizeMake(1, 1);

// การจัดตำแหน่ง
label.textAlignment = NSTextAlignmentLeft;
label.textAlignment = NSTextAlignmentCenter;
label.textAlignment = NSTextAlignmentRight;
label.textAlignment = NSTextAlignmentNatural; // ตามภาษา

// หลายบรรทัด
label.numberOfLines = 0;      // ไม่จำกัดบรรทัด
label.numberOfLines = 3;      // จำกัด 3 บรรทัด
label.lineBreakMode = NSLineBreakByWordWrapping;
label.lineBreakMode = NSLineBreakByTruncatingTail; // ตัด... ท้าย

// ปรับขนาดตัวอักษรตาม frame
label.adjustsFontSizeToFitWidth = YES;
label.minimumScaleFactor = 0.5;

// Attributed String (ข้อความที่มี style ต่างกัน)
NSMutableAttributedString *attrString = 
    [[NSMutableAttributedString alloc] initWithString:@"Hello World"];
    
// ทำ "Hello" เป็น bold สีแดง
[attrString addAttribute:NSFontAttributeName 
                   value:[UIFont boldSystemFontOfSize:20]
                   range:NSMakeRange(0, 5)];
[attrString addAttribute:NSForegroundColorAttributeName 
                   value:[UIColor systemRedColor]
                   range:NSMakeRange(0, 5)];

// ทำ "World" เป็น underline
[attrString addAttribute:NSUnderlineStyleAttributeName
                   value:@(NSUnderlineStyleSingle)
                   range:NSMakeRange(6, 5)];

label.attributedText = attrString;

// ปรับขนาด Label ให้พอดีกับ text
[label sizeToFit];
CGSize textSize = [label.text sizeWithAttributes:@{
    NSFontAttributeName: label.font
}];
```

### UIButton

```objc
// สร้าง UIButton
UIButton *button = [UIButton buttonWithType:UIButtonTypeSystem];
button.frame = CGRectMake(20, 200, 150, 44);

// Title
[button setTitle:@"กดฉัน" forState:UIControlStateNormal];
[button setTitle:@"กำลังกด..." forState:UIControlStateHighlighted];
[button setTitle:@"ปิดใช้งาน" forState:UIControlStateDisabled];

// Title Color
[button setTitleColor:[UIColor systemBlueColor] 
             forState:UIControlStateNormal];
[button setTitleColor:[UIColor grayColor] 
             forState:UIControlStateDisabled];

// Font
button.titleLabel.font = [UIFont boldSystemFontOfSize:16];

// Image
UIImage *icon = [UIImage systemImageNamed:@"star.fill"];
[button setImage:icon forState:UIControlStateNormal];
button.imageView.contentMode = UIViewContentModeScaleAspectFit;

// Background Color
button.backgroundColor = [UIColor systemBlueColor];
[button setBackgroundColor:[UIColor systemBlue] 
                  forState:UIControlStateNormal];

// Styling
button.layer.cornerRadius = 10;
button.layer.borderWidth = 1.0;
button.layer.borderColor = [UIColor systemBlueColor].CGColor;

// Content Insets
button.contentEdgeInsets = UIEdgeInsetsMake(8, 16, 8, 16);
button.titleEdgeInsets = UIEdgeInsetsMake(0, 8, 0, 0); // ขยับ title
button.imageEdgeInsets = UIEdgeInsetsMake(0, 0, 0, 8); // ขยับ image

// สถานะ
button.enabled = YES;
button.selected = NO;
button.highlighted = NO;

// เพิ่ม Action
[button addTarget:self 
           action:@selector(buttonTapped:)
 forControlEvents:UIControlEventTouchUpInside];

// ลบ Action
[button removeTarget:self 
              action:@selector(buttonTapped:)
    forControlEvents:UIControlEventTouchUpInside];

// iOS 15+ Button Configuration
if (@available(iOS 15.0, *)) {
    UIButtonConfiguration *config = [UIButtonConfiguration filledButtonConfiguration];
    config.title = @"iOS 15+ Button";
    config.image = [UIImage systemImageNamed:@"plus"];
    config.imagePadding = 8;
    config.cornerStyle = UIButtonConfigurationCornerStyleMedium;
    config.baseBackgroundColor = [UIColor systemPurpleColor];
    button.configuration = config;
}

- (void)buttonTapped:(UIButton *)sender {
    NSLog(@"ปุ่มถูกกด: %@", sender.currentTitle);
}
```

### UITextField

```objc
// สร้าง UITextField
UITextField *textField = [[UITextField alloc] initWithFrame:
    CGRectMake(20, 300, 300, 44)];

// Placeholder
textField.placeholder = @"กรอกข้อความ...";

// Attributed Placeholder (เปลี่ยนสี)
NSAttributedString *attrPlaceholder = 
    [[NSAttributedString alloc] 
        initWithString:@"กรอกข้อความ..."
            attributes:@{
                NSForegroundColorAttributeName: [UIColor systemGrayColor]
            }];
textField.attributedPlaceholder = attrPlaceholder;

// Style
textField.borderStyle = UITextBorderStyleRoundedRect;
textField.borderStyle = UITextBorderStyleLine;
textField.borderStyle = UITextBorderStyleBezel;
textField.borderStyle = UITextBorderStyleNone;

// Font และ Color
textField.font = [UIFont systemFontOfSize:16];
textField.textColor = [UIColor labelColor];

// Keyboard
textField.keyboardType = UIKeyboardTypeDefault;
textField.keyboardType = UIKeyboardTypeEmailAddress;
textField.keyboardType = UIKeyboardTypeNumberPad;
textField.keyboardType = UIKeyboardTypePhonePad;
textField.keyboardType = UIKeyboardTypeURL;
textField.keyboardType = UIKeyboardTypeDecimalPad;

// Return Key
textField.returnKeyType = UIReturnKeyDone;
textField.returnKeyType = UIReturnKeyNext;
textField.returnKeyType = UIReturnKeySearch;
textField.returnKeyType = UIReturnKeySend;

// Secure (Password)
textField.secureTextEntry = YES;

// Autocorrect
textField.autocorrectionType = UITextAutocorrectionTypeNo;
textField.autocapitalizationType = UITextAutocapitalizationTypeNone;

// Clear Button
textField.clearButtonMode = UITextFieldViewModeWhileEditing;
textField.clearButtonMode = UITextFieldViewModeAlways;
textField.clearButtonMode = UITextFieldViewModeNever;

// Left/Right View
UIView *paddingView = [[UIView alloc] initWithFrame:CGRectMake(0, 0, 8, 44)];
textField.leftView = paddingView;
textField.leftViewMode = UITextFieldViewModeAlways;

UIImageView *iconView = [[UIImageView alloc] initWithImage:
    [UIImage systemImageNamed:@"magnifyingglass"]];
iconView.frame = CGRectMake(0, 0, 36, 44);
iconView.contentMode = UIViewContentModeCenter;
iconView.tintColor = [UIColor secondaryLabelColor];
textField.leftView = iconView;
textField.leftViewMode = UITextFieldViewModeAlways;

// Delegate
textField.delegate = self;

// Keyboard Handling
[[NSNotificationCenter defaultCenter] addObserver:self
    selector:@selector(keyboardWillShow:)
        name:UIKeyboardWillShowNotification
      object:nil];

[[NSNotificationCenter defaultCenter] addObserver:self
    selector:@selector(keyboardWillHide:)
        name:UIKeyboardWillHideNotification
      object:nil];

// UITextFieldDelegate methods
- (BOOL)textFieldShouldReturn:(UITextField *)textField {
    [textField resignFirstResponder]; // ซ่อน keyboard
    return YES;
}

- (BOOL)textField:(UITextField *)textField 
    shouldChangeCharactersInRange:(NSRange)range 
                replacementString:(NSString *)string {
    
    // จำกัดจำนวนตัวอักษร
    NSString *newText = [textField.text 
        stringByReplacingCharactersInRange:range withString:string];
    return newText.length <= 50;
}

- (void)textFieldDidBeginEditing:(UITextField *)textField {
    NSLog(@"เริ่มพิมพ์");
}

- (void)textFieldDidEndEditing:(UITextField *)textField {
    NSLog(@"หยุดพิมพ์: %@", textField.text);
}
```

### UITextView

```objc
// สร้าง UITextView
UITextView *textView = [[UITextView alloc] initWithFrame:
    CGRectMake(20, 400, 300, 150)];

// ข้อความ
textView.text = @"นี่คือ UITextView สำหรับข้อความหลายบรรทัด\nสามารถ scroll และแก้ไขได้";

// Styling
textView.font = [UIFont systemFontOfSize:16];
textView.textColor = [UIColor labelColor];
textView.backgroundColor = [UIColor systemGray6Color];
textView.layer.cornerRadius = 8;
textView.textContainerInset = UIEdgeInsetsMake(12, 12, 12, 12);

// Editable
textView.editable = YES;
textView.selectable = YES;

// Keyboard
textView.keyboardType = UIKeyboardTypeDefault;
textView.returnKeyType = UIReturnKeyDefault;

// Scrollable
textView.scrollEnabled = YES;
textView.showsVerticalScrollIndicator = YES;

// Link detection
textView.dataDetectorTypes = UIDataDetectorTypeLink;
textView.dataDetectorTypes = UIDataDetectorTypePhoneNumber;
textView.dataDetectorTypes = UIDataDetectorTypeAll;

// Delegate
textView.delegate = self;

// UITextViewDelegate
- (void)textViewDidChange:(UITextView *)textView {
    NSLog(@"ข้อความเปลี่ยน: %lu ตัวอักษร", textView.text.length);
}

- (BOOL)textView:(UITextView *)textView 
    shouldChangeTextInRange:(NSRange)range 
            replacementText:(NSString *)text {
    
    // กด Return ปิด keyboard
    if ([text isEqualToString:@"\n"]) {
        [textView resignFirstResponder];
        return NO;
    }
    return YES;
}
```

---

## 42.5 UIImageView

```objc
// สร้าง UIImageView
UIImageView *imageView = [[UIImageView alloc] initWithFrame:
    CGRectMake(20, 50, 200, 200)];

// โหลดรูปจาก Assets
UIImage *image = [UIImage imageNamed:@"photo"];
imageView.image = image;

// SF Symbols (iOS 13+)
UIImage *sfImage = [UIImage systemImageNamed:@"heart.fill"];
imageView.image = sfImage;
imageView.tintColor = [UIColor systemRedColor];

// โหลดรูปจาก URL (ต้องทำ async)
NSURL *imageURL = [NSURL URLWithString:@"https://example.com/image.jpg"];
dispatch_async(dispatch_get_global_queue(DISPATCH_QUEUE_PRIORITY_DEFAULT, 0), ^{
    NSData *imageData = [NSData dataWithContentsOfURL:imageURL];
    UIImage *downloadedImage = [UIImage imageWithData:imageData];
    
    dispatch_async(dispatch_get_main_queue(), ^{
        imageView.image = downloadedImage;
    });
});

// Content Mode
imageView.contentMode = UIViewContentModeScaleAspectFit;  // คง aspect ratio ไม่ครอบ
imageView.contentMode = UIViewContentModeScaleAspectFill; // คง aspect ratio ครอบ
imageView.contentMode = UIViewContentModeScaleToFill;     // ยืด/หดให้เต็ม
imageView.contentMode = UIViewContentModeCenter;          // กลาง ไม่ scale
imageView.contentMode = UIViewContentModeTop;             // บน
imageView.contentMode = UIViewContentModeRedraw;          // วาดใหม่เสมอ

// Clip
imageView.clipsToBounds = YES;

// Tint (สำหรับ template image)
UIImage *templateImage = [[UIImage imageNamed:@"icon"] 
    imageWithRenderingMode:UIImageRenderingModeAlwaysTemplate];
imageView.image = templateImage;
imageView.tintColor = [UIColor systemBlueColor];

// Animation
NSArray *animationImages = @[
    [UIImage imageNamed:@"frame1"],
    [UIImage imageNamed:@"frame2"],
    [UIImage imageNamed:@"frame3"],
    [UIImage imageNamed:@"frame4"]
];
imageView.animationImages = animationImages;
imageView.animationDuration = 0.5;
imageView.animationRepeatCount = 0; // 0 = วนไม่จำกัด
[imageView startAnimating];
[imageView stopAnimating];

// ทำ UIImageView เป็น circle
imageView.layer.cornerRadius = imageView.frame.size.width / 2;
imageView.clipsToBounds = YES;

// Border
imageView.layer.borderWidth = 2.0;
imageView.layer.borderColor = [UIColor systemBlueColor].CGColor;
```

---

## 42.6 UISwitch, UISlider, UISegmentedControl, UIProgressView

### UISwitch

```objc
// สร้าง UISwitch
UISwitch *mySwitch = [[UISwitch alloc] initWithFrame:
    CGRectMake(20, 100, 0, 0)]; // frame ถูก ignore, ใช้ intrinsicContentSize

// ตั้งค่า
mySwitch.on = YES;

// สี
mySwitch.onTintColor = [UIColor systemGreenColor];
mySwitch.thumbTintColor = [UIColor whiteColor];

// เพิ่ม Action
[mySwitch addTarget:self 
             action:@selector(switchChanged:)
   forControlEvents:UIControlEventValueChanged];

- (void)switchChanged:(UISwitch *)sender {
    NSLog(@"Switch: %@", sender.isOn ? @"เปิด" : @"ปิด");
    
    // เปลี่ยนค่าด้วย animation
    [sender setOn:!sender.isOn animated:YES];
}
```

### UISlider

```objc
// สร้าง UISlider
UISlider *slider = [[UISlider alloc] initWithFrame:
    CGRectMake(20, 150, 300, 44)];

// ค่า
slider.minimumValue = 0.0;
slider.maximumValue = 100.0;
slider.value = 50.0;

// สี
slider.minimumTrackTintColor = [UIColor systemBlueColor];
slider.maximumTrackTintColor = [UIColor systemGrayColor];
slider.thumbTintColor = [UIColor systemBlueColor];

// Custom Images
UIImage *thumbImage = [UIImage imageNamed:@"thumb"];
[slider setThumbImage:thumbImage forState:UIControlStateNormal];

UIImage *minImage = [UIImage imageNamed:@"min"];
UIImage *maxImage = [UIImage imageNamed:@"max"];
slider.minimumValueImage = minImage;
slider.maximumValueImage = maxImage;

// Continuous
slider.continuous = YES; // เรียก action ขณะลาก
// slider.continuous = NO; // เรียก action เฉพาะเมื่อปล่อย

// Action
[slider addTarget:self 
           action:@selector(sliderChanged:)
 forControlEvents:UIControlEventValueChanged];

- (void)sliderChanged:(UISlider *)sender {
    NSLog(@"Slider Value: %.1f", sender.value);
    
    // เปลี่ยน step (snapping)
    float step = 10.0;
    float roundedValue = roundf(sender.value / step) * step;
    [sender setValue:roundedValue animated:YES];
}
```

### UISegmentedControl

```objc
// สร้าง UISegmentedControl
NSArray *items = @[@"เล็ก", @"กลาง", @"ใหญ่"];
UISegmentedControl *segControl = 
    [[UISegmentedControl alloc] initWithItems:items];
segControl.frame = CGRectMake(20, 200, 300, 32);

// เลือก segment
segControl.selectedSegmentIndex = 0;

// สี (iOS 13+)
segControl.selectedSegmentTintColor = [UIColor systemBlueColor];
segControl.backgroundColor = [UIColor systemGray5Color];

// ตั้งค่า title
[segControl setTitle:@"ขนาดใหม่" forSegmentAtIndex:0];

// ตั้งค่า image
UIImage *icon = [UIImage systemImageNamed:@"star"];
[segControl setImage:icon forSegmentAtIndex:1];

// เพิ่ม/ลบ segment
[segControl insertSegmentWithTitle:@"ใหม่" atIndex:2 animated:YES];
[segControl removeSegmentAtIndex:2 animated:YES];

// Width
[segControl setWidth:80 forSegmentAtIndex:0];
segControl.apportionsSegmentWidthsByContent = YES;

// Enabled
[segControl setEnabled:NO forSegmentAtIndex:2];

// Action
[segControl addTarget:self 
               action:@selector(segmentChanged:)
     forControlEvents:UIControlEventValueChanged];

- (void)segmentChanged:(UISegmentedControl *)sender {
    NSLog(@"เลือก: Index=%ld, Title=%@", 
          sender.selectedSegmentIndex,
          [sender titleForSegmentAtIndex:sender.selectedSegmentIndex]);
    
    switch (sender.selectedSegmentIndex) {
        case 0: NSLog(@"เล็ก"); break;
        case 1: NSLog(@"กลาง"); break;
        case 2: NSLog(@"ใหญ่"); break;
    }
}
```

### UIProgressView

```objc
// สร้าง UIProgressView
UIProgressView *progressView = [[UIProgressView alloc] 
    initWithProgressViewStyle:UIProgressViewStyleDefault];
progressView.frame = CGRectMake(20, 250, 300, 10);

// Progress (0.0 - 1.0)
progressView.progress = 0.5; // 50%

// สี
progressView.progressTintColor = [UIColor systemBlueColor];
progressView.trackTintColor = [UIColor systemGray5Color];

// Custom Images
progressView.progressImage = [UIImage imageNamed:@"progressFill"];
progressView.trackImage = [UIImage imageNamed:@"progressTrack"];

// Animate progress
[progressView setProgress:0.8 animated:YES];

// ตัวอย่าง: อัปเดต progress จาก background task
NSTimer *progressTimer = [NSTimer scheduledTimerWithTimeInterval:0.1
    target:self
    selector:@selector(updateProgress:)
    userInfo:nil
    repeats:YES];

- (void)updateProgress:(NSTimer *)timer {
    if (self.progressView.progress >= 1.0) {
        [timer invalidate];
        NSLog(@"เสร็จแล้ว!");
        return;
    }
    [self.progressView setProgress:self.progressView.progress + 0.01 
                          animated:YES];
}
```

---

## 42.7 Creating UI Programmatically vs Interface Builder

### Interface Builder (Storyboard / XIB)

**ข้อดี:**
- เห็น UI ได้ทันที
- ง่ายสำหรับมือใหม่
- Auto Layout ง่ายกว่า

**ข้อเสีย:**
- Merge conflict ในทีม
- ยากในการ reuse
- ช้ากว่าในโปรเจกต์ใหญ่

### Programmatic UI

**ข้อดี:**
- ควบคุมได้สมบูรณ์
- Merge ง่าย
- Reuse ง่าย
- เหมาะกับทีมขนาดใหญ่

**ข้อเสีย:**
- โค้ดมากกว่า
- ต้องสร้าง constraints ด้วยมือ

### ตัวอย่าง Programmatic UI

```objc
// ViewController ที่ไม่ใช้ Storyboard เลย

#import "ProgrammaticViewController.h"

@interface ProgrammaticViewController ()

@property (nonatomic, strong) UILabel *titleLabel;
@property (nonatomic, strong) UIButton *actionButton;
@property (nonatomic, strong) UITextField *inputField;
@property (nonatomic, strong) UITableView *tableView;

@end

@implementation ProgrammaticViewController

- (void)loadView {
    // สร้าง root view เอง (ถ้าไม่ใช้ Storyboard)
    self.view = [[UIView alloc] initWithFrame:UIScreen.mainScreen.bounds];
    self.view.backgroundColor = [UIColor systemBackgroundColor];
}

- (void)viewDidLoad {
    [super viewDidLoad];
    [self setupUI];
    [self setupConstraints];
}

- (void)setupUI {
    // Title Label
    self.titleLabel = [[UILabel alloc] init];
    self.titleLabel.translatesAutoresizingMaskIntoConstraints = NO;
    self.titleLabel.text = @"หัวข้อหน้าจอ";
    self.titleLabel.font = [UIFont boldSystemFontOfSize:24];
    self.titleLabel.textAlignment = NSTextAlignmentCenter;
    [self.view addSubview:self.titleLabel];
    
    // Input Field
    self.inputField = [[UITextField alloc] init];
    self.inputField.translatesAutoresizingMaskIntoConstraints = NO;
    self.inputField.placeholder = @"กรอกข้อมูล";
    self.inputField.borderStyle = UITextBorderStyleRoundedRect;
    self.inputField.delegate = self;
    [self.view addSubview:self.inputField];
    
    // Action Button
    self.actionButton = [UIButton buttonWithType:UIButtonTypeSystem];
    self.actionButton.translatesAutoresizingMaskIntoConstraints = NO;
    [self.actionButton setTitle:@"ดำเนินการ" forState:UIControlStateNormal];
    self.actionButton.backgroundColor = [UIColor systemBlueColor];
    [self.actionButton setTitleColor:[UIColor whiteColor] 
                            forState:UIControlStateNormal];
    self.actionButton.layer.cornerRadius = 10;
    [self.actionButton addTarget:self 
                          action:@selector(actionButtonTapped)
                forControlEvents:UIControlEventTouchUpInside];
    [self.view addSubview:self.actionButton];
    
    // TableView
    self.tableView = [[UITableView alloc] init];
    self.tableView.translatesAutoresizingMaskIntoConstraints = NO;
    self.tableView.delegate = self;
    self.tableView.dataSource = self;
    [self.tableView registerClass:[UITableViewCell class] 
           forCellReuseIdentifier:@"Cell"];
    [self.view addSubview:self.tableView];
}

- (void)setupConstraints {
    UILayoutGuide *safeArea = self.view.safeAreaLayoutGuide;
    
    [NSLayoutConstraint activateConstraints:@[
        // Title Label
        [self.titleLabel.topAnchor constraintEqualToAnchor:safeArea.topAnchor 
                                                  constant:20],
        [self.titleLabel.leadingAnchor constraintEqualToAnchor:self.view.leadingAnchor 
                                                      constant:20],
        [self.titleLabel.trailingAnchor constraintEqualToAnchor:self.view.trailingAnchor 
                                                       constant:-20],
        
        // Input Field
        [self.inputField.topAnchor constraintEqualToAnchor:self.titleLabel.bottomAnchor 
                                                  constant:16],
        [self.inputField.leadingAnchor constraintEqualToAnchor:self.view.leadingAnchor 
                                                      constant:20],
        [self.inputField.trailingAnchor constraintEqualToAnchor:self.view.trailingAnchor 
                                                       constant:-20],
        [self.inputField.heightAnchor constraintEqualToConstant:44],
        
        // Action Button
        [self.actionButton.topAnchor constraintEqualToAnchor:self.inputField.bottomAnchor 
                                                   constant:16],
        [self.actionButton.leadingAnchor constraintEqualToAnchor:self.view.leadingAnchor 
                                                       constant:20],
        [self.actionButton.trailingAnchor constraintEqualToAnchor:self.view.trailingAnchor 
                                                        constant:-20],
        [self.actionButton.heightAnchor constraintEqualToConstant:44],
        
        // TableView
        [self.tableView.topAnchor constraintEqualToAnchor:self.actionButton.bottomAnchor 
                                                constant:16],
        [self.tableView.leadingAnchor constraintEqualToAnchor:self.view.leadingAnchor],
        [self.tableView.trailingAnchor constraintEqualToAnchor:self.view.trailingAnchor],
        [self.tableView.bottomAnchor constraintEqualToAnchor:safeArea.bottomAnchor],
    ]];
}

@end
```

---

## 42.8 Auto Layout Introduction

### Auto Layout คืออะไร?

Auto Layout เป็นระบบ constraint-based layout ที่ช่วยให้ UI ปรับตัวตามขนาดหน้าจอต่างๆ โดยอัตโนมัติ

### หลักการ Auto Layout

```
ทุก View ต้องมี:
1. ตำแหน่ง (Position): X, Y หรือ Leading/Trailing, Top/Bottom
2. ขนาด (Size): Width, Height

ถ้าขาดอย่างใดอย่างหนึ่ง = Ambiguous Layout (เตือน)
ถ้ามีขัดแย้งกัน = Conflicting Constraints (Error)
```

### Anchor-based Constraints (แนะนำ)

```objc
// ตั้งค่า translatesAutoresizingMaskIntoConstraints = NO ก่อนเสมอ
myView.translatesAutoresizingMaskIntoConstraints = NO;
[self.view addSubview:myView];

UILayoutGuide *safeArea = self.view.safeAreaLayoutGuide;

// วิธีที่ 1: activate ทีละ constraint
[myView.topAnchor constraintEqualToAnchor:safeArea.topAnchor 
                                 constant:20].active = YES;
[myView.leadingAnchor constraintEqualToAnchor:self.view.leadingAnchor 
                                     constant:20].active = YES;
[myView.trailingAnchor constraintEqualToAnchor:self.view.trailingAnchor 
                                      constant:-20].active = YES;
[myView.heightAnchor constraintEqualToConstant:100].active = YES;

// วิธีที่ 2: activateConstraints (แนะนำ)
[NSLayoutConstraint activateConstraints:@[
    [myView.topAnchor constraintEqualToAnchor:safeArea.topAnchor 
                                     constant:20],
    [myView.leadingAnchor constraintEqualToAnchor:self.view.leadingAnchor 
                                         constant:20],
    [myView.trailingAnchor constraintEqualToAnchor:self.view.trailingAnchor 
                                          constant:-20],
    [myView.heightAnchor constraintEqualToConstant:100],
]];

// Constraint Types
// Equal
[view.widthAnchor constraintEqualToConstant:100]
// Greater Than or Equal
[view.widthAnchor constraintGreaterThanOrEqualToConstant:50]
// Less Than or Equal
[view.widthAnchor constraintLessThanOrEqualToConstant:200]

// Multiplier
[view.widthAnchor constraintEqualToAnchor:self.view.widthAnchor 
                               multiplier:0.5]  // ครึ่งหน้าจอ

// Priority (ใช้กับ conflicting constraints)
NSLayoutConstraint *constraint = 
    [view.heightAnchor constraintEqualToConstant:100];
constraint.priority = UILayoutPriorityDefaultHigh; // 750
// UILayoutPriorityRequired = 1000 (ต้องทำตาม)
// UILayoutPriorityDefaultHigh = 750
// UILayoutPriorityDefaultLow = 250

// Center
[view.centerXAnchor constraintEqualToAnchor:self.view.centerXAnchor]
[view.centerYAnchor constraintEqualToAnchor:self.view.centerYAnchor]

// Constant แบบ dynamic (เก็บ reference)
NSLayoutConstraint *heightConstraint = 
    [view.heightAnchor constraintEqualToConstant:100];
heightConstraint.active = YES;

// เปลี่ยนทีหลัง
heightConstraint.constant = 200;
[UIView animateWithDuration:0.3 animations:^{
    [self.view layoutIfNeeded];
}];
```

---

## 42.9 NSLayoutConstraint

### การสร้าง NSLayoutConstraint โดยตรง

```objc
// NSLayoutConstraint:
// item.attribute relation toItem.attribute * multiplier + constant

// view.left = superview.left * 1.0 + 20
NSLayoutConstraint *leftConstraint = 
    [NSLayoutConstraint constraintWithItem:view
                                 attribute:NSLayoutAttributeLeading
                                 relatedBy:NSLayoutRelationEqual
                                    toItem:self.view
                                 attribute:NSLayoutAttributeLeading
                                multiplier:1.0
                                  constant:20];

// view.width = superview.width * 0.5
NSLayoutConstraint *widthConstraint = 
    [NSLayoutConstraint constraintWithItem:view
                                 attribute:NSLayoutAttributeWidth
                                 relatedBy:NSLayoutRelationEqual
                                    toItem:self.view
                                 attribute:NSLayoutAttributeWidth
                                multiplier:0.5
                                  constant:0];

// Activate
[NSLayoutConstraint activateConstraints:@[leftConstraint, widthConstraint]];

// Deactivate
[NSLayoutConstraint deactivateConstraints:@[leftConstraint]];

// Attributes
// NSLayoutAttributeLeft / Right
// NSLayoutAttributeLeading / Trailing (รองรับ RTL)
// NSLayoutAttributeTop / Bottom
// NSLayoutAttributeWidth / Height
// NSLayoutAttributeCenterX / CenterY
// NSLayoutAttributeFirstBaseline / LastBaseline
// NSLayoutAttributeLeftMargin / RightMargin
// NSLayoutAttributeTopMargin / BottomMargin
```

---

## 42.10 Visual Format Language (VFL)

### VFL คืออะไร?

VFL เป็น string syntax สำหรับสร้าง constraints หลายอันพร้อมกัน

```objc
UIView *view1 = [[UIView alloc] init];
UIView *view2 = [[UIView alloc] init];
view1.translatesAutoresizingMaskIntoConstraints = NO;
view2.translatesAutoresizingMaskIntoConstraints = NO;
[self.view addSubview:view1];
[self.view addSubview:view2];

NSDictionary *views = @{@"view1": view1, @"view2": view2, 
                         @"superview": self.view};
NSDictionary *metrics = @{@"margin": @20, @"spacing": @10};

// Horizontal: |-(margin)-[view1(100)]-(spacing)-[view2]-(margin)-|
// |-       = superview leading edge
// -|       = superview trailing edge
// (100)    = width = 100
// -(20)-   = spacing = 20
NSArray *horizontalConstraints = 
    [NSLayoutConstraint constraintsWithVisualFormat:
        @"H:|-(margin)-[view1(100)]-(spacing)-[view2]-(margin)-|"
                                            options:0
                                            metrics:metrics
                                              views:views];

// Vertical
NSArray *verticalConstraints = 
    [NSLayoutConstraint constraintsWithVisualFormat:
        @"V:|-20-[view1(44)]"
                                            options:0
                                            metrics:nil
                                              views:views];

[NSLayoutConstraint activateConstraints:horizontalConstraints];
[NSLayoutConstraint activateConstraints:verticalConstraints];

// Options สำหรับ alignment
// NSLayoutFormatAlignAllLeft    - จัดชิดซ้ายให้ views ทั้งหมด
// NSLayoutFormatAlignAllCenterY - จัด center Y ให้ views ทั้งหมด
// NSLayoutFormatDirectionLeadingToTrailing - จาก leading ไป trailing
```

---

## 42.11 UIStackView

### UIStackView คืออะไร?

`UIStackView` จัดการ subviews แบบ linear (แนวนอนหรือแนวตั้ง) โดยอัตโนมัติ ลด constraints ที่ต้องเขียนมาก

```objc
// Horizontal Stack
UIStackView *hStack = [[UIStackView alloc] init];
hStack.translatesAutoresizingMaskIntoConstraints = NO;
hStack.axis = UILayoutConstraintAxisHorizontal;
hStack.distribution = UIStackViewDistributionFillEqually;
hStack.alignment = UIStackViewAlignmentCenter;
hStack.spacing = 10;
[self.view addSubview:hStack];

// Vertical Stack
UIStackView *vStack = [[UIStackView alloc] init];
vStack.axis = UILayoutConstraintAxisVertical;
vStack.distribution = UIStackViewDistributionFill;
vStack.alignment = UIStackViewAlignmentFill;
vStack.spacing = 16;

// เพิ่ม views
[hStack addArrangedSubview:button1];
[hStack addArrangedSubview:button2];
[hStack addArrangedSubview:button3];

// ลบ view
[hStack removeArrangedSubview:button2];
[button2 removeFromSuperview]; // ต้องทำด้วย

// แทรก view
[hStack insertArrangedSubview:newButton atIndex:1];

// ซ่อน view (StackView จะปรับ layout อัตโนมัติ)
button1.hidden = YES; // StackView จัดใหม่เอง

// Distribution Types:
// UIStackViewDistributionFill              - View แรกขยายเต็ม
// UIStackViewDistributionFillEqually       - ทุก view เท่ากัน
// UIStackViewDistributionFillProportionally- ตาม intrinsicContentSize
// UIStackViewDistributionEqualSpacing      - spacing เท่ากัน
// UIStackViewDistributionEqualCentering    - center เท่ากัน

// Alignment Types (Horizontal Stack):
// UIStackViewAlignmentFill          - ขยายตาม cross axis
// UIStackViewAlignmentLeading       - ชิดบน
// UIStackViewAlignmentCenter        - กึ่งกลาง
// UIStackViewAlignmentTrailing      - ชิดล่าง
// UIStackViewAlignmentFirstBaseline - จาก baseline แรก
// UIStackViewAlignmentLastBaseline  - จาก baseline สุดท้าย

// Nested Stack Views
UIStackView *outerStack = [[UIStackView alloc] init];
outerStack.axis = UILayoutConstraintAxisVertical;
outerStack.spacing = 16;

UIStackView *row1 = [[UIStackView alloc] init];
row1.axis = UILayoutConstraintAxisHorizontal;
row1.spacing = 8;

[row1 addArrangedSubview:label1];
[row1 addArrangedSubview:textField1];

[outerStack addArrangedSubview:row1];
[outerStack addArrangedSubview:row2];

// ตัวอย่าง Login Form ด้วย StackView
- (UIStackView *)createLoginForm {
    UIStackView *formStack = [[UIStackView alloc] init];
    formStack.axis = UILayoutConstraintAxisVertical;
    formStack.spacing = 16;
    formStack.translatesAutoresizingMaskIntoConstraints = NO;
    
    // Email Row
    UIStackView *emailRow = [[UIStackView alloc] init];
    emailRow.axis = UILayoutConstraintAxisVertical;
    emailRow.spacing = 4;
    
    UILabel *emailLabel = [[UILabel alloc] init];
    emailLabel.text = @"อีเมล";
    emailLabel.font = [UIFont systemFontOfSize:14];
    emailLabel.textColor = [UIColor secondaryLabelColor];
    
    UITextField *emailField = [[UITextField alloc] init];
    emailField.placeholder = @"your@email.com";
    emailField.borderStyle = UITextBorderStyleRoundedRect;
    emailField.keyboardType = UIKeyboardTypeEmailAddress;
    [emailField heightAnchor].constant = 44; // ผ่าน constraint
    
    [emailRow addArrangedSubview:emailLabel];
    [emailRow addArrangedSubview:emailField];
    
    // Password Row (similar structure)
    UIStackView *passwordRow = [self createFieldWithLabel:@"รหัสผ่าน" 
                                              placeholder:@"••••••••" 
                                                  secure:YES];
    
    // Login Button
    UIButton *loginButton = [UIButton buttonWithType:UIButtonTypeSystem];
    [loginButton setTitle:@"เข้าสู่ระบบ" forState:UIControlStateNormal];
    loginButton.backgroundColor = [UIColor systemBlueColor];
    [loginButton setTitleColor:[UIColor whiteColor] 
                      forState:UIControlStateNormal];
    loginButton.layer.cornerRadius = 10;
    loginButton.heightAnchor.constant = 44;
    
    [formStack addArrangedSubview:emailRow];
    [formStack addArrangedSubview:passwordRow];
    [formStack addArrangedSubview:loginButton];
    
    return formStack;
}
```

---

## 42.12 Target-Action Pattern

### Target-Action คืออะไร?

Target-Action เป็น pattern การสื่อสารระหว่าง Control (เช่น UIButton) กับ Handler (Controller) โดยไม่ต้องใช้ delegation

```objc
// เพิ่ม Action
[button addTarget:self                              // target
           action:@selector(buttonTapped:)         // action selector
 forControlEvents:UIControlEventTouchUpInside];    // event

// Events ที่รองรับ:
// UIControlEventTouchDown           - กดค้าง
// UIControlEventTouchUpInside       - กดแล้วปล่อยในขอบเขต (ที่ใช้บ่อย)
// UIControlEventTouchUpOutside      - ปล่อยนอกขอบเขต
// UIControlEventTouchDragInside     - ลากอยู่ภายใน
// UIControlEventTouchDragOutside    - ลากออกไปนอก
// UIControlEventTouchDragEnter      - ลากกลับเข้ามา
// UIControlEventTouchDragExit       - ลากออกไป
// UIControlEventValueChanged        - ค่าเปลี่ยน (slider, switch)
// UIControlEventEditingChanged      - text เปลี่ยน (textfield)
// UIControlEventEditingDidBegin     - เริ่มแก้ไข
// UIControlEventEditingDidEnd       - หยุดแก้ไข
// UIControlEventEditingDidEndOnExit - กด Return
// UIControlEventAllEvents           - ทุก events

// Handler Signatures:
// ไม่มี parameter
- (void)buttonTapped { }
// มี sender
- (void)buttonTapped:(UIButton *)sender { }
// มี sender และ event
- (void)buttonTapped:(UIButton *)sender forEvent:(UIEvent *)event { }

// ลบ action
[button removeTarget:self 
              action:@selector(buttonTapped:)
    forControlEvents:UIControlEventTouchUpInside];

// ลบทุก actions
[button removeTarget:nil 
              action:nil
    forControlEvents:UIControlEventAllEvents];

// ดู targets ทั้งหมด
NSSet *targets = [button allTargets];
for (id target in targets) {
    NSArray *actions = [button actionsForTarget:target 
                                forControlEvent:UIControlEventTouchUpInside];
    NSLog(@"Target: %@, Actions: %@", target, actions);
}
```

---

## 42.13 Practice: Simple Calculator UI

### สร้าง Calculator UI แบบ Programmatic

```objc
// CalculatorViewController.h
#import <UIKit/UIKit.h>

@interface CalculatorViewController : UIViewController
@end
```

```objc
// CalculatorViewController.m
#import "CalculatorViewController.h"

@interface CalculatorViewController ()

@property (nonatomic, strong) UILabel *displayLabel;
@property (nonatomic, strong) UILabel *expressionLabel;
@property (nonatomic, strong) UIStackView *buttonGrid;

@property (nonatomic, strong) NSString *currentInput;
@property (nonatomic, strong) NSString *previousInput;
@property (nonatomic, strong) NSString *currentOperation;
@property (nonatomic, assign) BOOL shouldResetDisplay;

@end

@implementation CalculatorViewController

- (void)viewDidLoad {
    [super viewDidLoad];
    self.view.backgroundColor = [UIColor blackColor];
    self.title = @"เครื่องคิดเลข";
    
    self.currentInput = @"0";
    self.shouldResetDisplay = NO;
    
    [self setupDisplayArea];
    [self setupButtonGrid];
    [self setupConstraints];
}

- (void)setupDisplayArea {
    // Expression Label (แสดงนิพจน์)
    self.expressionLabel = [[UILabel alloc] init];
    self.expressionLabel.translatesAutoresizingMaskIntoConstraints = NO;
    self.expressionLabel.text = @"";
    self.expressionLabel.textColor = [UIColor colorWithWhite:0.7 alpha:1.0];
    self.expressionLabel.font = [UIFont systemFontOfSize:20];
    self.expressionLabel.textAlignment = NSTextAlignmentRight;
    [self.view addSubview:self.expressionLabel];
    
    // Display Label (แสดงผลลัพธ์)
    self.displayLabel = [[UILabel alloc] init];
    self.displayLabel.translatesAutoresizingMaskIntoConstraints = NO;
    self.displayLabel.text = @"0";
    self.displayLabel.textColor = [UIColor whiteColor];
    self.displayLabel.font = [UIFont systemFontOfSize:72 weight:UIFontWeightLight];
    self.displayLabel.textAlignment = NSTextAlignmentRight;
    self.displayLabel.adjustsFontSizeToFitWidth = YES;
    self.displayLabel.minimumScaleFactor = 0.3;
    [self.view addSubview:self.displayLabel];
}

- (void)setupButtonGrid {
    // Button Layout (เหมือน iOS Calculator)
    NSArray *buttonTitles = @[
        @"AC", @"+/-", @"%", @"÷",
        @"7",  @"8",   @"9", @"×",
        @"4",  @"5",   @"6", @"−",
        @"1",  @"2",   @"3", @"+",
        @"0",  @"",    @".", @"="   // 0 จะกว้างเป็น 2 column
    ];
    
    self.buttonGrid = [[UIStackView alloc] init];
    self.buttonGrid.translatesAutoresizingMaskIntoConstraints = NO;
    self.buttonGrid.axis = UILayoutConstraintAxisVertical;
    self.buttonGrid.spacing = 12;
    self.buttonGrid.distribution = UIStackViewDistributionFillEqually;
    [self.view addSubview:self.buttonGrid];
    
    NSInteger rows = 5;
    NSInteger cols = 4;
    
    for (int row = 0; row < rows; row++) {
        UIStackView *rowStack = [[UIStackView alloc] init];
        rowStack.axis = UILayoutConstraintAxisHorizontal;
        rowStack.spacing = 12;
        rowStack.distribution = UIStackViewDistributionFillEqually;
        
        for (int col = 0; col < cols; col++) {
            NSInteger index = row * cols + col;
            if (index >= buttonTitles.count) break;
            
            NSString *title = buttonTitles[index];
            
            // Row 5: 0 กว้างเป็น 2 columns
            if (row == 4 && col == 0) {
                UIButton *zeroButton = [self createButtonWithTitle:@"0"];
                zeroButton.contentHorizontalAlignment = UIControlContentHorizontalAlignmentLeft;
                zeroButton.contentEdgeInsets = UIEdgeInsetsMake(0, 28, 0, 0);
                [rowStack addArrangedSubview:zeroButton];
                
                // Skip col 1 (empty)
                UIView *spacer = [[UIView alloc] init];
                spacer.hidden = YES;
                [rowStack addArrangedSubview:spacer];
                col++; // skip next
                continue;
            }
            
            if (title.length == 0) {
                UIView *spacer = [[UIView alloc] init];
                [rowStack addArrangedSubview:spacer];
                continue;
            }
            
            UIButton *button = [self createButtonWithTitle:title];
            [rowStack addArrangedSubview:button];
        }
        
        [self.buttonGrid addArrangedSubview:rowStack];
    }
}

- (UIButton *)createButtonWithTitle:(NSString *)title {
    UIButton *button = [UIButton buttonWithType:UIButtonTypeCustom];
    [button setTitle:title forState:UIControlStateNormal];
    button.titleLabel.font = [UIFont systemFontOfSize:30 weight:UIFontWeightRegular];
    button.layer.cornerRadius = 40;
    button.clipsToBounds = YES;
    
    // สีปุ่มตาม type
    if ([title isEqualToString:@"AC"] || 
        [title isEqualToString:@"+/-"] ||
        [title isEqualToString:@"%"]) {
        // Function buttons (gray)
        button.backgroundColor = [UIColor colorWithRed:0.6 green:0.6 blue:0.6 alpha:1.0];
        [button setTitleColor:[UIColor blackColor] forState:UIControlStateNormal];
    } else if ([title isEqualToString:@"÷"] ||
               [title isEqualToString:@"×"] ||
               [title isEqualToString:@"−"] ||
               [title isEqualToString:@"+"] ||
               [title isEqualToString:@"="]) {
        // Operator buttons (orange)
        button.backgroundColor = [UIColor systemOrangeColor];
        [button setTitleColor:[UIColor whiteColor] forState:UIControlStateNormal];
    } else {
        // Number buttons (dark gray)
        button.backgroundColor = [UIColor colorWithRed:0.2 green:0.2 blue:0.2 alpha:1.0];
        [button setTitleColor:[UIColor whiteColor] forState:UIControlStateNormal];
    }
    
    // Highlight state
    [button addTarget:self 
               action:@selector(buttonHighlighted:)
     forControlEvents:UIControlEventTouchDown];
    
    // Action
    [button addTarget:self 
               action:@selector(buttonTapped:)
     forControlEvents:UIControlEventTouchUpInside];
    
    return button;
}

- (void)setupConstraints {
    UILayoutGuide *safeArea = self.view.safeAreaLayoutGuide;
    
    [NSLayoutConstraint activateConstraints:@[
        // Expression Label
        [self.expressionLabel.bottomAnchor 
            constraintEqualToAnchor:self.displayLabel.topAnchor constant:-4],
        [self.expressionLabel.leadingAnchor 
            constraintEqualToAnchor:self.view.leadingAnchor constant:20],
        [self.expressionLabel.trailingAnchor 
            constraintEqualToAnchor:self.view.trailingAnchor constant:-20],
        
        // Display Label
        [self.displayLabel.bottomAnchor 
            constraintEqualToAnchor:self.buttonGrid.topAnchor constant:-20],
        [self.displayLabel.leadingAnchor 
            constraintEqualToAnchor:self.view.leadingAnchor constant:20],
        [self.displayLabel.trailingAnchor 
            constraintEqualToAnchor:self.view.trailingAnchor constant:-20],
        
        // Button Grid
        [self.buttonGrid.leadingAnchor 
            constraintEqualToAnchor:self.view.leadingAnchor constant:20],
        [self.buttonGrid.trailingAnchor 
            constraintEqualToAnchor:self.view.trailingAnchor constant:-20],
        [self.buttonGrid.bottomAnchor 
            constraintEqualToAnchor:safeArea.bottomAnchor constant:-20],
        [self.buttonGrid.heightAnchor 
            constraintEqualToConstant:5 * 80 + 4 * 12], // 5 rows * 80 + 4 gaps
    ]];
}

#pragma mark - Button Actions

- (void)buttonHighlighted:(UIButton *)sender {
    // ให้ feedback เมื่อกด
    [UIView animateWithDuration:0.1 animations:^{
        sender.alpha = 0.7;
    }];
}

- (void)buttonTapped:(UIButton *)sender {
    [UIView animateWithDuration:0.1 animations:^{
        sender.alpha = 1.0;
    }];
    
    NSString *title = sender.currentTitle;
    
    if ([title isEqualToString:@"AC"]) {
        [self clearAll];
    } else if ([title isEqualToString:@"+/-"]) {
        [self toggleSign];
    } else if ([title isEqualToString:@"%"]) {
        [self percentage];
    } else if ([title isEqualToString:@"÷"] ||
               [title isEqualToString:@"×"] ||
               [title isEqualToString:@"−"] ||
               [title isEqualToString:@"+"]) {
        [self setOperation:title];
    } else if ([title isEqualToString:@"="]) {
        [self calculate];
    } else if ([title isEqualToString:@"."]) {
        [self addDecimalPoint];
    } else {
        [self appendDigit:title];
    }
}

- (void)clearAll {
    self.currentInput = @"0";
    self.previousInput = nil;
    self.currentOperation = nil;
    self.shouldResetDisplay = NO;
    self.displayLabel.text = @"0";
    self.expressionLabel.text = @"";
}

- (void)appendDigit:(NSString *)digit {
    if (self.shouldResetDisplay) {
        self.currentInput = digit;
        self.shouldResetDisplay = NO;
    } else if ([self.currentInput isEqualToString:@"0"]) {
        self.currentInput = digit;
    } else {
        self.currentInput = [self.currentInput stringByAppendingString:digit];
    }
    self.displayLabel.text = self.currentInput;
}

- (void)addDecimalPoint {
    if ([self.currentInput containsString:@"."]) return;
    self.currentInput = [self.currentInput stringByAppendingString:@"."];
    self.displayLabel.text = self.currentInput;
}

- (void)setOperation:(NSString *)operation {
    self.previousInput = self.currentInput;
    self.currentOperation = operation;
    self.shouldResetDisplay = YES;
    self.expressionLabel.text = [NSString stringWithFormat:@"%@ %@",
                                 self.previousInput, operation];
}

- (void)calculate {
    if (!self.previousInput || !self.currentOperation) return;
    
    double a = [self.previousInput doubleValue];
    double b = [self.currentInput doubleValue];
    double result = 0;
    
    if ([self.currentOperation isEqualToString:@"+"]) result = a + b;
    else if ([self.currentOperation isEqualToString:@"−"]) result = a - b;
    else if ([self.currentOperation isEqualToString:@"×"]) result = a * b;
    else if ([self.currentOperation isEqualToString:@"÷"]) {
        if (b == 0) {
            self.displayLabel.text = @"Error";
            return;
        }
        result = a / b;
    }
    
    self.expressionLabel.text = [NSString stringWithFormat:@"%@ %@ %@ =",
                                 self.previousInput, self.currentOperation, self.currentInput];
    
    // Format result
    if (result == (long long)result) {
        self.currentInput = [NSString stringWithFormat:@"%lld", (long long)result];
    } else {
        self.currentInput = [NSString stringWithFormat:@"%.10g", result];
    }
    
    self.displayLabel.text = self.currentInput;
    self.previousInput = nil;
    self.currentOperation = nil;
    self.shouldResetDisplay = YES;
}

- (void)toggleSign {
    double value = [self.currentInput doubleValue];
    value = -value;
    if (value == (long long)value) {
        self.currentInput = [NSString stringWithFormat:@"%lld", (long long)value];
    } else {
        self.currentInput = [NSString stringWithFormat:@"%g", value];
    }
    self.displayLabel.text = self.currentInput;
}

- (void)percentage {
    double value = [self.currentInput doubleValue];
    value = value / 100.0;
    self.currentInput = [NSString stringWithFormat:@"%g", value];
    self.displayLabel.text = self.currentInput;
}

@end
```

---

## 42.14 สรุปและแบบฝึกหัด

### สรุปหัวข้อที่เรียน

1. **UIKit Framework Overview** - ภาพรวมของ UIKit
2. **UIView Hierarchy** - โครงสร้างแบบ tree ของ views
3. **UIWindow** - Root container ของ UI
4. **UI Controls** - UILabel, UIButton, UITextField, UITextView, UIImageView, UISwitch, UISlider, UISegmentedControl, UIProgressView
5. **Programmatic vs IB** - ข้อดีข้อเสีย
6. **Auto Layout** - ระบบ constraint-based layout
7. **NSLayoutConstraint** - สร้าง constraints โดยตรง
8. **VFL** - Visual Format Language
9. **UIStackView** - จัดการ layout แบบ stack
10. **Target-Action** - Pattern การสื่อสาร Control-Handler
11. **Calculator UI** - ตัวอย่างจริง

### แบบฝึกหัด

```
แบบฝึกหัดที่ 1: Profile Card
สร้าง View ที่มี:
- UIImageView สำหรับ avatar (ทำเป็น circle)
- UILabel สำหรับชื่อ (bold, ใหญ่)
- UILabel สำหรับ bio (หลายบรรทัด, สีรอง)
- UIButton สำหรับ Follow
- UIStackView สำหรับ stats (Posts, Followers, Following)
ทั้งหมดใช้ Auto Layout

แบบฝึกหัดที่ 2: Settings Screen
สร้างหน้าตั้งค่าที่มี:
- UISwitch สำหรับ Dark Mode
- UISlider สำหรับ Font Size (พร้อม UILabel แสดงค่า)
- UISegmentedControl สำหรับ Theme (Light/Dark/Auto)
- UITextField สำหรับ Username
- UIButton สำหรับ Save

แบบฝึกหัดที่ 3: ปรับปรุง Calculator
- เพิ่มปุ่ม Memory (M+, M-, MR, MC)
- เพิ่มการแสดงประวัติการคำนวณ (UITextView)
- เพิ่ม haptic feedback เมื่อกดปุ่ม
- รองรับ Landscape Mode (เพิ่มปุ่ม scientific)
```

---

*ตอนที่ 42 จบแล้ว - ไปต่อที่ตอนที่ 43: UIViewController*
