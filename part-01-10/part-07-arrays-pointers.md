# ตอนที่ 7: Arrays และ Pointers ใน Objective-C

## บทนำ

Arrays (อาร์เรย์) และ Pointers (พอยน์เตอร์) เป็นพื้นฐานสำคัญในการเขียนโปรแกรมภาษา C และ Objective-C การเข้าใจแนวคิดเหล่านี้จะช่วยให้คุณเขียนโปรแกรมได้อย่างมีประสิทธิภาพมากขึ้น ในบทนี้เราจะเรียนรู้ทั้ง C arrays แบบดั้งเดิม และ NSArray/NSMutableArray ของ Objective-C

---

## 7.1 C Arrays - อาร์เรย์แบบ C

### การประกาศและใช้งาน C Array

C Array คือกลุ่มของตัวแปรที่มีชนิดข้อมูลเดียวกัน เรียงกันอยู่ในหน่วยความจำแบบต่อเนื่อง

```objc
#import <Foundation/Foundation.h>

int main(int argc, const char * argv[]) {
    @autoreleasepool {
        
        // การประกาศ array แบบต่างๆ
        int numbers[5];                          // ประกาศ array ขนาด 5
        int scores[5] = {90, 85, 78, 92, 88};   // ประกาศและกำหนดค่าพร้อมกัน
        int temps[] = {30, 31, 28, 33, 29};      // คอมไพเลอร์กำหนดขนาดให้อัตโนมัติ
        
        // การเข้าถึงสมาชิก
        NSLog(@"คะแนนที่ 1: %d", scores[0]);   // index เริ่มจาก 0
        NSLog(@"คะแนนที่ 3: %d", scores[2]);
        
        // การวนลูปผ่าน array
        NSLog(@"\n=== คะแนนทั้งหมด ===");
        for (int i = 0; i < 5; i++) {
            NSLog(@"คะแนน[%d] = %d", i, scores[i]);
        }
        
        // การคำนวณค่าเฉลี่ย
        int sum = 0;
        for (int i = 0; i < 5; i++) {
            sum += scores[i];
        }
        double average = (double)sum / 5;
        NSLog(@"\nค่าเฉลี่ยคะแนน: %.2f", average);
        
        // การหาค่าสูงสุด
        int maxScore = scores[0];
        for (int i = 1; i < 5; i++) {
            if (scores[i] > maxScore) {
                maxScore = scores[i];
            }
        }
        NSLog(@"คะแนนสูงสุด: %d", maxScore);
    }
    return 0;
}
```

### การแก้ไขค่าใน Array

```objc
#import <Foundation/Foundation.h>

int main(int argc, const char * argv[]) {
    @autoreleasepool {
        
        int data[10];
        
        // กำหนดค่าด้วยลูป
        for (int i = 0; i < 10; i++) {
            data[i] = (i + 1) * 10;  // 10, 20, 30, ..., 100
        }
        
        // แสดงข้อมูล
        NSLog(@"ข้อมูลใน array:");
        for (int i = 0; i < 10; i++) {
            NSLog(@"data[%d] = %d", i, data[i]);
        }
        
        // แก้ไขค่า
        data[4] = 999;
        NSLog(@"\nหลังแก้ไข data[4] = %d", data[4]);
        
        // Bubble Sort ง่ายๆ
        int arr[] = {64, 34, 25, 12, 22, 11, 90};
        int n = 7;
        
        NSLog(@"\n=== Bubble Sort ===");
        NSLog(@"ก่อนเรียง:");
        for (int i = 0; i < n; i++) {
            NSLog(@"arr[%d] = %d", i, arr[i]);
        }
        
        for (int i = 0; i < n - 1; i++) {
            for (int j = 0; j < n - i - 1; j++) {
                if (arr[j] > arr[j + 1]) {
                    int temp = arr[j];
                    arr[j] = arr[j + 1];
                    arr[j + 1] = temp;
                }
            }
        }
        
        NSLog(@"\nหลังเรียง:");
        for (int i = 0; i < n; i++) {
            NSLog(@"arr[%d] = %d", i, arr[i]);
        }
    }
    return 0;
}
```

### ขนาดของ Array

```objc
#import <Foundation/Foundation.h>

int main(int argc, const char * argv[]) {
    @autoreleasepool {
        
        int numbers[] = {1, 2, 3, 4, 5, 6, 7, 8, 9, 10};
        
        // หาขนาดของ array ด้วย sizeof
        int arraySize = sizeof(numbers) / sizeof(numbers[0]);
        NSLog(@"ขนาดของ array: %d", arraySize);
        NSLog(@"ขนาดทั้งหมด (bytes): %lu", sizeof(numbers));
        NSLog(@"ขนาดต่อ element (bytes): %lu", sizeof(numbers[0]));
        
        // วนลูปโดยใช้ขนาดที่คำนวณได้
        int sum = 0;
        for (int i = 0; i < arraySize; i++) {
            sum += numbers[i];
        }
        NSLog(@"ผลรวมทั้งหมด: %d", sum);
        
        // Array ของ float
        float prices[] = {29.99f, 49.99f, 15.50f, 99.00f};
        int priceCount = sizeof(prices) / sizeof(prices[0]);
        
        float totalPrice = 0;
        for (int i = 0; i < priceCount; i++) {
            totalPrice += prices[i];
        }
        NSLog(@"\nราคารวม: %.2f บาท", totalPrice);
        
        // Array ของ char (string แบบ C)
        char name[] = "Hello";
        NSLog(@"\nข้อความ: %s", name);
        NSLog(@"ความยาว: %lu", strlen(name));
    }
    return 0;
}
```

---

## 7.2 Multi-dimensional Arrays - อาร์เรย์หลายมิติ

### 2D Array (Matrix)

