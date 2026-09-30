# ส่วนที่ 05: การทำซ้ำ (Loops) ใน Objective-C

## บทนำ

การทำซ้ำ (Loops) ช่วยให้เราสามารถทำงานซ้ำๆ ได้โดยไม่ต้องเขียนโค้ดซ้ำกันหลายครั้ง ใน Objective-C มีหลายรูปแบบการวนซ้ำที่เหมาะสมกับงานแต่ละประเภท ในบทนี้เราจะศึกษาทุกรูปแบบพร้อมตัวอย่างการใช้งานจริง

---

## 5.1 for Loop แบบ C-style

### รูปแบบพื้นฐาน

```
for (initialization; condition; update) {
    // code block
}
```

```objc
#import <Foundation/Foundation.h>

int main(int argc, const char * argv[]) {
    @autoreleasepool {
        // for loop พื้นฐาน: นับ 1 ถึง 10
        NSLog(@"=== นับ 1 ถึง 10 ===");
        for (int i = 1; i <= 10; i++) {
            NSLog(@"%d", i);
        }
        
        // นับถอยหลัง
        NSLog(@"\n=== นับถอยหลัง ===");
        for (int i = 10; i >= 1; i--) {
            NSLog(@"%d", i);
        }
        
        // นับทีละ 2
        NSLog(@"\n=== เลขคู่ 2-20 ===");
        for (int i = 2; i <= 20; i += 2) {
            NSLog(@"%d", i);
        }
    }
    return 0;
}
```

### for loop กับ Array

```objc
#import <Foundation/Foundation.h>

int main(int argc, const char * argv[]) {
    @autoreleasepool {
        // C-style array
        int numbers[] = {5, 12, 8, 3, 17, 9, 1, 14};
        int count = sizeof(numbers) / sizeof(numbers[0]);
        
        NSLog(@"=== ค่าในอาร์เรย์ ===");
        for (int i = 0; i < count; i++) {
            NSLog(@"numbers[%d] = %d", i, numbers[i]);
        }
        
        // หาค่ามากที่สุดและน้อยที่สุด
        int max = numbers[0];
        int min = numbers[0];
        int sum = 0;
        
        for (int i = 0; i < count; i++) {
            if (numbers[i] > max) max = numbers[i];
            if (numbers[i] < min) min = numbers[i];
            sum += numbers[i];
        }
        
        NSLog(@"\nค่ามากที่สุด: %d", max);
        NSLog(@"ค่าน้อยที่สุด: %d", min);
        NSLog(@"ผลรวม: %d", sum);
        NSLog(@"ค่าเฉลี่ย: %.2f", (double)sum / count);
    }
    return 0;
}
```

### for loop แบบ Multiple Initialization

```objc
#import <Foundation/Foundation.h>

int main(int argc, const char * argv[]) {
    @autoreleasepool {
        // ตัวแปรหลายตัวใน for loop
        for (int i = 0, j = 10; i < j; i++, j--) {
            NSLog(@"i = %d, j = %d", i, j);
        }
        
        // ใช้ for ค้นหาใน array
        int data[] = {3, 7, 2, 9, 5, 1, 8, 4, 6};
        int size = sizeof(data) / sizeof(data[0]);
        int target = 5;
        int foundIndex = -1;
        
        for (int i = 0; i < size; i++) {
            if (data[i] == target) {
                foundIndex = i;
                break; // หยุดทันทีที่พบ
            }
        }
        
        if (foundIndex >= 0) {
            NSLog(@"พบค่า %d ที่ index %d", target, foundIndex);
        } else {
            NSLog(@"ไม่พบค่า %d", target);
        }
    }
    return 0;
}
```

---

## 5.2 while Loop

### รูปแบบพื้นฐาน

```objc
#import <Foundation/Foundation.h>

int main(int argc, const char * argv[]) {
    @autoreleasepool {
        // while loop พื้นฐาน
        int count = 1;
        while (count <= 5) {
            NSLog(@"ครั้งที่ %d", count);
            count++;
        }
        
        // while กับเงื่อนไขที่ซับซ้อน
        NSLog(@"\n=== หาจำนวนหลักของตัวเลข ===");
        int number = 123456789;
        int digits = 0;
        int temp = number;
        
        while (temp != 0) {
            temp /= 10;
            digits++;
        }
        
        NSLog(@"เลข %d มี %d หลัก", number, digits);
    }
    return 0;
}
```

### while loop กับการประมวลผลข้อมูล

```objc
#import <Foundation/Foundation.h>

int main(int argc, const char * argv[]) {
    @autoreleasepool {
        // คำนวณ Fibonacci sequence
        NSLog(@"=== Fibonacci ===");
        int a = 0, b = 1;
        int limit = 100;
        
        NSLog(@"Fibonacci จนถึง %d:", limit);
        while (a <= limit) {
            NSLog(@"%d", a);
            int next = a + b;
            a = b;
            b = next;
        }
        
        // การหา GCD (Greatest Common Divisor) ด้วย Euclidean Algorithm
        NSLog(@"\n=== หา GCD ===");
        int x = 48, y = 18;
        int originalX = x, originalY = y;
        
        while (y != 0) {
            int remainder = x % y;
            x = y;
            y = remainder;
        }
        
        NSLog(@"GCD(%d, %d) = %d", originalX, originalY, x);
    }
    return 0;
}
```

### while loop สำหรับประมวลผล String

```objc
#import <Foundation/Foundation.h>

int main(int argc, const char * argv[]) {
    @autoreleasepool {
        // นับสระในประโยค
        NSString *sentence = @"Hello World this is Objective C programming";
        NSString *vowels = @"aeiouAEIOU";
        NSUInteger length = [sentence length];
        int vowelCount = 0;
        int i = 0;
        
        while (i < length) {
            unichar ch = [sentence characterAtIndex:i];
            NSString *charStr = [NSString stringWithFormat:@"%c", ch];
            
            if ([vowels containsString:charStr]) {
                vowelCount++;
            }
            i++;
        }
        
        NSLog(@"ประโยค: %@", sentence);
        NSLog(@"จำนวนสระ: %d", vowelCount);
    }
    return 0;
}
```

