# ส่วนที่ 59: Core Animation ใน Objective-C

## บทนำ

Core Animation เป็น framework ที่ทรงพลังสำหรับสร้าง animation ระดับ layer บน iOS ซึ่งทำงานบน GPU ทำให้ได้ประสิทธิภาพสูงและ smooth ที่ 60fps หรือมากกว่า

---

## 1. CALayer พื้นฐาน

### 1.1 ทำความเข้าใจ CALayer

ทุก UIView มี `CALayer` ที่เป็น backing layer ซึ่งเป็น model object ที่กำหนดว่า view จะดูเป็นอย่างไร

```objc
#import <QuartzCore/QuartzCore.h>

// เข้าถึง layer ของ view
CALayer *viewLayer = self.view.layer;
NSLog(@"Layer class: %@", NSStringFromClass([viewLayer class]));

// ตั้งค่า properties พื้นฐาน
viewLayer.backgroundColor = [UIColor systemBlueColor].CGColor;
viewLayer.cornerRadius = 12.0;
viewLayer.borderWidth = 2.0;
viewLayer.borderColor = [UIColor whiteColor].CGColor;
viewLayer.shadowColor = [UIColor blackColor].CGColor;
viewLayer.shadowOffset = CGSizeMake(0, 3);
viewLayer.shadowOpacity = 0.3;
viewLayer.shadowRadius = 5.0;
```

### 1.2 สร้าง Layer แบบ Standalone

```objc
- (void)createStandaloneLayer {
    CALayer *layer = [CALayer layer];
    layer.frame = CGRectMake(50, 100, 200, 150);
    layer.backgroundColor = [UIColor systemBlueColor].CGColor;
    layer.cornerRadius = 16.0;
    layer.masksToBounds = YES;
    
    // เพิ่ม sublayer
    CALayer *innerLayer = [CALayer layer];
    innerLayer.frame = CGRectMake(20, 20, 60, 60);
    innerLayer.backgroundColor = [UIColor whiteColor].CGColor;
    innerLayer.cornerRadius = 30.0;
    [layer addSublayer:innerLayer];
    
    // เพิ่มเข้า view's layer
    [self.view.layer addSublayer:layer];
}
```

### 1.3 CALayer Properties

```objc
- (void)demonstrateLayerProperties {
    CALayer *layer = [CALayer layer];
    layer.frame = CGRectMake(50, 50, 200, 200);
    
    // Visual properties
    layer.backgroundColor = [UIColor systemBlueColor].CGColor;
    layer.opacity = 0.8;
    layer.isHidden = NO;
    
    // Geometry
    layer.frame = CGRectMake(50, 50, 200, 200);
    layer.bounds = CGRectMake(0, 0, 200, 200);
    layer.position = CGPointMake(150, 150); // center point
    layer.anchorPoint = CGPointMake(0.5, 0.5); // default: center
    layer.zPosition = 10.0;
    
    // Rounded corners
    layer.cornerRadius = 20.0;
    layer.maskedCorners = kCALayerMinXMinYCorner | kCALayerMaxXMinYCorner; // บนสองมุม
    
    // Border
    layer.borderWidth = 3.0;
    layer.borderColor = [UIColor whiteColor].CGColor;
    
    // Shadow
    layer.shadowColor = [UIColor blackColor].CGColor;
    layer.shadowOffset = CGSizeMake(2, 4);
    layer.shadowOpacity = 0.5;
    layer.shadowRadius = 8.0;
    layer.shadowPath = [UIBezierPath bezierPathWithRoundedRect:layer.bounds 
                                                  cornerRadius:20].CGPath;
    
    // Content
    UIImage *image = [UIImage imageNamed:@"photo"];
    layer.contents = (__bridge id)image.CGImage;
    layer.contentsGravity = kCAGravityResizeAspectFill;
    layer.contentsScale = [UIScreen mainScreen].scale;
    
    // Masks bounds
    layer.masksToBounds = YES;
    
    [self.view.layer addSublayer:layer];
}
```

### 1.4 Custom CALayer Subclass

```objc
// GradientLayer.h
@interface GradientLayer : CALayer
@property (nonatomic, strong) UIColor *startColor;
@property (nonatomic, strong) UIColor *endColor;
@end

// GradientLayer.m
@implementation GradientLayer

+ (BOOL)needsDisplayForKey:(NSString *)key {
    if ([key isEqualToString:@"startColor"] || [key isEqualToString:@"endColor"]) {
        return YES;
    }
    return [super needsDisplayForKey:key];
}

- (void)drawInContext:(CGContextRef)ctx {
    CGColorSpaceRef colorSpace = CGColorSpaceCreateDeviceRGB();
    NSArray *colors = @[
        (__bridge id)(self.startColor ?: [UIColor systemBlueColor]).CGColor,
        (__bridge id)(self.endColor ?: [UIColor systemPurpleColor]).CGColor
    ];
    
    CGGradientRef gradient = CGGradientCreateWithColors(
        colorSpace, (__bridge CFArrayRef)colors, NULL);
    
    CGContextDrawLinearGradient(ctx, gradient,
                                CGPointMake(0, 0),
                                CGPointMake(0, self.bounds.size.height),
                                0);
    
    CGGradientRelease(gradient);
    CGColorSpaceRelease(colorSpace);
}

@end
```

---

## 2. CAAnimation

### 2.1 Animation พื้นฐาน

```objc
// CAAnimation เป็น abstract base class
// ลูกคลาสที่ใช้จริง:
// - CABasicAnimation
// - CAKeyframeAnimation
// - CAAnimationGroup
// - CATransition

// Properties ทั่วไปของ CAAnimation
CABasicAnimation *anim = [CABasicAnimation animationWithKeyPath:@"opacity"];
anim.duration = 1.0;           // ความยาวของ animation (วินาที)
anim.beginTime = 0.0;          // delay ก่อนเริ่ม
anim.repeatCount = 3;          // จำนวนรอบ (INFINITY = วนซ้ำตลอด)
anim.autoreverses = YES;       // เล่นย้อนกลับ
anim.fillMode = kCAFillModeForwards; // ค้างค่าสุดท้าย
anim.removedOnCompletion = NO; // ไม่ remove animation เมื่อจบ (ใช้กับ fillMode)
anim.timingFunction = [CAMediaTimingFunction functionWithName:kCAMediaTimingFunctionEaseInEaseOut];
```

---

## 3. CABasicAnimation

### 3.1 Animate Properties พื้นฐาน

