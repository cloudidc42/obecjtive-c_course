# Part 49: Auto Layout ใน Objective-C

## บทนำ (Introduction)

Auto Layout เป็นระบบจัดการ Layout ที่ทรงพลังของ iOS และ macOS ซึ่งช่วยให้แอปพลิเคชันของเราสามารถปรับขนาดและตำแหน่งของ UI elements ได้อย่างอัตโนมัติตามขนาดหน้าจอ ทิศทาง (orientation) และการตั้งค่าต่างๆ ของผู้ใช้

Auto Layout ใช้ระบบ **Constraints** (ข้อจำกัด) เพื่อกำหนดความสัมพันธ์ระหว่าง UI elements และ superview หรือระหว่าง elements ด้วยกันเอง

---

## 49.1 แนวคิด Auto Layout (Auto Layout Concepts)

### ทำไมต้องใช้ Auto Layout?

ก่อนยุค Auto Layout นักพัฒนาต้องกำหนดขนาดและตำแหน่งของ views ด้วย `frame` แบบ hardcode ซึ่งทำให้เกิดปัญหาเมื่อ:
- หน้าจอมีขนาดต่างกัน (iPhone SE vs iPhone 15 Pro Max)
- หมุนหน้าจอ (Portrait vs Landscape)
- ผู้ใช้เปลี่ยนขนาด font (Dynamic Type)
- รองรับ iPad Split View

Auto Layout แก้ปัญหาเหล่านี้โดยใช้ระบบ **constraint-based layout** ที่คำนวณขนาดและตำแหน่งของ views ในขณะ runtime

### หลักการพื้นฐาน

1. **Constraints** คือกฎที่กำหนดความสัมพันธ์ระหว่าง attributes ของ views
2. ทุก view ต้องมี constraints เพียงพอที่จะกำหนด **position** (x, y) และ **size** (width, height)
3. Constraints ที่ขัดแย้งกัน (conflicting) จะทำให้เกิด runtime warning
4. Constraints ที่ไม่เพียงพอ (ambiguous) จะทำให้ layout ไม่แน่นอน

```objc
// ตัวอย่างพื้นฐาน - การ disable autoresizing mask
UIView *myView = [[UIView alloc] init];
myView.translatesAutoresizingMaskIntoConstraints = NO; // สำคัญมาก!
myView.backgroundColor = [UIColor blueColor];
[self.view addSubview:myView];
```

> **สำคัญ**: เมื่อใช้ Auto Layout แบบ programmatic ต้องตั้ง `translatesAutoresizingMaskIntoConstraints = NO` เสมอ มิฉะนั้น system จะสร้าง constraints อัตโนมัติจาก frame ซึ่งอาจขัดแย้งกับ constraints ที่เราสร้าง

---

## 49.2 Constraints คืออะไร (What are Constraints?)

Constraint คือสมการเชิงเส้น (linear equation) ที่มีรูปแบบ:

```
view1.attribute = multiplier × view2.attribute + constant
```

ตัวอย่างเช่น:
- `button.leading = superview.leading + 20` (ระยะห่างจากขอบซ้าย 20 points)
- `label.width = 0.5 × superview.width` (กว้างครึ่งหนึ่งของ superview)
- `imageView.centerX = superview.centerX` (อยู่กลางแนวนอน)

### NSLayoutAttribute

Attributes ที่ใช้ใน constraints:

```objc
typedef NS_ENUM(NSInteger, NSLayoutAttribute) {
    NSLayoutAttributeLeft = 1,       // ขอบซ้าย
    NSLayoutAttributeRight,          // ขอบขวา
    NSLayoutAttributeTop,            // ขอบบน
    NSLayoutAttributeBottom,         // ขอบล่าง
    NSLayoutAttributeLeading,        // ด้านนำ (ซ้ายใน LTR, ขวาใน RTL)
    NSLayoutAttributeTrailing,       // ด้านตาม (ขวาใน LTR, ซ้ายใน RTL)
    NSLayoutAttributeWidth,          // ความกว้าง
    NSLayoutAttributeHeight,         // ความสูง
    NSLayoutAttributeCenterX,        // จุดกึ่งกลางแนวนอน
    NSLayoutAttributeCenterY,        // จุดกึ่งกลางแนวตั้ง
    NSLayoutAttributeLastBaseline,   // เส้น baseline ล่างสุด
    NSLayoutAttributeFirstBaseline,  // เส้น baseline แรก
    NSLayoutAttributeNotAnAttribute  // ใช้เมื่อไม่มี second item
};
```

---

## 49.3 NSLayoutConstraint - การสร้าง Constraints แบบ Classic

### รูปแบบพื้นฐาน

```objc
// NSLayoutConstraint constraintWithItem:attribute:relatedBy:toItem:attribute:multiplier:constant:
NSLayoutConstraint *constraint = [NSLayoutConstraint 
    constraintWithItem:view1
    attribute:NSLayoutAttributeLeading
    relatedBy:NSLayoutRelationEqual
    toItem:view2
    attribute:NSLayoutAttributeLeading
    multiplier:1.0
    constant:20.0];

[self.view addConstraint:constraint];
```

### ตัวอย่างครบถ้วน - การจัด View แบบ Classic

```objc
#import "ViewController.h"

@interface ViewController ()
@property (nonatomic, strong) UIView *redBox;
@property (nonatomic, strong) UIView *blueBox;
@end

@implementation ViewController

- (void)viewDidLoad {
    [super viewDidLoad];
    
    // สร้าง Red Box
    self.redBox = [[UIView alloc] init];
    self.redBox.backgroundColor = [UIColor redColor];
    self.redBox.translatesAutoresizingMaskIntoConstraints = NO;
    [self.view addSubview:self.redBox];
    
    // สร้าง Blue Box
    self.blueBox = [[UIView alloc] init];
    self.blueBox.backgroundColor = [UIColor blueColor];
    self.blueBox.translatesAutoresizingMaskIntoConstraints = NO;
    [self.view addSubview:self.blueBox];
    
    [self setupConstraints];
}

- (void)setupConstraints {
    // Red Box constraints
    // Leading (ซ้าย)
    [self.view addConstraint:[NSLayoutConstraint 
        constraintWithItem:self.redBox
        attribute:NSLayoutAttributeLeading
        relatedBy:NSLayoutRelationEqual
        toItem:self.view
        attribute:NSLayoutAttributeLeading
        multiplier:1.0
        constant:20.0]];
    
    // Top (บน)
    [self.view addConstraint:[NSLayoutConstraint 
        constraintWithItem:self.redBox
        attribute:NSLayoutAttributeTop
        relatedBy:NSLayoutRelationEqual
        toItem:self.view
        attribute:NSLayoutAttributeTop
        multiplier:1.0
        constant:100.0]];
    
    // Width
    [self.view addConstraint:[NSLayoutConstraint 
        constraintWithItem:self.redBox
        attribute:NSLayoutAttributeWidth
        relatedBy:NSLayoutRelationEqual
        toItem:nil
        attribute:NSLayoutAttributeNotAnAttribute
        multiplier:1.0
        constant:100.0]];
    
    // Height
    [self.view addConstraint:[NSLayoutConstraint 
        constraintWithItem:self.redBox
        attribute:NSLayoutAttributeHeight
        relatedBy:NSLayoutRelationEqual
        toItem:nil
        attribute:NSLayoutAttributeNotAnAttribute
        multiplier:1.0
        constant:100.0]];
    
    // Blue Box constraints - อยู่ทางขวาของ Red Box
    // Leading ของ Blue = Trailing ของ Red + 20
    [self.view addConstraint:[NSLayoutConstraint 
        constraintWithItem:self.blueBox
        attribute:NSLayoutAttributeLeading
        relatedBy:NSLayoutRelationEqual
        toItem:self.redBox
        attribute:NSLayoutAttributeTrailing
        multiplier:1.0
        constant:20.0]];
    
    // Top ตรงกับ Red Box
    [self.view addConstraint:[NSLayoutConstraint 
        constraintWithItem:self.blueBox
        attribute:NSLayoutAttributeTop
        relatedBy:NSLayoutRelationEqual
        toItem:self.redBox
        attribute:NSLayoutAttributeTop
        multiplier:1.0
        constant:0.0]];
    
    // Width เท่ากับ Red Box
    [self.view addConstraint:[NSLayoutConstraint 
        constraintWithItem:self.blueBox
        attribute:NSLayoutAttributeWidth
        relatedBy:NSLayoutRelationEqual
        toItem:self.redBox
        attribute:NSLayoutAttributeWidth
        multiplier:1.0
        constant:0.0]];
    
    // Height เท่ากับ Red Box
    [self.view addConstraint:[NSLayoutConstraint 
        constraintWithItem:self.blueBox
        attribute:NSLayoutAttributeHeight
        relatedBy:NSLayoutRelationEqual
        toItem:self.redBox
        attribute:NSLayoutAttributeHeight
        multiplier:1.0
        constant:0.0]];
}

@end
```

### การเพิ่ม Constraints แบบ Array

```objc
- (void)setupConstraintsWithArray {
    NSArray *constraints = @[
        // Leading
        [NSLayoutConstraint constraintWithItem:self.redBox
                                     attribute:NSLayoutAttributeLeading
                                     relatedBy:NSLayoutRelationEqual
                                        toItem:self.view
                                     attribute:NSLayoutAttributeLeading
                                    multiplier:1.0
                                      constant:20.0],
        // Top  
        [NSLayoutConstraint constraintWithItem:self.redBox
                                     attribute:NSLayoutAttributeTop
                                     relatedBy:NSLayoutRelationEqual
                                        toItem:self.view
                                     attribute:NSLayoutAttributeTop
                                    multiplier:1.0
                                      constant:100.0],
        // Width
        [NSLayoutConstraint constraintWithItem:self.redBox
                                     attribute:NSLayoutAttributeWidth
                                     relatedBy:NSLayoutRelationEqual
                                        toItem:nil
                                     attribute:NSLayoutAttributeNotAnAttribute
                                    multiplier:1.0
                                      constant:150.0],
        // Height
        [NSLayoutConstraint constraintWithItem:self.redBox
                                     attribute:NSLayoutAttributeHeight
                                     relatedBy:NSLayoutRelationEqual
                                        toItem:nil
                                     attribute:NSLayoutAttributeNotAnAttribute
                                    multiplier:1.0
                                      constant:150.0],
    ];
    
    [self.view addConstraints:constraints];
}
```

---