```objc
#import <Foundation/Foundation.h>

int main(int argc, const char * argv[]) {
    @autoreleasepool {
        
        // การประกาศ 2D array (3 แถว x 4 คอลัมน์)
        int matrix[3][4] = {
            {1,  2,  3,  4},
            {5,  6,  7,  8},
            {9, 10, 11, 12}
        };
        
        // แสดงเมทริกซ์
        NSLog(@"=== เมทริกซ์ 3x4 ===");
        for (int row = 0; row < 3; row++) {
            NSMutableString *line = [NSMutableString string];
            for (int col = 0; col < 4; col++) {
                [line appendFormat:@"%4d", matrix[row][col]];
            }
            NSLog(@"%@", line);
        }
        
        // การเข้าถึงสมาชิก
        NSLog(@"\nmatrix[1][2] = %d", matrix[1][2]);  // แถว 1, คอลัมน์ 2
        
        // การคำนวณผลรวมของแต่ละแถว
        NSLog(@"\n=== ผลรวมแต่ละแถว ===");
        for (int row = 0; row < 3; row++) {
            int rowSum = 0;
            for (int col = 0; col < 4; col++) {
                rowSum += matrix[row][col];
            }
            NSLog(@"แถว %d: %d", row + 1, rowSum);
        }
        
        // การบวกเมทริกซ์
        int a[2][2] = {{1, 2}, {3, 4}};
        int b[2][2] = {{5, 6}, {7, 8}};
        int result[2][2];
        
        for (int i = 0; i < 2; i++) {
            for (int j = 0; j < 2; j++) {
                result[i][j] = a[i][j] + b[i][j];
            }
        }
        
        NSLog(@"\n=== ผลรวมเมทริกซ์ ===");
        for (int i = 0; i < 2; i++) {
            NSLog(@"[%d, %d]", result[i][0], result[i][1]);
        }
    }
    return 0;
}
```

### 3D Array

```objc
#import <Foundation/Foundation.h>

int main(int argc, const char * argv[]) {
    @autoreleasepool {
        
        // 3D array (2 ชั้น x 3 แถว x 3 คอลัมน์)
        int cube[2][3][3] = {
            {  // ชั้นที่ 0
                {1, 2, 3},
                {4, 5, 6},
                {7, 8, 9}
            },
            {  // ชั้นที่ 1
                {10, 11, 12},
                {13, 14, 15},
                {16, 17, 18}
            }
        };
        
        NSLog(@"=== 3D Array ===");
        for (int layer = 0; layer < 2; layer++) {
            NSLog(@"\nชั้นที่ %d:", layer);
            for (int row = 0; row < 3; row++) {
                NSMutableString *line = [NSMutableString string];
                for (int col = 0; col < 3; col++) {
                    [line appendFormat:@"%4d", cube[layer][row][col]];
                }
                NSLog(@"%@", line);
            }
        }
        
        // การเข้าถึงชั้น แถว คอลัมน์
        NSLog(@"\ncube[1][2][1] = %d", cube[1][2][1]);  // ชั้น 1, แถว 2, คอลัมน์ 1
    }
    return 0;
}
```

### ตารางคูณด้วย 2D Array

```objc
#import <Foundation/Foundation.h>

int main(int argc, const char * argv[]) {
    @autoreleasepool {
        
        // สร้างตารางคูณ
        int table[10][10];
        
        for (int i = 1; i <= 10; i++) {
            for (int j = 1; j <= 10; j++) {
                table[i-1][j-1] = i * j;
            }
        }
        
        // แสดงตารางคูณ 1-5
        NSLog(@"=== ตารางคูณ (1-5) ===");
        NSLog(@"     1    2    3    4    5");
        NSLog(@"   -------------------------");
        for (int i = 0; i < 5; i++) {
            NSMutableString *row = [NSMutableString stringWithFormat:@"%d |", i + 1];
            for (int j = 0; j < 5; j++) {
                [row appendFormat:@"%4d ", table[i][j]];
            }
            NSLog(@"%@", row);
        }
    }
    return 0;
}
```

---

## 7.3 Pointers - พอยน์เตอร์

### พอยน์เตอร์คืออะไร?

พอยน์เตอร์คือตัวแปรที่เก็บ **ที่อยู่ในหน่วยความจำ** (memory address) แทนที่จะเก็บค่าโดยตรง

```
ตัวแปรปกติ:     int x = 42;
                  [42] ← ค่าที่เก็บ
                  address: 0x1000

พอยน์เตอร์:      int *ptr = &x;
                  [0x1000] ← เก็บ address ของ x
                  address: 0x1004
```

### การใช้งาน * และ &

- `&variable` = address-of operator (หา address ของตัวแปร)
- `*pointer` = dereference operator (เข้าถึงค่าที่ address นั้น)

```objc
#import <Foundation/Foundation.h>

int main(int argc, const char * argv[]) {
    @autoreleasepool {
        
        int x = 42;
        int *ptr;      // ประกาศพอยน์เตอร์ชี้ไปที่ int
        
        ptr = &x;      // ptr เก็บ address ของ x
        
        NSLog(@"ค่าของ x: %d", x);
        NSLog(@"Address ของ x: %p", &x);
        NSLog(@"ค่าใน ptr (address): %p", ptr);
        NSLog(@"ค่าที่ ptr ชี้ถึง: %d", *ptr);  // dereference
        
        // แก้ไขค่าผ่านพอยน์เตอร์
        *ptr = 100;
        NSLog(@"\nหลังแก้ไขผ่านพอยน์เตอร์:");
        NSLog(@"ค่าของ x: %d", x);      // x ก็เปลี่ยนด้วย!
        NSLog(@"ค่าที่ ptr ชี้ถึง: %d", *ptr);
        
        // พอยน์เตอร์ชนิดต่างๆ
        double pi = 3.14159;
        double *dPtr = &pi;
        NSLog(@"\npi = %.5f", *dPtr);
        
        char letter = 'A';
        char *cPtr = &letter;
        NSLog(@"ตัวอักษร: %c", *cPtr);
        
        // NULL pointer
        int *nullPtr = NULL;  // หรือ nil ใน Objective-C
        NSLog(@"\nNULL pointer: %p", nullPtr);
        
        // ตรวจสอบก่อนใช้เสมอ!
        if (nullPtr != NULL) {
            NSLog(@"ค่า: %d", *nullPtr);
        } else {
            NSLog(@"ptr เป็น NULL ไม่สามารถ dereference ได้");
        }
    }
    return 0;
}
```

