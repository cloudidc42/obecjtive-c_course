# ส่วนที่ 04: การควบคุมการทำงาน (Control Flow) ใน Objective-C

## บทนำ

การควบคุมการทำงาน (Control Flow) คือหัวใจสำคัญของการเขียนโปรแกรม ช่วยให้โปรแกรมสามารถตัดสินใจได้ว่าจะทำงานอะไรภายใต้เงื่อนไขใด ใน Objective-C เราสามารถใช้คำสั่งควบคุมการทำงานได้หลายรูปแบบ ซึ่งเราจะศึกษาทีละหัวข้อในบทนี้

---

## 4.1 คำสั่ง if, else if, else

### พื้นฐาน if Statement

คำสั่ง `if` ใช้ตรวจสอบเงื่อนไข ถ้าเงื่อนไขเป็นจริง (true) จะทำงานในบล็อก if ถ้าไม่จริง (false) จะข้ามไป

```objc
#import <Foundation/Foundation.h>

int main(int argc, const char * argv[]) {
    @autoreleasepool {
        int age = 20;
        
        // if เดี่ยว
        if (age >= 18) {
            NSLog(@"คุณเป็นผู้ใหญ่");
        }
        
        // if-else
        if (age >= 18) {
            NSLog(@"คุณบรรลุนิติภาวะแล้ว");
        } else {
            NSLog(@"คุณยังเป็นผู้เยาว์");
        }
    }
    return 0;
}
```

**ผลลัพธ์:**
```
คุณเป็นผู้ใหญ่
คุณบรรลุนิติภาวะแล้ว
```

### if-else if-else Chain

เมื่อมีหลายเงื่อนไขให้ตรวจสอบตามลำดับ:

```objc
#import <Foundation/Foundation.h>

int main(int argc, const char * argv[]) {
    @autoreleasepool {
        int score = 75;
        NSString *grade;
        
        if (score >= 90) {
            grade = @"A";
        } else if (score >= 80) {
            grade = @"B";
        } else if (score >= 70) {
            grade = @"C";
        } else if (score >= 60) {
            grade = @"D";
        } else {
            grade = @"F";
        }
        
        NSLog(@"คะแนน %d ได้เกรด: %@", score, grade);
        
        // ตัวอย่างเพิ่มเติม: ตรวจสอบเดือน
        int month = 4;
        NSString *season;
        
        if (month >= 3 && month <= 5) {
            season = @"ฤดูใบไม้ผลิ";
        } else if (month >= 6 && month <= 8) {
            season = @"ฤดูร้อน";
        } else if (month >= 9 && month <= 11) {
            season = @"ฤดูใบไม้ร่วง";
        } else {
            season = @"ฤดูหนาว";
        }
        
        NSLog(@"เดือน %d เป็น: %@", month, season);
    }
    return 0;
}
```

**ผลลัพธ์:**
```
คะแนน 75 ได้เกรด: C
เดือน 4 เป็น: ฤดูใบไม้ผลิ
```

### การเปรียบเทียบ NSString

ใน Objective-C การเปรียบเทียบ String ต้องใช้เมธอด `isEqualToString:` ไม่ใช่ `==`

```objc
#import <Foundation/Foundation.h>

int main(int argc, const char * argv[]) {
    @autoreleasepool {
        NSString *name = @"สมชาย";
        
        // ผิด! ไม่ควรใช้ == กับ NSString
        // if (name == @"สมชาย") { ... }
        
        // ถูก! ใช้ isEqualToString:
        if ([name isEqualToString:@"สมชาย"]) {
            NSLog(@"ยินดีต้อนรับ คุณ%@", name);
        }
        
        // เปรียบเทียบแบบ case-insensitive
        NSString *input = @"YES";
        if ([input caseInsensitiveCompare:@"yes"] == NSOrderedSame) {
            NSLog(@"ผู้ใช้ตอบ Yes");
        }
    }
    return 0;
}
```

---

## 4.2 คำสั่ง switch-case

`switch` ใช้เลือกเส้นทางการทำงานตามค่าของตัวแปร เหมาะสำหรับเมื่อมีหลายค่าที่ต้องตรวจสอบ

### switch-case พื้นฐาน

```objc
#import <Foundation/Foundation.h>

int main(int argc, const char * argv[]) {
    @autoreleasepool {
        int dayOfWeek = 3;
        NSString *dayName;
        
        switch (dayOfWeek) {
            case 1:
                dayName = @"วันจันทร์";
                break;
            case 2:
                dayName = @"วันอังคาร";
                break;
            case 3:
                dayName = @"วันพุธ";
                break;
            case 4:
                dayName = @"วันพฤหัสบดี";
                break;
            case 5:
                dayName = @"วันศุกร์";
                break;
            case 6:
                dayName = @"วันเสาร์";
                break;
            case 7:
                dayName = @"วันอาทิตย์";
                break;
            default:
                dayName = @"ไม่ทราบวัน";
                break;
        }
        
        NSLog(@"วันที่ %d คือ %@", dayOfWeek, dayName);
    }
    return 0;
}
```

**ผลลัพธ์:**
```
วันที่ 3 คือ วันพุธ
```

### Fall-through ใน switch

เมื่อไม่มี `break` โปรแกรมจะ "fall-through" ไปยัง case ถัดไป:

```objc
#import <Foundation/Foundation.h>

int main(int argc, const char * argv[]) {
    @autoreleasepool {
        int month = 4;
        int daysInMonth;
        
        switch (month) {
            case 1:
            case 3:
            case 5:
            case 7:
            case 8:
            case 10:
            case 12:
                daysInMonth = 31;
                break;
            case 4:
            case 6:
            case 9:
            case 11:
                daysInMonth = 30;
                break;
            case 2:
                daysInMonth = 28; // ไม่นับปีอธิกสุรทิน
                break;
            default:
                daysInMonth = -1;
                NSLog(@"เดือนไม่ถูกต้อง");
                break;
        }
        
        if (daysInMonth > 0) {
            NSLog(@"เดือน %d มี %d วัน", month, daysInMonth);
        }
    }
    return 0;
}
```