---

## 5.3 do-while Loop

do-while จะทำงานอย่างน้อย 1 ครั้งเสมอ ก่อนตรวจสอบเงื่อนไข

```objc
#import <Foundation/Foundation.h>

int main(int argc, const char * argv[]) {
    @autoreleasepool {
        // do-while พื้นฐาน
        int count = 1;
        do {
            NSLog(@"ทำซ้ำครั้งที่: %d", count);
            count++;
        } while (count <= 5);
        
        // ความต่างระหว่าง while และ do-while
        NSLog(@"\n=== ทดสอบเงื่อนไขเป็นเท็จตั้งแต่แรก ===");
        
        int x = 10;
        
        NSLog(@"while (เงื่อนไขเท็จตั้งแต่แรก):");
        while (x < 5) {
            NSLog(@"while: ทำงาน x = %d", x);
            x++;
        }
        NSLog(@"while: ไม่ทำงานเลย");
        
        NSLog(@"\ndo-while (เงื่อนไขเท็จตั้งแต่แรก):");
        x = 10;
        do {
            NSLog(@"do-while: ทำงาน x = %d", x); // จะทำงาน 1 ครั้ง!
            x++;
        } while (x < 5);
        NSLog(@"do-while: ทำงาน 1 ครั้งแม้เงื่อนไขเท็จ");
    }
    return 0;
}
```

### ตัวอย่าง do-while: เมนูโปรแกรม

```objc
#import <Foundation/Foundation.h>

void showMenu(void) {
    NSLog(@"\n=== เมนูหลัก ===");
    NSLog(@"1. ดูข้อมูล");
    NSLog(@"2. เพิ่มข้อมูล");
    NSLog(@"3. แก้ไขข้อมูล");
    NSLog(@"0. ออกจากโปรแกรม");
}

int main(int argc, const char * argv[]) {
    @autoreleasepool {
        // จำลองการใช้เมนู (ในโปรแกรมจริงจะรับค่าจากผู้ใช้)
        int choices[] = {1, 2, 3, 0}; // จำลอง input
        int choiceIndex = 0;
        int choice;
        
        do {
            showMenu();
            choice = choices[choiceIndex++]; // ดึงค่า simulated
            NSLog(@"เลือก: %d", choice);
            
            switch (choice) {
                case 1:
                    NSLog(@"กำลังดูข้อมูล...");
                    break;
                case 2:
                    NSLog(@"กำลังเพิ่มข้อมูล...");
                    break;
                case 3:
                    NSLog(@"กำลังแก้ไขข้อมูล...");
                    break;
                case 0:
                    NSLog(@"ออกจากโปรแกรม");
                    break;
                default:
                    NSLog(@"กรุณาเลือกตัวเลข 0-3");
                    break;
            }
        } while (choice != 0);
    }
    return 0;
}
```

---

## 5.4 for-in Loop (Fast Enumeration)

for-in ใน Objective-C ใช้กับ NSArray, NSDictionary, NSSet และ Collection อื่นๆ

### for-in กับ NSArray

```objc
#import <Foundation/Foundation.h>

int main(int argc, const char * argv[]) {
    @autoreleasepool {
        // for-in กับ NSArray of Strings
        NSArray *fruits = @[@"แอปเปิ้ล", @"กล้วย", @"ส้ม", @"มะม่วง", @"องุ่น"];
        
        NSLog(@"=== รายการผลไม้ ===");
        for (NSString *fruit in fruits) {
            NSLog(@"- %@", fruit);
        }
        
        // for-in กับ NSArray of Numbers
        NSArray *numbers = @[@10, @25, @5, @30, @15];
        int total = 0;
        int max = [numbers[0] intValue];
        
        for (NSNumber *num in numbers) {
            int value = [num intValue];
            total += value;
            if (value > max) max = value;
        }
        
        NSLog(@"\nผลรวม: %d", total);
        NSLog(@"ค่ามากที่สุด: %d", max);
        NSLog(@"ค่าเฉลี่ย: %.2f", (double)total / [numbers count]);
    }
    return 0;
}
```

### for-in กับ NSDictionary

```objc
#import <Foundation/Foundation.h>

int main(int argc, const char * argv[]) {
    @autoreleasepool {
        NSDictionary *studentScores = @{
            @"สมชาย": @85,
            @"สมหญิง": @92,
            @"อนันต์": @78,
            @"วิภา": @95,
            @"ประสิทธิ์": @88
        };
        
        NSLog(@"=== คะแนนนักเรียน ===");
        double totalScore = 0;
        int count = 0;
        
        for (NSString *studentName in studentScores) {
            NSNumber *score = studentScores[studentName];
            NSLog(@"%@: %@", studentName, score);
            totalScore += [score doubleValue];
            count++;
        }
        
        NSLog(@"\nคะแนนเฉลี่ย: %.2f", totalScore / count);
        
        // วนซ้ำผ่าน keys เท่านั้น
        NSLog(@"\n=== รายชื่อนักเรียน ===");
        for (NSString *name in [studentScores allKeys]) {
            NSLog(@"- %@", name);
        }
        
        // วนซ้ำผ่าน values เท่านั้น
        NSLog(@"\n=== คะแนนทั้งหมด ===");
        for (NSNumber *score in [studentScores allValues]) {
            NSLog(@"- %@", score);
        }
    }
    return 0;
}
```

### for-in กับ NSSet

```objc
#import <Foundation/Foundation.h>

int main(int argc, const char * argv[]) {
    @autoreleasepool {
        NSSet *uniqueColors = [NSSet setWithArray:@[
            @"แดง", @"น้ำเงิน", @"เขียว", @"แดง", @"เหลือง", @"น้ำเงิน"
        ]];
        
        NSLog(@"สีที่ไม่ซ้ำกัน (%lu สี):", (unsigned long)[uniqueColors count]);
        for (NSString *color in uniqueColors) {
            NSLog(@"- %@", color);
        }
    }
    return 0;
}
```

---

## 5.5 break และ continue

### break - หยุดออกจาก Loop