### การส่งพอยน์เตอร์ให้ฟังก์ชัน

```objc
#import <Foundation/Foundation.h>

// ฟังก์ชันที่รับพอยน์เตอร์ - สามารถแก้ไขค่าต้นฉบับได้
void doubleValue(int *n) {
    *n = *n * 2;
}

// ฟังก์ชัน swap
void swap(int *a, int *b) {
    int temp = *a;
    *a = *b;
    *b = temp;
}

// ฟังก์ชันคืนค่าหลายค่าผ่านพอยน์เตอร์
void minMax(int arr[], int size, int *min, int *max) {
    *min = arr[0];
    *max = arr[0];
    
    for (int i = 1; i < size; i++) {
        if (arr[i] < *min) *min = arr[i];
        if (arr[i] > *max) *max = arr[i];
    }
}

int main(int argc, const char * argv[]) {
    @autoreleasepool {
        
        int value = 10;
        NSLog(@"ก่อน doubleValue: %d", value);
        doubleValue(&value);
        NSLog(@"หลัง doubleValue: %d", value);
        
        int a = 5, b = 10;
        NSLog(@"\nก่อน swap: a=%d, b=%d", a, b);
        swap(&a, &b);
        NSLog(@"หลัง swap: a=%d, b=%d", a, b);
        
        int numbers[] = {3, 7, 1, 9, 4, 6, 2, 8, 5};
        int n = 9;
        int minimum, maximum;
        minMax(numbers, n, &minimum, &maximum);
        NSLog(@"\nค่าต่ำสุด: %d, ค่าสูงสุด: %d", minimum, maximum);
    }
    return 0;
}
```

---

## 7.4 Pointer Arithmetic - การคำนวณกับพอยน์เตอร์

```objc
#import <Foundation/Foundation.h>

int main(int argc, const char * argv[]) {
    @autoreleasepool {
        
        int arr[] = {10, 20, 30, 40, 50};
        int *ptr = arr;   // ptr ชี้ไปที่ arr[0]
        
        NSLog(@"=== Pointer Arithmetic ===");
        NSLog(@"ptr = %p, *ptr = %d", ptr, *ptr);
        
        ptr++;            // เลื่อน pointer ไปข้างหน้า 1 ตำแหน่ง (4 bytes สำหรับ int)
        NSLog(@"ptr++ → ptr = %p, *ptr = %d", ptr, *ptr);
        
        ptr += 2;         // เลื่อน pointer ไปข้างหน้า 2 ตำแหน่ง
        NSLog(@"ptr += 2 → ptr = %p, *ptr = %d", ptr, *ptr);
        
        ptr--;            // เลื่อนกลับ 1 ตำแหน่ง
        NSLog(@"ptr-- → ptr = %p, *ptr = %d", ptr, *ptr);
        
        // การลบพอยน์เตอร์
        int *start = arr;
        int *end = &arr[4];
        ptrdiff_t distance = end - start;
        NSLog(@"\nระยะห่างระหว่าง start และ end: %td", distance);
        
        // วนลูปผ่าน array ด้วยพอยน์เตอร์
        NSLog(@"\n=== วนลูปด้วย pointer ===");
        int *p = arr;
        int *arrEnd = arr + 5;  // หลัง element สุดท้าย
        
        while (p < arrEnd) {
            NSLog(@"ค่า: %d (address: %p)", *p, p);
            p++;
        }
        
        // Pointer กับ char array (string)
        char str[] = "Hello!";
        char *cp = str;
        
        NSLog(@"\n=== Pointer กับ String ===");
        while (*cp != '\0') {
            NSLog(@"'%c' (ASCII: %d)", *cp, *cp);
            cp++;
        }
        
        // ขนาดตามชนิดข้อมูล
        char c;
        short s;
        int i;
        long l;
        double d;
        
        char *cp2 = &c;
        short *sp = &s;
        int *ip = &i;
        long *lp = &l;
        double *dp = &d;
        
        NSLog(@"\n=== ขนาด pointer increment ===");
        NSLog(@"char*: %lu byte(s)", sizeof(*cp2));
        NSLog(@"short*: %lu byte(s)", sizeof(*sp));
        NSLog(@"int*: %lu byte(s)", sizeof(*ip));
        NSLog(@"long*: %lu byte(s)", sizeof(*lp));
        NSLog(@"double*: %lu byte(s)", sizeof(*dp));
    }
    return 0;
}
```

---

## 7.5 ความสัมพันธ์ระหว่าง Arrays และ Pointers

```objc
#import <Foundation/Foundation.h>

int main(int argc, const char * argv[]) {
    @autoreleasepool {
        
        int arr[] = {10, 20, 30, 40, 50};
        
        NSLog(@"=== ความสัมพันธ์ Array กับ Pointer ===");
        NSLog(@"arr = %p", arr);
        NSLog(@"&arr[0] = %p", &arr[0]);
        NSLog(@"ค่าเดียวกัน: %s", (arr == &arr[0]) ? "YES" : "NO");
        
        // การเข้าถึงด้วยวิธีต่างๆ
        for (int i = 0; i < 5; i++) {
            NSLog(@"arr[%d] = *(arr+%d) = %d = %d",
                  i, i, arr[i], *(arr + i));
        }
        
        // ฟังก์ชันที่รับ array
        // int sum(int arr[], int n) เหมือน int sum(int *arr, int n)
    }
    return 0;
}

// ฟังก์ชันรับ array - ทั้งสองแบบเหมือนกัน
int sumArray(int arr[], int n) {
    int total = 0;
    for (int i = 0; i < n; i++) {
        total += arr[i];
    }
    return total;
}

int sumPointer(int *ptr, int n) {
    int total = 0;
    for (int i = 0; i < n; i++) {
        total += *(ptr + i);  // หรือ ptr[i] ก็ได้
    }
    return total;
}
```

### -> Operator

