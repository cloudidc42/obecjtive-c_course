# ส่วนที่ 58: Core Graphics ใน Objective-C

## บทนำ

Core Graphics (หรือ Quartz 2D) เป็น framework สำหรับการวาดกราฟิก 2D ระดับต่ำของ Apple ที่มีประสิทธิภาพสูง ช่วยให้คุณสร้าง custom UI components และกราฟิกที่ซับซ้อนได้อย่างละเอียด

---

## 1. CGContext - หัวใจของ Core Graphics

### 1.1 ทำความเข้าใจ CGContext

`CGContextRef` คือ "canvas" ที่ทุกการวาดจะเกิดขึ้น ระบบมีหลายประเภท:

```objc
// 1. UIView's drawRect: context (ได้อัตโนมัติ)
- (void)drawRect:(CGRect)rect {
    CGContextRef ctx = UIGraphicsGetCurrentContext();
    // วาดที่นี่
}

// 2. Bitmap context (สร้างเอง)
CGColorSpaceRef colorSpace = CGColorSpaceCreateDeviceRGB();
CGContextRef bitmapCtx = CGBitmapContextCreate(
    NULL,          // data (NULL = จัดการเอง)
    200, 200,      // width, height
    8,             // bits per component
    0,             // bytes per row (0 = คำนวณอัตโนมัติ)
    colorSpace,
    kCGImageAlphaPremultipliedLast
);
CGColorSpaceRelease(colorSpace);

// 3. UIGraphicsImageRenderer context (iOS 10+)
UIGraphicsImageRenderer *renderer = [[UIGraphicsImageRenderer alloc] 
                                      initWithSize:CGSizeMake(200, 200)];
```

### 1.2 State Machine ของ Context

```objc
- (void)drawRect:(CGRect)rect {
    CGContextRef ctx = UIGraphicsGetCurrentContext();
    
    // บันทึก state ปัจจุบัน
    CGContextSaveGState(ctx);
    
    // เปลี่ยน state
    CGContextSetFillColorWithColor(ctx, [UIColor redColor].CGColor);
    CGContextSetLineWidth(ctx, 3.0);
    
    // วาด
    CGContextFillRect(ctx, CGRectMake(10, 10, 100, 100));
    
    // คืนค่า state เดิม
    CGContextRestoreGState(ctx);
    
    // State กลับมาเป็นเดิมแล้ว
    CGContextFillRect(ctx, CGRectMake(120, 10, 100, 100)); // สีที่เปลี่ยนไปไม่ส่งผล
}
```

---

## 2. Drawing in drawRect:

### 2.1 Custom View พื้นฐาน

```objc
// CustomView.h
@interface CustomView : UIView
@property (nonatomic, strong) UIColor *fillColor;
@property (nonatomic, strong) UIColor *strokeColor;
@property (nonatomic, assign) CGFloat lineWidth;
@end

// CustomView.m
@implementation CustomView

- (instancetype)initWithFrame:(CGRect)frame {
    self = [super initWithFrame:frame];
    if (self) {
        _fillColor = [UIColor systemBlueColor];
        _strokeColor = [UIColor systemBlueColor];
        _lineWidth = 2.0;
        self.backgroundColor = [UIColor clearColor];
    }
    return self;
}

- (void)drawRect:(CGRect)rect {
    CGContextRef ctx = UIGraphicsGetCurrentContext();
    
    // วาดพื้นหลัง
    CGContextSetFillColorWithColor(ctx, [UIColor systemGray6Color].CGColor);
    CGContextFillRect(ctx, rect);
    
    // วาดรูปทรงตัวอย่าง
    [self drawRoundedRectInContext:ctx];
}

- (void)drawRoundedRectInContext:(CGContextRef)ctx {
    CGRect drawRect = CGRectInset(self.bounds, 10, 10);
    CGFloat cornerRadius = 12.0;
    
    UIBezierPath *path = [UIBezierPath bezierPathWithRoundedRect:drawRect 
                                                    cornerRadius:cornerRadius];
    
    // Fill
    CGContextSetFillColorWithColor(ctx, self.fillColor.CGColor);
    CGContextAddPath(ctx, path.CGPath);
    CGContextFillPath(ctx);
    
    // Stroke
    CGContextSetStrokeColorWithColor(ctx, self.strokeColor.CGColor);
    CGContextSetLineWidth(ctx, self.lineWidth);
    CGContextAddPath(ctx, path.CGPath);
    CGContextStrokePath(ctx);
}

// เรียก setNeedsDisplay เมื่อ properties เปลี่ยน
- (void)setFillColor:(UIColor *)fillColor {
    _fillColor = fillColor;
    [self setNeedsDisplay];
}

@end
```

---

## 3. Lines, Shapes, Paths

### 3.1 เส้นตรง (Lines)

```objc
- (void)drawLines:(CGContextRef)ctx {
    // เส้นเดี่ยว
    CGContextSetStrokeColorWithColor(ctx, [UIColor blackColor].CGColor);
    CGContextSetLineWidth(ctx, 2.0);
    CGContextMoveToPoint(ctx, 10, 50);
    CGContextAddLineToPoint(ctx, 290, 50);
    CGContextStrokePath(ctx);
    
    // เส้นประ (dashed)
    CGFloat dashPattern[] = {10, 5, 3, 5};
    CGContextSetLineDash(ctx, 0, dashPattern, 4);
    CGContextMoveToPoint(ctx, 10, 80);
    CGContextAddLineToPoint(ctx, 290, 80);
    CGContextStrokePath(ctx);
    
    // รีเซ็ต dash pattern
    CGContextSetLineDash(ctx, 0, NULL, 0);
    
    // เส้นแบบ thick
    CGContextSetLineWidth(ctx, 8.0);
    CGContextSetLineCap(ctx, kCGLineCapRound);
    CGContextMoveToPoint(ctx, 10, 110);
    CGContextAddLineToPoint(ctx, 290, 110);
    CGContextStrokePath(ctx);
}
```

### 3.2 รูปทรงพื้นฐาน

```objc
- (void)drawBasicShapes:(CGContextRef)ctx {
    // สี่เหลี่ยม (stroke only)
    CGContextSetStrokeColorWithColor(ctx, [UIColor systemRedColor].CGColor);
    CGContextSetLineWidth(ctx, 2.0);
    CGContextStrokeRect(ctx, CGRectMake(10, 10, 80, 60));
    
    // สี่เหลี่ยม (fill)
    CGContextSetFillColorWithColor(ctx, [UIColor systemBlueColor].CGColor);
    CGContextFillRect(ctx, CGRectMake(110, 10, 80, 60));
    
    // สี่เหลี่ยม (fill + stroke)
    CGContextSetFillColorWithColor(ctx, [UIColor systemGreenColor].CGColor);
    CGContextSetStrokeColorWithColor(ctx, [UIColor systemGreenColor].CGColor);
    CGContextAddRect(ctx, CGRectMake(210, 10, 80, 60));
    CGContextDrawPath(ctx, kCGPathFillStroke);
    
    // วงรี
    CGContextSetFillColorWithColor(ctx, [UIColor systemOrangeColor].CGColor);
    CGContextFillEllipseInRect(ctx, CGRectMake(10, 90, 100, 60));
    
    // วงกลม
    CGContextSetFillColorWithColor(ctx, [UIColor systemPurpleColor].CGColor);
    CGContextFillEllipseInRect(ctx, CGRectMake(130, 90, 60, 60));
}
```