## 49.4 Visual Format Language (VFL)

VFL เป็น syntax แบบ ASCII art สำหรับสร้าง constraints แบบย่อ

```objc
- (void)setupWithVFL {
    UIView *view1 = [[UIView alloc] init];
    UIView *view2 = [[UIView alloc] init];
    view1.translatesAutoresizingMaskIntoConstraints = NO;
    view2.translatesAutoresizingMaskIntoConstraints = NO;
    view1.backgroundColor = [UIColor redColor];
    view2.backgroundColor = [UIColor blueColor];
    [self.view addSubview:view1];
    [self.view addSubview:view2];
    
    NSDictionary *views = NSDictionaryOfVariableBindings(view1, view2);
    NSDictionary *metrics = @{@"margin": @20, @"boxSize": @100};
    
    // แนวนอน: |-(margin)-[view1(boxSize)]-(margin)-[view2(boxSize)]-(margin)-|
    NSArray *hConstraints = [NSLayoutConstraint 
        constraintsWithVisualFormat:@"H:|-(margin)-[view1(boxSize)]-(margin)-[view2(boxSize)]-(margin)-|"
        options:0
        metrics:metrics
        views:views];
    
    // แนวตั้ง
    NSArray *vConstraints1 = [NSLayoutConstraint 
        constraintsWithVisualFormat:@"V:|-(100)-[view1(boxSize)]"
        options:0
        metrics:metrics
        views:views];
    
    NSArray *vConstraints2 = [NSLayoutConstraint 
        constraintsWithVisualFormat:@"V:|-(100)-[view2(boxSize)]"
        options:0
        metrics:metrics
        views:views];
    
    [self.view addConstraints:hConstraints];
    [self.view addConstraints:vConstraints1];
    [self.view addConstraints:vConstraints2];
}
```

### VFL Syntax Reference

```
H:  = แนวนอน (Horizontal)
V:  = แนวตั้ง (Vertical)
|   = edge ของ superview
-   = standard spacing (8 points)
-(20)- = spacing 20 points
[view] = ใช้ชื่อ view
[view(100)] = view กว้าง/สูง 100
[view(>=50)] = view กว้าง/สูง >= 50
[view(<=200)] = view กว้าง/สูง <= 200
[view1][view2] = ต่อกันโดยไม่มี spacing
```

---

## 49.5 Multiplier และ Constant

### Multiplier

Multiplier ใช้สร้าง proportional constraints:

```objc
// ให้ view กว้างครึ่งหนึ่งของ superview
NSLayoutConstraint *halfWidthConstraint = [NSLayoutConstraint 
    constraintWithItem:myView
    attribute:NSLayoutAttributeWidth
    relatedBy:NSLayoutRelationEqual
    toItem:self.view
    attribute:NSLayoutAttributeWidth
    multiplier:0.5  // <-- multiplier
    constant:0.0];

[self.view addConstraint:halfWidthConstraint];
```

```objc
// ตัวอย่าง: สร้าง aspect ratio 16:9
- (void)setupAspectRatio {
    UIView *videoView = [[UIView alloc] init];
    videoView.translatesAutoresizingMaskIntoConstraints = NO;
    videoView.backgroundColor = [UIColor blackColor];
    [self.view addSubview:videoView];
    
    // Leading และ Trailing กับ superview
    [NSLayoutConstraint activateConstraints:@[
        [videoView.leadingAnchor constraintEqualToAnchor:self.view.leadingAnchor],
        [videoView.trailingAnchor constraintEqualToAnchor:self.view.trailingAnchor],
        [videoView.topAnchor constraintEqualToAnchor:self.view.safeAreaLayoutGuide.topAnchor],
    ]];
    
    // Aspect ratio 16:9 โดยใช้ multiplier
    // height = width × (9/16)
    NSLayoutConstraint *aspectRatio = [NSLayoutConstraint 
        constraintWithItem:videoView
        attribute:NSLayoutAttributeHeight
        relatedBy:NSLayoutRelationEqual
        toItem:videoView
        attribute:NSLayoutAttributeWidth
        multiplier:9.0/16.0
        constant:0.0];
    [aspectRatio setActive:YES];
}
```

### Constant

Constant ใช้เพิ่มหรือลด offset:

```objc
// ตัวอย่าง: เปลี่ยน constant เพื่อ animate
@property (nonatomic, strong) NSLayoutConstraint *topConstraint;

- (void)setupAnimatableConstraint {
    UIButton *button = [UIButton buttonWithType:UIButtonTypeSystem];
    button.translatesAutoresizingMaskIntoConstraints = NO;
    [button setTitle:@"Animate" forState:UIControlStateNormal];
    [self.view addSubview:button];
    
    // เก็บ constraint ไว้เพื่อ animate ภายหลัง
    self.topConstraint = [button.topAnchor 
        constraintEqualToAnchor:self.view.safeAreaLayoutGuide.topAnchor
        constant:50.0];
    
    [NSLayoutConstraint activateConstraints:@[
        self.topConstraint,
        [button.centerXAnchor constraintEqualToAnchor:self.view.centerXAnchor],
        [button.widthAnchor constraintEqualToConstant:200],
        [button.heightAnchor constraintEqualToConstant:44],
    ]];
    
    [button addTarget:self 
               action:@selector(animateButton) 
     forControlEvents:UIControlEventTouchUpInside];
}

- (void)animateButton {
    self.topConstraint.constant = (self.topConstraint.constant == 50.0) ? 300.0 : 50.0;
    
    [UIView animateWithDuration:0.3 animations:^{
        [self.view layoutIfNeeded];
    }];
}
```

---

## 49.6 Priorities (UILayoutPriority)

Priority กำหนดความสำคัญของ constraint เมื่อมี constraints ขัดแย้งกัน

```objc
// ค่า priority มาตรฐาน
UILayoutPriorityRequired          = 1000  // ต้องปฏิบัติตามเสมอ
UILayoutPriorityDefaultHigh       = 750   // สูง
UILayoutPriorityDefaultLow        = 250   // ต่ำ
UILayoutPriorityFittingSizeLevel  = 50    // ต่ำมาก
```

### ตัวอย่างการใช้ Priority

```objc
- (void)setupPriorityExample {
    UIView *container = [[UIView alloc] init];
    container.translatesAutoresizingMaskIntoConstraints = NO;
    container.backgroundColor = [UIColor lightGrayColor];
    [self.view addSubview:container];
    
    UILabel *label = [[UILabel alloc] init];
    label.translatesAutoresizingMaskIntoConstraints = NO;
    label.text = @"This is a label";
    label.backgroundColor = [UIColor yellowColor];
    [container addSubview:label];
    
    // Container มีความกว้างที่ต้องการ (optional)
    NSLayoutConstraint *preferredWidth = [container.widthAnchor 
        constraintEqualToConstant:300];
    preferredWidth.priority = UILayoutPriorityDefaultHigh; // 750
    
    // Container มีความกว้างสูงสุด (required)
    NSLayoutConstraint *maxWidth = [container.widthAnchor 
        constraintLessThanOrEqualToConstant:250];
    maxWidth.priority = UILayoutPriorityRequired; // 1000
    
    [NSLayoutConstraint activateConstraints:@[
        preferredWidth,  // จะพยายามกว้าง 300 แต่จะแพ้ max constraint
        maxWidth,        // จะกว้างไม่เกิน 250
        [container.centerXAnchor constraintEqualToAnchor:self.view.centerXAnchor],
        [container.centerYAnchor constraintEqualToAnchor:self.view.centerYAnchor],
        [container.heightAnchor constraintEqualToConstant:100],
        
        // Label ภายใน container
        [label.centerXAnchor constraintEqualToAnchor:container.centerXAnchor],
        [label.centerYAnchor constraintEqualToAnchor:container.centerYAnchor],
    ]];
}
```

### Custom Priority

```objc
- (void)setupCustomPriority {
    UIView *view1 = [[UIView alloc] init];
    UIView *view2 = [[UIView alloc] init];
    view1.translatesAutoresizingMaskIntoConstraints = NO;
    view2.translatesAutoresizingMaskIntoConstraints = NO;
    view1.backgroundColor = [UIColor redColor];
    view2.backgroundColor = [UIColor blueColor];
    [self.view addSubview:view1];
    [self.view addSubview:view2];
    
    // ทั้งสอง views ต้องการกว้าง 200
    NSLayoutConstraint *width1 = [view1.widthAnchor constraintEqualToConstant:200];
    NSLayoutConstraint *width2 = [view2.widthAnchor constraintEqualToConstant:200];
    
    // แต่ view1 มี priority สูงกว่า จะได้พื้นที่ก่อน
    width1.priority = 900;
    width2.priority = 600;
    
    [NSLayoutConstraint activateConstraints:@[
        width1, width2,
        [view1.leadingAnchor constraintEqualToAnchor:self.view.leadingAnchor constant:10],
        [view2.leadingAnchor constraintEqualToAnchor:view1.trailingAnchor constant:10],
        [view1.topAnchor constraintEqualToAnchor:self.view.safeAreaLayoutGuide.topAnchor constant:50],
        [view2.topAnchor constraintEqualToAnchor:view1.topAnchor],
        [view1.heightAnchor constraintEqualToConstant:100],
        [view2.heightAnchor constraintEqualToConstant:100],
    ]];
}
```

---

## 49.7 Content Hugging และ Compression Resistance

สำหรับ views ที่มี intrinsic content size (เช่น UILabel, UIButton, UIImageView)

### Content Hugging Priority

**Content Hugging** กำหนดว่า view ต้องการแนบชิดกับ content มากแค่ไหน (ต่อต้านการขยาย)

- Priority สูง = ต้านทานการขยายมาก (ไม่อยากโต)
- Priority ต่ำ = ยอมขยายง่าย

### Compression Resistance Priority

**Compression Resistance** กำหนดว่า view ต้านทานการถูกบีบอัดมากแค่ไหน

- Priority สูง = ต้านทานการบีบอัดมาก (ไม่ยอมเล็กลง)
- Priority ต่ำ = ยอมถูกบีบอัดง่าย