```objc
#import <Foundation/Foundation.h>

// struct สำหรับตัวอย่าง -> operator
typedef struct {
    char name[50];
    int age;
    float score;
} Student;

int main(int argc, const char * argv[]) {
    @autoreleasepool {
        
        Student student1 = {"สมชาย ใจดี", 20, 85.5f};
        Student *ptr = &student1;
        
        NSLog(@"=== การใช้ -> Operator ===");
        
        // สองวิธีนี้เหมือนกัน
        NSLog(@"ชื่อ (dot): %s", student1.name);
        NSLog(@"ชื่อ (arrow): %s", ptr->name);  // ptr->name คือ (*ptr).name
        
        NSLog(@"อายุ: %d", ptr->age);
        NSLog(@"คะแนน: %.1f", ptr->score);
        
        // แก้ไขผ่านพอยน์เตอร์
        ptr->age = 21;
        ptr->score = 90.0f;
        
        NSLog(@"\nหลังแก้ไข:");
        NSLog(@"อายุ: %d", student1.age);  // เปลี่ยนด้วย
        NSLog(@"คะแนน: %.1f", student1.score);
        
        // Array ของ struct
        Student class[3] = {
            {"นักเรียน A", 18, 75.0f},
            {"นักเรียน B", 19, 88.5f},
            {"นักเรียน C", 20, 92.0f}
        };
        
        NSLog(@"\n=== รายชื่อนักเรียน ===");
        for (int i = 0; i < 3; i++) {
            Student *s = &class[i];
            NSLog(@"%d. %s (อายุ: %d, คะแนน: %.1f)",
                  i + 1, s->name, s->age, s->score);
        }
    }
    return 0;
}
```

---

## 7.6 NSArray - อาร์เรย์แบบ Objective-C (Immutable)

NSArray คืออาร์เรย์ที่ไม่สามารถเปลี่ยนแปลงได้หลังจากสร้างแล้ว (immutable) และสามารถเก็บ **Objective-C objects** เท่านั้น

### การสร้าง NSArray

```objc
#import <Foundation/Foundation.h>

int main(int argc, const char * argv[]) {
    @autoreleasepool {
        
        // วิธีที่ 1: ใช้ literal syntax (แนะนำ)
        NSArray *fruits = @[@"Apple", @"Banana", @"Cherry", @"Date"];
        
        // วิธีที่ 2: arrayWithObjects:
        NSArray *numbers = [NSArray arrayWithObjects:
                            @1, @2, @3, @4, @5, nil];
        
        // วิธีที่ 3: arrayWithArray: (คัดลอกจาก array อื่น)
        NSArray *copy = [NSArray arrayWithArray:fruits];
        
        // วิธีที่ 4: array เปล่า
        NSArray *empty = @[];
        
        // การเข้าถึงสมาชิก
        NSLog(@"=== การเข้าถึง NSArray ===");
        NSLog(@"ผลไม้ที่ 1: %@", fruits[0]);
        NSLog(@"ผลไม้ที่ 2: %@", [fruits objectAtIndex:1]);
        NSLog(@"ผลไม้สุดท้าย: %@", [fruits lastObject]);
        NSLog(@"ผลไม้แรก: %@", [fruits firstObject]);
        
        // ขนาดของ array
        NSLog(@"\nจำนวนผลไม้: %lu", [fruits count]);
        NSLog(@"จำนวน (ว่าง): %lu", [empty count]);
        
        // ตรวจสอบว่ามีอยู่ใน array หรือไม่
        NSLog(@"\nมี 'Apple' ใน fruits: %@",
              [fruits containsObject:@"Apple"] ? @"YES" : @"NO");
        NSLog(@"มี 'Mango' ใน fruits: %@",
              [fruits containsObject:@"Mango"] ? @"YES" : @"NO");
        
        // หา index ของ object
        NSUInteger index = [fruits indexOfObject:@"Cherry"];
        NSLog(@"\nIndex ของ 'Cherry': %lu", index);
        
        // NSLog ทั้ง array
        NSLog(@"\nทั้ง array: %@", fruits);
        NSLog(@"Numbers: %@", numbers);
    }
    return 0;
}
```

### การวนลูปผ่าน NSArray

```objc
#import <Foundation/Foundation.h>

int main(int argc, const char * argv[]) {
    @autoreleasepool {
        
        NSArray *cities = @[@"กรุงเทพ", @"เชียงใหม่", @"ภูเก็ต",
                            @"พัทยา", @"อยุธยา"];
        
        // วิธีที่ 1: for loop แบบดั้งเดิม
        NSLog(@"=== for loop แบบดั้งเดิม ===");
        for (NSUInteger i = 0; i < [cities count]; i++) {
            NSLog(@"%lu. %@", i + 1, cities[i]);
        }
        
        // วิธีที่ 2: Fast enumeration (for-in) - แนะนำ
        NSLog(@"\n=== Fast Enumeration ===");
        for (NSString *city in cities) {
            NSLog(@"เมือง: %@", city);
        }
        
        // วิธีที่ 3: enumerateObjectsUsingBlock:
        NSLog(@"\n=== Block Enumeration ===");
        [cities enumerateObjectsUsingBlock:^(NSString *city, NSUInteger idx, BOOL *stop) {
            NSLog(@"[%lu] %@", idx, city);
            if ([city isEqualToString:@"ภูเก็ต"]) {
                NSLog(@"   ^ พบเมืองที่ต้องการ! หยุดค้นหา");
                *stop = YES;  // หยุดการวนลูป
            }
        }];
        
        // วิธีที่ 4: makeObjectsPerformSelector:
        NSLog(@"\n=== ผลไม้ทั้งหมด (uppercase) ===");
        NSArray *fruits = @[@"apple", @"banana", @"cherry"];
        for (NSString *fruit in fruits) {
            NSLog(@"%@", [fruit uppercaseString]);
        }
        
        // Reverse enumeration
        NSLog(@"\n=== วนย้อนกลับ ===");
        NSEnumerator *reverseEnum = [cities reverseObjectEnumerator];
        NSString *city;
        while ((city = [reverseEnum nextObject]) != nil) {
            NSLog(@"- %@", city);
        }
    }
    return 0;
}
```