**ผลลัพธ์:**
```
เดือน 4 มี 30 วัน
```

### switch กับ NSString (ผ่าน hash)

ใน Objective-C ปกติ switch ใช้กับ integer เท่านั้น แต่เราสามารถจัดการ NSString ได้:

```objc
#import <Foundation/Foundation.h>

int main(int argc, const char * argv[]) {
    @autoreleasepool {
        NSString *command = @"start";
        
        // วิธีที่ 1: ใช้ if-else if แทน switch สำหรับ NSString
        if ([command isEqualToString:@"start"]) {
            NSLog(@"เริ่มต้นระบบ...");
        } else if ([command isEqualToString:@"stop"]) {
            NSLog(@"หยุดระบบ...");
        } else if ([command isEqualToString:@"restart"]) {
            NSLog(@"รีสตาร์ทระบบ...");
        } else {
            NSLog(@"คำสั่งไม่รู้จัก: %@", command);
        }
        
        // วิธีที่ 2: ใช้ NSDictionary เป็น dispatch table
        NSDictionary *actions = @{
            @"start": @"เริ่มต้นระบบ",
            @"stop": @"หยุดระบบ",
            @"restart": @"รีสตาร์ทระบบ"
        };
        
        NSString *action = actions[command];
        if (action) {
            NSLog(@"กำลังดำเนินการ: %@", action);
        }
    }
    return 0;
}
```

---

## 4.3 Nested Conditions (เงื่อนไขซ้อน)

เงื่อนไขซ้อนคือการมีคำสั่ง if ภายใน if อีกชั้นหนึ่ง

```objc
#import <Foundation/Foundation.h>

int main(int argc, const char * argv[]) {
    @autoreleasepool {
        int age = 25;
        BOOL hasLicense = YES;
        BOOL hasCar = NO;
        
        if (age >= 18) {
            NSLog(@"คุณมีอายุพอที่จะขับรถได้");
            
            if (hasLicense) {
                NSLog(@"และมีใบขับขี่แล้ว");
                
                if (hasCar) {
                    NSLog(@"และมีรถ คุณพร้อมขับรถได้เลย!");
                } else {
                    NSLog(@"แต่ไม่มีรถ ต้องเช่าหรือซื้อก่อน");
                }
            } else {
                NSLog(@"แต่ยังไม่มีใบขับขี่ ต้องไปสอบก่อน");
            }
        } else {
            NSLog(@"คุณอายุยังน้อยเกินไปที่จะขับรถ");
        }
    }
    return 0;
}
```

**ผลลัพธ์:**
```
คุณมีอายุพอที่จะขับรถได้
และมีใบขับขี่แล้ว
แต่ไม่มีรถ ต้องเช่าหรือซื้อก่อน
```

### ตัวอย่างขั้นสูง: ระบบจัดเรียงสินค้า

```objc
#import <Foundation/Foundation.h>

int main(int argc, const char * argv[]) {
    @autoreleasepool {
        NSString *category = @"electronics";
        double price = 15000.0;
        int quantity = 3;
        NSString *result;
        
        if ([category isEqualToString:@"electronics"]) {
            if (price > 10000) {
                if (quantity > 2) {
                    result = @"ส่วนลด 15% สำหรับอิเล็กทรอนิกส์ราคาสูง ซื้อมากกว่า 2 ชิ้น";
                } else {
                    result = @"ส่วนลด 10% สำหรับอิเล็กทรอนิกส์ราคาสูง";
                }
            } else {
                result = @"ส่วนลด 5% สำหรับอิเล็กทรอนิกส์ทั่วไป";
            }
        } else if ([category isEqualToString:@"clothing"]) {
            if (quantity >= 3) {
                result = @"ซื้อ 3 ฟรี 1 สำหรับเสื้อผ้า";
            } else {
                result = @"ราคาปกติสำหรับเสื้อผ้า";
            }
        } else {
            result = @"ไม่มีโปรโมชั่นพิเศษ";
        }
        
        NSLog(@"โปรโมชั่น: %@", result);
        NSLog(@"ยอดรวม: %.2f บาท", price * quantity);
    }
    return 0;
}
```

---

## 4.4 Boolean Expressions (นิพจน์บูลีน)

### ตัวดำเนินการทางตรรกะ

| ตัวดำเนินการ | ความหมาย | ตัวอย่าง |
|---|---|---|
| `&&` | AND (และ) | `a > 0 && b > 0` |
| `\|\|` | OR (หรือ) | `a > 0 \|\| b > 0` |
| `!` | NOT (ไม่) | `!isValid` |

```objc
#import <Foundation/Foundation.h>

int main(int argc, const char * argv[]) {
    @autoreleasepool {
        int temperature = 28;
        BOOL isRaining = NO;
        BOOL isWeekend = YES;
        
        // AND: ทั้งสองเงื่อนไขต้องเป็นจริง
        if (temperature > 25 && !isRaining) {
            NSLog(@"อากาศดี เหมาะกับการออกไปเที่ยว");
        }
        
        // OR: อย่างน้อยหนึ่งเงื่อนไขต้องเป็นจริง
        if (temperature < 15 || isRaining) {
            NSLog(@"ควรพกร่มหรือเสื้อกันหนาว");
        } else {
            NSLog(@"ไม่จำเป็นต้องพกร่ม");
        }
        
        // NOT: กลับค่าบูลีน
        if (!isRaining) {
            NSLog(@"ไม่ฝนตก");
        }
        
        // ผสมหลายเงื่อนไข
        BOOL shouldGoOut = (temperature > 20) && !isRaining && isWeekend;
        if (shouldGoOut) {
            NSLog(@"ควรออกไปเที่ยว!");
        }
        
        // ตัวอย่างการตรวจสอบ range
        int score = 75;
        BOOL isPass = score >= 50 && score <= 100;
        BOOL isExcellent = score >= 90;
        
        NSLog(@"ผ่าน: %@", isPass ? @"ใช่" : @"ไม่");
        NSLog(@"ดีเยี่ยม: %@", isExcellent ? @"ใช่" : @"ไม่");
    }
    return 0;
}
```

