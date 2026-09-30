# Part 03: Operators และ Expressions ใน Objective-C

---

## สารบัญ (Table of Contents)

1. Arithmetic Operators (ตัวดำเนินการทางคณิตศาสตร์)
2. Comparison Operators (ตัวดำเนินการเปรียบเทียบ)
3. Logical Operators (ตัวดำเนินการตรรกะ)
4. Bitwise Operators (ตัวดำเนินการระดับบิต)
5. Assignment Operators (ตัวดำเนินการกำหนดค่า)
6. Ternary Operator (ตัวดำเนินการสามส่วน)
7. Increment และ Decrement Operators
8. Operator Precedence (ลำดับความสำคัญ)
9. Expressions ขั้นสูง
10. ตัวอย่างโค้ดสมบูรณ์
11. แบบฝึกหัดพร้อมเฉลย

---

## 1. Arithmetic Operators (ตัวดำเนินการทางคณิตศาสตร์)

### ตัวดำเนินการพื้นฐาน

```objc
#import <Foundation/Foundation.h>

int main(int argc, const char * argv[]) {
    @autoreleasepool {
        
        int a = 17;
        int b = 5;
        
        // ========== Arithmetic Operators ==========
        
        // + (บวก)
        int sum = a + b;
        NSLog(@"%d + %d = %d", a, b, sum);         // 17 + 5 = 22
        
        // - (ลบ)
        int diff = a - b;
        NSLog(@"%d - %d = %d", a, b, diff);        // 17 - 5 = 12
        
        // * (คูณ)
        int product = a * b;
        NSLog(@"%d * %d = %d", a, b, product);     // 17 * 5 = 85
        
        // / (หาร)
        int quotient = a / b;
        NSLog(@"%d / %d = %d", a, b, quotient);    // 17 / 5 = 3 (integer division!)
        
        // % (modulo - เศษจากการหาร)
        int remainder = a % b;
        NSLog(@"%d %% %d = %d", a, b, remainder);  // 17 % 5 = 2
        
        // - (unary minus - ติดลบ)
        int negative = -a;
        NSLog(@"-(%d) = %d", a, negative);          // -(17) = -17
        
        // + (unary plus - ติดบวก, ไม่ค่อยมีประโยชน์)
        int positive = +a;
        NSLog(@"+(%d) = %d", a, positive);          // +(17) = 17
        
        NSLog(@"\n--- Integer Division Gotcha ---");
        
        // ข้อควรระวัง: Integer Division ตัดทศนิยมทิ้ง (ไม่ปัดเศษ)
        NSLog(@"7 / 2 = %d (ไม่ใช่ 3.5!)", 7 / 2);    // 3
        NSLog(@"1 / 4 = %d (ไม่ใช่ 0.25!)", 1 / 4);   // 0
        NSLog(@"-7 / 2 = %d (truncates toward zero)", -7 / 2);  // -3 (ไม่ใช่ -4)
        
        // การแก้: Cast ก่อน divide
        double correctDiv = (double)7 / 2;
        NSLog(@"(double)7 / 2 = %.1f", correctDiv);  // 3.5
        
    }
    return 0;
}
```

### Arithmetic กับ Floating Point

```objc
#import <Foundation/Foundation.h>

int main(int argc, const char * argv[]) {
    @autoreleasepool {
        
        double x = 3.7;
        double y = 2.1;
        
        NSLog(@"x = %.1f, y = %.1f", x, y);
        NSLog(@"x + y = %.2f", x + y);    // 5.80
        NSLog(@"x - y = %.2f", x - y);    // 1.60
        NSLog(@"x * y = %.2f", x * y);    // 7.77
        NSLog(@"x / y = %.4f", x / y);    // 1.7619
        
        // % ไม่ทำงานกับ floating point โดยตรง
        // ต้องใช้ fmod() จาก math.h
        #include <math.h>
        double mod = fmod(x, y);
        NSLog(@"fmod(%.1f, %.1f) = %.4f", x, y, mod);  // 1.6000
        
        // Floating point precision issue
        NSLog(@"\n--- Floating Point Precision ---");
        double result = 0.1 + 0.2;
        NSLog(@"0.1 + 0.2 = %.17f", result);  // 0.30000000000000004!
        NSLog(@"0.1 + 0.2 == 0.3: %@", result == 0.3 ? @"YES" : @"NO");  // NO!
        
        // วิธีแก้: ใช้ epsilon comparison
        double epsilon = 1e-10;
        if (fabs(result - 0.3) < epsilon) {
            NSLog(@"0.1 + 0.2 ≈ 0.3 (ด้วย epsilon comparison)");
        }
        
        // math.h functions ที่ใช้บ่อย
        NSLog(@"\n--- Math Functions ---");
        NSLog(@"fabs(-5.5) = %.1f", fabs(-5.5));       // 5.5
        NSLog(@"sqrt(16) = %.1f", sqrt(16.0));          // 4.0
        NSLog(@"pow(2, 10) = %.0f", pow(2, 10));        // 1024
        NSLog(@"ceil(3.1) = %.1f", ceil(3.1));           // 4.0
        NSLog(@"floor(3.9) = %.1f", floor(3.9));         // 3.0
        NSLog(@"round(3.5) = %.1f", round(3.5));         // 4.0
        NSLog(@"round(3.4) = %.1f", round(3.4));         // 3.0
        NSLog(@"fmax(5, 3) = %.1f", fmax(5, 3));         // 5.0
        NSLog(@"fmin(5, 3) = %.1f", fmin(5, 3));         // 3.0
        NSLog(@"log(M_E) = %.4f", log(M_E));              // 1.0000
        NSLog(@"log10(1000) = %.4f", log10(1000));         // 3.0000
        NSLog(@"sin(M_PI/2) = %.4f", sin(M_PI/2));        // 1.0000
        NSLog(@"cos(0) = %.4f", cos(0));                   // 1.0000
        
    }
    return 0;
}
```

### Modulo (%) ใช้งานจริง

```objc
#import <Foundation/Foundation.h>

int main(int argc, const char * argv[]) {
    @autoreleasepool {
        
        NSLog(@"--- การใช้งาน Modulo จริง ---\n");
        
        // 1. ตรวจสอบเลขคู่/คี่
        for (int i = 1; i <= 10; i++) {
            NSLog(@"%d เป็น %@", i, (i % 2 == 0) ? @"เลขคู่" : @"เลขคี่");
        }
        
        NSLog(@"");
        
        // 2. วนซ้ำเป็นวงกลม (circular indexing)
        NSArray *days = @[@"อาทิตย์", @"จันทร์", @"อังคาร", @"พุธ", 
                          @"พฤหัสบดี", @"ศุกร์", @"เสาร์"];
        
        for (int i = 0; i < 14; i++) {
            NSLog(@"วันที่ %d: %@", i + 1, days[i % days.count]);
        }
        
        NSLog(@"");
        
        // 3. แบ่งกลุ่ม (grouping)
        NSLog(@"แบ่งเป็นกลุ่มๆ ละ 3:");
        for (int i = 1; i <= 12; i++) {
            if (i % 3 == 0) {
                NSLog(@"กลุ่มที่ %d จบ", i / 3);
            }
        }
        
        NSLog(@"");
        
        // 4. นาฬิกา - แปลง 24 ชั่วโมงเป็น 12 ชั่วโมง
        for (int hour = 0; hour <= 23; hour++) {
            int h12 = hour % 12;
            if (h12 == 0) h12 = 12;
            NSString *period = (hour < 12) ? @"AM" : @"PM";
            if (hour % 6 == 0) {  // แสดงทุก 6 ชั่วโมง
                NSLog(@"%02d:00 = %02d:00 %@", hour, h12, period);
            }
        }
        
        NSLog(@"");
        
        // 5. หาตัวหารร่วมมาก (GCD) ด้วย Euclidean algorithm
        int gcd(int a, int b) {
            while (b != 0) {
                int temp = b;
                b = a % b;
                a = temp;
            }
            return a;
        }
        
        int p = 48, q = 18;
        NSLog(@"GCD(%d, %d) = %d", p, q, gcd(p, q));  // 6
        
    }
    return 0;
}
```

---

## 2. Comparison Operators (ตัวดำเนินการเปรียบเทียบ)

### ตัวดำเนินการเปรียบเทียบทั้งหมด

```objc
#import <Foundation/Foundation.h>

int main(int argc, const char * argv[]) {
    @autoreleasepool {
        
        int a = 10, b = 20, c = 10;
        
        NSLog(@"a = %d, b = %d, c = %d", a, b, c);
        NSLog(@"");
        
        // == (เท่ากัน)
        NSLog(@"a == b: %@", (a == b) ? @"YES" : @"NO");   // NO
        NSLog(@"a == c: %@", (a == c) ? @"YES" : @"NO");   // YES
        
        // != (ไม่เท่ากัน)
        NSLog(@"a != b: %@", (a != b) ? @"YES" : @"NO");   // YES
        NSLog(@"a != c: %@", (a != c) ? @"YES" : @"NO");   // NO
        
        // < (น้อยกว่า)
        NSLog(@"a < b: %@", (a < b) ? @"YES" : @"NO");     // YES
        NSLog(@"b < a: %@", (b < a) ? @"YES" : @"NO");     // NO
        
        // > (มากกว่า)
        NSLog(@"a > b: %@", (a > b) ? @"YES" : @"NO");     // NO
        NSLog(@"b > a: %@", (b > a) ? @"YES" : @"NO");     // YES
        
        // <= (น้อยกว่าหรือเท่ากัน)
        NSLog(@"a <= b: %@", (a <= b) ? @"YES" : @"NO");   // YES
        NSLog(@"a <= c: %@", (a <= c) ? @"YES" : @"NO");   // YES
        NSLog(@"b <= a: %@", (b <= a) ? @"YES" : @"NO");   // NO
        
        // >= (มากกว่าหรือเท่ากัน)
        NSLog(@"b >= a: %@", (b >= a) ? @"YES" : @"NO");   // YES
        NSLog(@"a >= c: %@", (a >= c) ? @"YES" : @"NO");   // YES
        NSLog(@"a >= b: %@", (a >= b) ? @"YES" : @"NO");   // NO
        
        // ผลลัพธ์ของ comparison เป็น int (0 หรือ 1) หรือ BOOL (NO หรือ YES)
        int result = (a < b);
        NSLog(@"\nResult ของ (a < b) คือ: %d", result);  // 1
        
        BOOL boolResult = (a == c);
        NSLog(@"BOOL result ของ (a == c): %@", boolResult ? @"YES" : @"NO");
        
    }
    return 0;
}
```

