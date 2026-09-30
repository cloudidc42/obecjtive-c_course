# ส่วนที่ 16: Categories ใน Objective-C

## บทนำ

Categories คือหนึ่งในฟีเจอร์ที่ทรงพลังที่สุดของ Objective-C ที่ช่วยให้เราสามารถเพิ่ม method ใหม่ให้กับ class ที่มีอยู่แล้วได้ โดยไม่ต้อง subclass และไม่ต้องมี source code ของ class นั้น แม้แต่ class ใน iOS SDK เช่น `NSString`, `NSArray`, หรือ `UIView` ก็สามารถเพิ่ม method ได้ด้วย categories

---

## 16.1 Categories คืออะไร?

### แนวคิดพื้นฐาน

Category ทำให้เราสามารถ:
1. เพิ่ม method ให้กับ existing class (รวมถึง system classes)
2. แบ่ง implementation ของ class ขนาดใหญ่ออกเป็นหลายไฟล์
3. สร้าง informal protocols
4. Override method ที่มีอยู่แล้ว (ระวัง: อันตราย!)

### เปรียบเทียบ Category กับ Subclass

```
Subclass:
- สร้าง class ใหม่
- สืบทอด method ทั้งหมด
- เพิ่ม instance variables ได้
- Override methods ได้
- ต้องใช้ class ใหม่

Category:
- ไม่สร้าง class ใหม่
- เพิ่ม method ให้ class เดิม
- ไม่สามารถเพิ่ม instance variables
- เพิ่ม method ใหม่ได้ (ระวัง override)
- ใช้ class เดิมได้เลย
```

---

## 16.2 Category Syntax

### การประกาศ Category

```objc
// รูปแบบ: @interface ClassName (CategoryName)

// ไฟล์: NSString+Validation.h
@interface NSString (Validation)

- (BOOL)isValidEmail;
- (BOOL)isValidPhoneNumber;
- (BOOL)isValidURL;
- (BOOL)isValidThaiID;
- (BOOL)containsOnlyNumbers;
- (BOOL)containsOnlyLetters;
- (BOOL)isBlank;

@end

// ไฟล์: NSString+Validation.m
@implementation NSString (Validation)

- (BOOL)isValidEmail {
    NSString *emailRegex = @"[A-Za-z0-9._%+-]+@[A-Za-z0-9.-]+\\.[A-Za-z]{2,64}";
    NSPredicate *emailTest = [NSPredicate predicateWithFormat:@"SELF MATCHES %@", emailRegex];
    return [emailTest evaluateWithObject:self];
}

- (BOOL)isValidPhoneNumber {
    // ตรวจสอบเบอร์โทรไทย: 0X-XXXX-XXXX หรือ 0XXXXXXXXX
    NSString *phoneRegex = @"0[0-9]{8,9}";
    NSPredicate *phoneTest = [NSPredicate predicateWithFormat:@"SELF MATCHES %@", phoneRegex];
    return [phoneTest evaluateWithObject:self];
}

- (BOOL)isValidURL {
    NSURL *url = [NSURL URLWithString:self];
    return url != nil && url.scheme != nil && url.host != nil;
}

- (BOOL)isValidThaiID {
    // เลขบัตรประชาชนไทย 13 หลัก
    if ([self length] != 13) return NO;
    
    NSCharacterSet *nonDigits = [[NSCharacterSet decimalDigitCharacterSet] invertedSet];
    if ([self rangeOfCharacterFromSet:nonDigits].location != NSNotFound) return NO;
    
    // ตรวจสอบ check digit
    NSInteger sum = 0;
    for (NSInteger i = 0; i < 12; i++) {
        sum += [[self substringWithRange:NSMakeRange(i, 1)] integerValue] * (13 - i);
    }
    
    NSInteger checkDigit = (11 - (sum % 11)) % 10;
    NSInteger lastDigit = [[self substringFromIndex:12] integerValue];
    
    return checkDigit == lastDigit;
}

- (BOOL)containsOnlyNumbers {
    NSCharacterSet *nonDigits = [[NSCharacterSet decimalDigitCharacterSet] invertedSet];
    return [self rangeOfCharacterFromSet:nonDigits].location == NSNotFound;
}

- (BOOL)containsOnlyLetters {
    NSCharacterSet *nonLetters = [[NSCharacterSet letterCharacterSet] invertedSet];
    return [self rangeOfCharacterFromSet:nonLetters].location == NSNotFound;
}

- (BOOL)isBlank {
    NSString *trimmed = [self stringByTrimmingCharactersInSet:
                         [NSCharacterSet whitespaceAndNewlineCharacterSet]];
    return [trimmed length] == 0;
}

@end
```

### การใช้งาน

```objc
#import "NSString+Validation.h"

NSString *email = @"test@example.com";
NSString *phone = @"0812345678";
NSString *thaiID = @"1234567890123";

NSLog(@"Email valid: %@", [email isValidEmail] ? @"YES" : @"NO");
NSLog(@"Phone valid: %@", [phone isValidPhoneNumber] ? @"YES" : @"NO");
NSLog(@"Thai ID valid: %@", [thaiID isValidThaiID] ? @"YES" : @"NO");

// Category method สามารถใช้บน string literal ได้เลย
BOOL isBlank = [@"   " isBlank]; // YES
NSLog(@"Is blank: %@", isBlank ? @"YES" : @"NO");
```

---

## 16.3 Categories บน NSString

### NSString+Extensions