```objc
#import <Foundation/Foundation.h>

int main(int argc, const char * argv[]) {
    @autoreleasepool {
        // break ใน for loop
        NSLog(@"=== ค้นหาค่าแรกที่หารด้วย 7 ลงตัว ===");
        for (int i = 1; i <= 100; i++) {
            if (i % 7 == 0) {
                NSLog(@"พบ: %d", i);
                break; // หยุดทันที
            }
        }
        
        // break ใน while loop
        NSLog(@"\n=== ค้นหาตัวเลขใน array ===");
        NSArray *data = @[@3, @7, @2, @15, @8, @1, @9, @4];
        int searchFor = 15;
        BOOL found = NO;
        int foundAt = -1;
        
        int i = 0;
        while (i < [data count]) {
            if ([data[i] intValue] == searchFor) {
                found = YES;
                foundAt = i;
                break;
            }
            i++;
        }
        
        if (found) {
            NSLog(@"พบ %d ที่ตำแหน่ง %d", searchFor, foundAt);
        } else {
            NSLog(@"ไม่พบ %d", searchFor);
        }
    }
    return 0;
}
```

### continue - ข้ามการทำซ้ำปัจจุบัน

```objc
#import <Foundation/Foundation.h>

int main(int argc, const char * argv[]) {
    @autoreleasepool {
        // continue ข้ามเลขคี่ แสดงเฉพาะเลขคู่
        NSLog(@"=== เลขคู่ 1-20 ===");
        for (int i = 1; i <= 20; i++) {
            if (i % 2 != 0) {
                continue; // ข้ามไปถ้าเป็นเลขคี่
            }
            NSLog(@"%d", i);
        }
        
        // continue กับ for-in
        NSLog(@"\n=== กรองค่าลบออก ===");
        NSArray *values = @[@5, @-3, @8, @-1, @12, @-7, @3, @9];
        int positiveSum = 0;
        
        for (NSNumber *num in values) {
            if ([num intValue] < 0) {
                continue; // ข้ามค่าลบ
            }
            positiveSum += [num intValue];
            NSLog(@"เพิ่ม: %@", num);
        }
        
        NSLog(@"ผลรวมค่าบวก: %d", positiveSum);
    }
    return 0;
}
```

### break และ continue ในกรณีซับซ้อน

```objc
#import <Foundation/Foundation.h>

int main(int argc, const char * argv[]) {
    @autoreleasepool {
        // ตัวอย่าง: กรองและประมวลผลข้อมูลพนักงาน
        NSArray *employees = @[
            @{@"name": @"สมชาย", @"salary": @25000, @"active": @YES},
            @{@"name": @"สมหญิง", @"salary": @35000, @"active": @YES},
            @{@"name": @"อนันต์",  @"salary": @28000, @"active": @NO},
            @{@"name": @"วิภา",    @"salary": @42000, @"active": @YES},
            @{@"name": @"ประสิทธิ์", @"salary": @55000, @"active": @YES}
        ];
        
        double totalSalary = 0;
        int activeCount = 0;
        double highSalaryCap = 40000;
        
        NSLog(@"=== พนักงานที่ active และเงินเดือนไม่เกิน %.0f ===", highSalaryCap);
        
        for (NSDictionary *emp in employees) {
            // ข้ามพนักงานที่ไม่ active
            if (![emp[@"active"] boolValue]) {
                NSLog(@"ข้าม: %@ (inactive)", emp[@"name"]);
                continue;
            }
            
            double salary = [emp[@"salary"] doubleValue];
            
            // หยุดถ้าพบเงินเดือนสูงเกินไป
            if (salary > highSalaryCap) {
                NSLog(@"หยุดที่: %@ (เงินเดือนสูงเกินไป: %.0f)", 
                      emp[@"name"], salary);
                break;
            }
            
            totalSalary += salary;
            activeCount++;
            NSLog(@"นับ: %@ (%.0f บาท)", emp[@"name"], salary);
        }
        
        NSLog(@"\nสรุป:");
        NSLog(@"จำนวนพนักงาน: %d คน", activeCount);
        NSLog(@"ยอดรวมเงินเดือน: %.0f บาท", totalSalary);
    }
    return 0;
}
```

---

## 5.6 Nested Loops (Loop ซ้อน)

### Loop ซ้อนพื้นฐาน

```objc
#import <Foundation/Foundation.h>

int main(int argc, const char * argv[]) {
    @autoreleasepool {
        // ตาราง Multiplication
        NSLog(@"=== ตาราง คูณ ===");
        NSLog(@"    ");
        
        // สร้าง header
        NSMutableString *header = [NSMutableString stringWithString:@"   |"];
        for (int j = 1; j <= 10; j++) {
            [header appendFormat:@" %3d", j];
        }
        NSLog(@"%@", header);
        
        // เส้นคั่น
        NSLog(@"---+----------------------------------------");
        
        for (int i = 1; i <= 10; i++) {
            NSMutableString *row = [NSMutableString stringWithFormat:@"%2d |", i];
            for (int j = 1; j <= 10; j++) {
                [row appendFormat:@" %3d", i * j];
            }
            NSLog(@"%@", row);
        }
    }
    return 0;
}
```

### Pattern Printing ด้วย Nested Loop

```objc
#import <Foundation/Foundation.h>

int main(int argc, const char * argv[]) {
    @autoreleasepool {
        int n = 5;
        
        // รูปสามเหลี่ยมชิดซ้าย
        NSLog(@"=== สามเหลี่ยมชิดซ้าย ===");
        for (int i = 1; i <= n; i++) {
            NSMutableString *row = [NSMutableString string];
            for (int j = 1; j <= i; j++) {
                [row appendString:@"* "];
            }
            NSLog(@"%@", row);
        }
        
        // รูปสามเหลี่ยมหัวกลับ
        NSLog(@"\n=== สามเหลี่ยมหัวกลับ ===");
        for (int i = n; i >= 1; i--) {
            NSMutableString *row = [NSMutableString string];
            for (int j = 1; j <= i; j++) {
                [row appendString:@"* "];
            }
            NSLog(@"%@", row);
        }
        
        // สี่เหลี่ยมกลวง
        NSLog(@"\n=== สี่เหลี่ยมกลวง ===");
        for (int i = 1; i <= n; i++) {
            NSMutableString *row = [NSMutableString string];
            for (int j = 1; j <= n; j++) {
                if (i == 1 || i == n || j == 1 || j == n) {
                    [row appendString:@"* "];
                } else {
                    [row appendString:@"  "];
                }
            }
            NSLog(@"%@", row);
        }
    }
    return 0;
}
```