### การเปรียบเทียบ NSString และ Objects

```objc
#import <Foundation/Foundation.h>

int main(int argc, const char * argv[]) {
    @autoreleasepool {
        
        // ========== String Comparison ==========
        NSLog(@"--- String Comparison ---");
        
        NSString *str1 = @"Hello";
        NSString *str2 = @"Hello";
        NSString *str3 = [NSString stringWithFormat:@"%@", @"Hello"];
        NSString *str4 = @"hello";
        
        // == เปรียบเทียบ pointer (memory address) ไม่ใช่เนื้อหา!
        NSLog(@"str1 == str2 (pointer): %@", (str1 == str2) ? @"YES" : @"NO");
        // อาจ YES เพราะ string interning (constant strings share pointer)
        NSLog(@"str1 == str3 (pointer): %@", (str1 == str3) ? @"YES" : @"NO");
        // อาจ NO เพราะ str3 สร้างใหม่จาก stringWithFormat
        
        // isEqualToString: เปรียบเทียบเนื้อหา (ถูกต้อง!)
        NSLog(@"str1 isEqualToString str2: %@", 
              [str1 isEqualToString:str2] ? @"YES" : @"NO");  // YES
        NSLog(@"str1 isEqualToString str3: %@", 
              [str1 isEqualToString:str3] ? @"YES" : @"NO");  // YES
        NSLog(@"str1 isEqualToString str4: %@", 
              [str1 isEqualToString:str4] ? @"YES" : @"NO");  // NO (case sensitive)
        
        // Case-insensitive comparison
        NSComparisonResult result = [str1 compare:str4 options:NSCaseInsensitiveSearch];
        NSLog(@"case-insensitive compare: %@", 
              result == NSOrderedSame ? @"equal" : @"not equal");  // equal
        
        // isEqual: (ทำงานกับทุก NSObject)
        NSLog(@"\n--- isEqual: ---");
        NSNumber *n1 = @42;
        NSNumber *n2 = @42;
        NSNumber *n3 = @43;
        
        NSLog(@"n1 == n2: %@", (n1 == n2) ? @"YES" : @"NO");  // อาจ YES (tagged pointer)
        NSLog(@"[n1 isEqual:n2]: %@", [n1 isEqual:n2] ? @"YES" : @"NO");  // YES
        NSLog(@"[n1 isEqual:n3]: %@", [n1 isEqual:n3] ? @"YES" : @"NO");  // NO
        
        // NSNumber comparison
        NSLog(@"\n--- NSNumber compare: ---");
        NSComparisonResult cmp = [n1 compare:n3];
        if (cmp == NSOrderedAscending) {
            NSLog(@"%@ < %@", n1, n3);
        } else if (cmp == NSOrderedDescending) {
            NSLog(@"%@ > %@", n1, n3);
        } else {
            NSLog(@"%@ == %@", n1, n3);
        }
        
        // Array/Dictionary comparison
        NSLog(@"\n--- Collection comparison ---");
        NSArray *arr1 = @[@1, @2, @3];
        NSArray *arr2 = @[@1, @2, @3];
        NSArray *arr3 = @[@1, @2, @4];
        
        NSLog(@"arr1 isEqual arr2: %@", [arr1 isEqual:arr2] ? @"YES" : @"NO");  // YES
        NSLog(@"arr1 isEqual arr3: %@", [arr1 isEqual:arr3] ? @"YES" : @"NO");  // NO
        
        NSDictionary *dict1 = @{@"a": @1, @"b": @2};
        NSDictionary *dict2 = @{@"a": @1, @"b": @2};
        NSDictionary *dict3 = @{@"a": @1, @"b": @3};
        
        NSLog(@"dict1 isEqual dict2: %@", [dict1 isEqual:dict2] ? @"YES" : @"NO");  // YES
        NSLog(@"dict1 isEqual dict3: %@", [dict1 isEqual:dict3] ? @"YES" : @"NO");  // NO
        
    }
    return 0;
}
```

---

## 3. Logical Operators (ตัวดำเนินการตรรกะ)

### ตัวดำเนินการตรรกะและ Truth Tables

```objc
#import <Foundation/Foundation.h>

int main(int argc, const char * argv[]) {
    @autoreleasepool {
        
        // ========== Logical AND (&&) ==========
        // คืนค่า YES ก็ต่อเมื่อทั้งสองเงื่อนไขเป็น YES
        NSLog(@"--- && (AND) Truth Table ---");
        NSLog(@"YES && YES = %@", (YES && YES) ? @"YES" : @"NO");  // YES
        NSLog(@"YES && NO  = %@", (YES && NO)  ? @"YES" : @"NO");  // NO
        NSLog(@"NO  && YES = %@", (NO  && YES) ? @"YES" : @"NO");  // NO
        NSLog(@"NO  && NO  = %@", (NO  && NO)  ? @"YES" : @"NO");  // NO
        
        // ========== Logical OR (||) ==========
        // คืนค่า YES ถ้าอย่างน้อยหนึ่งเงื่อนไขเป็น YES
        NSLog(@"\n--- || (OR) Truth Table ---");
        NSLog(@"YES || YES = %@", (YES || YES) ? @"YES" : @"NO");  // YES
        NSLog(@"YES || NO  = %@", (YES || NO)  ? @"YES" : @"NO");  // YES
        NSLog(@"NO  || YES = %@", (NO  || YES) ? @"YES" : @"NO");  // YES
        NSLog(@"NO  || NO  = %@", (NO  || NO)  ? @"YES" : @"NO");  // NO
        
        // ========== Logical NOT (!) ==========
        // กลับค่า boolean
        NSLog(@"\n--- ! (NOT) ---");
        NSLog(@"!YES = %@", !YES ? @"YES" : @"NO");  // NO
        NSLog(@"!NO  = %@", !NO  ? @"YES" : @"NO");  // YES
        NSLog(@"!0   = %@", !0   ? @"YES" : @"NO");  // YES
        NSLog(@"!1   = %@", !1   ? @"YES" : @"NO");  // NO
        NSLog(@"!42  = %@", !42  ? @"YES" : @"NO");  // NO (non-zero = truthy)
        
        // ========== Short-Circuit Evaluation ==========
        // && - ถ้าเงื่อนไขแรก NO จะไม่ evaluate เงื่อนไขที่สอง
        // || - ถ้าเงื่อนไขแรก YES จะไม่ evaluate เงื่อนไขที่สอง
        NSLog(@"\n--- Short-Circuit Evaluation ---");
        
        int callCount = 0;
        
        BOOL alwaysFalse(void) {
            callCount++;
            NSLog(@"alwaysFalse() called (count: %d)", callCount);
            return NO;
        }
        
        BOOL alwaysTrue(void) {
            callCount++;
            NSLog(@"alwaysTrue() called (count: %d)", callCount);
            return YES;
        }
        
        NSLog(@"\nNO && alwaysTrue():");
        callCount = 0;
        BOOL r1 = NO && alwaysTrue();  // alwaysTrue ไม่ถูกเรียก!
        NSLog(@"Result: %@, calls: %d", r1 ? @"YES" : @"NO", callCount);
        
        NSLog(@"\nYES || alwaysFalse():");
        callCount = 0;
        BOOL r2 = YES || alwaysFalse();  // alwaysFalse ไม่ถูกเรียก!
        NSLog(@"Result: %@, calls: %d", r2 ? @"YES" : @"NO", callCount);
        
        NSLog(@"\nYES && alwaysTrue():");
        callCount = 0;
        BOOL r3 = YES && alwaysTrue();  // alwaysTrue ถูกเรียก
        NSLog(@"Result: %@, calls: %d", r3 ? @"YES" : @"NO", callCount);
        
    }
    return 0;
}
```

### ตัวอย่างการใช้งานจริง