```objc
// ไฟล์: NSString+Extensions.h
@interface NSString (Extensions)

// Case conversion
- (NSString *)capitalizeFirstLetter;
- (NSString *)lowercaseFirstLetter;
- (NSString *)camelCaseToSnakeCase;
- (NSString *)snakeCaseToCamelCase;

// Trimming
- (NSString *)trimWhitespace;
- (NSString *)trimPrefix:(NSString *)prefix;
- (NSString *)trimSuffix:(NSString *)suffix;

// Checking
- (BOOL)hasPrefix:(NSString *)prefix ignoreCase:(BOOL)ignoreCase;
- (BOOL)hasSuffix:(NSString *)suffix ignoreCase:(BOOL)ignoreCase;
- (BOOL)containsString:(NSString *)substring ignoreCase:(BOOL)ignoreCase;

// Conversion
- (NSInteger)toInteger;
- (double)toDouble;
- (BOOL)toBool;
- (NSArray<NSString *> *)toWords;

// Thai specific
- (NSInteger)thaiCharacterCount;
- (BOOL)containsThai;

// Formatting
- (NSString *)repeatString:(NSInteger)times;
- (NSString *)paddedToLength:(NSInteger)length withCharacter:(unichar)character;
- (NSString *)truncateToLength:(NSInteger)length withEllipsis:(BOOL)ellipsis;

@end

// ไฟล์: NSString+Extensions.m
@implementation NSString (Extensions)

- (NSString *)capitalizeFirstLetter {
    if ([self length] == 0) return self;
    return [NSString stringWithFormat:@"%@%@",
            [[self substringToIndex:1] uppercaseString],
            [self substringFromIndex:1]];
}

- (NSString *)lowercaseFirstLetter {
    if ([self length] == 0) return self;
    return [NSString stringWithFormat:@"%@%@",
            [[self substringToIndex:1] lowercaseString],
            [self substringFromIndex:1]];
}

- (NSString *)camelCaseToSnakeCase {
    // MyClassName -> my_class_name
    NSMutableString *result = [NSMutableString string];
    
    for (NSInteger i = 0; i < [self length]; i++) {
        unichar c = [self characterAtIndex:i];
        if (isupper(c) && i > 0) {
            [result appendString:@"_"];
        }
        [result appendFormat:@"%c", tolower(c)];
    }
    
    return [result copy];
}

- (NSString *)snakeCaseToCamelCase {
    // my_class_name -> myClassName
    NSArray *components = [self componentsSeparatedByString:@"_"];
    NSMutableString *result = [NSMutableString string];
    
    for (NSInteger i = 0; i < [components count]; i++) {
        if (i == 0) {
            [result appendString:[components[0] lowercaseString]];
        } else {
            [result appendString:[components[i] capitalizeFirstLetter]];
        }
    }
    
    return [result copy];
}

- (NSString *)trimWhitespace {
    return [self stringByTrimmingCharactersInSet:
            [NSCharacterSet whitespaceAndNewlineCharacterSet]];
}

- (NSString *)trimPrefix:(NSString *)prefix {
    if ([self hasPrefix:prefix]) {
        return [self substringFromIndex:[prefix length]];
    }
    return self;
}

- (NSString *)trimSuffix:(NSString *)suffix {
    if ([self hasSuffix:suffix]) {
        return [self substringToIndex:[self length] - [suffix length]];
    }
    return self;
}

- (BOOL)hasPrefix:(NSString *)prefix ignoreCase:(BOOL)ignoreCase {
    if (ignoreCase) {
        NSString *lowSelf = [self lowercaseString];
        NSString *lowPrefix = [prefix lowercaseString];
        return [lowSelf hasPrefix:lowPrefix];
    }
    return [self hasPrefix:prefix];
}

- (BOOL)hasSuffix:(NSString *)suffix ignoreCase:(BOOL)ignoreCase {
    if (ignoreCase) {
        return [[self lowercaseString] hasSuffix:[suffix lowercaseString]];
    }
    return [self hasSuffix:suffix];
}

- (BOOL)containsString:(NSString *)substring ignoreCase:(BOOL)ignoreCase {
    if (ignoreCase) {
        return [[self lowercaseString] rangeOfString:[substring lowercaseString]].location != NSNotFound;
    }
    return [self rangeOfString:substring].location != NSNotFound;
}

- (NSInteger)toInteger {
    return [self integerValue];
}

- (double)toDouble {
    return [self doubleValue];
}

- (BOOL)toBool {
    NSString *lower = [self lowercaseString];
    return [lower isEqualToString:@"true"] || 
           [lower isEqualToString:@"yes"] || 
           [lower isEqualToString:@"1"];
}

- (NSArray<NSString *> *)toWords {
    return [self componentsSeparatedByCharactersInSet:
            [NSCharacterSet whitespaceAndNewlineCharacterSet]];
}

- (NSInteger)thaiCharacterCount {
    NSInteger count = 0;
    for (NSInteger i = 0; i < [self length]; i++) {
        unichar c = [self characterAtIndex:i];
        if (c >= 0x0E00 && c <= 0x0E7F) { // Thai Unicode range
            count++;
        }
    }
    return count;
}

- (BOOL)containsThai {
    return [self thaiCharacterCount] > 0;
}

- (NSString *)repeatString:(NSInteger)times {
    NSMutableString *result = [NSMutableString string];
    for (NSInteger i = 0; i < times; i++) {
        [result appendString:self];
    }
    return [result copy];
}

- (NSString *)paddedToLength:(NSInteger)length withCharacter:(unichar)character {
    NSInteger paddingNeeded = length - (NSInteger)[self length];
    if (paddingNeeded <= 0) return self;
    
    NSString *padding = [[NSString stringWithFormat:@"%C", character] repeatString:paddingNeeded];
    return [self stringByAppendingString:padding];
}

- (NSString *)truncateToLength:(NSInteger)length withEllipsis:(BOOL)ellipsis {
    if ((NSInteger)[self length] <= length) return self;
    
    if (ellipsis && length > 3) {
        return [[self substringToIndex:length - 3] stringByAppendingString:@"..."];
    }
    return [self substringToIndex:length];
}

@end
```

