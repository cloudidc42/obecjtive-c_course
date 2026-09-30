# Part 02: Syntax, Variables และ Data Types ใน Objective-C

---

## สารบัญ (Table of Contents)

1. ประเภทข้อมูลพื้นฐาน (Primitive Data Types)
2. การประกาศตัวแปร (Variable Declaration)
3. NSString
4. NSInteger, NSUInteger, CGFloat
5. Constants (ค่าคงที่)
6. Type Casting (การแปลงประเภทข้อมูล)
7. nil vs NULL vs Nil
8. BOOL (YES/NO) vs bool (true/false)
9. NSLog Format Specifiers
10. sizeof Operator
11. ตัวอย่างโค้ดสมบูรณ์
12. แบบฝึกหัดพร้อมเฉลย

---

## 1. ประเภทข้อมูลพื้นฐาน (Primitive Data Types)

Objective-C สืบทอดประเภทข้อมูลทั้งหมดจาก C และเพิ่ม Objective-C types ใหม่

### Integer Types

```objc
#import <Foundation/Foundation.h>

int main(int argc, const char * argv[]) {
    @autoreleasepool {
        
        // ========== Integer Types ==========
        
        // char - 1 byte, -128 ถึง 127 (signed) หรือ 0 ถึง 255 (unsigned)
        char ch = 'A';                    // ตัวอักษร ASCII
        char num = 65;                    // ค่าตัวเลข (เท่ากับ 'A')
        signed char sCh = -128;           // signed char
        unsigned char uCh = 255;          // unsigned char
        
        NSLog(@"char: %c (decimal: %d)", ch, ch);
        NSLog(@"char from number: %c", num);
        NSLog(@"signed char min: %d", sCh);
        NSLog(@"unsigned char max: %d", uCh);
        
        // short - 2 bytes, -32,768 ถึง 32,767
        short s = 1000;
        short minShort = -32768;
        unsigned short maxUShort = 65535;
        
        NSLog(@"short: %d", s);
        NSLog(@"short min: %d", minShort);
        NSLog(@"unsigned short max: %d", maxUShort);
        
        // int - 4 bytes บน 32/64-bit, -2,147,483,648 ถึง 2,147,483,647
        int i = 42;
        int negative = -100;
        unsigned int uInt = 4294967295U;
        
        NSLog(@"int: %d", i);
        NSLog(@"negative int: %d", negative);
        NSLog(@"unsigned int max: %u", uInt);
        
        // long - 4 bytes (32-bit) หรือ 8 bytes (64-bit)
        long l = 1234567890L;
        unsigned long ul = 4294967295UL;
        
        NSLog(@"long: %ld", l);
        NSLog(@"unsigned long: %lu", ul);
        
        // long long - 8 bytes, -9,223,372,036,854,775,808 ถึง 9,223,372,036,854,775,807
        long long ll = 9223372036854775807LL;
        unsigned long long ull = 18446744073709551615ULL;
        
        NSLog(@"long long max: %lld", ll);
        NSLog(@"unsigned long long max: %llu", ull);
        
    }
    return 0;
}
```

### Floating-Point Types

```objc
#import <Foundation/Foundation.h>
#include <float.h>  // สำหรับ FLT_MAX, DBL_MAX

int main(int argc, const char * argv[]) {
    @autoreleasepool {
        
        // float - 4 bytes, ~7 decimal digits precision
        float f = 3.14f;                // ต้องมี 'f' suffix
        float fMax = FLT_MAX;           // ค่าสูงสุดของ float
        float fMin = FLT_MIN;           // ค่าต่ำสุดของ float (positive)
        
        NSLog(@"float: %f", f);
        NSLog(@"float (2 decimals): %.2f", f);
        NSLog(@"float max: %e", fMax);
        NSLog(@"float min positive: %e", fMin);
        
        // double - 8 bytes, ~15-16 decimal digits precision
        double d = 3.14159265358979323846;
        double dMax = DBL_MAX;
        
        NSLog(@"double: %.20f", d);     // แสดง 20 decimal places
        NSLog(@"double max: %e", dMax);
        
        // long double - 10-16 bytes (depends on platform)
        long double ld = 3.14159265358979323846L;
        NSLog(@"long double: %.20Lf", ld);
        
        // Arithmetic ระหว่าง float กับ int
        int a = 10;
        int b = 3;
        float result = (float)a / b;    // ต้อง cast เพื่อให้ได้ float result
        
        NSLog(@"10 / 3 = %d (integer division)", a / b);  // 3
        NSLog(@"10 / 3 = %.4f (float division)", result);  // 3.3333
        
        // Special values
        double posInf = 1.0 / 0.0;     // Positive infinity
        double negInf = -1.0 / 0.0;    // Negative infinity
        double notANum = 0.0 / 0.0;    // NaN (Not a Number)
        
        NSLog(@"Positive infinity: %f", posInf);
        NSLog(@"Negative infinity: %f", negInf);
        NSLog(@"NaN: %f", notANum);
        
        // ตรวจสอบ special values
        if (isinf(posInf)) NSLog(@"posInf is infinite");
        if (isnan(notANum)) NSLog(@"notANum is NaN");
        if (isfinite(d)) NSLog(@"d is finite");
        
    }
    return 0;
}
```

### Boolean Type

```objc
#import <Foundation/Foundation.h>

int main(int argc, const char * argv[]) {
    @autoreleasepool {
        
        // BOOL (Objective-C) - ใช้ YES/NO
        BOOL isTrue = YES;
        BOOL isFalse = NO;
        BOOL isEqual = (10 == 10);      // YES
        BOOL isGreater = (5 > 10);      // NO
        
        NSLog(@"BOOL YES: %d", isTrue);   // แสดงเป็น 1
        NSLog(@"BOOL NO: %d", isFalse);   // แสดงเป็น 0
        NSLog(@"BOOL as string: %@", isTrue ? @"YES" : @"NO");
        
        // bool (C99) - ใช้ true/false
        // ต้อง #include <stdbool.h> หรือ #import <Foundation/Foundation.h>
        bool cBool = true;
        bool cFalse = false;
        
        NSLog(@"bool true: %d", cBool);
        NSLog(@"bool false: %d", cFalse);
        
        // ความแตกต่าง: BOOL เป็น typedef ของ signed char
        // bool เป็น C99 standard boolean type
        
        // BOOL กับค่า non-zero
        BOOL nonZero = 256;  // CAUTION! overflow ทำให้ได้ 0 (NO)!
        NSLog(@"BOOL 256: %d", nonZero);  // อาจได้ 0 หรือ 1 ขึ้นกับ platform
        
        // วิธีที่ถูกต้อง: ใช้ !! (double negation) เพื่อ normalize
        BOOL safe = !!(256);    // 1 เสมอ
        NSLog(@"Safe BOOL: %d", safe);
        
    }
    return 0;
}
```

### void และ id Types

```objc
#import <Foundation/Foundation.h>

// void - ไม่มีค่า, ใช้สำหรับ functions/methods ที่ไม่คืนค่า
void printMessage(NSString *message) {
    NSLog(@"%@", message);
    // ไม่มี return statement (หรือ return; แบบไม่มีค่า)
}

// id - pointer ที่ชี้ไปยัง Objective-C object ใดก็ได้
// เหมือน void* แต่สำหรับ ObjC objects
void printAnything(id object) {
    NSLog(@"Object: %@", object);
}

int main(int argc, const char * argv[]) {
    @autoreleasepool {
        
        // void function
        printMessage(@"Hello from void function!");
        
        // id type - can hold any ObjC object
        id str = @"I am a string";
        id num = @42;
        id arr = @[@"a", @"b", @"c"];
        
        printAnything(str);
        printAnything(num);
        printAnything(arr);
        
        // ตรวจสอบประเภท
        if ([str isKindOfClass:[NSString class]]) {
            NSLog(@"str is NSString");
        }
        
        if ([num isKindOfClass:[NSNumber class]]) {
            NSLog(@"num is NSNumber");
        }
        
        // id ไม่ต้องใช้ * ต่อท้าย (เพราะ id เป็น pointer อยู่แล้ว)
        // id obj = ...; (ถูกต้อง)
        // id *obj = ...; (ผิด สำหรับ normal usage)
        
    }
    return 0;
}
```

---

## 2. การประกาศตัวแปร (Variable Declaration)

### Syntax การประกาศตัวแปร

