# ตอนที่ 44: UIView Deep Dive (เจาะลึก UIView)

## บทนำ

`UIView` เป็น building block พื้นฐานของ UI ใน iOS ทุก UI element สืบทอดมาจาก UIView ในตอนนี้เราจะเจาะลึกคุณสมบัติและความสามารถของ UIView ตั้งแต่ properties พื้นฐาน การวาด animation ไปจนถึง gesture recognizers และ touch handling

---

## 44.1 UIView Properties

### Frame, Bounds, Center

```objc
UIView *myView = [[UIView alloc] init];

// Frame: position และขนาดใน coordinate space ของ superview
// CGRect{origin: CGPoint{x, y}, size: CGSize{width, height}}
myView.frame = CGRectMake(50, 100, 200, 150);
// x=50, y=100 จาก superview origin
// width=200, height=150

// ดึงค่าแต่ละส่วน
CGFloat x = myView.frame.origin.x;       // 50
CGFloat y = myView.frame.origin.y;       // 100
CGFloat w = myView.frame.size.width;     // 200
CGFloat h = myView.frame.size.height;    // 150

// Bounds: ขนาดใน coordinate space ของตัวเอง
// bounds.origin มักเป็น (0,0) ยกเว้น scroll views
myView.bounds = CGRectMake(0, 0, 200, 150);
CGRect bounds = myView.bounds; // {0, 0, 200, 150}

// Center: จุดกึ่งกลางใน coordinate space ของ superview
myView.center = CGPointMake(150, 175); // ตรงกลาง frame ข้างบน
CGPoint center = myView.center;

// ความสัมพันธ์: frame.origin = center - bounds.size/2
// center.x = frame.origin.x + frame.size.width / 2
// center.y = frame.origin.y + frame.size.height / 2

// เปลี่ยนตำแหน่งโดยใช้ center (ดีกว่า frame เพราะไม่ต้องคำนวณ origin)
myView.center = CGPointMake(
    CGRectGetMidX(self.view.bounds),  // กึ่งกลางแนวนอน
    CGRectGetMidY(self.view.bounds)   // กึ่งกลางแนวตั้ง
);

// CGRect helpers
CGRect rect = CGRectMake(10, 20, 100, 50);
CGFloat midX = CGRectGetMidX(rect);    // 60
CGFloat midY = CGRectGetMidY(rect);    // 45
CGFloat maxX = CGRectGetMaxX(rect);    // 110
CGFloat maxY = CGRectGetMaxY(rect);    // 70
CGFloat minX = CGRectGetMinX(rect);    // 10
CGFloat width = CGRectGetWidth(rect);  // 100
CGFloat height = CGRectGetHeight(rect); // 50

// CGRect operations
CGRect inset = CGRectInset(rect, 10, 10);  // ลดขนาดทุกด้าน 10
CGRect offset = CGRectOffset(rect, 5, 5);  // เลื่อน 5, 5
CGRect union_ = CGRectUnion(rect, otherRect);        // รวม 2 rect
CGRect intersection = CGRectIntersection(rect, otherRect); // ส่วนที่ทับกัน
BOOL contains = CGRectContainsPoint(rect, CGPointMake(50, 30)); // YES
BOOL intersects = CGRectIntersectsRect(rect, otherRect);
```

### Transform

```objc
UIView *view = [[UIView alloc] initWithFrame:CGRectMake(100, 100, 100, 100)];
[self.view addSubview:view];

// Identity (รีเซ็ต)
view.transform = CGAffineTransformIdentity;

// Rotation (เป็น radians)
CGFloat degrees = 45.0;
CGFloat radians = degrees * M_PI / 180.0;
view.transform = CGAffineTransformMakeRotation(radians);

// Scale
view.transform = CGAffineTransformMakeScale(1.5, 1.5);   // ขยาย 1.5x
view.transform = CGAffineTransformMakeScale(0.5, 0.5);   // ย่อ 50%
view.transform = CGAffineTransformMakeScale(2.0, 1.0);   // ขยายแค่แนวนอน

// Translation
view.transform = CGAffineTransformMakeTranslation(50, 0); // เลื่อนขวา 50
view.transform = CGAffineTransformMakeTranslation(0, -100); // เลื่อนขึ้น 100

// Combine Transforms
CGAffineTransform rotated = CGAffineTransformMakeRotation(M_PI / 4);
CGAffineTransform scaled = CGAffineTransformMakeScale(1.5, 1.5);
CGAffineTransform combined = CGAffineTransformConcat(rotated, scaled);
view.transform = combined;

// หรือแบบสั้นกว่า
view.transform = CGAffineTransformRotate(
    CGAffineTransformMakeScale(1.5, 1.5), 
    M_PI / 4
);

// 3D Transform (ต้องใช้ layer.transform)
CATransform3D transform3D = CATransform3DIdentity;
transform3D.m34 = -1.0 / 500.0;  // perspective
transform3D = CATransform3DRotate(transform3D, M_PI / 6, 0, 1, 0); // หมุนรอบแกน Y
view.layer.transform = transform3D;

// ตรวจสอบ transform
BOOL isIdentity = CGAffineTransformIsIdentity(view.transform);
```

### Visual Properties

```objc
// Alpha (ความโปร่งใส) 0.0-1.0
view.alpha = 0.8;

// Hidden
view.hidden = YES;
view.hidden = NO;

// Background Color
view.backgroundColor = [UIColor systemBlueColor];
view.backgroundColor = [UIColor clearColor];  // โปร่งใส

// Tint Color
view.tintColor = [UIColor systemOrangeColor];

// Clip to Bounds
view.clipsToBounds = YES;  // ตัด subviews ที่เกินขอบ

// Layer Properties
view.layer.cornerRadius = 12;
view.layer.borderWidth = 1.0;
view.layer.borderColor = [UIColor systemGrayColor].CGColor;
view.layer.opacity = 0.9;

// Shadow
view.layer.shadowColor = [UIColor blackColor].CGColor;
view.layer.shadowOpacity = 0.3;
view.layer.shadowOffset = CGSizeMake(0, 4);
view.layer.shadowRadius = 8;

// ต้องใช้ shadow path เพื่อประสิทธิภาพ
view.layer.shadowPath = [UIBezierPath 
    bezierPathWithRoundedRect:view.bounds 
                 cornerRadius:12].CGPath;

// masksToBounds (= clipsToBounds สำหรับ layer)
// ถ้า masksToBounds = YES จะไม่เห็น shadow!
view.layer.masksToBounds = NO;  // ต้องเป็น NO เพื่อให้เห็น shadow

// Rasterization (เพิ่มประสิทธิภาพสำหรับ static content)
view.layer.shouldRasterize = YES;
view.layer.rasterizationScale = [UIScreen mainScreen].scale;

// Content Scale
view.contentScaleFactor = [UIScreen mainScreen].scale;

// autoresizingMask (สำหรับ non-Auto Layout)
view.autoresizingMask = UIViewAutoresizingFlexibleWidth | UIViewAutoresizingFlexibleHeight;
// ขยายตามขนาด superview

// Content Mode
view.contentMode = UIViewContentModeRedraw;         // วาดใหม่เมื่อขนาดเปลี่ยน
view.contentMode = UIViewContentModeScaleAspectFit; // scale ไม่เกินขอบ
```

---

## 44.2 View Hierarchy

### การจัดการ View Hierarchy

```objc
// เพิ่ม subview
[parentView addSubview:childView];

// เพิ่ม subview ที่ตำแหน่งเฉพาะ (z-order)
[parentView insertSubview:childView atIndex:0];   // ด้านล่างสุด
[parentView insertSubview:childView aboveSubview:otherView]; // บน otherView
[parentView insertSubview:childView belowSubview:otherView]; // ใต้ otherView

// ลบออก
[childView removeFromSuperview];

// เปลี่ยน z-order
[parentView bringSubviewToFront:childView]; // นำขึ้นด้านหน้า
[parentView sendSubviewToBack:childView];   // ส่งไปด้านหลัง
[parentView exchangeSubviewAtIndex:0 withSubviewAtIndex:1]; // สลับตำแหน่ง

// ค้นหา Views
NSArray *subviews = parentView.subviews;  // ดึง subviews ทั้งหมด
UIView *superview = childView.superview;  // ดึง superview

// ค้นหาด้วย Tag
UIView *taggedView = [parentView viewWithTag:42];

// ดึง View Controller ที่ contain view
UIResponder *responder = childView;
while (responder) {
    if ([responder isKindOfClass:[UIViewController class]]) {
        NSLog(@"Found VC: %@", responder);
        break;
    }
    responder = responder.nextResponder;
}

// Convert Coordinates
// View ที่อยู่ต่างระดับใน hierarchy มี coordinate space ต่างกัน
CGPoint pointInParent = [childView convertPoint:CGPointMake(10, 10) 
                                         toView:parentView];
CGPoint pointInChild = [parentView convertPoint:pointInParent 
                                       fromView:childView];

// Convert Rect
CGRect rectInSuperview = [childView convertRect:childView.bounds 
                                         toView:self.view];

// ตรวจสอบ overlap
BOOL overlaps = CGRectIntersectsRect(view1.frame, view2.frame);

// ตรวจสอบว่า point อยู่ใน view
BOOL contains = [view pointInside:[view convertPoint:tapPoint fromView:self.view] 
                         withEvent:nil];
```