### Nested Loop กับ 2D Array

```objc
#import <Foundation/Foundation.h>

int main(int argc, const char * argv[]) {
    @autoreleasepool {
        // เมทริกซ์ 3x3
        int matrix[3][3] = {
            {1, 2, 3},
            {4, 5, 6},
            {7, 8, 9}
        };
        
        NSLog(@"=== เมทริกซ์ต้นฉบับ ===");
        for (int i = 0; i < 3; i++) {
            NSMutableString *row = [NSMutableString string];
            for (int j = 0; j < 3; j++) {
                [row appendFormat:@"%3d", matrix[i][j]];
            }
            NSLog(@"%@", row);
        }
        
        // คำนวณ Transpose
        int transpose[3][3];
        for (int i = 0; i < 3; i++) {
            for (int j = 0; j < 3; j++) {
                transpose[j][i] = matrix[i][j];
            }
        }
        
        NSLog(@"\n=== Transpose ===");
        for (int i = 0; i < 3; i++) {
            NSMutableString *row = [NSMutableString string];
            for (int j = 0; j < 3; j++) {
                [row appendFormat:@"%3d", transpose[i][j]];
            }
            NSLog(@"%@", row);
        }
        
        // ผลรวม diagonal
        int diagSum = 0;
        for (int i = 0; i < 3; i++) {
            diagSum += matrix[i][i];
        }
        NSLog(@"\nผลรวม diagonal: %d", diagSum);
    }
    return 0;
}
```

### Nested Loop กับ break ออกจาก outer loop

```objc
#import <Foundation/Foundation.h>

int main(int argc, const char * argv[]) {
    @autoreleasepool {
        // ค้นหาใน 2D array - ต้องการออกจาก outer loop ด้วย
        int grid[4][4] = {
            {1,  2,  3,  4},
            {5,  6,  7,  8},
            {9,  10, 11, 12},
            {13, 14, 15, 16}
        };
        
        int target = 11;
        int foundRow = -1, foundCol = -1;
        BOOL found = NO;
        
        for (int row = 0; row < 4 && !found; row++) {
            for (int col = 0; col < 4; col++) {
                if (grid[row][col] == target) {
                    foundRow = row;
                    foundCol = col;
                    found = YES;
                    break; // ออกจาก inner loop
                    // แต่ยังอยู่ใน outer loop
                    // outer loop จะหยุดเพราะ !found เป็น NO
                }
            }
        }
        
        if (found) {
            NSLog(@"พบ %d ที่ [%d][%d]", target, foundRow, foundCol);
        }
        
        // วิธีอื่น: ใช้ goto (ไม่แนะนำ แต่เป็น valid syntax)
        found = NO;
        for (int row = 0; row < 4; row++) {
            for (int col = 0; col < 4; col++) {
                if (grid[row][col] == 7) {
                    foundRow = row;
                    foundCol = col;
                    found = YES;
                    goto done; // ออกจาก nested loop ทันที
                }
            }
        }
        done:
        if (found) {
            NSLog(@"พบ 7 ที่ [%d][%d] (ด้วย goto)", foundRow, foundCol);
        }
    }
    return 0;
}
```

---

## 5.7 Loop กับ NSArray และ Collections

### การใช้ enumerateObjectsUsingBlock

```objc
#import <Foundation/Foundation.h>

int main(int argc, const char * argv[]) {
    @autoreleasepool {
        NSArray *colors = @[@"แดง", @"น้ำเงิน", @"เขียว", @"เหลือง", @"ม่วง"];
        
        // วิธี 1: for-in
        NSLog(@"=== for-in ===");
        for (NSString *color in colors) {
            NSLog(@"- %@", color);
        }
        
        // วิธี 2: enumerateObjectsUsingBlock (พร้อม index)
        NSLog(@"\n=== enumerateObjectsUsingBlock ===");
        [colors enumerateObjectsUsingBlock:^(id obj, NSUInteger idx, BOOL *stop) {
            NSLog(@"%lu: %@", (unsigned long)idx, obj);
            
            // สามารถ stop การวนซ้ำได้
            if (idx == 2) {
                *stop = YES; // หยุดที่ index 2
            }
        }];
        
        // วิธี 3: วนซ้ำย้อนหลัง
        NSLog(@"\n=== วนซ้ำย้อนหลัง ===");
        [colors enumerateObjectsWithOptions:NSEnumerationReverse
                                 usingBlock:^(id obj, NSUInteger idx, BOOL *stop) {
            NSLog(@"%lu: %@", (unsigned long)idx, obj);
        }];
    }
    return 0;
}
```

### การกรองข้อมูลด้วย Loop

```objc
#import <Foundation/Foundation.h>

int main(int argc, const char * argv[]) {
    @autoreleasepool {
        NSArray *products = @[
            @{@"name": @"แอปเปิ้ล",   @"price": @25,  @"inStock": @YES},
            @{@"name": @"มะม่วง",     @"price": @40,  @"inStock": @NO},
            @{@"name": @"องุ่น",       @"price": @120, @"inStock": @YES},
            @{@"name": @"กล้วย",      @"price": @15,  @"inStock": @YES},
            @{@"name": @"ทุเรียน",    @"price": @450, @"inStock": @NO},
            @{@"name": @"สตรอเบอร์รี่", @"price": @85, @"inStock": @YES}
        ];
        
        // กรองสินค้าที่มีในคลัง ราคาไม่เกิน 100 บาท
        NSMutableArray *affordable = [NSMutableArray array];
        
        for (NSDictionary *product in products) {
            BOOL inStock = [product[@"inStock"] boolValue];
            double price = [product[@"price"] doubleValue];
            
            if (inStock && price <= 100) {
                [affordable addObject:product];
            }
        }
        
        NSLog(@"=== สินค้าในคลัง ราคาไม่เกิน 100 บาท ===");
        for (NSDictionary *product in affordable) {
            NSLog(@"%@ - %.0f บาท", product[@"name"], [product[@"price"] doubleValue]);
        }
        
        // หาราคาเฉลี่ยของสินค้าในคลัง
        double totalPrice = 0;
        int stockCount = 0;
        
        for (NSDictionary *product in products) {
            if ([product[@"inStock"] boolValue]) {
                totalPrice += [product[@"price"] doubleValue];
                stockCount++;
            }
        }
        
        NSLog(@"\nราคาเฉลี่ยสินค้าในคลัง: %.2f บาท",
              stockCount > 0 ? totalPrice / stockCount : 0);
    }
    return 0;
}
```