### 3.3 Paths ซับซ้อน

```objc
- (void)drawComplexPath:(CGContextRef)ctx {
    // รูปดาว 5 แฉก
    CGPoint center = CGPointMake(150, 200);
    CGFloat outerRadius = 80.0;
    CGFloat innerRadius = 35.0;
    
    CGContextBeginPath(ctx);
    
    for (NSInteger i = 0; i < 10; i++) {
        CGFloat radius = (i % 2 == 0) ? outerRadius : innerRadius;
        CGFloat angle = (i * M_PI / 5.0) - M_PI_2;
        
        CGFloat x = center.x + radius * cos(angle);
        CGFloat y = center.y + radius * sin(angle);
        
        if (i == 0) {
            CGContextMoveToPoint(ctx, x, y);
        } else {
            CGContextAddLineToPoint(ctx, x, y);
        }
    }
    
    CGContextClosePath(ctx);
    
    // Fill ด้วยสีทอง
    CGContextSetFillColorWithColor(ctx, [UIColor systemYellowColor].CGColor);
    CGContextSetStrokeColorWithColor(ctx, [UIColor systemOrangeColor].CGColor);
    CGContextSetLineWidth(ctx, 2.0);
    CGContextDrawPath(ctx, kCGPathFillStroke);
    
    // รูปหัวใจ
    [self drawHeartInContext:ctx atPoint:CGPointMake(150, 350) size:100];
}

- (void)drawHeartInContext:(CGContextRef)ctx atPoint:(CGPoint)point size:(CGFloat)size {
    CGFloat scale = size / 100.0;
    
    CGContextSaveGState(ctx);
    CGContextTranslateCTM(ctx, point.x, point.y);
    CGContextScaleCTM(ctx, scale, scale);
    
    CGContextBeginPath(ctx);
    CGContextMoveToPoint(ctx, 0, -20);
    
    // ครึ่งซ้าย
    CGContextAddCurveToPoint(ctx, -55, -80, -100, 10, 0, 60);
    
    // ครึ่งขวา
    CGContextAddCurveToPoint(ctx, 100, 10, 55, -80, 0, -20);
    
    CGContextSetFillColorWithColor(ctx, [UIColor systemRedColor].CGColor);
    CGContextFillPath(ctx);
    
    CGContextRestoreGState(ctx);
}
```

### 3.4 Bezier Curves

```objc
- (void)drawBezierCurves:(CGContextRef)ctx {
    CGContextSetStrokeColorWithColor(ctx, [UIColor systemBlueColor].CGColor);
    CGContextSetLineWidth(ctx, 3.0);
    
    // Quadratic curve (1 control point)
    CGContextMoveToPoint(ctx, 20, 200);
    CGContextAddQuadCurveToPoint(ctx, 160, 50, 300, 200);  // control, end
    CGContextStrokePath(ctx);
    
    // Cubic curve (2 control points)
    CGContextMoveToPoint(ctx, 20, 300);
    CGContextAddCurveToPoint(ctx, 80, 150, 220, 150, 300, 300);  // cp1, cp2, end
    CGContextStrokePath(ctx);
    
    // แสดง control points
    CGContextSetFillColorWithColor(ctx, [UIColor systemRedColor].CGColor);
    CGContextFillEllipseInRect(ctx, CGRectMake(75, 45, 10, 10));
    CGContextFillEllipseInRect(ctx, CGRectMake(75, 145, 10, 10));
    CGContextFillEllipseInRect(ctx, CGRectMake(215, 145, 10, 10));
}
```

---

## 4. Colors and Fills

### 4.1 สีต่างๆ

```objc
- (void)demonstrateColors:(CGContextRef)ctx {
    // สีพื้นฐาน
    CGContextSetFillColorWithColor(ctx, [UIColor redColor].CGColor);
    CGContextFillRect(ctx, CGRectMake(10, 10, 50, 50));
    
    // RGB สี
    CGContextSetRGBFillColor(ctx, 0.0, 0.5, 1.0, 1.0);
    CGContextFillRect(ctx, CGRectMake(70, 10, 50, 50));
    
    // สีกึ่งโปร่งใส
    CGContextSetFillColorWithColor(ctx, 
        [[UIColor systemBlueColor] colorWithAlphaComponent:0.5].CGColor);
    CGContextFillRect(ctx, CGRectMake(130, 10, 50, 50));
    
    // สี UIColor สมัยใหม่ (iOS 13+)
    if (@available(iOS 13.0, *)) {
        CGContextSetFillColorWithColor(ctx, [UIColor systemBackgroundColor].CGColor);
    }
    
    // สีแบบ CMYK
    CGColorSpaceRef cmykSpace = CGColorSpaceCreateDeviceCMYK();
    CGFloat cmykComponents[] = {0.0, 0.5, 1.0, 0.0, 1.0}; // C, M, Y, K, A
    CGColorRef cmykColor = CGColorCreate(cmykSpace, cmykComponents);
    CGContextSetFillColorWithColor(ctx, cmykColor);
    CGColorRelease(cmykColor);
    CGColorSpaceRelease(cmykSpace);
    CGContextFillRect(ctx, CGRectMake(190, 10, 50, 50));
}
```

### 4.2 Pattern Fill

```objc
// Callback สำหรับ pattern
static void drawPattern(void *info, CGContextRef ctx) {
    CGContextSetFillColorWithColor(ctx, [UIColor systemBlueColor].CGColor);
    CGContextFillRect(ctx, CGRectMake(2, 2, 6, 6));
    
    CGContextSetFillColorWithColor(ctx, [UIColor systemCyanColor].CGColor);
    CGContextFillRect(ctx, CGRectMake(12, 12, 6, 6));
}

- (void)drawPatternFill:(CGContextRef)ctx {
    // กำหนด pattern
    CGPatternCallbacks callbacks = {0, drawPattern, NULL};
    CGRect patternCell = CGRectMake(0, 0, 20, 20);
    
    CGPatternRef pattern = CGPatternCreate(
        NULL,
        patternCell,
        CGAffineTransformIdentity,
        20, 20,                    // x, y step
        kCGPatternTilingConstantSpacing,
        true,                      // colored pattern
        &callbacks
    );
    
    CGColorSpaceRef patternSpace = CGColorSpaceCreatePattern(NULL);
    CGContextSetFillColorSpace(ctx, patternSpace);
    
    CGFloat alpha = 1.0;
    CGContextSetFillPattern(ctx, pattern, &alpha);
    CGContextFillRect(ctx, CGRectMake(0, 0, 300, 200));
    
    CGPatternRelease(pattern);
    CGColorSpaceRelease(patternSpace);
}
```

---

## 5. Text Drawing

### 5.1 วาดข้อความด้วย UIKit