```objc
- (void)animateOpacity {
    CABasicAnimation *fadeOut = [CABasicAnimation animationWithKeyPath:@"opacity"];
    fadeOut.fromValue = @1.0;
    fadeOut.toValue = @0.0;
    fadeOut.duration = 1.0;
    fadeOut.autoreverses = YES;
    fadeOut.repeatCount = HUGE_VALF;
    
    [self.myLayer addAnimation:fadeOut forKey:@"opacityPulse"];
}

- (void)animatePosition {
    CABasicAnimation *move = [CABasicAnimation animationWithKeyPath:@"position"];
    move.fromValue = [NSValue valueWithCGPoint:CGPointMake(50, 100)];
    move.toValue = [NSValue valueWithCGPoint:CGPointMake(300, 100)];
    move.duration = 0.8;
    move.timingFunction = [CAMediaTimingFunction functionWithName:kCAMediaTimingFunctionEaseInEaseOut];
    
    // อัปเดตตำแหน่งจริงด้วย (ไม่งั้นจะ snap กลับ)
    self.myLayer.position = CGPointMake(300, 100);
    
    [self.myLayer addAnimation:move forKey:@"moveAnim"];
}

- (void)animateScale {
    CABasicAnimation *scale = [CABasicAnimation animationWithKeyPath:@"transform.scale"];
    scale.fromValue = @1.0;
    scale.toValue = @1.5;
    scale.duration = 0.3;
    scale.autoreverses = YES;
    
    [self.myLayer addAnimation:scale forKey:@"scaleAnim"];
}

- (void)animateRotation {
    CABasicAnimation *rotate = [CABasicAnimation animationWithKeyPath:@"transform.rotation.z"];
    rotate.fromValue = @0;
    rotate.toValue = @(M_PI * 2);
    rotate.duration = 2.0;
    rotate.repeatCount = HUGE_VALF;
    rotate.timingFunction = [CAMediaTimingFunction functionWithName:kCAMediaTimingFunctionLinear];
    
    [self.myLayer addAnimation:rotate forKey:@"rotateAnim"];
}
```

### 3.2 Animate Colors

```objc
- (void)animateBackgroundColor {
    CABasicAnimation *colorAnim = [CABasicAnimation animationWithKeyPath:@"backgroundColor"];
    colorAnim.fromValue = (__bridge id)[UIColor systemBlueColor].CGColor;
    colorAnim.toValue = (__bridge id)[UIColor systemPurpleColor].CGColor;
    colorAnim.duration = 1.5;
    colorAnim.autoreverses = YES;
    colorAnim.repeatCount = HUGE_VALF;
    
    [self.myLayer addAnimation:colorAnim forKey:@"colorAnim"];
}
```

### 3.3 Animate Bounds (เปลี่ยนขนาด)

```objc
- (void)animateBounds {
    CGRect fromBounds = CGRectMake(0, 0, 100, 100);
    CGRect toBounds = CGRectMake(0, 0, 200, 200);
    
    CABasicAnimation *boundsAnim = [CABasicAnimation animationWithKeyPath:@"bounds"];
    boundsAnim.fromValue = [NSValue valueWithCGRect:fromBounds];
    boundsAnim.toValue = [NSValue valueWithCGRect:toBounds];
    boundsAnim.duration = 0.5;
    
    self.myLayer.bounds = toBounds;
    [self.myLayer addAnimation:boundsAnim forKey:@"boundsAnim"];
}
```

### 3.4 Path Animation (เคลื่อนที่ตาม Path)

```objc
- (void)animateAlongPath {
    UIBezierPath *path = [UIBezierPath bezierPath];
    [path moveToPoint:CGPointMake(50, 300)];
    [path addCurveToPoint:CGPointMake(300, 300)
            controlPoint1:CGPointMake(100, 100)
            controlPoint2:CGPointMake(250, 500)];
    
    CAKeyframeAnimation *pathAnim = [CAKeyframeAnimation animationWithKeyPath:@"position"];
    pathAnim.path = path.CGPath;
    pathAnim.duration = 2.0;
    pathAnim.rotationMode = kCAAnimationRotateAuto; // หมุนตามทิศทาง path
    
    [self.myLayer addAnimation:pathAnim forKey:@"pathAnim"];
}
```

---

## 4. CAKeyframeAnimation

### 4.1 Keyframe Animation พื้นฐาน

```objc
- (void)bounceAnimation {
    // Animation กระเด้ง
    CAKeyframeAnimation *bounce = [CAKeyframeAnimation animationWithKeyPath:@"transform.translation.y"];
    bounce.values = @[@0, @(-60), @0, @(-30), @0, @(-10), @0];
    bounce.keyTimes = @[@0, @0.2, @0.4, @0.6, @0.7, @0.85, @1.0];
    bounce.duration = 1.2;
    bounce.timingFunctions = @[
        [CAMediaTimingFunction functionWithName:kCAMediaTimingFunctionEaseOut],
        [CAMediaTimingFunction functionWithName:kCAMediaTimingFunctionEaseIn],
        [CAMediaTimingFunction functionWithName:kCAMediaTimingFunctionEaseOut],
        [CAMediaTimingFunction functionWithName:kCAMediaTimingFunctionEaseIn],
        [CAMediaTimingFunction functionWithName:kCAMediaTimingFunctionEaseOut],
        [CAMediaTimingFunction functionWithName:kCAMediaTimingFunctionEaseIn]
    ];
    
    [self.myLayer addAnimation:bounce forKey:@"bounce"];
}
```

### 4.2 Shake Animation

```objc
- (void)shakeAnimation:(CALayer *)layer {
    CAKeyframeAnimation *shake = [CAKeyframeAnimation animationWithKeyPath:@"transform.translation.x"];
    shake.timingFunction = [CAMediaTimingFunction functionWithName:kCAMediaTimingFunctionLinear];
    shake.duration = 0.6;
    shake.values = @[@0, @(-12), @12, @(-10), @10, @(-8), @8, @(-5), @5, @0];
    
    [layer addAnimation:shake forKey:@"shake"];
}
```

### 4.3 Pulse Animation

```objc
- (void)pulseAnimation:(CALayer *)layer {
    CAKeyframeAnimation *pulse = [CAKeyframeAnimation animationWithKeyPath:@"transform.scale"];
    pulse.values = @[@1.0, @1.1, @0.95, @1.05, @1.0];
    pulse.keyTimes = @[@0, @0.25, @0.5, @0.75, @1.0];
    pulse.duration = 0.5;
    
    [layer addAnimation:pulse forKey:@"pulse"];
}
```

### 4.4 Color Cycle Animation

```objc
- (void)colorCycleAnimation:(CALayer *)layer {
    CAKeyframeAnimation *colorAnim = [CAKeyframeAnimation animationWithKeyPath:@"backgroundColor"];
    colorAnim.values = @[
        (__bridge id)[UIColor systemRedColor].CGColor,
        (__bridge id)[UIColor systemOrangeColor].CGColor,
        (__bridge id)[UIColor systemYellowColor].CGColor,
        (__bridge id)[UIColor systemGreenColor].CGColor,
        (__bridge id)[UIColor systemBlueColor].CGColor,
        (__bridge id)[UIColor systemPurpleColor].CGColor,
        (__bridge id)[UIColor systemRedColor].CGColor
    ];
    colorAnim.calculationMode = kCAAnimationPaced;
    colorAnim.duration = 3.0;
    colorAnim.repeatCount = HUGE_VALF;
    
    [layer addAnimation:colorAnim forKey:@"colorCycle"];
}
```

---

## 5. CAAnimationGroup

### 5.1 รวม Animations หลายอย่าง