### Ternary Operator

ตัวดำเนินการ ternary `? :` ใช้เขียน if-else ในบรรทัดเดียว:

```objc
#import <Foundation/Foundation.h>

int main(int argc, const char * argv[]) {
    @autoreleasepool {
        int x = 10;
        int y = 20;
        
        // แบบยาว
        int max;
        if (x > y) {
            max = x;
        } else {
            max = y;
        }
        
        // แบบสั้น ternary operator
        int maxShort = (x > y) ? x : y;
        
        NSLog(@"ค่ามากที่สุด (แบบยาว): %d", max);
        NSLog(@"ค่ามากที่สุด (ternary): %d", maxShort);
        
        // ใช้กับ NSString
        NSString *status = (x > 0) ? @"บวก" : @"ไม่บวก";
        NSLog(@"ค่า %d เป็น%@", x, status);
        
        // ternary ซ้อนกัน (ควรระวัง อ่านยาก)
        int num = 0;
        NSString *sign = (num > 0) ? @"บวก" : (num < 0) ? @"ลบ" : @"ศูนย์";
        NSLog(@"ค่า %d เป็น%@", num, sign);
    }
    return 0;
}
```

---

## 4.5 Short-Circuit Evaluation

Short-circuit evaluation คือการที่ Objective-C จะหยุดประเมินเงื่อนไขทันทีที่รู้ผลแล้ว

```objc
#import <Foundation/Foundation.h>

// ฟังก์ชันช่วยสาธิต
BOOL checkAge(int age) {
    NSLog(@"กำลังตรวจสอบอายุ: %d", age);
    return age >= 18;
}

BOOL checkIncome(double income) {
    NSLog(@"กำลังตรวจสอบรายได้: %.0f", income);
    return income >= 15000;
}

int main(int argc, const char * argv[]) {
    @autoreleasepool {
        NSLog(@"=== AND Short-circuit ===");
        int age = 15;
        double income = 20000;
        
        // เมื่อ checkAge คืนค่า NO
        // checkIncome จะไม่ถูกเรียก! (short-circuit)
        if (checkAge(age) && checkIncome(income)) {
            NSLog(@"ผ่านเงื่อนไขทั้งหมด");
        } else {
            NSLog(@"ไม่ผ่านเงื่อนไข");
        }
        
        NSLog(@"\n=== OR Short-circuit ===");
        age = 20;
        income = 5000;
        
        // เมื่อ checkAge คืนค่า YES
        // checkIncome จะไม่ถูกเรียก! (short-circuit)
        if (checkAge(age) || checkIncome(income)) {
            NSLog(@"ผ่านอย่างน้อยหนึ่งเงื่อนไข");
        }
        
        NSLog(@"\n=== ป้องกัน nil pointer ===");
        NSString *text = nil;
        
        // Safe: ถ้า text เป็น nil จะไม่เรียก length
        // เพราะ (text != nil) จะ short-circuit เป็น NO
        if (text != nil && [text length] > 0) {
            NSLog(@"ข้อความ: %@", text);
        } else {
            NSLog(@"ข้อความว่างเปล่าหรือ nil");
        }
    }
    return 0;
}
```

**ผลลัพธ์:**
```
=== AND Short-circuit ===
กำลังตรวจสอบอายุ: 15
ไม่ผ่านเงื่อนไข

=== OR Short-circuit ===
กำลังตรวจสอบอายุ: 20
ผ่านอย่างน้อยหนึ่งเงื่อนไข

=== ป้องกัน nil pointer ===
ข้อความว่างเปล่าหรือ nil
```

---

## 4.6 Guard Clauses Pattern

Guard clause เป็นรูปแบบการเขียนโค้ดที่ตรวจสอบเงื่อนไขข้อผิดพลาดก่อน แล้วออกจากฟังก์ชันทันทีถ้าพบปัญหา ทำให้โค้ดอ่านง่ายขึ้น

### ปัญหาของ Nested if (Pyramid of Doom)

```objc
#import <Foundation/Foundation.h>

// ❌ แบบ BAD - Pyramid of Doom
void processOrderBad(NSString *userId, NSString *productId, int quantity) {
    if (userId != nil) {
        if ([userId length] > 0) {
            if (productId != nil) {
                if ([productId length] > 0) {
                    if (quantity > 0) {
                        // ประมวลผลออร์เดอร์
                        NSLog(@"ประมวลผลออร์เดอร์: User=%@, Product=%@, Qty=%d",
                              userId, productId, quantity);
                    } else {
                        NSLog(@"จำนวนสินค้าต้องมากกว่า 0");
                    }
                } else {
                    NSLog(@"productId ต้องไม่ว่าง");
                }
            } else {
                NSLog(@"productId ต้องไม่เป็น nil");
            }
        } else {
            NSLog(@"userId ต้องไม่ว่าง");
        }
    } else {
        NSLog(@"userId ต้องไม่เป็น nil");
    }
}

// ✅ แบบ GOOD - Guard Clauses
void processOrderGood(NSString *userId, NSString *productId, int quantity) {
    // ตรวจสอบและออกเร็วถ้าพบปัญหา
    if (userId == nil) {
        NSLog(@"Error: userId ต้องไม่เป็น nil");
        return;
    }
    
    if ([userId length] == 0) {
        NSLog(@"Error: userId ต้องไม่ว่าง");
        return;
    }
    
    if (productId == nil) {
        NSLog(@"Error: productId ต้องไม่เป็น nil");
        return;
    }
    
    if ([productId length] == 0) {
        NSLog(@"Error: productId ต้องไม่ว่าง");
        return;
    }
    
    if (quantity <= 0) {
        NSLog(@"Error: จำนวนสินค้าต้องมากกว่า 0");
        return;
    }
    
    // โค้ดหลักอยู่ที่นี่ ไม่มี indentation มากเกินไป
    NSLog(@"ประมวลผลออร์เดอร์: User=%@, Product=%@, Qty=%d",
          userId, productId, quantity);
}

int main(int argc, const char * argv[]) {
    @autoreleasepool {
        NSLog(@"=== แบบ BAD ===");
        processOrderBad(@"U001", @"P001", 5);
        processOrderBad(nil, @"P001", 5);
        
        NSLog(@"\n=== แบบ GOOD ===");
        processOrderGood(@"U001", @"P001", 5);
        processOrderGood(nil, @"P001", 5);
        processOrderGood(@"U001", @"", 5);
        processOrderGood(@"U001", @"P001", -1);
    }
    return 0;
}
```