```objc
#import <Foundation/Foundation.h>

// ฟังก์ชัน validation ต่างๆ
BOOL isValidAge(NSInteger age) {
    return age >= 0 && age <= 150;
}

BOOL isValidEmail(NSString *email) {
    return email != nil 
        && email.length > 0 
        && [email containsString:@"@"]
        && [email containsString:@"."];
}

BOOL isValidPassword(NSString *password) {
    if (!password) return NO;
    if (password.length < 8) return NO;
    // อาจเพิ่ม complexity check ได้
    return YES;
}

BOOL canAccessAdminPanel(BOOL isLoggedIn, BOOL isAdmin, BOOL isSuperAdmin) {
    return isLoggedIn && (isAdmin || isSuperAdmin);
}

int main(int argc, const char * argv[]) {
    @autoreleasepool {
        
        NSLog(@"--- Validation Examples ---");
        
        // Age validation
        NSInteger ages[] = {-1, 0, 25, 100, 151};
        int ageCount = sizeof(ages) / sizeof(ages[0]);
        for (int i = 0; i < ageCount; i++) {
            NSLog(@"อายุ %ld: %@", (long)ages[i], 
                  isValidAge(ages[i]) ? @"ถูกต้อง" : @"ไม่ถูกต้อง");
        }
        
        NSLog(@"");
        
        // Email validation
        NSArray *emails = @[@"user@example.com", @"invalid", @"no.at.sign", 
                            @"@nodomain", @"user@domain.com"];
        for (NSString *email in emails) {
            NSLog(@"Email '%@': %@", email, 
                  isValidEmail(email) ? @"ถูกต้อง" : @"ไม่ถูกต้อง");
        }
        
        NSLog(@"");
        
        // Admin access
        struct { BOOL loggedIn, isAdmin, isSuperAdmin; NSString *role; } users[] = {
            {NO, NO, NO, @"Guest"},
            {YES, NO, NO, @"User"},
            {YES, YES, NO, @"Admin"},
            {YES, NO, YES, @"SuperAdmin"},
            {YES, YES, YES, @"Admin+Super"}
        };
        
        for (int i = 0; i < 5; i++) {
            BOOL canAccess = canAccessAdminPanel(users[i].loggedIn, 
                                                users[i].isAdmin, 
                                                users[i].isSuperAdmin);
            NSLog(@"%-15@ เข้า admin ได้: %@", users[i].role, 
                  canAccess ? @"YES" : @"NO");
        }
        
        NSLog(@"");
        
        // Nil-safe operations ด้วย &&
        NSString *str = nil;
        NSUInteger safeLength = (str != nil) ? str.length : 0;
        NSLog(@"safeLength of nil: %lu", (unsigned long)safeLength);
        
        // Nil guard pattern
        NSString *name = nil;
        if (name && [name isEqualToString:@"admin"]) {
            NSLog(@"Is admin");  // ไม่ถูกเรียกเพราะ name เป็น nil
        } else {
            NSLog(@"Not admin (name is nil: %@)", name == nil ? @"YES" : @"NO");
        }
        
    }
    return 0;
}
```

---

## 4. Bitwise Operators (ตัวดำเนินการระดับบิต)

### ตัวดำเนินการระดับบิตทั้งหมด

```objc
#import <Foundation/Foundation.h>

// Helper function: แสดงเลขฐาน 2
NSString *toBinary(int n, int bits) {
    NSMutableString *result = [NSMutableString string];
    for (int i = bits - 1; i >= 0; i--) {
        [result appendFormat:@"%d", (n >> i) & 1];
        if (i % 4 == 0 && i > 0) [result appendString:@" "];
    }
    return result;
}

int main(int argc, const char * argv[]) {
    @autoreleasepool {
        
        int a = 60;   // 0011 1100
        int b = 13;   // 0000 1101
        
        NSLog(@"a = %d = %@", a, toBinary(a, 8));
        NSLog(@"b = %d = %@", b, toBinary(b, 8));
        NSLog(@"");
        
        // ========== & (Bitwise AND) ==========
        // 1 & 1 = 1, ทั้งหมดอื่น = 0
        int andResult = a & b;   // 0000 1100 = 12
        NSLog(@"a & b = %d = %@", andResult, toBinary(andResult, 8));
        
        // ========== | (Bitwise OR) ==========
        // 0 | 0 = 0, ทั้งหมดอื่น = 1
        int orResult = a | b;    // 0011 1101 = 61
        NSLog(@"a | b = %d = %@", orResult, toBinary(orResult, 8));
        
        // ========== ^ (Bitwise XOR) ==========
        // เหมือนกัน = 0, ต่างกัน = 1
        int xorResult = a ^ b;   // 0011 0001 = 49
        NSLog(@"a ^ b = %d = %@", xorResult, toBinary(xorResult, 8));
        
        // ========== ~ (Bitwise NOT/Complement) ==========
        // กลับทุก bit
        int notA = ~a;           // 1100 0011 = -61 (two's complement)
        NSLog(@"~a = %d = %@", notA, toBinary(notA, 8));
        
        // ========== << (Left Shift) ==========
        // เลื่อน bits ไปซ้าย n ตำแหน่ง (คูณด้วย 2^n)
        int leftShift = a << 2;  // 1111 0000 = 240 (60 * 4)
        NSLog(@"a << 2 = %d = %@ (a * 4)", leftShift, toBinary(leftShift, 8));
        
        // ========== >> (Right Shift) ==========
        // เลื่อน bits ไปขวา n ตำแหน่ง (หารด้วย 2^n)
        int rightShift = a >> 2; // 0000 1111 = 15 (60 / 4)
        NSLog(@"a >> 2 = %d = %@ (a / 4)", rightShift, toBinary(rightShift, 8));
        
        NSLog(@"\n--- Bit Manipulation Examples ---");
        
        // ตรวจสอบว่า bit n ถูก set หรือไม่
        int num = 0b10110100;  // binary literal (C99)
        for (int i = 7; i >= 0; i--) {
            BOOL bitSet = (num >> i) & 1;
            NSLog(@"bit %d of %d: %@", i, num, bitSet ? @"1" : @"0");
        }
        
        NSLog(@"");
        
        // Set bit n
        int setBit(int n, int pos) {
            return n | (1 << pos);
        }
        
        // Clear bit n
        int clearBit(int n, int pos) {
            return n & ~(1 << pos);
        }
        
        // Toggle bit n
        int toggleBit(int n, int pos) {
            return n ^ (1 << pos);
        }
        
        int value = 0b00000000;
        NSLog(@"Initial: %@ (%d)", toBinary(value, 8), value);
        
        value = setBit(value, 3);  // set bit 3
        NSLog(@"Set bit 3: %@ (%d)", toBinary(value, 8), value);
        
        value = setBit(value, 5);  // set bit 5
        NSLog(@"Set bit 5: %@ (%d)", toBinary(value, 8), value);
        
        value = clearBit(value, 3);  // clear bit 3
        NSLog(@"Clear bit 3: %@ (%d)", toBinary(value, 8), value);
        
        value = toggleBit(value, 0);  // toggle bit 0
        NSLog(@"Toggle bit 0: %@ (%d)", toBinary(value, 8), value);
        
    }
    return 0;
}
```

### Bitwise ในการใช้งานจริง: Flags และ Permissions

```objc
#import <Foundation/Foundation.h>

// ========== Permission Flags ==========
typedef NS_OPTIONS(NSUInteger, FilePermission) {
    FilePermissionNone    = 0,
    FilePermissionRead    = 1 << 0,   // 0001 = 1
    FilePermissionWrite   = 1 << 1,   // 0010 = 2
    FilePermissionExecute = 1 << 2,   // 0100 = 4
    FilePermissionAll     = FilePermissionRead | FilePermissionWrite | FilePermissionExecute
};

// ========== Interface ==========
@interface FileSystem : NSObject
- (BOOL)hasPermission:(FilePermission)permission forFile:(NSString *)filename;
- (void)grantPermission:(FilePermission)permission forFile:(NSString *)filename;
- (void)revokePermission:(FilePermission)permission forFile:(NSString *)filename;
- (void)printPermissions:(NSString *)filename;
@end

@implementation FileSystem {
    NSMutableDictionary *_filePermissions;
}

- (instancetype)init {
    self = [super init];
    if (self) {
        _filePermissions = [NSMutableDictionary dictionary];
    }
    return self;
}

- (FilePermission)permissionsForFile:(NSString *)filename {
    NSNumber *perms = _filePermissions[filename];
    return perms ? [perms unsignedIntegerValue] : FilePermissionNone;
}

- (BOOL)hasPermission:(FilePermission)permission forFile:(NSString *)filename {
    FilePermission current = [self permissionsForFile:filename];
    return (current & permission) == permission;  // bitwise AND เพื่อตรวจสอบ
}

- (void)grantPermission:(FilePermission)permission forFile:(NSString *)filename {
    FilePermission current = [self permissionsForFile:filename];
    _filePermissions[filename] = @(current | permission);  // bitwise OR เพื่อเพิ่ม
}

- (void)revokePermission:(FilePermission)permission forFile:(NSString *)filename {
    FilePermission current = [self permissionsForFile:filename];
    _filePermissions[filename] = @(current & ~permission);  // AND NOT เพื่อลบ
}

- (void)printPermissions:(NSString *)filename {
    FilePermission perms = [self permissionsForFile:filename];
    NSLog(@"Permissions for '%@':", filename);
    NSLog(@"  Read:    %@", (perms & FilePermissionRead) ? @"✓" : @"✗");
    NSLog(@"  Write:   %@", (perms & FilePermissionWrite) ? @"✓" : @"✗");
    NSLog(@"  Execute: %@", (perms & FilePermissionExecute) ? @"✓" : @"✗");
    NSLog(@"  (raw: %lu)", (unsigned long)perms);
}

@end

int main(int argc, const char * argv[]) {
    @autoreleasepool {
        
        FileSystem *fs = [[FileSystem alloc] init];
        NSString *filename = @"config.txt";
        
        NSLog(@"=== File Permission System ===\n");
        
        // เริ่มต้น
        [fs printPermissions:filename];
        
        // ให้ permission อ่าน
        [fs grantPermission:FilePermissionRead forFile:filename];
        NSLog(@"\nหลัง grant Read:");
        [fs printPermissions:filename];
        
        // ให้ permission เขียนและรัน
        [fs grantPermission:FilePermissionWrite | FilePermissionExecute forFile:filename];
        NSLog(@"\nหลัง grant Write+Execute:");
        [fs printPermissions:filename];
        
        // ยกเลิก execute
        [fs revokePermission:FilePermissionExecute forFile:filename];
        NSLog(@"\nหลัง revoke Execute:");
        [fs printPermissions:filename];
        
        NSLog(@"\nHas Read: %@", [fs hasPermission:FilePermissionRead forFile:filename] ? @"YES" : @"NO");
        NSLog(@"Has Execute: %@", [fs hasPermission:FilePermissionExecute forFile:filename] ? @"YES" : @"NO");
        NSLog(@"Has All: %@", [fs hasPermission:FilePermissionAll forFile:filename] ? @"YES" : @"NO");
        
    }
    return 0;
}
```