```objc
- (void)setupContentHugging {
    UILabel *shortLabel = [[UILabel alloc] init];
    shortLabel.translatesAutoresizingMaskIntoConstraints = NO;
    shortLabel.text = @"Short";
    shortLabel.backgroundColor = [UIColor yellowColor];
    
    UILabel *longLabel = [[UILabel alloc] init];
    longLabel.translatesAutoresizingMaskIntoConstraints = NO;
    longLabel.text = @"Much longer label text here";
    longLabel.backgroundColor = [UIColor cyanColor];
    
    [self.view addSubview:shortLabel];
    [self.view addSubview:longLabel];
    
    // กำหนด Content Hugging Priority
    // shortLabel มี hugging priority สูงกว่า จะไม่ขยาย
    [shortLabel setContentHuggingPriority:UILayoutPriorityDefaultHigh 
                                  forAxis:UILayoutConstraintAxisHorizontal];
    [longLabel setContentHuggingPriority:UILayoutPriorityDefaultLow 
                                 forAxis:UILayoutConstraintAxisHorizontal];
    
    // กำหนด Compression Resistance
    // longLabel ต้านการบีบอัดมากกว่า
    [shortLabel setContentCompressionResistancePriority:UILayoutPriorityDefaultLow 
                                                forAxis:UILayoutConstraintAxisHorizontal];
    [longLabel setContentCompressionResistancePriority:UILayoutPriorityDefaultHigh 
                                               forAxis:UILayoutConstraintAxisHorizontal];
    
    [NSLayoutConstraint activateConstraints:@[
        [shortLabel.leadingAnchor constraintEqualToAnchor:self.view.leadingAnchor constant:20],
        [longLabel.leadingAnchor constraintEqualToAnchor:shortLabel.trailingAnchor constant:10],
        [longLabel.trailingAnchor constraintEqualToAnchor:self.view.trailingAnchor constant:-20],
        [shortLabel.centerYAnchor constraintEqualToAnchor:self.view.centerYAnchor],
        [longLabel.centerYAnchor constraintEqualToAnchor:shortLabel.centerYAnchor],
    ]];
}
```

### Intrinsic Content Size

Views ที่มี intrinsic content size:

```objc
- (void)explainIntrinsicContentSize {
    // UILabel - ขนาดตาม text
    UILabel *label = [[UILabel alloc] init];
    label.text = @"Hello World";
    CGSize labelSize = label.intrinsicContentSize;
    NSLog(@"Label intrinsic size: %@", NSStringFromCGSize(labelSize));
    
    // UIButton - ขนาดตาม title
    UIButton *button = [UIButton buttonWithType:UIButtonTypeSystem];
    [button setTitle:@"Tap me" forState:UIControlStateNormal];
    CGSize buttonSize = button.intrinsicContentSize;
    NSLog(@"Button intrinsic size: %@", NSStringFromCGSize(buttonSize));
    
    // UIImageView - ขนาดตาม image
    UIImageView *imageView = [[UIImageView alloc] initWithImage:[UIImage systemImageNamed:@"star"]];
    CGSize imageSize = imageView.intrinsicContentSize;
    NSLog(@"ImageView intrinsic size: %@", NSStringFromCGSize(imageSize));
    
    // Custom view ที่กำหนด intrinsic size เอง
    // ต้อง override intrinsicContentSize ใน subclass
}
```

### Custom Intrinsic Content Size

```objc
// CustomView.h
@interface CustomView : UIView
@end

// CustomView.m
@implementation CustomView

- (CGSize)intrinsicContentSize {
    // คืนค่าขนาดที่เหมาะสมของ view
    return CGSizeMake(200, 100);
}

// เรียกเมื่อ intrinsic size เปลี่ยน
- (void)updateContent {
    // ... update content ...
    [self invalidateIntrinsicContentSize]; // บอก system ว่า size เปลี่ยน
}

@end
```

---

## 49.8 UILayoutGuide

`UILayoutGuide` เป็น invisible rectangular region ที่ใช้เป็น proxy สำหรับ constraints โดยไม่ต้องสร้าง view จริง

```objc
- (void)setupWithLayoutGuide {
    UIView *topView = [[UIView alloc] init];
    topView.translatesAutoresizingMaskIntoConstraints = NO;
    topView.backgroundColor = [UIColor redColor];
    
    UIView *bottomView = [[UIView alloc] init];
    bottomView.translatesAutoresizingMaskIntoConstraints = NO;
    bottomView.backgroundColor = [UIColor blueColor];
    
    [self.view addSubview:topView];
    [self.view addSubview:bottomView];
    
    // สร้าง layout guide เพื่อใช้เป็น spacer ระหว่าง views
    UILayoutGuide *spacerGuide = [[UILayoutGuide alloc] init];
    [self.view addLayoutGuide:spacerGuide];
    
    [NSLayoutConstraint activateConstraints:@[
        // topView อยู่บน
        [topView.topAnchor constraintEqualToAnchor:self.view.safeAreaLayoutGuide.topAnchor constant:20],
        [topView.centerXAnchor constraintEqualToAnchor:self.view.centerXAnchor],
        [topView.widthAnchor constraintEqualToConstant:100],
        [topView.heightAnchor constraintEqualToConstant:100],
        
        // spacerGuide อยู่ระหว่าง topView และ bottomView
        [spacerGuide.topAnchor constraintEqualToAnchor:topView.bottomAnchor],
        [spacerGuide.bottomAnchor constraintEqualToAnchor:bottomView.topAnchor],
        [spacerGuide.heightAnchor constraintGreaterThanOrEqualToConstant:20],
        
        // bottomView อยู่ล่าง
        [bottomView.centerXAnchor constraintEqualToAnchor:self.view.centerXAnchor],
        [bottomView.widthAnchor constraintEqualToConstant:100],
        [bottomView.heightAnchor constraintEqualToConstant:100],
    ]];
}
```

### การใช้ Layout Guide เพื่อ Center Views

```objc
- (void)centerThreeViewsEqually {
    UIView *view1 = [self makeBoxWithColor:[UIColor redColor]];
    UIView *view2 = [self makeBoxWithColor:[UIColor greenColor]];
    UIView *view3 = [self makeBoxWithColor:[UIColor blueColor]];
    
    [self.view addSubview:view1];
    [self.view addSubview:view2];
    [self.view addSubview:view3];
    
    // สร้าง guides สำหรับ spacing
    UILayoutGuide *leftGuide = [[UILayoutGuide alloc] init];
    UILayoutGuide *midLeftGuide = [[UILayoutGuide alloc] init];
    UILayoutGuide *midRightGuide = [[UILayoutGuide alloc] init];
    UILayoutGuide *rightGuide = [[UILayoutGuide alloc] init];
    
    [self.view addLayoutGuide:leftGuide];
    [self.view addLayoutGuide:midLeftGuide];
    [self.view addLayoutGuide:midRightGuide];
    [self.view addLayoutGuide:rightGuide];
    
    NSArray *constraints = @[
        // Guides มีความกว้างเท่ากัน
        [leftGuide.widthAnchor constraintEqualToAnchor:midLeftGuide.widthAnchor],
        [midLeftGuide.widthAnchor constraintEqualToAnchor:midRightGuide.widthAnchor],
        [midRightGuide.widthAnchor constraintEqualToAnchor:rightGuide.widthAnchor],
        
        // Layout horizontal
        [leftGuide.leadingAnchor constraintEqualToAnchor:self.view.leadingAnchor],
        [leftGuide.trailingAnchor constraintEqualToAnchor:view1.leadingAnchor],
        [view1.trailingAnchor constraintEqualToAnchor:midLeftGuide.leadingAnchor],
        [midLeftGuide.trailingAnchor constraintEqualToAnchor:view2.leadingAnchor],
        [view2.trailingAnchor constraintEqualToAnchor:midRightGuide.leadingAnchor],
        [midRightGuide.trailingAnchor constraintEqualToAnchor:view3.leadingAnchor],
        [view3.trailingAnchor constraintEqualToAnchor:rightGuide.leadingAnchor],
        [rightGuide.trailingAnchor constraintEqualToAnchor:self.view.trailingAnchor],
        
        // Vertical
        [view1.centerYAnchor constraintEqualToAnchor:self.view.centerYAnchor],
        [view2.centerYAnchor constraintEqualToAnchor:self.view.centerYAnchor],
        [view3.centerYAnchor constraintEqualToAnchor:self.view.centerYAnchor],
        
        // Size
        [view1.widthAnchor constraintEqualToConstant:80],
        [view1.heightAnchor constraintEqualToConstant:80],
        [view2.widthAnchor constraintEqualToConstant:80],
        [view2.heightAnchor constraintEqualToConstant:80],
        [view3.widthAnchor constraintEqualToConstant:80],
        [view3.heightAnchor constraintEqualToConstant:80],
    ];
    
    [NSLayoutConstraint activateConstraints:constraints];
}

- (UIView *)makeBoxWithColor:(UIColor *)color {
    UIView *view = [[UIView alloc] init];
    view.translatesAutoresizingMaskIntoConstraints = NO;
    view.backgroundColor = color;
    return view;
}
```

---

## 49.9 Anchors API (NSLayoutAnchor) - วิธีที่ทันสมัย

`NSLayoutAnchor` API เปิดตัวใน iOS 9 เป็นวิธีที่แนะนำในปัจจุบัน เพราะอ่านง่ายกว่าและมี type safety

### ประเภทของ Anchors

```objc
// Position Anchors
view.topAnchor        // NSLayoutYAxisAnchor
view.bottomAnchor     // NSLayoutYAxisAnchor
view.leadingAnchor    // NSLayoutXAxisAnchor
view.trailingAnchor   // NSLayoutXAxisAnchor
view.leftAnchor       // NSLayoutXAxisAnchor
view.rightAnchor      // NSLayoutXAxisAnchor
view.centerXAnchor    // NSLayoutXAxisAnchor
view.centerYAnchor    // NSLayoutYAxisAnchor
view.firstBaselineAnchor  // NSLayoutYAxisAnchor
view.lastBaselineAnchor   // NSLayoutYAxisAnchor

// Dimension Anchors
view.widthAnchor      // NSLayoutDimension
view.heightAnchor     // NSLayoutDimension
```

### การสร้าง Constraints ด้วย Anchors

```objc
- (void)setupWithAnchors {
    UIView *myView = [[UIView alloc] init];
    myView.translatesAutoresizingMaskIntoConstraints = NO;
    myView.backgroundColor = [UIColor purpleColor];
    [self.view addSubview:myView];
    
    // วิธีที่ 1: ใช้ constraintEqualToAnchor:
    NSLayoutConstraint *topConstraint = [myView.topAnchor 
        constraintEqualToAnchor:self.view.safeAreaLayoutGuide.topAnchor
        constant:20.0];
    
    // วิธีที่ 2: ใช้ activateConstraints: (แนะนำ)
    [NSLayoutConstraint activateConstraints:@[
        [myView.topAnchor constraintEqualToAnchor:self.view.safeAreaLayoutGuide.topAnchor constant:20],
        [myView.leadingAnchor constraintEqualToAnchor:self.view.leadingAnchor constant:20],
        [myView.trailingAnchor constraintEqualToAnchor:self.view.trailingAnchor constant:-20],
        [myView.heightAnchor constraintEqualToConstant:200],
    ]];
}
```

