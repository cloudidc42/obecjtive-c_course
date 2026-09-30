# Part 26: NSNumber และ NSValue

## บทนำ

ใน Objective-C มีการแยกระหว่าง **Primitive Types** (int, float, double, BOOL, char) และ **Object Types** (NSString, NSArray เป็นต้น) ปัญหาเกิดขึ้นเมื่อเราต้องการเก็บค่า primitive ใน Collection เช่น NSArray หรือ NSDictionary ซึ่งรับได้เฉพาะ objects เท่านั้น

**NSNumber** และ **NSValue** เป็นคำตอบของปัญหานี้ - พวกมันทำหน้าที่เป็น "กล่อง" (wrapper/box) ที่ห่อค่า primitive ให้กลายเป็น object

---

## 26.1 NSNumber - Wrapper สำหรับ Primitive Types

### แนวคิด Boxing และ Unboxing

```
int value = 42;        ← Primitive type
NSNumber *boxed = @42; ← Boxed (wrapped) เป็น Object
int unboxed = [boxed intValue]; ← Unboxed กลับเป็น primitive
```

### NSNumber รองรับ Primitive Types ต่อไปนี้

| Primitive Type | NSNumber Method | ค่าตัวอย่าง |
|---------------|-----------------|------------|
| int | intValue | @42 |
| long | longValue | @42L |
| long long | longLongValue | @42LL |
| float | floatValue | @3.14f |
| double | doubleValue | @3.14 |
| BOOL | boolValue | @YES / @NO |
| char | charValue | @'A' |
| unsigned int | unsignedIntValue | @42U |

---

## 26.2 การสร้าง NSNumber

### วิธีแบบดั้งเดิม (Traditional Way)

```objc
#import <Foundation/Foundation.h>

int main(int argc, const char * argv[]) {
    @autoreleasepool {
        
        // วิธีแบบดั้งเดิม - verbose
        NSNumber *intNum = [NSNumber numberWithInt:42];
        NSNumber *floatNum = [NSNumber numberWithFloat:3.14f];
        NSNumber *doubleNum = [NSNumber numberWithDouble:3.14159265];
        NSNumber *boolNum = [NSNumber numberWithBool:YES];
        NSNumber *longNum = [NSNumber numberWithLong:1000000L];
        NSNumber *charNum = [NSNumber numberWithChar:'A'];
        
        NSLog(@"Int: %@", intNum);
        NSLog(@"Float: %@", floatNum);
        NSLog(@"Double: %@", doubleNum);
        NSLog(@"Bool: %@", boolNum);
        NSLog(@"Long: %@", longNum);
        NSLog(@"Char: %@", charNum);
        
    }
    return 0;
}
```

### วิธีใช้ Literal Syntax (Modern Way)

```objc
#import <Foundation/Foundation.h>

int main(int argc, const char * argv[]) {
    @autoreleasepool {
        
        // NSNumber Literals - สั้นกระชับกว่า
        NSNumber *intNum    = @42;
        NSNumber *negInt    = @-10;
        NSNumber *floatNum  = @3.14f;
        NSNumber *doubleNum = @3.14159265;
        NSNumber *boolYes   = @YES;
        NSNumber *boolNo    = @NO;
        NSNumber *longNum   = @1000000L;
        NSNumber *charNum   = @'A';
        
        NSLog(@"@42       = %@", intNum);
        NSLog(@"@-10      = %@", negInt);
        NSLog(@"@3.14f    = %@", floatNum);
        NSLog(@"@3.14159  = %@", doubleNum);
        NSLog(@"@YES      = %@", boolYes);
        NSLog(@"@NO       = %@", boolNo);
        NSLog(@"@1000000L = %@", longNum);
        NSLog(@"@'A'      = %@", charNum);
        
        // Literal จาก expression
        int x = 10;
        NSNumber *fromVar = @(x);     // ต้องใช้ @() สำหรับ expression
        NSNumber *computed = @(x * 2 + 1);
        NSLog(@"@(x) = %@", fromVar);
        NSLog(@"@(x*2+1) = %@", computed);
        
        // Literal จาก enum
        typedef NS_ENUM(NSInteger, Direction) {
            DirectionNorth = 0,
            DirectionSouth,
            DirectionEast,
            DirectionWest
        };
        Direction dir = DirectionEast;
        NSNumber *dirNum = @(dir);
        NSLog(@"Direction: %@", dirNum);
        
    }
    return 0;
}
```

---

## 26.3 การแปลง NSNumber กลับเป็น Primitive (Unboxing)

```objc
#import <Foundation/Foundation.h>

int main(int argc, const char * argv[]) {
    @autoreleasepool {
        
        NSNumber *num = @42;
        
        // Unboxing เป็นประเภทต่าง ๆ
        int intVal         = [num intValue];
        long longVal       = [num longValue];
        long long llVal    = [num longLongValue];
        float floatVal     = [num floatValue];
        double doubleVal   = [num doubleValue];
        BOOL boolVal       = [num boolValue];
        char charVal       = [num charValue];
        NSInteger nsintVal = [num integerValue];    // ขนาดขึ้นอยู่กับ platform
        
        NSLog(@"int:       %d", intVal);
        NSLog(@"long:      %ld", longVal);
        NSLog(@"longLong:  %lld", llVal);
        NSLog(@"float:     %f", floatVal);
        NSLog(@"double:    %lf", doubleVal);
        NSLog(@"bool:      %@", boolVal ? @"YES" : @"NO");
        NSLog(@"char:      %c", charVal);
        NSLog(@"NSInteger: %ld", (long)nsintVal);
        
        // ระวัง: การแปลงที่อาจสูญเสียข้อมูล
        NSNumber *bigNum = @(3.9999);
        NSLog(@"\n@3.9999 as int: %d", bigNum.intValue); // ได้ 3 (truncated)
        NSLog(@"@3.9999 as double: %f", bigNum.doubleValue); // ได้ 3.9999
        
        // แปลงเป็น NSString
        NSString *str = [num stringValue];
        NSLog(@"\nAs string: %@", str);
        
        // แปลงเป็น NSDecimalNumber
        NSDecimalNumber *decimal = [num decimalValue];
        // จริงๆ ต้องทำผ่าน stringValue
        NSDecimalNumber *dec = [NSDecimalNumber decimalNumberWithString:str];
        NSLog(@"As decimal: %@", dec);
        
    }
    return 0;
}
```