---

## 5. Assignment Operators (ตัวดำเนินการกำหนดค่า)

### ตัวดำเนินการ Assignment ทั้งหมด

```objc
#import <Foundation/Foundation.h>

int main(int argc, const char * argv[]) {
    @autoreleasepool {
        
        int x = 10;
        NSLog(@"Initial x = %d", x);
        
        // ========== Basic Assignment (=) ==========
        x = 20;
        NSLog(@"x = 20 -> x = %d", x);
        
        // ========== Compound Assignment Operators ==========
        
        x = 10;  // reset
        
        // += (บวกและกำหนด)
        x += 5;    // x = x + 5
        NSLog(@"x += 5 -> x = %d", x);  // 15
        
        // -= (ลบและกำหนด)
        x -= 3;    // x = x - 3
        NSLog(@"x -= 3 -> x = %d", x);  // 12
        
        // *= (คูณและกำหนด)
        x *= 2;    // x = x * 2
        NSLog(@"x *= 2 -> x = %d", x);  // 24
        
        // /= (หารและกำหนด)
        x /= 4;    // x = x / 4
        NSLog(@"x /= 4 -> x = %d", x);  // 6
        
        // %= (modulo และกำหนด)
        x %= 4;    // x = x % 4
        NSLog(@"x %%= 4 -> x = %d", x);  // 2
        
        // ========== Bitwise Assignment ==========
        
        x = 0b11110000;  // 240
        NSLog(@"\nBitwise Assignment:");
        NSLog(@"Initial x = %d (binary: ...11110000)", x);
        
        // &= (AND กำหนด)
        x &= 0b10101010;  // 170
        NSLog(@"x &= 0xAA -> x = %d", x);  // 10100000 = 160
        
        x = 0b00001111;  // 15
        
        // |= (OR กำหนด)
        x |= 0b11110000;  // 240
        NSLog(@"x |= 0xF0 -> x = %d", x);  // 11111111 = 255
        
        x = 0b11001100;  // 204
        
        // ^= (XOR กำหนด)
        x ^= 0b11111111;  // 255
        NSLog(@"x ^= 0xFF -> x = %d", x);  // 00110011 = 51
        
        // <<= (Left shift กำหนด)
        x = 1;
        x <<= 3;  // x = x << 3
        NSLog(@"1 <<= 3 -> x = %d", x);  // 8
        
        // >>= (Right shift กำหนด)
        x = 64;
        x >>= 2;  // x = x >> 2
        NSLog(@"64 >>= 2 -> x = %d", x);  // 16
        
        // ========== Multiple Assignment ==========
        int a, b, c;
        a = b = c = 5;  // chain assignment (ขวาไปซ้าย)
        NSLog(@"\na = b = c = 5: a=%d, b=%d, c=%d", a, b, c);
        
        // ========== Assignment ใน Expression ==========
        int value;
        int result = (value = 42) * 2;  // ใช้ได้แต่ไม่แนะนำ
        NSLog(@"result = (value=42)*2: %d, value=%d", result, value);
        
    }
    return 0;
}
```

### Compound Assignment ในชีวิตจริง

```objc
#import <Foundation/Foundation.h>

@interface ShoppingCart : NSObject
@property (nonatomic, assign) double totalPrice;
@property (nonatomic, assign) NSInteger itemCount;
@property (nonatomic, assign) double discountPercent;

- (void)addItem:(double)price;
- (void)removeItem:(double)price;
- (void)applyDiscount:(double)percent;
- (double)finalPrice;
- (void)printSummary;
@end

@implementation ShoppingCart

- (instancetype)init {
    self = [super init];
    if (self) {
        _totalPrice = 0.0;
        _itemCount = 0;
        _discountPercent = 0.0;
    }
    return self;
}

- (void)addItem:(double)price {
    self.totalPrice += price;      // += compound assignment
    self.itemCount++;               // increment
    NSLog(@"เพิ่มสินค้า ราคา %.2f บาท", price);
}

- (void)removeItem:(double)price {
    if (self.totalPrice >= price) {
        self.totalPrice -= price;   // -= compound assignment
        self.itemCount--;           // decrement
        NSLog(@"ลบสินค้า ราคา %.2f บาท", price);
    }
}

- (void)applyDiscount:(double)percent {
    self.discountPercent += percent;  // += compound assignment
    NSLog(@"ใช้ส่วนลด %.1f%%", percent);
}

- (double)finalPrice {
    double discountAmount = self.totalPrice * (self.discountPercent / 100.0);
    return self.totalPrice - discountAmount;
}

- (void)printSummary {
    NSLog(@"=== สรุปตะกร้าสินค้า ===");
    NSLog(@"จำนวนสินค้า: %ld ชิ้น", (long)self.itemCount);
    NSLog(@"ราคาก่อนลด: %.2f บาท", self.totalPrice);
    if (self.discountPercent > 0) {
        NSLog(@"ส่วนลด: %.1f%% (%.2f บาท)", 
              self.discountPercent, 
              self.totalPrice * self.discountPercent / 100.0);
        NSLog(@"ราคาหลังลด: %.2f บาท", [self finalPrice]);
    }
    NSLog(@"========================");
}

@end

int main(int argc, const char * argv[]) {
    @autoreleasepool {
        
        ShoppingCart *cart = [[ShoppingCart alloc] init];
        
        [cart addItem:150.0];
        [cart addItem:299.0];
        [cart addItem:85.0];
        [cart addItem:499.0];
        [cart removeItem:85.0];
        [cart applyDiscount:10.0];
        [cart applyDiscount:5.0];  // รวมส่วนลด 15%
        
        NSLog(@"");
        [cart printSummary];
        
    }
    return 0;
}
```

---

## 6. Ternary Operator (ตัวดำเนินการสามส่วน)

### Syntax และการใช้งาน

```objc
#import <Foundation/Foundation.h>

int main(int argc, const char * argv[]) {
    @autoreleasepool {
        
        // ========== Ternary Operator ==========
        // Syntax: condition ? valueIfTrue : valueIfFalse
        
        int x = 10;
        
        // แบบ if-else ปกติ
        NSString *sign1;
        if (x > 0) {
            sign1 = @"บวก";
        } else if (x < 0) {
            sign1 = @"ลบ";
        } else {
            sign1 = @"ศูนย์";
        }
        NSLog(@"if-else: %@", sign1);
        
        // แบบ ternary (compact)
        NSString *sign2 = (x > 0) ? @"บวก" : (x < 0) ? @"ลบ" : @"ศูนย์";
        NSLog(@"ternary: %@", sign2);
        
        // ========== ตัวอย่างการใช้งาน ==========
        
        // 1. แสดงผลแบบ conditional
        int score = 75;
        NSLog(@"คะแนน %d: %@", score, score >= 50 ? @"ผ่าน" : @"ไม่ผ่าน");
        
        // 2. ค่า default (nil coalescing pattern)
        NSString *name = nil;
        NSString *displayName = name ? name : @"ไม่ระบุ";
        NSLog(@"ชื่อ: %@", displayName);
        
        // 3. ใน NSLog โดยตรง
        BOOL isLoggedIn = YES;
        NSLog(@"สถานะ: %@", isLoggedIn ? @"เข้าสู่ระบบแล้ว" : @"ยังไม่เข้าสู่ระบบ");
        
        // 4. ใน expression
        int a = 5, b = 10;
        int max = (a > b) ? a : b;
        int min = (a < b) ? a : b;
        NSLog(@"max(%d, %d) = %d", a, b, max);
        NSLog(@"min(%d, %d) = %d", a, b, min);
        
        // 5. Nested ternary (ระวัง! อาจอ่านยาก)
        int num = 0;
        NSString *numType = (num > 0) ? @"บวก" : 
                            (num < 0) ? @"ลบ" : 
                                        @"ศูนย์";
        NSLog(@"numType: %@", numType);
        
        // 6. ใน Array literal
        BOOL isDarkMode = YES;
        NSArray *colors = @[
            isDarkMode ? @"#FFFFFF" : @"#000000",  // text color
            isDarkMode ? @"#000000" : @"#FFFFFF"   // background color
        ];
        NSLog(@"text: %@, bg: %@", colors[0], colors[1]);
        
        // ========== ตัวดำเนินการ ?: (เฉพาะ GCC/Clang extension) ==========
        // condition ?: valueIfFalse (ถ้า condition เป็น truthy ส่งคืน condition เอง)
        NSString *maybeNil = nil;
        NSString *safeValue = maybeNil ?: @"default value";
        NSLog(@"safeValue: %@", safeValue);  // default value
        
        NSString *notNil = @"I exist";
        NSString *safeValue2 = notNil ?: @"default value";
        NSLog(@"safeValue2: %@", safeValue2);  // I exist
        
    }
    return 0;
}
```

---

## 7. Increment และ Decrement Operators

### Pre vs Post Increment/Decrement