---

## 5.8 Performance Considerations

### เปรียบเทียบประสิทธิภาพของ Loop

```objc
#import <Foundation/Foundation.h>

int main(int argc, const char * argv[]) {
    @autoreleasepool {
        NSMutableArray *largeArray = [NSMutableArray arrayWithCapacity:100000];
        for (int i = 0; i < 100000; i++) {
            [largeArray addObject:@(i)];
        }
        
        // วัดเวลา - วิธี 1: for-in (Fast Enumeration) - เร็วที่สุด
        NSDate *start = [NSDate date];
        
        long sum1 = 0;
        for (NSNumber *num in largeArray) {
            sum1 += [num longValue];
        }
        
        NSTimeInterval time1 = [[NSDate date] timeIntervalSinceDate:start];
        NSLog(@"Fast Enumeration: %.4f วินาที, sum=%ld", time1, sum1);
        
        // วิธี 2: C-style for loop กับ objectAtIndex (ช้ากว่า)
        start = [NSDate date];
        
        long sum2 = 0;
        NSUInteger count = [largeArray count];
        for (NSUInteger i = 0; i < count; i++) {
            sum2 += [largeArray[i] longValue];
        }
        
        NSTimeInterval time2 = [[NSDate date] timeIntervalSinceDate:start];
        NSLog(@"Index-based: %.4f วินาที, sum=%ld", time2, sum2);
        
        // คำแนะนำ
        NSLog(@"\n=== คำแนะนำ ===");
        NSLog(@"1. ใช้ Fast Enumeration (for-in) สำหรับ Cocoa collections");
        NSLog(@"2. หลีกเลี่ยงการเรียก [array count] ใน loop condition");
        NSLog(@"3. อย่าแก้ไข collection ขณะ enumerate");
        NSLog(@"4. ใช้ enumerateObjectsUsingBlock เมื่อต้องการ index");
    }
    return 0;
}
```

### Infinite Loop และการหลีกเลี่ยง

```objc
#import <Foundation/Foundation.h>

int main(int argc, const char * argv[]) {
    @autoreleasepool {
        // ตัวอย่าง infinite loop (ระวัง!)
        // while (YES) { ... } // อย่าทำแบบนี้โดยไม่มี break condition
        
        // ✅ วิธีที่ถูกต้อง: มีเงื่อนไขหยุดเสมอ
        int attempts = 0;
        int maxAttempts = 5;
        BOOL success = NO;
        
        while (attempts < maxAttempts && !success) {
            attempts++;
            NSLog(@"ความพยายามที่ %d", attempts);
            
            // จำลองความสำเร็จที่ครั้งที่ 3
            if (attempts == 3) {
                success = YES;
            }
        }
        
        if (success) {
            NSLog(@"สำเร็จหลังจาก %d ครั้ง", attempts);
        } else {
            NSLog(@"ล้มเหลวหลังจากพยายาม %d ครั้ง", maxAttempts);
        }
    }
    return 0;
}
```

---

## 5.9 ตัวอย่างโปรแกรมที่ซับซ้อน

### โปรแกรมเรียงลำดับ Bubble Sort

```objc
#import <Foundation/Foundation.h>

void bubbleSort(int arr[], int n) {
    for (int i = 0; i < n - 1; i++) {
        BOOL swapped = NO;
        
        for (int j = 0; j < n - i - 1; j++) {
            if (arr[j] > arr[j + 1]) {
                // สลับค่า
                int temp = arr[j];
                arr[j] = arr[j + 1];
                arr[j + 1] = temp;
                swapped = YES;
            }
        }
        
        // ถ้าไม่มีการสลับ แสดงว่าเรียงแล้ว
        if (!swapped) break;
    }
}

void printArray(int arr[], int n, NSString *label) {
    NSMutableString *result = [NSMutableString stringWithFormat:@"%@: [", label];
    for (int i = 0; i < n; i++) {
        [result appendFormat:@"%d", arr[i]];
        if (i < n - 1) [result appendString:@", "];
    }
    [result appendString:@"]"];
    NSLog(@"%@", result);
}

int main(int argc, const char * argv[]) {
    @autoreleasepool {
        int data[] = {64, 34, 25, 12, 22, 11, 90};
        int n = sizeof(data) / sizeof(data[0]);
        
        printArray(data, n, @"ก่อนเรียง");
        bubbleSort(data, n);
        printArray(data, n, @"หลังเรียง");
    }
    return 0;
}
```

### โปรแกรมหาจำนวนเฉพาะ (Prime Numbers)

```objc
#import <Foundation/Foundation.h>

BOOL isPrime(int n) {
    if (n < 2) return NO;
    if (n == 2) return YES;
    if (n % 2 == 0) return NO;
    
    for (int i = 3; i * i <= n; i += 2) {
        if (n % i == 0) return NO;
    }
    
    return YES;
}

int main(int argc, const char * argv[]) {
    @autoreleasepool {
        int limit = 100;
        NSMutableArray *primes = [NSMutableArray array];
        
        for (int i = 2; i <= limit; i++) {
            if (isPrime(i)) {
                [primes addObject:@(i)];
            }
        }
        
        NSLog(@"จำนวนเฉพาะ 1-%d:", limit);
        NSLog(@"%@", primes);
        NSLog(@"มีทั้งหมด %lu จำนวน", (unsigned long)[primes count]);
        
        // หาผลรวมของจำนวนเฉพาะ
        long sum = 0;
        for (NSNumber *p in primes) {
            sum += [p longValue];
        }
        NSLog(@"ผลรวม: %ld", sum);
    }
    return 0;
}
```