### Subarrays และ Array Operations

```objc
#import <Foundation/Foundation.h>

int main(int argc, const char * argv[]) {
    @autoreleasepool {
        
        NSArray *numbers = @[@1, @5, @3, @8, @2, @9, @4, @7, @6];
        
        // Subarray
        NSArray *sub = [numbers subarrayWithRange:NSMakeRange(2, 4)];
        NSLog(@"Subarray (2,4): %@", sub);
        
        // รวม arrays
        NSArray *more = @[@10, @11, @12];
        NSArray *combined = [numbers arrayByAddingObjectsFromArray:more];
        NSLog(@"\nรวม array: %@", combined);
        
        // เพิ่ม object เดียว
        NSArray *withExtra = [numbers arrayByAddingObject:@99];
        NSLog(@"\nเพิ่ม 99: %@", withExtra);
        
        // แปลงเป็น string
        NSString *joined = [numbers componentsJoinedByString:@", "];
        NSLog(@"\nจอยน์ด้วย ', ': %@", joined);
        
        // สร้างจาก string แยก
        NSString *csv = @"แดง,เขียว,น้ำเงิน,เหลือง";
        NSArray *colors = [csv componentsSeparatedByString:@","];
        NSLog(@"\nสีจาก CSV: %@", colors);
        
        // ตรวจสอบ array
        NSLog(@"\narray ว่างเปล่า: %@", [numbers count] == 0 ? @"YES" : @"NO");
        NSLog(@"array มีสมาชิก: %lu", [numbers count]);
        
        // หา object ด้วย predicate
        NSArray *bigNumbers = [numbers filteredArrayUsingPredicate:
                               [NSPredicate predicateWithBlock:^BOOL(NSNumber *n, NSDictionary *bindings) {
            return [n intValue] > 5;
        }]];
        NSLog(@"\nตัวเลขที่มากกว่า 5: %@", bigNumbers);
    }
    return 0;
}
```

---

## 7.7 NSMutableArray - อาร์เรย์ที่แก้ไขได้

```objc
#import <Foundation/Foundation.h>

int main(int argc, const char * argv[]) {
    @autoreleasepool {
        
        // สร้าง NSMutableArray
        NSMutableArray *list = [NSMutableArray array];
        NSMutableArray *numbers = [NSMutableArray arrayWithObjects:
                                   @1, @2, @3, @4, @5, nil];
        NSMutableArray *fromLiteral = [@[@"a", @"b", @"c"] mutableCopy];
        
        NSLog(@"=== การเพิ่มสมาชิก ===");
        // เพิ่มท้าย
        [list addObject:@"สมชาย"];
        [list addObject:@"สมหญิง"];
        [list addObject:@"มนัส"];
        NSLog(@"หลังเพิ่ม: %@", list);
        
        // เพิ่มตำแหน่งที่กำหนด
        [list insertObject:@"แมวน้อย" atIndex:1];
        NSLog(@"หลัง insert ที่ index 1: %@", list);
        
        // เพิ่มหลาย objects พร้อมกัน
        [numbers addObjectsFromArray:@[@6, @7, @8]];
        NSLog(@"\nNumbers: %@", numbers);
        
        NSLog(@"\n=== การลบสมาชิก ===");
        // ลบ object ที่รู้จัก
        [list removeObject:@"มนัส"];
        NSLog(@"หลังลบ 'มนัส': %@", list);
        
        // ลบตาม index
        [list removeObjectAtIndex:0];
        NSLog(@"หลังลบ index 0: %@", list);
        
        // ลบตัวแรก
        [list removeFirstObject];  // ถ้ามี
        // ลบตัวสุดท้าย
        [numbers removeLastObject];
        NSLog(@"\nNumbers หลังลบสุดท้าย: %@", numbers);
        
        // ลบทั้งหมด
        NSMutableArray *temp = [@[@1, @2, @3] mutableCopy];
        [temp removeAllObjects];
        NSLog(@"\nTemp หลัง removeAll: %@ (จำนวน: %lu)", temp, [temp count]);
        
        NSLog(@"\n=== การแก้ไขสมาชิก ===");
        NSMutableArray *fruits = [@[@"Apple", @"Banana", @"Cherry"] mutableCopy];
        NSLog(@"ก่อน: %@", fruits);
        
        // แก้ไขตาม index
        [fruits replaceObjectAtIndex:1 withObject:@"Blueberry"];
        NSLog(@"หลังแก้ไข index 1: %@", fruits);
        
        // แก้ไขด้วย subscript
        fruits[2] = @"Cranberry";
        NSLog(@"หลังแก้ไข index 2: %@", fruits);
        
        // สลับตำแหน่ง
        [fruits exchangeObjectAtIndex:0 withObjectAtIndex:2];
        NSLog(@"หลังสลับ 0 กับ 2: %@", fruits);
        
        NSLog(@"\n=== Stack operations ===");
        NSMutableArray *stack = [NSMutableArray array];
        
        // Push
        [stack addObject:@"ชั้น 1"];
        [stack addObject:@"ชั้น 2"];
        [stack addObject:@"ชั้น 3"];
        NSLog(@"Stack: %@", stack);
        
        // Pop (ลบตัวสุดท้าย)
        NSString *popped = [stack lastObject];
        [stack removeLastObject];
        NSLog(@"Pop: %@, Stack เหลือ: %@", popped, stack);
        
        NSLog(@"\n=== Queue operations ===");
        NSMutableArray *queue = [NSMutableArray array];
        
        // Enqueue
        [queue addObject:@"คนที่ 1"];
        [queue addObject:@"คนที่ 2"];
        [queue addObject:@"คนที่ 3"];
        NSLog(@"Queue: %@", queue);
        
        // Dequeue (ลบตัวแรก)
        NSString *dequeued = [queue firstObject];
        [queue removeObjectAtIndex:0];
        NSLog(@"Dequeue: %@, Queue เหลือ: %@", dequeued, queue);
    }
    return 0;
}
```

---

## 7.8 การเรียงลำดับ Array

### การเรียงด้วย NSSortDescriptor