```objc
#import <Foundation/Foundation.h>

int main(int argc, const char * argv[]) {
    @autoreleasepool {
        
        // ========== Increment (++) ==========
        
        // Pre-increment: เพิ่มค่าก่อน แล้วค่อยส่งคืน
        int a = 5;
        int preInc = ++a;  // a กลายเป็น 6 ก่อน แล้ว preInc = 6
        NSLog(@"Pre-increment: a=%d, preInc=%d", a, preInc);  // a=6, preInc=6
        
        // Post-increment: ส่งคืนค่าเดิมก่อน แล้วค่อยเพิ่ม
        int b = 5;
        int postInc = b++;  // postInc = 5 (ค่าเดิม), แล้ว b กลายเป็น 6
        NSLog(@"Post-increment: b=%d, postInc=%d", b, postInc);  // b=6, postInc=5
        
        NSLog(@"");
        
        // ========== Decrement (--) ==========
        
        // Pre-decrement
        int c = 5;
        int preDec = --c;  // c กลายเป็น 4 ก่อน แล้ว preDec = 4
        NSLog(@"Pre-decrement: c=%d, preDec=%d", c, preDec);  // c=4, preDec=4
        
        // Post-decrement
        int d = 5;
        int postDec = d--;  // postDec = 5, แล้ว d กลายเป็น 4
        NSLog(@"Post-decrement: d=%d, postDec=%d", d, postDec);  // d=4, postDec=5
        
        NSLog(@"");
        
        // ========== ในบริบทต่างๆ ==========
        
        // ในการวนซ้ำ (ทั้ง ++ และ -- เหมือนกัน)
        NSLog(@"Forward loop:");
        for (int i = 0; i < 5; i++) {     // หรือ ++i (ผลลัพธ์เหมือนกัน)
            NSLog(@"  i = %d", i);
        }
        
        NSLog(@"Backward loop:");
        for (int i = 4; i >= 0; i--) {    // หรือ --i
            NSLog(@"  i = %d", i);
        }
        
        NSLog(@"");
        
        // ตัวอย่างที่น่าระวัง
        int e = 5;
        int result = e++ + e++;  // undefined behavior! อย่าทำแบบนี้
        NSLog(@"DANGER - e++ + e++ = %d, e = %d (undefined!)", result, e);
        
        // ควรทำแบบนี้แทน
        int f = 5;
        int r1 = f + (f + 1);  // ชัดเจนและปลอดภัย
        NSLog(@"Safe: f + (f+1) = %d", r1);
        
        NSLog(@"");
        
        // ========== ++ และ -- กับ Pointers ==========
        
        // ++/-- ยังใช้กับ pointer ได้ (เลื่อน pointer ไปยัง element ถัดไป)
        int arr[] = {10, 20, 30, 40, 50};
        int *ptr = arr;
        
        NSLog(@"Pointer increment:");
        NSLog(@"*ptr = %d", *ptr);     // 10
        ptr++;
        NSLog(@"After ptr++: *ptr = %d", *ptr);  // 20
        ptr++;
        NSLog(@"After ptr++: *ptr = %d", *ptr);  // 30
        
        // ========== ++ ใน NSMutableArray ==========
        NSMutableArray *counters = [NSMutableArray array];
        for (int i = 0; i < 5; i++) {
            [counters addObject:@(i)];
        }
        
        // เพิ่มค่าใน array
        for (NSUInteger i = 0; i < counters.count; i++) {
            NSInteger val = [counters[i] integerValue];
            val++;
            counters[i] = @(val);
        }
        NSLog(@"Incremented counters: %@", counters);
        
    }
    return 0;
}
```

---

## 8. Operator Precedence (ลำดับความสำคัญ)

### ตารางลำดับความสำคัญ

```objc
#import <Foundation/Foundation.h>

int main(int argc, const char * argv[]) {
    @autoreleasepool {
        
        // ลำดับความสำคัญ (สูงไปต่ำ):
        // 1. () [] . ->             Postfix
        // 2. ++ -- ! ~ (type) * & sizeof  Unary
        // 3. * / %                  Multiplicative
        // 4. + -                    Additive
        // 5. << >>                  Shift
        // 6. < > <= >=              Relational
        // 7. == !=                  Equality
        // 8. &                      Bitwise AND
        // 9. ^                      Bitwise XOR
        // 10. |                     Bitwise OR
        // 11. &&                    Logical AND
        // 12. ||                    Logical OR
        // 13. ?:                    Ternary
        // 14. = += -= *= /= %= &= ^= |= <<= >>=  Assignment
        
        NSLog(@"--- Operator Precedence Examples ---\n");
        
        // ตัวอย่าง 1: คณิตศาสตร์พื้นฐาน
        int r1 = 2 + 3 * 4;      // * ก่อน +
        NSLog(@"2 + 3 * 4 = %d (ไม่ใช่ %d)", r1, (2+3)*4);  // 14 ไม่ใช่ 20
        
        int r2 = (2 + 3) * 4;    // () เปลี่ยน precedence
        NSLog(@"(2 + 3) * 4 = %d", r2);  // 20
        
        // ตัวอย่าง 2: Shift กับ Addition
        int r3 = 1 + 2 << 3;    // + ก่อน <<
        NSLog(@"1 + 2 << 3 = %d", r3);   // (1+2)<<3 = 3<<3 = 24
        
        int r4 = 1 + (2 << 3);  // << ก่อน +
        NSLog(@"1 + (2 << 3) = %d", r4); // 1 + 16 = 17
        
        // ตัวอย่าง 3: Comparison กับ Bitwise
        int a = 5, b = 3;
        BOOL r5 = a & b == 1;   // == ก่อน &! เหมือน a & (b == 1) = a & 0 = 0
        BOOL r6 = (a & b) == 1; // แก้ด้วย parentheses: (5 & 3) = 1, 1 == 1 = YES
        
        NSLog(@"a & b == 1 (ผิด): %@", r5 ? @"YES" : @"NO");   // NO (a & (b==1) = 5&0 = 0)
        NSLog(@"(a & b) == 1 (ถูก): %@", r6 ? @"YES" : @"NO"); // YES
        
        // ตัวอย่าง 4: Logical operators
        BOOL p = YES, q = NO, r = YES;
        BOOL r7 = p || q && r;   // && ก่อน ||: p || (q && r) = YES || NO = YES
        BOOL r8 = (p || q) && r; // (YES || NO) && YES = YES
        
        NSLog(@"p || q && r = %@", r7 ? @"YES" : @"NO");    // YES
        NSLog(@"(p || q) && r = %@", r8 ? @"YES" : @"NO");  // YES
        
        // เปลี่ยนค่า
        p = NO; q = YES; r = YES;
        r7 = p || q && r;   // p || (q && r) = NO || YES = YES
        r8 = (p || q) && r; // (NO || YES) && YES = YES
        
        p = YES; q = NO; r = NO;
        r7 = p || q && r;   // YES || (NO && NO) = YES || NO = YES
        r8 = (p || q) && r; // (YES || NO) && NO = YES && NO = NO
        
        NSLog(@"(YES || NO) && NO = %@", r8 ? @"YES" : @"NO");  // NO!
        NSLog(@"YES || NO && NO  = %@", r7 ? @"YES" : @"NO");   // YES
        
        // ตัวอย่าง 5: Assignment กับ Comparison (common mistake)
        int x = 5;
        if (x = 10) {  // Assignment ไม่ใช่ Comparison! กำหนด x = 10 แล้ว test ว่า 10 != 0
            NSLog(@"x = %d (this will always execute!)", x);
        }
        
        if (x == 10) {  // Comparison ที่ถูกต้อง
            NSLog(@"x == 10 is true");
        }
        
        // Yoda Conditions - เพื่อป้องกัน = mistake
        // if (10 == x) {...}  // ถ้าเขียน = จะเป็น compile error!
        
        // ตัวอย่าง 6: Postfix สูงกว่า Prefix
        int n = 5;
        int r9 = -n++;    // -(n++) = -(5) then n becomes 6
        NSLog(@"-n++ with n=5: result=%d, n=%d", r9, n);  // result=-5, n=6
        
        n = 5;
        int r10 = (-n)++; // ERROR: (-n) ไม่ใช่ lvalue, ไม่สามารถ ++ ได้
        // ใช้ได้ถ้า declare variable ก่อน:
        int negN = -n;
        negN++;
        NSLog(@"(-5)++: %d", negN);  // -4
        
    }
    return 0;
}
```

---

## 9. Expressions ขั้นสูง

### Comma Operator และ Complex Expressions

```objc
#import <Foundation/Foundation.h>

int main(int argc, const char * argv[]) {
    @autoreleasepool {
        
        NSLog(@"--- Complex Expressions ---\n");
        
        // ========== Comma Operator ==========
        // ประเมินทั้งสอง expression, ส่งคืนค่าของ expression ขวาสุด
        int x = 0;
        int result = (x = 1, x + 5);  // x ถูกตั้งเป็น 1, result = 1 + 5 = 6
        NSLog(@"(x=1, x+5) = %d, x=%d", result, x);
        
        // Comma ใน for loop (multiple initialization หรือ update)
        NSLog(@"\nMultiple vars in for loop:");
        for (int i = 0, j = 10; i < 5; i++, j -= 2) {
            NSLog(@"  i=%d, j=%d", i, j);
        }
        
        // ========== Compound Conditions ==========
        
        double temperature = 25.0;
        double humidity = 60.0;
        BOOL isRaining = NO;
        
        // Complex condition
        BOOL niceWeather = temperature >= 20 && temperature <= 30 &&
                           humidity < 80 && !isRaining;
        
        NSLog(@"\nอากาศดีไหม: %@", niceWeather ? @"ดี" : @"ไม่ดี");
        
        // ========== Expression ใน NSString ==========
        int a = 10, b = 3;
        NSLog(@"%d + %d * %d = %d (ไม่ใช่ %d)", 
              a, b, b, a + b * b, (a + b) * b);
        
        // ========== ternary chain เพื่อ grade ==========
        NSInteger score = 78;
        NSString *grade = 
            score >= 90 ? @"A" :
            score >= 80 ? @"B" :
            score >= 70 ? @"C" :
            score >= 60 ? @"D" : @"F";
        NSLog(@"คะแนน %ld -> เกรด %@", (long)score, grade);
        
        // ========== Overflow และ Wrap-around ==========
        NSLog(@"\n--- Overflow ---");
        
        // signed overflow (undefined behavior!)
        // int maxInt = INT_MAX;
        // int overflow = maxInt + 1;  // Undefined behavior!
        
        // unsigned overflow (well-defined: wraps around)
        unsigned int maxUInt = UINT_MAX;  // 4294967295
        unsigned int wrapped = maxUInt + 1;  // 0
        NSLog(@"UINT_MAX + 1 = %u (wraps to 0)", wrapped);
        
        unsigned int wrapped2 = 0 - 1;  // UINT_MAX
        NSLog(@"0 - 1 (unsigned) = %u", wrapped2);
        
        // ป้องกัน overflow
        NSInteger bigNum = NSIntegerMax;
        BOOL willOverflow = (bigNum > NSIntegerMax - 10);
        if (willOverflow) {
            NSLog(@"Will overflow! Cannot add 10 to NSIntegerMax");
        } else {
            bigNum += 10;
            NSLog(@"Result: %ld", (long)bigNum);
        }
        
    }
    return 0;
}
```