```objc
- (void)combinedAnimation:(CALayer *)layer {
    // Animation 1: ขยาย
    CABasicAnimation *scaleAnim = [CABasicAnimation animationWithKeyPath:@"transform.scale"];
    scaleAnim.fromValue = @1.0;
    scaleAnim.toValue = @1.3;
    scaleAnim.duration = 0.3;
    scaleAnim.timingFunction = [CAMediaTimingFunction functionWithName:kCAMediaTimingFunctionEaseOut];
    
    // Animation 2: ย้าย
    CABasicAnimation *moveAnim = [CABasicAnimation animationWithKeyPath:@"position.y"];
    moveAnim.fromValue = @(layer.position.y);
    moveAnim.toValue = @(layer.position.y - 50);
    moveAnim.duration = 0.3;
    
    // Animation 3: เปลี่ยนสี
    CABasicAnimation *colorAnim = [CABasicAnimation animationWithKeyPath:@"backgroundColor"];
    colorAnim.fromValue = (__bridge id)[UIColor systemBlueColor].CGColor;
    colorAnim.toValue = (__bridge id)[UIColor systemGreenColor].CGColor;
    colorAnim.duration = 0.3;
    
    // รวมเป็น group
    CAAnimationGroup *group = [CAAnimationGroup animation];
    group.animations = @[scaleAnim, moveAnim, colorAnim];
    group.duration = 0.3;
    group.timingFunction = [CAMediaTimingFunction functionWithName:kCAMediaTimingFunctionEaseInEaseOut];
    
    [layer addAnimation:group forKey:@"combined"];
}
```

### 5.2 Sequential Animations ด้วย beginTime

```objc
- (void)sequentialAnimation:(CALayer *)layer {
    CABasicAnimation *step1 = [CABasicAnimation animationWithKeyPath:@"position.x"];
    step1.byValue = @100;
    step1.duration = 0.4;
    step1.beginTime = 0.0;
    
    CABasicAnimation *step2 = [CABasicAnimation animationWithKeyPath:@"position.y"];
    step2.byValue = @(-80);
    step2.duration = 0.3;
    step2.beginTime = 0.4; // เริ่มหลัง step1
    
    CABasicAnimation *step3 = [CABasicAnimation animationWithKeyPath:@"transform.scale"];
    step3.toValue = @0.5;
    step3.duration = 0.3;
    step3.beginTime = 0.7; // เริ่มหลัง step2
    
    CAAnimationGroup *sequence = [CAAnimationGroup animation];
    sequence.animations = @[step1, step2, step3];
    sequence.duration = 1.0;
    sequence.fillMode = kCAFillModeForwards;
    sequence.removedOnCompletion = NO;
    
    [layer addAnimation:sequence forKey:@"sequence"];
}
```

---

## 6. CATransition

### 6.1 Transition Animations

```objc
- (void)transitionFade:(UIView *)view {
    CATransition *transition = [CATransition animation];
    transition.type = kCATransitionFade;
    transition.duration = 0.5;
    transition.timingFunction = [CAMediaTimingFunction functionWithName:
                                  kCAMediaTimingFunctionEaseInEaseOut];
    
    [view.layer addAnimation:transition forKey:kCATransition];
    
    // เปลี่ยน content ของ view
    view.backgroundColor = [UIColor systemPurpleColor];
}

- (void)transitionSlide:(UIView *)view fromLeft:(BOOL)fromLeft {
    CATransition *transition = [CATransition animation];
    transition.type = kCATransitionPush;
    transition.subtype = fromLeft ? kCATransitionFromLeft : kCATransitionFromRight;
    transition.duration = 0.4;
    
    [view.layer addAnimation:transition forKey:kCATransition];
}

- (void)transitionReveal:(UIView *)view {
    CATransition *transition = [CATransition animation];
    transition.type = kCATransitionReveal;
    transition.subtype = kCATransitionFromBottom;
    transition.duration = 0.4;
    
    [view.layer addAnimation:transition forKey:kCATransition];
}
```

### 6.2 Custom Transition บน UIViewController

```objc
- (void)switchToViewController:(UIViewController *)newVC {
    CATransition *transition = [CATransition animation];
    transition.type = kCATransitionFade;
    transition.duration = 0.3;
    transition.timingFunction = [CAMediaTimingFunction functionWithName:
                                  kCAMediaTimingFunctionEaseInEaseOut];
    
    [self.view.window.layer addAnimation:transition forKey:kCATransition];
    
    self.view.window.rootViewController = newVC;
}
```

---

## 7. Timing Functions

### 7.1 Built-in Timing Functions

```objc
- (void)demonstrateTimingFunctions {
    NSArray *timingNames = @[
        kCAMediaTimingFunctionLinear,
        kCAMediaTimingFunctionEaseIn,
        kCAMediaTimingFunctionEaseOut,
        kCAMediaTimingFunctionEaseInEaseOut,
        kCAMediaTimingFunctionDefault
    ];
    
    for (NSInteger i = 0; i < timingNames.count; i++) {
        CALayer *layer = [CALayer layer];
        layer.frame = CGRectMake(0, 60 + i * 50, 30, 30);
        layer.backgroundColor = [UIColor systemBlueColor].CGColor;
        layer.cornerRadius = 15;
        [self.view.layer addSublayer:layer];
        
        CABasicAnimation *anim = [CABasicAnimation animationWithKeyPath:@"position.x"];
        anim.fromValue = @15;
        anim.toValue = @285;
        anim.duration = 1.5;
        anim.timingFunction = [CAMediaTimingFunction functionWithName:timingNames[i]];
        anim.repeatCount = HUGE_VALF;
        anim.autoreverses = YES;
        
        [layer addAnimation:anim forKey:@"move"];
    }
}
```

### 7.2 Custom Timing Function (Cubic Bezier)

```objc
- (CAMediaTimingFunction *)springTimingFunction {
    // Control points สำหรับ spring effect
    return [CAMediaTimingFunction functionWithControlPoints:0.5 :1.8 :0.5 :0.8];
}

- (void)animateWithSpringTiming:(CALayer *)layer {
    CABasicAnimation *anim = [CABasicAnimation animationWithKeyPath:@"transform.scale"];
    anim.fromValue = @0.0;
    anim.toValue = @1.0;
    anim.duration = 0.5;
    anim.timingFunction = [self springTimingFunction];
    
    [layer addAnimation:anim forKey:@"springScale"];
}
```

### 7.3 Spring Animation (iOS 9+)

```objc
- (void)springAnimation:(UIView *)view {
    // UIView spring animation
    [UIView animateWithDuration:0.6
                          delay:0
         usingSpringWithDamping:0.5
          initialSpringVelocity:0.5
                        options:UIViewAnimationOptionCurveEaseInOut
                     animations:^{
        view.transform = CGAffineTransformIdentity;
    } completion:nil];
}
```

---

## 8. Animation Delegates

### 8.1 CAAnimationDelegate Protocol