### Traversing the Hierarchy

```objc
// วน loop ผ่าน subviews ทั้งหมด (deep)
- (void)traverseView:(UIView *)view depth:(NSInteger)depth {
    NSString *indent = [@"" stringByPaddingToLength:depth * 2 
                                         withString:@"  " 
                                startingAtIndex:0];
    NSLog(@"%@%@", indent, NSStringFromClass(view.class));
    
    for (UIView *subview in view.subviews) {
        [self traverseView:subview depth:depth + 1];
    }
}

// เรียกใช้
[self traverseView:self.view depth:0];

// ค้นหา views ทุกชนิดใน hierarchy
- (NSArray<UIView *> *)findViewsOfClass:(Class)viewClass 
                                 inView:(UIView *)view {
    NSMutableArray *found = [NSMutableArray array];
    
    if ([view isKindOfClass:viewClass]) {
        [found addObject:view];
    }
    
    for (UIView *subview in view.subviews) {
        [found addObjectsFromArray:[self findViewsOfClass:viewClass 
                                                   inView:subview]];
    }
    
    return found;
}

// ใช้งาน
NSArray *allLabels = [self findViewsOfClass:[UILabel class] 
                                     inView:self.view];
NSLog(@"Found %lu labels", allLabels.count);
```

---

## 44.3 View Drawing (drawRect: และ Core Graphics)

### Override drawRect:

```objc
// Custom View ที่วาดเอง
@interface CustomChartView : UIView

@property (nonatomic, strong) NSArray<NSNumber *> *values;
@property (nonatomic, strong) UIColor *fillColor;
@property (nonatomic, strong) UIColor *strokeColor;

@end

@implementation CustomChartView

- (instancetype)initWithFrame:(CGRect)frame {
    self = [super initWithFrame:frame];
    if (self) {
        self.backgroundColor = [UIColor clearColor];
        self.opaque = NO; // ต้องตั้งเมื่อ background เป็น clear
    }
    return self;
}

// วาดเสมอใน Main Thread เท่านั้น
- (void)drawRect:(CGRect)rect {
    // ดึง Graphics Context
    CGContextRef context = UIGraphicsGetCurrentContext();
    if (!context) return;
    
    CGFloat width = rect.size.width;
    CGFloat height = rect.size.height;
    
    // --- วาด Background ---
    CGContextSetFillColorWithColor(context, 
        [UIColor systemGray6Color].CGColor);
    CGContextFillRect(context, rect);
    
    // --- วาด Bar Chart ---
    if (self.values.count == 0) return;
    
    NSInteger count = self.values.count;
    CGFloat barWidth = (width - 40) / count - 10;
    CGFloat maxValue = [[self.values valueForKeyPath:@"@max.self"] floatValue];
    
    for (NSInteger i = 0; i < count; i++) {
        CGFloat value = [self.values[i] floatValue];
        CGFloat barHeight = (value / maxValue) * (height - 60);
        CGFloat x = 20 + i * (barWidth + 10);
        CGFloat y = height - 40 - barHeight;
        
        CGRect barRect = CGRectMake(x, y, barWidth, barHeight);
        
        // วาด bar
        UIBezierPath *barPath = [UIBezierPath bezierPathWithRoundedRect:barRect
                                                          cornerRadius:4];
        CGContextSetFillColorWithColor(context, 
            (self.fillColor ?: [UIColor systemBlueColor]).CGColor);
        CGContextAddPath(context, barPath.CGPath);
        CGContextFillPath(context);
        
        // วาด value text
        NSString *valueText = [NSString stringWithFormat:@"%.0f", value];
        NSDictionary *attrs = @{
            NSFontAttributeName: [UIFont systemFontOfSize:10],
            NSForegroundColorAttributeName: [UIColor labelColor]
        };
        CGRect textRect = CGRectMake(x, y - 20, barWidth, 16);
        [valueText drawInRect:textRect withAttributes:attrs];
    }
    
    // --- วาดเส้นแกน X ---
    CGContextSetStrokeColorWithColor(context, 
        [UIColor systemGrayColor].CGColor);
    CGContextSetLineWidth(context, 1.0);
    CGContextMoveToPoint(context, 15, height - 40);
    CGContextAddLineToPoint(context, width - 15, height - 40);
    CGContextStrokePath(context);
}

// สั่งวาดใหม่
- (void)setValues:(NSArray<NSNumber *> *)values {
    _values = values;
    [self setNeedsDisplay]; // จะเรียก drawRect: ในรอบถัดไป
}

// setNeedsDisplayInRect: สำหรับวาดแค่บางส่วน (ประสิทธิภาพดีกว่า)
- (void)updateBarAtIndex:(NSInteger)index value:(CGFloat)value {
    CGFloat barWidth = (self.bounds.size.width - 40) / self.values.count - 10;
    CGFloat x = 20 + index * (barWidth + 10);
    CGRect dirtyRect = CGRectMake(x - 5, 0, barWidth + 20, self.bounds.size.height);
    [self setNeedsDisplayInRect:dirtyRect];
}

@end
```

### UIBezierPath

```objc
// UIBezierPath สำหรับวาด shapes ต่างๆ

// วงกลม
UIBezierPath *circle = [UIBezierPath bezierPathWithOvalInRect:
    CGRectMake(10, 10, 100, 100)];

// สี่เหลี่ยมมน
UIBezierPath *roundedRect = [UIBezierPath 
    bezierPathWithRoundedRect:CGRectMake(10, 10, 200, 100)
                 cornerRadius:12];

// สี่เหลี่ยมมนแบบเลือกมุม
UIBezierPath *customCorner = [UIBezierPath
    bezierPathWithRoundedRect:CGRectMake(10, 10, 200, 100)
           byRoundingCorners:UIRectCornerTopLeft | UIRectCornerTopRight
                 cornerRadii:CGSizeMake(20, 20)];

// เส้นและ curves
UIBezierPath *path = [UIBezierPath bezierPath];
[path moveToPoint:CGPointMake(10, 50)];
[path addLineToPoint:CGPointMake(100, 50)];
[path addLineToPoint:CGPointMake(100, 100)];
[path closePath];  // ปิด path

// Quadratic Bezier Curve
[path moveToPoint:CGPointMake(10, 100)];
[path addQuadCurveToPoint:CGPointMake(200, 100) 
             controlPoint:CGPointMake(100, 0)];

// Cubic Bezier Curve
[path moveToPoint:CGPointMake(10, 100)];
[path addCurveToPoint:CGPointMake(300, 100)
        controlPoint1:CGPointMake(100, 0)
        controlPoint2:CGPointMake(200, 200)];

// Arc
[path moveToPoint:CGPointMake(100, 100)];
[path addArcWithCenter:CGPointMake(100, 100)
                radius:50
            startAngle:0
              endAngle:M_PI
             clockwise:YES];

// ใช้ path ใน drawRect:
- (void)drawRect:(CGRect)rect {
    CGContextRef context = UIGraphicsGetCurrentContext();
    
    // Stroke (เส้นขอบ)
    UIBezierPath *path = [UIBezierPath bezierPathWithOvalInRect:
        CGRectInset(rect, 10, 10)];
    
    [[UIColor systemBlueColor] setStroke];
    path.lineWidth = 3.0;
    path.lineCapStyle = kCGLineCapRound;
    path.lineJoinStyle = kCGLineJoinRound;
    [path setLineDash:(CGFloat[]){8, 4} count:2 phase:0]; // เส้นประ
    [path stroke];
    
    // Fill (เติมสี)
    [[UIColor systemBlueColor] colorWithAlphaComponent:0.2] setFill];
    [path fill];
    
    // Fill และ Stroke พร้อมกัน
    [[UIColor systemBlueColor] setFill];
    [[UIColor systemBlueColor] setStroke];
    path.lineWidth = 2.0;
    [path fill];
    [path stroke];
}

// เช็คว่า point อยู่ใน path
BOOL isInPath = [path containsPoint:CGPointMake(50, 50)];

// Path เป็น mask
CAShapeLayer *maskLayer = [CAShapeLayer layer];
maskLayer.path = path.CGPath;
view.layer.mask = maskLayer;
```

### Core Graphics Context Operations