---

## 16.4 Categories บน NSArray

```objc
// ไฟล์: NSArray+Functional.h
@interface NSArray (Functional)

// Functional operations
- (NSArray *)map:(id(^)(id obj))block;
- (NSArray *)filter:(BOOL(^)(id obj))block;
- (id)reduce:(id)initial block:(id(^)(id accumulator, id obj))block;
- (void)forEach:(void(^)(id obj, NSInteger index))block;
- (NSArray *)flatMap:(NSArray *(^)(id obj))block;

// Utility
- (id)firstWhere:(BOOL(^)(id obj))condition;
- (id)lastWhere:(BOOL(^)(id obj))condition;
- (BOOL)any:(BOOL(^)(id obj))condition;
- (BOOL)all:(BOOL(^)(id obj))condition;
- (BOOL)none:(BOOL(^)(id obj))condition;
- (NSInteger)countWhere:(BOOL(^)(id obj))condition;

// Grouping/Sorting
- (NSDictionary *)groupBy:(id(^)(id obj))keyBlock;
- (NSArray *)sortedWithComparator:(NSComparisonResult(^)(id a, id b))comparator;
- (NSArray *)unique;
- (NSArray *)uniqueBy:(id(^)(id obj))keyBlock;

// Math (สำหรับ NSNumber arrays)
- (NSNumber *)sum;
- (NSNumber *)average;
- (NSNumber *)max;
- (NSNumber *)min;

// Utility
- (NSArray *)reversed;
- (NSArray *)shuffled;
- (NSArray *)take:(NSInteger)count;
- (NSArray *)drop:(NSInteger)count;
- (NSArray *)chunksOfSize:(NSInteger)size;
- (id)randomObject;

@end

// ไฟล์: NSArray+Functional.m
@implementation NSArray (Functional)

- (NSArray *)map:(id(^)(id obj))block {
    NSMutableArray *result = [NSMutableArray arrayWithCapacity:[self count]];
    for (id obj in self) {
        id mapped = block(obj);
        if (mapped) [result addObject:mapped];
    }
    return [result copy];
}

- (NSArray *)filter:(BOOL(^)(id obj))block {
    NSMutableArray *result = [NSMutableArray array];
    for (id obj in self) {
        if (block(obj)) {
            [result addObject:obj];
        }
    }
    return [result copy];
}

- (id)reduce:(id)initial block:(id(^)(id accumulator, id obj))block {
    id accumulator = initial;
    for (id obj in self) {
        accumulator = block(accumulator, obj);
    }
    return accumulator;
}

- (void)forEach:(void(^)(id obj, NSInteger index))block {
    [self enumerateObjectsUsingBlock:^(id obj, NSUInteger idx, BOOL *stop) {
        block(obj, (NSInteger)idx);
    }];
}

- (NSArray *)flatMap:(NSArray *(^)(id obj))block {
    NSMutableArray *result = [NSMutableArray array];
    for (id obj in self) {
        NSArray *mapped = block(obj);
        if (mapped) {
            [result addObjectsFromArray:mapped];
        }
    }
    return [result copy];
}

- (id)firstWhere:(BOOL(^)(id obj))condition {
    for (id obj in self) {
        if (condition(obj)) return obj;
    }
    return nil;
}

- (id)lastWhere:(BOOL(^)(id obj))condition {
    id found = nil;
    for (id obj in self) {
        if (condition(obj)) found = obj;
    }
    return found;
}

- (BOOL)any:(BOOL(^)(id obj))condition {
    for (id obj in self) {
        if (condition(obj)) return YES;
    }
    return NO;
}

- (BOOL)all:(BOOL(^)(id obj))condition {
    for (id obj in self) {
        if (!condition(obj)) return NO;
    }
    return YES;
}

- (BOOL)none:(BOOL(^)(id obj))condition {
    return ![self any:condition];
}

- (NSInteger)countWhere:(BOOL(^)(id obj))condition {
    NSInteger count = 0;
    for (id obj in self) {
        if (condition(obj)) count++;
    }
    return count;
}

- (NSDictionary *)groupBy:(id(^)(id obj))keyBlock {
    NSMutableDictionary *result = [NSMutableDictionary dictionary];
    for (id obj in self) {
        id key = keyBlock(obj);
        if (!key) continue;
        
        NSMutableArray *group = result[key];
        if (!group) {
            group = [NSMutableArray array];
            result[key] = group;
        }
        [group addObject:obj];
    }
    return [result copy];
}

- (NSArray *)sortedWithComparator:(NSComparisonResult(^)(id a, id b))comparator {
    return [self sortedArrayUsingComparator:comparator];
}

- (NSArray *)unique {
    return [[NSOrderedSet orderedSetWithArray:self] array];
}

- (NSArray *)uniqueBy:(id(^)(id obj))keyBlock {
    NSMutableArray *result = [NSMutableArray array];
    NSMutableSet *seen = [NSMutableSet set];
    
    for (id obj in self) {
        id key = keyBlock(obj);
        if (![seen containsObject:key]) {
            [seen addObject:key];
            [result addObject:obj];
        }
    }
    return [result copy];
}

- (NSNumber *)sum {
    return [self reduce:@0 block:^(NSNumber *acc, NSNumber *obj) {
        return @([acc doubleValue] + [obj doubleValue]);
    }];
}

- (NSNumber *)average {
    if ([self count] == 0) return @0;
    return @([[self sum] doubleValue] / [self count]);
}

- (NSNumber *)max {
    return [self reduce:nil block:^(NSNumber *acc, NSNumber *obj) {
        if (!acc) return obj;
        return [obj doubleValue] > [acc doubleValue] ? obj : acc;
    }];
}

- (NSNumber *)min {
    return [self reduce:nil block:^(NSNumber *acc, NSNumber *obj) {
        if (!acc) return obj;
        return [obj doubleValue] < [acc doubleValue] ? obj : acc;
    }];
}

- (NSArray *)reversed {
    return [[self reverseObjectEnumerator] allObjects];
}

- (NSArray *)shuffled {
    NSMutableArray *result = [self mutableCopy];
    NSUInteger count = [result count];
    
    for (NSUInteger i = count - 1; i > 0; i--) {
        NSUInteger j = arc4random_uniform((uint32_t)(i + 1));
        [result exchangeObjectAtIndex:i withObjectAtIndex:j];
    }
    return [result copy];
}

- (NSArray *)take:(NSInteger)count {
    NSInteger actualCount = MIN(count, (NSInteger)[self count]);
    return [self subarrayWithRange:NSMakeRange(0, actualCount)];
}

- (NSArray *)drop:(NSInteger)count {
    NSInteger actualCount = MIN(count, (NSInteger)[self count]);
    return [self subarrayWithRange:NSMakeRange(actualCount, [self count] - actualCount)];
}

- (NSArray *)chunksOfSize:(NSInteger)size {
    NSMutableArray *chunks = [NSMutableArray array];
    NSInteger total = [self count];
    
    for (NSInteger i = 0; i < total; i += size) {
        NSInteger end = MIN(i + size, total);
        [chunks addObject:[self subarrayWithRange:NSMakeRange(i, end - i)]];
    }
    return [chunks copy];
}

- (id)randomObject {
    if ([self count] == 0) return nil;
    return self[arc4random_uniform((uint32_t)[self count])];
}

@end
```