```objc
#import <Foundation/Foundation.h>

@interface Person : NSObject
@property (nonatomic, strong) NSString *name;
@property (nonatomic, assign) NSInteger age;
@property (nonatomic, assign) double score;

+ (instancetype)personWithName:(NSString *)name age:(NSInteger)age score:(double)score;
@end

@implementation Person
+ (instancetype)personWithName:(NSString *)name age:(NSInteger)age score:(double)score {
    Person *p = [[Person alloc] init];
    p.name = name;
    p.age = age;
    p.score = score;
    return p;
}

- (NSString *)description {
    return [NSString stringWithFormat:@"Person{name='%@', age=%ld, score=%.1f}",
            self.name, (long)self.age, self.score];
}
@end

int main(int argc, const char * argv[]) {
    @autoreleasepool {
        
        NSArray *people = @[
            [Person personWithName:@"สมชาย" age:25 score:88.5],
            [Person personWithName:@"มณี" age:30 score:92.0],
            [Person personWithName:@"อนุชา" age:22 score:75.5],
            [Person personWithName:@"วิภา" age:28 score:88.5],
            [Person personWithName:@"ประยุทธ์" age:35 score:95.0]
        ];
        
        NSLog(@"=== ข้อมูลเดิม ===");
        for (Person *p in people) {
            NSLog(@"%@", p);
        }
        
        // เรียงตามชื่อ
        NSSortDescriptor *nameDesc = [NSSortDescriptor sortDescriptorWithKey:@"name"
                                                                   ascending:YES];
        NSArray *sortedByName = [people sortedArrayUsingDescriptors:@[nameDesc]];
        
        NSLog(@"\n=== เรียงตามชื่อ (A-Z) ===");
        for (Person *p in sortedByName) {
            NSLog(@"%@", p);
        }
        
        // เรียงตาม score มากไปน้อย
        NSSortDescriptor *scoreDesc = [NSSortDescriptor sortDescriptorWithKey:@"score"
                                                                    ascending:NO];
        NSArray *sortedByScore = [people sortedArrayUsingDescriptors:@[scoreDesc]];
        
        NSLog(@"\n=== เรียงตามคะแนน (สูง→ต่ำ) ===");
        for (Person *p in sortedByScore) {
            NSLog(@"%@", p);
        }
        
        // เรียงหลายเงื่อนไข: score มากไปน้อย แล้ว age น้อยไปมาก
        NSArray *sortedMulti = [people sortedArrayUsingDescriptors:@[scoreDesc,
            [NSSortDescriptor sortDescriptorWithKey:@"age" ascending:YES]]];
        
        NSLog(@"\n=== เรียงตาม score แล้ว age ===");
        for (Person *p in sortedMulti) {
            NSLog(@"%@", p);
        }
        
        // เรียงด้วย comparator block
        NSArray *sortedByLength = @[@"กล้วย", @"แอปเปิ้ล", @"ส้ม", @"สตรอว์เบอร์รี่", @"มะม่วง"];
        sortedByLength = [sortedByLength sortedArrayUsingComparator:^NSComparisonResult(NSString *a, NSString *b) {
            return [@(a.length) compare:@(b.length)];
        }];
        NSLog(@"\n=== เรียงตามความยาวชื่อ ===");
        for (NSString *s in sortedByLength) {
            NSLog(@"%@ (%lu)", s, s.length);
        }
    }
    return 0;
}
```

### การเรียง NSMutableArray

```objc
#import <Foundation/Foundation.h>

int main(int argc, const char * argv[]) {
    @autoreleasepool {
        
        NSMutableArray *numbers = [@[@5, @2, @8, @1, @9, @3, @7, @4, @6] mutableCopy];
        NSLog(@"ก่อนเรียง: %@", numbers);
        
        // เรียง in-place ด้วย NSSortDescriptor
        NSSortDescriptor *ascending = [NSSortDescriptor sortDescriptorWithKey:@"self"
                                                                    ascending:YES];
        [numbers sortUsingDescriptors:@[ascending]];
        NSLog(@"เรียงน้อยไปมาก: %@", numbers);
        
        // เรียงด้วย comparator
        [numbers sortUsingComparator:^NSComparisonResult(NSNumber *a, NSNumber *b) {
            return [b compare:a];  // สลับ a และ b เพื่อเรียงจากมากไปน้อย
        }];
        NSLog(@"เรียงมากไปน้อย: %@", numbers);
        
        // เรียง strings
        NSMutableArray *names = [@[@"Charlie", @"Alice", @"Bob", @"David"] mutableCopy];
        [names sortUsingSelector:@selector(localizedCaseInsensitiveCompare:)];
        NSLog(@"\nชื่อเรียง: %@", names);
    }
    return 0;
}
```

---

## 7.9 การกรอง Array ด้วย NSPredicate