---

## 26.4 NSNumber Arithmetic - การคำนวณ

NSNumber เองไม่มี arithmetic operators โดยตรง ต้องแปลงกลับเป็น primitive ก่อน

```objc
#import <Foundation/Foundation.h>

int main(int argc, const char * argv[]) {
    @autoreleasepool {
        
        NSNumber *a = @10;
        NSNumber *b = @3;
        
        // ต้องแปลงเป็น primitive ก่อน
        int sum   = [a intValue] + [b intValue];
        int diff  = [a intValue] - [b intValue];
        int prod  = [a intValue] * [b intValue];
        double div = [a doubleValue] / [b doubleValue];
        int mod   = [a intValue] % [b intValue];
        
        NSLog(@"%@ + %@ = %@", a, b, @(sum));
        NSLog(@"%@ - %@ = %@", a, b, @(diff));
        NSLog(@"%@ * %@ = %@", a, b, @(prod));
        NSLog(@"%@ / %@ = %@", a, b, @(div));
        NSLog(@"%@ %% %@ = %@", a, b, @(mod));
        
        // สร้าง helper ที่ดูดีกว่า
        NSLog(@"\nUsing expressions:");
        NSNumber *result1 = @([a intValue] + [b intValue]);
        NSNumber *result2 = @([a doubleValue] / [b doubleValue]);
        NSLog(@"a + b = %@", result1);
        NSLog(@"a / b = %@", result2);
        
        // ตัวอย่างการคำนวณใน loop
        NSArray *prices = @[@29.99, @9.50, @149.00, @4.99];
        double total = 0;
        for (NSNumber *price in prices) {
            total += [price doubleValue];
        }
        NSLog(@"\nTotal: $%.2f", total);
        NSLog(@"As NSNumber: %@", @(total));
        
    }
    return 0;
}
```

### Helper Category สำหรับ Arithmetic

```objc
#import <Foundation/Foundation.h>

@interface NSNumber (Arithmetic)
- (NSNumber *)add:(NSNumber *)other;
- (NSNumber *)subtract:(NSNumber *)other;
- (NSNumber *)multiply:(NSNumber *)other;
- (NSNumber *)divide:(NSNumber *)other;
@end

@implementation NSNumber (Arithmetic)

- (NSNumber *)add:(NSNumber *)other {
    return @(self.doubleValue + other.doubleValue);
}

- (NSNumber *)subtract:(NSNumber *)other {
    return @(self.doubleValue - other.doubleValue);
}

- (NSNumber *)multiply:(NSNumber *)other {
    return @(self.doubleValue * other.doubleValue);
}

- (NSNumber *)divide:(NSNumber *)other {
    if (other.doubleValue == 0) return nil;
    return @(self.doubleValue / other.doubleValue);
}

@end

int main(int argc, const char * argv[]) {
    @autoreleasepool {
        
        NSNumber *x = @15;
        NSNumber *y = @4;
        
        NSLog(@"%@ + %@ = %@", x, y, [x add:y]);
        NSLog(@"%@ - %@ = %@", x, y, [x subtract:y]);
        NSLog(@"%@ * %@ = %@", x, y, [x multiply:y]);
        NSLog(@"%@ / %@ = %@", x, y, [x divide:y]);
        
    }
    return 0;
}
```

---

## 26.5 NSNumber Comparison - การเปรียบเทียบ

```objc
#import <Foundation/Foundation.h>

int main(int argc, const char * argv[]) {
    @autoreleasepool {
        
        NSNumber *num1 = @10;
        NSNumber *num2 = @20;
        NSNumber *num3 = @10;
        
        // compare: ส่งคืน NSComparisonResult
        NSComparisonResult result = [num1 compare:num2];
        switch (result) {
            case NSOrderedAscending:
                NSLog(@"%@ < %@", num1, num2);
                break;
            case NSOrderedDescending:
                NSLog(@"%@ > %@", num1, num2);
                break;
            case NSOrderedSame:
                NSLog(@"%@ == %@", num1, num2);
                break;
        }
        
        // isEqualToNumber:
        if ([num1 isEqualToNumber:num3]) {
            NSLog(@"%@ equals %@", num1, num3);
        }
        
        // ระวัง: อย่าใช้ == สำหรับ NSNumber objects
        NSNumber *a = @100;
        NSNumber *b = @100;
        
        // อาจไม่ถูกต้องสำหรับทุกค่า (object identity, not value)
        if (a == b) {
            NSLog(@"Same object (might work for small ints due to caching)");
        }
        
        // ถูกต้อง
        if ([a isEqualToNumber:b]) {
            NSLog(@"Same value: %@ == %@", a, b);
        }
        
        // เรียงลำดับ NSNumber ใน Array
        NSArray *numbers = @[@5, @2, @8, @1, @9, @3];
        NSArray *sorted = [numbers sortedArrayUsingSelector:@selector(compare:)];
        NSLog(@"\nOriginal: %@", numbers);
        NSLog(@"Sorted: %@", sorted);
        
        // เรียงจากมากไปน้อย
        NSArray *descending = [numbers sortedArrayUsingComparator:
            ^NSComparisonResult(NSNumber *a, NSNumber *b) {
                return [b compare:a]; // กลับ a, b
            }];
        NSLog(@"Descending: %@", descending);
        
        // หาค่า min/max
        NSNumber *minNum = [sorted firstObject];
        NSNumber *maxNum = [sorted lastObject];
        NSLog(@"\nMin: %@, Max: %@", minNum, maxNum);
        
        // ด้วย KVC
        NSArray *values = @[@3, @1, @4, @1, @5, @9, @2, @6];
        NSLog(@"Min via KVC: %@", [values valueForKeyPath:@"@min.self"]);
        NSLog(@"Max via KVC: %@", [values valueForKeyPath:@"@max.self"]);
        NSLog(@"Sum via KVC: %@", [values valueForKeyPath:@"@sum.self"]);
        NSLog(@"Avg via KVC: %@", [values valueForKeyPath:@"@avg.self"]);
        
    }
    return 0;
}
```

---

## 26.6 NSNumber กับ Collections