### โปรแกรมวิเคราะห์ข้อความ

```objc
#import <Foundation/Foundation.h>

int main(int argc, const char * argv[]) {
    @autoreleasepool {
        NSString *text = @"The quick brown fox jumps over the lazy dog";
        NSArray *words = [text componentsSeparatedByString:@" "];
        
        // นับความถี่ของตัวอักษร
        NSMutableDictionary *charFreq = [NSMutableDictionary dictionary];
        
        for (int i = 0; i < [text length]; i++) {
            unichar c = [text characterAtIndex:i];
            if (c == ' ') continue;
            
            NSString *charStr = [NSString stringWithFormat:@"%c", 
                                 (char)tolower(c)];
            int current = [charFreq[charStr] intValue];
            charFreq[charStr] = @(current + 1);
        }
        
        NSLog(@"=== วิเคราะห์ข้อความ ===");
        NSLog(@"ข้อความ: %@", text);
        NSLog(@"จำนวนตัวอักษร: %lu", (unsigned long)[text length]);
        NSLog(@"จำนวนคำ: %lu", (unsigned long)[words count]);
        
        // หาตัวอักษรที่ปรากฏบ่อยที่สุด
        NSString *mostFrequent = nil;
        int maxFreq = 0;
        
        for (NSString *ch in charFreq) {
            int freq = [charFreq[ch] intValue];
            if (freq > maxFreq) {
                maxFreq = freq;
                mostFrequent = ch;
            }
        }
        
        NSLog(@"ตัวอักษรที่ปรากฏมากที่สุด: '%@' (%d ครั้ง)", 
              mostFrequent, maxFreq);
        
        // หาคำที่ยาวที่สุด
        NSString *longestWord = @"";
        for (NSString *word in words) {
            if ([word length] > [longestWord length]) {
                longestWord = word;
            }
        }
        NSLog(@"คำที่ยาวที่สุด: '%@' (%lu ตัวอักษร)", 
              longestWord, (unsigned long)[longestWord length]);
    }
    return 0;
}
```

---

## 5.10 แบบฝึกหัดพร้อมเฉลย

### แบบฝึกหัดที่ 1: สร้าง FizzBuzz

**โจทย์:** พิมพ์ตัวเลข 1-100 แต่ถ้าหารด้วย 3 ลงตัวพิมพ์ "Fizz" ถ้าหารด้วย 5 ลงตัวพิมพ์ "Buzz" ถ้าหารด้วยทั้ง 3 และ 5 พิมพ์ "FizzBuzz"

```objc
#import <Foundation/Foundation.h>

int main(int argc, const char * argv[]) {
    @autoreleasepool {
        for (int i = 1; i <= 100; i++) {
            if (i % 15 == 0) {
                NSLog(@"FizzBuzz");
            } else if (i % 3 == 0) {
                NSLog(@"Fizz");
            } else if (i % 5 == 0) {
                NSLog(@"Buzz");
            } else {
                NSLog(@"%d", i);
            }
        }
    }
    return 0;
}
```

---

### แบบฝึกหัดที่ 2: ตาราง Factorial

**โจทย์:** คำนวณและแสดง factorial ของ 1-10

```objc
#import <Foundation/Foundation.h>

int main(int argc, const char * argv[]) {
    @autoreleasepool {
        NSLog(@"=== ตาราง Factorial ===");
        long factorial = 1;
        
        for (int n = 1; n <= 12; n++) {
            factorial *= n;
            NSLog(@"%2d! = %ld", n, factorial);
        }
    }
    return 0;
}
```

---

### แบบฝึกหัดที่ 3: ผลรวม Arithmetic Series

**โจทย์:** คำนวณผลรวมของอนุกรมเลขคณิต a, a+d, a+2d, ... จนถึง n พจน์

```objc
#import <Foundation/Foundation.h>

int main(int argc, const char * argv[]) {
    @autoreleasepool {
        int a = 2;   // พจน์แรก
        int d = 3;   // ผลต่างร่วม
        int n = 10;  // จำนวนพจน์
        
        NSLog(@"อนุกรมเลขคณิต: a=%d, d=%d, n=%d", a, d, n);
        
        long sum = 0;
        NSMutableString *series = [NSMutableString string];
        
        for (int i = 0; i < n; i++) {
            int term = a + (i * d);
            sum += term;
            [series appendFormat:@"%d", term];
            if (i < n - 1) [series appendString:@" + "];
        }
        
        NSLog(@"%@ = %ld", series, sum);
        
        // ตรวจสอบด้วยสูตร: S = n/2 * (2a + (n-1)d)
        long formulaSum = (long)n * (2*a + (n-1)*d) / 2;
        NSLog(@"ผลรวมจากสูตร: %ld", formulaSum);
    }
    return 0;
}
```

---

### แบบฝึกหัดที่ 4: วิเคราะห์ Array

**โจทย์:** รับ array ของตัวเลขแล้วแสดงสถิติต่างๆ