```objc
// ใช้งาน delegate
@interface MyViewController () <CAAnimationDelegate>
@end

@implementation MyViewController

- (void)startAnimation {
    CABasicAnimation *anim = [CABasicAnimation animationWithKeyPath:@"position"];
    anim.toValue = [NSValue valueWithCGPoint:CGPointMake(200, 400)];
    anim.duration = 1.0;
    anim.delegate = self;
    anim.setValue:@"movingAnimation" forKey:@"animationName"];
    
    [self.myLayer addAnimation:anim forKey:@"move"];
}

- (void)animationDidStart:(CAAnimation *)anim {
    NSString *name = [anim valueForKey:@"animationName"];
    NSLog(@"Animation เริ่ม: %@", name);
}

- (void)animationDidStop:(CAAnimation *)anim finished:(BOOL)flag {
    NSString *name = [anim valueForKey:@"animationName"];
    
    if (flag) {
        NSLog(@"Animation %@ เสร็จสมบูรณ์", name);
    } else {
        NSLog(@"Animation %@ ถูกยกเลิก", name);
    }
    
    // เริ่ม animation ถัดไป
    if ([name isEqualToString:@"movingAnimation"]) {
        [self startSecondAnimation];
    }
}

- (void)startSecondAnimation {
    CABasicAnimation *anim = [CABasicAnimation animationWithKeyPath:@"backgroundColor"];
    anim.toValue = (__bridge id)[UIColor systemGreenColor].CGColor;
    anim.duration = 0.5;
    anim.fillMode = kCAFillModeForwards;
    anim.removedOnCompletion = NO;
    
    [self.myLayer addAnimation:anim forKey:@"color"];
}

@end
```

---

## 9. Layer Properties ขั้นสูง

### 9.1 Mask Layer

```objc
- (void)applyCircularMask:(UIView *)view {
    CALayer *maskLayer = [CALayer layer];
    maskLayer.frame = view.bounds;
    maskLayer.contents = (__bridge id)[self circleImage:view.bounds.size].CGImage;
    maskLayer.contentsGravity = kCAGravityResizeAspect;
    
    view.layer.mask = maskLayer;
}

- (UIImage *)circleImage:(CGSize)size {
    UIGraphicsBeginImageContextWithOptions(size, NO, 0.0);
    CGContextRef ctx = UIGraphicsGetCurrentContext();
    
    CGContextSetFillColorWithColor(ctx, [UIColor blackColor].CGColor);
    CGContextAddEllipseInRect(ctx, CGRectMake(0, 0, size.width, size.height));
    CGContextFillPath(ctx);
    
    UIImage *img = UIGraphicsGetImageFromCurrentImageContext();
    UIGraphicsEndImageContext();
    return img;
}
```

### 9.2 CAGradientLayer

```objc
- (void)addGradientLayerToView:(UIView *)view {
    CAGradientLayer *gradient = [CAGradientLayer layer];
    gradient.frame = view.bounds;
    gradient.colors = @[
        (__bridge id)[UIColor systemBlueColor].CGColor,
        (__bridge id)[UIColor systemPurpleColor].CGColor
    ];
    gradient.startPoint = CGPointMake(0, 0);
    gradient.endPoint = CGPointMake(1, 1);
    
    [view.layer insertSublayer:gradient atIndex:0];
}

// Animate gradient colors
- (void)animateGradient:(CAGradientLayer *)gradientLayer {
    CABasicAnimation *colorAnim = [CABasicAnimation animationWithKeyPath:@"colors"];
    colorAnim.fromValue = @[
        (__bridge id)[UIColor systemBlueColor].CGColor,
        (__bridge id)[UIColor systemPurpleColor].CGColor
    ];
    colorAnim.toValue = @[
        (__bridge id)[UIColor systemOrangeColor].CGColor,
        (__bridge id)[UIColor systemPinkColor].CGColor
    ];
    colorAnim.duration = 2.0;
    colorAnim.autoreverses = YES;
    colorAnim.repeatCount = HUGE_VALF;
    
    [gradientLayer addAnimation:colorAnim forKey:@"gradientColors"];
}
```

### 9.3 CAShapeLayer

```objc
- (void)demonstrateShapeLayer {
    CAShapeLayer *shapeLayer = [CAShapeLayer layer];
    shapeLayer.frame = CGRectMake(50, 100, 250, 250);
    
    // สร้าง path
    UIBezierPath *path = [UIBezierPath bezierPath];
    [path moveToPoint:CGPointMake(125, 10)];
    [path addLineToPoint:CGPointMake(200, 240)];
    [path addLineToPoint:CGPointMake(10, 90)];
    [path addLineToPoint:CGPointMake(240, 90)];
    [path addLineToPoint:CGPointMake(50, 240)];
    [path closePath];
    
    shapeLayer.path = path.CGPath;
    shapeLayer.fillColor = [UIColor systemYellowColor].CGColor;
    shapeLayer.strokeColor = [UIColor systemOrangeColor].CGColor;
    shapeLayer.lineWidth = 3.0;
    shapeLayer.lineDashPattern = @[@10, @5];
    
    [self.view.layer addSublayer:shapeLayer];
}

// CAShapeLayer สำหรับ stroke animation
- (void)animateStrokeDrawing {
    UIBezierPath *path = [UIBezierPath bezierPathWithOvalInRect:
                          CGRectMake(50, 50, 200, 200)];
    
    CAShapeLayer *shape = [CAShapeLayer layer];
    shape.path = path.CGPath;
    shape.strokeColor = [UIColor systemBlueColor].CGColor;
    shape.fillColor = [UIColor clearColor].CGColor;
    shape.lineWidth = 4.0;
    shape.strokeEnd = 0;
    
    [self.view.layer addSublayer:shape];
    
    // Animate การวาดเส้น
    CABasicAnimation *drawAnim = [CABasicAnimation animationWithKeyPath:@"strokeEnd"];
    drawAnim.fromValue = @0;
    drawAnim.toValue = @1;
    drawAnim.duration = 2.0;
    drawAnim.timingFunction = [CAMediaTimingFunction functionWithName:
                                kCAMediaTimingFunctionEaseInEaseOut];
    
    [shape addAnimation:drawAnim forKey:@"strokeDraw"];
}
```

### 9.4 CATextLayer

```objc
- (void)addTextLayer {
    CATextLayer *textLayer = [CATextLayer layer];
    textLayer.frame = CGRectMake(20, 200, 300, 60);
    textLayer.string = @"Hello, Core Animation!";
    textLayer.font = (__bridge CFTypeRef)[UIFont boldSystemFontOfSize:20];
    textLayer.fontSize = 20;
    textLayer.foregroundColor = [UIColor systemBlueColor].CGColor;
    textLayer.backgroundColor = [UIColor systemGray6Color].CGColor;
    textLayer.alignmentMode = kCAAlignmentCenter;
    textLayer.contentsScale = [UIScreen mainScreen].scale;
    textLayer.cornerRadius = 8;
    textLayer.masksToBounds = YES;
    
    [self.view.layer addSublayer:textLayer];
    
    // Animate ข้อความ
    CABasicAnimation *textAnim = [CABasicAnimation animationWithKeyPath:@"string"];
    textAnim.fromValue = @"Hello!";
    textAnim.toValue = @"Core Animation!";
    textAnim.duration = 1.0;
    
    [textLayer addAnimation:textAnim forKey:@"text"];
}
```

---

## 10. 3D Transforms (CATransform3D)

### 10.1 พื้นฐาน CATransform3D