```objc
#import <Foundation/Foundation.h>

int main(int argc, const char * argv[]) {
    @autoreleasepool {
        
        // เก็บ NSNumber ใน NSArray
        NSArray *scores = @[@95, @87, @92, @78, @88, @96];
        NSLog(@"Scores: %@", scores);
        
        // คำนวณค่าต่าง ๆ
        double sum = 0;
        NSNumber *highest = nil;
        NSNumber *lowest = nil;
        
        for (NSNumber *score in scores) {
            sum += score.doubleValue;
            if (!highest || [score compare:highest] == NSOrderedDescending) {
                highest = score;
            }
            if (!lowest || [score compare:lowest] == NSOrderedAscending) {
                lowest = score;
            }
        }
        
        double average = sum / scores.count;
        NSLog(@"Sum: %.0f", sum);
        NSLog(@"Average: %.2f", average);
        NSLog(@"Highest: %@", highest);
        NSLog(@"Lowest: %@", lowest);
        
        // เก็บใน NSDictionary
        NSDictionary *config = @{
            @"maxRetries"  : @3,
            @"timeout"     : @30.0,
            @"debugMode"   : @NO,
            @"version"     : @2
        };
        
        NSInteger retries = [config[@"maxRetries"] integerValue];
        double timeout = [config[@"timeout"] doubleValue];
        BOOL debug = [config[@"debugMode"] boolValue];
        
        NSLog(@"\nMax retries: %ld", (long)retries);
        NSLog(@"Timeout: %.1f", timeout);
        NSLog(@"Debug mode: %@", debug ? @"ON" : @"OFF");
        
        // ใช้ NSSet กับ NSNumber
        NSSet *uniqueScores = [NSSet setWithArray:@[@85, @92, @85, @78, @92]];
        NSLog(@"\nUnique scores: %@", 
              [uniqueScores.allObjects sortedArrayUsingSelector:@selector(compare:)]);
        
    }
    return 0;
}
```

---

## 26.7 NSValue - Wrapper สำหรับ Structs และ Pointers

NSValue ใช้สำหรับ wrap ค่าที่ไม่ใช่ object เช่น structs (CGPoint, CGRect, CGSize) และ pointers

> หมายเหตุ: NSNumber เป็น subclass ของ NSValue

### NSValue กับ CGPoint

```objc
#import <Foundation/Foundation.h>
#import <CoreGraphics/CoreGraphics.h>

int main(int argc, const char * argv[]) {
    @autoreleasepool {
        
        // CGPoint
        CGPoint point = CGPointMake(10.5, 20.3);
        
        // Boxing: CGPoint -> NSValue
        NSValue *pointValue = [NSValue valueWithCGPoint:point];
        NSLog(@"Boxed point: %@", pointValue);
        
        // Unboxing: NSValue -> CGPoint
        CGPoint unboxedPoint = [pointValue CGPointValue];
        NSLog(@"x: %.1f, y: %.1f", unboxedPoint.x, unboxedPoint.y);
        
        // เก็บใน Array
        NSMutableArray *waypoints = [NSMutableArray array];
        [waypoints addObject:[NSValue valueWithCGPoint:CGPointMake(0, 0)]];
        [waypoints addObject:[NSValue valueWithCGPoint:CGPointMake(100, 0)]];
        [waypoints addObject:[NSValue valueWithCGPoint:CGPointMake(100, 200)]];
        [waypoints addObject:[NSValue valueWithCGPoint:CGPointMake(0, 200)]];
        
        NSLog(@"\nWaypoints:");
        for (NSValue *v in waypoints) {
            CGPoint p = [v CGPointValue];
            NSLog(@"  (%.0f, %.0f)", p.x, p.y);
        }
        
    }
    return 0;
}
```

### NSValue กับ CGRect

```objc
#import <Foundation/Foundation.h>
#import <CoreGraphics/CoreGraphics.h>

int main(int argc, const char * argv[]) {
    @autoreleasepool {
        
        // CGRect
        CGRect rect = CGRectMake(10, 20, 200, 100);
        
        NSValue *rectValue = [NSValue valueWithCGRect:rect];
        CGRect unboxed = [rectValue CGRectValue];
        
        NSLog(@"Rect: origin=(%.0f,%.0f) size=(%.0fx%.0f)",
              unboxed.origin.x, unboxed.origin.y,
              unboxed.size.width, unboxed.size.height);
        
        // CGSize
        CGSize size = CGSizeMake(320, 568);
        NSValue *sizeValue = [NSValue valueWithCGSize:size];
        CGSize unboxedSize = [sizeValue CGSizeValue];
        NSLog(@"Size: %.0f x %.0f", unboxedSize.width, unboxedSize.height);
        
        // ใช้ประโยชน์จริง: เก็บ frame ของ views
        NSDictionary *frames = @{
            @"header"  : [NSValue valueWithCGRect:CGRectMake(0, 0, 320, 64)],
            @"content" : [NSValue valueWithCGRect:CGRectMake(0, 64, 320, 400)],
            @"footer"  : [NSValue valueWithCGRect:CGRectMake(0, 464, 320, 49)]
        };
        
        for (NSString *view in frames) {
            CGRect frame = [frames[view] CGRectValue];
            NSLog(@"%@: y=%.0f height=%.0f", view, frame.origin.y, frame.size.height);
        }
        
    }
    return 0;
}
```

### NSValue กับ NSRange

```objc
#import <Foundation/Foundation.h>

int main(int argc, const char * argv[]) {
    @autoreleasepool {
        
        // NSRange
        NSRange range = NSMakeRange(5, 10);
        
        NSValue *rangeValue = [NSValue valueWithRange:range];
        NSRange unboxed = [rangeValue rangeValue];
        
        NSLog(@"Range: location=%lu, length=%lu",
              (unsigned long)unboxed.location,
              (unsigned long)unboxed.length);
        
        // ใช้ประโยชน์: เก็บ ranges ของ text formatting
        NSString *text = @"Hello, World! Welcome to Objective-C.";
        
        NSMutableArray *boldRanges = [NSMutableArray array];
        
        NSRange helloRange = [text rangeOfString:@"Hello"];
        NSRange worldRange = [text rangeOfString:@"World"];
        NSRange objcRange  = [text rangeOfString:@"Objective-C"];
        
        if (helloRange.location != NSNotFound)
            [boldRanges addObject:[NSValue valueWithRange:helloRange]];
        if (worldRange.location != NSNotFound)
            [boldRanges addObject:[NSValue valueWithRange:worldRange]];
        if (objcRange.location != NSNotFound)
            [boldRanges addObject:[NSValue valueWithRange:objcRange]];
        
        NSLog(@"\nText: %@", text);
        NSLog(@"Bold ranges:");
        for (NSValue *rv in boldRanges) {
            NSRange r = [rv rangeValue];
            NSString *word = [text substringWithRange:r];
            NSLog(@"  '%@' at %lu, length %lu", word,
                  (unsigned long)r.location, (unsigned long)r.length);
        }
        
    }
    return 0;
}
```