```objc
#import <Foundation/Foundation.h>

void analyzeArray(NSArray *numbers) {
    if ([numbers count] == 0) {
        NSLog(@"Array ว่างเปล่า");
        return;
    }
    
    double sum = 0, min, max;
    min = max = [numbers[0] doubleValue];
    
    for (NSNumber *num in numbers) {
        double val = [num doubleValue];
        sum += val;
        if (val < min) min = val;
        if (val > max) max = val;
    }
    
    double mean = sum / [numbers count];
    
    // หา median
    NSArray *sorted = [numbers sortedArrayUsingSelector:@selector(compare:)];
    double median;
    NSUInteger count = [sorted count];
    
    if (count % 2 == 0) {
        median = ([sorted[count/2 - 1] doubleValue] + [sorted[count/2] doubleValue]) / 2.0;
    } else {
        median = [sorted[count/2] doubleValue];
    }
    
    // หา variance
    double variance = 0;
    for (NSNumber *num in numbers) {
        double diff = [num doubleValue] - mean;
        variance += diff * diff;
    }
    variance /= [numbers count];
    
    NSLog(@"จำนวนข้อมูล: %lu", (unsigned long)[numbers count]);
    NSLog(@"ผลรวม: %.2f", sum);
    NSLog(@"ค่าเฉลี่ย: %.2f", mean);
    NSLog(@"Median: %.2f", median);
    NSLog(@"ค่าต่ำสุด: %.2f", min);
    NSLog(@"ค่าสูงสุด: %.2f", max);
    NSLog(@"Variance: %.2f", variance);
    NSLog(@"Std Deviation: %.2f", sqrt(variance));
}

int main(int argc, const char * argv[]) {
    @autoreleasepool {
        NSArray *data = @[@4, @7, @13, @2, @9, @11, @5, @8, @3, @15];
        NSLog(@"=== สถิติของข้อมูล ===");
        analyzeArray(data);
    }
    return 0;
}
```

---

### แบบฝึกหัดที่ 5: สร้าง Pascal's Triangle

**โจทย์:** สร้างและแสดง Pascal's Triangle จำนวน n แถว

```objc
#import <Foundation/Foundation.h>

int main(int argc, const char * argv[]) {
    @autoreleasepool {
        int rows = 6;
        int triangle[10][10] = {0};
        
        // คำนวณ Pascal's Triangle
        for (int i = 0; i < rows; i++) {
            triangle[i][0] = 1;
            for (int j = 1; j <= i; j++) {
                triangle[i][j] = triangle[i-1][j-1] + triangle[i-1][j];
            }
        }
        
        // แสดงผล
        NSLog(@"=== Pascal's Triangle (%d แถว) ===", rows);
        for (int i = 0; i < rows; i++) {
            NSMutableString *row = [NSMutableString string];
            
            // Padding
            for (int p = 0; p < (rows - i - 1); p++) {
                [row appendString:@"  "];
            }
            
            for (int j = 0; j <= i; j++) {
                [row appendFormat:@"%4d", triangle[i][j]];
            }
            NSLog(@"%@", row);
        }
    }
    return 0;
}
```

---

### แบบฝึกหัดที่ 6: นับคำในประโยค

**โจทย์:** นับจำนวนครั้งที่แต่ละคำปรากฏในประโยค

```objc
#import <Foundation/Foundation.h>

int main(int argc, const char * argv[]) {
    @autoreleasepool {
        NSString *text = @"the cat sat on the mat the cat is fat";
        NSArray *words = [text componentsSeparatedByString:@" "];
        NSMutableDictionary *wordCount = [NSMutableDictionary dictionary];
        
        // นับความถี่ของแต่ละคำ
        for (NSString *word in words) {
            NSString *lowercased = [word lowercaseString];
            int count = [wordCount[lowercased] intValue];
            wordCount[lowercased] = @(count + 1);
        }
        
        // เรียงลำดับตามความถี่
        NSArray *sortedWords = [[wordCount allKeys] sortedArrayUsingComparator:
            ^NSComparisonResult(NSString *a, NSString *b) {
                return [wordCount[b] compare:wordCount[a]];
            }];
        
        NSLog(@"=== ความถี่ของคำ ===");
        for (NSString *word in sortedWords) {
            NSLog(@"'%@': %@ ครั้ง", word, wordCount[word]);
        }
    }
    return 0;
}
```

---

### แบบฝึกหัดที่ 7: เกมเดาตัวเลข

**โจทย์:** จำลองเกมเดาตัวเลข 1-100

```objc
#import <Foundation/Foundation.h>

int main(int argc, const char * argv[]) {
    @autoreleasepool {
        int secretNumber = 42; // ในเกมจริงจะใช้ random
        int maxGuesses = 7;
        int guesses[] = {50, 25, 37, 43, 40, 41, 42}; // จำลอง guesses
        int guessCount = 0;
        BOOL won = NO;
        
        NSLog(@"=== เกมเดาตัวเลข (1-100) ===");
        NSLog(@"คุณมี %d โอกาสในการเดา", maxGuesses);
        
        for (int i = 0; i < maxGuesses; i++) {
            guessCount++;
            int guess = guesses[i];
            NSLog(@"\nครั้งที่ %d เดา: %d", guessCount, guess);
            
            if (guess == secretNumber) {
                NSLog(@"ถูกต้อง! คุณชนะใน %d ครั้ง!", guessCount);
                won = YES;
                break;
            } else if (guess < secretNumber) {
                NSLog(@"น้อยเกินไป! ลองค่าที่มากกว่า");
            } else {
                NSLog(@"มากเกินไป! ลองค่าที่น้อยกว่า");
            }
        }
        
        if (!won) {
            NSLog(@"\nหมดโอกาสแล้ว! ตัวเลขที่ถูกคือ %d", secretNumber);
        }
    }
    return 0;
}
```

---

### แบบฝึกหัดที่ 8: Matrix Multiplication

**โจทย์:** คูณเมทริกซ์ 2 ตัว

```objc
#import <Foundation/Foundation.h>

void multiplyMatrices(int a[2][3], int b[3][2], int result[2][2]) {
    for (int i = 0; i < 2; i++) {
        for (int j = 0; j < 2; j++) {
            result[i][j] = 0;
            for (int k = 0; k < 3; k++) {
                result[i][j] += a[i][k] * b[k][j];
            }
        }
    }
}

void printMatrix(int m[][2], int rows, int cols) {
    for (int i = 0; i < rows; i++) {
        NSMutableString *row = [NSMutableString string];
        for (int j = 0; j < cols; j++) {
            [row appendFormat:@"%5d", m[i][j]];
        }
        NSLog(@"%@", row);
    }
}

int main(int argc, const char * argv[]) {
    @autoreleasepool {
        int A[2][3] = {{1, 2, 3}, {4, 5, 6}};
        int B[3][2] = {{7, 8}, {9, 10}, {11, 12}};
        int C[2][2];
        
        multiplyMatrices(A, B, C);
        
        NSLog(@"=== Matrix A (2x3) ===");
        for (int i = 0; i < 2; i++) {
            NSLog(@"%4d %4d %4d", A[i][0], A[i][1], A[i][2]);
        }
        
        NSLog(@"\n=== Matrix B (3x2) ===");
        for (int i = 0; i < 3; i++) {
            NSLog(@"%4d %4d", B[i][0], B[i][1]);
        }
        
        NSLog(@"\n=== A x B (2x2) ===");
        printMatrix(C, 2, 2);
    }
    return 0;
}
```