```objc
- (void)drawRect:(CGRect)rect {
    CGContextRef ctx = UIGraphicsGetCurrentContext();
    
    // Save/Restore State (สำคัญมาก!)
    CGContextSaveGState(ctx);
    
    // Clip (ตัดขอบเขตการวาด)
    CGContextAddEllipseInRect(ctx, CGRectMake(50, 50, 100, 100));
    CGContextClip(ctx);
    
    // วาดใน clipped region
    CGContextSetFillColorWithColor(ctx, [UIColor systemRedColor].CGColor);
    CGContextFillRect(ctx, rect);
    
    // Restore state กลับมา
    CGContextRestoreGState(ctx);
    
    // Translate (ย้าย origin)
    CGContextTranslateCTM(ctx, 50, 50);
    
    // Rotate
    CGContextRotateCTM(ctx, M_PI / 6);
    
    // Scale
    CGContextScaleCTM(ctx, 2.0, 2.0);
    
    // วาดรูปที่ transform แล้ว
    CGContextSetFillColorWithColor(ctx, [UIColor systemBlueColor].CGColor);
    CGContextFillRect(ctx, CGRectMake(0, 0, 50, 50));
    
    // Gradient
    CGColorSpaceRef colorSpace = CGColorSpaceCreateDeviceRGB();
    NSArray *colors = @[
        (id)[UIColor systemBlueColor].CGColor,
        (id)[UIColor systemPurpleColor].CGColor
    ];
    CGFloat locations[] = {0.0, 1.0};
    CGGradientRef gradient = CGGradientCreateWithColors(colorSpace, 
        (__bridge CFArrayRef)colors, locations);
    
    CGContextDrawLinearGradient(ctx, gradient,
        CGPointMake(0, 0), CGPointMake(rect.size.width, rect.size.height),
        kCGGradientDrawsBeforeStartLocation | kCGGradientDrawsAfterEndLocation);
    
    CGGradientRelease(gradient);
    CGColorSpaceRelease(colorSpace);
    
    // วาดข้อความ
    NSString *text = @"Hello";
    NSDictionary *attrs = @{
        NSFontAttributeName: [UIFont boldSystemFontOfSize:24],
        NSForegroundColorAttributeName: [UIColor whiteColor],
        NSShadowAttributeName: ({
            NSShadow *shadow = [[NSShadow alloc] init];
            shadow.shadowColor = [UIColor blackColor];
            shadow.shadowOffset = CGSizeMake(1, 1);
            shadow.shadowBlurRadius = 2;
            shadow;
        })
    };
    [text drawAtPoint:CGPointMake(10, 10) withAttributes:attrs];
    
    // วาดรูปภาพ
    UIImage *image = [UIImage imageNamed:@"logo"];
    [image drawInRect:CGRectMake(0, 0, 100, 100)];
    [image drawAtPoint:CGPointMake(0, 0)];
}
```

---

## 44.4 Animations

### UIView Block Animations

```objc
// Basic Animation
[UIView animateWithDuration:0.3 animations:^{
    view.alpha = 0.0;
    view.transform = CGAffineTransformMakeScale(0.1, 0.1);
}];

// Animation with Completion
[UIView animateWithDuration:0.3
                 animations:^{
    view.frame = CGRectMake(100, 200, 100, 100);
} completion:^(BOOL finished) {
    if (finished) {
        NSLog(@"Animation เสร็จแล้ว");
        [view removeFromSuperview];
    }
}];

// Animation with Options
[UIView animateWithDuration:0.5
                      delay:0.2
                    options:UIViewAnimationOptionCurveEaseInOut |
                            UIViewAnimationOptionRepeat |
                            UIViewAnimationOptionAutoreverse
                 animations:^{
    view.transform = CGAffineTransformMakeTranslation(100, 0);
} completion:nil];

// UIViewAnimationOptions:
// UIViewAnimationOptionCurveEaseIn        - เร่งตอนต้น
// UIViewAnimationOptionCurveEaseOut       - ชะลอตอนท้าย
// UIViewAnimationOptionCurveEaseInOut     - เร่งแล้วชะลอ (Default)
// UIViewAnimationOptionCurveLinear        - เร็วสม่ำเสมอ
// UIViewAnimationOptionRepeat             - วนซ้ำไม่จำกัด
// UIViewAnimationOptionAutoreverse        - กลับไปกลับมา
// UIViewAnimationOptionAllowUserInteraction - รับ input ระหว่าง animate
// UIViewAnimationOptionBeginFromCurrentState - เริ่มจากตำแหน่งปัจจุบัน
// UIViewAnimationOptionTransitionFlipFromLeft - Flip ซ้าย
// UIViewAnimationOptionTransitionCrossDissolve - Fade
```

### Spring Animations

```objc
// Spring Animation - ให้ความรู้สึก bouncy
[UIView animateWithDuration:0.6
                      delay:0.0
     usingSpringWithDamping:0.7  // 0.0=สั่นมาก, 1.0=ไม่สั่น
      initialSpringVelocity:0.5  // ความเร็วเริ่มต้น (0=เริ่มจาก 0)
                    options:UIViewAnimationOptionCurveEaseOut
                 animations:^{
    view.center = self.view.center;
    view.transform = CGAffineTransformIdentity;
} completion:^(BOOL finished) {
    NSLog(@"Spring animation เสร็จ");
}];

// ตัวอย่าง: Pop-in animation สำหรับแสดง view
- (void)showViewWithSpringAnimation:(UIView *)view {
    view.transform = CGAffineTransformMakeScale(0.1, 0.1);
    view.alpha = 0;
    view.hidden = NO;
    
    [UIView animateWithDuration:0.4
                          delay:0
         usingSpringWithDamping:0.6
          initialSpringVelocity:0.8
                        options:0
                     animations:^{
        view.transform = CGAffineTransformIdentity;
        view.alpha = 1.0;
    } completion:nil];
}

// Dismiss animation
- (void)dismissViewWithAnimation:(UIView *)view {
    [UIView animateWithDuration:0.25
                     animations:^{
        view.transform = CGAffineTransformMakeScale(0.1, 0.1);
        view.alpha = 0;
    } completion:^(BOOL finished) {
        view.hidden = YES;
        view.transform = CGAffineTransformIdentity;
    }];
}
```

### UIViewPropertyAnimator (iOS 10+)

```objc
// Property Animator - ควบคุม animation ได้มากกว่า
UIViewPropertyAnimator *animator = 
    [[UIViewPropertyAnimator alloc] initWithDuration:0.5
                                               curve:UIViewAnimationCurveEaseOut
                                          animations:^{
    view.alpha = 0.0;
    view.transform = CGAffineTransformMakeTranslation(0, 100);
}];

// เพิ่ม animations หลายๆ ครั้ง
[animator addAnimations:^{
    view.backgroundColor = [UIColor systemRedColor];
}];

// Completion handler
[animator addCompletion:^(UIViewAnimatingPosition finalPosition) {
    if (finalPosition == UIViewAnimatingPositionEnd) {
        [view removeFromSuperview];
    }
}];

// เริ่ม animation
[animator startAnimation];

// Pause/Resume
[animator pauseAnimation];
[animator startAnimation];

// ควบคุม progress ด้วย slider/gesture
animator.fractionComplete = 0.5; // 50% ของ animation

// Reversible animation
animator.reversed = YES;

// Spring-based PropertyAnimator
UIViewPropertyAnimator *springAnimator = 
    [[UIViewPropertyAnimator alloc] 
        initWithDuration:0.5
        dampingRatio:0.7
        animations:^{
    view.center = targetPoint;
}];
[springAnimator startAnimation];

// ตัวอย่าง Interactive Animation กับ Pan Gesture
@property (nonatomic, strong) UIViewPropertyAnimator *animator;

- (void)handlePan:(UIPanGestureRecognizer *)gesture {
    switch (gesture.state) {
        case UIGestureRecognizerStateBegan: {
            // สร้าง animator ใหม่
            self.animator = [[UIViewPropertyAnimator alloc] 
                initWithDuration:0.5
                dampingRatio:0.8
                animations:^{
                    self.cardView.center = self.dismissPosition;
                }];
            [self.animator pauseAnimation];
            break;
        }
        case UIGestureRecognizerStateChanged: {
            CGPoint translation = [gesture translationInView:self.view];
            CGFloat progress = fabs(translation.y) / 300.0;
            self.animator.fractionComplete = MIN(1.0, MAX(0, progress));
            break;
        }
        case UIGestureRecognizerStateEnded: {
            CGFloat velocity = [gesture velocityInView:self.view].y;
            if (self.animator.fractionComplete > 0.5 || velocity > 500) {
                [self.animator continueAnimationWithTimingParameters:nil 
                                                    durationFactor:1.0];
            } else {
                self.animator.reversed = YES;
                [self.animator continueAnimationWithTimingParameters:nil 
                                                    durationFactor:1.0];
            }
            break;
        }
        default: break;
    }
}
```

### Keyframe Animations