**ผลลัพธ์:**
```
=== แบบ BAD ===
ประมวลผลออร์เดอร์: User=U001, Product=P001, Qty=5
userId ต้องไม่เป็น nil

=== แบบ GOOD ===
ประมวลผลออร์เดอร์: User=U001, Product=P001, Qty=5
Error: userId ต้องไม่เป็น nil
Error: productId ต้องไม่ว่าง
Error: จำนวนสินค้าต้องมากกว่า 0
```

### Guard Clauses ในระบบจริง

```objc
#import <Foundation/Foundation.h>

// ตัวอย่าง: ระบบสมัครสมาชิก
typedef struct {
    BOOL success;
    NSString *message;
} RegistrationResult;

void registerUser(NSString *username, NSString *email, NSString *password) {
    // Guard: ตรวจสอบ username
    if (username == nil || [username length] == 0) {
        NSLog(@"Error: ชื่อผู้ใช้ต้องไม่ว่างเปล่า");
        return;
    }
    
    if ([username length] < 3) {
        NSLog(@"Error: ชื่อผู้ใช้ต้องมีอย่างน้อย 3 ตัวอักษร");
        return;
    }
    
    // Guard: ตรวจสอบ email
    if (email == nil || [email length] == 0) {
        NSLog(@"Error: อีเมลต้องไม่ว่างเปล่า");
        return;
    }
    
    if ([email rangeOfString:@"@"].location == NSNotFound) {
        NSLog(@"Error: รูปแบบอีเมลไม่ถูกต้อง");
        return;
    }
    
    // Guard: ตรวจสอบ password
    if (password == nil || [password length] == 0) {
        NSLog(@"Error: รหัสผ่านต้องไม่ว่างเปล่า");
        return;
    }
    
    if ([password length] < 8) {
        NSLog(@"Error: รหัสผ่านต้องมีอย่างน้อย 8 ตัวอักษร");
        return;
    }
    
    // ถ้าผ่านทุก guard แล้ว ทำงานหลัก
    NSLog(@"สมัครสมาชิกสำเร็จ!");
    NSLog(@"ชื่อผู้ใช้: %@", username);
    NSLog(@"อีเมล: %@", email);
}

int main(int argc, const char * argv[]) {
    @autoreleasepool {
        NSLog(@"=== ทดสอบการสมัครสมาชิก ===\n");
        
        NSLog(@"ทดสอบ 1: ข้อมูลถูกต้องทั้งหมด");
        registerUser(@"สมชาย123", @"somchai@email.com", @"password123");
        
        NSLog(@"\nทดสอบ 2: ชื่อผู้ใช้สั้นเกินไป");
        registerUser(@"ab", @"test@email.com", @"password123");
        
        NSLog(@"\nทดสอบ 3: อีเมลไม่ถูกต้อง");
        registerUser(@"testuser", @"invalidemail", @"password123");
        
        NSLog(@"\nทดสอบ 4: รหัสผ่านสั้นเกินไป");
        registerUser(@"testuser", @"test@email.com", @"123");
    }
    return 0;
}
```

---

## 4.7 ตัวอย่างโปรแกรมที่ซับซ้อน

### โปรแกรมคำนวณภาษี

```objc
#import <Foundation/Foundation.h>

double calculateTax(double income) {
    double tax = 0;
    
    if (income <= 0) {
        return 0;
    } else if (income <= 150000) {
        tax = 0; // ยกเว้นภาษี
    } else if (income <= 300000) {
        tax = (income - 150000) * 0.05;
    } else if (income <= 500000) {
        tax = 150000 * 0.05 + (income - 300000) * 0.10;
    } else if (income <= 750000) {
        tax = 150000 * 0.05 + 200000 * 0.10 + (income - 500000) * 0.15;
    } else if (income <= 1000000) {
        tax = 150000 * 0.05 + 200000 * 0.10 + 250000 * 0.15 + (income - 750000) * 0.20;
    } else {
        tax = 150000 * 0.05 + 200000 * 0.10 + 250000 * 0.15 + 250000 * 0.20 + (income - 1000000) * 0.35;
    }
    
    return tax;
}

void printTaxSummary(double income) {
    double tax = calculateTax(income);
    double netIncome = income - tax;
    double effectiveRate = (income > 0) ? (tax / income * 100) : 0;
    
    NSLog(@"รายได้: %,.0f บาท", income);
    NSLog(@"ภาษีที่ต้องชำระ: %,.0f บาท", tax);
    NSLog(@"รายได้สุทธิ: %,.0f บาท", netIncome);
    NSLog(@"อัตราภาษีจริง: %.2f%%", effectiveRate);
    NSLog(@"---");
}

int main(int argc, const char * argv[]) {
    @autoreleasepool {
        NSLog(@"=== ระบบคำนวณภาษีเงินได้บุคคลธรรมดา ===\n");
        
        printTaxSummary(100000);
        printTaxSummary(200000);
        printTaxSummary(500000);
        printTaxSummary(800000);
        printTaxSummary(1500000);
    }
    return 0;
}
```