```objc
#import <Foundation/Foundation.h>

int main(int argc, const char * argv[]) {
    @autoreleasepool {
        
        NSArray *numbers = @[@1, @5, @12, @3, @18, @7, @25, @9, @15, @2];
        
        NSLog(@"=== กรองด้วย NSPredicate ===");
        NSLog(@"ข้อมูลเดิม: %@", numbers);
        
        // กรองตัวเลขที่มากกว่า 10
        NSPredicate *biggerThan10 = [NSPredicate predicateWithFormat:@"SELF > 10"];
        NSArray *filtered = [numbers filteredArrayUsingPredicate:biggerThan10];
        NSLog(@"\nตัวเลข > 10: %@", filtered);
        
        // กรองตัวเลขคี่
        NSPredicate *oddNumbers = [NSPredicate predicateWithBlock:^BOOL(NSNumber *n, NSDictionary *bindings) {
            return [n intValue] % 2 != 0;
        }];
        NSArray *odds = [numbers filteredArrayUsingPredicate:oddNumbers];
        NSLog(@"ตัวเลขคี่: %@", odds);
        
        // กรองช่วง
        NSPredicate *range = [NSPredicate predicateWithFormat:@"SELF >= 5 AND SELF <= 15"];
        NSArray *inRange = [numbers filteredArrayUsingPredicate:range];
        NSLog(@"ตัวเลข 5-15: %@", inRange);
        
        // กรอง strings
        NSArray *fruits = @[@"Apple", @"Banana", @"Apricot", @"Blueberry", @"Cherry", @"Avocado"];
        
        // กรองที่ขึ้นต้นด้วย 'A'
        NSPredicate *startsWithA = [NSPredicate predicateWithFormat:@"SELF BEGINSWITH 'A'"];
        NSArray *aFruits = [fruits filteredArrayUsingPredicate:startsWithA];
        NSLog(@"\nผลไม้ขึ้นต้นด้วย A: %@", aFruits);
        
        // กรองที่มีตัวอักษรบางตัว (case insensitive)
        NSPredicate *containsN = [NSPredicate predicateWithFormat:@"SELF CONTAINS[cd] 'an'"];
        NSArray *containsAN = [fruits filteredArrayUsingPredicate:containsN];
        NSLog(@"ผลไม้ที่มี 'an': %@", containsAN);
        
        // กรองด้วย properties ของ object
        NSArray *people = @[
            @{@"name": @"สมชาย", @"age": @25},
            @{@"name": @"มณี", @"age": @17},
            @{@"name": @"อนุชา", @"age": @22},
            @{@"name": @"เด็กน้อย", @"age": @15}
        ];
        
        NSPredicate *adultPredicate = [NSPredicate predicateWithFormat:@"age >= 18"];
        NSArray *adults = [people filteredArrayUsingPredicate:adultPredicate];
        NSLog(@"\nผู้ใหญ่ (อายุ >= 18): %@", adults);
        
        // กรองด้วย IN
        NSArray *selectedCities = @[@"กรุงเทพ", @"เชียงใหม่"];
        NSArray *allCities = @[@"กรุงเทพ", @"ภูเก็ต", @"เชียงใหม่", @"พัทยา", @"อยุธยา"];
        NSPredicate *inCities = [NSPredicate predicateWithFormat:@"SELF IN %@", selectedCities];
        NSArray *matchCities = [allCities filteredArrayUsingPredicate:inCities];
        NSLog(@"\nเมืองที่เลือก: %@", matchCities);
        
        // NSMutableArray กรอง
        NSMutableArray *mutableNumbers = [@[@1, @2, @3, @4, @5, @6, @7, @8, @9, @10] mutableCopy];
        NSPredicate *evenPred = [NSPredicate predicateWithFormat:@"SELF %% 2 == 0"];
        [mutableNumbers filterUsingPredicate:evenPred];
        NSLog(@"\nตัวเลขคู่ใน mutable array: %@", mutableNumbers);
    }
    return 0;
}
```

---

## 7.10 การคัดลอก Array

```objc
#import <Foundation/Foundation.h>

int main(int argc, const char * argv[]) {
    @autoreleasepool {
        
        // Shallow copy vs Deep copy
        NSArray *original = @[@"A", @"B", @"C"];
        
        // copy → คืน NSArray (immutable)
        NSArray *shallowCopy = [original copy];
        
        // mutableCopy → คืน NSMutableArray
        NSMutableArray *mutableCopy = [original mutableCopy];
        
        NSLog(@"=== Copying Arrays ===");
        NSLog(@"Original: %@", original);
        NSLog(@"Shallow copy: %@", shallowCopy);
        NSLog(@"Mutable copy: %@", mutableCopy);
        
        // แก้ไข mutableCopy ไม่กระทบ original
        [mutableCopy addObject:@"D"];
        NSLog(@"\nหลังเพิ่ม D ใน mutableCopy:");
        NSLog(@"Original: %@", original);
        NSLog(@"MutableCopy: %@", mutableCopy);
        
        // Shallow copy ของ NSMutableArray
        NSMutableArray *mut1 = [NSMutableArray arrayWithObjects:
                                [NSMutableString stringWithString:@"Hello"],
                                [NSMutableString stringWithString:@"World"],
                                nil];
        
        NSMutableArray *mut2 = [mut1 mutableCopy];  // shallow copy
        
        // เพิ่ม element ใน mut2 ไม่กระทบ mut1
        [mut2 addObject:@"New"];
        NSLog(@"\nmut1 count: %lu", [mut1 count]);
        NSLog(@"mut2 count: %lu", [mut2 count]);
        
        // แต่ถ้าแก้ไข object ที่แชร์กัน จะกระทบทั้งคู่ (shallow copy)
        NSMutableString *shared = (NSMutableString *)mut1[0];
        [shared appendString:@" World"];
        NSLog(@"\nmut1[0]: %@", mut1[0]);
        NSLog(@"mut2[0]: %@", mut2[0]);  // เปลี่ยนด้วยเพราะชี้ object เดียวกัน
        
        // Deep copy ด้วย archiving
        NSMutableArray *original2 = [NSMutableArray arrayWithObjects:
                                     [NSMutableString stringWithString:@"X"],
                                     [NSMutableString stringWithString:@"Y"],
                                     nil];
        
        NSMutableArray *deepCopy = (NSMutableArray *)[NSKeyedUnarchiver
                                                       unarchivedObjectOfClass:[NSMutableArray class]
                                                                      fromData:[NSKeyedArchiver archivedDataWithRootObject:original2
                                                                                                     requiringSecureCoding:NO
                                                                                                                    error:nil]
                                                                         error:nil];
        
        // แก้ไข object ใน deepCopy จะไม่กระทบ original2
        NSMutableString *x = (NSMutableString *)deepCopy[0];
        [x appendString:@"_modified"];
        NSLog(@"\nDeep copy test:");
        NSLog(@"original2[0]: %@", original2[0]);  // ไม่เปลี่ยน
        NSLog(@"deepCopy[0]: %@", deepCopy[0]);    // เปลี่ยน
    }
    return 0;
}
```

---

## 7.11 สรุปเนื้อหา