```objc
- (void)apply3DTransforms {
    CALayer *layer = self.cardLayer;
    
    // Perspective
    CATransform3D perspective = CATransform3DIdentity;
    perspective.m34 = -1.0 / 500.0; // กำหนด perspective
    self.view.layer.sublayerTransform = perspective;
    
    // Rotate X (เอียงหน้าหลัง)
    CATransform3D rotateX = CATransform3DMakeRotation(M_PI / 4, 1, 0, 0);
    layer.transform = rotateX;
    
    // Rotate Y (พลิกซ้ายขวา)
    CATransform3D rotateY = CATransform3DMakeRotation(M_PI / 4, 0, 1, 0);
    layer.transform = rotateY;
    
    // Rotate Z (หมุนในระนาบ 2D)
    CATransform3D rotateZ = CATransform3DMakeRotation(M_PI / 4, 0, 0, 1);
    layer.transform = rotateZ;
    
    // รวม transforms
    CATransform3D combined = CATransform3DIdentity;
    combined = CATransform3DRotate(combined, M_PI / 6, 0, 1, 0);
    combined = CATransform3DScale(combined, 1.2, 1.2, 1.0);
    layer.transform = combined;
}
```

### 10.2 Card Flip Animation

```objc
- (void)flipCardAnimation {
    CALayer *frontLayer = self.frontLayer;
    CALayer *backLayer = self.backLayer;
    
    // ตั้ง perspective
    CATransform3D perspective = CATransform3DIdentity;
    perspective.m34 = -1.0 / 500.0;
    
    // Front จาก 0 ไป 90 องศา
    CABasicAnimation *frontFlip = [CABasicAnimation animationWithKeyPath:@"transform"];
    frontFlip.fromValue = [NSValue valueWithCATransform3D:perspective];
    
    CATransform3D frontEnd = perspective;
    frontEnd = CATransform3DRotate(frontEnd, -M_PI_2, 0, 1, 0);
    frontFlip.toValue = [NSValue valueWithCATransform3D:frontEnd];
    frontFlip.duration = 0.4;
    frontFlip.timingFunction = [CAMediaTimingFunction functionWithName:kCAMediaTimingFunctionEaseIn];
    frontFlip.fillMode = kCAFillModeForwards;
    frontFlip.removedOnCompletion = NO;
    
    [frontLayer addAnimation:frontFlip forKey:@"frontFlip"];
    
    // Back จาก -90 ไป 0 องศา
    dispatch_after(dispatch_time(DISPATCH_TIME_NOW, (int64_t)(0.35 * NSEC_PER_SEC)),
                   dispatch_get_main_queue(), ^{
        
        backLayer.hidden = NO;
        frontLayer.hidden = YES;
        
        CABasicAnimation *backFlip = [CABasicAnimation animationWithKeyPath:@"transform"];
        
        CATransform3D backStart = perspective;
        backStart = CATransform3DRotate(backStart, M_PI_2, 0, 1, 0);
        backFlip.fromValue = [NSValue valueWithCATransform3D:backStart];
        backFlip.toValue = [NSValue valueWithCATransform3D:perspective];
        backFlip.duration = 0.4;
        backFlip.timingFunction = [CAMediaTimingFunction functionWithName:kCAMediaTimingFunctionEaseOut];
        
        [backLayer addAnimation:backFlip forKey:@"backFlip"];
    });
}
```

### 10.3 3D Carousel

```objc
- (void)setup3DCarousel {
    NSInteger itemCount = 6;
    CGFloat radius = 150.0;
    
    for (NSInteger i = 0; i < itemCount; i++) {
        CALayer *itemLayer = [CALayer layer];
        itemLayer.frame = CGRectMake(-60, -80, 120, 160);
        itemLayer.backgroundColor = [UIColor colorWithHue:(CGFloat)i / itemCount 
                                               saturation:0.8 brightness:0.9 alpha:1.0].CGColor;
        itemLayer.cornerRadius = 10;
        
        // วาง layer รอบวงกลม
        CGFloat angle = (CGFloat)i / itemCount * M_PI * 2;
        
        CATransform3D t = CATransform3DIdentity;
        t.m34 = -1.0 / 500.0;
        t = CATransform3DTranslate(t, radius * sin(angle), 0, radius * cos(angle));
        t = CATransform3DRotate(t, angle, 0, 1, 0);
        
        itemLayer.transform = t;
        [self.view.layer addSublayer:itemLayer];
    }
}
```

---

## 11. Particle Systems (CAEmitterLayer)

### 11.1 Snow Effect

```objc
- (void)addSnowEffect {
    CAEmitterLayer *emitter = [CAEmitterLayer layer];
    emitter.emitterPosition = CGPointMake(self.view.bounds.size.width / 2, -10);
    emitter.emitterSize = CGSizeMake(self.view.bounds.size.width, 1);
    emitter.emitterShape = kCAEmitterLayerLine;
    
    CAEmitterCell *snowflake = [CAEmitterCell emitterCell];
    snowflake.name = @"snowflake";
    snowflake.birthRate = 50;       // จำนวน particle ต่อวินาที
    snowflake.lifetime = 8.0;       // อายุ particle
    snowflake.velocity = 50;        // ความเร็ว
    snowflake.velocityRange = 30;   // ความแตกต่างความเร็ว
    snowflake.emissionLongitude = M_PI; // ทิศทาง (ลงล่าง)
    snowflake.emissionRange = M_PI / 6; // กระจาย
    snowflake.xAcceleration = 10;   // drift ซ้ายขวา
    snowflake.yAcceleration = 0;
    snowflake.scale = 0.03;
    snowflake.scaleRange = 0.02;
    snowflake.spin = 0.5;
    snowflake.spinRange = 1.0;
    snowflake.color = [UIColor whiteColor].CGColor;
    snowflake.alphaRange = 0.5;
    snowflake.alphaSpeed = -0.1;
    
    // สร้างรูป snowflake
    snowflake.contents = (__bridge id)[self createSnowflakeImage].CGImage;
    
    emitter.emitterCells = @[snowflake];
    [self.view.layer addSublayer:emitter];
}

- (UIImage *)createSnowflakeImage {
    UIGraphicsBeginImageContextWithOptions(CGSizeMake(20, 20), NO, 0.0);
    CGContextRef ctx = UIGraphicsGetCurrentContext();
    
    CGContextSetStrokeColorWithColor(ctx, [UIColor whiteColor].CGColor);
    CGContextSetLineWidth(ctx, 2.0);
    
    for (int i = 0; i < 6; i++) {
        CGFloat angle = i * M_PI / 3.0;
        CGContextMoveToPoint(ctx, 10, 10);
        CGContextAddLineToPoint(ctx, 10 + 8 * cos(angle), 10 + 8 * sin(angle));
    }
    CGContextStrokePath(ctx);
    
    UIImage *img = UIGraphicsGetImageFromCurrentImageContext();
    UIGraphicsEndImageContext();
    return img;
}
```

### 11.2 Fireworks Effect

```objc
- (void)launchFirework:(CGPoint)point {
    CAEmitterLayer *emitter = [CAEmitterLayer layer];
    emitter.emitterPosition = point;
    emitter.emitterShape = kCAEmitterLayerPoint;
    emitter.renderMode = kCAEmitterLayerAdditive;
    
    CAEmitterCell *spark = [CAEmitterCell emitterCell];
    spark.name = @"spark";
    spark.birthRate = 0;
    spark.lifetime = 1.5;
    spark.velocity = 200;
    spark.velocityRange = 100;
    spark.emissionLongitude = -M_PI_2;
    spark.emissionRange = M_PI * 2;
    spark.yAcceleration = 100;
    spark.scale = 0.1;
    spark.scaleRange = 0.08;
    spark.alphaSpeed = -0.7;
    spark.color = [UIColor systemOrangeColor].CGColor;
    spark.contents = (__bridge id)[UIImage imageNamed:@"particle"].CGImage;
    
    emitter.emitterCells = @[spark];
    [self.view.layer addSublayer:emitter];
    
    // เริ่ม burst
    spark.birthRate = 500;
    
    dispatch_after(dispatch_time(DISPATCH_TIME_NOW, (int64_t)(0.1 * NSEC_PER_SEC)),
                   dispatch_get_main_queue(), ^{
        spark.birthRate = 0;
        
        dispatch_after(dispatch_time(DISPATCH_TIME_NOW, (int64_t)(2.0 * NSEC_PER_SEC)),
                       dispatch_get_main_queue(), ^{
            [emitter removeFromSuperlayer];
        });
    });
}
```