### Anchor Methods

```objc
// Equal to another anchor
[view.topAnchor constraintEqualToAnchor:otherView.bottomAnchor]
[view.topAnchor constraintEqualToAnchor:otherView.bottomAnchor constant:10.0]

// Greater than or equal
[view.widthAnchor constraintGreaterThanOrEqualToConstant:100.0]
[view.widthAnchor constraintGreaterThanOrEqualToAnchor:otherView.widthAnchor]

// Less than or equal
[view.heightAnchor constraintLessThanOrEqualToConstant:200.0]

// NSLayoutDimension specific - dimension anchors รองรับ multiplier
[view.widthAnchor constraintEqualToAnchor:superview.widthAnchor multiplier:0.5]
[view.widthAnchor constraintEqualToAnchor:superview.widthAnchor multiplier:0.5 constant:10.0]
```

---

## 49.10 topAnchor, bottomAnchor, leadingAnchor, trailingAnchor

```objc
- (void)demonstratePositionAnchors {
    UIView *cardView = [[UIView alloc] init];
    cardView.translatesAutoresizingMaskIntoConstraints = NO;
    cardView.backgroundColor = [UIColor whiteColor];
    cardView.layer.cornerRadius = 12;
    cardView.layer.shadowColor = [UIColor blackColor].CGColor;
    cardView.layer.shadowOpacity = 0.2;
    cardView.layer.shadowRadius = 4;
    cardView.layer.shadowOffset = CGSizeMake(0, 2);
    [self.view addSubview:cardView];
    
    UILabel *titleLabel = [[UILabel alloc] init];
    titleLabel.translatesAutoresizingMaskIntoConstraints = NO;
    titleLabel.text = @"Card Title";
    titleLabel.font = [UIFont boldSystemFontOfSize:18];
    [cardView addSubview:titleLabel];
    
    UILabel *subtitleLabel = [[UILabel alloc] init];
    subtitleLabel.translatesAutoresizingMaskIntoConstraints = NO;
    subtitleLabel.text = @"Card subtitle goes here";
    subtitleLabel.font = [UIFont systemFontOfSize:14];
    subtitleLabel.textColor = [UIColor grayColor];
    [cardView addSubview:subtitleLabel];
    
    // Card view constraints
    [NSLayoutConstraint activateConstraints:@[
        // topAnchor - ระยะจาก safe area ด้านบน
        [cardView.topAnchor constraintEqualToAnchor:self.view.safeAreaLayoutGuide.topAnchor constant:20],
        
        // leadingAnchor - ระยะจากขอบซ้าย
        [cardView.leadingAnchor constraintEqualToAnchor:self.view.leadingAnchor constant:16],
        
        // trailingAnchor - ระยะจากขอบขวา (ใช้ค่าลบเพราะมันเป็น trailing)
        [cardView.trailingAnchor constraintEqualToAnchor:self.view.trailingAnchor constant:-16],
    ]];
    
    // Title label constraints (อยู่บนสุดของ card)
    [NSLayoutConstraint activateConstraints:@[
        [titleLabel.topAnchor constraintEqualToAnchor:cardView.topAnchor constant:16],
        [titleLabel.leadingAnchor constraintEqualToAnchor:cardView.leadingAnchor constant:16],
        [titleLabel.trailingAnchor constraintEqualToAnchor:cardView.trailingAnchor constant:-16],
    ]];
    
    // Subtitle label constraints (อยู่ใต้ title)
    [NSLayoutConstraint activateConstraints:@[
        [subtitleLabel.topAnchor constraintEqualToAnchor:titleLabel.bottomAnchor constant:8],
        [subtitleLabel.leadingAnchor constraintEqualToAnchor:cardView.leadingAnchor constant:16],
        [subtitleLabel.trailingAnchor constraintEqualToAnchor:cardView.trailingAnchor constant:-16],
        
        // bottomAnchor ของ subtitle เป็น bottom ของ card
        [subtitleLabel.bottomAnchor constraintEqualToAnchor:cardView.bottomAnchor constant:-16],
    ]];
}
```

---

## 49.11 widthAnchor และ heightAnchor

```objc
- (void)demonstrateDimensionAnchors {
    UIView *squareView = [[UIView alloc] init];
    squareView.translatesAutoresizingMaskIntoConstraints = NO;
    squareView.backgroundColor = [UIColor orangeColor];
    [self.view addSubview:squareView];
    
    UIView *halfWidthView = [[UIView alloc] init];
    halfWidthView.translatesAutoresizingMaskIntoConstraints = NO;
    halfWidthView.backgroundColor = [UIColor greenColor];
    [self.view addSubview:halfWidthView];
    
    [NSLayoutConstraint activateConstraints:@[
        // Square view - กว้างและสูง 100 points
        [squareView.centerXAnchor constraintEqualToAnchor:self.view.centerXAnchor],
        [squareView.centerYAnchor constraintEqualToAnchor:self.view.centerYAnchor constant:-100],
        [squareView.widthAnchor constraintEqualToConstant:100],  // กำหนด width
        [squareView.heightAnchor constraintEqualToAnchor:squareView.widthAnchor],  // height = width (square!)
        
        // Half width view - กว้างครึ่งหนึ่งของ superview
        [halfWidthView.centerXAnchor constraintEqualToAnchor:self.view.centerXAnchor],
        [halfWidthView.topAnchor constraintEqualToAnchor:squareView.bottomAnchor constant:20],
        [halfWidthView.widthAnchor constraintEqualToAnchor:self.view.widthAnchor multiplier:0.5],  // width = 50% of superview
        [halfWidthView.heightAnchor constraintEqualToConstant:60],
    ]];
}
```

---

## 49.12 centerXAnchor และ centerYAnchor

```objc
- (void)demonstrateCenterAnchors {
    // Center view ใน superview
    UIView *centeredView = [[UIView alloc] init];
    centeredView.translatesAutoresizingMaskIntoConstraints = NO;
    centeredView.backgroundColor = [UIColor systemPinkColor];
    [self.view addSubview:centeredView];
    
    [NSLayoutConstraint activateConstraints:@[
        // Center X และ Y ใน superview
        [centeredView.centerXAnchor constraintEqualToAnchor:self.view.centerXAnchor],
        [centeredView.centerYAnchor constraintEqualToAnchor:self.view.centerYAnchor],
        [centeredView.widthAnchor constraintEqualToConstant:150],
        [centeredView.heightAnchor constraintEqualToConstant:150],
    ]];
    
    // Center view relative to another view
    UILabel *label = [[UILabel alloc] init];
    label.translatesAutoresizingMaskIntoConstraints = NO;
    label.text = @"Center";
    label.textColor = [UIColor whiteColor];
    label.font = [UIFont boldSystemFontOfSize:20];
    [centeredView addSubview:label];
    
    [NSLayoutConstraint activateConstraints:@[
        // Label อยู่กลาง centeredView
        [label.centerXAnchor constraintEqualToAnchor:centeredView.centerXAnchor],
        [label.centerYAnchor constraintEqualToAnchor:centeredView.centerYAnchor],
    ]];
    
    // Center with offset
    UIView *offsetView = [[UIView alloc] init];
    offsetView.translatesAutoresizingMaskIntoConstraints = NO;
    offsetView.backgroundColor = [UIColor systemTealColor];
    [self.view addSubview:offsetView];
    
    [NSLayoutConstraint activateConstraints:@[
        // Center แต่เลื่อนไปทางขวา 50 points
        [offsetView.centerXAnchor constraintEqualToAnchor:self.view.centerXAnchor constant:50],
        // Center แต่เลื่อนลงมา 100 points
        [offsetView.centerYAnchor constraintEqualToAnchor:self.view.centerYAnchor constant:100],
        [offsetView.widthAnchor constraintEqualToConstant:80],
        [offsetView.heightAnchor constraintEqualToConstant:80],
    ]];
}
```

---

## 49.13 การ Activate และ Deactivate Constraints

```objc
@interface ViewController ()
@property (nonatomic, strong) NSArray<NSLayoutConstraint *> *portraitConstraints;
@property (nonatomic, strong) NSArray<NSLayoutConstraint *> *landscapeConstraints;
@property (nonatomic, strong) UIView *adaptiveView;
@end

@implementation ViewController

- (void)viewDidLoad {
    [super viewDidLoad];
    
    self.adaptiveView = [[UIView alloc] init];
    self.adaptiveView.translatesAutoresizingMaskIntoConstraints = NO;
    self.adaptiveView.backgroundColor = [UIColor systemIndigoColor];
    [self.view addSubview:self.adaptiveView];
    
    [self setupAdaptiveConstraints];
    [self updateConstraintsForOrientation];
}

- (void)setupAdaptiveConstraints {
    // Constraints สำหรับ portrait
    self.portraitConstraints = @[
        [self.adaptiveView.topAnchor constraintEqualToAnchor:self.view.safeAreaLayoutGuide.topAnchor constant:20],
        [self.adaptiveView.leadingAnchor constraintEqualToAnchor:self.view.leadingAnchor constant:20],
        [self.adaptiveView.trailingAnchor constraintEqualToAnchor:self.view.trailingAnchor constant:-20],
        [self.adaptiveView.heightAnchor constraintEqualToConstant:200],
    ];
    
    // Constraints สำหรับ landscape
    self.landscapeConstraints = @[
        [self.adaptiveView.topAnchor constraintEqualToAnchor:self.view.safeAreaLayoutGuide.topAnchor constant:10],
        [self.adaptiveView.leadingAnchor constraintEqualToAnchor:self.view.leadingAnchor constant:20],
        [self.adaptiveView.widthAnchor constraintEqualToAnchor:self.view.widthAnchor multiplier:0.5 constant:-30],
        [self.adaptiveView.bottomAnchor constraintEqualToAnchor:self.view.safeAreaLayoutGuide.bottomAnchor constant:-10],
    ];
}

- (void)updateConstraintsForOrientation {
    BOOL isPortrait = UIDevice.currentDevice.orientation == UIDeviceOrientationPortrait ||
                      UIDevice.currentDevice.orientation == UIDeviceOrientationPortraitUpsideDown;
    
    if (isPortrait) {
        // Deactivate landscape, activate portrait
        [NSLayoutConstraint deactivateConstraints:self.landscapeConstraints];
        [NSLayoutConstraint activateConstraints:self.portraitConstraints];
    } else {
        // Deactivate portrait, activate landscape
        [NSLayoutConstraint deactivateConstraints:self.portraitConstraints];
        [NSLayoutConstraint activateConstraints:self.landscapeConstraints];
    }
    
    [UIView animateWithDuration:0.3 animations:^{
        [self.view layoutIfNeeded];
    }];
}

- (void)viewWillTransitionToSize:(CGSize)size 
       withTransitionCoordinator:(id<UIViewControllerTransitionCoordinator>)coordinator {
    [super viewWillTransitionToSize:size withTransitionCoordinator:coordinator];
    
    [coordinator animateAlongsideTransition:^(id<UIViewControllerTransitionCoordinatorContext> context) {
        [self updateConstraintsForOrientation];
    } completion:nil];
}

@end
```