```objc
- (void)drawText:(CGContextRef)ctx {
    // วิธีง่ายที่สุด - ใช้ NSString category
    NSDictionary *attrs = @{
        NSFontAttributeName: [UIFont systemFontOfSize:24 weight:UIFontWeightBold],
        NSForegroundColorAttributeName: [UIColor systemBlueColor],
        NSBackgroundColorAttributeName: [UIColor clearColor]
    };
    
    [@"สวัสดี Core Graphics!" drawAtPoint:CGPointMake(20, 50) withAttributes:attrs];
    
    // วาดในพื้นที่จำกัด
    NSString *longText = @"นี่คือข้อความยาวที่ต้องการพื้นที่หลายบรรทัด โดยจะ wrap อัตโนมัติ";
    
    NSMutableParagraphStyle *paraStyle = [[NSMutableParagraphStyle alloc] init];
    paraStyle.alignment = NSTextAlignmentCenter;
    paraStyle.lineSpacing = 4.0;
    
    NSDictionary *multilineAttrs = @{
        NSFontAttributeName: [UIFont systemFontOfSize:16],
        NSForegroundColorAttributeName: [UIColor darkGrayColor],
        NSParagraphStyleAttributeName: paraStyle
    };
    
    CGRect textRect = CGRectMake(20, 100, 280, 100);
    [longText drawInRect:textRect withAttributes:multilineAttrs];
}
```

### 5.2 วาดข้อความด้วย Core Text

```objc
#import <CoreText/CoreText.h>

- (void)drawCoreText:(CGContextRef)ctx {
    // กลับแกน Y (Core Text ใช้ coordinate system ต่างจาก UIKit)
    CGContextSaveGState(ctx);
    CGContextTranslateCTM(ctx, 0, self.bounds.size.height);
    CGContextScaleCTM(ctx, 1.0, -1.0);
    
    // สร้าง attributed string
    NSMutableAttributedString *attrStr = [[NSMutableAttributedString alloc] 
                                           initWithString:@"Core Text Drawing"];
    
    CTFontRef font = CTFontCreateWithName(CFSTR("Helvetica-Bold"), 24, NULL);
    [attrStr addAttribute:(NSString *)kCTFontAttributeName 
                    value:(__bridge id)font 
                    range:NSMakeRange(0, attrStr.length)];
    
    CGColorRef color = [UIColor systemPurpleColor].CGColor;
    [attrStr addAttribute:(NSString *)kCTForegroundColorAttributeName 
                    value:(__bridge id)color 
                    range:NSMakeRange(0, attrStr.length)];
    
    // วาด
    CTLineRef line = CTLineCreateWithAttributedString((__bridge CFAttributedStringRef)attrStr);
    CGContextSetTextPosition(ctx, 20, 60);
    CTLineDraw(line, ctx);
    
    CFRelease(line);
    CFRelease(font);
    
    CGContextRestoreGState(ctx);
}
```

### 5.3 Text บน Path

```objc
- (void)drawTextOnCurve:(CGContextRef)ctx {
    CGContextSaveGState(ctx);
    CGContextTranslateCTM(ctx, 0, self.bounds.size.height);
    CGContextScaleCTM(ctx, 1.0, -1.0);
    
    NSString *text = @"Text Following a Curved Path!";
    
    CTFontRef font = CTFontCreateWithName(CFSTR("Helvetica"), 16, NULL);
    NSDictionary *attrs = @{
        (NSString *)kCTFontAttributeName: (__bridge id)font,
        (NSString *)kCTForegroundColorAttributeName: (__bridge id)[UIColor systemIndigoColor].CGColor
    };
    
    NSAttributedString *attrStr = [[NSAttributedString alloc] initWithString:text 
                                                                  attributes:attrs];
    CTLineRef line = CTLineCreateWithAttributedString((__bridge CFAttributedStringRef)attrStr);
    
    // วาดตัวอักษรทีละตัวบน arc
    CGFloat radius = 120;
    CGPoint center = CGPointMake(160, 200);
    CGFloat charCount = text.length;
    CGFloat totalAngle = M_PI;
    
    for (NSInteger i = 0; i < charCount; i++) {
        CGContextSaveGState(ctx);
        
        NSRange range = NSMakeRange(i, 1);
        CTLineRef charLine = CTLineCreateWithAttributedString(
            (__bridge CFAttributedStringRef)[attrStr attributedSubstringFromRange:range]);
        
        CGFloat charAngle = M_PI + (i / charCount) * totalAngle;
        CGFloat x = center.x + radius * cos(charAngle);
        CGFloat y = center.y + radius * sin(charAngle);
        
        CGContextTranslateCTM(ctx, x, y);
        CGContextRotateCTM(ctx, charAngle + M_PI_2);
        CGContextSetTextPosition(ctx, 0, 0);
        CTLineDraw(charLine, ctx);
        
        CFRelease(charLine);
        CGContextRestoreGState(ctx);
    }
    
    CFRelease(line);
    CFRelease(font);
    
    CGContextRestoreGState(ctx);
}
```

---

## 6. Image Drawing

### 6.1 วาดรูปภาพด้วย Core Graphics

```objc
- (void)drawImages:(CGContextRef)ctx {
    UIImage *image = [UIImage imageNamed:@"photo"];
    
    // วาดที่ตำแหน่งและขนาดที่กำหนด
    CGContextDrawImage(ctx, CGRectMake(10, 10, 150, 150), image.CGImage);
    
    // หมายเหตุ: CGContextDrawImage วาดแบบ upside-down
    // ต้องกลับ transform ก่อน
    CGContextSaveGState(ctx);
    CGContextTranslateCTM(ctx, 170, 10 + 150);
    CGContextScaleCTM(ctx, 1.0, -1.0);
    CGContextDrawImage(ctx, CGRectMake(0, 0, 150, 150), image.CGImage);
    CGContextRestoreGState(ctx);
}

// วิธีที่ถูกต้องและง่ายกว่า - ใช้ UIImage drawInRect
- (void)drawImageCorrectly {
    UIGraphicsBeginImageContextWithOptions(CGSizeMake(200, 200), NO, 0.0);
    
    UIImage *image = [UIImage imageNamed:@"photo"];
    [image drawInRect:CGRectMake(0, 0, 200, 200)];
    
    UIImage *result = UIGraphicsGetImageFromCurrentImageContext();
    UIGraphicsEndImageContext();
    
    self.imageView.image = result;
}
```

### 6.2 Tiling Image

```objc
- (void)drawTiledImage:(CGContextRef)ctx {
    UIImage *tileImage = [UIImage imageNamed:@"tile_pattern"];
    CGImageRef tileRef = tileImage.CGImage;
    
    // กำหนด phase และ size
    CGContextDrawTiledImage(ctx, 
                            CGRectMake(0, 0, tileImage.size.width, tileImage.size.height), 
                            tileRef);
}
```

---

## 7. Gradients

### 7.1 Linear Gradient

```objc
- (void)drawLinearGradient:(CGContextRef)ctx inRect:(CGRect)rect {
    CGColorSpaceRef colorSpace = CGColorSpaceCreateDeviceRGB();
    
    // กำหนดสี
    NSArray *colors = @[
        (__bridge id)[UIColor systemBlueColor].CGColor,
        (__bridge id)[UIColor systemPurpleColor].CGColor,
        (__bridge id)[UIColor systemPinkColor].CGColor
    ];
    
    CGFloat locations[] = {0.0, 0.5, 1.0};
    
    CGGradientRef gradient = CGGradientCreateWithColors(
        colorSpace,
        (__bridge CFArrayRef)colors,
        locations
    );
    
    // วาด gradient แนวตั้ง
    CGPoint startPoint = CGPointMake(CGRectGetMidX(rect), CGRectGetMinY(rect));
    CGPoint endPoint = CGPointMake(CGRectGetMidX(rect), CGRectGetMaxY(rect));
    
    CGContextDrawLinearGradient(ctx, gradient, startPoint, endPoint, 0);
    
    CGGradientRelease(gradient);
    CGColorSpaceRelease(colorSpace);
}
```

### 7.2 Radial Gradient