```objc
// Keyframe Animation - กำหนด keyframes
[UIView animateKeyframesWithDuration:2.0
                               delay:0
                             options:UIViewKeyframeAnimationOptionCalculationModeCubic
                          animations:^{
    
    // Keyframe ที่ 0% - 25%
    [UIView addKeyframeWithRelativeStartTime:0.0
                            relativeDuration:0.25
                                  animations:^{
        view.frame = CGRectMake(100, 100, 100, 100);
    }];
    
    // Keyframe ที่ 25% - 50%
    [UIView addKeyframeWithRelativeStartTime:0.25
                            relativeDuration:0.25
                                  animations:^{
        view.frame = CGRectMake(200, 200, 150, 150);
        view.backgroundColor = [UIColor systemRedColor];
    }];
    
    // Keyframe ที่ 50% - 75%
    [UIView addKeyframeWithRelativeStartTime:0.5
                            relativeDuration:0.25
                                  animations:^{
        view.transform = CGAffineTransformMakeRotation(M_PI);
    }];
    
    // Keyframe ที่ 75% - 100%
    [UIView addKeyframeWithRelativeStartTime:0.75
                            relativeDuration:0.25
                                  animations:^{
        view.frame = CGRectMake(50, 300, 100, 100);
        view.transform = CGAffineTransformIdentity;
        view.backgroundColor = [UIColor systemBlueColor];
    }];
    
} completion:^(BOOL finished) {
    NSLog(@"Keyframe animation เสร็จ");
}];
```

---

## 44.5 Core Animation

### CALayer และ CABasicAnimation

```objc
// CABasicAnimation - animate layer property
CABasicAnimation *animation = [CABasicAnimation animationWithKeyPath:@"opacity"];
animation.fromValue = @1.0;
animation.toValue = @0.0;
animation.duration = 1.0;
animation.repeatCount = HUGE_VALF;  // วนไม่จำกัด
animation.autoreverses = YES;
animation.timingFunction = [CAMediaTimingFunction 
    functionWithName:kCAMediaTimingFunctionEaseInEaseOut];

[view.layer addAnimation:animation forKey:@"pulseAnimation"];

// ลบ animation
[view.layer removeAnimationForKey:@"pulseAnimation"];
[view.layer removeAllAnimations];

// CAKeyframeAnimation
CAKeyframeAnimation *keyframeAnim = 
    [CAKeyframeAnimation animationWithKeyPath:@"position"];
keyframeAnim.values = @[
    [NSValue valueWithCGPoint:CGPointMake(50, 50)],
    [NSValue valueWithCGPoint:CGPointMake(150, 100)],
    [NSValue valueWithCGPoint:CGPointMake(250, 50)],
    [NSValue valueWithCGPoint:CGPointMake(350, 200)],
];
keyframeAnim.keyTimes = @[@0.0, @0.33, @0.66, @1.0];
keyframeAnim.duration = 2.0;
keyframeAnim.calculationMode = kCAAnimationCubicPaced;
[view.layer addAnimation:keyframeAnim forKey:@"moveAnimation"];

// CAAnimationGroup - รัน animations พร้อมกัน
CABasicAnimation *posAnim = [CABasicAnimation animationWithKeyPath:@"position.y"];
posAnim.fromValue = @100;
posAnim.toValue = @300;

CABasicAnimation *opacityAnim = [CABasicAnimation animationWithKeyPath:@"opacity"];
opacityAnim.fromValue = @1.0;
opacityAnim.toValue = @0.0;

CAAnimationGroup *group = [CAAnimationGroup animation];
group.animations = @[posAnim, opacityAnim];
group.duration = 0.5;
group.fillMode = kCAFillModeForwards;
group.removedOnCompletion = NO;
[view.layer addAnimation:group forKey:@"dropAndFade"];

// CASpringAnimation (iOS 9+)
CASpringAnimation *spring = [CASpringAnimation animationWithKeyPath:@"transform.scale"];
spring.fromValue = @0.5;
spring.toValue = @1.0;
spring.mass = 1.0;
spring.stiffness = 200;
spring.damping = 10;
spring.initialVelocity = 5;
spring.duration = spring.settlingDuration;
[view.layer addAnimation:spring forKey:@"springScale"];

// CATransition - transition ระหว่าง content
CATransition *transition = [CATransition animation];
transition.type = kCATransitionPush;
transition.subtype = kCATransitionFromRight;
transition.duration = 0.4;
transition.timingFunction = [CAMediaTimingFunction 
    functionWithName:kCAMediaTimingFunctionEaseInEaseOut];
[view.layer addTransition:transition];
// เปลี่ยน content หลังจากนี้จะมี transition
view.layer.contents = (__bridge id)[UIImage imageNamed:@"newImage"].CGImage;
```

### CAShapeLayer

```objc
// CAShapeLayer สำหรับวาด vector shapes
CAShapeLayer *shapeLayer = [CAShapeLayer layer];
shapeLayer.frame = CGRectMake(50, 50, 200, 200);

// Path
UIBezierPath *path = [UIBezierPath bezierPathWithOvalInRect:
    CGRectMake(0, 0, 200, 200)];
shapeLayer.path = path.CGPath;

// Fill
shapeLayer.fillColor = [UIColor systemBlueColor].CGColor;
shapeLayer.fillRule = kCAFillRuleNonZero;

// Stroke
shapeLayer.strokeColor = [UIColor systemOrangeColor].CGColor;
shapeLayer.lineWidth = 3.0;
shapeLayer.lineCap = kCALineCapRound;
shapeLayer.lineJoin = kCALineJoinRound;

// Dash pattern
shapeLayer.lineDashPattern = @[@8, @4];  // เส้นประ

// Stroke progress animation (draw path animation)
shapeLayer.strokeStart = 0.0;
shapeLayer.strokeEnd = 0.0;  // เริ่มจากไม่มีเส้น

[view.layer addSublayer:shapeLayer];

// Animate การวาดเส้น
CABasicAnimation *drawAnimation = 
    [CABasicAnimation animationWithKeyPath:@"strokeEnd"];
drawAnimation.fromValue = @0.0;
drawAnimation.toValue = @1.0;
drawAnimation.duration = 2.0;
drawAnimation.timingFunction = [CAMediaTimingFunction 
    functionWithName:kCAMediaTimingFunctionEaseInEaseOut];
[shapeLayer addAnimation:drawAnimation forKey:@"drawLine"];

// Loading Circle Animation
- (void)setupLoadingIndicator {
    CAShapeLayer *circle = [CAShapeLayer layer];
    circle.frame = CGRectMake(0, 0, 60, 60);
    circle.position = self.view.center;
    
    UIBezierPath *circlePath = [UIBezierPath bezierPathWithOvalInRect:
        CGRectMake(5, 5, 50, 50)];
    circle.path = circlePath.CGPath;
    circle.fillColor = [UIColor clearColor].CGColor;
    circle.strokeColor = [UIColor systemBlueColor].CGColor;
    circle.lineWidth = 4.0;
    circle.strokeStart = 0.0;
    circle.strokeEnd = 0.75;
    circle.lineCap = kCALineCapRound;
    
    [self.view.layer addSublayer:circle];
    
    // หมุน
    CABasicAnimation *rotation = 
        [CABasicAnimation animationWithKeyPath:@"transform.rotation.z"];
    rotation.toValue = @(2 * M_PI);
    rotation.duration = 1.0;
    rotation.repeatCount = HUGE_VALF;
    rotation.timingFunction = [CAMediaTimingFunction 
        functionWithName:kCAMediaTimingFunctionLinear];
    [circle addAnimation:rotation forKey:@"spin"];
}
```

---

## 44.6 Gesture Recognizers

### UITapGestureRecognizer

```objc
// Single Tap
UITapGestureRecognizer *singleTap = 
    [[UITapGestureRecognizer alloc] initWithTarget:self
                                            action:@selector(handleSingleTap:)];
singleTap.numberOfTapsRequired = 1;
singleTap.numberOfTouchesRequired = 1; // นิ้วเดียว
[view addGestureRecognizer:singleTap];

// Double Tap
UITapGestureRecognizer *doubleTap = 
    [[UITapGestureRecognizer alloc] initWithTarget:self
                                            action:@selector(handleDoubleTap:)];
doubleTap.numberOfTapsRequired = 2;
[view addGestureRecognizer:doubleTap];

// Single tap ต้องรอ double tap ก่อน
[singleTap requireGestureRecognizerToFail:doubleTap];

// 2-Finger Tap
UITapGestureRecognizer *twoFingerTap = 
    [[UITapGestureRecognizer alloc] initWithTarget:self
                                            action:@selector(handleTwoFingerTap:)];
twoFingerTap.numberOfTouchesRequired = 2;
[view addGestureRecognizer:twoFingerTap];

- (void)handleSingleTap:(UITapGestureRecognizer *)gesture {
    CGPoint location = [gesture locationInView:self.view];
    NSLog(@"Tap at: %.0f, %.0f", location.x, location.y);
    
    // ตำแหน่งใน view ที่ติด gesture
    CGPoint locationInView = [gesture locationInView:gesture.view];
}

- (void)handleDoubleTap:(UITapGestureRecognizer *)gesture {
    NSLog(@"Double Tap!");
    
    // Toggle scale
    if (CGAffineTransformIsIdentity(view.transform)) {
        [UIView animateWithDuration:0.3 animations:^{
            view.transform = CGAffineTransformMakeScale(2.0, 2.0);
        }];
    } else {
        [UIView animateWithDuration:0.3 animations:^{
            view.transform = CGAffineTransformIdentity;
        }];
    }
}
```

### UIPanGestureRecognizer