### การใช้งาน NSArray+Functional

```objc
#import "NSArray+Functional.h"

NSArray *numbers = @[@1, @5, @2, @8, @3, @9, @4, @7, @6];

// Map: คูณด้วย 2
NSArray *doubled = [numbers map:^id(NSNumber *n) {
    return @([n intValue] * 2);
}];
NSLog(@"Doubled: %@", doubled);
// [2, 10, 4, 16, 6, 18, 8, 14, 12]

// Filter: เฉพาะเลขคู่
NSArray *evens = [numbers filter:^BOOL(NSNumber *n) {
    return [n intValue] % 2 == 0;
}];
NSLog(@"Evens: %@", evens);
// [2, 8, 4, 6]

// Reduce: หาผลรวม
NSNumber *sum = [numbers reduce:@0 block:^id(NSNumber *acc, NSNumber *n) {
    return @([acc intValue] + [n intValue]);
}];
NSLog(@"Sum: %@", sum);
// 45

// Chain operations
NSNumber *sumOfSquaredEvens = [[[[numbers filter:^BOOL(NSNumber *n) {
    return [n intValue] % 2 == 0;
}] map:^id(NSNumber *n) {
    NSInteger v = [n intValue];
    return @(v * v);
}] sum];

NSLog(@"Sum of squares of evens: %@", sumOfSquaredEvens);

// GroupBy
NSArray *people = @[
    @{@"name": @"Alice", @"dept": @"Engineering"},
    @{@"name": @"Bob", @"dept": @"Marketing"},
    @{@"name": @"Charlie", @"dept": @"Engineering"},
    @{@"name": @"Diana", @"dept": @"Marketing"},
    @{@"name": @"Eve", @"dept": @"HR"},
];

NSDictionary *byDept = [people groupBy:^id(NSDictionary *p) {
    return p[@"dept"];
}];
NSLog(@"By department: %@", byDept);
```

---

## 16.5 Categories บน NSNumber