### โปรแกรม ATM จำลอง

```objc
#import <Foundation/Foundation.h>

void performATMOperation(int operation, double balance, double amount) {
    NSLog(@"ยอดคงเหลือปัจจุบัน: %.2f บาท", balance);
    
    switch (operation) {
        case 1: { // ฝากเงิน
            if (amount <= 0) {
                NSLog(@"จำนวนเงินต้องมากกว่า 0");
                break;
            }
            if (amount > 1000000) {
                NSLog(@"ฝากได้สูงสุด 1,000,000 บาทต่อครั้ง");
                break;
            }
            double newBalance = balance + amount;
            NSLog(@"ฝากเงิน %.2f บาท สำเร็จ", amount);
            NSLog(@"ยอดคงเหลือใหม่: %.2f บาท", newBalance);
            break;
        }
        case 2: { // ถอนเงิน
            if (amount <= 0) {
                NSLog(@"จำนวนเงินต้องมากกว่า 0");
                break;
            }
            if (amount > balance) {
                NSLog(@"ยอดเงินไม่พอ ไม่สามารถถอนได้");
                break;
            }
            if ((int)amount % 100 != 0) {
                NSLog(@"จำนวนเงินต้องเป็นทวีคูณของ 100");
                break;
            }
            double newBalance = balance - amount;
            NSLog(@"ถอนเงิน %.2f บาท สำเร็จ", amount);
            NSLog(@"ยอดคงเหลือใหม่: %.2f บาท", newBalance);
            break;
        }
        case 3: // ตรวจสอบยอด
            NSLog(@"ยอดเงินในบัญชี: %.2f บาท", balance);
            break;
        default:
            NSLog(@"รายการไม่ถูกต้อง");
            break;
    }
}

int main(int argc, const char * argv[]) {
    @autoreleasepool {
        double myBalance = 5000.0;
        
        NSLog(@"=== ATM จำลอง ===\n");
        
        NSLog(@"-- ฝากเงิน 1,000 บาท --");
        performATMOperation(1, myBalance, 1000);
        myBalance += 1000;
        
        NSLog(@"\n-- ถอนเงิน 2,000 บาท --");
        performATMOperation(2, myBalance, 2000);
        myBalance -= 2000;
        
        NSLog(@"\n-- ถอนเงิน 10,000 บาท (เกินยอด) --");
        performATMOperation(2, myBalance, 10000);
        
        NSLog(@"\n-- ตรวจสอบยอด --");
        performATMOperation(3, myBalance, 0);
    }
    return 0;
}
```

---

## 4.8 แบบฝึกหัดพร้อมเฉลย

### แบบฝึกหัดที่ 1: ตรวจสอบจำนวนคู่/คี่

**โจทย์:** เขียนโปรแกรมรับตัวเลข แล้วแสดงว่าเป็นเลขคู่หรือเลขคี่ และเป็นบวกหรือลบหรือศูนย์

```objc
#import <Foundation/Foundation.h>

void checkNumber(int num) {
    NSString *parity = (num % 2 == 0) ? @"เลขคู่" : @"เลขคี่";
    NSString *sign;
    
    if (num > 0) sign = @"จำนวนบวก";
    else if (num < 0) sign = @"จำนวนลบ";
    else sign = @"ศูนย์";
    
    NSLog(@"%d เป็น%@และเป็น%@", num, parity, sign);
}

int main(int argc, const char * argv[]) {
    @autoreleasepool {
        checkNumber(4);
        checkNumber(-7);
        checkNumber(0);
        checkNumber(15);
    }
    return 0;
}
```

**ผลลัพธ์:**
```
4 เป็นเลขคู่และเป็นจำนวนบวก
-7 เป็นเลขคี่และเป็นจำนวนลบ
0 เป็นเลขคู่และเป็นศูนย์
15 เป็นเลขคี่และเป็นจำนวบวก
```

---

### แบบฝึกหัดที่ 2: ระบบให้เกรด

**โจทย์:** เขียนฟังก์ชันที่รับคะแนนและคืนเกรด พร้อมข้อความประเมิน

```objc
#import <Foundation/Foundation.h>

void evaluateScore(int score) {
    if (score < 0 || score > 100) {
        NSLog(@"คะแนนไม่ถูกต้อง (ต้อง 0-100)");
        return;
    }
    
    NSString *grade;
    NSString *evaluation;
    
    if (score >= 90) {
        grade = @"A";
        evaluation = @"ยอดเยี่ยม";
    } else if (score >= 80) {
        grade = @"B";
        evaluation = @"ดีมาก";
    } else if (score >= 70) {
        grade = @"C";
        evaluation = @"ดี";
    } else if (score >= 60) {
        grade = @"D";
        evaluation = @"พอใช้";
    } else {
        grade = @"F";
        evaluation = @"ต้องปรับปรุง";
    }
    
    NSLog(@"คะแนน %d | เกรด %@ | ประเมิน: %@", score, grade, evaluation);
}

int main(int argc, const char * argv[]) {
    @autoreleasepool {
        int scores[] = {95, 83, 71, 65, 45, 100, 0};
        int count = sizeof(scores) / sizeof(scores[0]);
        
        for (int i = 0; i < count; i++) {
            evaluateScore(scores[i]);
        }
    }
    return 0;
}
```

---

### แบบฝึกหัดที่ 3: ตรวจสอบปีอธิกสุรทิน

**โจทย์:** เขียนโปรแกรมตรวจสอบว่าปีที่กำหนดเป็นปีอธิกสุรทิน (leap year) หรือไม่