### NSValue กับ Custom Structs

```objc
#import <Foundation/Foundation.h>

// Custom struct
typedef struct {
    int red;
    int green;
    int blue;
    float alpha;
} RGBAColor;

int main(int argc, const char * argv[]) {
    @autoreleasepool {
        
        RGBAColor red = {255, 0, 0, 1.0f};
        RGBAColor green = {0, 255, 0, 1.0f};
        RGBAColor blue = {0, 0, 255, 0.5f};
        
        // Boxing custom struct
        // ต้องใช้ @encode เพื่อบอก type encoding
        NSValue *redValue = [NSValue value:&red
                              withObjCType:@encode(RGBAColor)];
        
        // Unboxing
        RGBAColor unboxed;
        [redValue getValue:&unboxed];
        NSLog(@"Red: rgb(%d,%d,%d) alpha:%.1f",
              unboxed.red, unboxed.green, unboxed.blue, unboxed.alpha);
        
        // เก็บใน Array
        NSArray *palette = @[
            [NSValue value:&red   withObjCType:@encode(RGBAColor)],
            [NSValue value:&green withObjCType:@encode(RGBAColor)],
            [NSValue value:&blue  withObjCType:@encode(RGBAColor)]
        ];
        
        NSLog(@"\nPalette has %lu colors", (unsigned long)palette.count);
        for (NSValue *cv in palette) {
            RGBAColor c;
            [cv getValue:&c];
            NSLog(@"  rgb(%d,%d,%d) alpha:%.1f", c.red, c.green, c.blue, c.alpha);
        }
        
    }
    return 0;
}
```

---

## 26.8 NSValue กับ Pointers

```objc
#import <Foundation/Foundation.h>

int main(int argc, const char * argv[]) {
    @autoreleasepool {
        
        // Boxing a pointer
        int x = 42;
        int *ptr = &x;
        
        NSValue *ptrValue = [NSValue valueWithPointer:ptr];
        NSLog(@"Pointer value: %@", ptrValue);
        
        // Unboxing
        int *unboxedPtr = (int *)[ptrValue pointerValue];
        NSLog(@"Dereferenced: %d", *unboxedPtr);
        
        // ระวัง: pointer ต้องยังคงถูกต้องเมื่อ unbox
        // ถ้า x ถูก deallocate ก่อน จะเกิด undefined behavior
        
        // ใช้ประโยชน์: เก็บ function pointer
        void (*funcPtr)(void) = ^{ NSLog(@"Hello from function pointer!"); };
        NSValue *funcValue = [NSValue valueWithPointer:funcPtr];
        
        void (*retrievedFunc)(void) = (void (*)(void))[funcValue pointerValue];
        retrievedFunc(); // เรียกใช้งาน
        
    }
    return 0;
}
```

---

## 26.9 Boxing และ Unboxing Patterns

### Pattern 1: ใช้ NSUserDefaults กับ Primitive Types

```objc
#import <Foundation/Foundation.h>

int main(int argc, const char * argv[]) {
    @autoreleasepool {
        
        NSUserDefaults *defaults = [NSUserDefaults standardUserDefaults];
        
        // NSUserDefaults รับ primitive ได้โดยตรง
        [defaults setInteger:25 forKey:@"userAge"];
        [defaults setFloat:1.75f forKey:@"userHeight"];
        [defaults setBool:YES forKey:@"darkMode"];
        [defaults setDouble:3.14159 forKey:@"pi"];
        
        // อ่านกลับ
        NSInteger age = [defaults integerForKey:@"userAge"];
        float height  = [defaults floatForKey:@"userHeight"];
        BOOL dark     = [defaults boolForKey:@"darkMode"];
        
        NSLog(@"Age: %ld", (long)age);
        NSLog(@"Height: %.2f", height);
        NSLog(@"Dark mode: %@", dark ? @"ON" : @"OFF");
        
        // หรือเก็บเป็น NSNumber ถ้าต้องการ
        [defaults setObject:@(42) forKey:@"level"];
        NSNumber *level = [defaults objectForKey:@"level"];
        NSLog(@"Level: %@", level);
        
    }
    return 0;
}
```

### Pattern 2: Conditional Boxing

```objc
#import <Foundation/Foundation.h>

int main(int argc, const char * argv[]) {
    @autoreleasepool {
        
        // ฟังก์ชันที่ส่งคืน nil เมื่อ invalid
        NSNumber *parseNumber(NSString *str) {
            if (str.length == 0) return nil;
            NSNumberFormatter *formatter = [[NSNumberFormatter alloc] init];
            return [formatter numberFromString:str];
        }
        
        NSArray *inputs = @[@"42", @"3.14", @"abc", @"", @"100"];
        
        for (NSString *input in inputs) {
            NSNumber *num = parseNumber(input);
            if (num) {
                NSLog(@"'%@' -> %@", input, num);
            } else {
                NSLog(@"'%@' -> invalid", input);
            }
        }
        
    }
    return 0;
}
```

### Pattern 3: JSON Serialization