```objc
// ไฟล์: NSNumber+Formatting.h
@interface NSNumber (Formatting)

// Currency formatting
- (NSString *)currencyStringWithSymbol:(NSString *)symbol;
- (NSString *)thaiCurrencyString;
- (NSString *)usdCurrencyString;

// Percentage
- (NSString *)percentageString;
- (NSString *)percentageStringWithDecimals:(NSInteger)decimals;

// File size
- (NSString *)fileSizeString;

// Ordinal
- (NSString *)ordinalString;

// Number checks
- (BOOL)isPositive;
- (BOOL)isNegative;
- (BOOL)isZero;
- (BOOL)isEven;
- (BOOL)isOdd;
- (BOOL)isPrime;

// Math
- (NSNumber *)squared;
- (NSNumber *)cubed;
- (NSNumber *)absoluteValue;
- (NSNumber *)clampedToMin:(NSNumber *)min max:(NSNumber *)max;

@end

// ไฟล์: NSNumber+Formatting.m
#import <math.h>

@implementation NSNumber (Formatting)

- (NSString *)currencyStringWithSymbol:(NSString *)symbol {
    NSNumberFormatter *formatter = [[NSNumberFormatter alloc] init];
    formatter.numberStyle = NSNumberFormatterDecimalStyle;
    formatter.minimumFractionDigits = 2;
    formatter.maximumFractionDigits = 2;
    formatter.groupingSeparator = @",";
    formatter.usesGroupingSeparator = YES;
    
    return [NSString stringWithFormat:@"%@%@", symbol, [formatter stringFromNumber:self]];
}

- (NSString *)thaiCurrencyString {
    return [self currencyStringWithSymbol:@"฿"];
}

- (NSString *)usdCurrencyString {
    return [self currencyStringWithSymbol:@"$"];
}

- (NSString *)percentageString {
    return [NSString stringWithFormat:@"%.0f%%", [self doubleValue] * 100];
}

- (NSString *)percentageStringWithDecimals:(NSInteger)decimals {
    NSString *format = [NSString stringWithFormat:@"%%.%ldf%%%%", (long)decimals];
    return [NSString stringWithFormat:format, [self doubleValue] * 100];
}

- (NSString *)fileSizeString {
    double bytes = [self doubleValue];
    
    if (bytes < 1024) return [NSString stringWithFormat:@"%.0f B", bytes];
    if (bytes < 1024 * 1024) return [NSString stringWithFormat:@"%.1f KB", bytes / 1024];
    if (bytes < 1024 * 1024 * 1024) return [NSString stringWithFormat:@"%.1f MB", bytes / (1024 * 1024)];
    return [NSString stringWithFormat:@"%.2f GB", bytes / (1024 * 1024 * 1024)];
}

- (NSString *)ordinalString {
    NSInteger n = [self integerValue];
    NSInteger mod10 = n % 10;
    NSInteger mod100 = n % 100;
    
    if (mod10 == 1 && mod100 != 11) return [NSString stringWithFormat:@"%ldst", (long)n];
    if (mod10 == 2 && mod100 != 12) return [NSString stringWithFormat:@"%ldnd", (long)n];
    if (mod10 == 3 && mod100 != 13) return [NSString stringWithFormat:@"%ldrd", (long)n];
    return [NSString stringWithFormat:@"%ldth", (long)n];
}

- (BOOL)isPositive { return [self doubleValue] > 0; }
- (BOOL)isNegative { return [self doubleValue] < 0; }
- (BOOL)isZero { return [self doubleValue] == 0; }
- (BOOL)isEven { return [self integerValue] % 2 == 0; }
- (BOOL)isOdd { return [self integerValue] % 2 != 0; }

- (BOOL)isPrime {
    NSInteger n = [self integerValue];
    if (n < 2) return NO;
    if (n == 2) return YES;
    if (n % 2 == 0) return NO;
    
    for (NSInteger i = 3; i <= (NSInteger)sqrt((double)n); i += 2) {
        if (n % i == 0) return NO;
    }
    return YES;
}

- (NSNumber *)squared {
    double v = [self doubleValue];
    return @(v * v);
}

- (NSNumber *)cubed {
    double v = [self doubleValue];
    return @(v * v * v);
}

- (NSNumber *)absoluteValue {
    return @(fabs([self doubleValue]));
}

- (NSNumber *)clampedToMin:(NSNumber *)min max:(NSNumber *)max {
    double val = [self doubleValue];
    double minVal = [min doubleValue];
    double maxVal = [max doubleValue];
    return @(MAX(minVal, MIN(maxVal, val)));
}

@end
```

---

## 16.6 ไม่สามารถเพิ่ม Instance Variables ใน Categories

### ปัญหาและ Workaround

```objc
// ❌ ไม่สามารถทำได้!
@interface NSString (MyCategory)
{
    NSInteger _count; // ERROR: Categories cannot add instance variables
}
@end

// ✅ ใช้ Associated Objects แทน
#import <objc/runtime.h>

static const char kCountKey;

@interface NSString (MyCategory)
@property (nonatomic, assign) NSInteger count;
@end

@implementation NSString (MyCategory)

- (void)setCount:(NSInteger)count {
    objc_setAssociatedObject(self, 
                             &kCountKey, 
                             @(count), 
                             OBJC_ASSOCIATION_RETAIN_NONATOMIC);
}

- (NSInteger)count {
    NSNumber *value = objc_getAssociatedObject(self, &kCountKey);
    return [value integerValue];
}

@end
```

### ทำไมไม่สามารถเพิ่ม Instance Variables?

```
ปัญหา:
- Memory layout ของ class ถูกกำหนดตอน compile time
- Category load ที่ runtime
- การเพิ่ม ivar จะทำให้ memory layout เปลี่ยน
- Object ที่ถูกสร้างไปแล้วจะผิดพลาด

Solutions:
1. Associated Objects (objc_setAssociatedObject)
2. External dictionary (NSMapTable)
3. Subclassing (ถ้าต้องการ state จริงๆ)
```

---

## 16.7 Method Name Conflicts

### ปัญหา Method Overriding ใน Category

```objc
// ❌ อันตราย! Override method ของ NSString
@interface NSString (Dangerous)
- (NSString *)uppercaseString; // override! ไม่ดี
@end

@implementation NSString (Dangerous)
- (NSString *)uppercaseString {
    return [NSString stringWithFormat:@"[UPPER: %@]", self];
}
@end

// ผล: ทุกที่ที่เรียก uppercaseString จะได้ behavior ที่เปลี่ยนไป!
// "hello" uppercaseString -> "[UPPER: hello]" แทนที่จะเป็น "HELLO"
```

### Best Practices สำหรับ Category Names