---

## 10. ตัวอย่างโค้ดสมบูรณ์: Calculator Class

```objc
// File: Calculator.h และ Calculator.m
// เครื่องคิดเลขที่แสดงการใช้ Operators ทั้งหมด

#import <Foundation/Foundation.h>

// ========== Calculator Interface ==========

typedef NS_ENUM(NSInteger, CalculatorOperation) {
    CalculatorOperationAdd,
    CalculatorOperationSubtract,
    CalculatorOperationMultiply,
    CalculatorOperationDivide,
    CalculatorOperationModulo,
    CalculatorOperationPower,
};

@interface Calculator : NSObject

@property (nonatomic, readonly) double result;
@property (nonatomic, strong, readonly) NSMutableArray *history;

// Basic arithmetic
- (double)add:(double)a to:(double)b;
- (double)subtract:(double)b from:(double)a;
- (double)multiply:(double)a by:(double)b;
- (double)divide:(double)a by:(double)b;
- (double)modulo:(double)a by:(double)b;
- (double)power:(double)base exponent:(double)exp;

// Bitwise operations
- (NSInteger)bitwiseAnd:(NSInteger)a with:(NSInteger)b;
- (NSInteger)bitwiseOr:(NSInteger)a with:(NSInteger)b;
- (NSInteger)bitwiseXor:(NSInteger)a with:(NSInteger)b;
- (NSInteger)bitwiseNot:(NSInteger)n;
- (NSInteger)leftShift:(NSInteger)n by:(NSInteger)bits;
- (NSInteger)rightShift:(NSInteger)n by:(NSInteger)bits;

// Utility
- (void)printHistory;
- (void)clearHistory;
- (BOOL)isEqual:(double)a to:(double)b epsilon:(double)epsilon;

@end

// ========== Calculator Implementation ==========

@implementation Calculator {
    double _result;
    NSMutableArray *_history;
}

- (instancetype)init {
    self = [super init];
    if (self) {
        _result = 0.0;
        _history = [NSMutableArray array];
    }
    return self;
}

- (double)result { return _result; }
- (NSMutableArray *)history { return _history; }

- (void)recordOperation:(NSString *)operation result:(double)result {
    NSString *entry = [NSString stringWithFormat:@"%@ = %.4g", operation, result];
    [_history addObject:entry];
    _result = result;
}

- (double)add:(double)a to:(double)b {
    double r = a + b;
    [self recordOperation:[NSString stringWithFormat:@"%.4g + %.4g", a, b] result:r];
    return r;
}

- (double)subtract:(double)b from:(double)a {
    double r = a - b;
    [self recordOperation:[NSString stringWithFormat:@"%.4g - %.4g", a, b] result:r];
    return r;
}

- (double)multiply:(double)a by:(double)b {
    double r = a * b;
    [self recordOperation:[NSString stringWithFormat:@"%.4g × %.4g", a, b] result:r];
    return r;
}

- (double)divide:(double)a by:(double)b {
    if (b == 0.0) {
        NSLog(@"Error: หารด้วยศูนย์ไม่ได้!");
        return NAN;
    }
    double r = a / b;
    [self recordOperation:[NSString stringWithFormat:@"%.4g ÷ %.4g", a, b] result:r];
    return r;
}

- (double)modulo:(double)a by:(double)b {
    if (b == 0.0) {
        NSLog(@"Error: หารด้วยศูนย์ไม่ได้!");
        return NAN;
    }
    double r = fmod(a, b);
    [self recordOperation:[NSString stringWithFormat:@"%.4g %% %.4g", a, b] result:r];
    return r;
}

- (double)power:(double)base exponent:(double)exp {
    double r = pow(base, exp);
    [self recordOperation:[NSString stringWithFormat:@"%.4g ^ %.4g", base, exp] result:r];
    return r;
}

- (NSInteger)bitwiseAnd:(NSInteger)a with:(NSInteger)b {
    NSInteger r = a & b;
    NSString *op = [NSString stringWithFormat:@"%ld & %ld", (long)a, (long)b];
    [_history addObject:[NSString stringWithFormat:@"%@ = %ld", op, (long)r]];
    return r;
}

- (NSInteger)bitwiseOr:(NSInteger)a with:(NSInteger)b {
    NSInteger r = a | b;
    [_history addObject:[NSString stringWithFormat:@"%ld | %ld = %ld", (long)a, (long)b, (long)r]];
    return r;
}

- (NSInteger)bitwiseXor:(NSInteger)a with:(NSInteger)b {
    NSInteger r = a ^ b;
    [_history addObject:[NSString stringWithFormat:@"%ld ^ %ld = %ld", (long)a, (long)b, (long)r]];
    return r;
}

- (NSInteger)bitwiseNot:(NSInteger)n {
    NSInteger r = ~n;
    [_history addObject:[NSString stringWithFormat:@"~%ld = %ld", (long)n, (long)r]];
    return r;
}

- (NSInteger)leftShift:(NSInteger)n by:(NSInteger)bits {
    NSInteger r = n << bits;
    [_history addObject:[NSString stringWithFormat:@"%ld << %ld = %ld", (long)n, (long)bits, (long)r]];
    return r;
}

- (NSInteger)rightShift:(NSInteger)n by:(NSInteger)bits {
    NSInteger r = n >> bits;
    [_history addObject:[NSString stringWithFormat:@"%ld >> %ld = %ld", (long)n, (long)bits, (long)r]];
    return r;
}

- (BOOL)isEqual:(double)a to:(double)b epsilon:(double)epsilon {
    return fabs(a - b) < epsilon;
}

- (void)printHistory {
    NSLog(@"=== ประวัติการคำนวณ ===");
    if (_history.count == 0) {
        NSLog(@"(ไม่มีประวัติ)");
    } else {
        for (NSUInteger i = 0; i < _history.count; i++) {
            NSLog(@"%2lu. %@", (unsigned long)(i + 1), _history[i]);
        }
    }
    NSLog(@"====================");
}

- (void)clearHistory {
    [_history removeAllObjects];
    _result = 0.0;
}

@end

// ========== Main Program ==========

int main(int argc, const char * argv[]) {
    @autoreleasepool {
        
        Calculator *calc = [[Calculator alloc] init];
        
        NSLog(@"===== เครื่องคิดเลข Objective-C =====\n");
        
        // Arithmetic operations
        NSLog(@"--- การคำนวณทางคณิตศาสตร์ ---");
        [calc add:15 to:7];
        [calc subtract:4 from:20];
        [calc multiply:6 by:7];
        [calc divide:22 by:7];
        [calc modulo:17 by:5];
        [calc power:2 exponent:10];
        [calc divide:5 by:0];  // Error case
        
        // Bitwise operations
        NSLog(@"\n--- การคำนวณระดับบิต ---");
        NSInteger a = 0b11001010;  // 202
        NSInteger b = 0b10110101;  // 181
        NSLog(@"a = %ld (binary: %@)", (long)a, toBinary((int)a, 8));
        NSLog(@"b = %ld (binary: %@)", (long)b, toBinary((int)b, 8));
        NSLog(@"");
        
        NSInteger andR = [calc bitwiseAnd:a with:b];
        NSLog(@"AND: %ld = %@", (long)andR, toBinary((int)andR, 8));
        
        NSInteger orR = [calc bitwiseOr:a with:b];
        NSLog(@"OR:  %ld = %@", (long)orR, toBinary((int)orR, 8));
        
        NSInteger xorR = [calc bitwiseXor:a with:b];
        NSLog(@"XOR: %ld = %@", (long)xorR, toBinary((int)xorR, 8));
        
        NSInteger lshiftR = [calc leftShift:5 by:3];
        NSLog(@"5 << 3 = %ld (5 × 8 = 40)", (long)lshiftR);
        
        NSInteger rshiftR = [calc rightShift:64 by:2];
        NSLog(@"64 >> 2 = %ld (64 ÷ 4 = 16)", (long)rshiftR);
        
        // Float comparison
        NSLog(@"\n--- Float Comparison ---");
        double r1 = 0.1 + 0.2;
        double r2 = 0.3;
        NSLog(@"0.1 + 0.2 == 0.3: %@", (r1 == r2) ? @"YES" : @"NO");
        NSLog(@"ด้วย epsilon: %@", [calc isEqual:r1 to:r2 epsilon:1e-9] ? @"YES" : @"NO");
        
        // Print history
        NSLog(@"");
        [calc printHistory];
        
        // Compound assignment demo
        NSLog(@"\n--- Compound Assignment Demo ---");
        int counter = 0;
        NSLog(@"counter = %d", counter);
        counter += 10; NSLog(@"+=10: %d", counter);
        counter *= 2;  NSLog(@"*=2:  %d", counter);
        counter -= 5;  NSLog(@"-=5:  %d", counter);
        counter /= 3;  NSLog(@"/=3:  %d", counter);
        counter %= 4;  NSLog(@"%%=4:  %d", counter);
        counter <<= 3; NSLog(@"<<=3: %d", counter);
        
    }
    return 0;
}
```

---

## 11. แบบฝึกหัดพร้อมเฉลย

### แบบฝึกหัดที่ 1: Operator Basics

**โจทย์:** เขียนโปรแกรมคำนวณพื้นที่และเส้นรอบวงของรูปทรงต่างๆ (วงกลม, สี่เหลี่ยม, สามเหลี่ยม)