### 11.3 Heart Particles

```objc
- (void)addHeartParticles:(UIView *)view {
    CAEmitterLayer *emitter = [CAEmitterLayer layer];
    emitter.frame = view.bounds;
    emitter.emitterPosition = CGPointMake(view.bounds.size.width / 2, 
                                           view.bounds.size.height);
    emitter.emitterSize = CGSizeMake(view.bounds.size.width, 0);
    emitter.emitterShape = kCAEmitterLayerLine;
    
    CAEmitterCell *heart = [CAEmitterCell emitterCell];
    heart.contents = (__bridge id)[self heartImage].CGImage;
    heart.birthRate = 5;
    heart.lifetime = 5.0;
    heart.velocity = 100;
    heart.velocityRange = 50;
    heart.emissionLongitude = -M_PI_2;
    heart.emissionRange = M_PI / 8;
    heart.scale = 0.05;
    heart.scaleRange = 0.03;
    heart.spin = 0.5;
    heart.spinRange = 1.0;
    heart.alphaSpeed = -0.2;
    
    NSArray *colors = @[
        (__bridge id)[UIColor systemRedColor].CGColor,
        (__bridge id)[UIColor systemPinkColor].CGColor,
        (__bridge id)[UIColor systemOrangeColor].CGColor
    ];
    
    NSMutableArray *cells = [NSMutableArray array];
    for (UIColor *color in colors) {
        CAEmitterCell *coloredHeart = [heart copy];
        coloredHeart.color = (__bridge CGColorRef)color;
        [cells addObject:coloredHeart];
    }
    
    emitter.emitterCells = cells;
    [view.layer addSublayer:emitter];
}

- (UIImage *)heartImage {
    UIGraphicsBeginImageContextWithOptions(CGSizeMake(30, 30), NO, 0.0);
    CGContextRef ctx = UIGraphicsGetCurrentContext();
    
    CGContextSetFillColorWithColor(ctx, [UIColor systemRedColor].CGColor);
    
    UIBezierPath *path = [UIBezierPath bezierPath];
    [path moveToPoint:CGPointMake(15, 25)];
    [path addCurveToPoint:CGPointMake(28, 8)
            controlPoint1:CGPointMake(25, 20)
            controlPoint2:CGPointMake(30, 12)];
    [path addCurveToPoint:CGPointMake(15, 10)
            controlPoint1:CGPointMake(26, 2)
            controlPoint2:CGPointMake(18, 5)];
    [path addCurveToPoint:CGPointMake(2, 8)
            controlPoint1:CGPointMake(12, 5)
            controlPoint2:CGPointMake(4, 2)];
    [path addCurveToPoint:CGPointMake(15, 25)
            controlPoint1:CGPointMake(0, 12)
            controlPoint2:CGPointMake(5, 20)];
    [path closePath];
    [path fill];
    
    UIImage *img = UIGraphicsGetImageFromCurrentImageContext();
    UIGraphicsEndImageContext();
    return img;
}
```

---

## 12. Custom Animations

### 12.1 Loading Spinner

```objc
@interface LoadingSpinner : UIView
- (void)startAnimating;
- (void)stopAnimating;
@end

@implementation LoadingSpinner {
    CAShapeLayer *_trackLayer;
    CAShapeLayer *_spinnerLayer;
    BOOL _isAnimating;
}

- (instancetype)initWithFrame:(CGRect)frame {
    self = [super initWithFrame:frame];
    if (self) {
        [self setupLayers];
    }
    return self;
}

- (void)setupLayers {
    CGPoint center = CGPointMake(self.bounds.size.width / 2, self.bounds.size.height / 2);
    CGFloat radius = MIN(self.bounds.size.width, self.bounds.size.height) / 2 - 5;
    
    UIBezierPath *circlePath = [UIBezierPath bezierPathWithArcCenter:center
                                                               radius:radius
                                                           startAngle:-M_PI_2
                                                             endAngle:M_PI + M_PI_2
                                                            clockwise:YES];
    
    // Track layer
    _trackLayer = [CAShapeLayer layer];
    _trackLayer.path = circlePath.CGPath;
    _trackLayer.fillColor = [UIColor clearColor].CGColor;
    _trackLayer.strokeColor = [UIColor systemGray5Color].CGColor;
    _trackLayer.lineWidth = 4.0;
    [self.layer addSublayer:_trackLayer];
    
    // Spinner layer
    _spinnerLayer = [CAShapeLayer layer];
    _spinnerLayer.path = circlePath.CGPath;
    _spinnerLayer.fillColor = [UIColor clearColor].CGColor;
    _spinnerLayer.strokeColor = [UIColor systemBlueColor].CGColor;
    _spinnerLayer.lineWidth = 4.0;
    _spinnerLayer.strokeStart = 0;
    _spinnerLayer.strokeEnd = 0.3;
    _spinnerLayer.lineCap = kCALineCapRound;
    [self.layer addSublayer:_spinnerLayer];
}

- (void)startAnimating {
    if (_isAnimating) return;
    _isAnimating = YES;
    
    CABasicAnimation *rotation = [CABasicAnimation animationWithKeyPath:@"transform.rotation.z"];
    rotation.fromValue = @0;
    rotation.toValue = @(M_PI * 2);
    rotation.duration = 1.0;
    rotation.repeatCount = HUGE_VALF;
    rotation.timingFunction = [CAMediaTimingFunction functionWithName:kCAMediaTimingFunctionLinear];
    
    [_spinnerLayer addAnimation:rotation forKey:@"spin"];
}

- (void)stopAnimating {
    if (!_isAnimating) return;
    _isAnimating = NO;
    [_spinnerLayer removeAnimationForKey:@"spin"];
}

@end
```

### 12.2 Typing Cursor Effect

```objc
- (void)addBlinkingCursor:(UILabel *)label {
    CALayer *cursor = [CALayer layer];
    cursor.frame = CGRectMake(label.frame.origin.x + label.intrinsicContentSize.width + 2,
                               label.frame.origin.y + 4,
                               2,
                               label.font.pointSize);
    cursor.backgroundColor = [UIColor labelColor].CGColor;
    
    [self.view.layer addSublayer:cursor];
    
    CABasicAnimation *blink = [CABasicAnimation animationWithKeyPath:@"opacity"];
    blink.fromValue = @1;
    blink.toValue = @0;
    blink.duration = 0.5;
    blink.repeatCount = HUGE_VALF;
    blink.autoreverses = YES;
    
    [cursor addAnimation:blink forKey:@"blink"];
}
```

### 12.3 Ripple Effect