### การ Activate/Deactivate แบบ Individual

```objc
@property (nonatomic, strong) NSLayoutConstraint *expandedHeightConstraint;
@property (nonatomic, strong) NSLayoutConstraint *collapsedHeightConstraint;

- (void)toggleHeight {
    if (self.expandedHeightConstraint.isActive) {
        // ยุบลง
        self.expandedHeightConstraint.active = NO;
        self.collapsedHeightConstraint.active = YES;
    } else {
        // ขยาย
        self.collapsedHeightConstraint.active = NO;
        self.expandedHeightConstraint.active = YES;
    }
    
    [UIView animateWithDuration:0.25 animations:^{
        [self.view layoutIfNeeded];
    }];
}
```

---

## 49.14 การ Animate Constraint Changes

Animation ของ Auto Layout ทำได้โดยเปลี่ยน constraint แล้วเรียก `layoutIfNeeded` ภายใน animation block

```objc
@interface AnimationViewController ()
@property (nonatomic, strong) UIView *animatedView;
@property (nonatomic, strong) NSLayoutConstraint *leftConstraint;
@property (nonatomic, strong) NSLayoutConstraint *topConstraint;
@property (nonatomic, strong) NSLayoutConstraint *widthConstraint;
@property (nonatomic, strong) NSLayoutConstraint *heightConstraint;
@end

@implementation AnimationViewController

- (void)viewDidLoad {
    [super viewDidLoad];
    self.view.backgroundColor = [UIColor systemBackgroundColor];
    
    self.animatedView = [[UIView alloc] init];
    self.animatedView.translatesAutoresizingMaskIntoConstraints = NO;
    self.animatedView.backgroundColor = [UIColor systemPurpleColor];
    self.animatedView.layer.cornerRadius = 10;
    [self.view addSubview:self.animatedView];
    
    // สร้าง constraints และเก็บไว้เพื่อแก้ไขทีหลัง
    self.leftConstraint = [self.animatedView.leadingAnchor 
        constraintEqualToAnchor:self.view.leadingAnchor constant:20];
    self.topConstraint = [self.animatedView.topAnchor 
        constraintEqualToAnchor:self.view.safeAreaLayoutGuide.topAnchor constant:50];
    self.widthConstraint = [self.animatedView.widthAnchor constraintEqualToConstant:100];
    self.heightConstraint = [self.animatedView.heightAnchor constraintEqualToConstant:100];
    
    [NSLayoutConstraint activateConstraints:@[
        self.leftConstraint,
        self.topConstraint,
        self.widthConstraint,
        self.heightConstraint,
    ]];
    
    // Add tap gesture
    UITapGestureRecognizer *tap = [[UITapGestureRecognizer alloc] 
        initWithTarget:self action:@selector(handleTap:)];
    [self.animatedView addGestureRecognizer:tap];
}

- (void)handleTap:(UITapGestureRecognizer *)recognizer {
    BOOL isSmall = (self.widthConstraint.constant == 100);
    
    if (isSmall) {
        // ขยาย
        self.leftConstraint.constant = 0;
        self.topConstraint.constant = 0;
        self.widthConstraint.constant = self.view.bounds.width;
        self.heightConstraint.constant = 300;
        self.animatedView.layer.cornerRadius = 0;
    } else {
        // ยุบ
        self.leftConstraint.constant = 20;
        self.topConstraint.constant = 50;
        self.widthConstraint.constant = 100;
        self.heightConstraint.constant = 100;
        self.animatedView.layer.cornerRadius = 10;
    }
    
    // Animation ด้วย spring
    [UIView animateWithDuration:0.5
                          delay:0
         usingSpringWithDamping:0.7
          initialSpringVelocity:0.3
                        options:UIViewAnimationOptionCurveEaseInOut
                     animations:^{
        [self.view layoutIfNeeded];
    } completion:nil];
}

@end
```

### การ Animate แบบ Toggle

```objc
- (void)animateConstraintToggle {
    // เปลี่ยน constant แล้ว animate
    self.centerYConstraint.constant = (self.centerYConstraint.constant == 0) ? -100 : 0;
    
    // ต้องเรียก layoutIfNeeded ภายใน animation block
    [UIView animateWithDuration:0.3 
                     animations:^{
        [self.view layoutIfNeeded];  // <-- สำคัญ!
    }];
}
```

---

## 49.15 Debugging Auto Layout

### ใช้ Xcode Debug View Hierarchy

ใน Xcode เมื่อ app กำลัง run:
1. กด **Debug > View Debugging > Capture View Hierarchy**
2. หรือกด icon 3D layers ใน Debug toolbar
3. ดู constraints ใน Size Inspector

### Console Debugging

```objc
- (void)debugConstraints {
    // ดู constraints ทั้งหมดของ view
    NSLog(@"All constraints: %@", self.view.constraints);
    
    // ดู constraints เฉพาะ
    for (NSLayoutConstraint *constraint in self.myView.constraints) {
        NSLog(@"Constraint: %@", constraint);
    }
    
    // ดู ambiguous layout
    if ([self.view hasAmbiguousLayout]) {
        NSLog(@"⚠️ Ambiguous layout detected!");
        [self.view exerciseAmbiguityInLayout];
    }
}
```

### ตั้งชื่อ Constraints เพื่อ Debug

```objc
- (void)setupNamedConstraints {
    NSLayoutConstraint *topConstraint = [myView.topAnchor 
        constraintEqualToAnchor:self.view.topAnchor constant:20];
    topConstraint.identifier = @"myView-top-20";  // ตั้งชื่อ constraint
    
    NSLayoutConstraint *widthConstraint = [myView.widthAnchor constraintEqualToConstant:100];
    widthConstraint.identifier = @"myView-width-100";
    
    [NSLayoutConstraint activateConstraints:@[topConstraint, widthConstraint]];
}
```

### Warning ที่พบบ่อย

```
// Conflicting constraints
"[LayoutConstraints] Unable to simultaneously satisfy constraints."
"Probably at least one of the constraints in the following list is one you don't want."

// Ambiguous layout  
"[LayoutConstraints] The view hierarchy is not prepared for the constraint..."
```

### Symbolic Breakpoint

เพิ่ม symbolic breakpoint ที่:
- `UIViewAlertForUnsatisfiableConstraints` - หยุดเมื่อ constraints ขัดแย้ง

```objc
// เพิ่ม identifier ให้กับ views เพื่อ debug
self.myView.accessibilityIdentifier = @"myView";
```

---

## 49.16 Safe Area Layout Guide

Safe Area ช่วยป้องกันไม่ให้ content ถูก overlap โดย status bar, navigation bar, home indicator, tab bar, etc.

```objc
- (void)setupWithSafeArea {
    UIView *contentView = [[UIView alloc] init];
    contentView.translatesAutoresizingMaskIntoConstraints = NO;
    contentView.backgroundColor = [UIColor systemBlueColor];
    [self.view addSubview:contentView];
    
    // ใช้ safeAreaLayoutGuide
    UILayoutGuide *safeArea = self.view.safeAreaLayoutGuide;
    
    [NSLayoutConstraint activateConstraints:@[
        [contentView.topAnchor constraintEqualToAnchor:safeArea.topAnchor constant:10],
        [contentView.bottomAnchor constraintEqualToAnchor:safeArea.bottomAnchor constant:-10],
        [contentView.leadingAnchor constraintEqualToAnchor:safeArea.leadingAnchor constant:10],
        [contentView.trailingAnchor constraintEqualToAnchor:safeArea.trailingAnchor constant:-10],
    ]];
}
```

### safeAreaInsets

```objc
- (void)viewSafeAreaInsetsDidChange {
    [super viewSafeAreaInsetsDidChange];
    
    // Safe area insets เปลี่ยน (เช่น เมื่อหมุนหน้าจอ)
    UIEdgeInsets insets = self.view.safeAreaInsets;
    NSLog(@"Safe area insets: top=%.0f, bottom=%.0f, left=%.0f, right=%.0f",
          insets.top, insets.bottom, insets.left, insets.right);
}
```

### Additional Safe Area Insets

```objc
// เพิ่ม safe area insets เพิ่มเติม (สำหรับ custom UI เช่น floating toolbar)
- (void)viewDidLoad {
    [super viewDidLoad];
    
    // บอก child view controllers ว่ามี custom area ที่ต้องหลีกเลี่ยง
    self.additionalSafeAreaInsets = UIEdgeInsetsMake(0, 0, 64, 0); // สำหรับ toolbar 64pt ด้านล่าง
}
```

---

## 49.17 ตัวอย่างครบถ้วน - Login Form