```objc
// รูปแบบทั่วไป:
// type variableName;                    // ประกาศ (ไม่กำหนดค่า)
// type variableName = initialValue;     // ประกาศพร้อมกำหนดค่า
// type var1, var2, var3;                // ประกาศหลายตัวพร้อมกัน
// type var1 = v1, var2 = v2;            // ประกาศหลายตัวพร้อมกำหนดค่า

#import <Foundation/Foundation.h>

int main(int argc, const char * argv[]) {
    @autoreleasepool {
        
        // ========== การประกาศตัวแปรแบบต่างๆ ==========
        
        // ประกาศโดยไม่กำหนดค่า (uninitialized - ระวัง! ค่าอาจไม่แน่นอน)
        int uninitalized;
        // NSLog(@"%d", uninitialized); // ค่าอาจเป็นอะไรก็ได้ หลีกเลี่ยงการใช้งาน
        
        // ประกาศพร้อมกำหนดค่า (แนะนำ)
        int count = 0;
        double price = 99.99;
        NSString *name = @"สมชาย";
        BOOL isActive = YES;
        
        // ประกาศหลายตัวในบรรทัดเดียว (ไม่แนะนำ อ่านยาก)
        int x = 10, y = 20, z = 30;
        NSLog(@"x=%d, y=%d, z=%d", x, y, z);
        
        // ========== Variable Scope ==========
        
        {
            // inner block
            int innerVar = 100;
            NSLog(@"innerVar: %d", innerVar);  // OK
        }
        // NSLog(@"innerVar: %d", innerVar);  // ERROR! innerVar ไม่มีอยู่แล้ว
        
        // ========== Local Variables ==========
        // ตัวแปรที่ประกาศในฟังก์ชัน/method - มีอยู่เฉพาะใน scope นั้น
        int localVar = 42;
        NSLog(@"Local var: %d", localVar);
        
        // ========== Static Variables ==========
        // มี scope ในฟังก์ชัน แต่ค่าคงอยู่ระหว่าง function calls
        static int callCount = 0;
        callCount++;
        NSLog(@"Call count: %d", callCount);
        
        // ========== Naming Conventions ==========
        
        // camelCase สำหรับ variables (แนะนำ)
        int studentCount = 50;
        NSString *firstName = @"สมชาย";
        BOOL isLoggedIn = NO;
        
        // PascalCase สำหรับ classes
        // NSString, NSArray, UIViewController
        
        // UPPER_CASE สำหรับ constants
        // MAX_COUNT, API_KEY
        
        // ขึ้นต้นด้วย _ สำหรับ private instance variables (convention)
        // _myPrivateVar
        
        // ขึ้นต้นด้วย k สำหรับ constants (อีก convention หนึ่ง)
        // kMaxRetryCount
        
        NSLog(@"Student count: %d, Name: %@, Logged in: %@", 
              studentCount, firstName, isLoggedIn ? @"YES" : @"NO");
        
        // ========== Type Inference (auto ใน C++11) ==========
        // Objective-C ไม่มี auto keyword แบบ C++
        // แต่ใช้ __auto_type หรือ typeof ได้
        
        __auto_type inferredInt = 42;          // infer เป็น int
        __auto_type inferredStr = @"Hello";    // infer เป็น NSString *
        
        NSLog(@"Inferred int: %d", inferredInt);
        NSLog(@"Inferred str: %@", inferredStr);
        
    }
    return 0;
}
```

### Instance Variables vs Properties

```objc
#import <Foundation/Foundation.h>

@interface BankAccount : NSObject {
    // Instance variables (ivar) - ประกาศใน @interface {} block
    // เข้าถึงได้ใน implementation โดยตรง
    double _balance;          // convention: prefix _
    NSString *_ownerName;
    NSInteger _transactionCount;
}

// Properties - สร้าง getter/setter อัตโนมัติ
@property (nonatomic, strong) NSString *accountNumber;
@property (nonatomic, assign) double interestRate;
@property (nonatomic, readonly) NSDate *createdDate;  // readonly - ไม่มี setter

// Methods
- (instancetype)initWithOwner:(NSString *)owner balance:(double)balance;
- (void)deposit:(double)amount;
- (BOOL)withdraw:(double)amount;
- (void)printStatement;
@end

@implementation BankAccount

// @synthesize - สร้าง getter/setter (ใน modern ObjC ไม่จำเป็นต้องเขียน)
// @synthesize accountNumber = _accountNumber;  // auto-synthesized

- (instancetype)initWithOwner:(NSString *)owner balance:(double)balance {
    self = [super init];
    if (self) {
        // Set instance variables โดยตรง ไม่ใช้ self. ใน init
        _ownerName = owner;
        _balance = balance;
        _transactionCount = 0;
        _accountNumber = [NSString stringWithFormat:@"ACC%06ld", (long)arc4random_uniform(999999)];
        _createdDate = [NSDate date];
        _interestRate = 0.025; // 2.5%
    }
    return self;
}

- (void)deposit:(double)amount {
    if (amount > 0) {
        _balance += amount;
        _transactionCount++;
        NSLog(@"ฝาก %.2f บาท, ยอดคงเหลือ: %.2f บาท", amount, _balance);
    } else {
        NSLog(@"จำนวนเงินต้องมากกว่า 0");
    }
}

- (BOOL)withdraw:(double)amount {
    if (amount > 0 && amount <= _balance) {
        _balance -= amount;
        _transactionCount++;
        NSLog(@"ถอน %.2f บาท, ยอดคงเหลือ: %.2f บาท", amount, _balance);
        return YES;
    } else {
        NSLog(@"ไม่สามารถถอนได้: ยอดไม่เพียงพอหรือจำนวนไม่ถูกต้อง");
        return NO;
    }
}

- (void)printStatement {
    NSLog(@"===== ใบแจ้งยอดบัญชี =====");
    NSLog(@"หมายเลขบัญชี: %@", self.accountNumber);
    NSLog(@"เจ้าของบัญชี: %@", _ownerName);
    NSLog(@"ยอดคงเหลือ: %.2f บาท", _balance);
    NSLog(@"อัตราดอกเบี้ย: %.1f%%", self.interestRate * 100);
    NSLog(@"จำนวนรายการ: %ld", (long)_transactionCount);
    NSLog(@"========================");
}

// Getter สำหรับ balance (readonly custom getter)
- (double)balance {
    return _balance;
}

@end

int main(int argc, const char * argv[]) {
    @autoreleasepool {
        BankAccount *account = [[BankAccount alloc] initWithOwner:@"สมชาย ใจดี" 
                                                          balance:1000.0];
        [account printStatement];
        
        [account deposit:500.0];
        [account withdraw:200.0];
        [account withdraw:2000.0]; // จะ fail
        
        NSLog(@"");
        [account printStatement];
    }
    return 0;
}
```

---

## 3. NSString

NSString เป็นหนึ่งใน classes ที่ใช้บ่อยที่สุดใน Objective-C

### การสร้าง NSString

```objc
#import <Foundation/Foundation.h>

int main(int argc, const char * argv[]) {
    @autoreleasepool {
        
        // ========== วิธีสร้าง NSString ==========
        
        // 1. String literal (แนะนำ)
        NSString *s1 = @"Hello, World!";
        
        // 2. stringWithFormat:
        NSString *s2 = [NSString stringWithFormat:@"ยินดีต้อนรับ %@!", @"สมชาย"];
        
        // 3. initWithString:
        NSString *s3 = [[NSString alloc] initWithString:@"Allocated"];
        
        // 4. จาก C string
        const char *cStr = "C String";
        NSString *s4 = [NSString stringWithUTF8String:cStr];
        NSString *s5 = [[NSString alloc] initWithCString:cStr encoding:NSUTF8StringEncoding];
        
        // 5. จาก NSData
        NSData *data = [@"Hello" dataUsingEncoding:NSUTF8StringEncoding];
        NSString *s6 = [[NSString alloc] initWithData:data encoding:NSUTF8StringEncoding];
        
        // 6. จาก format (เหมือน sprintf)
        int age = 25;
        double height = 175.5;
        NSString *profile = [NSString stringWithFormat:@"อายุ %d ปี สูง %.1f ซม.", age, height];
        
        NSLog(@"s1: %@", s1);
        NSLog(@"s2: %@", s2);
        NSLog(@"s3: %@", s3);
        NSLog(@"s4: %@", s4);
        NSLog(@"s6: %@", s6);
        NSLog(@"profile: %@", profile);
        
        // ========== NSString Methods ==========
        
        NSString *test = @"  Hello, Objective-C World!  ";
        
        // ความยาว
        NSLog(@"Length: %lu", (unsigned long)[test length]);
        
        // Uppercase/Lowercase
        NSLog(@"Upper: %@", [test uppercaseString]);
        NSLog(@"Lower: %@", [test lowercaseString]);
        NSLog(@"Capitalized: %@", [test capitalizedString]);
        
        // Trim whitespace
        NSString *trimmed = [test stringByTrimmingCharactersInSet:
                             [NSCharacterSet whitespaceAndNewlineCharacterSet]];
        NSLog(@"Trimmed: '%@'", trimmed);
        
        // Contains/HasPrefix/HasSuffix
        NSLog(@"Contains 'World': %@", [test containsString:@"World"] ? @"YES" : @"NO");
        NSLog(@"HasPrefix '  Hello': %@", [test hasPrefix:@"  Hello"] ? @"YES" : @"NO");
        NSLog(@"HasSuffix 'World!  ': %@", [test hasSuffix:@"World!  "] ? @"YES" : @"NO");
        
        // Range ของ substring
        NSRange range = [test rangeOfString:@"Objective-C"];
        if (range.location != NSNotFound) {
            NSLog(@"Found 'Objective-C' at location: %lu, length: %lu",
                  (unsigned long)range.location, (unsigned long)range.length);
        }
        
        // Substring
        NSString *sub1 = [trimmed substringFromIndex:7];   // จาก index 7 ถึงจบ
        NSString *sub2 = [trimmed substringToIndex:5];     // จากต้นถึง index 5
        NSRange midRange = NSMakeRange(7, 11);              // location, length
        NSString *sub3 = [trimmed substringWithRange:midRange];
        
        NSLog(@"substringFromIndex(7): %@", sub1);
        NSLog(@"substringToIndex(5): %@", sub2);
        NSLog(@"substringWithRange(7,11): %@", sub3);
        
        // Replace
        NSString *replaced = [trimmed stringByReplacingOccurrencesOfString:@"World" 
                                                                withString:@"ไทย"];
        NSLog(@"Replaced: %@", replaced);
        
        // Split (componentsSeparatedByString)
        NSString *csv = @"apple,banana,cherry,date";
        NSArray *parts = [csv componentsSeparatedByString:@","];
        NSLog(@"Parts: %@", parts);
        
        // Join (componentsJoinedByString)
        NSArray *words = @[@"Hello", @"World", @"from", @"ObjC"];
        NSString *joined = [words componentsJoinedByString:@" "];
        NSLog(@"Joined: %@", joined);
        
        // String to Number
        NSLog(@"integerValue: %ld", (long)[@"42" integerValue]);
        NSLog(@"doubleValue: %.2f", [@"3.14" doubleValue]);
        NSLog(@"floatValue: %.2f", [@"3.14" floatValue]);
        NSLog(@"boolValue: %d", [@"YES" boolValue]);
        
        // Compare
        NSString *a = @"Apple";
        NSString *b = @"banana";
        NSComparisonResult result = [a compare:b options:NSCaseInsensitiveSearch];
        if (result == NSOrderedAscending) {
            NSLog(@"'%@' comes before '%@'", a, b);
        } else if (result == NSOrderedDescending) {
            NSLog(@"'%@' comes after '%@'", a, b);
        } else {
            NSLog(@"'%@' equals '%@'", a, b);
        }
        
        // isEqualToString - เปรียบเทียบเนื้อหา (ไม่ใช่ pointer)
        NSString *str1 = @"Hello";
        NSString *str2 = [NSString stringWithFormat:@"%@", @"Hello"];
        NSLog(@"== comparison: %@", (str1 == str2) ? @"same pointer" : @"different pointer");
        NSLog(@"isEqualToString: %@", [str1 isEqualToString:str2] ? @"equal" : @"not equal");
        
    }
    return 0;
}
```