```objc
// ✅ ใช้ prefix เพื่อหลีกเลี่ยง conflicts
@interface NSString (APP_Validation)
- (BOOL)app_isValidEmail;
- (BOOL)app_isValidPhone;
@end

@interface NSString (ABC_StringUtils)
- (NSString *)abc_trimWhitespace;
- (NSString *)abc_capitalizeFirstLetter;
@end

// ตัวอย่าง: ถ้า project ชื่อ MyApp
@interface UIColor (MA_AppColors)
+ (UIColor *)ma_primaryColor;
+ (UIColor *)ma_secondaryColor;
+ (UIColor *)ma_backgroundColor;
@end
```

### Category Loading Order

```
ถ้ามี category หลายตัว override method เดียวกัน:
- ลำดับ loading ไม่ถูก guarantee
- Behavior จะเป็น undefined
- ควรหลีกเลี่ยงการ override อย่างเด็ดขาด
```

---

## 16.8 Informal Protocols กับ Categories

### ก่อนจะมี @protocol

ก่อนที่ Objective-C จะมี formal protocols เราใช้ category เพื่อกำหนด informal protocol

```objc
// Informal Protocol ด้วย Category บน NSObject
@interface NSObject (DataSourceProtocol)
- (NSInteger)numberOfItems;
- (id)itemAtIndex:(NSInteger)index;
@end

// ไม่มี implementation ใน NSObject
// Class ที่ต้องการ implement ก็ implement methods เหล่านี้เอง
@interface MyDataSource : NSObject

@property (nonatomic, strong) NSArray *items;

@end

@implementation MyDataSource

- (NSInteger)numberOfItems {
    return [_items count];
}

- (id)itemAtIndex:(NSInteger)index {
    return _items[index];
}

@end
```

### ปัจจุบัน: ใช้ Formal Protocols แทน

```objc
// ✅ Modern approach - ใช้ formal protocol
@protocol DataSource <NSObject>
@required
- (NSInteger)numberOfItems;
- (id)itemAtIndex:(NSInteger)index;
@end

// Informal protocol ยังคงมีใน legacy code
// แต่ไม่แนะนำสำหรับ code ใหม่
```

---

## 16.9 +load และ +initialize

### +load Method

`+load` ถูกเรียกครั้งเดียวเมื่อ class ถูก load เข้า runtime (ก่อน `main()`)

```objc
@interface NSString (Swizzling)
@end

@implementation NSString (Swizzling)

+ (void)load {
    NSLog(@"NSString+Swizzling category loaded!");
    // ทำ method swizzling หรือ setup อื่นๆ ที่นี่
    
    // ตัวอย่าง: Method Swizzling
    // สลับ implementation ของสอง methods
    Method original = class_getInstanceMethod([NSString class], @selector(description));
    Method swizzled = class_getInstanceMethod([NSString class], @selector(my_description));
    method_exchangeImplementations(original, swizzled);
}

- (NSString *)my_description {
    return [NSString stringWithFormat:@"[Custom: %@]", [self my_description]];
}

@end
```

### +initialize Method

`+initialize` ถูกเรียกครั้งแรกที่ class ถูกใช้งาน

```objc
@interface MyManager : NSObject
@end

@implementation MyManager

+ (void)initialize {
    if (self == [MyManager class]) {
        // ทำ one-time initialization
        NSLog(@"MyManager initialized");
    }
}

@end

// Category ยังสามารถ implement +initialize ได้
@interface MyManager (Setup)
@end

@implementation MyManager (Setup)

+ (void)initialize {
    if (self == [MyManager class]) {
        // WARNING: ถ้ามีหลาย category implement +initialize
        // ทุกตัวจะถูกเรียก (ลำดับไม่แน่นอน)
        NSLog(@"MyManager+Setup initialize");
    }
}

@end
```

---

## 16.10 ตัวอย่างปฏิบัติ: UIColor+AppColors

```objc
// ไฟล์: UIColor+AppColors.h
// (ใช้ NSColor บน macOS แทน UIColor)

// สร้าง Color system สำหรับ app

// Design system color tokens
typedef NS_ENUM(NSInteger, AppColorScheme) {
    AppColorSchemeLight,
    AppColorSchemeDark
};

@interface UIColor (AppColors)

// Primary colors
+ (UIColor *)primaryColor;
+ (UIColor *)primaryLightColor;
+ (UIColor *)primaryDarkColor;

// Secondary colors
+ (UIColor *)secondaryColor;
+ (UIColor *)accentColor;

// Status colors
+ (UIColor *)successColor;
+ (UIColor *)warningColor;
+ (UIColor *)errorColor;
+ (UIColor *)infoColor;

// Neutral colors
+ (UIColor *)backgroundColor;
+ (UIColor *)surfaceColor;
+ (UIColor *)textPrimaryColor;
+ (UIColor *)textSecondaryColor;
+ (UIColor *)dividerColor;

// Convenience constructors
+ (UIColor *)colorWithHex:(NSString *)hexString;
+ (UIColor *)colorWithHex:(NSString *)hexString alpha:(CGFloat)alpha;
+ (UIColor *)colorWithRed255:(NSInteger)red 
                      green:(NSInteger)green 
                       blue:(NSInteger)blue;

// Utility
- (NSString *)hexString;
- (BOOL)isLight;
- (UIColor *)lighterColor;
- (UIColor *)darkerColor;
- (UIColor *)colorWithOpacity:(CGFloat)opacity;

@end
```