```objc
- (void)drawRadialGradient:(CGContextRef)ctx center:(CGPoint)center radius:(CGFloat)radius {
    CGColorSpaceRef colorSpace = CGColorSpaceCreateDeviceRGB();
    
    CGFloat colors[] = {
        1.0, 1.0, 0.0, 1.0,  // เหลือง
        1.0, 0.5, 0.0, 1.0,  // ส้ม
        0.8, 0.0, 0.0, 1.0   // แดงเข้ม
    };
    
    CGFloat locations[] = {0.0, 0.5, 1.0};
    
    CGGradientRef gradient = CGGradientCreateWithColorComponents(
        colorSpace, colors, locations, 3);
    
    CGContextDrawRadialGradient(
        ctx, gradient,
        center, 0,         // start center, start radius
        center, radius,    // end center, end radius
        0
    );
    
    CGGradientRelease(gradient);
    CGColorSpaceRelease(colorSpace);
}
```

### 7.3 Gradient ภายใน Shape

```objc
- (void)drawGradientButton:(CGContextRef)ctx inRect:(CGRect)rect {
    CGContextSaveGState(ctx);
    
    // สร้าง clipping path เป็นปุ่มโค้ง
    UIBezierPath *buttonPath = [UIBezierPath bezierPathWithRoundedRect:rect 
                                                          cornerRadius:12];
    CGContextAddPath(ctx, buttonPath.CGPath);
    CGContextClip(ctx);
    
    // วาด gradient ภายใน
    [self drawLinearGradient:ctx inRect:rect];
    
    // วาด highlight บนสุด
    CGColorSpaceRef colorSpace = CGColorSpaceCreateDeviceRGB();
    NSArray *highlightColors = @[
        (__bridge id)[[UIColor whiteColor] colorWithAlphaComponent:0.3].CGColor,
        (__bridge id)[[UIColor whiteColor] colorWithAlphaComponent:0.0].CGColor
    ];
    
    CGGradientRef highlightGradient = CGGradientCreateWithColors(
        colorSpace, (__bridge CFArrayRef)highlightColors, NULL);
    
    CGContextDrawLinearGradient(
        ctx, highlightGradient,
        CGPointMake(CGRectGetMidX(rect), CGRectGetMinY(rect)),
        CGPointMake(CGRectGetMidX(rect), CGRectGetMidY(rect)),
        0
    );
    
    CGGradientRelease(highlightGradient);
    CGColorSpaceRelease(colorSpace);
    
    CGContextRestoreGState(ctx);
    
    // วาด border
    CGContextSetStrokeColorWithColor(ctx, [[UIColor blackColor] colorWithAlphaComponent:0.2].CGColor);
    CGContextSetLineWidth(ctx, 1.0);
    CGContextAddPath(ctx, buttonPath.CGPath);
    CGContextStrokePath(ctx);
    
    // วาดข้อความ
    NSDictionary *textAttrs = @{
        NSFontAttributeName: [UIFont boldSystemFontOfSize:18],
        NSForegroundColorAttributeName: [UIColor whiteColor]
    };
    NSString *title = @"คลิก!";
    CGSize titleSize = [title sizeWithAttributes:textAttrs];
    CGPoint titlePoint = CGPointMake(
        CGRectGetMidX(rect) - titleSize.width / 2,
        CGRectGetMidY(rect) - titleSize.height / 2
    );
    [title drawAtPoint:titlePoint withAttributes:textAttrs];
}
```

---

## 8. Clipping

### 8.1 พื้นฐาน Clipping

```objc
- (void)demonstrateClipping:(CGContextRef)ctx {
    CGContextSaveGState(ctx);
    
    // สร้าง clipping region เป็นรูปดาว
    CGFloat cx = 150, cy = 150;
    CGContextBeginPath(ctx);
    
    for (NSInteger i = 0; i < 10; i++) {
        CGFloat radius = (i % 2 == 0) ? 100.0 : 45.0;
        CGFloat angle = (i * M_PI / 5.0) - M_PI_2;
        CGFloat x = cx + radius * cos(angle);
        CGFloat y = cy + radius * sin(angle);
        
        if (i == 0) {
            CGContextMoveToPoint(ctx, x, y);
        } else {
            CGContextAddLineToPoint(ctx, x, y);
        }
    }
    CGContextClosePath(ctx);
    CGContextClip(ctx);
    
    // วาดรูปภาพในพื้นที่ clip
    UIImage *image = [UIImage imageNamed:@"landscape"];
    [image drawInRect:CGRectMake(cx - 110, cy - 110, 220, 220)];
    
    CGContextRestoreGState(ctx);
    
    // วาดกรอบดาว
    CGContextSetStrokeColorWithColor(ctx, [UIColor systemGoldColor].CGColor);
    // ต้องวาด path อีกครั้ง เพราะ state ถูก restore แล้ว
    CGContextSetLineWidth(ctx, 2.0);
}
```

### 8.2 Even-Odd Rule Clipping

```objc
- (void)demonstrateEvenOddClipping:(CGContextRef)ctx {
    CGContextSaveGState(ctx);
    
    // วงกลมใหญ่
    CGContextAddEllipseInRect(ctx, CGRectMake(50, 50, 200, 200));
    // วงกลมเล็กตรงกลาง (จะกลายเป็นโปร่งใส)
    CGContextAddEllipseInRect(ctx, CGRectMake(100, 100, 100, 100));
    
    // Even-odd rule: พื้นที่ที่ path ซ้อนกันจำนวนคู่จะไม่ถูก clip
    CGContextEOClip(ctx);
    
    // วาดสีแดงในพื้นที่ clip (จะเป็นวงแหวน)
    CGContextSetFillColorWithColor(ctx, [UIColor systemRedColor].CGColor);
    CGContextFillRect(ctx, CGRectMake(0, 0, 300, 300));
    
    CGContextRestoreGState(ctx);
}
```

---

## 9. Transforms

### 9.1 Affine Transforms

```objc
- (void)demonstrateTransforms:(CGContextRef)ctx {
    // บันทึก state เดิม
    CGContextSaveGState(ctx);
    
    // 1. Translation (เลื่อนตำแหน่ง)
    CGContextTranslateCTM(ctx, 100, 100);
    CGContextSetFillColorWithColor(ctx, [UIColor systemBlueColor].CGColor);
    CGContextFillRect(ctx, CGRectMake(-25, -25, 50, 50));
    
    // 2. Rotation
    CGContextRotateCTM(ctx, M_PI / 4.0);  // 45 องศา
    CGContextSetFillColorWithColor(ctx, [UIColor systemRedColor].CGColor);
    CGContextFillRect(ctx, CGRectMake(50, -10, 50, 20));
    
    // 3. Scale
    CGContextScaleCTM(ctx, 2.0, 0.5);
    CGContextSetFillColorWithColor(ctx, [UIColor systemGreenColor].CGColor);
    CGContextFillRect(ctx, CGRectMake(60, 10, 30, 40));
    
    CGContextRestoreGState(ctx);
}
```

### 9.2 CGAffineTransform