```objc
- (void)addRippleAt:(CGPoint)point inView:(UIView *)view {
    CALayer *ripple = [CALayer layer];
    ripple.frame = CGRectMake(point.x - 2, point.y - 2, 4, 4);
    ripple.backgroundColor = [[UIColor systemBlueColor] colorWithAlphaComponent:0.4].CGColor;
    ripple.cornerRadius = 2;
    
    [view.layer addSublayer:ripple];
    
    CAAnimationGroup *rippleAnim = [CAAnimationGroup animation];
    
    CABasicAnimation *scaleAnim = [CABasicAnimation animationWithKeyPath:@"transform.scale"];
    scaleAnim.fromValue = @1;
    scaleAnim.toValue = @30;
    
    CABasicAnimation *fadeAnim = [CABasicAnimation animationWithKeyPath:@"opacity"];
    fadeAnim.fromValue = @0.5;
    fadeAnim.toValue = @0;
    
    rippleAnim.animations = @[scaleAnim, fadeAnim];
    rippleAnim.duration = 0.6;
    rippleAnim.timingFunction = [CAMediaTimingFunction functionWithName:kCAMediaTimingFunctionEaseOut];
    
    [ripple addAnimation:rippleAnim forKey:@"ripple"];
    
    dispatch_after(dispatch_time(DISPATCH_TIME_NOW, (int64_t)(0.6 * NSEC_PER_SEC)),
                   dispatch_get_main_queue(), ^{
        [ripple removeFromSuperlayer];
    });
}
```

### 12.4 Progress Bar Animation

```objc
@interface AnimatedProgressBar : UIView
@property (nonatomic, assign) CGFloat progress;
@end

@implementation AnimatedProgressBar {
    CALayer *_trackLayer;
    CALayer *_progressLayer;
    CAGradientLayer *_gradientLayer;
}

- (instancetype)initWithFrame:(CGRect)frame {
    self = [super initWithFrame:frame];
    if (self) {
        [self setupLayers];
    }
    return self;
}

- (void)setupLayers {
    // Track
    _trackLayer = [CALayer layer];
    _trackLayer.frame = self.bounds;
    _trackLayer.backgroundColor = [UIColor systemGray5Color].CGColor;
    _trackLayer.cornerRadius = self.bounds.size.height / 2;
    [self.layer addSublayer:_trackLayer];
    
    // Gradient progress
    _gradientLayer = [CAGradientLayer layer];
    _gradientLayer.frame = CGRectMake(0, 0, 0, self.bounds.size.height);
    _gradientLayer.colors = @[
        (__bridge id)[UIColor systemBlueColor].CGColor,
        (__bridge id)[UIColor systemCyanColor].CGColor
    ];
    _gradientLayer.startPoint = CGPointMake(0, 0.5);
    _gradientLayer.endPoint = CGPointMake(1, 0.5);
    _gradientLayer.cornerRadius = self.bounds.size.height / 2;
    [self.layer addSublayer:_gradientLayer];
}

- (void)setProgress:(CGFloat)progress animated:(BOOL)animated {
    _progress = MAX(0, MIN(1, progress));
    CGFloat targetWidth = self.bounds.size.width * _progress;
    
    if (animated) {
        CABasicAnimation *anim = [CABasicAnimation animationWithKeyPath:@"bounds"];
        anim.toValue = [NSValue valueWithCGRect:CGRectMake(0, 0, targetWidth, self.bounds.size.height)];
        anim.duration = 0.5;
        anim.timingFunction = [CAMediaTimingFunction functionWithName:kCAMediaTimingFunctionEaseOut];
        anim.fillMode = kCAFillModeForwards;
        anim.removedOnCompletion = NO;
        
        [_gradientLayer addAnimation:anim forKey:@"progress"];
    } else {
        _gradientLayer.bounds = CGRectMake(0, 0, targetWidth, self.bounds.size.height);
    }
}

@end
```

---

## 13. Performance Tips

### 13.1 Layer การตั้งค่าเพื่อประสิทธิภาพ

```objc
- (void)optimizeLayerPerformance:(CALayer *)layer {
    // Rasterize layer เพื่อ cache บน GPU
    layer.shouldRasterize = YES;
    layer.rasterizationScale = [UIScreen mainScreen].scale;
    
    // หลีกเลี่ยง off-screen rendering
    // ใช้ corner radius แต่ไม่ masksToBounds บน layer ที่ไม่มี content
    
    // ใช้ opaque content เมื่อทำได้
    layer.opaque = YES;
    layer.backgroundColor = [UIColor whiteColor].CGColor;
    
    // ระบุว่า content ไม่เปลี่ยนแปลง
    layer.drawsAsynchronously = NO;
}
```

### 13.2 ตรวจสอบ Animation FPS

```objc
- (void)monitorFPS {
    CADisplayLink *displayLink = [CADisplayLink displayLinkWithTarget:self 
                                                             selector:@selector(onFrame:)];
    [displayLink addToRunLoop:[NSRunLoop mainRunLoop] forMode:NSRunLoopCommonModes];
    self.displayLink = displayLink;
    self.lastTimestamp = 0;
    self.frameCount = 0;
}

- (void)onFrame:(CADisplayLink *)link {
    if (self.lastTimestamp == 0) {
        self.lastTimestamp = link.timestamp;
        return;
    }
    
    self.frameCount++;
    CFTimeInterval elapsed = link.timestamp - self.lastTimestamp;
    
    if (elapsed >= 1.0) {
        NSLog(@"FPS: %.1f", self.frameCount / elapsed);
        self.frameCount = 0;
        self.lastTimestamp = link.timestamp;
    }
}
```

---

## แบบฝึกหัด (Practice Exercises)

### แบบฝึกหัดที่ 1: Menu Animation
```objc
// สร้าง animated hamburger menu
@interface HamburgerMenuView : UIView
- (void)toggleMenu;
@property (nonatomic, assign) BOOL isOpen;
@end

@implementation HamburgerMenuView {
    CALayer *_topBar;
    CALayer *_midBar;
    CALayer *_bottomBar;
}

- (instancetype)initWithFrame:(CGRect)frame {
    self = [super initWithFrame:frame];
    if (self) {
        [self setupBars];
    }
    return self;
}

- (void)setupBars {
    CGFloat barWidth = 30;
    CGFloat barHeight = 3;
    CGFloat spacing = 8;
    CGFloat startX = (self.bounds.size.width - barWidth) / 2;
    CGFloat midY = self.bounds.size.height / 2;
    
    _topBar = [self createBarLayerAt:CGPointMake(startX, midY - spacing)
                               width:barWidth height:barHeight];
    _midBar = [self createBarLayerAt:CGPointMake(startX, midY - barHeight/2)
                               width:barWidth height:barHeight];
    _bottomBar = [self createBarLayerAt:CGPointMake(startX, midY + spacing - barHeight)
                                  width:barWidth height:barHeight];
}

- (CALayer *)createBarLayerAt:(CGPoint)origin width:(CGFloat)w height:(CGFloat)h {
    CALayer *bar = [CALayer layer];
    bar.frame = CGRectMake(origin.x, origin.y, w, h);
    bar.backgroundColor = [UIColor labelColor].CGColor;
    bar.cornerRadius = h / 2;
    [self.layer addSublayer:bar];
    return bar;
}

- (void)toggleMenu {
    _isOpen = !_isOpen;
    
    [CATransaction begin];
    [CATransaction setAnimationDuration:0.3];
    
    if (_isOpen) {
        // Top bar: หมุน 45 องศาที่กลางไม้
        _topBar.transform = CATransform3DConcat(
            CATransform3DMakeTranslation(0, 8, 0),
            CATransform3DMakeRotation(M_PI / 4, 0, 0, 1)
        );
        // Mid bar: fade out
        _midBar.opacity = 0;
        // Bottom bar: หมุน -45 องศา
        _bottomBar.transform = CATransform3DConcat(
            CATransform3DMakeTranslation(0, -8, 0),
            CATransform3DMakeRotation(-M_PI / 4, 0, 0, 1)
        );
    } else {
        _topBar.transform = CATransform3DIdentity;
        _midBar.opacity = 1;
        _bottomBar.transform = CATransform3DIdentity;
    }
    
    [CATransaction commit];
}

@end
```