```objc
#import <Foundation/Foundation.h>

BOOL isLeapYear(int year) {
    // กฎ: หาร 400 ลงตัว หรือ (หาร 4 ลงตัว และ ไม่หาร 100 ลงตัว)
    if (year % 400 == 0) return YES;
    if (year % 100 == 0) return NO;
    if (year % 4 == 0) return YES;
    return NO;
}

int main(int argc, const char * argv[]) {
    @autoreleasepool {
        int years[] = {2000, 1900, 2024, 2023, 1600, 2100};
        int count = sizeof(years) / sizeof(years[0]);
        
        for (int i = 0; i < count; i++) {
            int year = years[i];
            NSLog(@"ปี %d: %@", year, isLeapYear(year) ? @"ปีอธิกสุรทิน" : @"ปีปกติ");
        }
    }
    return 0;
}
```

**ผลลัพธ์:**
```
ปี 2000: ปีอธิกสุรทิน
ปี 1900: ปีปกติ
ปี 2024: ปีอธิกสุรทิน
ปี 2023: ปีปกติ
ปี 1600: ปีอธิกสุรทิน
ปี 2100: ปีปกติ
```

---

### แบบฝึกหัดที่ 4: เครื่องคิดเลข

**โจทย์:** เขียนเครื่องคิดเลขที่รองรับ +, -, *, / ด้วย switch-case

```objc
#import <Foundation/Foundation.h>

void calculate(double a, char op, double b) {
    double result;
    BOOL valid = YES;
    
    switch (op) {
        case '+':
            result = a + b;
            break;
        case '-':
            result = a - b;
            break;
        case '*':
            result = a * b;
            break;
        case '/':
            if (b == 0) {
                NSLog(@"Error: ไม่สามารถหารด้วยศูนย์ได้");
                valid = NO;
            } else {
                result = a / b;
            }
            break;
        default:
            NSLog(@"Error: ตัวดำเนินการ '%c' ไม่รู้จัก", op);
            valid = NO;
            break;
    }
    
    if (valid) {
        NSLog(@"%.2f %c %.2f = %.2f", a, op, b, result);
    }
}

int main(int argc, const char * argv[]) {
    @autoreleasepool {
        calculate(10, '+', 5);
        calculate(10, '-', 3);
        calculate(4, '*', 7);
        calculate(20, '/', 4);
        calculate(10, '/', 0);
        calculate(10, '%', 3);
    }
    return 0;
}
```

---

### แบบฝึกหัดที่ 5: ระบบจำแนกอายุ

**โจทย์:** จำแนกอายุออกเป็นกลุ่มต่างๆ

```objc
#import <Foundation/Foundation.h>

void classifyAge(int age) {
    if (age < 0) {
        NSLog(@"อายุไม่ถูกต้อง");
        return;
    }
    
    NSString *group;
    NSString *advice;
    
    if (age < 1) {
        group = @"ทารก";
        advice = @"ต้องการการดูแลอย่างใกล้ชิด";
    } else if (age < 3) {
        group = @"เด็กวัยเตาะแตะ";
        advice = @"กำลังพัฒนาทักษะการเดินและพูด";
    } else if (age < 12) {
        group = @"เด็ก";
        advice = @"วัยเรียนรู้และสำรวจโลก";
    } else if (age < 18) {
        group = @"วัยรุ่น";
        advice = @"วัยพัฒนาตัวตนและสร้างความสัมพันธ์";
    } else if (age < 30) {
        group = @"ผู้ใหญ่ตอนต้น";
        advice = @"วัยสร้างฐานอาชีพและครอบครัว";
    } else if (age < 60) {
        group = @"ผู้ใหญ่";
        advice = @"วัยทำงานและรับผิดชอบ";
    } else {
        group = @"ผู้สูงอายุ";
        advice = @"วัยพักผ่อนและถ่ายทอดประสบการณ์";
    }
    
    NSLog(@"อายุ %d ปี: กลุ่ม \"%@\" - %@", age, group, advice);
}

int main(int argc, const char * argv[]) {
    @autoreleasepool {
        classifyAge(0);
        classifyAge(2);
        classifyAge(8);
        classifyAge(16);
        classifyAge(25);
        classifyAge(45);
        classifyAge(70);
    }
    return 0;
}
```

---

### แบบฝึกหัดที่ 6: ตรวจสอบรหัสผ่าน

**โจทย์:** ตรวจสอบความแข็งแกร่งของรหัสผ่าน

```objc
#import <Foundation/Foundation.h>

void checkPasswordStrength(NSString *password) {
    if (password == nil) {
        NSLog(@"รหัสผ่านเป็น nil");
        return;
    }
    
    NSUInteger length = [password length];
    BOOL hasUppercase = NO;
    BOOL hasLowercase = NO;
    BOOL hasNumber = NO;
    BOOL hasSpecial = NO;
    
    NSCharacterSet *uppercaseSet = [NSCharacterSet uppercaseLetterCharacterSet];
    NSCharacterSet *lowercaseSet = [NSCharacterSet lowercaseLetterCharacterSet];
    NSCharacterSet *digitSet = [NSCharacterSet decimalDigitCharacterSet];
    NSCharacterSet *specialSet = [NSCharacterSet punctuationCharacterSet];
    
    for (int i = 0; i < length; i++) {
        unichar c = [password characterAtIndex:i];
        if ([uppercaseSet characterIsMember:c]) hasUppercase = YES;
        if ([lowercaseSet characterIsMember:c]) hasLowercase = YES;
        if ([digitSet characterIsMember:c]) hasNumber = YES;
        if ([specialSet characterIsMember:c]) hasSpecial = YES;
    }
    
    int strengthScore = 0;
    NSMutableArray *feedback = [NSMutableArray array];
    
    if (length >= 8) { strengthScore++; }
    else { [feedback addObject:@"ต้องมีอย่างน้อย 8 ตัวอักษร"]; }
    
    if (length >= 12) { strengthScore++; }
    
    if (hasUppercase) { strengthScore++; }
    else { [feedback addObject:@"ควรมีตัวอักษรพิมพ์ใหญ่"]; }
    
    if (hasLowercase) { strengthScore++; }
    else { [feedback addObject:@"ควรมีตัวอักษรพิมพ์เล็ก"]; }
    
    if (hasNumber) { strengthScore++; }
    else { [feedback addObject:@"ควรมีตัวเลข"]; }
    
    if (hasSpecial) { strengthScore++; }
    else { [feedback addObject:@"ควรมีอักขระพิเศษ"]; }
    
    NSString *strength;
    if (strengthScore <= 2) strength = @"อ่อนแอ";
    else if (strengthScore <= 4) strength = @"ปานกลาง";
    else strength = @"แข็งแกร่ง";
    
    NSLog(@"รหัสผ่าน: %@ | ความแข็งแกร่ง: %@ (%d/6)",
          password, strength, strengthScore);
    
    if ([feedback count] > 0) {
        NSLog(@"คำแนะนำ: %@", [feedback componentsJoinedByString:@", "]);
    }
}

int main(int argc, const char * argv[]) {
    @autoreleasepool {
        checkPasswordStrength(@"123");
        checkPasswordStrength(@"password");
        checkPasswordStrength(@"Password1");
        checkPasswordStrength(@"P@ssw0rd!");
        checkPasswordStrength(@"MyStr0ng!Password");
    }
    return 0;
}
```