| หัวข้อ | รายละเอียด |
|--------|-----------|
| C Array | เก็บข้อมูลชนิดเดียวกัน มีขนาดคงที่ |
| Multi-dim Array | อาร์เรย์หลายมิติ เช่น matrix |
| Pointer | ตัวแปรเก็บ memory address |
| * operator | Dereference - เข้าถึงค่าที่ address |
| & operator | Address-of - หา address ของตัวแปร |
| -> operator | เข้าถึง member ผ่าน pointer |
| NSArray | Immutable array ของ Objective-C objects |
| NSMutableArray | Mutable array แก้ไขได้ |
| NSSortDescriptor | เรียงลำดับ array |
| NSPredicate | กรอง array ตามเงื่อนไข |

---

## แบบฝึกหัด

### ข้อที่ 1: ผลรวมและค่าเฉลี่ย
```objc
// เขียนโปรแกรมที่รับตัวเลข 10 ตัวใส่ใน C array
// แล้วคำนวณผลรวม ค่าเฉลี่ย ค่าต่ำสุด และค่าสูงสุด
int scores[] = {75, 88, 92, 65, 70, 85, 90, 78, 82, 95};
// หาค่าต่างๆ และแสดงผล
```

### ข้อที่ 2: Matrix transpose
```objc
// เขียนฟังก์ชัน transpose matrix 3x3
int matrix[3][3] = {{1, 2, 3}, {4, 5, 6}, {7, 8, 9}};
// ผลลัพธ์ที่ต้องการ: {{1,4,7}, {2,5,8}, {3,6,9}}
```

### ข้อที่ 3: การใช้ pointer swap
```objc
// เขียนฟังก์ชัน swap ที่ใช้ pointer
// แล้วใช้ใน bubble sort สำหรับ array
void swap(int *a, int *b) {
    // TODO: implement
}
```

### ข้อที่ 4: NSArray ของ numbers
```objc
// สร้าง NSArray ของ NSNumber (1-20)
// กรองเอาเฉพาะเลขคู่
// เรียงจากมากไปน้อย
// แสดงผลรวม
```

### ข้อที่ 5: คลังหนังสือ
```objc
// สร้าง NSMutableArray เก็บข้อมูลหนังสือ (dictionary)
// ฟังก์ชันเพิ่ม ลบ ค้นหา และแสดงผล
NSMutableArray *library = [NSMutableArray array];
// เพิ่มหนังสือ: title, author, year
// ค้นหาหนังสือตาม author
// ลบหนังสือตาม title
```

### ข้อที่ 6: การเรียงข้อมูลนักศึกษา
```objc
// สร้าง array ของ Person objects
// เรียงตาม GPA จากสูงไปต่ำ
// ถ้า GPA เท่ากัน ให้เรียงตามชื่อ
```

### ข้อที่ 7: กรองข้อมูล
```objc
// สร้าง array ของ products (dictionary มี name, price, category)
// กรองเฉพาะ category "Electronics"
// ที่ราคาน้อยกว่า 5000
```

### ข้อที่ 8: String array operations
```objc
// จากข้อความ: "the quick brown fox jumps over the lazy dog"
// แยกเป็น words array
// กรองเฉพาะ words ที่ยาวกว่า 3 ตัวอักษร
// เรียง alphabetically
// join กลับเป็น string ด้วย space
```

### ข้อที่ 9: Pointer ของ struct
```objc
typedef struct {
    char brand[30];
    int year;
    float price;
} Car;

// สร้าง array ของ Car
// เขียนฟังก์ชันรับ Car* และพิมพ์ข้อมูล
// เขียนฟังก์ชันหา Car ที่ราคาถูกที่สุด
```

### ข้อที่ 10: Stack ด้วย NSMutableArray
```objc
// สร้าง Stack class โดยใช้ NSMutableArray ภายใน
// มี methods: push:, pop, peek, isEmpty, size
@interface Stack : NSObject
- (void)push:(id)object;
- (id)pop;
- (id)peek;
- (BOOL)isEmpty;
- (NSUInteger)size;
@end
```

### ข้อที่ 11: Frequency count
```objc
// จาก array ของตัวเลข
// นับความถี่ของแต่ละค่า
// แสดงค่าที่พบบ่อยที่สุด
NSArray *data = @[@3, @1, @4, @1, @5, @9, @2, @6, @5, @3, @5];
// ผลลัพธ์: 5 พบ 3 ครั้ง
```

### ข้อที่ 12 (ท้าทาย): Queue ด้วย NSMutableArray
```objc
// สร้าง Queue (FIFO) ด้วย NSMutableArray
// ใช้ใน simulation คนเข้าคิว
// ทุก 1 วินาที มีคนเข้า 1-3 คน
// ทุก 1 วินาที ให้บริการ 2 คน
// simulate 5 รอบ แสดงคิวแต่ละรอบ
```

---

## เฉลยตัวอย่าง (ข้อที่ 4)

```objc
#import <Foundation/Foundation.h>

int main(int argc, const char * argv[]) {
    @autoreleasepool {
        
        // สร้าง NSArray ของ NSNumber 1-20
        NSMutableArray *tempArray = [NSMutableArray array];
        for (int i = 1; i <= 20; i++) {
            [tempArray addObject:@(i)];
        }
        NSArray *numbers = [tempArray copy];
        
        // กรองเลขคู่
        NSPredicate *evenPred = [NSPredicate predicateWithFormat:@"SELF %% 2 == 0"];
        NSArray *evenNumbers = [numbers filteredArrayUsingPredicate:evenPred];
        NSLog(@"เลขคู่: %@", evenNumbers);
        
        // เรียงจากมากไปน้อย
        NSArray *sorted = [evenNumbers sortedArrayUsingComparator:^NSComparisonResult(NSNumber *a, NSNumber *b) {
            return [b compare:a];
        }];
        NSLog(@"เรียงจากมากไปน้อย: %@", sorted);
        
        // ผลรวม
        NSInteger sum = 0;
        for (NSNumber *n in sorted) {
            sum += [n integerValue];
        }
        NSLog(@"ผลรวมเลขคู่ 1-20: %ld", (long)sum);
    }
    return 0;
}
```

---

*จบบทที่ 7: Arrays และ Pointers* | ไปยัง [บทที่ 8: NSString Deep Dive →](part-08-nsstring.md)