```objc
// Pan (ลาก)
UIPanGestureRecognizer *pan = 
    [[UIPanGestureRecognizer alloc] initWithTarget:self
                                            action:@selector(handlePan:)];
pan.minimumNumberOfTouches = 1;
pan.maximumNumberOfTouches = 1;
pan.delegate = self;
[view addGestureRecognizer:pan];

- (void)handlePan:(UIPanGestureRecognizer *)gesture {
    // Translation จาก starting position
    CGPoint translation = [gesture translationInView:self.view];
    
    // Velocity
    CGPoint velocity = [gesture velocityInView:self.view];
    
    UIView *dragView = gesture.view;
    
    switch (gesture.state) {
        case UIGestureRecognizerStateBegan:
            NSLog(@"เริ่มลาก");
            break;
            
        case UIGestureRecognizerStateChanged: {
            // ย้าย view
            CGPoint newCenter = CGPointMake(
                dragView.center.x + translation.x,
                dragView.center.y + translation.y
            );
            
            // จำกัดไม่ให้เกินขอบจอ
            newCenter.x = MAX(dragView.frame.size.width/2,
                MIN(self.view.bounds.size.width - dragView.frame.size.width/2, 
                    newCenter.x));
            newCenter.y = MAX(dragView.frame.size.height/2,
                MIN(self.view.bounds.size.height - dragView.frame.size.height/2, 
                    newCenter.y));
            
            dragView.center = newCenter;
            [gesture setTranslation:CGPointZero inView:self.view]; // Reset
            break;
        }
            
        case UIGestureRecognizerStateEnded:
        case UIGestureRecognizerStateCancelled: {
            NSLog(@"หยุดลาก - velocity: %.0f, %.0f", velocity.x, velocity.y);
            
            // Momentum (inertia) effect
            CGFloat dampingFactor = 0.7;
            CGPoint targetPoint = CGPointMake(
                dragView.center.x + velocity.x * dampingFactor,
                dragView.center.y + velocity.y * dampingFactor
            );
            
            [UIView animateWithDuration:0.5
                                  delay:0
                 usingSpringWithDamping:0.8
                  initialSpringVelocity:0.5
                                options:UIViewAnimationOptionCurveEaseOut
                             animations:^{
                dragView.center = targetPoint;
            } completion:nil];
            break;
        }
        default: break;
    }
}
```

### UIPinchGestureRecognizer

```objc
// Pinch (Zoom)
UIPinchGestureRecognizer *pinch = 
    [[UIPinchGestureRecognizer alloc] initWithTarget:self
                                              action:@selector(handlePinch:)];
[imageView addGestureRecognizer:pinch];

@property (nonatomic, assign) CGFloat currentScale;

- (void)viewDidLoad {
    [super viewDidLoad];
    self.currentScale = 1.0;
}

- (void)handlePinch:(UIPinchGestureRecognizer *)gesture {
    UIView *pinchView = gesture.view;
    
    switch (gesture.state) {
        case UIGestureRecognizerStateChanged: {
            CGFloat newScale = self.currentScale * gesture.scale;
            newScale = MAX(0.5, MIN(3.0, newScale)); // จำกัด scale
            pinchView.transform = CGAffineTransformMakeScale(newScale, newScale);
            break;
        }
        case UIGestureRecognizerStateEnded:
        case UIGestureRecognizerStateCancelled:
            self.currentScale = pinchView.transform.a; // a = scale x
            break;
        default: break;
    }
}
```

### UIRotationGestureRecognizer

```objc
// Rotation
UIRotationGestureRecognizer *rotation = 
    [[UIRotationGestureRecognizer alloc] initWithTarget:self
                                                 action:@selector(handleRotation:)];
[view addGestureRecognizer:rotation];

@property (nonatomic, assign) CGFloat currentRotation;

- (void)handleRotation:(UIRotationGestureRecognizer *)gesture {
    UIView *rotateView = gesture.view;
    
    switch (gesture.state) {
        case UIGestureRecognizerStateChanged:
            rotateView.transform = CGAffineTransformMakeRotation(
                self.currentRotation + gesture.rotation);
            break;
        case UIGestureRecognizerStateEnded:
        case UIGestureRecognizerStateCancelled:
            self.currentRotation += gesture.rotation;
            break;
        default: break;
    }
}
```

### UILongPressGestureRecognizer

```objc
// Long Press
UILongPressGestureRecognizer *longPress = 
    [[UILongPressGestureRecognizer alloc] initWithTarget:self
                                                  action:@selector(handleLongPress:)];
longPress.minimumPressDuration = 0.5;  // กี่วินาที
longPress.numberOfTapsRequired = 0;
longPress.numberOfTouchesRequired = 1;
longPress.allowableMovement = 10;  // เคลื่อนได้กี่ points
[view addGestureRecognizer:longPress];

- (void)handleLongPress:(UILongPressGestureRecognizer *)gesture {
    if (gesture.state == UIGestureRecognizerStateBegan) {
        NSLog(@"Long Press!");
        CGPoint location = [gesture locationInView:self.view];
        
        // Haptic Feedback
        UIImpactFeedbackGenerator *feedback = 
            [[UIImpactFeedbackGenerator alloc] 
                initWithStyle:UIImpactFeedbackStyleMedium];
        [feedback impactOccurred];
        
        // แสดง Context Menu
        [self showContextMenuAtLocation:location forView:gesture.view];
    }
}
```

### UISwipeGestureRecognizer

```objc
// Swipe
for (UISwipeGestureRecognizerDirection direction in @[
    @(UISwipeGestureRecognizerDirectionLeft),
    @(UISwipeGestureRecognizerDirectionRight),
    @(UISwipeGestureRecognizerDirectionUp),
    @(UISwipeGestureRecognizerDirectionDown)
]) {
    UISwipeGestureRecognizer *swipe = 
        [[UISwipeGestureRecognizer alloc] 
            initWithTarget:self action:@selector(handleSwipe:)];
    swipe.direction = direction.integerValue;
    [view addGestureRecognizer:swipe];
}

- (void)handleSwipe:(UISwipeGestureRecognizer *)gesture {
    NSString *directionName;
    switch (gesture.direction) {
        case UISwipeGestureRecognizerDirectionLeft:
            directionName = @"ซ้าย";
            break;
        case UISwipeGestureRecognizerDirectionRight:
            directionName = @"ขวา";
            break;
        case UISwipeGestureRecognizerDirectionUp:
            directionName = @"ขึ้น";
            break;
        case UISwipeGestureRecognizerDirectionDown:
            directionName = @"ลง";
            break;
    }
    NSLog(@"Swipe: %@", directionName);
}
```

### UIGestureRecognizerDelegate

```objc
@interface MyViewController () <UIGestureRecognizerDelegate>
@end

// อนุญาตหลาย gestures พร้อมกัน
- (BOOL)gestureRecognizer:(UIGestureRecognizer *)gestureRecognizer 
    shouldRecognizeSimultaneouslyWithGestureRecognizer:
        (UIGestureRecognizer *)otherGestureRecognizer {
    
    // อนุญาตให้ pinch และ rotate พร้อมกัน
    if ([gestureRecognizer isKindOfClass:[UIPinchGestureRecognizer class]] &&
        [otherGestureRecognizer isKindOfClass:[UIRotationGestureRecognizer class]]) {
        return YES;
    }
    return NO;
}

// ตรวจสอบก่อนเริ่ม gesture
- (BOOL)gestureRecognizerShouldBegin:(UIGestureRecognizer *)gestureRecognizer {
    if ([gestureRecognizer isKindOfClass:[UIPanGestureRecognizer class]]) {
        UIPanGestureRecognizer *pan = (UIPanGestureRecognizer *)gestureRecognizer;
        CGPoint velocity = [pan velocityInView:gestureRecognizer.view];
        // เฉพาะ horizontal pan เท่านั้น
        return fabs(velocity.x) > fabs(velocity.y);
    }
    return YES;
}

// ตรวจสอบว่า gesture ควรรับ touch
- (BOOL)gestureRecognizer:(UIGestureRecognizer *)gestureRecognizer
        shouldReceiveTouch:(UITouch *)touch {
    
    // ไม่รับ touch บน UIButton
    if ([touch.view isKindOfClass:[UIButton class]]) {
        return NO;
    }
    return YES;
}
```

---

## 44.7 Custom UIView Subclass

### สร้าง Custom UIView

```objc
// RatingView.h - Star Rating View

#import <UIKit/UIKit.h>

@class RatingView;

@protocol RatingViewDelegate <NSObject>
- (void)ratingView:(RatingView *)ratingView didChangeRating:(CGFloat)rating;
@end

@interface RatingView : UIView

@property (nonatomic, assign) CGFloat rating;
@property (nonatomic, assign) NSInteger maxRating;
@property (nonatomic, strong) UIColor *starColor;
@property (nonatomic, strong) UIColor *emptyStarColor;
@property (nonatomic, assign) BOOL userInteractionEnabled;
@property (nonatomic, weak) id<RatingViewDelegate> delegate;

- (instancetype)initWithMaxRating:(NSInteger)maxRating;

@end
```