---

### แบบฝึกหัดที่ 7: ระบบจองตั๋วภาพยนตร์

**โจทย์:** คำนวณราคาตั๋วตามประเภทผู้ชมและเวลา

```objc
#import <Foundation/Foundation.h>

double calculateTicketPrice(NSString *viewerType, int hour) {
    // ราคาฐาน
    double basePrice;
    
    if ([viewerType isEqualToString:@"เด็ก"]) {
        basePrice = 80.0;
    } else if ([viewerType isEqualToString:@"ผู้สูงอายุ"] || 
               [viewerType isEqualToString:@"นักศึกษา"]) {
        basePrice = 100.0;
    } else {
        basePrice = 150.0; // ผู้ใหญ่ทั่วไป
    }
    
    // ส่วนลดตามเวลา
    double discount = 0;
    if (hour >= 10 && hour < 14) {
        discount = 0.20; // รอบเช้า ลด 20%
    } else if (hour >= 22 || hour < 6) {
        discount = 0.10; // รอบดึก ลด 10%
    }
    
    double finalPrice = basePrice * (1 - discount);
    NSLog(@"ประเภท: %@ | เวลา %02d:00 | ราคา: %.0f บาท", 
          viewerType, hour, finalPrice);
    
    return finalPrice;
}

int main(int argc, const char * argv[]) {
    @autoreleasepool {
        NSLog(@"=== ราคาตั๋วภาพยนตร์ ===");
        calculateTicketPrice(@"ผู้ใหญ่", 11);
        calculateTicketPrice(@"ผู้ใหญ่", 19);
        calculateTicketPrice(@"เด็ก", 11);
        calculateTicketPrice(@"นักศึกษา", 14);
        calculateTicketPrice(@"ผู้สูงอายุ", 23);
    }
    return 0;
}
```

---

### แบบฝึกหัดที่ 8: Game Rock-Paper-Scissors

**โจทย์:** เกมเป่ายิ้งฉุบ

```objc
#import <Foundation/Foundation.h>

void playRPS(int playerChoice, int computerChoice) {
    NSArray *choices = @[@"ค้อน", @"กรรไกร", @"กระดาษ"];
    
    if (playerChoice < 0 || playerChoice > 2 || 
        computerChoice < 0 || computerChoice > 2) {
        NSLog(@"การเลือกไม่ถูกต้อง");
        return;
    }
    
    NSLog(@"ผู้เล่น: %@ vs คอมพิวเตอร์: %@",
          choices[playerChoice], choices[computerChoice]);
    
    if (playerChoice == computerChoice) {
        NSLog(@"ผลลัพธ์: เสมอ!");
    } else if ((playerChoice == 0 && computerChoice == 1) ||
               (playerChoice == 1 && computerChoice == 2) ||
               (playerChoice == 2 && computerChoice == 0)) {
        NSLog(@"ผลลัพธ์: ผู้เล่นชนะ!");
    } else {
        NSLog(@"ผลลัพธ์: คอมพิวเตอร์ชนะ!");
    }
}

int main(int argc, const char * argv[]) {
    @autoreleasepool {
        NSLog(@"=== เกมเป่ายิ้งฉุบ ===");
        playRPS(0, 1); // ค้อน vs กรรไกร
        playRPS(1, 2); // กรรไกร vs กระดาษ
        playRPS(2, 0); // กระดาษ vs ค้อน
        playRPS(0, 0); // เสมอ
        playRPS(1, 0); // กรรไกร vs ค้อน
    }
    return 0;
}
```

---

### แบบฝึกหัดที่ 9: ระบบจำแนกอุณหภูมิ

**โจทย์:** รับอุณหภูมิและแสดงข้อมูลที่เกี่ยวข้อง