```objc
#import "LoginViewController.h"

@interface LoginViewController ()
@property (nonatomic, strong) UIScrollView *scrollView;
@property (nonatomic, strong) UIView *contentView;
@property (nonatomic, strong) UIImageView *logoImageView;
@property (nonatomic, strong) UILabel *titleLabel;
@property (nonatomic, strong) UITextField *emailTextField;
@property (nonatomic, strong) UITextField *passwordTextField;
@property (nonatomic, strong) UIButton *loginButton;
@property (nonatomic, strong) UIButton *forgotPasswordButton;
@property (nonatomic, strong) NSLayoutConstraint *contentViewHeight;
@end

@implementation LoginViewController

- (void)viewDidLoad {
    [super viewDidLoad];
    self.view.backgroundColor = [UIColor systemBackgroundColor];
    
    [self setupScrollView];
    [self setupUI];
    [self setupConstraints];
    [self setupKeyboardObservers];
}

- (void)setupScrollView {
    self.scrollView = [[UIScrollView alloc] init];
    self.scrollView.translatesAutoresizingMaskIntoConstraints = NO;
    [self.view addSubview:self.scrollView];
    
    self.contentView = [[UIView alloc] init];
    self.contentView.translatesAutoresizingMaskIntoConstraints = NO;
    [self.scrollView addSubview:self.contentView];
    
    [NSLayoutConstraint activateConstraints:@[
        [self.scrollView.topAnchor constraintEqualToAnchor:self.view.safeAreaLayoutGuide.topAnchor],
        [self.scrollView.leadingAnchor constraintEqualToAnchor:self.view.leadingAnchor],
        [self.scrollView.trailingAnchor constraintEqualToAnchor:self.view.trailingAnchor],
        [self.scrollView.bottomAnchor constraintEqualToAnchor:self.view.bottomAnchor],
        
        [self.contentView.topAnchor constraintEqualToAnchor:self.scrollView.topAnchor],
        [self.contentView.leadingAnchor constraintEqualToAnchor:self.scrollView.leadingAnchor],
        [self.contentView.trailingAnchor constraintEqualToAnchor:self.scrollView.trailingAnchor],
        [self.contentView.bottomAnchor constraintEqualToAnchor:self.scrollView.bottomAnchor],
        [self.contentView.widthAnchor constraintEqualToAnchor:self.scrollView.widthAnchor],
    ]];
    
    // Content height อย่างน้อยเท่ากับ scroll view
    self.contentViewHeight = [self.contentView.heightAnchor 
        constraintGreaterThanOrEqualToAnchor:self.scrollView.heightAnchor];
    self.contentViewHeight.active = YES;
}

- (void)setupUI {
    // Logo
    self.logoImageView = [[UIImageView alloc] init];
    self.logoImageView.translatesAutoresizingMaskIntoConstraints = NO;
    self.logoImageView.image = [UIImage systemImageNamed:@"person.circle.fill"];
    self.logoImageView.tintColor = [UIColor systemBlueColor];
    self.logoImageView.contentMode = UIViewContentModeScaleAspectFit;
    [self.contentView addSubview:self.logoImageView];
    
    // Title
    self.titleLabel = [[UILabel alloc] init];
    self.titleLabel.translatesAutoresizingMaskIntoConstraints = NO;
    self.titleLabel.text = @"Welcome Back";
    self.titleLabel.font = [UIFont boldSystemFontOfSize:28];
    self.titleLabel.textAlignment = NSTextAlignmentCenter;
    [self.contentView addSubview:self.titleLabel];
    
    // Email text field
    self.emailTextField = [self createTextField:@"Email" isSecure:NO];
    [self.contentView addSubview:self.emailTextField];
    
    // Password text field
    self.passwordTextField = [self createTextField:@"Password" isSecure:YES];
    [self.contentView addSubview:self.passwordTextField];
    
    // Login button
    self.loginButton = [UIButton buttonWithType:UIButtonTypeSystem];
    self.loginButton.translatesAutoresizingMaskIntoConstraints = NO;
    [self.loginButton setTitle:@"Log In" forState:UIControlStateNormal];
    self.loginButton.titleLabel.font = [UIFont boldSystemFontOfSize:18];
    self.loginButton.backgroundColor = [UIColor systemBlueColor];
    [self.loginButton setTitleColor:[UIColor whiteColor] forState:UIControlStateNormal];
    self.loginButton.layer.cornerRadius = 12;
    [self.contentView addSubview:self.loginButton];
    
    // Forgot password button
    self.forgotPasswordButton = [UIButton buttonWithType:UIButtonTypeSystem];
    self.forgotPasswordButton.translatesAutoresizingMaskIntoConstraints = NO;
    [self.forgotPasswordButton setTitle:@"Forgot Password?" forState:UIControlStateNormal];
    [self.contentView addSubview:self.forgotPasswordButton];
}

- (UITextField *)createTextField:(NSString *)placeholder isSecure:(BOOL)secure {
    UITextField *textField = [[UITextField alloc] init];
    textField.translatesAutoresizingMaskIntoConstraints = NO;
    textField.placeholder = placeholder;
    textField.borderStyle = UITextBorderStyleRoundedRect;
    textField.secureTextEntry = secure;
    textField.font = [UIFont systemFontOfSize:16];
    return textField;
}

- (void)setupConstraints {
    CGFloat padding = 24.0;
    
    [NSLayoutConstraint activateConstraints:@[
        // Logo
        [self.logoImageView.topAnchor constraintEqualToAnchor:self.contentView.topAnchor constant:60],
        [self.logoImageView.centerXAnchor constraintEqualToAnchor:self.contentView.centerXAnchor],
        [self.logoImageView.widthAnchor constraintEqualToConstant:100],
        [self.logoImageView.heightAnchor constraintEqualToConstant:100],
        
        // Title
        [self.titleLabel.topAnchor constraintEqualToAnchor:self.logoImageView.bottomAnchor constant:20],
        [self.titleLabel.leadingAnchor constraintEqualToAnchor:self.contentView.leadingAnchor constant:padding],
        [self.titleLabel.trailingAnchor constraintEqualToAnchor:self.contentView.trailingAnchor constant:-padding],
        
        // Email
        [self.emailTextField.topAnchor constraintEqualToAnchor:self.titleLabel.bottomAnchor constant:40],
        [self.emailTextField.leadingAnchor constraintEqualToAnchor:self.contentView.leadingAnchor constant:padding],
        [self.emailTextField.trailingAnchor constraintEqualToAnchor:self.contentView.trailingAnchor constant:-padding],
        [self.emailTextField.heightAnchor constraintEqualToConstant:50],
        
        // Password
        [self.passwordTextField.topAnchor constraintEqualToAnchor:self.emailTextField.bottomAnchor constant:16],
        [self.passwordTextField.leadingAnchor constraintEqualToAnchor:self.contentView.leadingAnchor constant:padding],
        [self.passwordTextField.trailingAnchor constraintEqualToAnchor:self.contentView.trailingAnchor constant:-padding],
        [self.passwordTextField.heightAnchor constraintEqualToConstant:50],
        
        // Login button
        [self.loginButton.topAnchor constraintEqualToAnchor:self.passwordTextField.bottomAnchor constant:32],
        [self.loginButton.leadingAnchor constraintEqualToAnchor:self.contentView.leadingAnchor constant:padding],
        [self.loginButton.trailingAnchor constraintEqualToAnchor:self.contentView.trailingAnchor constant:-padding],
        [self.loginButton.heightAnchor constraintEqualToConstant:52],
        
        // Forgot password
        [self.forgotPasswordButton.topAnchor constraintEqualToAnchor:self.loginButton.bottomAnchor constant:16],
        [self.forgotPasswordButton.centerXAnchor constraintEqualToAnchor:self.contentView.centerXAnchor],
        [self.forgotPasswordButton.bottomAnchor constraintEqualToAnchor:self.contentView.bottomAnchor constant:-40],
    ]];
}

- (void)setupKeyboardObservers {
    [[NSNotificationCenter defaultCenter] addObserver:self
                                             selector:@selector(keyboardWillShow:)
                                                 name:UIKeyboardWillShowNotification
                                               object:nil];
    [[NSNotificationCenter defaultCenter] addObserver:self
                                             selector:@selector(keyboardWillHide:)
                                                 name:UIKeyboardWillHideNotification
                                               object:nil];
}

- (void)keyboardWillShow:(NSNotification *)notification {
    NSDictionary *info = notification.userInfo;
    CGRect keyboardFrame = [info[UIKeyboardFrameEndUserInfoKey] CGRectValue];
    NSTimeInterval duration = [info[UIKeyboardAnimationDurationUserInfoKey] doubleValue];
    
    self.scrollView.contentInset = UIEdgeInsetsMake(0, 0, keyboardFrame.size.height, 0);
    
    [UIView animateWithDuration:duration animations:^{
        [self.view layoutIfNeeded];
    }];
}

- (void)keyboardWillHide:(NSNotification *)notification {
    self.scrollView.contentInset = UIEdgeInsetsZero;
}

- (void)dealloc {
    [[NSNotificationCenter defaultCenter] removeObserver:self];
}

@end
```

---

## 49.18 Stack Views กับ Auto Layout

`UIStackView` ช่วยลดความซับซ้อนของ Auto Layout สำหรับ linear arrangements

```objc
- (void)setupStackView {
    // สร้าง views
    UIView *view1 = [self makeColoredView:[UIColor redColor] size:CGSizeMake(100, 100)];
    UIView *view2 = [self makeColoredView:[UIColor greenColor] size:CGSizeMake(100, 100)];
    UIView *view3 = [self makeColoredView:[UIColor blueColor] size:CGSizeMake(100, 100)];
    
    // สร้าง horizontal stack view
    UIStackView *stackView = [[UIStackView alloc] initWithArrangedSubviews:@[view1, view2, view3]];
    stackView.translatesAutoresizingMaskIntoConstraints = NO;
    stackView.axis = UILayoutConstraintAxisHorizontal;
    stackView.distribution = UIStackViewDistributionEqualSpacing;
    stackView.alignment = UIStackViewAlignmentCenter;
    stackView.spacing = 20;
    [self.view addSubview:stackView];
    
    // Stack view constraints
    [NSLayoutConstraint activateConstraints:@[
        [stackView.centerXAnchor constraintEqualToAnchor:self.view.centerXAnchor],
        [stackView.centerYAnchor constraintEqualToAnchor:self.view.centerYAnchor],
        [stackView.leadingAnchor constraintGreaterThanOrEqualToAnchor:self.view.leadingAnchor constant:20],
        [stackView.trailingAnchor constraintLessThanOrEqualToAnchor:self.view.trailingAnchor constant:-20],
    ]];
}

- (UIView *)makeColoredView:(UIColor *)color size:(CGSize)size {
    UIView *view = [[UIView alloc] init];
    view.translatesAutoresizingMaskIntoConstraints = NO;
    view.backgroundColor = color;
    
    [NSLayoutConstraint activateConstraints:@[
        [view.widthAnchor constraintEqualToConstant:size.width],
        [view.heightAnchor constraintEqualToConstant:size.height],
    ]];
    
    return view;
}
```

### Nested Stack Views