### แบบฝึกหัดที่ 2: Confetti Effect
```objc
- (void)launchConfetti {
    NSArray *colors = @[
        [UIColor systemRedColor],
        [UIColor systemBlueColor],
        [UIColor systemYellowColor],
        [UIColor systemGreenColor],
        [UIColor systemOrangeColor],
        [UIColor systemPurpleColor]
    ];
    
    CAEmitterLayer *confettiEmitter = [CAEmitterLayer layer];
    confettiEmitter.frame = CGRectMake(0, -20, self.view.bounds.size.width, 20);
    confettiEmitter.emitterPosition = CGPointMake(self.view.bounds.size.width / 2, -10);
    confettiEmitter.emitterSize = CGSizeMake(self.view.bounds.size.width, 0);
    confettiEmitter.emitterShape = kCAEmitterLayerLine;
    confettiEmitter.renderMode = kCAEmitterLayerUnordered;
    
    NSMutableArray *cells = [NSMutableArray array];
    
    for (UIColor *color in colors) {
        CAEmitterCell *cell = [CAEmitterCell emitterCell];
        cell.contents = (__bridge id)[self confettiImageWithColor:color].CGImage;
        cell.birthRate = 5;
        cell.lifetime = 7.0;
        cell.velocity = 200;
        cell.velocityRange = 80;
        cell.emissionLongitude = M_PI;
        cell.emissionRange = M_PI / 5;
        cell.xAcceleration = 30;
        cell.yAcceleration = 60;
        cell.spin = 4;
        cell.spinRange = 6;
        cell.scale = 0.07;
        cell.scaleRange = 0.03;
        cell.alphaSpeed = -0.15;
        cell.color = color.CGColor;
        
        [cells addObject:cell];
    }
    
    confettiEmitter.emitterCells = cells;
    [self.view.layer addSublayer:confettiEmitter];
    
    dispatch_after(dispatch_time(DISPATCH_TIME_NOW, (int64_t)(3 * NSEC_PER_SEC)),
                   dispatch_get_main_queue(), ^{
        confettiEmitter.birthRate = 0;
        
        dispatch_after(dispatch_time(DISPATCH_TIME_NOW, (int64_t)(8 * NSEC_PER_SEC)),
                       dispatch_get_main_queue(), ^{
            [confettiEmitter removeFromSuperlayer];
        });
    });
}

- (UIImage *)confettiImageWithColor:(UIColor *)color {
    UIGraphicsBeginImageContextWithOptions(CGSizeMake(8, 8), NO, 0.0);
    [color setFill];
    [[UIBezierPath bezierPathWithRect:CGRectMake(0, 0, 8, 8)] fill];
    UIImage *img = UIGraphicsGetImageFromCurrentImageContext();
    UIGraphicsEndImageContext();
    return img;
}
```

### แบบฝึกหัดที่ 3: Animated Page Indicator
```objc
@interface AnimatedPageControl : UIView
@property (nonatomic, assign) NSInteger numberOfPages;
@property (nonatomic, assign) NSInteger currentPage;
@end

@implementation AnimatedPageControl {
    NSMutableArray<CALayer *> *_dots;
}

- (void)setNumberOfPages:(NSInteger)count {
    _numberOfPages = count;
    [self setupDots];
}

- (void)setupDots {
    // ลบ dots เดิม
    for (CALayer *dot in _dots) {
        [dot removeFromSuperlayer];
    }
    _dots = [NSMutableArray array];
    
    CGFloat dotSize = 8.0;
    CGFloat spacing = 16.0;
    CGFloat totalWidth = _numberOfPages * dotSize + (_numberOfPages - 1) * spacing;
    CGFloat startX = (self.bounds.size.width - totalWidth) / 2;
    CGFloat y = (self.bounds.size.height - dotSize) / 2;
    
    for (NSInteger i = 0; i < _numberOfPages; i++) {
        CALayer *dot = [CALayer layer];
        dot.frame = CGRectMake(startX + i * (dotSize + spacing), y, dotSize, dotSize);
        dot.backgroundColor = [UIColor systemGrayColor].CGColor;
        dot.cornerRadius = dotSize / 2;
        [self.layer addSublayer:dot];
        [_dots addObject:dot];
    }
    
    [self updateDotColors];
}

- (void)setCurrentPage:(NSInteger)page {
    NSInteger old = _currentPage;
    _currentPage = page;
    
    if (_dots.count == 0) return;
    
    [CATransaction begin];
    [CATransaction setAnimationDuration:0.3];
    
    // ขยาย dot ปัจจุบัน
    for (NSInteger i = 0; i < _dots.count; i++) {
        if (i == _currentPage) {
            _dots[i].backgroundColor = [UIColor systemBlueColor].CGColor;
            _dots[i].transform = CATransform3DMakeScale(1.5, 1.5, 1.0);
        } else {
            _dots[i].backgroundColor = [UIColor systemGrayColor].CGColor;
            _dots[i].transform = CATransform3DIdentity;
        }
    }
    
    [CATransaction commit];
}

- (void)updateDotColors {
    for (NSInteger i = 0; i < _dots.count; i++) {
        _dots[i].backgroundColor = (i == _currentPage) 
            ? [UIColor systemBlueColor].CGColor 
            : [UIColor systemGrayColor].CGColor;
    }
}

@end
```

---

## สรุป

ในบทนี้เราได้เรียนรู้ Core Animation อย่างครอบคลุม:

1. **CALayer** - พื้นฐานของทุก animation ใน iOS
2. **CAAnimation** - Base class ของ animations
3. **CABasicAnimation** - Animation จากค่า A ถึง B
4. **CAKeyframeAnimation** - Animation หลายจุด
5. **CAAnimationGroup** - รวม animations หลายอย่าง
6. **CATransition** - Transition ระหว่าง states
7. **Timing Functions** - ควบคุมความเร็ว animation
8. **Animation Delegates** - รับรู้เมื่อ animation จบ
9. **Layer Properties** - CAGradientLayer, CAShapeLayer, CATextLayer
10. **CATransform3D** - Perspective และ 3D transforms
11. **CAEmitterLayer** - Particle systems
12. **Custom Animations** - Loading spinner, ripple, progress bar

ในบทต่อไปจะเรียนรู้เรื่อง Audio และ Video ด้วย AVFoundation