**เฉลย:**
```objc
#import <Foundation/Foundation.h>
#include <math.h>

// โครงสร้างสำหรับเก็บผลลัพธ์
typedef struct {
    double area;
    double perimeter;
} ShapeResult;

// คำนวณวงกลม
ShapeResult circleResult(double radius) {
    ShapeResult r;
    r.area = M_PI * radius * radius;
    r.perimeter = 2 * M_PI * radius;
    return r;
}

// คำนวณสี่เหลี่ยมผืนผ้า
ShapeResult rectangleResult(double width, double height) {
    ShapeResult r;
    r.area = width * height;
    r.perimeter = 2 * (width + height);
    return r;
}

// คำนวณสามเหลี่ยมด้วย Heron's formula
ShapeResult triangleResult(double a, double b, double c) {
    ShapeResult r;
    r.perimeter = a + b + c;
    double s = r.perimeter / 2.0;  // semi-perimeter
    r.area = sqrt(s * (s-a) * (s-b) * (s-c));
    return r;
}

int main(int argc, const char * argv[]) {
    @autoreleasepool {
        
        NSLog(@"===== เครื่องคำนวณรูปทรง =====\n");
        
        // วงกลม
        double radius = 7.5;
        ShapeResult circle = circleResult(radius);
        NSLog(@"วงกลม (r=%.1f):", radius);
        NSLog(@"  พื้นที่ = %.4f", circle.area);
        NSLog(@"  เส้นรอบวง = %.4f", circle.perimeter);
        
        NSLog(@"");
        
        // สี่เหลี่ยมผืนผ้า
        double w = 10.0, h = 6.0;
        ShapeResult rect = rectangleResult(w, h);
        NSLog(@"สี่เหลี่ยมผืนผ้า (w=%.1f, h=%.1f):", w, h);
        NSLog(@"  พื้นที่ = %.2f", rect.area);
        NSLog(@"  เส้นรอบวง = %.2f", rect.perimeter);
        
        NSLog(@"");
        
        // สามเหลี่ยม (3, 4, 5 triangle)
        double sa = 3.0, sb = 4.0, sc = 5.0;
        ShapeResult tri = triangleResult(sa, sb, sc);
        NSLog(@"สามเหลี่ยม (a=%.1f, b=%.1f, c=%.1f):", sa, sb, sc);
        NSLog(@"  พื้นที่ = %.2f", tri.area);
        NSLog(@"  เส้นรอบรูป = %.2f", tri.perimeter);
        
        // ตรวจสอบว่าสามเหลี่ยมถูกต้อง (Triangle Inequality)
        BOOL validTriangle = (sa + sb > sc) && (sb + sc > sa) && (sa + sc > sb);
        NSLog(@"  สามเหลี่ยมถูกต้อง: %@", validTriangle ? @"YES" : @"NO");
        
    }
    return 0;
}
```

### แบบฝึกหัดที่ 2: Bitwise Magic

**โจทย์:** เขียนโปรแกรมที่:
1. แสดง bit representation ของตัวเลข
2. นับจำนวน bit ที่เป็น 1 (popcount)
3. ตรวจสอบว่าตัวเลขเป็น power of 2 หรือไม่
4. หา position ของ highest set bit

**เฉลย:**
```objc
#import <Foundation/Foundation.h>

// แสดง bits
NSString *showBits(unsigned int n) {
    NSMutableString *result = [NSMutableString string];
    for (int i = 31; i >= 0; i--) {
        [result appendString:((n >> i) & 1) ? @"1" : @"0"];
        if (i > 0 && i % 4 == 0) [result appendString:@" "];
    }
    return result;
}

// นับจำนวน set bits (Brian Kernighan's algorithm)
NSInteger popcount(unsigned int n) {
    NSInteger count = 0;
    while (n) {
        n &= (n - 1);  // ลบ lowest set bit
        count++;
    }
    return count;
}

// ตรวจสอบ power of 2
BOOL isPowerOfTwo(unsigned int n) {
    return n > 0 && (n & (n - 1)) == 0;
}

// หา position ของ highest set bit (0-indexed)
NSInteger highestSetBit(unsigned int n) {
    if (n == 0) return -1;
    NSInteger pos = 0;
    while (n > 1) {
        n >>= 1;
        pos++;
    }
    return pos;
}

// Swap สองตัวเลขโดยไม่ใช้ temporary variable (ด้วย XOR)
void swapXOR(int *a, int *b) {
    *a ^= *b;  // a = a XOR b
    *b ^= *a;  // b = b XOR (a XOR b) = a
    *a ^= *b;  // a = (a XOR b) XOR a = b
}

int main(int argc, const char * argv[]) {
    @autoreleasepool {
        
        NSLog(@"===== Bitwise Magic =====\n");
        
        // แสดง bit representation
        NSLog(@"--- Bit Representation ---");
        unsigned int nums[] = {0, 1, 7, 15, 255, 1024, 65535};
        int numCount = sizeof(nums) / sizeof(nums[0]);
        
        for (int i = 0; i < numCount; i++) {
            NSLog(@"%5u = %@", nums[i], showBits(nums[i]));
        }
        
        NSLog(@"");
        
        // Count set bits
        NSLog(@"--- Popcount (จำนวน 1-bits) ---");
        for (int i = 0; i < numCount; i++) {
            NSLog(@"popcount(%5u) = %ld", nums[i], (long)popcount(nums[i]));
        }
        
        NSLog(@"");
        
        // Power of 2
        NSLog(@"--- Power of 2 ---");
        for (int i = 0; i <= 20; i++) {
            if (isPowerOfTwo(i)) {
                NSLog(@"%d = 2^%ld", i, (long)highestSetBit(i));
            }
        }
        
        NSLog(@"");
        
        // XOR swap
        NSLog(@"--- XOR Swap ---");
        int x = 42, y = 17;
        NSLog(@"ก่อน swap: x=%d, y=%d", x, y);
        swapXOR(&x, &y);
        NSLog(@"หลัง swap: x=%d, y=%d", x, y);
        
        NSLog(@"");
        
        // Tricks ต่างๆ
        NSLog(@"--- Bitwise Tricks ---");
        int n = 37;
        NSLog(@"n = %d", n);
        NSLog(@"n & 1 (เลขคู่/คี่): %@ (%@)", @(n & 1), (n & 1) ? @"คี่" : @"คู่");
        NSLog(@"n >> 1 (n/2): %d", n >> 1);
        NSLog(@"n << 1 (n*2): %d", n << 1);
        NSLog(@"n | 1 (ทำให้เป็นเลขคี่): %d", n | 1);
        NSLog(@"n & ~1 (ทำให้เป็นเลขคู่): %d", n & ~1);
        NSLog(@"n ^ n (XOR ตัวเอง = 0): %d", n ^ n);
        
    }
    return 0;
}
```

### แบบฝึกหัดที่ 3: Operator Precedence Puzzle

**โจทย์:** ทำนายผลลัพธ์ของ expressions ต่อไปนี้โดยไม่รันโปรแกรม แล้วตรวจสอบด้วยโปรแกรม

**เฉลย:**
```objc
#import <Foundation/Foundation.h>

int main(int argc, const char * argv[]) {
    @autoreleasepool {
        
        NSLog(@"===== Operator Precedence Puzzle =====\n");
        
        // Puzzle 1
        int r1 = 2 + 3 * 4 - 1;
        NSLog(@"2 + 3 * 4 - 1 = %d (expected: 13)", r1);
        // 2 + (3*4) - 1 = 2 + 12 - 1 = 13
        
        // Puzzle 2
        int r2 = 10 / 2 + 3 * 2;
        NSLog(@"10 / 2 + 3 * 2 = %d (expected: 11)", r2);
        // (10/2) + (3*2) = 5 + 6 = 11
        
        // Puzzle 3
        int r3 = 15 % 4 + 2 * 3;
        NSLog(@"15 %% 4 + 2 * 3 = %d (expected: 9)", r3);
        // (15%4) + (2*3) = 3 + 6 = 9
        
        // Puzzle 4
        BOOL r4 = 5 > 3 && 2 < 10 || 1 > 5;
        NSLog(@"5 > 3 && 2 < 10 || 1 > 5 = %@ (expected: YES)", r4 ? @"YES" : @"NO");
        // (5>3 && 2<10) || 1>5 = (YES && YES) || NO = YES || NO = YES
        
        // Puzzle 5
        int r5 = 1 << 3 + 1;
        NSLog(@"1 << 3 + 1 = %d (expected: 16)", r5);
        // 1 << (3+1) = 1 << 4 = 16 (+ มีลำดับสูงกว่า <<)
        
        // Puzzle 6
        int x = 5;
        int r6 = x++ + ++x;  // undefined behavior แต่ทดสอบเพื่อความเข้าใจ
        NSLog(@"WARNING: x=5, x++ + ++x = %d, x=%d (undefined!)", r6, x);
        
        // Puzzle 7
        int r7 = (int)(3.7) + (int)(2.3);
        NSLog(@"(int)3.7 + (int)2.3 = %d (expected: 5, not 6!)", r7);
        // truncation: 3 + 2 = 5
        
        // Puzzle 8
        BOOL r8 = !0 + !1;
        NSLog(@"!0 + !1 = %d (expected: 1)", r8);
        // !0 = 1, !1 = 0, 1 + 0 = 1
        
        // Puzzle 9
        int a = 3, b = 4, c = 5;
        BOOL r9 = a < b < c;  // ระวัง! ใน C ไม่ใช่ chaining
        NSLog(@"3 < 4 < 5 = %@ (unexpected! เพราะ (3<4)<5 = 1<5 = YES)",
              r9 ? @"YES" : @"NO");
        
        BOOL r9correct = a < b && b < c;  // วิธีที่ถูกต้อง
        NSLog(@"3 < 4 && 4 < 5 = %@ (correct)", r9correct ? @"YES" : @"NO");
        
        // Puzzle 10
        int r10 = sizeof(int) * 2 + sizeof(double);
        NSLog(@"sizeof(int)*2 + sizeof(double) = %d (on 64-bit: 4*2+8=16)", r10);
        
        NSLog(@"\n--- คำตอบสรุป ---");
        NSLog(@"1. %d", r1);
        NSLog(@"2. %d", r2);
        NSLog(@"3. %d", r3);
        NSLog(@"4. %@", r4 ? @"YES" : @"NO");
        NSLog(@"5. %d", r5);
        NSLog(@"7. %d", r7);
        NSLog(@"8. %d", r8);
        NSLog(@"9. %@", r9 ? @"YES" : @"NO");
        NSLog(@"10. %d", r10);
        
    }
    return 0;
}
```