```objc
#import <Foundation/Foundation.h>

int main(int argc, const char * argv[]) {
    @autoreleasepool {
        
        // NSNumber ทำงานได้กับ NSJSONSerialization
        NSDictionary *data = @{
            @"name"   : @"Alice",
            @"age"    : @30,
            @"score"  : @99.5,
            @"active" : @YES
        };
        
        // Serialize เป็น JSON
        NSError *error = nil;
        NSData *jsonData = [NSJSONSerialization dataWithJSONObject:data
                                                           options:NSJSONWritingPrettyPrinted
                                                             error:&error];
        NSString *jsonString = [[NSString alloc] initWithData:jsonData 
                                                     encoding:NSUTF8StringEncoding];
        NSLog(@"JSON:\n%@", jsonString);
        
        // Parse กลับ
        NSDictionary *parsed = [NSJSONSerialization JSONObjectWithData:jsonData
                                                               options:0
                                                                 error:&error];
        
        NSString *name = parsed[@"name"];
        NSInteger age  = [parsed[@"age"] integerValue];
        double score   = [parsed[@"score"] doubleValue];
        BOOL active    = [parsed[@"active"] boolValue];
        
        NSLog(@"\nParsed - Name: %@, Age: %ld, Score: %.1f, Active: %@",
              name, (long)age, score, active ? @"YES" : @"NO");
        
    }
    return 0;
}
```

---

## 26.10 NSDecimalNumber - การคำนวณแบบ Precise

NSDecimalNumber เหมาะสำหรับการคำนวณทางการเงินที่ต้องการความแม่นยำสูง เพราะ float/double มีปัญหาเรื่อง floating-point precision

### ปัญหาของ float/double

```objc
#import <Foundation/Foundation.h>

int main(int argc, const char * argv[]) {
    @autoreleasepool {
        
        // ปัญหาของ floating-point
        double price = 0.1 + 0.2;
        NSLog(@"0.1 + 0.2 = %.17f", price); // ได้ 0.30000000000000004 !!!
        NSLog(@"0.1 + 0.2 = %@", @(price)); // ได้ 0.3 (NSNumber rounds)
        
        // ปัญหาใหญ่กว่าเมื่อคำนวณซ้ำหลายรอบ
        double total = 0;
        for (int i = 0; i < 10; i++) {
            total += 0.1;
        }
        NSLog(@"\n0.1 * 10 = %.17f", total); // ไม่ได้ 1.0 แน่นอน!
        
    }
    return 0;
}
```

### การใช้ NSDecimalNumber

```objc
#import <Foundation/Foundation.h>

int main(int argc, const char * argv[]) {
    @autoreleasepool {
        
        // NSDecimalNumber แก้ปัญหา floating-point
        NSDecimalNumber *price1 = [NSDecimalNumber decimalNumberWithString:@"0.1"];
        NSDecimalNumber *price2 = [NSDecimalNumber decimalNumberWithString:@"0.2"];
        
        NSDecimalNumber *sum = [price1 decimalNumberByAdding:price2];
        NSLog(@"0.1 + 0.2 = %@", sum); // ได้ 0.3 แน่นอน!
        
        // ทดสอบ 0.1 * 10
        NSDecimalNumber *tenth = [NSDecimalNumber decimalNumberWithString:@"0.1"];
        NSDecimalNumber *total = [NSDecimalNumber zero];
        for (int i = 0; i < 10; i++) {
            total = [total decimalNumberByAdding:tenth];
        }
        NSLog(@"0.1 * 10 = %@", total); // ได้ 1.0 แน่นอน!
        
    }
    return 0;
}
```

### NSDecimalNumber Operations

```objc
#import <Foundation/Foundation.h>

int main(int argc, const char * argv[]) {
    @autoreleasepool {
        
        NSDecimalNumber *a = [NSDecimalNumber decimalNumberWithString:@"100.50"];
        NSDecimalNumber *b = [NSDecimalNumber decimalNumberWithString:@"7.25"];
        
        // Operations
        NSDecimalNumber *add  = [a decimalNumberByAdding:b];
        NSDecimalNumber *sub  = [a decimalNumberBySubtracting:b];
        NSDecimalNumber *mul  = [a decimalNumberByMultiplyingBy:b];
        NSDecimalNumber *div  = [a decimalNumberByDividingBy:b];
        NSDecimalNumber *pow  = [a decimalNumberByRaisingToPower:2];
        
        NSLog(@"%@ + %@ = %@", a, b, add);
        NSLog(@"%@ - %@ = %@", a, b, sub);
        NSLog(@"%@ * %@ = %@", a, b, mul);
        NSLog(@"%@ / %@ = %@", a, b, div);
        NSLog(@"%@^2 = %@", a, pow);
        
        // Rounding
        NSDecimalNumberHandler *handler = [NSDecimalNumberHandler
            decimalNumberHandlerWithRoundingMode:NSRoundPlain
                                           scale:2  // ทศนิยม 2 ตำแหน่ง
                                raiseOnExactness:NO
                                 raiseOnOverflow:NO
                                raiseOnUnderflow:NO
                             raiseOnDivideByZero:YES];
        
        NSDecimalNumber *rounded = [div decimalNumberByRoundingAccordingToBehavior:handler];
        NSLog(@"\n%@ / %@ rounded to 2dp = %@", a, b, rounded);
        
        // Comparison
        NSComparisonResult result = [a compare:b];
        if (result == NSOrderedDescending) {
            NSLog(@"\n%@ > %@", a, b);
        }
        
        // Special values
        NSLog(@"\nNaN: %@", [NSDecimalNumber notANumber]);
        NSLog(@"Zero: %@", [NSDecimalNumber zero]);
        NSLog(@"One: %@", [NSDecimalNumber one]);
        
        // ตรวจสอบ NaN
        NSDecimalNumber *nan = [NSDecimalNumber notANumber];
        if ([nan isEqual:[NSDecimalNumber notANumber]]) {
            NSLog(@"Is NaN: YES");
        }
        
    }
    return 0;
}
```

### ตัวอย่างระบบคำนวณราคา