### NSMutableString

```objc
#import <Foundation/Foundation.h>

int main(int argc, const char * argv[]) {
    @autoreleasepool {
        
        // NSMutableString - แก้ไขเนื้อหาได้
        NSMutableString *mStr = [NSMutableString stringWithString:@"Hello"];
        
        // Append
        [mStr appendString:@", World!"];
        NSLog(@"After append: %@", mStr);
        
        // Insert
        [mStr insertString:@"Beautiful " atIndex:7];
        NSLog(@"After insert: %@", mStr);
        
        // Delete range
        NSRange deleteRange = [mStr rangeOfString:@"Beautiful "];
        [mStr deleteCharactersInRange:deleteRange];
        NSLog(@"After delete: %@", mStr);
        
        // Replace
        [mStr replaceOccurrencesOfString:@"World" 
                              withString:@"Objective-C" 
                                 options:0 
                                   range:NSMakeRange(0, mStr.length)];
        NSLog(@"After replace: %@", mStr);
        
        // Append format
        [mStr appendFormat:@" (version %d)", 2];
        NSLog(@"After appendFormat: %@", mStr);
        
        // setString - แทนที่เนื้อหาทั้งหมด
        [mStr setString:@"Fresh start"];
        NSLog(@"After setString: %@", mStr);
        
        // สร้างจาก NSString ที่ immutable
        NSString *immutable = @"Can't change me";
        NSMutableString *mutable = [immutable mutableCopy];
        [mutable appendString:@" ...or can you?"];
        NSLog(@"Original: %@", immutable);  // ยังเหมือนเดิม
        NSLog(@"Mutable copy: %@", mutable);
        
    }
    return 0;
}
```

---

## 4. NSInteger, NSUInteger, CGFloat

เหล่านี้คือ type aliases ที่ Foundation และ CoreGraphics กำหนด

### NSInteger และ NSUInteger

```objc
#import <Foundation/Foundation.h>

int main(int argc, const char * argv[]) {
    @autoreleasepool {
        
        // NSInteger - signed integer ขนาดเท่ากับ pointer ของ platform
        // บน 32-bit: เท่ากับ int (4 bytes)
        // บน 64-bit: เท่ากับ long (8 bytes)
        NSInteger count = 100;
        NSInteger negative = -50;
        NSInteger maxNSInteger = NSIntegerMax;  // INT_MAX หรือ LONG_MAX
        NSInteger minNSInteger = NSIntegerMin;  // INT_MIN หรือ LONG_MIN
        
        NSLog(@"NSInteger: %ld", (long)count);
        NSLog(@"NSInteger negative: %ld", (long)negative);
        NSLog(@"NSInteger Max: %ld", (long)maxNSInteger);
        NSLog(@"NSInteger Min: %ld", (long)minNSInteger);
        
        // NSUInteger - unsigned version ของ NSInteger
        NSUInteger arrayCount = 10;
        NSUInteger maxNSUInteger = NSUIntegerMax;  // UINT_MAX หรือ ULONG_MAX
        
        NSLog(@"NSUInteger: %lu", (unsigned long)arrayCount);
        NSLog(@"NSUInteger Max: %lu", (unsigned long)maxNSUInteger);
        
        // ทำไมต้องใช้ NSInteger แทน int?
        // 1. Platform independent - ทำงานถูกต้องทั้ง 32-bit และ 64-bit
        // 2. Matches Apple's API - NSArray.count, NSString.length ส่งคืน NSUInteger
        // 3. Safer arithmetic - ลด overflow risk บน 64-bit
        
        // ตัวอย่างการใช้กับ Collections
        NSArray *items = @[@"a", @"b", @"c", @"d", @"e"];
        NSUInteger itemCount = items.count;  // count ส่งคืน NSUInteger
        
        for (NSUInteger i = 0; i < itemCount; i++) {
            NSLog(@"items[%lu]: %@", (unsigned long)i, items[i]);
        }
        
        // การแปลงระหว่าง NSInteger และ int
        int regularInt = 42;
        NSInteger nsInt = (NSInteger)regularInt;  // explicit cast
        int backToInt = (int)nsInt;               // cast กลับ (ระวัง overflow บน 64-bit)
        
        NSLog(@"regular int: %d", regularInt);
        NSLog(@"NSInteger: %ld", (long)nsInt);
        NSLog(@"back to int: %d", backToInt);
        
    }
    return 0;
}
```

### CGFloat

```objc
#import <Foundation/Foundation.h>
#import <CoreGraphics/CoreGraphics.h>  // หรือ #import <UIKit/UIKit.h> บน iOS

int main(int argc, const char * argv[]) {
    @autoreleasepool {
        
        // CGFloat - floating point ขนาดเท่ากับ pointer ของ platform
        // บน 32-bit: เท่ากับ float (4 bytes)
        // บน 64-bit: เท่ากับ double (8 bytes)
        
        CGFloat width = 320.0f;
        CGFloat height = 568.0f;
        CGFloat scale = 2.0f;
        CGFloat screenArea = width * height;
        
        NSLog(@"Width: %g", width);
        NSLog(@"Height: %g", height);
        NSLog(@"Scale: %g", scale);
        NSLog(@"Screen Area: %g", screenArea);
        
        // CGFloat constants
        CGFloat cgMax = CGFLOAT_MAX;
        CGFloat cgMin = CGFLOAT_MIN;
        
        NSLog(@"CGFloat Max: %g", cgMax);
        NSLog(@"CGFloat Min: %g", cgMin);
        
        // CGRect, CGSize, CGPoint - ใช้ CGFloat เป็น components
        CGRect rect = CGRectMake(0.0f, 0.0f, width, height);
        CGSize size = CGSizeMake(width, height);
        CGPoint center = CGPointMake(width / 2.0f, height / 2.0f);
        
        NSLog(@"Rect: {{%.1f, %.1f}, {%.1f, %.1f}}", 
              rect.origin.x, rect.origin.y, rect.size.width, rect.size.height);
        NSLog(@"Size: {%.1f, %.1f}", size.width, size.height);
        NSLog(@"Center: {%.1f, %.1f}", center.x, center.y);
        
        // CGFloat arithmetic
        CGFloat radius = 50.0f;
        CGFloat circumference = 2 * M_PI * radius;  // M_PI คือ pi ≈ 3.14159
        CGFloat area = M_PI * radius * radius;
        
        NSLog(@"Circle radius: %.1f", radius);
        NSLog(@"Circumference: %.4f", circumference);
        NSLog(@"Area: %.4f", area);
        
        // ทำไมต้องใช้ CGFloat แทน float?
        // UIKit/AppKit API ทั้งหมดใช้ CGFloat
        // ทำให้ code portable ระหว่าง 32-bit และ 64-bit
        // ลด warning จาก implicit conversion
        
    }
    return 0;
}
```

---

## 5. Constants (ค่าคงที่)

### #define Macro