### แบบฝึกหัดที่ 4: Mini Text Processor

**โจทย์:** สร้าง text processor ที่ใช้ operator ทุกประเภท

**เฉลย:**
```objc
#import <Foundation/Foundation.h>

@interface TextProcessor : NSObject

// Logical operations
- (BOOL)textContains:(NSString *)text keyword:(NSString *)keyword 
          ignoreCase:(BOOL)ignoreCase;

// Ternary-based operations
- (NSString *)truncateText:(NSString *)text maxLength:(NSInteger)maxLength;
- (NSString *)formatScore:(double)score;

// Bitwise-based operations (text encoding flags)
- (NSString *)encodeText:(NSString *)text flags:(NSUInteger)flags;

// Statistics
- (NSDictionary *)analyzeText:(NSString *)text;

@end

@implementation TextProcessor

#define FLAG_UPPERCASE    (1 << 0)   // 0001
#define FLAG_TRIM         (1 << 1)   // 0010
#define FLAG_REVERSE      (1 << 2)   // 0100
#define FLAG_REMOVE_VOWEL (1 << 3)   // 1000

- (BOOL)textContains:(NSString *)text keyword:(NSString *)keyword 
          ignoreCase:(BOOL)ignoreCase {
    if (!text || !keyword || text.length == 0 || keyword.length == 0) {
        return NO;
    }
    NSStringCompareOptions options = ignoreCase ? NSCaseInsensitiveSearch : 0;
    return [text rangeOfString:keyword options:options].location != NSNotFound;
}

- (NSString *)truncateText:(NSString *)text maxLength:(NSInteger)maxLength {
    return text.length <= maxLength 
        ? text 
        : [[text substringToIndex:maxLength - 3] stringByAppendingString:@"..."];
}

- (NSString *)formatScore:(double)score {
    NSString *grade = score >= 90 ? @"A" :
                      score >= 80 ? @"B" :
                      score >= 70 ? @"C" :
                      score >= 60 ? @"D" : @"F";
    
    NSString *emoji = score >= 90 ? @"(ดีเยี่ยม)" :
                      score >= 70 ? @"(ผ่าน)" : @"(ไม่ผ่าน)";
    
    return [NSString stringWithFormat:@"%.1f/100 เกรด %@ %@", score, grade, emoji];
}

- (NSString *)encodeText:(NSString *)text flags:(NSUInteger)flags {
    NSString *result = text;
    
    // Apply flags ตาม bit
    if (flags & FLAG_TRIM) {
        result = [result stringByTrimmingCharactersInSet:
                  [NSCharacterSet whitespaceCharacterSet]];
    }
    
    if (flags & FLAG_UPPERCASE) {
        result = [result uppercaseString];
    }
    
    if (flags & FLAG_REVERSE) {
        NSMutableString *reversed = [NSMutableString string];
        NSInteger len = result.length;
        for (NSInteger i = len - 1; i >= 0; i--) {
            [reversed appendFormat:@"%C", [result characterAtIndex:i]];
        }
        result = reversed;
    }
    
    return result;
}

- (NSDictionary *)analyzeText:(NSString *)text {
    if (!text) return @{};
    
    NSInteger charCount = text.length;
    NSArray *words = [[text stringByTrimmingCharactersInSet:
                       [NSCharacterSet whitespaceCharacterSet]] 
                      componentsSeparatedByCharactersInSet:
                      [NSCharacterSet whitespaceCharacterSet]];
    
    NSPredicate *nonEmpty = [NSPredicate predicateWithFormat:@"length > 0"];
    NSInteger wordCount = [words filteredArrayUsingPredicate:nonEmpty].count;
    
    NSInteger sentenceCount = [[text componentsSeparatedByCharactersInSet:
                                [NSCharacterSet characterSetWithCharactersInString:@".!?"]]
                               filteredArrayUsingPredicate:nonEmpty].count;
    
    double avgWordLength = wordCount > 0 ? (double)charCount / wordCount : 0;
    
    return @{
        @"characters": @(charCount),
        @"words": @(wordCount),
        @"sentences": @(sentenceCount),
        @"avgWordLength": @(avgWordLength)
    };
}

@end

int main(int argc, const char * argv[]) {
    @autoreleasepool {
        
        TextProcessor *tp = [[TextProcessor alloc] init];
        
        NSString *sampleText = @"  Hello World! This is Objective-C programming.  ";
        
        NSLog(@"=== Text Processor ===\n");
        
        // Truncate
        NSLog(@"ตัดข้อความ (max 20):");
        NSLog(@"'%@'", [tp truncateText:sampleText maxLength:20]);
        
        NSLog(@"");
        
        // Search
        NSLog(@"ค้นหาคำ:");
        NSLog(@"มี 'World': %@", 
              [tp textContains:sampleText keyword:@"World" ignoreCase:NO] ? @"YES" : @"NO");
        NSLog(@"มี 'world' (ignore case): %@", 
              [tp textContains:sampleText keyword:@"world" ignoreCase:YES] ? @"YES" : @"NO");
        NSLog(@"มี 'Python': %@", 
              [tp textContains:sampleText keyword:@"Python" ignoreCase:NO] ? @"YES" : @"NO");
        
        NSLog(@"");
        
        // Encoding with flags
        NSLog(@"Encoding flags:");
        NSString *testText = @"  hello world  ";
        NSLog(@"Original: '%@'", testText);
        NSLog(@"TRIM: '%@'", [tp encodeText:testText flags:FLAG_TRIM]);
        NSLog(@"UPPER: '%@'", [tp encodeText:testText flags:FLAG_UPPERCASE]);
        NSLog(@"TRIM+UPPER: '%@'", [tp encodeText:testText flags:FLAG_TRIM | FLAG_UPPERCASE]);
        NSLog(@"REVERSE: '%@'", [tp encodeText:testText flags:FLAG_REVERSE]);
        NSLog(@"ALL: '%@'", [tp encodeText:testText flags:FLAG_TRIM | FLAG_UPPERCASE | FLAG_REVERSE]);
        
        NSLog(@"");
        
        // Scores
        NSLog(@"คะแนน:");
        NSArray *scores = @[@95.0, @82.5, @71.0, @55.0, @45.0];
        for (NSNumber *score in scores) {
            NSLog(@"  %@", [tp formatScore:[score doubleValue]]);
        }
        
        NSLog(@"");
        
        // Analysis
        NSLog(@"วิเคราะห์ข้อความ:");
        NSString *article = @"Objective-C is a powerful language. It was created in 1983. Apple used it for iOS development.";
        NSDictionary *stats = [tp analyzeText:article];
        NSLog(@"Text: '%@'", article);
        NSLog(@"ตัวอักษร: %@", stats[@"characters"]);
        NSLog(@"คำ: %@", stats[@"words"]);
        NSLog(@"ประโยค: %@", stats[@"sentences"]);
        NSLog(@"ความยาวเฉลี่ยต่อคำ: %.1f ตัวอักษร", [stats[@"avgWordLength"] doubleValue]);
        
    }
    return 0;
}
```

---

## บทสรุป

ใน Part 03 เราได้เรียนรู้ Operators ทั้งหมดใน Objective-C:

| ประเภท | Operators | ตัวอย่าง |
|--------|-----------|---------|
| Arithmetic | +, -, *, /, % | `a + b`, `a % b` |
| Comparison | ==, !=, <, >, <=, >= | `a == b`, `a > b` |
| Logical | &&, \|\|, ! | `a && b`, `!flag` |
| Bitwise | &, \|, ^, ~, <<, >> | `a & b`, `n << 3` |
| Assignment | =, +=, -=, *=, /=, %= | `x += 5` |
| Bitwise Assignment | &=, \|=, ^=, <<=, >>= | `flags \|= FLAG_READ` |
| Ternary | ?: | `a > b ? a : b` |
| Increment | ++, -- | `i++`, `--j` |

### Key Concepts

1. **Integer Division**: `7 / 2 = 3` ไม่ใช่ 3.5 ต้อง cast เป็น double ก่อน
2. **Short-Circuit**: `&&` และ `||` หยุด evaluate ทันทีที่รู้ผลแล้ว
3. **Bitwise Flags**: ใช้ bit operators สำหรับ permissions และ options
4. **Ternary**: เหมาะสำหรับ simple if-else แต่ไม่ควร nest ลึกเกิน
5. **Precedence**: ใช้ parentheses `()` เสมอเมื่อไม่แน่ใจ
6. **BOOL vs integer**: ระวัง overflow เมื่อ assign ไปยัง BOOL

## บทต่อไป

ใน **Part 04** เราจะเรียนรู้เรื่อง:
- Control Flow: if, else if, else
- Switch Statement
- For Loop, While Loop, Do-While Loop
- Break, Continue, Return
- Goto (ไม่แนะนำ แต่ควรรู้จัก)
- Nested Loops
- Fast Enumeration ใน Objective-C

---

*เอกสารนี้เป็นส่วนหนึ่งของหลักสูตร Objective-C Programming*  
*Part 03 of 100*