---

### แบบฝึกหัดที่ 9: คำนวณดอกเบี้ยทบต้น

**โจทย์:** คำนวณการเติบโตของเงินฝากด้วยดอกเบี้ยทบต้น

```objc
#import <Foundation/Foundation.h>

int main(int argc, const char * argv[]) {
    @autoreleasepool {
        double principal = 10000;  // เงินต้น
        double annualRate = 0.05;  // อัตราดอกเบี้ยต่อปี 5%
        int years = 10;
        
        NSLog(@"=== การเติบโตของเงินฝาก ===");
        NSLog(@"เงินต้น: %.2f บาท", principal);
        NSLog(@"อัตราดอกเบี้ย: %.0f%%/ปี", annualRate * 100);
        NSLog(@"");
        NSLog(@"ปี  | เงินต้น       | ดอกเบี้ย     | ยอดรวม");
        NSLog(@"----|--------------|-------------|-------------");
        
        double balance = principal;
        
        for (int year = 1; year <= years; year++) {
            double interest = balance * annualRate;
            balance += interest;
            
            NSLog(@"%3d | %12.2f | %11.2f | %11.2f",
                  year, principal, interest, balance);
        }
        
        NSLog(@"\nยอดรวมสุดท้าย: %.2f บาท (เพิ่มขึ้น %.2f%%)",
              balance,
              ((balance - principal) / principal) * 100);
    }
    return 0;
}
```

---

### แบบฝึกหัดที่ 10: ระบบ Inventory จำลอง

**โจทย์:** จัดการสินค้าคงคลัง

```objc
#import <Foundation/Foundation.h>

int main(int argc, const char * argv[]) {
    @autoreleasepool {
        NSMutableArray *inventory = [NSMutableArray arrayWithArray:@[
            @{@"id": @"P001", @"name": @"แอปเปิ้ล",  @"qty": @50, @"price": @25},
            @{@"id": @"P002", @"name": @"กล้วย",     @"qty": @80, @"price": @15},
            @{@"id": @"P003", @"name": @"ส้ม",        @"qty": @30, @"price": @35},
            @{@"id": @"P004", @"name": @"มะม่วง",    @"qty": @20, @"price": @45},
            @{@"id": @"P005", @"name": @"สตรอเบอร์รี่", @"qty": @10, @"price": @90}
        ]];
        
        // แสดงรายการสินค้า
        NSLog(@"=== รายการสินค้าคงคลัง ===");
        NSLog(@"%-6s %-15s %8s %10s %12s",
              "ID", "ชื่อสินค้า", "จำนวน", "ราคา/ชิ้น", "มูลค่ารวม");
        NSLog(@"------------------------------------------------------");
        
        double totalValue = 0;
        NSMutableArray *lowStock = [NSMutableArray array];
        
        for (NSDictionary *item in inventory) {
            int qty = [item[@"qty"] intValue];
            double price = [item[@"price"] doubleValue];
            double value = qty * price;
            totalValue += value;
            
            NSLog(@"%-6@ %-15@ %8d %10.2f %12.2f",
                  item[@"id"], item[@"name"], qty, price, value);
            
            if (qty < 25) {
                [lowStock addObject:item[@"name"]];
            }
        }
        
        NSLog(@"------------------------------------------------------");
        NSLog(@"มูลค่าสินค้าทั้งหมด: %.2f บาท", totalValue);
        
        if ([lowStock count] > 0) {
            NSLog(@"\n⚠️ สินค้าที่ต้องสั่งเพิ่ม: %@",
                  [lowStock componentsJoinedByString:@", "]);
        }
        
        // หาสินค้าที่ขายดี (ราคาสูง x จำนวนมาก)
        NSDictionary *bestseller = inventory[0];
        double maxValue = [inventory[0][@"qty"] doubleValue] * [inventory[0][@"price"] doubleValue];
        
        for (NSDictionary *item in inventory) {
            double val = [item[@"qty"] doubleValue] * [item[@"price"] doubleValue];
            if (val > maxValue) {
                maxValue = val;
                bestseller = item;
            }
        }
        
        NSLog(@"\nสินค้ามูลค่าสูงสุด: %@ (%.2f บาท)", bestseller[@"name"], maxValue);
    }
    return 0;
}
```

---

## สรุปบทที่ 5

ในบทนี้เราได้เรียนรู้:

1. **for loop** - วนซ้ำตามจำนวนครั้งที่กำหนด
2. **while loop** - วนซ้ำตามเงื่อนไข
3. **do-while loop** - ทำงานอย่างน้อย 1 ครั้งก่อนตรวจเงื่อนไข
4. **for-in loop** - Fast Enumeration สำหรับ Cocoa collections
5. **break** - ออกจาก loop ทันที
6. **continue** - ข้ามการทำซ้ำปัจจุบัน
7. **Nested loops** - loop ซ้อนกัน สำหรับปัญหา 2 มิติ

### เคล็ดลับสำคัญ

- ใช้ **for-in** (Fast Enumeration) กับ Objective-C collections เสมอ
- เก็บ `[array count]` ในตัวแปรก่อนวน loop เพื่อประสิทธิภาพ
- ระวัง **infinite loop** - มี break condition เสมอ
- ใช้ **break** เพื่อออกจาก loop เมื่อพบสิ่งที่ต้องการแล้ว
- ใช้ **continue** เพื่อข้ามรายการที่ไม่ต้องการ

---

*บทต่อไป: ส่วนที่ 06 - ฟังก์ชัน (Functions)*