```objc
#import <Foundation/Foundation.h>

// #define - preprocessor macro
// ไม่มี type safety, ไม่มี scope, แค่ text substitution
#define MAX_STUDENTS 50
#define PI_VALUE 3.14159265358979
#define APP_NAME @"MyApp"
#define GREETING(name) [NSString stringWithFormat:@"สวัสดี %@!", name]

// ควรใส่ parentheses รอบ expression เพื่อความปลอดภัย
#define SQUARE(x) ((x) * (x))
#define MAX_VAL(a, b) ((a) > (b) ? (a) : (b))

int main(int argc, const char * argv[]) {
    @autoreleasepool {
        
        NSLog(@"Max students: %d", MAX_STUDENTS);
        NSLog(@"PI: %f", PI_VALUE);
        NSLog(@"App name: %@", APP_NAME);
        NSLog(@"%@", GREETING(@"สมชาย"));
        NSLog(@"5 squared: %d", SQUARE(5));
        NSLog(@"Max(10, 20): %d", MAX_VAL(10, 20));
        
        // อันตรายของ #define ที่ไม่มี parentheses
        // #define BAD_SQUARE(x) x * x
        // BAD_SQUARE(2 + 3) = 2 + 3 * 2 + 3 = 11 (ผิด!)
        // SQUARE(2 + 3) = (2+3) * (2+3) = 25 (ถูก)
        
    }
    return 0;
}
```

### const Variables

```objc
#import <Foundation/Foundation.h>

// Global const - ใช้ static เพื่อจำกัด visibility ใน file นี้
static const NSInteger kMaxRetryCount = 3;
static const double kGravity = 9.81;
static const NSString * const kAppVersion = @"1.0.0";

// Global ที่ใช้ใน multiple files - ประกาศใน .h และ define ใน .m
// MyConstants.h: extern NSString * const kAPIBaseURL;
// MyConstants.m: NSString * const kAPIBaseURL = @"https://api.example.com";

int main(int argc, const char * argv[]) {
    @autoreleasepool {
        
        // Local const
        const int MAX_SIZE = 100;
        const double PI = 3.14159265358979;
        
        // NSString const
        NSString * const kGreeting = @"สวัสดี";  // pointer เป็น const
        
        NSLog(@"Max size: %d", MAX_SIZE);
        NSLog(@"PI: %.10f", PI);
        NSLog(@"Greeting: %@", kGreeting);
        
        // Global consts
        NSLog(@"Max retry: %ld", (long)kMaxRetryCount);
        NSLog(@"Gravity: %.2f", kGravity);
        NSLog(@"App version: %@", kAppVersion);
        
        // ความแตกต่างระหว่าง const pointers
        NSString *str = @"Hello";
        
        // NSString * const ptr - pointer เป็น const (ชี้ไปที่เดิมเสมอ)
        NSString * const constPtr = str;
        // constPtr = @"World";  // ERROR! ไม่สามารถเปลี่ยน pointer
        
        // const NSString * ptr - object ที่ชี้ไปเป็น const
        const NSString *ptrToConst = str;
        ptrToConst = @"World";  // OK: เปลี่ยน pointer ได้
        // ไม่สามารถแก้ไข ptrToConst ผ่าน pointer นี้
        
        // const NSString * const ptr - ทั้ง pointer และ object เป็น const
        const NSString * const fullyConst = @"Immutable";
        
        NSLog(@"constPtr: %@", constPtr);
        NSLog(@"ptrToConst: %@", ptrToConst);
        NSLog(@"fullyConst: %@", fullyConst);
        
    }
    return 0;
}
```

### static const (แนะนำที่สุด)

```objc
#import <Foundation/Foundation.h>

// ========== Constants File Pattern ==========
// AppConstants.h - header สำหรับประกาศ
// AppConstants.m - implementation สำหรับ define

// สำหรับ NSString constants ที่ใช้ข้าม files
extern NSString * const kNotificationUserLoggedIn;
extern NSString * const kNotificationUserLoggedOut;
extern NSString * const kUserDefaultsKeyTheme;

// AppConstants.m
NSString * const kNotificationUserLoggedIn = @"UserDidLogIn";
NSString * const kNotificationUserLoggedOut = @"UserDidLogOut";  
NSString * const kUserDefaultsKeyTheme = @"selectedTheme";

// ========== Class-level Constants Pattern ==========
@interface Configuration : NSObject

// Class properties (Objective-C ไม่มี class properties แบบ Swift)
// ใช้ class methods แทน
+ (NSString *)baseURL;
+ (NSInteger)timeoutSeconds;
+ (NSString *)appVersion;

@end

@implementation Configuration

+ (NSString *)baseURL {
    return @"https://api.example.com/v1";
}

+ (NSInteger)timeoutSeconds {
    return 30;
}

+ (NSString *)appVersion {
    return @"2.0.1";
}

@end

int main(int argc, const char * argv[]) {
    @autoreleasepool {
        
        // ใช้ Configuration constants
        NSLog(@"Base URL: %@", [Configuration baseURL]);
        NSLog(@"Timeout: %ld seconds", (long)[Configuration timeoutSeconds]);
        NSLog(@"Version: %@", [Configuration appVersion]);
        
        // Enumeration สำหรับ integer constants
        typedef NS_ENUM(NSInteger, DayOfWeek) {
            DayOfWeekSunday = 0,
            DayOfWeekMonday,
            DayOfWeekTuesday,
            DayOfWeekWednesday,
            DayOfWeekThursday,
            DayOfWeekFriday,
            DayOfWeekSaturday
        };
        
        DayOfWeek today = DayOfWeekWednesday;
        NSLog(@"Today: %ld", (long)today);
        
        NSString *dayName;
        switch (today) {
            case DayOfWeekSunday: dayName = @"อาทิตย์"; break;
            case DayOfWeekMonday: dayName = @"จันทร์"; break;
            case DayOfWeekTuesday: dayName = @"อังคาร"; break;
            case DayOfWeekWednesday: dayName = @"พุธ"; break;
            case DayOfWeekThursday: dayName = @"พฤหัสบดี"; break;
            case DayOfWeekFriday: dayName = @"ศุกร์"; break;
            case DayOfWeekSaturday: dayName = @"เสาร์"; break;
            default: dayName = @"ไม่ทราบ"; break;
        }
        NSLog(@"วัน%@", dayName);
        
        // NS_OPTIONS สำหรับ bitmask flags
        typedef NS_OPTIONS(NSUInteger, Permissions) {
            PermissionRead    = 1 << 0,  // 0001 = 1
            PermissionWrite   = 1 << 1,  // 0010 = 2
            PermissionExecute = 1 << 2,  // 0100 = 4
            PermissionAdmin   = 1 << 3   // 1000 = 8
        };
        
        // รวม permissions
        Permissions userPerms = PermissionRead | PermissionWrite;
        NSLog(@"User permissions: %lu", (unsigned long)userPerms);
        
        // ตรวจสอบ permission
        if (userPerms & PermissionRead) {
            NSLog(@"มีสิทธิ์อ่าน");
        }
        if (userPerms & PermissionWrite) {
            NSLog(@"มีสิทธิ์เขียน");
        }
        if (!(userPerms & PermissionExecute)) {
            NSLog(@"ไม่มีสิทธิ์รัน");
        }
        
    }
    return 0;
}
```

---

## 6. Type Casting (การแปลงประเภทข้อมูล)

### Implicit vs Explicit Casting