```objc
- (void)setupNestedStackViews {
    // Row 1: Labels
    UILabel *label1 = [self makeLabelWithText:@"Name:" bold:YES];
    UILabel *value1 = [self makeLabelWithText:@"John Doe" bold:NO];
    UIStackView *row1 = [self makeHStackWithViews:@[label1, value1]];
    
    // Row 2
    UILabel *label2 = [self makeLabelWithText:@"Email:" bold:YES];
    UILabel *value2 = [self makeLabelWithText:@"john@example.com" bold:NO];
    UIStackView *row2 = [self makeHStackWithViews:@[label2, value2]];
    
    // Row 3
    UILabel *label3 = [self makeLabelWithText:@"Phone:" bold:YES];
    UILabel *value3 = [self makeLabelWithText:@"+1 234 567 890" bold:NO];
    UIStackView *row3 = [self makeHStackWithViews:@[label3, value3]];
    
    // Vertical stack (nested)
    UIStackView *verticalStack = [[UIStackView alloc] initWithArrangedSubviews:@[row1, row2, row3]];
    verticalStack.translatesAutoresizingMaskIntoConstraints = NO;
    verticalStack.axis = UILayoutConstraintAxisVertical;
    verticalStack.spacing = 12;
    verticalStack.distribution = UIStackViewDistributionFill;
    [self.view addSubview:verticalStack];
    
    [NSLayoutConstraint activateConstraints:@[
        [verticalStack.topAnchor constraintEqualToAnchor:self.view.safeAreaLayoutGuide.topAnchor constant:40],
        [verticalStack.leadingAnchor constraintEqualToAnchor:self.view.leadingAnchor constant:20],
        [verticalStack.trailingAnchor constraintEqualToAnchor:self.view.trailingAnchor constant:-20],
    ]];
}

- (UIStackView *)makeHStackWithViews:(NSArray *)views {
    UIStackView *stack = [[UIStackView alloc] initWithArrangedSubviews:views];
    stack.axis = UILayoutConstraintAxisHorizontal;
    stack.spacing = 8;
    stack.distribution = UIStackViewDistributionFill;
    return stack;
}

- (UILabel *)makeLabelWithText:(NSString *)text bold:(BOOL)bold {
    UILabel *label = [[UILabel alloc] init];
    label.text = text;
    label.font = bold ? [UIFont boldSystemFontOfSize:16] : [UIFont systemFontOfSize:16];
    if (bold) {
        [label setContentHuggingPriority:UILayoutPriorityRequired forAxis:UILayoutConstraintAxisHorizontal];
    }
    return label;
}
```

---

## 49.19 Self-Sizing Table View Cells

```objc
// ใน UITableViewController หรือ UIViewController ที่มี UITableView

- (void)setupSelfSizingCells {
    self.tableView.estimatedRowHeight = 100;
    self.tableView.rowHeight = UITableViewAutomaticDimension;
}

// Custom Cell
@interface CustomCell : UITableViewCell
@property (nonatomic, strong) UILabel *titleLabel;
@property (nonatomic, strong) UILabel *subtitleLabel;
@property (nonatomic, strong) UIImageView *thumbnailView;
@end

@implementation CustomCell

- (instancetype)initWithStyle:(UITableViewCellStyle)style reuseIdentifier:(NSString *)reuseIdentifier {
    self = [super initWithStyle:style reuseIdentifier:reuseIdentifier];
    if (self) {
        [self setupUI];
    }
    return self;
}

- (void)setupUI {
    self.thumbnailView = [[UIImageView alloc] init];
    self.thumbnailView.translatesAutoresizingMaskIntoConstraints = NO;
    self.thumbnailView.contentMode = UIViewContentModeScaleAspectFill;
    self.thumbnailView.clipsToBounds = YES;
    self.thumbnailView.backgroundColor = [UIColor systemGrayColor];
    [self.contentView addSubview:self.thumbnailView];
    
    self.titleLabel = [[UILabel alloc] init];
    self.titleLabel.translatesAutoresizingMaskIntoConstraints = NO;
    self.titleLabel.font = [UIFont boldSystemFontOfSize:16];
    self.titleLabel.numberOfLines = 0; // multiline
    [self.contentView addSubview:self.titleLabel];
    
    self.subtitleLabel = [[UILabel alloc] init];
    self.subtitleLabel.translatesAutoresizingMaskIntoConstraints = NO;
    self.subtitleLabel.font = [UIFont systemFontOfSize:14];
    self.subtitleLabel.textColor = [UIColor systemGrayColor];
    self.subtitleLabel.numberOfLines = 0; // multiline
    [self.contentView addSubview:self.subtitleLabel];
    
    CGFloat padding = 12.0;
    
    [NSLayoutConstraint activateConstraints:@[
        // Thumbnail
        [self.thumbnailView.leadingAnchor constraintEqualToAnchor:self.contentView.leadingAnchor constant:padding],
        [self.thumbnailView.topAnchor constraintEqualToAnchor:self.contentView.topAnchor constant:padding],
        [self.thumbnailView.widthAnchor constraintEqualToConstant:60],
        [self.thumbnailView.heightAnchor constraintEqualToConstant:60],
        // bottomAnchor ไม่ required เพื่อให้ cell ยืดตาม text
        
        // Title
        [self.titleLabel.leadingAnchor constraintEqualToAnchor:self.thumbnailView.trailingAnchor constant:padding],
        [self.titleLabel.topAnchor constraintEqualToAnchor:self.contentView.topAnchor constant:padding],
        [self.titleLabel.trailingAnchor constraintEqualToAnchor:self.contentView.trailingAnchor constant:-padding],
        
        // Subtitle (ใต้ title, และต้องกำหนด bottom เพื่อให้ cell ยืด)
        [self.subtitleLabel.leadingAnchor constraintEqualToAnchor:self.titleLabel.leadingAnchor],
        [self.subtitleLabel.topAnchor constraintEqualToAnchor:self.titleLabel.bottomAnchor constant:4],
        [self.subtitleLabel.trailingAnchor constraintEqualToAnchor:self.contentView.trailingAnchor constant:-padding],
        [self.subtitleLabel.bottomAnchor constraintEqualToAnchor:self.contentView.bottomAnchor constant:-padding],  // <-- สำคัญ!
    ]];
}

@end
```

---

## 49.20 แบบฝึกหัด (Practice Exercises)

### แบบฝึกหัดที่ 1: Profile Card

สร้าง Profile Card ที่มี:
- รูปโปรไฟล์วงกลมด้านบน (100x100)
- ชื่อผู้ใช้ (bold, center)
- อีเมล (gray, center)
- ปุ่ม Follow และ Message แบบ side-by-side
- Card มี shadow และ rounded corner
- รองรับทุกขนาดหน้าจอ

```objc
@interface ProfileCardViewController ()
@property (nonatomic, strong) UIView *cardView;
@property (nonatomic, strong) UIImageView *avatarView;
@property (nonatomic, strong) UILabel *nameLabel;
@property (nonatomic, strong) UILabel *emailLabel;
@property (nonatomic, strong) UIButton *followButton;
@property (nonatomic, strong) UIButton *messageButton;
@end

@implementation ProfileCardViewController

- (void)viewDidLoad {
    [super viewDidLoad];
    self.view.backgroundColor = [UIColor systemGroupedBackgroundColor];
    [self buildCard];
}

- (void)buildCard {
    // Card container
    self.cardView = [[UIView alloc] init];
    self.cardView.translatesAutoresizingMaskIntoConstraints = NO;
    self.cardView.backgroundColor = [UIColor systemBackgroundColor];
    self.cardView.layer.cornerRadius = 16;
    self.cardView.layer.shadowColor = [UIColor blackColor].CGColor;
    self.cardView.layer.shadowOpacity = 0.15;
    self.cardView.layer.shadowRadius = 8;
    self.cardView.layer.shadowOffset = CGSizeMake(0, 4);
    [self.view addSubview:self.cardView];
    
    // Avatar
    self.avatarView = [[UIImageView alloc] init];
    self.avatarView.translatesAutoresizingMaskIntoConstraints = NO;
    self.avatarView.backgroundColor = [UIColor systemGray4Color];
    self.avatarView.layer.cornerRadius = 50;
    self.avatarView.clipsToBounds = YES;
    self.avatarView.contentMode = UIViewContentModeScaleAspectFill;
    self.avatarView.image = [UIImage systemImageNamed:@"person.circle.fill"];
    self.avatarView.tintColor = [UIColor systemGrayColor];
    [self.cardView addSubview:self.avatarView];
    
    // Name
    self.nameLabel = [[UILabel alloc] init];
    self.nameLabel.translatesAutoresizingMaskIntoConstraints = NO;
    self.nameLabel.text = @"John Doe";
    self.nameLabel.font = [UIFont boldSystemFontOfSize:22];
    self.nameLabel.textAlignment = NSTextAlignmentCenter;
    [self.cardView addSubview:self.nameLabel];
    
    // Email
    self.emailLabel = [[UILabel alloc] init];
    self.emailLabel.translatesAutoresizingMaskIntoConstraints = NO;
    self.emailLabel.text = @"john.doe@example.com";
    self.emailLabel.font = [UIFont systemFontOfSize:14];
    self.emailLabel.textColor = [UIColor systemGrayColor];
    self.emailLabel.textAlignment = NSTextAlignmentCenter;
    [self.cardView addSubview:self.emailLabel];
    
    // Follow button
    self.followButton = [self makeButtonWithTitle:@"Follow" primary:YES];
    [self.cardView addSubview:self.followButton];
    
    // Message button
    self.messageButton = [self makeButtonWithTitle:@"Message" primary:NO];
    [self.cardView addSubview:self.messageButton];
    
    // Constraints
    [NSLayoutConstraint activateConstraints:@[
        // Card
        [self.cardView.centerXAnchor constraintEqualToAnchor:self.view.centerXAnchor],
        [self.cardView.centerYAnchor constraintEqualToAnchor:self.view.centerYAnchor],
        [self.cardView.leadingAnchor constraintEqualToAnchor:self.view.leadingAnchor constant:30],
        [self.cardView.trailingAnchor constraintEqualToAnchor:self.view.trailingAnchor constant:-30],
        
        // Avatar (อยู่บนสุด, center)
        [self.avatarView.topAnchor constraintEqualToAnchor:self.cardView.topAnchor constant:30],
        [self.avatarView.centerXAnchor constraintEqualToAnchor:self.cardView.centerXAnchor],
        [self.avatarView.widthAnchor constraintEqualToConstant:100],
        [self.avatarView.heightAnchor constraintEqualToConstant:100],
        
        // Name
        [self.nameLabel.topAnchor constraintEqualToAnchor:self.avatarView.bottomAnchor constant:16],
        [self.nameLabel.leadingAnchor constraintEqualToAnchor:self.cardView.leadingAnchor constant:20],
        [self.nameLabel.trailingAnchor constraintEqualToAnchor:self.cardView.trailingAnchor constant:-20],
        
        // Email
        [self.emailLabel.topAnchor constraintEqualToAnchor:self.nameLabel.bottomAnchor constant:6],
        [self.emailLabel.leadingAnchor constraintEqualToAnchor:self.cardView.leadingAnchor constant:20],
        [self.emailLabel.trailingAnchor constraintEqualToAnchor:self.cardView.trailingAnchor constant:-20],
        
        // Buttons
        [self.followButton.topAnchor constraintEqualToAnchor:self.emailLabel.bottomAnchor constant:24],
        [self.followButton.leadingAnchor constraintEqualToAnchor:self.cardView.leadingAnchor constant:20],
        [self.followButton.bottomAnchor constraintEqualToAnchor:self.cardView.bottomAnchor constant:-24],
        [self.followButton.heightAnchor constraintEqualToConstant:44],
        
        [self.messageButton.topAnchor constraintEqualToAnchor:self.followButton.topAnchor],
        [self.messageButton.leadingAnchor constraintEqualToAnchor:self.followButton.trailingAnchor constant:12],
        [self.messageButton.trailingAnchor constraintEqualToAnchor:self.cardView.trailingAnchor constant:-20],
        [self.messageButton.heightAnchor constraintEqualToConstant:44],
        [self.messageButton.widthAnchor constraintEqualToAnchor:self.followButton.widthAnchor],
    ]];
}

- (UIButton *)makeButtonWithTitle:(NSString *)title primary:(BOOL)primary {
    UIButton *button = [UIButton buttonWithType:UIButtonTypeSystem];
    button.translatesAutoresizingMaskIntoConstraints = NO;
    [button setTitle:title forState:UIControlStateNormal];
    button.titleLabel.font = [UIFont boldSystemFontOfSize:16];
    button.layer.cornerRadius = 10;
    
    if (primary) {
        button.backgroundColor = [UIColor systemBlueColor];
        [button setTitleColor:[UIColor whiteColor] forState:UIControlStateNormal];
    } else {
        button.backgroundColor = [UIColor systemGray5Color];
        [button setTitleColor:[UIColor systemBlueColor] forState:UIControlStateNormal];
    }
    
    return button;
}

@end
```