```objc
// ไฟล์: UIColor+AppColors.m
@implementation UIColor (AppColors)

+ (UIColor *)primaryColor {
    return [UIColor colorWithHex:@"#2196F3"]; // Material Blue
}

+ (UIColor *)primaryLightColor {
    return [UIColor colorWithHex:@"#64B5F6"];
}

+ (UIColor *)primaryDarkColor {
    return [UIColor colorWithHex:@"#1565C0"];
}

+ (UIColor *)secondaryColor {
    return [UIColor colorWithHex:@"#FF5722"]; // Material Deep Orange
}

+ (UIColor *)accentColor {
    return [UIColor colorWithHex:@"#FF9800"]; // Material Orange
}

+ (UIColor *)successColor {
    return [UIColor colorWithHex:@"#4CAF50"]; // Material Green
}

+ (UIColor *)warningColor {
    return [UIColor colorWithHex:@"#FFC107"]; // Material Amber
}

+ (UIColor *)errorColor {
    return [UIColor colorWithHex:@"#F44336"]; // Material Red
}

+ (UIColor *)infoColor {
    return [UIColor colorWithHex:@"#2196F3"]; // Material Blue
}

+ (UIColor *)backgroundColor {
    return [UIColor colorWithHex:@"#F5F5F5"];
}

+ (UIColor *)surfaceColor {
    return [UIColor whiteColor];
}

+ (UIColor *)textPrimaryColor {
    return [UIColor colorWithRed:0 green:0 blue:0 alpha:0.87];
}

+ (UIColor *)textSecondaryColor {
    return [UIColor colorWithRed:0 green:0 blue:0 alpha:0.60];
}

+ (UIColor *)dividerColor {
    return [UIColor colorWithRed:0 green:0 blue:0 alpha:0.12];
}

+ (UIColor *)colorWithHex:(NSString *)hexString {
    return [UIColor colorWithHex:hexString alpha:1.0];
}

+ (UIColor *)colorWithHex:(NSString *)hexString alpha:(CGFloat)alpha {
    NSString *hex = hexString;
    
    // ลบ # ออก
    if ([hex hasPrefix:@"#"]) {
        hex = [hex substringFromIndex:1];
    }
    
    // จัดการ 3 หลัก (#RGB)
    if ([hex length] == 3) {
        NSString *r = [hex substringWithRange:NSMakeRange(0, 1)];
        NSString *g = [hex substringWithRange:NSMakeRange(1, 1)];
        NSString *b = [hex substringWithRange:NSMakeRange(2, 1)];
        hex = [NSString stringWithFormat:@"%@%@%@%@%@%@", r, r, g, g, b, b];
    }
    
    unsigned int rgbValue = 0;
    NSScanner *scanner = [NSScanner scannerWithString:hex];
    [scanner scanHexInt:&rgbValue];
    
    return [UIColor colorWithRed:((rgbValue & 0xFF0000) >> 16) / 255.0
                           green:((rgbValue & 0x00FF00) >> 8) / 255.0
                            blue:(rgbValue & 0x0000FF) / 255.0
                           alpha:alpha];
}

+ (UIColor *)colorWithRed255:(NSInteger)red 
                      green:(NSInteger)green 
                       blue:(NSInteger)blue {
    return [UIColor colorWithRed:red/255.0 
                           green:green/255.0 
                            blue:blue/255.0 
                           alpha:1.0];
}

- (NSString *)hexString {
    CGFloat r, g, b, a;
    [self getRed:&r green:&g blue:&b alpha:&a];
    
    return [NSString stringWithFormat:@"#%02lX%02lX%02lX",
            (long)(r * 255),
            (long)(g * 255),
            (long)(b * 255)];
}

- (BOOL)isLight {
    CGFloat r, g, b, a;
    [self getRed:&r green:&g blue:&b alpha:&a];
    
    // Luminance formula
    double luminance = 0.2126 * r + 0.7152 * g + 0.0722 * b;
    return luminance > 0.5;
}

- (UIColor *)lighterColor {
    CGFloat h, s, v, a;
    [self getHue:&h saturation:&s brightness:&v alpha:&a];
    return [UIColor colorWithHue:h saturation:s brightness:MIN(v * 1.3, 1.0) alpha:a];
}

- (UIColor *)darkerColor {
    CGFloat h, s, v, a;
    [self getHue:&h saturation:&s brightness:&v alpha:&a];
    return [UIColor colorWithHue:h saturation:s brightness:v * 0.7 alpha:a];
}

- (UIColor *)colorWithOpacity:(CGFloat)opacity {
    CGFloat r, g, b, a;
    [self getRed:&r green:&g blue:&b alpha:&a];
    return [UIColor colorWithRed:r green:g blue:b alpha:opacity];
}

@end
```

---

## 16.11 Organization Pattern: Splitting Large Classes

```objc
// แบ่ง ViewController ขนาดใหญ่ออกเป็นหลาย categories

// ViewController.h
@interface ViewController : UIViewController
@property (nonatomic, strong) NSArray *data;
- (void)loadData;
@end

// ViewController+TableView.h
@interface ViewController (TableView) <UITableViewDataSource, UITableViewDelegate>
- (void)setupTableView;
@end

// ViewController+Navigation.h
@interface ViewController (Navigation)
- (void)navigateToDetail:(id)item;
- (void)navigateBack;
@end

// ViewController+DataLoading.h
@interface ViewController (DataLoading)
- (void)fetchDataFromAPI;
- (void)processResponse:(NSDictionary *)response;
@end

// ViewController+Analytics.h
@interface ViewController (Analytics)
- (void)trackScreenView;
- (void)trackUserAction:(NSString *)action;
@end
```