```objc
#import <Foundation/Foundation.h>

int main(int argc, const char * argv[]) {
    @autoreleasepool {
        
        // ========== Implicit Casting (อัตโนมัติ) ==========
        
        // ขยาย (widening) - ปลอดภัย ไม่สูญเสียข้อมูล
        int i = 42;
        long l = i;         // int -> long (OK)
        float f = i;        // int -> float (OK, แต่อาจสูญ precision)
        double d = f;       // float -> double (OK)
        
        NSLog(@"int: %d -> long: %ld -> float: %f -> double: %f", i, l, f, d);
        
        // ขนาดลด (narrowing) - อาจสูญเสียข้อมูล, compiler อาจแสดง warning
        double big = 3.99;
        int truncated = big;  // ตัดทศนิยมทิ้ง (ไม่ปัดเศษ!)
        NSLog(@"%.2f -> %d (truncated, not rounded)", big, truncated);
        
        // ========== Explicit Casting ==========
        
        // C-style cast: (type)expression
        int a = 10, b = 3;
        
        // Integer division (ผิด - ได้ 3 ไม่ใช่ 3.333)
        double wrongResult = a / b;
        NSLog(@"ผิด: %d / %d = %.4f", a, b, wrongResult);
        
        // Float division (ถูก - cast ก่อน divide)
        double correctResult = (double)a / b;
        NSLog(@"ถูก: (double)%d / %d = %.4f", a, b, correctResult);
        
        // Cast ระหว่าง numeric types
        long longVal = 1234567890L;
        int intVal = (int)longVal;     // อาจ overflow ถ้าค่าใหญ่เกิน
        char charVal = (char)65;       // 65 = 'A'
        
        NSLog(@"long -> int: %ld -> %d", longVal, intVal);
        NSLog(@"int -> char: 65 -> '%c'", charVal);
        
        // float -> int (ตัดทศนิยม)
        float fVal = 9.99f;
        int iVal = (int)fVal;          // 9 (ไม่ปัดเศษ)
        NSLog(@"%.2f -> %d", fVal, iVal);
        
        // ปัดเศษ ต้องใช้ round/ceil/floor จาก math.h
        #include <math.h>
        NSLog(@"round(%.2f) = %.0f", fVal, round(fVal));
        NSLog(@"ceil(%.2f) = %.0f", fVal, ceil(fVal));
        NSLog(@"floor(%.2f) = %.0f", fVal, floor(fVal));
        
        // ========== ObjC Object Casting ==========
        
        // Upcasting - object -> superclass (implicit, always safe)
        NSMutableString *mStr = [NSMutableString stringWithString:@"Hello"];
        NSString *str = mStr;  // NSMutableString is-a NSString (OK)
        NSLog(@"Upcast: %@", str);
        
        // Downcasting - superclass -> subclass (explicit, may fail at runtime)
        id someObject = @"Hello World";  // ใช้ id เพื่อเก็บ any object
        
        // ตรวจสอบก่อน downcast (แนะนำ)
        if ([someObject isKindOfClass:[NSString class]]) {
            NSString *castStr = (NSString *)someObject;
            NSLog(@"Downcast OK: %@", castStr);
        }
        
        // ไม่ปลอดภัย - อาจ crash ถ้า object ไม่ใช่ type ที่ต้องการ
        // NSNumber *wrongCast = (NSNumber *)someObject; // ไม่ crash แต่พฤติกรรมไม่แน่นอน
        
        // ใช้ as (เหมือน Swift) ผ่าน respondsToSelector ก่อน
        if ([someObject respondsToSelector:@selector(uppercaseString)]) {
            NSString *upper = [(NSString *)someObject uppercaseString];
            NSLog(@"uppercase: %@", upper);
        }
        
        // ========== NSNumber และ Primitive Conversion ==========
        
        // Primitive -> NSNumber (boxing)
        NSNumber *intNSNum = @42;                    // int literal syntax
        NSNumber *floatNSNum = @3.14f;               // float literal
        NSNumber *doubleNSNum = @3.14159;            // double literal
        NSNumber *boolNSNum = @YES;                  // BOOL literal
        NSNumber *longNSNum = @(1234567890L);        // expression syntax
        
        NSLog(@"NSNumber int: %@", intNSNum);
        NSLog(@"NSNumber float: %@", floatNSNum);
        NSLog(@"NSNumber bool: %@", boolNSNum);
        
        // NSNumber -> Primitive (unboxing)
        int unboxedInt = [intNSNum intValue];
        float unboxedFloat = [floatNSNum floatValue];
        double unboxedDouble = [doubleNSNum doubleValue];
        BOOL unboxedBool = [boolNSNum boolValue];
        NSInteger unboxedNSInt = [intNSNum integerValue];
        
        NSLog(@"Unboxed int: %d", unboxedInt);
        NSLog(@"Unboxed float: %.2f", unboxedFloat);
        NSLog(@"Unboxed BOOL: %@", unboxedBool ? @"YES" : @"NO");
        
        // String <-> Number conversion
        NSString *numStr = @"123.45";
        double num = [numStr doubleValue];
        NSString *backToStr = [NSString stringWithFormat:@"%.2f", num];
        
        NSLog(@"String to num: '%@' -> %.2f", numStr, num);
        NSLog(@"Num to string: %.2f -> '%@'", num, backToStr);
        
        // NSNumber comparison
        NSNumber *n1 = @100;
        NSNumber *n2 = @200;
        NSComparisonResult cmp = [n1 compare:n2];
        NSLog(@"100 vs 200: %@", cmp == NSOrderedAscending ? @"น้อยกว่า" : @"มากกว่าหรือเท่ากัน");
        
    }
    return 0;
}
```

---

## 7. nil vs NULL vs Nil

### ความแตกต่างและการใช้งาน

```objc
#import <Foundation/Foundation.h>
#import <objc/runtime.h>

int main(int argc, const char * argv[]) {
    @autoreleasepool {
        
        // ========== nil ==========
        // ใช้กับ Objective-C object pointers
        // ค่าเท่ากับ 0 / (id)0
        
        NSString *str = nil;          // NSString pointer เป็น nil
        NSArray *arr = nil;           // NSArray pointer เป็น nil
        id obj = nil;                 // id เป็น nil
        
        NSLog(@"str is nil: %@", str == nil ? @"YES" : @"NO");
        
        // Safe: ส่ง message ไปยัง nil ไม่ crash
        NSUInteger len = [str length];  // ส่งคืน 0 แทนที่จะ crash
        NSLog(@"Length of nil string: %lu", (unsigned long)len);  // 0
        
        // ตรวจสอบ nil
        if (str == nil) {
            NSLog(@"str is nil");
        }
        
        if (!str) {
            NSLog(@"str is falsy (nil)");  // nil เป็น falsy
        }
        
        // ========== NULL ==========
        // ใช้กับ C pointer (void *, int *, struct *, etc.)
        // เหมาะสำหรับ C functions
        
        int *intPtr = NULL;           // C int pointer เป็น NULL
        void *voidPtr = NULL;         // C void pointer เป็น NULL
        char *cStr = NULL;            // C string pointer เป็น NULL
        
        NSLog(@"intPtr is NULL: %@", intPtr == NULL ? @"YES" : @"NO");
        
        // ตรวจสอบ NULL ก่อนใช้งาน
        if (intPtr != NULL) {
            NSLog(@"value: %d", *intPtr);
        } else {
            NSLog(@"intPtr is NULL, cannot dereference");
        }
        
        // NULL ในบริบทของ C functions
        const char *cResult = [str UTF8String];  // ส่งคืน NULL ถ้า str เป็น nil
        if (cResult != NULL) {
            printf("C string: %s\n", cResult);
        } else {
            NSLog(@"C string is NULL");
        }
        
        // ========== Nil (capital N) ==========
        // ใช้กับ Class pointers
        // ไม่ค่อยได้ใช้ในการเขียนโปรแกรมปกติ
        
        Class cls = Nil;              // Class pointer เป็น Nil
        NSLog(@"cls is Nil: %@", cls == Nil ? @"YES" : @"NO");
        
        // ตัวอย่างการใช้ Nil
        Class stringClass = [NSString class];
        Class unknownClass = Nil;
        
        if (stringClass != Nil) {
            NSLog(@"stringClass: %@", NSStringFromClass(stringClass));
        }
        
        // ========== nil vs NSNull ==========
        // NSNull - singleton object ที่แทน "ไม่มีค่า" ใน collections
        // ใช้เมื่อต้องการเก็บ "empty" ใน NSArray หรือ NSDictionary
        // (เพราะ NSArray ไม่สามารถเก็บ nil ได้โดยตรง)
        
        // ERROR: ใส่ nil ใน array โดยตรงไม่ได้
        // NSArray *arr = @[@"first", nil, @"third"];  // หยุดที่ nil!
        
        // ถูก: ใช้ NSNull แทน
        NSArray *arr2 = @[@"first", [NSNull null], @"third"];
        NSLog(@"Array with NSNull: %@", arr2);
        
        // ตรวจสอบ NSNull
        for (id item in arr2) {
            if ([item isEqual:[NSNull null]]) {
                NSLog(@"Found NSNull");
            } else {
                NSLog(@"Item: %@", item);
            }
        }
        
        // Dictionary กับ nil
        NSMutableDictionary *dict = [NSMutableDictionary dictionary];
        dict[@"name"] = @"สมชาย";
        dict[@"age"] = [NSNull null];  // แทน nil ด้วย NSNull
        // dict[@"city"] = nil;         // ERROR: removes key!
        [dict removeObjectForKey:@"city"];  // วิธีที่ถูกต้องในการ "ลบ" key
        
        for (NSString *key in dict) {
            id value = dict[key];
            if ([value isEqual:[NSNull null]]) {
                NSLog(@"%@: (null)", key);
            } else {
                NSLog(@"%@: %@", key, value);
            }
        }
        
    }
    return 0;
}
```

---

## 8. BOOL (YES/NO) vs bool (true/false)

### ความแตกต่างและข้อควรระวัง