```objc
// RatingView.m

#import "RatingView.h"

@interface RatingView ()
@property (nonatomic, strong) NSMutableArray<CAShapeLayer *> *starLayers;
@end

@implementation RatingView

- (instancetype)initWithMaxRating:(NSInteger)maxRating {
    self = [super init];
    if (self) {
        _maxRating = maxRating > 0 ? maxRating : 5;
        _rating = 0;
        _starColor = [UIColor systemYellowColor];
        _emptyStarColor = [UIColor systemGray4Color];
        _userInteractionEnabled = YES;
        
        [self setupStars];
    }
    return self;
}

- (instancetype)initWithFrame:(CGRect)frame {
    self = [super initWithFrame:frame];
    if (self) {
        _maxRating = 5;
        _rating = 0;
        _starColor = [UIColor systemYellowColor];
        _emptyStarColor = [UIColor systemGray4Color];
        _userInteractionEnabled = YES;
        
        [self setupStars];
    }
    return self;
}

- (void)setupStars {
    self.starLayers = [NSMutableArray array];
    self.backgroundColor = [UIColor clearColor];
}

// Intrinsic Content Size
- (CGSize)intrinsicContentSize {
    CGFloat starSize = 30;
    CGFloat spacing = 4;
    return CGSizeMake(self.maxRating * starSize + (self.maxRating - 1) * spacing, starSize);
}

- (void)layoutSubviews {
    [super layoutSubviews];
    [self updateStarLayers];
}

- (void)updateStarLayers {
    // ลบ layers เก่า
    for (CAShapeLayer *layer in self.starLayers) {
        [layer removeFromSuperlayer];
    }
    [self.starLayers removeAllObjects];
    
    CGFloat starSize = MIN(self.bounds.size.height, 
                          (self.bounds.size.width - (self.maxRating - 1) * 4) / self.maxRating);
    CGFloat spacing = (self.bounds.size.width - self.maxRating * starSize) / (self.maxRating - 1);
    
    for (NSInteger i = 0; i < self.maxRating; i++) {
        CAShapeLayer *starLayer = [CAShapeLayer layer];
        
        CGFloat x = i * (starSize + spacing);
        starLayer.frame = CGRectMake(x, 0, starSize, starSize);
        starLayer.path = [self starPathInRect:CGRectMake(0, 0, starSize, starSize)].CGPath;
        
        // สีตาม rating
        CGFloat fillAmount = MIN(1.0, MAX(0.0, self.rating - i));
        if (fillAmount >= 1.0) {
            starLayer.fillColor = self.starColor.CGColor;
        } else if (fillAmount > 0) {
            // Half star
            starLayer.fillColor = [self.starColor colorWithAlphaComponent:fillAmount].CGColor;
        } else {
            starLayer.fillColor = self.emptyStarColor.CGColor;
        }
        
        [self.layer addSublayer:starLayer];
        [self.starLayers addObject:starLayer];
    }
}

// สร้าง star path
- (UIBezierPath *)starPathInRect:(CGRect)rect {
    CGFloat centerX = CGRectGetMidX(rect);
    CGFloat centerY = CGRectGetMidY(rect);
    CGFloat outerRadius = MIN(rect.size.width, rect.size.height) / 2 * 0.9;
    CGFloat innerRadius = outerRadius * 0.4;
    NSInteger points = 5;
    
    UIBezierPath *path = [UIBezierPath bezierPath];
    
    for (NSInteger i = 0; i < points * 2; i++) {
        CGFloat radius = (i % 2 == 0) ? outerRadius : innerRadius;
        CGFloat angle = (CGFloat)i * M_PI / points - M_PI / 2;
        CGFloat x = centerX + radius * cos(angle);
        CGFloat y = centerY + radius * sin(angle);
        
        if (i == 0) {
            [path moveToPoint:CGPointMake(x, y)];
        } else {
            [path addLineToPoint:CGPointMake(x, y)];
        }
    }
    
    [path closePath];
    return path;
}

// Touch handling
- (void)touchesBegan:(NSSet<UITouch *> *)touches withEvent:(UIEvent *)event {
    if (!self.userInteractionEnabled) return;
    [self updateRatingWithTouches:touches];
}

- (void)touchesMoved:(NSSet<UITouch *> *)touches withEvent:(UIEvent *)event {
    if (!self.userInteractionEnabled) return;
    [self updateRatingWithTouches:touches];
}

- (void)touchesEnded:(NSSet<UITouch *> *)touches withEvent:(UIEvent *)event {
    if (!self.userInteractionEnabled) return;
    [self.delegate ratingView:self didChangeRating:self.rating];
    
    // Haptic feedback
    UISelectionFeedbackGenerator *feedback = [[UISelectionFeedbackGenerator alloc] init];
    [feedback selectionChanged];
}

- (void)updateRatingWithTouches:(NSSet<UITouch *> *)touches {
    UITouch *touch = touches.anyObject;
    CGPoint location = [touch locationInView:self];
    
    CGFloat starWidth = self.bounds.size.width / self.maxRating;
    CGFloat newRating = location.x / starWidth;
    
    newRating = MAX(0, MIN(self.maxRating, ceilf(newRating)));
    
    if (newRating != self.rating) {
        self.rating = newRating;
    }
}

- (void)setRating:(CGFloat)rating {
    _rating = MAX(0, MIN(self.maxRating, rating));
    [self updateStarLayers];
}

@end
```

---

## 44.8 Responder Chain

### Responder Chain คืออะไร?

Responder Chain คือ sequence ของ objects ที่รับและจัดการ events โดยจะส่งต่อจาก View ไปถึง ViewController และ Application

```
UIView (First Responder)
    ↓ (ถ้าไม่จัดการ)
UIView (Superview)
    ↓
...
    ↓
UIViewController
    ↓
UIWindow
    ↓
UIApplication
    ↓
AppDelegate
```

### ตัวอย่างการใช้ Responder Chain

```objc
// UIView เป็น UIResponder
@interface CustomView : UIView
@end

@implementation CustomView

// ตรวจสอบว่า view นี้สามารถเป็น first responder ได้
- (BOOL)canBecomeFirstResponder {
    return YES;
}

// เมื่อกลายเป็น first responder
- (BOOL)becomeFirstResponder {
    BOOL became = [super becomeFirstResponder];
    if (became) {
        self.layer.borderWidth = 2.0;
        self.layer.borderColor = [UIColor systemBlueColor].CGColor;
    }
    return became;
}

// เมื่อหมดจากการเป็น first responder
- (BOOL)resignFirstResponder {
    BOOL resigned = [super resignFirstResponder];
    if (resigned) {
        self.layer.borderWidth = 0;
    }
    return resigned;
}

@end

// ส่ง Custom Actions ผ่าน Responder Chain
// สร้าง Protocol สำหรับ actions
@protocol CustomMenuActions <NSObject>
- (void)handleCopyAction:(id)sender;
- (void)handleShareAction:(id)sender;
@end

// ส่ง action ขึ้นไปใน responder chain
UIApplication *app = [UIApplication sharedApplication];
[app sendAction:@selector(handleCopyAction:) 
             to:nil    // nil = ส่งขึ้นไปใน chain
           from:self 
       forEvent:nil];

// ตรวจสอบว่า responder chain มี handler สำหรับ action
BOOL canHandle = [[UIApplication sharedApplication] 
    canPerformAction:@selector(handleCopyAction:) 
          withSender:self];
```

---

## 44.9 Touch Handling

### touchesBegan, touchesMoved, touchesEnded, touchesCancelled