```objc
- (void)applyCustomTransform:(CGContextRef)ctx {
    CGContextSaveGState(ctx);
    
    // สร้าง transform ที่รวม translate + rotate
    CGAffineTransform t = CGAffineTransformIdentity;
    t = CGAffineTransformTranslate(t, 150, 200);
    t = CGAffineTransformRotate(t, M_PI / 6.0);  // 30 องศา
    t = CGAffineTransformScale(t, 1.5, 1.5);
    
    CGContextConcatCTM(ctx, t);
    
    // วาดหลังจาก transform
    CGContextSetFillColorWithColor(ctx, [UIColor systemPurpleColor].CGColor);
    CGContextFillRect(ctx, CGRectMake(-30, -30, 60, 60));
    
    CGContextRestoreGState(ctx);
}
```

---

## 10. Bitmap Context

### 10.1 สร้างและใช้งาน Bitmap Context

```objc
- (UIImage *)createComplexBitmapImage {
    CGSize size = CGSizeMake(400, 400);
    UIGraphicsBeginImageContextWithOptions(size, YES, 0.0);
    CGContextRef ctx = UIGraphicsGetCurrentContext();
    
    // วาดพื้นหลัง
    CGContextSetFillColorWithColor(ctx, [UIColor blackColor].CGColor);
    CGContextFillRect(ctx, CGRectMake(0, 0, size.width, size.height));
    
    // วาด grid
    CGContextSetStrokeColorWithColor(ctx, [[UIColor whiteColor] 
                                           colorWithAlphaComponent:0.1].CGColor);
    CGContextSetLineWidth(ctx, 0.5);
    
    for (int x = 0; x < 400; x += 20) {
        CGContextMoveToPoint(ctx, x, 0);
        CGContextAddLineToPoint(ctx, x, 400);
    }
    for (int y = 0; y < 400; y += 20) {
        CGContextMoveToPoint(ctx, 0, y);
        CGContextAddLineToPoint(ctx, 400, y);
    }
    CGContextStrokePath(ctx);
    
    // วาดกราฟ sine wave
    CGContextSetStrokeColorWithColor(ctx, [UIColor systemGreenColor].CGColor);
    CGContextSetLineWidth(ctx, 2.0);
    
    BOOL firstPoint = YES;
    for (CGFloat x = 0; x < 400; x += 1) {
        CGFloat radians = (x / 400.0) * M_PI * 4.0;
        CGFloat y = 200 + 80 * sin(radians);
        
        if (firstPoint) {
            CGContextMoveToPoint(ctx, x, y);
            firstPoint = NO;
        } else {
            CGContextAddLineToPoint(ctx, x, y);
        }
    }
    CGContextStrokePath(ctx);
    
    UIImage *result = UIGraphicsGetImageFromCurrentImageContext();
    UIGraphicsEndImageContext();
    
    return result;
}
```

### 10.2 Raw Pixel Access

```objc
- (UIImage *)createNoiseImage:(CGSize)size {
    NSUInteger width = (NSUInteger)size.width;
    NSUInteger height = (NSUInteger)size.height;
    
    unsigned char *data = (unsigned char *)malloc(width * height * 4);
    
    for (NSUInteger y = 0; y < height; y++) {
        for (NSUInteger x = 0; x < width; x++) {
            NSUInteger idx = (y * width + x) * 4;
            unsigned char value = arc4random() % 256;
            data[idx] = value;     // R
            data[idx + 1] = value; // G
            data[idx + 2] = value; // B
            data[idx + 3] = 255;   // A
        }
    }
    
    CGColorSpaceRef colorSpace = CGColorSpaceCreateDeviceRGB();
    CGContextRef ctx = CGBitmapContextCreate(
        data, width, height, 8, width * 4, colorSpace,
        kCGImageAlphaPremultipliedLast);
    
    CGImageRef imageRef = CGBitmapContextCreateImage(ctx);
    UIImage *image = [UIImage imageWithCGImage:imageRef];
    
    CGImageRelease(imageRef);
    CGContextRelease(ctx);
    CGColorSpaceRelease(colorSpace);
    free(data);
    
    return image;
}
```

---

## 11. PDF Drawing

### 11.1 สร้างไฟล์ PDF

```objc
- (void)createPDFDocument {
    NSString *filename = @"document.pdf";
    NSString *docPath = [[NSSearchPathForDirectoriesInDomains(
        NSDocumentDirectory, NSUserDomainMask, YES) firstObject] 
        stringByAppendingPathComponent:filename];
    
    UIGraphicsBeginPDFContextToFile(docPath, CGRectMake(0, 0, 595, 842), nil);
    
    // หน้าแรก
    UIGraphicsBeginPDFPageWithInfo(CGRectMake(0, 0, 595, 842), nil);
    
    [self drawPDFPage1];
    
    // หน้าสอง
    UIGraphicsBeginPDFPageWithInfo(CGRectMake(0, 0, 595, 842), nil);
    [self drawPDFPage2];
    
    UIGraphicsEndPDFContext();
    
    NSLog(@"บันทึก PDF ที่: %@", docPath);
}

- (void)drawPDFPage1 {
    CGContextRef ctx = UIGraphicsGetCurrentContext();
    
    // หัวเรื่อง
    NSDictionary *titleAttrs = @{
        NSFontAttributeName: [UIFont boldSystemFontOfSize:36],
        NSForegroundColorAttributeName: [UIColor blackColor]
    };
    [@"รายงานการขาย" drawAtPoint:CGPointMake(50, 50) withAttributes:titleAttrs];
    
    // เส้น separator
    CGContextSetStrokeColorWithColor(ctx, [UIColor darkGrayColor].CGColor);
    CGContextSetLineWidth(ctx, 0.5);
    CGContextMoveToPoint(ctx, 50, 100);
    CGContextAddLineToPoint(ctx, 545, 100);
    CGContextStrokePath(ctx);
    
    // ตาราง
    NSArray *headers = @[@"เดือน", @"ยอดขาย", @"เป้าหมาย", @"ผลต่าง"];
    NSArray *data = @[
        @[@"มกราคม", @"150,000", @"120,000", @"+30,000"],
        @[@"กุมภาพันธ์", @"135,000", @"120,000", @"+15,000"],
        @[@"มีนาคม", @"160,000", @"140,000", @"+20,000"],
    ];
    
    NSDictionary *headerAttrs = @{
        NSFontAttributeName: [UIFont boldSystemFontOfSize:14],
        NSForegroundColorAttributeName: [UIColor whiteColor]
    };
    
    NSDictionary *cellAttrs = @{
        NSFontAttributeName: [UIFont systemFontOfSize:12],
        NSForegroundColorAttributeName: [UIColor blackColor]
    };
    
    CGFloat cellWidth = 120, cellHeight = 30;
    CGFloat startX = 50, startY = 120;
    
    // วาด header
    CGContextSetFillColorWithColor(ctx, [UIColor systemBlueColor].CGColor);
    CGContextFillRect(ctx, CGRectMake(startX, startY, cellWidth * 4, cellHeight));
    
    for (NSInteger i = 0; i < headers.count; i++) {
        [headers[i] drawAtPoint:CGPointMake(startX + i * cellWidth + 5, startY + 8) 
                 withAttributes:headerAttrs];
    }
    
    // วาด rows
    for (NSInteger row = 0; row < data.count; row++) {
        CGFloat rowY = startY + (row + 1) * cellHeight;
        
        if (row % 2 == 0) {
            CGContextSetFillColorWithColor(ctx, [UIColor systemGray6Color].CGColor);
            CGContextFillRect(ctx, CGRectMake(startX, rowY, cellWidth * 4, cellHeight));
        }
        
        for (NSInteger col = 0; col < [data[row] count]; col++) {
            [data[row][col] drawAtPoint:CGPointMake(startX + col * cellWidth + 5, rowY + 8) 
                         withAttributes:cellAttrs];
        }
    }
}

- (void)drawPDFPage2 {
    NSDictionary *attrs = @{
        NSFontAttributeName: [UIFont systemFontOfSize:14],
        NSForegroundColorAttributeName: [UIColor blackColor]
    };
    [@"หน้าที่ 2 - กราฟและสรุป" drawAtPoint:CGPointMake(50, 50) withAttributes:attrs];
}
```