---

## แบบฝึกหัด

### ระดับเริ่มต้น

**แบบฝึกหัดที่ 1**: NSString+Thai
```objc
// สร้าง category สำหรับจัดการข้อความภาษาไทย:
// - (NSString *)removeThaiToneMarks;
// - (NSInteger)thaiWordCount;
// - (BOOL)isThaiFormalAddress; // ตรวจสอบว่าใช้คำเป็นทางการ
// - (NSString *)convertThaiNumbersToArabic; // ๑๒๓ -> 123
```

**แบบฝึกหัดที่ 2**: NSDate+Formatting
```objc
// สร้าง NSDate category:
// - (NSString *)thaiDateString; // วันที่แบบไทย
// - (NSString *)relativeTimeString; // "2 ชั่วโมงที่แล้ว"
// - (BOOL)isToday;
// - (BOOL)isYesterday;
// - (BOOL)isThisWeek;
// - (NSDate *)startOfDay;
// - (NSDate *)endOfDay;
// - (NSInteger)daysBetween:(NSDate *)otherDate;
```

**แบบฝึกหัดที่ 3**: NSNumber+Thai
```objc
// สร้าง NSNumber category:
// - (NSString *)thaiNumberString; // 123 -> "หนึ่งร้อยยี่สิบสาม"
// - (NSString *)thaiCurrencyString; // 1234.50 -> "หนึ่งพันสองร้อยสามสิบสี่บาทห้าสิบสตางค์"
```

### ระดับกลาง

**แบบฝึกหัดที่ 4**: NSDictionary+Utilities
```objc
// สร้าง NSDictionary category:
// - (id)safeValueForKey:(NSString *)key;
// - (NSString *)safeStringForKey:(NSString *)key defaultValue:(NSString *)def;
// - (NSInteger)safeIntegerForKey:(NSString *)key defaultValue:(NSInteger)def;
// - (NSDictionary *)dictionaryByMerging:(NSDictionary *)other;
// - (NSDictionary *)dictionaryWithKeys:(NSArray *)keys;
// - (NSDictionary *)dictionaryExcludingKeys:(NSArray *)keys;
```

**แบบฝึกหัดที่ 5**: NSArray+Statistics
```objc
// NSArray category สำหรับ statistics (สำหรับ array ของ NSNumber):
// - (NSNumber *)median;
// - (NSNumber *)standardDeviation;
// - (NSNumber *)variance;
// - (NSArray *)topN:(NSInteger)n;
// - (NSArray *)bottomN:(NSInteger)n;
// - (NSDictionary *)frequencyDistribution;
```

**แบบฝึกหัดที่ 6**: UIView+Layout (หรือ NSView บน macOS)
```objc
// Category สำหรับ layout helper:
// - (void)centerInParent;
// - (void)pinToEdges;
// - (void)roundCorners:(CGFloat)radius;
// - (void)addShadow;
// - (void)shake; // animation
// - (CGPoint)center;
```

### ระดับสูง

**แบบฝึกหัดที่ 7**: NSObject+Introspection
```objc
// สร้าง NSObject category ที่ใช้ Objective-C runtime:
// - (NSArray *)allPropertyNames;
// - (NSArray *)allMethodNames;
// - (NSDictionary *)dictionaryRepresentation;
// + (instancetype)fromDictionary:(NSDictionary *)dict;
// - (id)deepCopy;
```

**แบบฝึกหัดที่ 8**: Splitting a Large Class
```objc
// สร้าง BankAccount class และแบ่งออกเป็น categories:
// BankAccount+Transactions (deposit, withdraw, transfer)
// BankAccount+Reporting (statement, summary)
// BankAccount+Validation (validateAmount, checkBalance)
// BankAccount+Notifications (notify on low balance, etc.)
```

**แบบฝึกหัดที่ 9**: NSString+Cryptography
```objc
// NSString category สำหรับ basic encoding:
// - (NSString *)base64Encoded;
// - (NSString *)base64Decoded;
// - (NSString *)md5Hash;
// - (NSString *)sha256Hash;
// - (NSString *)urlEncoded;
// - (NSString *)urlDecoded;
```

**แบบฝึกหัดที่ 10**: NSArray+Diff
```objc
// สร้าง NSArray category สำหรับ diff operations:
// - (NSDictionary *)diffFromArray:(NSArray *)other; // returns added/removed/unchanged
// - (NSArray *)intersection:(NSArray *)other;
// - (NSArray *)union:(NSArray *)other;
// - (NSArray *)subtraction:(NSArray *)other;
```

---

## สรุปบทเรียน

ในบทนี้เราได้เรียนรู้:
- **Categories คืออะไร**: วิธีเพิ่ม method ให้ existing class
- **Category Syntax**: `@interface ClassName (CategoryName)`
- **Categories บน System Classes**: NSString, NSArray, NSNumber
- **ข้อจำกัด**: ไม่สามารถเพิ่ม instance variables
- **Method Conflicts**: ปัญหาและวิธีหลีกเลี่ยง
- **Naming Convention**: ใช้ prefix เพื่อหลีกเลี่ยง collision
- **+load และ +initialize**: การทำงานของ class loading
- **Practical Examples**: UIColor+AppColors, NSArray+Functional
- **Code Organization**: แบ่ง large class ด้วย categories

> **Best Practice**: ตั้งชื่อ category method ด้วย prefix เสมอ เช่น `app_`, `ma_`, เพื่อหลีกเลี่ยงการ clash กับ Apple API หรือ third-party libraries