```objc
#import <Foundation/Foundation.h>
#include <stdbool.h>  // สำหรับ bool, true, false

int main(int argc, const char * argv[]) {
    @autoreleasepool {
        
        // ========== BOOL (Objective-C) ==========
        // typedef signed char BOOL;
        // YES = (BOOL)1
        // NO = (BOOL)0
        
        BOOL isTrue = YES;
        BOOL isFalse = NO;
        
        NSLog(@"YES = %d", YES);     // 1
        NSLog(@"NO = %d", NO);       // 0
        
        // Conditional ด้วย BOOL
        if (isTrue) {
            NSLog(@"isTrue is truthy");
        }
        
        if (!isFalse) {
            NSLog(@"!isFalse is truthy");
        }
        
        // BOOL จาก comparison
        BOOL isEqual = (10 == 10);   // YES
        BOOL isGreater = (5 > 10);   // NO
        NSLog(@"10 == 10: %@", isEqual ? @"YES" : @"NO");
        NSLog(@"5 > 10: %@", isGreater ? @"YES" : @"NO");
        
        // ========== BOOL Pitfall: Overflow ==========
        // เพราะ BOOL เป็น signed char (1 byte)
        // ค่า 256 จะ overflow เป็น 0 (NO)!
        
        BOOL problematic = 256;      // Overflow! 256 & 0xFF = 0 = NO
        NSLog(@"BOOL 256: %d (expected 1, got 0!)", problematic);
        
        // Fix: ใช้ !! (double negation) เพื่อ normalize เป็น 0 หรือ 1
        BOOL safe = !!(256);
        NSLog(@"!!256: %d", safe);   // 1 = YES
        
        // หรือเปรียบเทียบ explicitly
        int count = 256;
        BOOL hasItems = (count != 0);  // ปลอดภัย
        NSLog(@"hasItems: %@", hasItems ? @"YES" : @"NO");
        
        // ========== bool (C99) ==========
        // _Bool เป็น type จริง, bool เป็น macro
        // true = 1, false = 0
        // ปลอดภัยกว่า BOOL - ไม่มีปัญหา overflow
        
        bool cTrue = true;
        bool cFalse = false;
        
        NSLog(@"true = %d", cTrue);    // 1
        NSLog(@"false = %d", cFalse);  // 0
        
        // bool ไม่มีปัญหา overflow
        bool safeBool = 256;  // ใน C99, 256 != 0 ดังนั้น = true (1)
        NSLog(@"bool 256: %d", safeBool);  // 1
        
        // ========== Interoperability ==========
        
        // BOOL -> bool (ปลอดภัย)
        BOOL objcBool = YES;
        bool cppBool = (bool)objcBool;  // หรือ !!objcBool
        NSLog(@"BOOL YES -> bool: %d", cppBool);  // 1
        
        // bool -> BOOL (ปลอดภัย)
        bool boolVal = true;
        BOOL objcBool2 = (BOOL)boolVal;
        NSLog(@"bool true -> BOOL: %d", objcBool2);  // 1
        
        // ========== Boolean ใน Collections ==========
        
        // ไม่สามารถใส่ BOOL โดยตรงใน NSArray
        // ต้องใช้ @YES / @NO (NSNumber literals)
        
        NSArray *flags = @[@YES, @NO, @YES, @YES, @NO];
        for (NSNumber *flag in flags) {
            BOOL b = [flag boolValue];
            NSLog(@"Flag: %@", b ? @"YES" : @"NO");
        }
        
        NSDictionary *settings = @{
            @"darkMode": @YES,
            @"notifications": @NO,
            @"autoSave": @YES
        };
        
        BOOL darkMode = [settings[@"darkMode"] boolValue];
        NSLog(@"Dark mode: %@", darkMode ? @"เปิด" : @"ปิด");
        
        // ========== NSNumber สำหรับ Boolean ==========
        NSNumber *boolNumber = @YES;
        NSLog(@"NSNumber YES: %@", boolNumber);     // 1
        NSLog(@"boolValue: %d", [boolNumber boolValue]);
        NSLog(@"intValue: %d", [boolNumber intValue]);
        
        // Equality
        if ([boolNumber isEqual:@YES]) {
            NSLog(@"boolNumber equals @YES");
        }
        
    }
    return 0;
}
```

---

## 9. NSLog Format Specifiers

### ตารางสรุป Format Specifiers ทั้งหมด

```objc
#import <Foundation/Foundation.h>

int main(int argc, const char * argv[]) {
    @autoreleasepool {
        
        NSLog(@"===== NSLog Format Specifiers =====\n");
        
        // ========== Integer Specifiers ==========
        int i = 42;
        int negative = -42;
        unsigned int ui = 4294967295U;
        
        NSLog(@"%%d   - signed decimal int:  %d", i);
        NSLog(@"%%i   - signed decimal int:  %i", i);
        NSLog(@"%%u   - unsigned decimal:    %u", ui);
        NSLog(@"%%o   - unsigned octal:      %o", i);
        NSLog(@"%%x   - unsigned hex lower:  %x", i);
        NSLog(@"%%X   - unsigned hex upper:  %X", i);
        NSLog(@"%%ld  - long decimal:        %ld", (long)i);
        NSLog(@"%%lu  - unsigned long:       %lu", (unsigned long)i);
        NSLog(@"%%lld - long long:           %lld", (long long)i);
        NSLog(@"%%llu - unsigned long long:  %llu", (unsigned long long)i);
        
        // NSInteger requires %ld cast
        NSInteger nsInt = 100;
        NSLog(@"NSInteger (%%ld):  %ld", (long)nsInt);
        NSUInteger nsUInt = 200;
        NSLog(@"NSUInteger (%%lu): %lu", (unsigned long)nsUInt);
        
        NSLog(@"");
        
        // ========== Floating-Point Specifiers ==========
        float f = 3.14159f;
        double d = 3.14159265358979;
        
        NSLog(@"%%f   - decimal:        %f", f);
        NSLog(@"%%e   - scientific:     %e", f);
        NSLog(@"%%E   - scientific CAP: %E", f);
        NSLog(@"%%g   - shorter of f/e: %g", f);
        NSLog(@"%%G   - shorter of f/E: %G", f);
        NSLog(@"%%.2f - 2 decimals:     %.2f", d);
        NSLog(@"%%10.3f - width 10:     %10.3f", d);
        NSLog(@"%%-10.3f - left align:  %-10.3f|", d);
        NSLog(@"%%010.3f - zero pad:    %010.3f", d);
        NSLog(@"%%Lf  - long double:    %Lf", (long double)d);
        
        // CGFloat ใช้ %f หรือ %g
        CGFloat cgf = 100.5f;
        NSLog(@"CGFloat (%%g): %g", (double)cgf);
        
        NSLog(@"");
        
        // ========== Character และ String Specifiers ==========
        char ch = 'A';
        const char *cStr = "Hello C";
        NSString *nsStr = @"Hello ObjC";
        
        NSLog(@"%%c   - char:           %c", ch);
        NSLog(@"%%s   - C string:       %s", cStr);
        NSLog(@"%%@   - NSObject:       %@", nsStr);
        NSLog(@"%%@   - NSNumber:       %@", @42);
        NSLog(@"%%@   - NSArray:        %@", @[@1, @2, @3]);
        
        NSLog(@"");
        
        // ========== Pointer และ อื่นๆ ==========
        int x = 10;
        NSLog(@"%%p   - pointer:        %p", &x);
        NSLog(@"%%%%  - literal %%:      100%%");
        
        NSLog(@"");
        
        // ========== Width และ Precision Formatting ==========
        NSLog(@"--- Width Formatting ---");
        int num = 42;
        NSLog(@"|%5d|   right-aligned, width 5", num);
        NSLog(@"|%-5d|  left-aligned, width 5", num);
        NSLog(@"|%05d|  zero-padded, width 5", num);
        NSLog(@"|%+d|   show sign", num);
        NSLog(@"|%+d|   show sign neg", -num);
        
        NSLog(@"");
        NSLog(@"--- Precision Formatting ---");
        double pi = 3.14159265358979;
        NSLog(@"|%f|      default precision", pi);
        NSLog(@"|%.0f|     0 decimals", pi);
        NSLog(@"|%.2f|     2 decimals", pi);
        NSLog(@"|%.10f|    10 decimals", pi);
        NSLog(@"|%10.3f|   width 10, 3 decimals", pi);
        NSLog(@"|%-10.3f|  left-aligned", pi);
        
        NSLog(@"");
        
        // ========== Multiple Values ==========
        NSString *name = @"สมชาย";
        int age = 25;
        double score = 87.5;
        BOOL passed = YES;
        
        NSLog(@"ชื่อ: %@, อายุ: %d ปี, คะแนน: %.1f, ผ่าน: %@",
              name, age, score, passed ? @"ใช่" : @"ไม่ใช่");
        
    }
    return 0;
}
```

---

## 10. sizeof Operator

```objc
#import <Foundation/Foundation.h>

int main(int argc, const char * argv[]) {
    @autoreleasepool {
        
        NSLog(@"===== sizeof Operator =====\n");
        
        // sizeof(type) - ขนาดของ type ใน bytes
        NSLog(@"sizeof(char):          %zu bytes", sizeof(char));
        NSLog(@"sizeof(short):         %zu bytes", sizeof(short));
        NSLog(@"sizeof(int):           %zu bytes", sizeof(int));
        NSLog(@"sizeof(long):          %zu bytes", sizeof(long));
        NSLog(@"sizeof(long long):     %zu bytes", sizeof(long long));
        NSLog(@"sizeof(float):         %zu bytes", sizeof(float));
        NSLog(@"sizeof(double):        %zu bytes", sizeof(double));
        NSLog(@"sizeof(long double):   %zu bytes", sizeof(long double));
        NSLog(@"sizeof(BOOL):          %zu bytes", sizeof(BOOL));
        NSLog(@"sizeof(bool):          %zu bytes", sizeof(bool));
        NSLog(@"sizeof(id):            %zu bytes", sizeof(id));
        NSLog(@"sizeof(NSInteger):     %zu bytes", sizeof(NSInteger));
        NSLog(@"sizeof(NSUInteger):    %zu bytes", sizeof(NSUInteger));
        NSLog(@"sizeof(CGFloat):       %zu bytes", sizeof(CGFloat));
        
        NSLog(@"");
        NSLog(@"sizeof pointers:");
        NSLog(@"sizeof(void *):        %zu bytes", sizeof(void *));
        NSLog(@"sizeof(NSString *):    %zu bytes", sizeof(NSString *));
        NSLog(@"sizeof(int *):         %zu bytes", sizeof(int *));
        
        NSLog(@"");
        
        // sizeof กับ variable
        int x = 0;
        double d = 0.0;
        int arr[10];
        
        NSLog(@"sizeof variable x:     %zu bytes", sizeof(x));
        NSLog(@"sizeof variable d:     %zu bytes", sizeof(d));
        NSLog(@"sizeof int[10] array:  %zu bytes", sizeof(arr));
        NSLog(@"Array count:           %zu elements", sizeof(arr) / sizeof(arr[0]));
        
        NSLog(@"");
        
        // sizeof struct
        typedef struct {
            char c;    // 1 byte
            int i;     // 4 bytes
            double d;  // 8 bytes
        } MyStruct;
        
        MyStruct s;
        NSLog(@"sizeof MyStruct:       %zu bytes", sizeof(MyStruct));
        NSLog(@"(อาจมากกว่า 13 เพราะ struct alignment/padding)");
        
        // ตรวจสอบ platform
        NSLog(@"");
        NSLog(@"--- Platform Info ---");
        if (sizeof(void *) == 8) {
            NSLog(@"Running on 64-bit platform");
        } else {
            NSLog(@"Running on 32-bit platform");
        }
        
        NSLog(@"NSInteger size: %zu (same as void* = %zu)",
              sizeof(NSInteger), sizeof(void *));
        
        // sizeof กับ array ที่รับผ่าน parameter ไม่ทำงาน
        // void wrongFunc(int arr[]) { sizeof(arr); } // ได้ขนาด pointer ไม่ใช่ array
        // ต้องส่งขนาดเป็น parameter แยก
        
    }
    return 0;
}
```