---

## 12. Custom Drawing Components

### 12.1 Progress Ring Component

```objc
@interface CircularProgressView : UIView
@property (nonatomic, assign) CGFloat progress;     // 0.0 - 1.0
@property (nonatomic, strong) UIColor *trackColor;
@property (nonatomic, strong) UIColor *progressColor;
@property (nonatomic, assign) CGFloat lineWidth;
@end

@implementation CircularProgressView

- (instancetype)initWithFrame:(CGRect)frame {
    self = [super initWithFrame:frame];
    if (self) {
        _progress = 0.0;
        _trackColor = [UIColor systemGray5Color];
        _progressColor = [UIColor systemBlueColor];
        _lineWidth = 10.0;
        self.backgroundColor = [UIColor clearColor];
    }
    return self;
}

- (void)setProgress:(CGFloat)progress {
    _progress = MAX(0.0, MIN(1.0, progress));
    [self setNeedsDisplay];
}

- (void)drawRect:(CGRect)rect {
    CGContextRef ctx = UIGraphicsGetCurrentContext();
    
    CGPoint center = CGPointMake(CGRectGetMidX(rect), CGRectGetMidY(rect));
    CGFloat radius = (MIN(rect.size.width, rect.size.height) - self.lineWidth) / 2.0;
    
    // วาด track (วงกลมพื้นหลัง)
    CGContextSetStrokeColorWithColor(ctx, self.trackColor.CGColor);
    CGContextSetLineWidth(ctx, self.lineWidth);
    CGContextSetLineCap(ctx, kCGLineCapRound);
    CGContextAddArc(ctx, center.x, center.y, radius, 0, M_PI * 2, 0);
    CGContextStrokePath(ctx);
    
    // วาด progress arc
    if (self.progress > 0) {
        CGFloat startAngle = -M_PI_2;  // เริ่มจากด้านบน
        CGFloat endAngle = startAngle + (self.progress * M_PI * 2);
        
        CGContextSetStrokeColorWithColor(ctx, self.progressColor.CGColor);
        CGContextSetLineWidth(ctx, self.lineWidth);
        CGContextSetLineCap(ctx, kCGLineCapRound);
        CGContextAddArc(ctx, center.x, center.y, radius, startAngle, endAngle, 0);
        CGContextStrokePath(ctx);
    }
    
    // วาดเปอร์เซ็นต์ตรงกลาง
    NSString *percentText = [NSString stringWithFormat:@"%.0f%%", self.progress * 100];
    NSDictionary *textAttrs = @{
        NSFontAttributeName: [UIFont boldSystemFontOfSize:24],
        NSForegroundColorAttributeName: [UIColor labelColor]
    };
    
    CGSize textSize = [percentText sizeWithAttributes:textAttrs];
    CGPoint textPoint = CGPointMake(
        center.x - textSize.width / 2,
        center.y - textSize.height / 2
    );
    [percentText drawAtPoint:textPoint withAttributes:textAttrs];
}

@end
```

### 12.2 Bar Chart Component

```objc
@interface BarChartView : UIView
@property (nonatomic, strong) NSArray<NSNumber *> *values;
@property (nonatomic, strong) NSArray<NSString *> *labels;
@property (nonatomic, strong) UIColor *barColor;
@end

@implementation BarChartView

- (void)drawRect:(CGRect)rect {
    if (!self.values || self.values.count == 0) return;
    
    CGContextRef ctx = UIGraphicsGetCurrentContext();
    
    CGFloat maxValue = [[self.values valueForKeyPath:@"@max.floatValue"] floatValue];
    CGFloat padding = 20;
    CGFloat bottomPadding = 40;
    CGFloat chartWidth = rect.size.width - 2 * padding;
    CGFloat chartHeight = rect.size.height - padding - bottomPadding;
    
    NSInteger barCount = self.values.count;
    CGFloat totalSpacing = chartWidth * 0.3;
    CGFloat barWidth = (chartWidth - totalSpacing) / barCount;
    CGFloat spacing = totalSpacing / (barCount + 1);
    
    // วาดแกน
    CGContextSetStrokeColorWithColor(ctx, [UIColor systemGrayColor].CGColor);
    CGContextSetLineWidth(ctx, 1.0);
    
    // แกน Y
    CGContextMoveToPoint(ctx, padding, padding);
    CGContextAddLineToPoint(ctx, padding, rect.size.height - bottomPadding);
    CGContextStrokePath(ctx);
    
    // แกน X
    CGContextMoveToPoint(ctx, padding, rect.size.height - bottomPadding);
    CGContextAddLineToPoint(ctx, rect.size.width - padding, rect.size.height - bottomPadding);
    CGContextStrokePath(ctx);
    
    // วาด bars
    for (NSInteger i = 0; i < barCount; i++) {
        CGFloat value = [self.values[i] floatValue];
        CGFloat barHeight = (value / maxValue) * chartHeight;
        CGFloat x = padding + spacing + i * (barWidth + spacing);
        CGFloat y = rect.size.height - bottomPadding - barHeight;
        
        CGRect barRect = CGRectMake(x, y, barWidth, barHeight);
        
        // Gradient bar
        CGContextSaveGState(ctx);
        UIBezierPath *barPath = [UIBezierPath bezierPathWithRoundedRect:barRect 
                                    byRoundingCorners:UIRectCornerTopLeft | UIRectCornerTopRight
                                          cornerRadii:CGSizeMake(4, 4)];
        CGContextAddPath(ctx, barPath.CGPath);
        CGContextClip(ctx);
        
        // วาด gradient
        CGColorSpaceRef colorSpace = CGColorSpaceCreateDeviceRGB();
        NSArray *gradColors = @[
            (__bridge id)(self.barColor ?: [UIColor systemBlueColor]).CGColor,
            (__bridge id)[[self.barColor ?: [UIColor systemBlueColor]] 
                          colorWithAlphaComponent:0.6].CGColor
        ];
        
        CGGradientRef gradient = CGGradientCreateWithColors(
            colorSpace, (__bridge CFArrayRef)gradColors, NULL);
        CGContextDrawLinearGradient(ctx, gradient,
                                    CGPointMake(x, y),
                                    CGPointMake(x, y + barHeight),
                                    0);
        CGGradientRelease(gradient);
        CGColorSpaceRelease(colorSpace);
        
        CGContextRestoreGState(ctx);
        
        // แสดง label
        if (i < self.labels.count) {
            NSDictionary *labelAttrs = @{
                NSFontAttributeName: [UIFont systemFontOfSize:10],
                NSForegroundColorAttributeName: [UIColor systemGrayColor]
            };
            
            CGSize labelSize = [self.labels[i] sizeWithAttributes:labelAttrs];
            CGPoint labelPoint = CGPointMake(
                x + barWidth / 2 - labelSize.width / 2,
                rect.size.height - bottomPadding + 5
            );
            [self.labels[i] drawAtPoint:labelPoint withAttributes:labelAttrs];
        }
        
        // แสดงค่า
        NSString *valueText = [NSString stringWithFormat:@"%.0f", value];
        NSDictionary *valueAttrs = @{
            NSFontAttributeName: [UIFont boldSystemFontOfSize:10],
            NSForegroundColorAttributeName: [UIColor whiteColor]
        };
        CGSize valSize = [valueText sizeWithAttributes:valueAttrs];
        CGPoint valPoint = CGPointMake(
            x + barWidth / 2 - valSize.width / 2,
            y + 4
        );
        [valueText drawAtPoint:valPoint withAttributes:valueAttrs];
    }
}

@end
```