### แบบฝึกหัดที่ 2: Animated Expandable Card

สร้าง card ที่ขยายเมื่อแตะ และยุบเมื่อแตะอีกครั้ง

```objc
@interface ExpandableCardViewController ()
@property (nonatomic, strong) UIView *card;
@property (nonatomic, strong) UILabel *headerLabel;
@property (nonatomic, strong) UILabel *detailLabel;
@property (nonatomic, strong) NSLayoutConstraint *expandedConstraint;
@property (nonatomic, strong) NSLayoutConstraint *collapsedConstraint;
@property (nonatomic) BOOL isExpanded;
@end

@implementation ExpandableCardViewController

- (void)viewDidLoad {
    [super viewDidLoad];
    self.view.backgroundColor = [UIColor systemGroupedBackgroundColor];
    self.isExpanded = NO;
    [self setupExpandableCard];
}

- (void)setupExpandableCard {
    self.card = [[UIView alloc] init];
    self.card.translatesAutoresizingMaskIntoConstraints = NO;
    self.card.backgroundColor = [UIColor systemBackgroundColor];
    self.card.layer.cornerRadius = 12;
    self.card.clipsToBounds = YES;
    [self.view addSubview:self.card];
    
    self.headerLabel = [[UILabel alloc] init];
    self.headerLabel.translatesAutoresizingMaskIntoConstraints = NO;
    self.headerLabel.text = @"Tap to expand";
    self.headerLabel.font = [UIFont boldSystemFontOfSize:18];
    [self.card addSubview:self.headerLabel];
    
    self.detailLabel = [[UILabel alloc] init];
    self.detailLabel.translatesAutoresizingMaskIntoConstraints = NO;
    self.detailLabel.text = @"This is the detail content that appears when expanded. It can be multiple lines of text.";
    self.detailLabel.font = [UIFont systemFontOfSize:14];
    self.detailLabel.textColor = [UIColor systemGrayColor];
    self.detailLabel.numberOfLines = 0;
    self.detailLabel.alpha = 0;
    [self.card addSubview:self.detailLabel];
    
    // Collapsed: header only
    self.collapsedConstraint = [self.card.heightAnchor constraintEqualToConstant:60];
    
    // Expanded: header + detail
    self.expandedConstraint = [self.detailLabel.bottomAnchor 
        constraintEqualToAnchor:self.card.bottomAnchor constant:-16];
    
    [NSLayoutConstraint activateConstraints:@[
        [self.card.topAnchor constraintEqualToAnchor:self.view.safeAreaLayoutGuide.topAnchor constant:20],
        [self.card.leadingAnchor constraintEqualToAnchor:self.view.leadingAnchor constant:20],
        [self.card.trailingAnchor constraintEqualToAnchor:self.view.trailingAnchor constant:-20],
        self.collapsedConstraint,  // เริ่มต้น collapsed
        
        [self.headerLabel.leadingAnchor constraintEqualToAnchor:self.card.leadingAnchor constant:16],
        [self.headerLabel.centerYAnchor constraintEqualToAnchor:self.card.topAnchor constant:30],
        
        [self.detailLabel.topAnchor constraintEqualToAnchor:self.headerLabel.bottomAnchor constant:12],
        [self.detailLabel.leadingAnchor constraintEqualToAnchor:self.card.leadingAnchor constant:16],
        [self.detailLabel.trailingAnchor constraintEqualToAnchor:self.card.trailingAnchor constant:-16],
    ]];
    
    UITapGestureRecognizer *tap = [[UITapGestureRecognizer alloc] 
        initWithTarget:self action:@selector(toggleExpansion)];
    [self.card addGestureRecognizer:tap];
}

- (void)toggleExpansion {
    self.isExpanded = !self.isExpanded;
    
    if (self.isExpanded) {
        self.collapsedConstraint.active = NO;
        self.expandedConstraint.active = YES;
        self.headerLabel.text = @"Tap to collapse";
    } else {
        self.expandedConstraint.active = NO;
        self.collapsedConstraint.active = YES;
        self.headerLabel.text = @"Tap to expand";
    }
    
    [UIView animateWithDuration:0.3 animations:^{
        self.detailLabel.alpha = self.isExpanded ? 1.0 : 0.0;
        [self.view layoutIfNeeded];
    }];
}

@end
```

### แบบฝึกหัดที่ 3: Responsive Grid

สร้าง grid layout ที่ปรับตาม orientation:
- Portrait: 2 columns
- Landscape: 4 columns

```objc
// ใช้ UICollectionView กับ UICollectionViewFlowLayout
- (void)setupResponsiveGrid {
    UICollectionViewFlowLayout *layout = [[UICollectionViewFlowLayout alloc] init];
    layout.minimumInteritemSpacing = 8;
    layout.minimumLineSpacing = 8;
    layout.sectionInset = UIEdgeInsetsMake(8, 8, 8, 8);
    
    UICollectionView *collectionView = [[UICollectionView alloc] 
        initWithFrame:CGRectZero 
        collectionViewLayout:layout];
    collectionView.translatesAutoresizingMaskIntoConstraints = NO;
    collectionView.dataSource = self;
    collectionView.delegate = self;
    [collectionView registerClass:[UICollectionViewCell class] 
      forCellWithReuseIdentifier:@"Cell"];
    [self.view addSubview:collectionView];
    
    [NSLayoutConstraint activateConstraints:@[
        [collectionView.topAnchor constraintEqualToAnchor:self.view.safeAreaLayoutGuide.topAnchor],
        [collectionView.leadingAnchor constraintEqualToAnchor:self.view.leadingAnchor],
        [collectionView.trailingAnchor constraintEqualToAnchor:self.view.trailingAnchor],
        [collectionView.bottomAnchor constraintEqualToAnchor:self.view.bottomAnchor],
    ]];
}

// UICollectionViewDelegateFlowLayout
- (CGSize)collectionView:(UICollectionView *)collectionView 
                  layout:(UICollectionViewLayout *)collectionViewLayout 
  sizeForItemAtIndexPath:(NSIndexPath *)indexPath {
    
    UICollectionViewFlowLayout *layout = (UICollectionViewFlowLayout *)collectionViewLayout;
    CGFloat spacing = layout.minimumInteritemSpacing;
    CGFloat inset = layout.sectionInset.left + layout.sectionInset.right;
    
    BOOL isLandscape = collectionView.bounds.width > collectionView.bounds.height;
    NSInteger columns = isLandscape ? 4 : 2;
    
    CGFloat width = (collectionView.bounds.width - inset - spacing * (columns - 1)) / columns;
    return CGSizeMake(width, width);
}
```

---

## สรุป (Summary)

- **Auto Layout** ใช้ระบบ constraints เพื่อกำหนดขนาดและตำแหน่งของ views
- **NSLayoutConstraint** เป็น API classic ที่ verbose แต่ยืดหยุ่น
- **NSLayoutAnchor** (Anchors API) เป็นวิธีที่แนะนำในปัจจุบัน อ่านง่ายกว่า
- **Priority** ใช้แก้ปัญหาเมื่อ constraints ขัดแย้งกัน
- **Content Hugging / Compression Resistance** สำคัญสำหรับ views ที่มี intrinsic size
- **Safe Area Layout Guide** ช่วยให้ content ไม่ถูก overlap
- **UILayoutGuide** ใช้เป็น invisible proxy สำหรับ spacing
- ใช้ `layoutIfNeeded` ภายใน animation block เพื่อ animate constraint changes
- ตั้ง `translatesAutoresizingMaskIntoConstraints = NO` เสมอเมื่อใช้ Auto Layout แบบ programmatic

Auto Layout เป็นทักษะสำคัญที่นักพัฒนา iOS ทุกคนต้องเชี่ยวชาญ เพราะเป็นรากฐานของการสร้าง UI ที่รองรับทุกขนาดหน้าจอและการตั้งค่าของผู้ใช้