```objc
#import <Foundation/Foundation.h>

@interface Product : NSObject
@property (nonatomic, strong) NSString *name;
@property (nonatomic, strong) NSDecimalNumber *price;
@property (nonatomic, assign) NSInteger quantity;

- (instancetype)initWithName:(NSString *)name price:(NSString *)price quantity:(NSInteger)qty;
- (NSDecimalNumber *)subtotal;
@end

@implementation Product

- (instancetype)initWithName:(NSString *)name price:(NSString *)price quantity:(NSInteger)qty {
    self = [super init];
    if (self) {
        _name = name;
        _price = [NSDecimalNumber decimalNumberWithString:price];
        _quantity = qty;
    }
    return self;
}

- (NSDecimalNumber *)subtotal {
    return [self.price decimalNumberByMultiplyingBy:
            [NSDecimalNumber decimalNumberWithString:@(self.quantity).stringValue]];
}

@end

int main(int argc, const char * argv[]) {
    @autoreleasepool {
        
        NSArray *cart = @[
            [[Product alloc] initWithName:@"iPhone Case" price:@"19.99" quantity:2],
            [[Product alloc] initWithName:@"Screen Protector" price:@"9.50" quantity:3],
            [[Product alloc] initWithName:@"USB Cable" price:@"15.00" quantity:1],
        ];
        
        NSDecimalNumber *subtotal = [NSDecimalNumber zero];
        NSLog(@"Cart Items:");
        for (Product *item in cart) {
            NSLog(@"  %@ x%ld @ $%@ = $%@",
                  item.name, (long)item.quantity, item.price, item.subtotal);
            subtotal = [subtotal decimalNumberByAdding:item.subtotal];
        }
        
        // คำนวณ tax (7%)
        NSDecimalNumber *taxRate = [NSDecimalNumber decimalNumberWithString:@"0.07"];
        NSDecimalNumber *tax = [subtotal decimalNumberByMultiplyingBy:taxRate];
        NSDecimalNumber *total = [subtotal decimalNumberByAdding:tax];
        
        // ปัดเศษ 2 ตำแหน่ง
        NSDecimalNumberHandler *rounding = [NSDecimalNumberHandler
            decimalNumberHandlerWithRoundingMode:NSRoundPlain
                                           scale:2
                                raiseOnExactness:NO
                                 raiseOnOverflow:NO
                                raiseOnUnderflow:NO
                             raiseOnDivideByZero:YES];
        
        tax   = [tax   decimalNumberByRoundingAccordingToBehavior:rounding];
        total = [total decimalNumberByRoundingAccordingToBehavior:rounding];
        
        NSLog(@"\nSubtotal: $%@", subtotal);
        NSLog(@"Tax (7%%): $%@", tax);
        NSLog(@"Total: $%@", total);
        
    }
    return 0;
}
```

---

## 26.11 NSNumberFormatter

```objc
#import <Foundation/Foundation.h>

int main(int argc, const char * argv[]) {
    @autoreleasepool {
        
        NSNumberFormatter *formatter = [[NSNumberFormatter alloc] init];
        
        // Currency format
        formatter.numberStyle = NSNumberFormatterCurrencyStyle;
        formatter.locale = [NSLocale localeWithLocaleIdentifier:@"en_US"];
        
        NSArray *prices = @[@1234.5, @0.99, @9999.00];
        for (NSNumber *price in prices) {
            NSString *formatted = [formatter stringFromNumber:price];
            NSLog(@"%@ -> %@", price, formatted);
        }
        
        // Parse currency string
        NSNumber *parsed = [formatter numberFromString:@"$1,234.50"];
        NSLog(@"\n'$1,234.50' parsed as: %@", parsed);
        
        // Decimal format
        formatter.numberStyle = NSNumberFormatterDecimalStyle;
        formatter.minimumFractionDigits = 2;
        formatter.maximumFractionDigits = 4;
        
        NSLog(@"\nDecimal format:");
        NSLog(@"%@", [formatter stringFromNumber:@3.14159265]);
        NSLog(@"%@", [formatter stringFromNumber:@1000000]);
        
        // Percent format
        formatter.numberStyle = NSNumberFormatterPercentStyle;
        NSLog(@"\nPercent format:");
        NSLog(@"%@", [formatter stringFromNumber:@0.75]);
        NSLog(@"%@", [formatter stringFromNumber:@1.5]);
        
        // Scientific format
        formatter.numberStyle = NSNumberFormatterScientificStyle;
        NSLog(@"\nScientific format:");
        NSLog(@"%@", [formatter stringFromNumber:@1234567890]);
        NSLog(@"%@", [formatter stringFromNumber:@0.000001]);
        
        // Thai locale
        formatter.numberStyle = NSNumberFormatterDecimalStyle;
        formatter.locale = [NSLocale localeWithLocaleIdentifier:@"th_TH"];
        NSLog(@"\nThai locale: %@", [formatter stringFromNumber:@1234567.89]);
        
    }
    return 0;
}
```

---

## 26.12 NSNumber กับ Objective-C Properties

```objc
#import <Foundation/Foundation.h>

@interface UserProfile : NSObject

// ใช้ NSNumber แทน primitive เมื่อต้องการ nil (optional)
@property (nonatomic, strong, nullable) NSNumber *age;         // optional int
@property (nonatomic, strong, nullable) NSNumber *rating;      // optional float
@property (nonatomic, strong, nullable) NSNumber *isVerified;  // optional bool

// primitive properties - ไม่สามารถเป็น nil
@property (nonatomic, assign) NSInteger loginCount;
@property (nonatomic, assign) BOOL isPremium;

@end

@implementation UserProfile
@end

int main(int argc, const char * argv[]) {
    @autoreleasepool {
        
        UserProfile *user = [[UserProfile alloc] init];
        
        // NSNumber สามารถเป็น nil (ยังไม่ได้ระบุ)
        NSLog(@"Age: %@", user.age);     // nil
        NSLog(@"Rating: %@", user.rating); // nil
        
        // กำหนดค่า
        user.age = @25;
        user.rating = @4.5f;
        user.isVerified = @YES;
        user.loginCount = 42;
        user.isPremium = YES;
        
        NSLog(@"Age: %@", user.age);
        NSLog(@"Rating: %@", user.rating);
        
        // ตรวจสอบ nil ก่อนใช้
        if (user.age) {
            NSLog(@"User is %ld years old", (long)user.age.integerValue);
        }
        
        // NSDictionary representation
        NSDictionary *dict = @{
            @"age"        : user.age ?: [NSNull null],
            @"rating"     : user.rating ?: [NSNull null],
            @"isVerified" : user.isVerified ?: @NO,
            @"loginCount" : @(user.loginCount),
            @"isPremium"  : @(user.isPremium)
        };
        NSLog(@"\nDictionary: %@", dict);
        
    }
    return 0;
}
```

---

## 26.13 ตัวอย่างการใช้งานจริง

### Use Case 1: การคำนวณสถิติ