```objc
// Custom Drawing View ที่รับ Touch
@interface DrawingView : UIView

@property (nonatomic, strong) UIColor *strokeColor;
@property (nonatomic, assign) CGFloat lineWidth;

- (void)clear;

@end

@implementation DrawingView {
    NSMutableArray<UIBezierPath *> *_paths;
    UIBezierPath *_currentPath;
}

- (instancetype)initWithFrame:(CGRect)frame {
    self = [super initWithFrame:frame];
    if (self) {
        _paths = [NSMutableArray array];
        _strokeColor = [UIColor blackColor];
        _lineWidth = 3.0;
        self.backgroundColor = [UIColor whiteColor];
        
        // รับ Multi-touch
        self.multipleTouchEnabled = YES;
    }
    return self;
}

// เริ่ม touch
- (void)touchesBegan:(NSSet<UITouch *> *)touches withEvent:(UIEvent *)event {
    UITouch *touch = touches.anyObject;
    CGPoint point = [touch locationInView:self];
    
    // สร้าง path ใหม่
    _currentPath = [UIBezierPath bezierPath];
    _currentPath.lineWidth = self.lineWidth;
    _currentPath.lineCapStyle = kCGLineCapRound;
    _currentPath.lineJoinStyle = kCGLineJoinRound;
    [_currentPath moveToPoint:point];
    
    [_paths addObject:_currentPath];
    [self setNeedsDisplay];
}

// ลาก touch
- (void)touchesMoved:(NSSet<UITouch *> *)touches withEvent:(UIEvent *)event {
    UITouch *touch = touches.anyObject;
    CGPoint current = [touch locationInView:self];
    CGPoint previous = [touch previousLocationInView:self];
    
    // ใช้ smooth curve แทน line ตรง
    CGPoint midPoint = CGPointMake(
        (current.x + previous.x) / 2,
        (current.y + previous.y) / 2
    );
    
    [_currentPath addQuadCurveToPoint:midPoint controlPoint:previous];
    [self setNeedsDisplay];
}

// สิ้นสุด touch
- (void)touchesEnded:(NSSet<UITouch *> *)touches withEvent:(UIEvent *)event {
    UITouch *touch = touches.anyObject;
    CGPoint point = [touch locationInView:self];
    [_currentPath addLineToPoint:point];
    
    _currentPath = nil;
    [self setNeedsDisplay];
}

// ยกเลิก (เช่น รับสายโทรศัพท์)
- (void)touchesCancelled:(NSSet<UITouch *> *)touches withEvent:(UIEvent *)event {
    // ลบ path สุดท้ายออก
    if (_paths.count > 0) {
        [_paths removeLastObject];
    }
    _currentPath = nil;
    [self setNeedsDisplay];
}

// วาด
- (void)drawRect:(CGRect)rect {
    for (UIBezierPath *path in _paths) {
        [self.strokeColor setStroke];
        [path stroke];
    }
}

- (void)clear {
    [_paths removeAllObjects];
    [self setNeedsDisplay];
}

// Undo
- (void)undo {
    if (_paths.count > 0) {
        [_paths removeLastObject];
        [self setNeedsDisplay];
    }
}

@end
```

### Haptic Feedback

```objc
// UIImpactFeedbackGenerator
UIImpactFeedbackGenerator *impact = [[UIImpactFeedbackGenerator alloc] 
    initWithStyle:UIImpactFeedbackStyleLight];
[impact prepare]; // เตรียมก่อนใช้ เพื่อลด latency
[impact impactOccurred];

UIImpactFeedbackGenerator *mediumImpact = [[UIImpactFeedbackGenerator alloc]
    initWithStyle:UIImpactFeedbackStyleMedium];
[mediumImpact impactOccurred];

UIImpactFeedbackGenerator *heavyImpact = [[UIImpactFeedbackGenerator alloc]
    initWithStyle:UIImpactFeedbackStyleHeavy];
[heavyImpact impactOccurred];

// UISelectionFeedbackGenerator (สำหรับ selection change)
UISelectionFeedbackGenerator *selection = [[UISelectionFeedbackGenerator alloc] init];
[selection prepare];
[selection selectionChanged];

// UINotificationFeedbackGenerator (Success, Warning, Error)
UINotificationFeedbackGenerator *notification = [[UINotificationFeedbackGenerator alloc] init];
[notification notificationOccurred:UINotificationFeedbackTypeSuccess];
[notification notificationOccurred:UINotificationFeedbackTypeWarning];
[notification notificationOccurred:UINotificationFeedbackTypeError];
```

---

## 44.10 Practice: Interactive Draggable Cards

```objc
// DraggableCardView.h
@interface DraggableCardView : UIView

@property (nonatomic, strong) UILabel *titleLabel;
@property (nonatomic, strong) UILabel *subtitleLabel;
@property (nonatomic, strong) UIImageView *imageView;

+ (instancetype)cardWithTitle:(NSString *)title 
                     subtitle:(NSString *)subtitle
                   imageName:(NSString *)imageName;
@end
```

```objc
// DraggableCardView.m
#import "DraggableCardView.h"

@interface DraggableCardView ()
@property (nonatomic, assign) CGPoint originalCenter;
@end

@implementation DraggableCardView

+ (instancetype)cardWithTitle:(NSString *)title 
                     subtitle:(NSString *)subtitle
                   imageName:(NSString *)imageName {
    DraggableCardView *card = [[DraggableCardView alloc] 
        initWithFrame:CGRectMake(0, 0, 300, 400)];
    card.titleLabel.text = title;
    card.subtitleLabel.text = subtitle;
    card.imageView.image = [UIImage imageNamed:imageName];
    return card;
}

- (instancetype)initWithFrame:(CGRect)frame {
    self = [super initWithFrame:frame];
    if (self) {
        [self setupUI];
        [self setupGestures];
    }
    return self;
}

- (void)setupUI {
    self.backgroundColor = [UIColor systemBackgroundColor];
    self.layer.cornerRadius = 20;
    self.layer.shadowColor = [UIColor blackColor].CGColor;
    self.layer.shadowOpacity = 0.2;
    self.layer.shadowOffset = CGSizeMake(0, 8);
    self.layer.shadowRadius = 16;
    
    // Image View
    self.imageView = [[UIImageView alloc] init];
    self.imageView.translatesAutoresizingMaskIntoConstraints = NO;
    self.imageView.contentMode = UIViewContentModeScaleAspectFill;
    self.imageView.clipsToBounds = YES;
    self.imageView.layer.cornerRadius = 16;
    self.imageView.layer.maskedCorners = 
        kCALayerMinXMinYCorner | kCALayerMaxXMinYCorner;
    [self addSubview:self.imageView];
    
    // Title
    self.titleLabel = [[UILabel alloc] init];
    self.titleLabel.translatesAutoresizingMaskIntoConstraints = NO;
    self.titleLabel.font = [UIFont boldSystemFontOfSize:20];
    [self addSubview:self.titleLabel];
    
    // Subtitle
    self.subtitleLabel = [[UILabel alloc] init];
    self.subtitleLabel.translatesAutoresizingMaskIntoConstraints = NO;
    self.subtitleLabel.font = [UIFont systemFontOfSize:14];
    self.subtitleLabel.textColor = [UIColor secondaryLabelColor];
    self.subtitleLabel.numberOfLines = 2;
    [self addSubview:self.subtitleLabel];
    
    [NSLayoutConstraint activateConstraints:@[
        [self.imageView.topAnchor constraintEqualToAnchor:self.topAnchor],
        [self.imageView.leadingAnchor constraintEqualToAnchor:self.leadingAnchor],
        [self.imageView.trailingAnchor constraintEqualToAnchor:self.trailingAnchor],
        [self.imageView.heightAnchor constraintEqualToAnchor:self.heightAnchor 
                                                  multiplier:0.65],
        
        [self.titleLabel.topAnchor constraintEqualToAnchor:self.imageView.bottomAnchor 
                                                  constant:16],
        [self.titleLabel.leadingAnchor constraintEqualToAnchor:self.leadingAnchor 
                                                      constant:16],
        [self.titleLabel.trailingAnchor constraintEqualToAnchor:self.trailingAnchor 
                                                       constant:-16],
        
        [self.subtitleLabel.topAnchor constraintEqualToAnchor:self.titleLabel.bottomAnchor 
                                                     constant:8],
        [self.subtitleLabel.leadingAnchor constraintEqualToAnchor:self.leadingAnchor 
                                                        constant:16],
        [self.subtitleLabel.trailingAnchor constraintEqualToAnchor:self.trailingAnchor 
                                                         constant:-16],
    ]];
}

- (void)setupGestures {
    UIPanGestureRecognizer *pan = [[UIPanGestureRecognizer alloc]
        initWithTarget:self action:@selector(handlePan:)];
    [self addGestureRecognizer:pan];
}

- (void)handlePan:(UIPanGestureRecognizer *)gesture {
    CGPoint translation = [gesture translationInView:self.superview];
    
    switch (gesture.state) {
        case UIGestureRecognizerStateBegan:
            self.originalCenter = self.center;
            // Scale up เล็กน้อยเมื่อยกขึ้น
            [UIView animateWithDuration:0.1 animations:^{
                self.transform = CGAffineTransformMakeScale(1.05, 1.05);
                self.layer.shadowOpacity = 0.3;
                self.layer.shadowRadius = 24;
            }];
            break;
            
        case UIGestureRecognizerStateChanged: {
            self.center = CGPointMake(
                self.originalCenter.x + translation.x,
                self.originalCenter.y + translation.y
            );
            
            // หมุนตามทิศทาง
            CGFloat rotation = translation.x / 300.0 * 0.4; // max ±0.4 radians
            self.transform = CGAffineTransformConcat(
                CGAffineTransformMakeScale(1.05, 1.05),
                CGAffineTransformMakeRotation(rotation)
            );
            break;
        }
            
        case UIGestureRecognizerStateEnded:
        case UIGestureRecognizerStateCancelled: {
            CGFloat velocity = [gesture velocityInView:self.superview].x;
            CGFloat distanceX = self.center.x - self.originalCenter.x;
            
            BOOL shouldDismissRight = distanceX > 100 || velocity > 800;
            BOOL shouldDismissLeft = distanceX < -100 || velocity < -800;
            
            if (shouldDismissRight || shouldDismissLeft) {
                // Swipe off screen
                CGFloat targetX = shouldDismissRight ? 
                    self.superview.bounds.size.width + 200 : -200;
                
                [UIView animateWithDuration:0.4 
                                      delay:0
                     usingSpringWithDamping:0.9
                      initialSpringVelocity:velocity / 1000
                                    options:0
                                 animations:^{
                    self.center = CGPointMake(targetX, self.center.y + 100);
                    self.transform = CGAffineTransformConcat(
                        CGAffineTransformMakeScale(0.8, 0.8),
                        CGAffineTransformMakeRotation(shouldDismissRight ? 0.5 : -0.5)
                    );
                    self.alpha = 0;
                } completion:^(BOOL finished) {
                    [self removeFromSuperview];
                }];
            } else {
                // กลับที่เดิม (spring)
                [UIView animateWithDuration:0.5
                                      delay:0
                     usingSpringWithDamping:0.7
                      initialSpringVelocity:0.5
                                    options:0
                                 animations:^{
                    self.center = self.originalCenter;
                    self.transform = CGAffineTransformIdentity;
                    self.layer.shadowOpacity = 0.2;
                    self.layer.shadowRadius = 16;
                } completion:nil];
            }
            break;
        }
        default: break;
    }
}

@end
```