---

## 11. ตัวอย่างโค้ดสมบูรณ์: Student Database

```objc
// File: main.m
// Student Database - แสดงการใช้ Data Types ทั้งหมด

#import <Foundation/Foundation.h>

// ========== Constants ==========
static const NSInteger kMinAge = 15;
static const NSInteger kMaxAge = 100;
static const double kPassingScore = 50.0;
static NSString * const kUnknownValue = @"ไม่ทราบ";

// ========== Enum ==========
typedef NS_ENUM(NSInteger, StudentStatus) {
    StudentStatusActive = 0,
    StudentStatusInactive,
    StudentStatusGraduated,
    StudentStatusExpelled
};

typedef NS_ENUM(NSInteger, Grade) {
    GradeA = 5,
    GradeB = 4,
    GradeC = 3,
    GradeD = 2,
    GradeF = 0
};

// ========== Interface ==========
@interface Student : NSObject

// Properties with various types
@property (nonatomic, strong) NSString *studentID;       // NSString
@property (nonatomic, strong) NSString *firstName;       // NSString
@property (nonatomic, strong) NSString *lastName;        // NSString
@property (nonatomic, assign) NSInteger age;             // NSInteger
@property (nonatomic, assign) double gpa;                // double
@property (nonatomic, assign) BOOL isScholarship;        // BOOL
@property (nonatomic, assign) StudentStatus status;      // Enum
@property (nonatomic, strong) NSMutableArray *scores;    // NSMutableArray
@property (nonatomic, strong) NSDate *enrollmentDate;    // NSDate

// Computed (readonly)
@property (nonatomic, readonly) NSString *fullName;
@property (nonatomic, readonly) double averageScore;
@property (nonatomic, readonly) Grade currentGrade;

// Initializers
- (instancetype)initWithFirstName:(NSString *)first 
                         lastName:(NSString *)last 
                              age:(NSInteger)age;

// Methods
- (void)addScore:(double)score;
- (BOOL)isEligibleForScholarship;
- (NSString *)statusDescription;
- (void)printReport;

+ (Student *)sampleStudent;

@end

// ========== Implementation ==========
@implementation Student

- (instancetype)initWithFirstName:(NSString *)first 
                         lastName:(NSString *)last 
                              age:(NSInteger)age {
    self = [super init];
    if (self) {
        // Validate and set
        _firstName = first ? first : kUnknownValue;
        _lastName = last ? last : kUnknownValue;
        _age = MAX(kMinAge, MIN(kMaxAge, age));
        _gpa = 0.0;
        _isScholarship = NO;
        _status = StudentStatusActive;
        _scores = [NSMutableArray array];
        _enrollmentDate = [NSDate date];
        
        // Generate student ID
        _studentID = [NSString stringWithFormat:@"STD%05d", 
                      (int)arc4random_uniform(99999)];
    }
    return self;
}

// Computed property: fullName
- (NSString *)fullName {
    return [NSString stringWithFormat:@"%@ %@", self.firstName, self.lastName];
}

// Computed property: averageScore
- (double)averageScore {
    if (self.scores.count == 0) return 0.0;
    
    double total = 0.0;
    for (NSNumber *score in self.scores) {
        total += [score doubleValue];
    }
    return total / self.scores.count;
}

// Computed property: currentGrade
- (Grade)currentGrade {
    double avg = self.averageScore;
    if (avg >= 80) return GradeA;
    if (avg >= 70) return GradeB;
    if (avg >= 60) return GradeC;
    if (avg >= 50) return GradeD;
    return GradeF;
}

- (void)addScore:(double)score {
    // Clamp score between 0 and 100
    double validScore = MAX(0.0, MIN(100.0, score));
    [self.scores addObject:@(validScore)];
    
    // Update GPA based on average
    self.gpa = self.averageScore / 20.0; // Scale 0-100 to 0-5
}

- (BOOL)isEligibleForScholarship {
    return self.averageScore >= 75.0 && self.status == StudentStatusActive;
}

- (NSString *)statusDescription {
    switch (self.status) {
        case StudentStatusActive:    return @"กำลังศึกษา";
        case StudentStatusInactive:  return @"พักการศึกษา";
        case StudentStatusGraduated: return @"สำเร็จการศึกษา";
        case StudentStatusExpelled:  return @"ถูกไล่ออก";
        default:                     return kUnknownValue;
    }
}

- (NSString *)gradeDescription {
    switch (self.currentGrade) {
        case GradeA: return @"A (ดีเยี่ยม)";
        case GradeB: return @"B (ดี)";
        case GradeC: return @"C (ปานกลาง)";
        case GradeD: return @"D (พอผ่าน)";
        case GradeF: return @"F (ไม่ผ่าน)";
        default:     return @"ไม่มีเกรด";
    }
}

- (void)printReport {
    NSLog(@"╔════════════════════════════════╗");
    NSLog(@"║      รายงานผลการเรียน          ║");
    NSLog(@"╠════════════════════════════════╣");
    NSLog(@"║ รหัสนักเรียน : %-16@ ║", self.studentID);
    NSLog(@"║ ชื่อ-นามสกุล : %-16@ ║", self.fullName);
    NSLog(@"║ อายุ         : %-3ld ปี            ║", (long)self.age);
    NSLog(@"║ สถานะ        : %-16@ ║", [self statusDescription]);
    NSLog(@"╠════════════════════════════════╣");
    NSLog(@"║ คะแนนทั้งหมด : %lu วิชา          ║", (unsigned long)self.scores.count);
    
    for (NSUInteger i = 0; i < self.scores.count; i++) {
        double score = [self.scores[i] doubleValue];
        NSString *pass = score >= kPassingScore ? @"✓" : @"✗";
        NSLog(@"║  วิชา %2lu: %5.1f %@              ║", 
              (unsigned long)(i+1), score, pass);
    }
    
    NSLog(@"╠════════════════════════════════╣");
    NSLog(@"║ คะแนนเฉลี่ย  : %5.1f             ║", self.averageScore);
    NSLog(@"║ เกรด         : %-16@ ║", [self gradeDescription]);
    NSLog(@"║ GPA          : %4.2f               ║", self.gpa);
    NSLog(@"║ ทุนการศึกษา  : %-16@ ║", [self isEligibleForScholarship] ? @"มีสิทธิ์" : @"ไม่มีสิทธิ์");
    NSLog(@"╚════════════════════════════════╝");
}

+ (Student *)sampleStudent {
    Student *s = [[Student alloc] initWithFirstName:@"สมชาย" 
                                           lastName:@"ใจดี" 
                                                age:20];
    [s addScore:85.0];
    [s addScore:78.5];
    [s addScore:92.0];
    [s addScore:70.0];
    [s addScore:88.5];
    return s;
}

- (NSString *)description {
    return [NSString stringWithFormat:@"Student(%@, avg=%.1f)", 
            self.fullName, self.averageScore];
}

@end

// ========== Main ==========
int main(int argc, const char * argv[]) {
    @autoreleasepool {
        
        NSLog(@"===== ระบบฐานข้อมูลนักเรียน =====\n");
        
        // สร้างนักเรียน
        NSArray *students = @[
            [Student sampleStudent],
            ({
                Student *s = [[Student alloc] initWithFirstName:@"สมหญิง" 
                                                       lastName:@"งามดี" 
                                                            age:19];
                [s addScore:65.0]; [s addScore:72.0]; [s addScore:58.0];
                s.isScholarship = YES;
                s;
            }),
            ({
                Student *s = [[Student alloc] initWithFirstName:@"วิชัย" 
                                                       lastName:@"เก่งกาจ" 
                                                            age:21];
                [s addScore:95.0]; [s addScore:98.0]; [s addScore:91.0];
                [s addScore:97.0]; [s addScore:93.0];
                s.isScholarship = YES;
                s;
            })
        ];
        
        // แสดงรายงาน
        for (Student *student in students) {
            [student printReport];
            NSLog(@"");
        }
        
        // สรุปห้องเรียน
        double classTotal = 0.0;
        NSInteger passingCount = 0;
        NSInteger scholarshipCount = 0;
        
        for (Student *student in students) {
            classTotal += student.averageScore;
            if (student.averageScore >= kPassingScore) passingCount++;
            if ([student isEligibleForScholarship]) scholarshipCount++;
        }
        
        double classAverage = classTotal / students.count;
        
        NSLog(@"===== สรุปผลห้องเรียน =====");
        NSLog(@"จำนวนนักเรียน: %lu คน", (unsigned long)students.count);
        NSLog(@"คะแนนเฉลี่ยห้อง: %.2f", classAverage);
        NSLog(@"จำนวนที่ผ่าน: %ld คน", (long)passingCount);
        NSLog(@"จำนวนที่ไม่ผ่าน: %ld คน", (long)(students.count - passingCount));
        NSLog(@"มีสิทธิ์ทุน: %ld คน", (long)scholarshipCount);
        
    }
    return 0;
}
```