```objc
#import <Foundation/Foundation.h>

void analyzeTemperature(double celsius) {
    NSString *description;
    NSString *advice;
    NSString *waterState;
    
    // จำแนกอุณหภูมิ
    if (celsius < -20) {
        description = @"หนาวจัดมาก";
        advice = @"อยู่แต่ในบ้าน แต่งตัวให้อุ่น";
    } else if (celsius < 0) {
        description = @"หนาวจัด";
        advice = @"ใส่เสื้อหนาหลายชั้น";
    } else if (celsius < 10) {
        description = @"หนาว";
        advice = @"ใส่เสื้อกันหนาว";
    } else if (celsius < 20) {
        description = @"เย็น";
        advice = @"ใส่เสื้อแจ็คเก็ต";
    } else if (celsius < 30) {
        description = @"สบาย";
        advice = @"อากาศดีเหมาะกับการออกไปข้างนอก";
    } else if (celsius < 35) {
        description = @"อุ่น";
        advice = @"ดื่มน้ำเยอะๆ";
    } else if (celsius < 40) {
        description = @"ร้อน";
        advice = @"หลีกเลี่ยงแดดจัด ดื่มน้ำมากๆ";
    } else {
        description = @"ร้อนจัด";
        advice = @"อันตราย! ควรอยู่ในที่ร่ม";
    }
    
    // สถานะของน้ำ
    if (celsius <= 0) {
        waterState = @"แข็ง (น้ำแข็ง)";
    } else if (celsius < 100) {
        waterState = @"ของเหลว (น้ำ)";
    } else {
        waterState = @"ก๊าซ (ไอน้ำ)";
    }
    
    double fahrenheit = celsius * 9.0/5.0 + 32;
    double kelvin = celsius + 273.15;
    
    NSLog(@"อุณหภูมิ: %.1f°C = %.1f°F = %.2fK", celsius, fahrenheit, kelvin);
    NSLog(@"สภาพอากาศ: %@", description);
    NSLog(@"คำแนะนำ: %@", advice);
    NSLog(@"สถานะน้ำ: %@", waterState);
    NSLog(@"---");
}

int main(int argc, const char * argv[]) {
    @autoreleasepool {
        analyzeTemperature(-25);
        analyzeTemperature(0);
        analyzeTemperature(25);
        analyzeTemperature(38);
        analyzeTemperature(100);
    }
    return 0;
}
```

---

### แบบฝึกหัดที่ 10: ระบบแนะนำสินค้า

**โจทย์:** แนะนำสินค้าตามงบประมาณและความต้องการ

```objc
#import <Foundation/Foundation.h>

void recommendProduct(NSString *category, double budget, BOOL forBusiness) {
    NSLog(@"หมวดหมู่: %@ | งบ: %.0f บาท | ธุรกิจ: %@",
          category, budget, forBusiness ? @"ใช่" : @"ไม่ใช่");
    
    NSString *recommendation;
    NSString *topPick;
    
    if ([category isEqualToString:@"laptop"]) {
        if (budget < 15000) {
            recommendation = @"โน้ตบุ๊คสำหรับงานพื้นฐาน";
            topPick = @"Acer Aspire 3 หรือ ASUS VivoBook";
        } else if (budget < 30000) {
            if (forBusiness) {
                recommendation = @"โน้ตบุ๊คธุรกิจระดับกลาง";
                topPick = @"Dell Latitude หรือ Lenovo ThinkPad E Series";
            } else {
                recommendation = @"โน้ตบุ๊คสำหรับงานทั่วไป";
                topPick = @"HP Pavilion หรือ ASUS ZenBook";
            }
        } else {
            if (forBusiness) {
                recommendation = @"โน้ตบุ๊คธุรกิจระดับสูง";
                topPick = @"Apple MacBook Pro หรือ Dell XPS";
            } else {
                recommendation = @"โน้ตบุ๊คประสิทธิภาพสูง";
                topPick = @"Apple MacBook Air M2 หรือ ASUS ROG";
            }
        }
    } else if ([category isEqualToString:@"phone"]) {
        if (budget < 5000) {
            recommendation = @"สมาร์ทโฟนราคาประหยัด";
            topPick = @"Samsung Galaxy A13 หรือ Redmi 12";
        } else if (budget < 15000) {
            recommendation = @"สมาร์ทโฟนระดับกลาง";
            topPick = @"Samsung Galaxy A54 หรือ iPhone SE";
        } else {
            recommendation = @"สมาร์ทโฟนเรือธง";
            topPick = @"iPhone 15 Pro หรือ Samsung Galaxy S24";
        }
    } else {
        recommendation = @"ไม่พบหมวดหมู่สินค้านี้";
        topPick = @"กรุณาระบุ laptop หรือ phone";
    }
    
    NSLog(@"แนะนำ: %@", recommendation);
    NSLog(@"สินค้าแนะนำ: %@", topPick);
    NSLog(@"---");
}

int main(int argc, const char * argv[]) {
    @autoreleasepool {
        NSLog(@"=== ระบบแนะนำสินค้า ===\n");
        recommendProduct(@"laptop", 12000, NO);
        recommendProduct(@"laptop", 25000, YES);
        recommendProduct(@"phone", 8000, NO);
        recommendProduct(@"phone", 35000, YES);
        recommendProduct(@"tablet", 15000, NO);
    }
    return 0;
}
```

---

## สรุปบทที่ 4

ในบทนี้เราได้เรียนรู้:

1. **if/else if/else** - การตัดสินใจแบบพื้นฐาน
2. **switch-case** - การเลือกจากหลายตัวเลือก พร้อม fall-through
3. **Nested conditions** - เงื่อนไขซ้อนกัน
4. **Boolean expressions** - การใช้ &&, ||, ! และ ternary operator
5. **Short-circuit evaluation** - การลดการประเมินเงื่อนไขที่ไม่จำเป็น
6. **Guard clauses** - รูปแบบการเขียนโค้ดที่อ่านง่ายและป้องกันข้อผิดพลาด

### เคล็ดลับสำคัญ

- ใช้ **Guard clauses** แทน nested if เพื่อให้โค้ดอ่านง่ายขึ้น
- เปรียบเทียบ `NSString` ด้วย `isEqualToString:` ไม่ใช่ `==`
- ใช้ **short-circuit** ป้องกัน nil pointer dereference
- ระวัง **fall-through** ใน switch-case ลืม `break`
- ใช้ `BOOL` สำหรับค่าบูลีน ไม่ใช่ int

---

*บทต่อไป: ส่วนที่ 05 - การทำซ้ำ (Loops)*