### 12.3 Custom Gauge View

```objc
@interface GaugeView : UIView
@property (nonatomic, assign) CGFloat value;  // 0.0 - 1.0
@property (nonatomic, strong) NSString *label;
@end

@implementation GaugeView

- (void)drawRect:(CGRect)rect {
    CGContextRef ctx = UIGraphicsGetCurrentContext();
    CGPoint center = CGPointMake(CGRectGetMidX(rect), CGRectGetMaxY(rect) - 20);
    CGFloat radius = MIN(rect.size.width / 2, rect.size.height) - 20;
    
    CGFloat startAngle = M_PI;
    CGFloat endAngle = 0;
    
    // วาด track
    CGContextSetStrokeColorWithColor(ctx, [UIColor systemGray5Color].CGColor);
    CGContextSetLineWidth(ctx, 15.0);
    CGContextAddArc(ctx, center.x, center.y, radius, startAngle, endAngle, 0);
    CGContextStrokePath(ctx);
    
    // วาด colored zones
    CGFloat zoneAngles[] = {M_PI, M_PI + M_PI/3, M_PI + 2*M_PI/3, 0};
    UIColor *zoneColors[] = {
        [UIColor systemGreenColor],
        [UIColor systemYellowColor],
        [UIColor systemRedColor]
    };
    
    for (int i = 0; i < 3; i++) {
        CGContextSetStrokeColorWithColor(ctx, zoneColors[i].CGColor);
        CGContextSetLineWidth(ctx, 15.0);
        CGContextAddArc(ctx, center.x, center.y, radius, 
                        zoneAngles[i], zoneAngles[i + 1], 0);
        CGContextStrokePath(ctx);
    }
    
    // วาดเข็ม
    CGFloat needleAngle = startAngle + self.value * M_PI;
    CGFloat needleLength = radius - 10;
    
    CGFloat needleEndX = center.x + needleLength * cos(needleAngle);
    CGFloat needleEndY = center.y + needleLength * sin(needleAngle);
    
    CGContextSetStrokeColorWithColor(ctx, [UIColor darkGrayColor].CGColor);
    CGContextSetLineWidth(ctx, 3.0);
    CGContextSetLineCap(ctx, kCGLineCapRound);
    CGContextMoveToPoint(ctx, center.x, center.y);
    CGContextAddLineToPoint(ctx, needleEndX, needleEndY);
    CGContextStrokePath(ctx);
    
    // วาดจุดศูนย์กลาง
    CGContextSetFillColorWithColor(ctx, [UIColor darkGrayColor].CGColor);
    CGContextFillEllipseInRect(ctx, CGRectMake(center.x - 8, center.y - 8, 16, 16));
    
    // แสดงค่า
    NSString *valueText = [NSString stringWithFormat:@"%.0f%%", self.value * 100];
    NSDictionary *attrs = @{
        NSFontAttributeName: [UIFont boldSystemFontOfSize:20],
        NSForegroundColorAttributeName: [UIColor labelColor]
    };
    CGSize textSize = [valueText sizeWithAttributes:attrs];
    [valueText drawAtPoint:CGPointMake(
        center.x - textSize.width / 2,
        center.y - 40) withAttributes:attrs];
}

@end
```

---

## 13. Shadow Effects

```objc
- (void)drawWithShadow:(CGContextRef)ctx {
    CGContextSaveGState(ctx);
    
    // ตั้งค่า shadow
    CGSize shadowOffset = CGSizeMake(5, 5);
    CGFloat shadowBlur = 10.0;
    CGContextSetShadowWithColor(ctx, shadowOffset, shadowBlur, 
                                 [UIColor systemGrayColor].CGColor);
    
    // วาดรูปที่จะมีเงา
    CGContextSetFillColorWithColor(ctx, [UIColor systemBlueColor].CGColor);
    UIBezierPath *path = [UIBezierPath bezierPathWithRoundedRect:
                          CGRectMake(50, 50, 200, 100) cornerRadius:12];
    CGContextAddPath(ctx, path.CGPath);
    CGContextFillPath(ctx);
    
    CGContextRestoreGState(ctx);
}
```

---

## 14. Line Styles

```objc
- (void)demonstrateLineStyles:(CGContextRef)ctx {
    NSArray *capStyles = @[@(kCGLineCapButt), @(kCGLineCapRound), @(kCGLineCapSquare)];
    NSArray *joinStyles = @[@(kCGLineJoinMiter), @(kCGLineJoinRound), @(kCGLineJoinBevel)];
    
    CGContextSetLineWidth(ctx, 8.0);
    CGContextSetStrokeColorWithColor(ctx, [UIColor systemBlueColor].CGColor);
    
    // Line caps
    for (NSInteger i = 0; i < capStyles.count; i++) {
        CGContextSetLineCap(ctx, [capStyles[i] intValue]);
        CGContextMoveToPoint(ctx, 20, 40 + i * 40);
        CGContextAddLineToPoint(ctx, 200, 40 + i * 40);
        CGContextStrokePath(ctx);
    }
    
    // Line joins
    for (NSInteger i = 0; i < joinStyles.count; i++) {
        CGContextSetLineJoin(ctx, [joinStyles[i] intValue]);
        CGContextMoveToPoint(ctx, 220, 20 + i * 50);
        CGContextAddLineToPoint(ctx, 270, 60 + i * 50);
        CGContextAddLineToPoint(ctx, 320, 20 + i * 50);
        CGContextStrokePath(ctx);
    }
}
```

---

## แบบฝึกหัด (Practice Exercises)