---

## 12. แบบฝึกหัดพร้อมเฉลย

### แบบฝึกหัดที่ 1: Data Types Explorer

**โจทย์:** เขียนโปรแกรมที่แสดงขนาดและช่วงค่าของ data types ทั้งหมด

**เฉลย:**
```objc
#import <Foundation/Foundation.h>
#include <limits.h>
#include <float.h>

int main(int argc, const char * argv[]) {
    @autoreleasepool {
        
        NSLog(@"%-20s %-10s %-20s %-20s", "Type", "Size", "Min", "Max");
        NSLog(@"%-20s %-10s %-20s %-20s", 
              "----", "----", "---", "---");
        
        NSLog(@"%-20s %-10zu %-20d %-20d", 
              "char", sizeof(char), CHAR_MIN, CHAR_MAX);
        NSLog(@"%-20s %-10zu %-20d %-20d", 
              "unsigned char", sizeof(unsigned char), 0, UCHAR_MAX);
        NSLog(@"%-20s %-10zu %-20d %-20d", 
              "short", sizeof(short), SHRT_MIN, SHRT_MAX);
        NSLog(@"%-20s %-10zu %-20d %-20d", 
              "int", sizeof(int), INT_MIN, INT_MAX);
        NSLog(@"%-20s %-10zu %-20ld %-20ld", 
              "long", sizeof(long), LONG_MIN, LONG_MAX);
        NSLog(@"%-20s %-10zu %-20lld %-20lld", 
              "long long", sizeof(long long), LLONG_MIN, LLONG_MAX);
        NSLog(@"%-20s %-10zu %-20.2e %-20.2e", 
              "float", sizeof(float), (double)FLT_MIN, (double)FLT_MAX);
        NSLog(@"%-20s %-10zu %-20.2e %-20.2e", 
              "double", sizeof(double), DBL_MIN, DBL_MAX);
        NSLog(@"%-20s %-10zu", "BOOL", sizeof(BOOL));
        NSLog(@"%-20s %-10zu", "NSInteger", sizeof(NSInteger));
        NSLog(@"%-20s %-10zu", "CGFloat", sizeof(CGFloat));
        
    }
    return 0;
}
```

### แบบฝึกหัดที่ 2: Temperature Converter

**โจทย์:** สร้าง program แปลงอุณหภูมิระหว่าง Celsius, Fahrenheit, Kelvin

**เฉลย:**
```objc
#import <Foundation/Foundation.h>

// Constants
static const double kAbsoluteZeroCelsius = -273.15;

// ฟังก์ชันแปลงอุณหภูมิ
double celsiusToFahrenheit(double celsius) {
    return celsius * 9.0 / 5.0 + 32.0;
}

double fahrenheitToCelsius(double fahrenheit) {
    return (fahrenheit - 32.0) * 5.0 / 9.0;
}

double celsiusToKelvin(double celsius) {
    return celsius - kAbsoluteZeroCelsius;
}

double kelvinToCelsius(double kelvin) {
    return kelvin + kAbsoluteZeroCelsius;
}

int main(int argc, const char * argv[]) {
    @autoreleasepool {
        
        NSArray *temperatures = @[@(-40), @0, @20, @37, @100, @(-273.15)];
        
        NSLog(@"%-15s %-15s %-15s", "Celsius (°C)", "Fahrenheit (°F)", "Kelvin (K)");
        NSLog(@"%-15s %-15s %-15s", "------------", "---------------", "---------");
        
        for (NSNumber *tempNum in temperatures) {
            double celsius = [tempNum doubleValue];
            double fahrenheit = celsiusToFahrenheit(celsius);
            double kelvin = celsiusToKelvin(celsius);
            
            NSLog(@"%-15.2f %-15.2f %-15.2f", celsius, fahrenheit, kelvin);
        }
        
        NSLog(@"\nตัวอย่าง:");
        double bodyTemp = 37.0;
        NSLog(@"อุณหภูมิร่างกาย: %.1f°C = %.1f°F = %.2f K",
              bodyTemp, 
              celsiusToFahrenheit(bodyTemp),
              celsiusToKelvin(bodyTemp));
        
    }
    return 0;
}
```

### แบบฝึกหัดที่ 3: String Manipulation

**โจทย์:** เขียนฟังก์ชันที่:
1. นับจำนวนคำในประโยค
2. กลับลำดับคำในประโยค
3. ตรวจสอบ palindrome (อ่านหน้าหลังเหมือนกัน)

**เฉลย:**
```objc
#import <Foundation/Foundation.h>

NSInteger countWords(NSString *sentence) {
    if (!sentence || sentence.length == 0) return 0;
    NSString *trimmed = [sentence stringByTrimmingCharactersInSet:
                         [NSCharacterSet whitespaceCharacterSet]];
    NSArray *words = [trimmed componentsSeparatedByCharactersInSet:
                      [NSCharacterSet whitespaceCharacterSet]];
    // กรองคำว่าง
    NSPredicate *predicate = [NSPredicate predicateWithFormat:@"length > 0"];
    NSArray *nonEmpty = [words filteredArrayUsingPredicate:predicate];
    return nonEmpty.count;
}

NSString *reverseWords(NSString *sentence) {
    NSArray *words = [sentence componentsSeparatedByString:@" "];
    NSMutableArray *reversed = [NSMutableArray arrayWithCapacity:words.count];
    for (NSInteger i = words.count - 1; i >= 0; i--) {
        [reversed addObject:words[i]];
    }
    return [reversed componentsJoinedByString:@" "];
}

BOOL isPalindrome(NSString *str) {
    // แปลงเป็น lowercase และลบ spaces
    NSString *clean = [[str lowercaseString] 
                       stringByReplacingOccurrencesOfString:@" " 
                                                withString:@""];
    NSInteger len = clean.length;
    for (NSInteger i = 0; i < len / 2; i++) {
        unichar left = [clean characterAtIndex:i];
        unichar right = [clean characterAtIndex:len - 1 - i];
        if (left != right) return NO;
    }
    return YES;
}

int main(int argc, const char * argv[]) {
    @autoreleasepool {
        
        NSString *sentence = @"Hello World from Objective-C";
        NSLog(@"ประโยค: '%@'", sentence);
        NSLog(@"จำนวนคำ: %ld", (long)countWords(sentence));
        NSLog(@"กลับลำดับ: '%@'", reverseWords(sentence));
        
        NSLog(@"");
        
        NSArray *words = @[@"racecar", @"hello", @"level", @"world", @"madam", @"kayak"];
        for (NSString *word in words) {
            NSLog(@"'%@' palindrome: %@", word, isPalindrome(word) ? @"YES" : @"NO");
        }
        
    }
    return 0;
}
```

---

## บทสรุป

ใน Part 02 เราได้เรียนรู้:

1. **Primitive Data Types** - char, short, int, long, float, double, BOOL
2. **Variable Declaration** - Syntax, scope, naming conventions
3. **NSString** - การสร้าง, methods, NSMutableString
4. **NSInteger/NSUInteger/CGFloat** - Platform-independent types
5. **Constants** - #define, const, static const, extern, NS_ENUM
6. **Type Casting** - Implicit, explicit, ObjC object casting
7. **nil/NULL/Nil** - ความแตกต่างและการใช้งาน, NSNull
8. **BOOL vs bool** - YES/NO vs true/false, overflow pitfall
9. **NSLog Format Specifiers** - %@, %d, %f, %s และอื่นๆ
10. **sizeof** - ขนาด data types บนแต่ละ platform

## บทต่อไป

ใน **Part 03** เราจะเรียนรู้เรื่อง:
- Arithmetic operators (+, -, *, /, %)
- Comparison operators (==, !=, <, >, <=, >=)
- Logical operators (&&, ||, !)
- Bitwise operators (&, |, ^, ~, <<, >>)
- Assignment operators
- Ternary operator
- Increment/Decrement
- Operator precedence

---

*เอกสารนี้เป็นส่วนหนึ่งของหลักสูตร Objective-C Programming*
*Part 02 of 100*