---

## 44.11 แบบฝึกหัด

### แบบฝึกหัดที่ 1: Custom Progress Ring

```objc
/*
สร้าง CustomProgressRingView ที่:
- วาดวงกลมด้วย CAShapeLayer
- มี property 'progress' (0.0-1.0)
- แสดง % ตรงกลาง
- มี animation เมื่อ progress เปลี่ยน
- รองรับ custom colors
- มี pulse animation เมื่อ progress = 1.0
*/
```

### แบบฝึกหัดที่ 2: Interactive Image Viewer

```objc
/*
สร้าง ImageViewerViewController ที่:
- แสดงรูปภาพขนาดใหญ่
- Pinch to Zoom (min 0.5x, max 5x)
- Pan ลากรูปได้
- Double Tap reset zoom
- Single Tap ซ่อน/แสดง UI elements
- Dismiss ด้วย swipe down
*/
```

### แบบฝึกหัดที่ 3: Drawing App

```objc
/*
สร้าง Drawing App ที่มี:
- DrawingView: วาดด้วย finger
- Toolbar: เลือกสี, ขนาด brush, eraser
- Undo/Redo (stack)
- Export เป็น UIImage
- UIImageWriteToSavedPhotosAlbum บันทึกลงเครื่อง
*/

// Hint: การ Export UIView เป็น UIImage
- (UIImage *)exportAsImage {
    UIGraphicsBeginImageContextWithOptions(self.bounds.size, 
                                          self.opaque, 
                                          [UIScreen mainScreen].scale);
    [self drawViewHierarchyInRect:self.bounds afterScreenUpdates:YES];
    UIImage *image = UIGraphicsGetImageFromCurrentImageContext();
    UIGraphicsEndImageContext();
    return image;
}

// หรือใช้ UIGraphicsImageRenderer (iOS 10+)
- (UIImage *)exportAsImageModern {
    UIGraphicsImageRenderer *renderer = [[UIGraphicsImageRenderer alloc] 
        initWithBounds:self.bounds];
    return [renderer imageWithActions:^(UIGraphicsImageRendererContext *ctx) {
        [self drawViewHierarchyInRect:self.bounds afterScreenUpdates:YES];
    }];
}
```

---

## 44.12 View Transitions

### UIView Transition Animations

```objc
// Transition ระหว่าง views ใน container
UIView *container = self.containerView;
UIView *fromView = self.currentView;
UIView *toView = self.nextView;

[UIView transitionFromView:fromView
                    toView:toView
                  duration:0.4
                   options:UIViewAnimationOptionTransitionCrossDissolve
               completion:^(BOOL finished) {
    NSLog(@"Transition เสร็จ");
}];

// เปลี่ยน content ใน view เดิม
[UIView transitionWithView:self.imageView
                  duration:0.3
                   options:UIViewAnimationOptionTransitionCrossDissolve
               animations:^{
    self.imageView.image = newImage;
} completion:nil];

// Flip Card Animation
- (void)flipCard {
    UIView *frontView = self.frontCardView;
    UIView *backView = self.backCardView;
    
    BOOL isShowingFront = !frontView.isHidden;
    
    [UIView transitionWithView:self.cardContainer
                      duration:0.6
                       options:UIViewAnimationOptionTransitionFlipFromRight
                   animations:^{
        frontView.hidden = !isShowingFront;
        backView.hidden = isShowingFront;
    } completion:nil];
}
```

---

## 44.13 Dark Mode Support

### รองรับ Dark Mode ใน Custom View

```objc
@implementation ThemedView

- (void)traitCollectionDidChange:(UITraitCollection *)previousTraitCollection {
    [super traitCollectionDidChange:previousTraitCollection];
    
    // เรียกเมื่อ Dark Mode เปลี่ยน
    if ([self.traitCollection hasDifferentColorAppearanceComparedToTraitCollection:
             previousTraitCollection]) {
        [self updateColors];
    }
}

- (void)updateColors {
    BOOL isDark = self.traitCollection.userInterfaceStyle == UIUserInterfaceStyleDark;
    
    self.backgroundColor = isDark ? 
        [UIColor colorWithRed:0.1 green:0.1 blue:0.1 alpha:1.0] :
        [UIColor whiteColor];
    
    // หรือใช้ Semantic Colors (อัตโนมัติ)
    self.backgroundColor = [UIColor systemBackgroundColor];
    self.titleLabel.textColor = [UIColor labelColor];
    
    // CGColor ต้องอัปเดตด้วยมือ
    self.layer.borderColor = [UIColor systemGrayColor].CGColor;
    
    // ถ้าวาดใน drawRect: ต้องสั่งวาดใหม่
    [self setNeedsDisplay];
}

// Override drawRect: ด้วย Semantic Colors
- (void)drawRect:(CGRect)rect {
    // Semantic colors รู้ Dark/Light Mode เองอัตโนมัติ
    [[UIColor systemBackgroundColor] setFill];
    UIRectFill(rect);
    
    [[UIColor labelColor] setStroke];
    // วาด...
}

@end
```

---

## 44.14 สรุปและ Best Practices

### สรุปหัวข้อที่เรียน

1. **UIView Properties** - frame, bounds, center, transform, alpha, layer
2. **View Hierarchy** - addSubview, removeFromSuperview, z-order, coordinate conversion
3. **View Drawing** - drawRect:, UIBezierPath, Core Graphics
4. **UIView Animations** - block animations, spring animations, keyframes
5. **UIViewPropertyAnimator** - interactive animations
6. **Core Animation** - CABasicAnimation, CAKeyframeAnimation, CAShapeLayer
7. **Gesture Recognizers** - Tap, Pan, Pinch, Rotation, LongPress, Swipe
8. **UIGestureRecognizerDelegate** - simultaneous gestures, conflicts
9. **Custom UIView** - subclass, intrinsic content size, custom drawing
10. **Responder Chain** - event propagation
11. **Touch Handling** - touchesBegan, touchesMoved, touchesEnded
12. **Haptic Feedback** - UIImpactFeedbackGenerator, UINotificationFeedbackGenerator
13. **Dark Mode** - traitCollectionDidChange, Semantic Colors

### Best Practices

```objc
// 1. ใช้ Semantic Colors เสมอ
// ❌ อย่าใช้ hardcoded colors
view.backgroundColor = [UIColor whiteColor];
// ✅ ใช้ semantic colors
view.backgroundColor = [UIColor systemBackgroundColor];

// 2. ตั้ง translatesAutoresizingMaskIntoConstraints = NO ก่อน Auto Layout
myView.translatesAutoresizingMaskIntoConstraints = NO;

// 3. อัปเดต UI บน Main Thread เสมอ
dispatch_async(dispatch_get_main_queue(), ^{
    self.label.text = @"Updated";
});

// 4. ใช้ setNeedsDisplay/setNeedsLayout แทนการเรียก draw/layout ตรงๆ
[myView setNeedsDisplay];   // จะเรียก drawRect: ในรอบถัดไป
[myView setNeedsLayout];    // จะเรียก layoutSubviews ในรอบถัดไป
[myView layoutIfNeeded];    // บังคับ layout ทันที (ใช้ใน animation)

// 5. Release resources ใน dealloc
- (void)dealloc {
    [[NSNotificationCenter defaultCenter] removeObserver:self];
    [self.timer invalidate];
    self.timer = nil;
}

// 6. ใช้ weak reference สำหรับ delegate
@property (nonatomic, weak) id<MyViewDelegate> delegate;

// 7. Override intrinsicContentSize สำหรับ custom views
- (CGSize)intrinsicContentSize {
    return CGSizeMake(100, 44);
}

// 8. invalidateIntrinsicContentSize เมื่อขนาดเปลี่ยน
- (void)setText:(NSString *)text {
    _text = text;
    [self setNeedsDisplay];
    [self invalidateIntrinsicContentSize];
}
```

---

*ตอนที่ 44 จบแล้ว - ไปต่อที่ตอนที่ 45: Table Views*