### แบบฝึกหัดที่ 1: Custom Clock View
```objc
// สร้าง analog clock ที่ update ทุกวินาที
@interface ClockView : UIView
@property (nonatomic, strong) NSTimer *timer;
@end

@implementation ClockView

- (void)startClock {
    self.timer = [NSTimer scheduledTimerWithTimeInterval:1.0
                                                 target:self
                                               selector:@selector(tick)
                                               userInfo:nil
                                                repeats:YES];
}

- (void)tick {
    [self setNeedsDisplay];
}

- (void)drawRect:(CGRect)rect {
    CGContextRef ctx = UIGraphicsGetCurrentContext();
    CGPoint center = CGPointMake(CGRectGetMidX(rect), CGRectGetMidY(rect));
    CGFloat radius = MIN(rect.size.width, rect.size.height) / 2 - 10;
    
    // วาดหน้าปัด
    CGContextSetFillColorWithColor(ctx, [UIColor whiteColor].CGColor);
    CGContextAddEllipseInRect(ctx, CGRectInset(rect, 10, 10));
    CGContextFillPath(ctx);
    
    CGContextSetStrokeColorWithColor(ctx, [UIColor darkGrayColor].CGColor);
    CGContextSetLineWidth(ctx, 3.0);
    CGContextAddEllipseInRect(ctx, CGRectInset(rect, 10, 10));
    CGContextStrokePath(ctx);
    
    // วาด tick marks
    for (int hour = 0; hour < 12; hour++) {
        CGFloat angle = (hour / 12.0) * M_PI * 2 - M_PI_2;
        CGFloat innerR = (hour % 3 == 0) ? radius - 15 : radius - 8;
        
        CGFloat startX = center.x + innerR * cos(angle);
        CGFloat startY = center.y + innerR * sin(angle);
        CGFloat endX = center.x + radius * cos(angle);
        CGFloat endY = center.y + radius * sin(angle);
        
        CGContextSetLineWidth(ctx, (hour % 3 == 0) ? 2.5 : 1.0);
        CGContextMoveToPoint(ctx, startX, startY);
        CGContextAddLineToPoint(ctx, endX, endY);
        CGContextStrokePath(ctx);
    }
    
    // วาดเข็มจากเวลาปัจจุบัน
    NSDate *now = [NSDate date];
    NSCalendar *cal = [NSCalendar currentCalendar];
    NSInteger hours = [cal component:NSCalendarUnitHour fromDate:now] % 12;
    NSInteger minutes = [cal component:NSCalendarUnitMinute fromDate:now];
    NSInteger seconds = [cal component:NSCalendarUnitSecond fromDate:now];
    
    // เข็มชั่วโมง
    CGFloat hourAngle = ((hours + minutes / 60.0) / 12.0) * M_PI * 2 - M_PI_2;
    [self drawHandInContext:ctx center:center angle:hourAngle length:radius * 0.5 width:4 color:[UIColor blackColor]];
    
    // เข็มนาที
    CGFloat minAngle = (minutes / 60.0) * M_PI * 2 - M_PI_2;
    [self drawHandInContext:ctx center:center angle:minAngle length:radius * 0.7 width:3 color:[UIColor blackColor]];
    
    // เข็มวินาที
    CGFloat secAngle = (seconds / 60.0) * M_PI * 2 - M_PI_2;
    [self drawHandInContext:ctx center:center angle:secAngle length:radius * 0.85 width:1.5 color:[UIColor systemRedColor]];
    
    // จุดกลาง
    CGContextSetFillColorWithColor(ctx, [UIColor systemRedColor].CGColor);
    CGContextFillEllipseInRect(ctx, CGRectMake(center.x - 5, center.y - 5, 10, 10));
}

- (void)drawHandInContext:(CGContextRef)ctx center:(CGPoint)center 
                   angle:(CGFloat)angle length:(CGFloat)length 
                   width:(CGFloat)width color:(UIColor *)color {
    
    CGContextSetStrokeColorWithColor(ctx, color.CGColor);
    CGContextSetLineWidth(ctx, width);
    CGContextSetLineCap(ctx, kCGLineCapRound);
    
    CGContextMoveToPoint(ctx, center.x, center.y);
    CGContextAddLineToPoint(ctx, 
                            center.x + length * cos(angle),
                            center.y + length * sin(angle));
    CGContextStrokePath(ctx);
}

@end
```

### แบบฝึกหัดที่ 2: Signature Pad
```objc
@interface SignaturePad : UIView
@property (nonatomic, strong) NSMutableArray<NSArray<NSValue *> *> *strokes;
@property (nonatomic, strong) NSMutableArray<NSValue *> *currentStroke;
@property (nonatomic, strong) UIColor *penColor;
@property (nonatomic, assign) CGFloat penWidth;

- (void)clear;
- (UIImage *)signatureImage;
@end

@implementation SignaturePad

- (instancetype)initWithFrame:(CGRect)frame {
    self = [super initWithFrame:frame];
    if (self) {
        _strokes = [NSMutableArray array];
        _penColor = [UIColor blackColor];
        _penWidth = 3.0;
        self.backgroundColor = [UIColor whiteColor];
    }
    return self;
}

- (void)touchesBegan:(NSSet<UITouch *> *)touches withEvent:(UIEvent *)event {
    UITouch *touch = [touches anyObject];
    CGPoint point = [touch locationInView:self];
    self.currentStroke = [NSMutableArray arrayWithObject:[NSValue valueWithCGPoint:point]];
}

- (void)touchesMoved:(NSSet<UITouch *> *)touches withEvent:(UIEvent *)event {
    UITouch *touch = [touches anyObject];
    CGPoint point = [touch locationInView:self];
    [self.currentStroke addObject:[NSValue valueWithCGPoint:point]];
    [self setNeedsDisplay];
}

- (void)touchesEnded:(NSSet<UITouch *> *)touches withEvent:(UIEvent *)event {
    if (self.currentStroke.count > 0) {
        [self.strokes addObject:[self.currentStroke copy]];
    }
    self.currentStroke = nil;
}

- (void)drawRect:(CGRect)rect {
    CGContextRef ctx = UIGraphicsGetCurrentContext();
    
    CGContextSetStrokeColorWithColor(ctx, self.penColor.CGColor);
    CGContextSetLineWidth(ctx, self.penWidth);
    CGContextSetLineCap(ctx, kCGLineCapRound);
    CGContextSetLineJoin(ctx, kCGLineJoinRound);
    
    NSArray *allStrokes = [self.strokes copy];
    if (self.currentStroke) {
        allStrokes = [allStrokes arrayByAddingObject:self.currentStroke];
    }
    
    for (NSArray<NSValue *> *stroke in allStrokes) {
        if (stroke.count < 2) continue;
        
        CGContextBeginPath(ctx);
        CGContextMoveToPoint(ctx, stroke[0].CGPointValue.x, stroke[0].CGPointValue.y);
        
        for (NSInteger i = 1; i < stroke.count; i++) {
            CGContextAddLineToPoint(ctx, stroke[i].CGPointValue.x, stroke[i].CGPointValue.y);
        }
        CGContextStrokePath(ctx);
    }
}

- (void)clear {
    [self.strokes removeAllObjects];
    self.currentStroke = nil;
    [self setNeedsDisplay];
}

- (UIImage *)signatureImage {
    UIGraphicsBeginImageContextWithOptions(self.bounds.size, YES, 0.0);
    [self drawRect:self.bounds];
    UIImage *image = UIGraphicsGetImageFromCurrentImageContext();
    UIGraphicsEndImageContext();
    return image;
}

@end
```

---

## สรุป

ในบทนี้เราได้เรียนรู้ Core Graphics อย่างครอบคลุม:

1. **CGContext** - หลักการทำงานและ state machine
2. **drawRect:** - การวาดใน custom UIView
3. **Lines & Shapes** - เส้น สี่เหลี่ยม วงกลม และรูปทรงซับซ้อน
4. **Bezier Curves** - Quadratic และ Cubic curves
5. **Colors & Fills** - สี pattern และ gradient fills
6. **Text Drawing** - UIKit attributes และ Core Text
7. **Image Drawing** - การวาดรูปภาพบน context
8. **Gradients** - Linear และ Radial gradients
9. **Clipping** - การจำกัดพื้นที่วาด
10. **Transforms** - Translation, rotation, scaling
11. **Bitmap Context** - การจัดการ pixel ระดับต่ำ
12. **PDF Drawing** - การสร้างไฟล์ PDF
13. **Custom Components** - Progress ring, Bar chart, Gauge, Clock

ในบทต่อไปจะเรียนรู้เรื่อง Core Animation ที่เพิ่มการเคลื่อนไหวให้กับ layers และ views