```objc
#import <Foundation/Foundation.h>

@interface StatCalculator : NSObject
+ (NSDictionary *)calculateStats:(NSArray *)numbers;
@end

@implementation StatCalculator

+ (NSDictionary *)calculateStats:(NSArray *)numbers {
    if (numbers.count == 0) return @{};
    
    NSArray *sorted = [numbers sortedArrayUsingSelector:@selector(compare:)];
    
    double sum = [[numbers valueForKeyPath:@"@sum.self"] doubleValue];
    double avg = sum / numbers.count;
    double min = [[sorted firstObject] doubleValue];
    double max = [[sorted lastObject] doubleValue];
    
    // Median
    double median;
    NSInteger count = sorted.count;
    if (count % 2 == 0) {
        median = ([sorted[count/2 - 1] doubleValue] + [sorted[count/2] doubleValue]) / 2.0;
    } else {
        median = [sorted[count/2] doubleValue];
    }
    
    // Standard deviation
    double variance = 0;
    for (NSNumber *n in numbers) {
        double diff = n.doubleValue - avg;
        variance += diff * diff;
    }
    variance /= numbers.count;
    double stdDev = sqrt(variance);
    
    return @{
        @"count"  : @(numbers.count),
        @"sum"    : @(sum),
        @"min"    : @(min),
        @"max"    : @(max),
        @"mean"   : @(avg),
        @"median" : @(median),
        @"stdDev" : @(stdDev)
    };
}

@end

int main(int argc, const char * argv[]) {
    @autoreleasepool {
        
        NSArray *scores = @[@85, @92, @78, @95, @88, @73, @91, @82, @86, @90];
        
        NSDictionary *stats = [StatCalculator calculateStats:scores];
        
        NSLog(@"Statistics for scores: %@", scores);
        NSLog(@"Count:  %@", stats[@"count"]);
        NSLog(@"Min:    %@", stats[@"min"]);
        NSLog(@"Max:    %@", stats[@"max"]);
        NSLog(@"Mean:   %.2f", [stats[@"mean"] doubleValue]);
        NSLog(@"Median: %.1f", [stats[@"median"] doubleValue]);
        NSLog(@"StdDev: %.2f", [stats[@"stdDev"] doubleValue]);
        
    }
    return 0;
}
```

### Use Case 2: Configuration System

```objc
#import <Foundation/Foundation.h>

@interface AppConfig : NSObject

+ (instancetype)sharedConfig;

- (void)setNumber:(NSNumber *)value forKey:(NSString *)key;
- (NSNumber *)numberForKey:(NSString *)key defaultValue:(NSNumber *)defaultValue;

@end

@implementation AppConfig {
    NSMutableDictionary *_storage;
}

+ (instancetype)sharedConfig {
    static AppConfig *instance = nil;
    static dispatch_once_t onceToken;
    dispatch_once(&onceToken, ^{
        instance = [[AppConfig alloc] init];
    });
    return instance;
}

- (instancetype)init {
    self = [super init];
    if (self) {
        _storage = [NSMutableDictionary dictionary];
    }
    return self;
}

- (void)setNumber:(NSNumber *)value forKey:(NSString *)key {
    _storage[key] = value;
}

- (NSNumber *)numberForKey:(NSString *)key defaultValue:(NSNumber *)defaultValue {
    NSNumber *value = _storage[key];
    return value ?: defaultValue;
}

@end

int main(int argc, const char * argv[]) {
    @autoreleasepool {
        
        AppConfig *config = [AppConfig sharedConfig];
        
        // ตั้งค่า
        [config setNumber:@3    forKey:@"maxRetries"];
        [config setNumber:@30.0 forKey:@"timeout"];
        [config setNumber:@YES  forKey:@"enableLogging"];
        
        // อ่านค่าพร้อม default
        NSInteger retries = [[config numberForKey:@"maxRetries"
                                     defaultValue:@5] integerValue];
        double timeout    = [[config numberForKey:@"timeout"
                                    defaultValue:@60.0] doubleValue];
        BOOL logging      = [[config numberForKey:@"enableLogging"
                                    defaultValue:@NO] boolValue];
        NSInteger missing = [[config numberForKey:@"missingKey"
                                    defaultValue:@-1] integerValue];
        
        NSLog(@"Max Retries: %ld", (long)retries);
        NSLog(@"Timeout: %.1f", timeout);
        NSLog(@"Logging: %@", logging ? @"ON" : @"OFF");
        NSLog(@"Missing (default -1): %ld", (long)missing);
        
    }
    return 0;
}
```

---

## 26.14 แบบฝึกหัด (Practice Exercises)

### แบบฝึกหัดที่ 1: Temperature Converter

```objc
#import <Foundation/Foundation.h>

// เขียน functions แปลงอุณหภูมิ ส่งคืน NSNumber
NSNumber *celsiusToFahrenheit(NSNumber *celsius) {
    double c = celsius.doubleValue;
    double f = (c * 9.0 / 5.0) + 32.0;
    return @(f);
}

NSNumber *fahrenheitToCelsius(NSNumber *fahrenheit) {
    double f = fahrenheit.doubleValue;
    double c = (f - 32.0) * 5.0 / 9.0;
    return @(c);
}

NSNumber *celsiusToKelvin(NSNumber *celsius) {
    return @(celsius.doubleValue + 273.15);
}

int main(int argc, const char * argv[]) {
    @autoreleasepool {
        
        NSArray *celsiusTemps = @[@0, @100, @-40, @37, @20];
        
        NSLog(@"%-10s %-15s %-15s %-15s", "Celsius", "Fahrenheit", "Kelvin", "");
        NSLog(@"%s", "-----------------------------------------------");
        
        for (NSNumber *c in celsiusTemps) {
            NSNumber *f = celsiusToFahrenheit(c);
            NSNumber *k = celsiusToKelvin(c);
            NSLog(@"%-10.1f %-15.1f %-15.2f",
                  c.doubleValue, f.doubleValue, k.doubleValue);
        }
        
        // แปลงกลับ
        NSNumber *bodyTemp = celsiusToFahrenheit(@37);
        NSNumber *backToC  = fahrenheitToCelsius(bodyTemp);
        NSLog(@"\n37°C = %.1f°F = %.1f°C (back)", 
              bodyTemp.doubleValue, backToC.doubleValue);
        
    }
    return 0;
}
```

### แบบฝึกหัดที่ 2: CGPoint Operations

```objc
#import <Foundation/Foundation.h>
#import <CoreGraphics/CoreGraphics.h>
#import <math.h>

// คำนวณ distance ระหว่าง 2 จุด
NSNumber *distanceBetween(NSValue *p1Value, NSValue *p2Value) {
    CGPoint p1 = [p1Value CGPointValue];
    CGPoint p2 = [p2Value CGPointValue];
    
    double dx = p2.x - p1.x;
    double dy = p2.y - p1.y;
    return @(sqrt(dx*dx + dy*dy));
}

// หาจุดกึ่งกลาง
NSValue *midpointBetween(NSValue *p1Value, NSValue *p2Value) {
    CGPoint p1 = [p1Value CGPointValue];
    CGPoint p2 = [p2Value CGPointValue];
    
    CGPoint mid = CGPointMake((p1.x + p2.x) / 2.0, (p1.y + p2.y) / 2.0);
    return [NSValue valueWithCGPoint:mid];
}

int main(int argc, const char * argv[]) {
    @autoreleasepool {
        
        NSValue *origin = [NSValue valueWithCGPoint:CGPointMake(0, 0)];
        NSValue *point1 = [NSValue valueWithCGPoint:CGPointMake(3, 4)];
        NSValue *point2 = [NSValue valueWithCGPoint:CGPointMake(6, 8)];
        
        // Distance
        NSNumber *dist1 = distanceBetween(origin, point1);
        NSNumber *dist2 = distanceBetween(point1, point2);
        NSLog(@"Distance origin -> (3,4): %.2f", dist1.doubleValue);
        NSLog(@"Distance (3,4) -> (6,8): %.2f", dist2.doubleValue);
        
        // Midpoint
        NSValue *midVal = midpointBetween(origin, point2);
        CGPoint mid = [midVal CGPointValue];
        NSLog(@"Midpoint of (0,0) and (6,8): (%.1f, %.1f)", mid.x, mid.y);
        
        // คำนวณ perimeter ของ polygon
        NSArray *polygon = @[
            [NSValue valueWithCGPoint:CGPointMake(0, 0)],
            [NSValue valueWithCGPoint:CGPointMake(4, 0)],
            [NSValue valueWithCGPoint:CGPointMake(4, 3)],
            [NSValue valueWithCGPoint:CGPointMake(0, 3)]
        ];
        
        double perimeter = 0;
        for (NSInteger i = 0; i < polygon.count; i++) {
            NSValue *current = polygon[i];
            NSValue *next = polygon[(i + 1) % polygon.count];
            perimeter += distanceBetween(current, next).doubleValue;
        }
        NSLog(@"\nRectangle perimeter: %.1f", perimeter);
        
    }
    return 0;
}
```

### แบบฝึกหัดที่ 3: Financial Calculator ด้วย NSDecimalNumber

```objc
#import <Foundation/Foundation.h>

// คำนวณ compound interest
NSDecimalNumber *compoundInterest(NSDecimalNumber *principal, 
                                   NSDecimalNumber *annualRate,
                                   NSInteger years,
                                   NSInteger timesPerYear) {
    // A = P(1 + r/n)^(nt)
    double p = principal.doubleValue;
    double r = annualRate.doubleValue;
    double n = timesPerYear;
    double t = years;
    
    double amount = p * pow(1 + r/n, n*t);
    
    NSDecimalNumberHandler *rounding = [NSDecimalNumberHandler
        decimalNumberHandlerWithRoundingMode:NSRoundPlain
                                       scale:2
                            raiseOnExactness:NO
                             raiseOnOverflow:NO
                            raiseOnUnderflow:NO
                         raiseOnDivideByZero:YES];
    
    NSDecimalNumber *result = [NSDecimalNumber decimalNumberWithString:
                               [NSString stringWithFormat:@"%.10f", amount]];
    return [result decimalNumberByRoundingAccordingToBehavior:rounding];
}

int main(int argc, const char * argv[]) {
    @autoreleasepool {
        
        NSDecimalNumber *principal = [NSDecimalNumber decimalNumberWithString:@"10000"];
        NSDecimalNumber *rate = [NSDecimalNumber decimalNumberWithString:@"0.05"];
        
        NSLog(@"Principal: $%@", principal);
        NSLog(@"Annual Rate: %@%%", [NSDecimalNumber decimalNumberWithString:@"5"]);
        NSLog(@"\nCompound Interest Results:");
        NSLog(@"%-10s %-20s %-20s", "Years", "Annual", "Monthly");
        NSLog(@"%s", "--------------------------------------------");
        
        for (int years = 1; years <= 10; years++) {
            NSDecimalNumber *annual  = compoundInterest(principal, rate, years, 1);
            NSDecimalNumber *monthly = compoundInterest(principal, rate, years, 12);
            
            NSDecimalNumber *gainA = [annual  decimalNumberBySubtracting:principal];
            NSDecimalNumber *gainM = [monthly decimalNumberBySubtracting:principal];
            
            NSLog(@"%-10d $%-19@ $%@", years, gainA, gainM);
        }
        
    }
    return 0;
}
```

---

## สรุปบทที่ 26

ในบทนี้เราได้เรียนรู้:

1. **NSNumber** - Wrapper สำหรับ primitive types (int, float, double, BOOL, char)
2. **NSNumber Literals** - @42, @3.14, @YES, @(expression) 
3. **Unboxing** - intValue, floatValue, doubleValue, boolValue
4. **NSNumber Arithmetic** - ต้องแปลงเป็น primitive ก่อนคำนวณ
5. **NSNumber Comparison** - compare:, isEqualToNumber:
6. **NSValue** - Wrapper สำหรับ structs (CGPoint, CGRect, CGSize, NSRange) และ pointers
7. **NSDecimalNumber** - การคำนวณแม่นยำสูงสำหรับทางการเงิน
8. **NSNumberFormatter** - จัดรูปแบบตัวเลขสำหรับแสดงผล

### คำถามทบทวน

1. ทำไมต้องใช้ NSNumber แทนที่จะเก็บค่า primitive โดยตรงใน Array?
2. ความแตกต่างระหว่าง NSNumber และ NSValue คืออะไร?
3. เมื่อไหรควรใช้ NSDecimalNumber แทน double?
4. `@YES` และ `[NSNumber numberWithBool:YES]` ต่างกันอย่างไร?

---

**ต่อไป:** Part 27 - NSData
