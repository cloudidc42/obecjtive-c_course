# ส่วนที่ 06: ฟังก์ชัน (Functions) ใน Objective-C

## บทนำ

ฟังก์ชัน (Functions) คือกลุ่มของคำสั่งที่รวมกันเพื่อทำงานเฉพาะอย่าง ช่วยให้โค้ดอ่านง่าย ใช้ซ้ำได้ และบำรุงรักษาง่ายขึ้น ใน Objective-C เราสามารถเขียนฟังก์ชันแบบ C-style ได้ ซึ่งเป็นพื้นฐานสำคัญก่อนจะเรียนรู้ Methods ของ Objective-C ในภายหลัง

---

## 6.1 การประกาศและนิยามฟังก์ชัน

### รูปแบบพื้นฐาน

```
returnType functionName(parameter1Type parameter1Name, ...) {
    // function body
    return value; // ถ้า returnType ไม่ใช่ void
}
```

```objc
#import <Foundation/Foundation.h>

// ฟังก์ชันง่ายๆ ไม่มี parameter ไม่มี return value
void sayHello(void) {
    NSLog(@"สวัสดี ยินดีต้อนรับ!");
}

// ฟังก์ชันรับ parameter และคืนค่า
int add(int a, int b) {
    return a + b;
}

// ฟังก์ชันรับ NSString และคืน NSString
NSString* greet(NSString *name) {
    return [NSString stringWithFormat:@"สวัสดี คุณ%@!", name];
}

// ฟังก์ชันคำนวณหลายค่า
double calculateCircleArea(double radius) {
    return M_PI * radius * radius;
}

int main(int argc, const char * argv[]) {
    @autoreleasepool {
        sayHello();
        
        int sum = add(5, 3);
        NSLog(@"5 + 3 = %d", sum);
        
        NSString *greeting = greet(@"สมชาย");
        NSLog(@"%@", greeting);
        
        double area = calculateCircleArea(5.0);
        NSLog(@"พื้นที่วงกลม radius=5: %.2f", area);
    }
    return 0;
}
```

**ผลลัพธ์:**
```
สวัสดี ยินดีต้อนรับ!
5 + 3 = 8
สวัสดี คุณสมชาย!
พื้นที่วงกลม radius=5: 78.54
```

---

## 6.2 Return Types และ Parameters

### ประเภทของ Return Types

```objc
#import <Foundation/Foundation.h>

// คืนค่า int
int square(int n) {
    return n * n;
}

// คืนค่า double
double hypotenuse(double a, double b) {
    return sqrt(a*a + b*b);
}

// คืนค่า BOOL
BOOL isEven(int n) {
    return (n % 2 == 0);
}

// คืนค่า NSString*
NSString* repeatString(NSString *str, int times) {
    NSMutableString *result = [NSMutableString string];
    for (int i = 0; i < times; i++) {
        [result appendString:str];
    }
    return [result copy];
}

// คืนค่า NSArray*
NSArray* createRange(int from, int to) {
    NSMutableArray *range = [NSMutableArray array];
    for (int i = from; i <= to; i++) {
        [range addObject:@(i)];
    }
    return [range copy];
}

int main(int argc, const char * argv[]) {
    @autoreleasepool {
        NSLog(@"square(7) = %d", square(7));
        NSLog(@"hypotenuse(3, 4) = %.2f", hypotenuse(3, 4));
        NSLog(@"isEven(8) = %@", isEven(8) ? @"YES" : @"NO");
        NSLog(@"isEven(7) = %@", isEven(7) ? @"YES" : @"NO");
        NSLog(@"repeatString('Hi ', 3) = %@", repeatString(@"Hi ", 3));
        NSLog(@"createRange(1, 5) = %@", createRange(1, 5));
    }
    return 0;
}
```

### Parameters หลายประเภท

```objc
#import <Foundation/Foundation.h>

// Parameters แบบผสมประเภท
NSString* formatPersonInfo(NSString *name, int age, double salary) {
    return [NSString stringWithFormat:
            @"ชื่อ: %@, อายุ: %d ปี, เงินเดือน: %.2f บาท",
            name, age, salary];
}

// Default-like behavior ด้วยค่าพิเศษ
void printMessage(NSString *message, NSString *prefix) {
    if (prefix == nil || [prefix length] == 0) {
        prefix = @"INFO";
    }
    NSLog(@"[%@] %@", prefix, message);
}

// Parameters Array/Collection
double calculateAverage(NSArray *numbers) {
    if ([numbers count] == 0) return 0.0;
    
    double sum = 0;
    for (NSNumber *n in numbers) {
        sum += [n doubleValue];
    }
    return sum / [numbers count];
}

int main(int argc, const char * argv[]) {
    @autoreleasepool {
        NSString *info = formatPersonInfo(@"สมชาย", 30, 35000.0);
        NSLog(@"%@", info);
        
        printMessage(@"ระบบเริ่มทำงาน", nil);
        printMessage(@"พบข้อผิดพลาด", @"ERROR");
        
        NSArray *scores = @[@85, @92, @78, @95, @88];
        NSLog(@"ค่าเฉลี่ย: %.2f", calculateAverage(scores));
    }
    return 0;
}
```

---

## 6.3 void Functions

void functions ทำงานโดยไม่คืนค่า เหมาะสำหรับงานที่มี side effects

```objc
#import <Foundation/Foundation.h>

// void function พิมพ์ข้อมูล
void printDivider(int width, char symbol) {
    NSMutableString *line = [NSMutableString string];
    for (int i = 0; i < width; i++) {
        [line appendFormat:@"%c", symbol];
    }
    NSLog(@"%@", line);
}

// void function แก้ไข array (via pointer)
void fillWithZeros(int *arr, int size) {
    for (int i = 0; i < size; i++) {
        arr[i] = 0;
    }
}

// void function แสดง report
void printSalesReport(NSArray *sales) {
    printDivider(40, '=');
    NSLog(@"รายงานยอดขาย");
    printDivider(40, '-');
    
    double total = 0;
    int count = 0;
    double maxSale = 0;
    NSString *topProduct = nil;
    
    for (NSDictionary *sale in sales) {
        double amount = [sale[@"amount"] doubleValue];
        total += amount;
        count++;
        
        if (amount > maxSale) {
            maxSale = amount;
            topProduct = sale[@"product"];
        }
        
        NSLog(@"%-15@ : %.2f บาท", sale[@"product"], amount);
    }
    
    printDivider(40, '-');
    NSLog(@"รวม %d รายการ : %.2f บาท", count, total);
    NSLog(@"สินค้าขายดีที่สุด: %@ (%.2f บาท)", topProduct, maxSale);
    printDivider(40, '=');
}

int main(int argc, const char * argv[]) {
    @autoreleasepool {
        NSArray *salesData = @[
            @{@"product": @"แอปเปิ้ล",   @"amount": @1250.0},
            @{@"product": @"กล้วย",      @"amount": @450.0},
            @{@"product": @"ส้ม",         @"amount": @890.0},
            @{@"product": @"ทุเรียน",    @"amount": @3500.0},
            @{@"product": @"มะม่วง",     @"amount": @760.0}
        ];
        
        printSalesReport(salesData);
        
        // void function กับ array
        int myArray[5];
        fillWithZeros(myArray, 5);
        for (int i = 0; i < 5; i++) {
            NSLog(@"arr[%d] = %d", i, myArray[i]);
        }
    }
    return 0;
}
```

---

## 6.4 Function Prototypes

Function prototype คือการประกาศฟังก์ชันก่อนใช้งาน ช่วยให้สามารถเรียกใช้ฟังก์ชันก่อนนิยาม

```objc
#import <Foundation/Foundation.h>

// Prototypes - ประกาศก่อน
int factorial(int n);
double power(double base, int exp);
BOOL isPrime(int n);
void printPrimes(int limit);

// main function ใช้ฟังก์ชันที่ประกาศไว้
int main(int argc, const char * argv[]) {
    @autoreleasepool {
        NSLog(@"5! = %d", factorial(5));
        NSLog(@"2^10 = %.0f", power(2, 10));
        NSLog(@"17 เป็นจำนวนเฉพาะ: %@", isPrime(17) ? @"ใช่" : @"ไม่ใช่");
        
        NSLog(@"\nจำนวนเฉพาะ 1-30:");
        printPrimes(30);
    }
    return 0;
}

// นิยามฟังก์ชัน (อยู่หลัง main ได้เพราะมี prototype)
int factorial(int n) {
    if (n < 0) return -1;
    if (n == 0) return 1;
    
    int result = 1;
    for (int i = 1; i <= n; i++) {
        result *= i;
    }
    return result;
}

double power(double base, int exp) {
    if (exp == 0) return 1;
    
    double result = 1;
    for (int i = 0; i < exp; i++) {
        result *= base;
    }
    return result;
}

BOOL isPrime(int n) {
    if (n < 2) return NO;
    if (n == 2) return YES;
    if (n % 2 == 0) return NO;
    
    for (int i = 3; i * i <= n; i += 2) {
        if (n % i == 0) return NO;
    }
    return YES;
}

void printPrimes(int limit) {
    for (int i = 2; i <= limit; i++) {
        if (isPrime(i)) {
            NSLog(@"%d", i);
        }
    }
}
```

---

## 6.5 Passing by Value vs Pointer

### Pass by Value (ค่าไม่เปลี่ยน)

```objc
#import <Foundation/Foundation.h>

void tryToDouble_ByValue(int num) {
    num = num * 2; // แก้ไข local copy เท่านั้น
    NSLog(@"ภายในฟังก์ชัน: %d", num);
}

// Pass by Pointer (ค่าเปลี่ยน)
void doubleByPointer(int *numPtr) {
    *numPtr = *numPtr * 2; // แก้ไขค่าจริงผ่าน pointer
    NSLog(@"ภายในฟังก์ชัน: %d", *numPtr);
}

// swap สองค่า (ต้องใช้ pointer)
void swapInts(int *a, int *b) {
    int temp = *a;
    *a = *b;
    *b = temp;
}

// แก้ไข NSMutableString (pass by pointer ผ่าน object reference)
void addSuffix(NSMutableString *str, NSString *suffix) {
    [str appendString:suffix];
}

int main(int argc, const char * argv[]) {
    @autoreleasepool {
        NSLog(@"=== Pass by Value ===");
        int x = 5;
        NSLog(@"ก่อนเรียก: %d", x);
        tryToDouble_ByValue(x);
        NSLog(@"หลังเรียก: %d", x); // ยังเป็น 5
        
        NSLog(@"\n=== Pass by Pointer ===");
        x = 5;
        NSLog(@"ก่อนเรียก: %d", x);
        doubleByPointer(&x);
        NSLog(@"หลังเรียก: %d", x); // เป็น 10 แล้ว
        
        NSLog(@"\n=== Swap ===");
        int a = 10, b = 20;
        NSLog(@"ก่อน swap: a=%d, b=%d", a, b);
        swapInts(&a, &b);
        NSLog(@"หลัง swap: a=%d, b=%d", a, b);
        
        NSLog(@"\n=== NSMutableString ===");
        NSMutableString *name = [NSMutableString stringWithString:@"สมชาย"];
        NSLog(@"ก่อน: %@", name);
        addSuffix(name, @" รัตนชัย");
        NSLog(@"หลัง: %@", name);
    }
    return 0;
}
```

**ผลลัพธ์:**
```
=== Pass by Value ===
ก่อนเรียก: 5
ภายในฟังก์ชัน: 10
หลังเรียก: 5

=== Pass by Pointer ===
ก่อนเรียก: 5
ภายในฟังก์ชัน: 10
หลังเรียก: 10

=== Swap ===
ก่อน swap: a=10, b=20
หลัง swap: a=20, b=10
```

### Pointer to Struct

```objc
#import <Foundation/Foundation.h>

typedef struct {
    char name[50];
    int age;
    double salary;
} Employee;

void promoteEmployee(Employee *emp, double raiseAmount) {
    emp->salary += raiseAmount;
    emp->age++; // สมมติว่าเวลาผ่านไป 1 ปี
    NSLog(@"เลื่อนขั้น: %s อายุ %d เงินเดือนใหม่: %.2f",
          emp->name, emp->age, emp->salary);
}

int main(int argc, const char * argv[]) {
    @autoreleasepool {
        Employee emp;
        strcpy(emp.name, "Somchai");
        emp.age = 28;
        emp.salary = 30000.0;
        
        NSLog(@"ก่อน: %s อายุ %d เงินเดือน: %.2f",
              emp.name, emp.age, emp.salary);
        
        promoteEmployee(&emp, 5000.0);
        
        NSLog(@"หลัง: %s อายุ %d เงินเดือน: %.2f",
              emp.name, emp.age, emp.salary);
    }
    return 0;
}
```

---

## 6.6 Recursive Functions

ฟังก์ชัน recursive คือฟังก์ชันที่เรียกตัวเองซ้ำๆ

### ตัวอย่าง Recursive Functions

```objc
#import <Foundation/Foundation.h>

// Factorial แบบ recursive
long factorialRecursive(int n) {
    if (n <= 1) return 1; // Base case
    return n * factorialRecursive(n - 1); // Recursive case
}

// Fibonacci แบบ recursive (แบบง่าย แต่ช้า)
long fibonacciSlow(int n) {
    if (n <= 1) return n; // Base case: F(0)=0, F(1)=1
    return fibonacciSlow(n - 1) + fibonacciSlow(n - 2);
}

// Fibonacci แบบ memoization (เร็วขึ้นมาก)
long memo[100];

long fibonacciFast(int n) {
    if (n <= 1) return n;
    if (memo[n] != -1) return memo[n];
    memo[n] = fibonacciFast(n - 1) + fibonacciFast(n - 2);
    return memo[n];
}

// Binary Search แบบ recursive
int binarySearch(int arr[], int left, int right, int target) {
    if (left > right) return -1; // Base case: ไม่พบ
    
    int mid = left + (right - left) / 2;
    
    if (arr[mid] == target) return mid; // Base case: พบ
    
    if (arr[mid] < target) {
        return binarySearch(arr, mid + 1, right, target); // ค้นด้านขวา
    } else {
        return binarySearch(arr, left, mid - 1, target); // ค้นด้านซ้าย
    }
}

// Power แบบ recursive (fast exponentiation)
double powerFast(double base, int exp) {
    if (exp == 0) return 1;
    if (exp % 2 == 0) {
        double half = powerFast(base, exp / 2);
        return half * half;
    }
    return base * powerFast(base, exp - 1);
}

int main(int argc, const char * argv[]) {
    @autoreleasepool {
        NSLog(@"=== Factorial ===");
        for (int i = 0; i <= 10; i++) {
            NSLog(@"%d! = %ld", i, factorialRecursive(i));
        }
        
        NSLog(@"\n=== Fibonacci (Fast with Memo) ===");
        memset(memo, -1, sizeof(memo));
        for (int i = 0; i <= 15; i++) {
            NSLog(@"F(%d) = %ld", i, fibonacciFast(i));
        }
        
        NSLog(@"\n=== Binary Search ===");
        int sortedArray[] = {2, 5, 8, 12, 16, 23, 38, 56, 72, 91};
        int n = sizeof(sortedArray) / sizeof(sortedArray[0]);
        
        int targets[] = {23, 100, 2, 91};
        for (int i = 0; i < 4; i++) {
            int idx = binarySearch(sortedArray, 0, n - 1, targets[i]);
            if (idx >= 0) {
                NSLog(@"พบ %d ที่ index %d", targets[i], idx);
            } else {
                NSLog(@"ไม่พบ %d", targets[i]);
            }
        }
        
        NSLog(@"\n=== Fast Power ===");
        NSLog(@"2^10 = %.0f", powerFast(2, 10));
        NSLog(@"3^5 = %.0f", powerFast(3, 5));
    }
    return 0;
}
```

### Recursive Tree Traversal

```objc
#import <Foundation/Foundation.h>

typedef struct TreeNode {
    int value;
    struct TreeNode *left;
    struct TreeNode *right;
} TreeNode;

TreeNode* createNode(int value) {
    TreeNode *node = (TreeNode *)malloc(sizeof(TreeNode));
    node->value = value;
    node->left = NULL;
    node->right = NULL;
    return node;
}

// In-order traversal: left -> root -> right
void inorderTraversal(TreeNode *node) {
    if (node == NULL) return;
    inorderTraversal(node->left);
    NSLog(@"%d", node->value);
    inorderTraversal(node->right);
}

// คำนวณความสูงของต้นไม้
int treeHeight(TreeNode *node) {
    if (node == NULL) return 0;
    int leftHeight = treeHeight(node->left);
    int rightHeight = treeHeight(node->right);
    return 1 + (leftHeight > rightHeight ? leftHeight : rightHeight);
}

void freeTree(TreeNode *node) {
    if (node == NULL) return;
    freeTree(node->left);
    freeTree(node->right);
    free(node);
}

int main(int argc, const char * argv[]) {
    @autoreleasepool {
        //      5
        //    /   \
        //   3     8
        //  / \   / \
        // 1   4 7   9
        
        TreeNode *root = createNode(5);
        root->left = createNode(3);
        root->right = createNode(8);
        root->left->left = createNode(1);
        root->left->right = createNode(4);
        root->right->left = createNode(7);
        root->right->right = createNode(9);
        
        NSLog(@"=== In-order Traversal (เรียงลำดับ) ===");
        inorderTraversal(root);
        
        NSLog(@"\nความสูงของต้นไม้: %d", treeHeight(root));
        
        freeTree(root);
    }
    return 0;
}
```

---

## 6.7 Static Functions

`static` function มองเห็นได้เฉพาะภายใน file เดียวกัน

```objc
#import <Foundation/Foundation.h>

// Static function - ใช้ได้เฉพาะใน file นี้
static double degreesToRadians(double degrees) {
    return degrees * M_PI / 180.0;
}

static double radiansToDegrees(double radians) {
    return radians * 180.0 / M_PI;
}

// Public functions ที่เรียกใช้ static helpers
double sinDegrees(double degrees) {
    return sin(degreesToRadians(degrees));
}

double cosDegrees(double degrees) {
    return cos(degreesToRadians(degrees));
}

// Static variable ใน function - คงค่าระหว่าง calls
int callCounter(void) {
    static int count = 0; // เริ่มต้นครั้งเดียว
    count++;
    return count;
}

int main(int argc, const char * argv[]) {
    @autoreleasepool {
        NSLog(@"=== Trigonometry Functions ===");
        NSLog(@"sin(30°) = %.4f", sinDegrees(30));
        NSLog(@"cos(60°) = %.4f", cosDegrees(60));
        NSLog(@"sin(90°) = %.4f", sinDegrees(90));
        
        NSLog(@"\n=== Static Variable Counter ===");
        for (int i = 0; i < 5; i++) {
            NSLog(@"เรียกครั้งที่ %d", callCounter());
        }
    }
    return 0;
}
```

---

## 6.8 Inline Functions

`inline` แนะนำให้ compiler แทนที่การเรียกฟังก์ชันด้วยโค้ดโดยตรง

```objc
#import <Foundation/Foundation.h>

// inline functions สำหรับ operations ที่เรียกบ่อย
static inline int maxInt(int a, int b) {
    return (a > b) ? a : b;
}

static inline int minInt(int a, int b) {
    return (a < b) ? a : b;
}

static inline int clamp(int value, int lo, int hi) {
    return maxInt(lo, minInt(value, hi));
}

static inline double lerp(double a, double b, double t) {
    return a + (b - a) * t;
}

static inline BOOL inRange(double value, double min, double max) {
    return value >= min && value <= max;
}

int main(int argc, const char * argv[]) {
    @autoreleasepool {
        NSLog(@"max(5, 8) = %d", maxInt(5, 8));
        NSLog(@"min(5, 8) = %d", minInt(5, 8));
        NSLog(@"clamp(15, 0, 10) = %d", clamp(15, 0, 10));
        NSLog(@"clamp(-5, 0, 10) = %d", clamp(-5, 0, 10));
        NSLog(@"clamp(7, 0, 10) = %d", clamp(7, 0, 10));
        
        NSLog(@"\nlerp(0, 100, 0.25) = %.1f", lerp(0, 100, 0.25));
        NSLog(@"lerp(0, 100, 0.75) = %.1f", lerp(0, 100, 0.75));
        
        NSLog(@"\ninRange(5.0, 0, 10) = %@", inRange(5.0, 0, 10) ? @"YES" : @"NO");
        NSLog(@"inRange(15.0, 0, 10) = %@", inRange(15.0, 0, 10) ? @"YES" : @"NO");
    }
    return 0;
}
```

---

## 6.9 Multiple Return Values (Using Pointers)

Objective-C ไม่รองรับ multiple return values โดยตรง แต่ทำได้ผ่าน pointer parameters

```objc
#import <Foundation/Foundation.h>

// Return multiple values ผ่าน output parameters
void divideWithRemainder(int dividend, int divisor, int *quotient, int *remainder) {
    if (divisor == 0) {
        *quotient = 0;
        *remainder = 0;
        return;
    }
    *quotient = dividend / divisor;
    *remainder = dividend % divisor;
}

// Return statistics หลายค่า
void calculateStats(double *values, int count,
                    double *outMin, double *outMax,
                    double *outMean, double *outStdDev) {
    if (count == 0) return;
    
    *outMin = *outMax = values[0];
    double sum = 0;
    
    for (int i = 0; i < count; i++) {
        if (values[i] < *outMin) *outMin = values[i];
        if (values[i] > *outMax) *outMax = values[i];
        sum += values[i];
    }
    
    *outMean = sum / count;
    
    double variance = 0;
    for (int i = 0; i < count; i++) {
        double diff = values[i] - *outMean;
        variance += diff * diff;
    }
    
    *outStdDev = sqrt(variance / count);
}

// Return point (2D coordinate)
void midpoint(double x1, double y1, double x2, double y2,
              double *mx, double *my) {
    *mx = (x1 + x2) / 2.0;
    *my = (y1 + y2) / 2.0;
}

// ใช้ struct เป็น return type (ดีกว่า multiple pointer params)
typedef struct {
    double x;
    double y;
} Point2D;

Point2D midpointStruct(double x1, double y1, double x2, double y2) {
    return (Point2D){(x1 + x2) / 2.0, (y1 + y2) / 2.0};
}

typedef struct {
    double min;
    double max;
    double mean;
    double stddev;
} Statistics;

Statistics calculateStatsStruct(double *values, int count) {
    Statistics stats = {0, 0, 0, 0};
    if (count == 0) return stats;
    
    stats.min = stats.max = values[0];
    double sum = 0;
    
    for (int i = 0; i < count; i++) {
        if (values[i] < stats.min) stats.min = values[i];
        if (values[i] > stats.max) stats.max = values[i];
        sum += values[i];
    }
    
    stats.mean = sum / count;
    
    double variance = 0;
    for (int i = 0; i < count; i++) {
        double diff = values[i] - stats.mean;
        variance += diff * diff;
    }
    stats.stddev = sqrt(variance / count);
    
    return stats;
}

int main(int argc, const char * argv[]) {
    @autoreleasepool {
        NSLog(@"=== Multiple Return via Pointers ===");
        int q, r;
        divideWithRemainder(17, 5, &q, &r);
        NSLog(@"17 / 5 = %d เศษ %d", q, r);
        
        NSLog(@"\n=== Statistics ===");
        double data[] = {4.0, 7.0, 13.0, 2.0, 9.0, 11.0, 5.0};
        Statistics stats = calculateStatsStruct(data, 7);
        NSLog(@"Min: %.2f", stats.min);
        NSLog(@"Max: %.2f", stats.max);
        NSLog(@"Mean: %.2f", stats.mean);
        NSLog(@"Std Dev: %.2f", stats.stddev);
        
        NSLog(@"\n=== Midpoint ===");
        Point2D mid = midpointStruct(0, 0, 6, 8);
        NSLog(@"จุดกลางระหว่าง (0,0) และ (6,8): (%.1f, %.1f)", mid.x, mid.y);
    }
    return 0;
}
```

---

## 6.10 Function Pointers

Function pointer คือตัวแปรที่เก็บ address ของฟังก์ชัน

```objc
#import <Foundation/Foundation.h>

// ฟังก์ชันที่จะชี้ถึง
int add(int a, int b) { return a + b; }
int subtract(int a, int b) { return a - b; }
int multiply(int a, int b) { return a * b; }
int divide(int a, int b) { return b != 0 ? a / b : 0; }

// ฟังก์ชันที่รับ function pointer เป็น parameter
int applyOperation(int a, int b, int (*operation)(int, int)) {
    return operation(a, b);
}

// ใช้ typedef เพื่อความสะดวก
typedef int (*MathOperation)(int, int);
typedef BOOL (*FilterFunction)(int);

BOOL isEven(int n) { return n % 2 == 0; }
BOOL isPositive(int n) { return n > 0; }
BOOL isLargeNumber(int n) { return n > 100; }

// กรอง array ด้วย function pointer
NSArray* filterArray(NSArray *numbers, FilterFunction filter) {
    NSMutableArray *result = [NSMutableArray array];
    for (NSNumber *num in numbers) {
        if (filter([num intValue])) {
            [result addObject:num];
        }
    }
    return [result copy];
}

// Callback function
typedef void (*CompletionCallback)(BOOL success, NSString *message);

void processData(NSArray *data, CompletionCallback callback) {
    if ([data count] == 0) {
        callback(NO, @"ข้อมูลว่างเปล่า");
        return;
    }
    
    // ประมวลผล...
    NSLog(@"กำลังประมวลผลข้อมูล %lu รายการ...", (unsigned long)[data count]);
    
    callback(YES, @"ประมวลผลสำเร็จ");
}

void onProcessComplete(BOOL success, NSString *message) {
    if (success) {
        NSLog(@"✓ %@", message);
    } else {
        NSLog(@"✗ Error: %@", message);
    }
}

int main(int argc, const char * argv[]) {
    @autoreleasepool {
        NSLog(@"=== Function Pointers พื้นฐาน ===");
        
        // ประกาศ function pointer
        int (*mathOp)(int, int);
        
        // ชี้ไปยังฟังก์ชันต่างๆ
        mathOp = add;
        NSLog(@"add(10, 5) = %d", mathOp(10, 5));
        
        mathOp = subtract;
        NSLog(@"subtract(10, 5) = %d", mathOp(10, 5));
        
        NSLog(@"\n=== Function Pointer เป็น Parameter ===");
        NSLog(@"applyOperation(10, 3, multiply) = %d",
              applyOperation(10, 3, multiply));
        NSLog(@"applyOperation(10, 3, divide) = %d",
              applyOperation(10, 3, divide));
        
        NSLog(@"\n=== Array of Function Pointers ===");
        MathOperation ops[] = {add, subtract, multiply, divide};
        NSString *opNames[] = {@"add", @"subtract", @"multiply", @"divide"};
        int numOps = 4;
        
        for (int i = 0; i < numOps; i++) {
            NSLog(@"%@(8, 4) = %d", opNames[i], ops[i](8, 4));
        }
        
        NSLog(@"\n=== Filter with Function Pointer ===");
        NSArray *numbers = @[@-5, @3, @-2, @150, @8, @-1, @200, @0, @45];
        
        NSLog(@"เลขคู่: %@", filterArray(numbers, isEven));
        NSLog(@"เลขบวก: %@", filterArray(numbers, isPositive));
        NSLog(@"เลขมากกว่า 100: %@", filterArray(numbers, isLargeNumber));
        
        NSLog(@"\n=== Callback Functions ===");
        NSArray *sampleData = @[@1, @2, @3, @4, @5];
        processData(sampleData, onProcessComplete);
        processData(@[], onProcessComplete);
    }
    return 0;
}
```

---

## 6.11 Variadic Functions

Variadic functions รับ arguments จำนวนไม่แน่นอน

```objc
#import <Foundation/Foundation.h>
#import <stdarg.h>

// ฟังก์ชันรับตัวเลขจำนวนไม่แน่นอน
int sumVariadic(int count, ...) {
    va_list args;
    va_start(args, count);
    
    int total = 0;
    for (int i = 0; i < count; i++) {
        total += va_arg(args, int);
    }
    
    va_end(args);
    return total;
}

// หาค่ามากที่สุดจากหลายค่า
int maxVariadic(int count, ...) {
    va_list args;
    va_start(args, count);
    
    int max = va_arg(args, int);
    for (int i = 1; i < count; i++) {
        int val = va_arg(args, int);
        if (val > max) max = val;
    }
    
    va_end(args);
    return max;
}

// Log function ที่ยืดหยุ่น
void customLog(NSString *level, NSString *format, ...) {
    va_list args;
    va_start(args, format);
    
    NSString *message = [[NSString alloc] initWithFormat:format arguments:args];
    
    va_end(args);
    
    NSLog(@"[%@] %@", level, message);
}

// สร้าง NSArray จาก variadic args
NSArray* makeArray(id firstObject, ...) {
    NSMutableArray *array = [NSMutableArray array];
    
    if (firstObject) {
        [array addObject:firstObject];
        
        va_list args;
        va_start(args, firstObject);
        
        id obj;
        while ((obj = va_arg(args, id)) != nil) {
            [array addObject:obj];
        }
        
        va_end(args);
    }
    
    return [array copy];
}

int main(int argc, const char * argv[]) {
    @autoreleasepool {
        NSLog(@"=== Variadic Sum ===");
        NSLog(@"sum(3 ค่า: 1,2,3) = %d", sumVariadic(3, 1, 2, 3));
        NSLog(@"sum(5 ค่า: 10,20,30,40,50) = %d", sumVariadic(5, 10, 20, 30, 40, 50));
        
        NSLog(@"\n=== Variadic Max ===");
        NSLog(@"max(1,5,3,9,2) = %d", maxVariadic(5, 1, 5, 3, 9, 2));
        
        NSLog(@"\n=== Custom Log ===");
        customLog(@"INFO", @"ระบบเริ่มทำงาน");
        customLog(@"WARNING", @"หน่วยความจำเหลือ %d%%", 15);
        customLog(@"ERROR", @"ไม่สามารถเชื่อมต่อ %@ ที่ port %d", @"server", 8080);
        
        NSLog(@"\n=== makeArray ===");
        NSArray *arr = makeArray(@"a", @"b", @"c", @"d", nil);
        NSLog(@"array: %@", arr);
    }
    return 0;
}
```

---

## 6.12 ตัวอย่าง Math Utilities Library

```objc
#import <Foundation/Foundation.h>

// ===== MathUtils.h (สมมติเป็น header) =====

// Constants
#define PI_APPROX 3.14159265358979
#define E_APPROX  2.71828182845905

// Basic math functions
int absolute(int n);
double absoluteDouble(double n);
int gcd(int a, int b);
int lcm(int a, int b);
BOOL isDivisible(int n, int divisor);

// Geometry functions
double circleArea(double radius);
double circlePerimeter(double radius);
double rectangleArea(double width, double height);
double triangleArea(double base, double height);
double triangleAreaByHeron(double a, double b, double c);

// Number theory
BOOL isPerfectSquare(int n);
int digitSum(int n);
int reverseDigits(int n);
BOOL isPalindrome(int n);

// ===== Implementation =====

int absolute(int n) { return n < 0 ? -n : n; }
double absoluteDouble(double n) { return n < 0 ? -n : n; }

int gcd(int a, int b) {
    a = absolute(a);
    b = absolute(b);
    while (b != 0) {
        int t = b;
        b = a % b;
        a = t;
    }
    return a;
}

int lcm(int a, int b) {
    return (a / gcd(a, b)) * b;
}

BOOL isDivisible(int n, int divisor) {
    return divisor != 0 && n % divisor == 0;
}

double circleArea(double radius) {
    return M_PI * radius * radius;
}

double circlePerimeter(double radius) {
    return 2 * M_PI * radius;
}

double rectangleArea(double width, double height) {
    return width * height;
}

double triangleArea(double base, double height) {
    return 0.5 * base * height;
}

double triangleAreaByHeron(double a, double b, double c) {
    double s = (a + b + c) / 2;
    return sqrt(s * (s - a) * (s - b) * (s - c));
}

BOOL isPerfectSquare(int n) {
    if (n < 0) return NO;
    int root = (int)sqrt(n);
    return root * root == n;
}

int digitSum(int n) {
    n = absolute(n);
    int sum = 0;
    while (n > 0) {
        sum += n % 10;
        n /= 10;
    }
    return sum;
}

int reverseDigits(int n) {
    BOOL negative = (n < 0);
    n = absolute(n);
    int reversed = 0;
    while (n > 0) {
        reversed = reversed * 10 + n % 10;
        n /= 10;
    }
    return negative ? -reversed : reversed;
}

BOOL isPalindrome(int n) {
    return n >= 0 && n == reverseDigits(n);
}

int main(int argc, const char * argv[]) {
    @autoreleasepool {
        NSLog(@"=== Math Utilities Demo ===\n");
        
        NSLog(@"--- Basic Math ---");
        NSLog(@"absolute(-7) = %d", absolute(-7));
        NSLog(@"gcd(48, 18) = %d", gcd(48, 18));
        NSLog(@"lcm(4, 6) = %d", lcm(4, 6));
        NSLog(@"isDivisible(15, 3) = %@", isDivisible(15, 3) ? @"YES" : @"NO");
        
        NSLog(@"\n--- Geometry ---");
        NSLog(@"circleArea(5) = %.4f", circleArea(5));
        NSLog(@"circlePerimeter(5) = %.4f", circlePerimeter(5));
        NSLog(@"rectangleArea(6, 4) = %.2f", rectangleArea(6, 4));
        NSLog(@"triangleArea(base=8, h=5) = %.2f", triangleArea(8, 5));
        NSLog(@"triangleAreaByHeron(3,4,5) = %.2f", triangleAreaByHeron(3, 4, 5));
        
        NSLog(@"\n--- Number Theory ---");
        NSLog(@"isPerfectSquare(16) = %@", isPerfectSquare(16) ? @"YES" : @"NO");
        NSLog(@"isPerfectSquare(15) = %@", isPerfectSquare(15) ? @"YES" : @"NO");
        NSLog(@"digitSum(1234) = %d", digitSum(1234));
        NSLog(@"reverseDigits(12345) = %d", reverseDigits(12345));
        NSLog(@"isPalindrome(121) = %@", isPalindrome(121) ? @"YES" : @"NO");
        NSLog(@"isPalindrome(123) = %@", isPalindrome(123) ? @"YES" : @"NO");
    }
    return 0;
}
```

---

## 6.13 ตัวอย่าง String Helper Functions

```objc
#import <Foundation/Foundation.h>

// ===== String Helpers =====

// ตัดช่องว่างหัวท้าย
NSString* trim(NSString *str) {
    return [str stringByTrimmingCharactersInSet:
            [NSCharacterSet whitespaceAndNewlineCharacterSet]];
}

// แปลงเป็น Title Case
NSString* toTitleCase(NSString *str) {
    NSArray *words = [str componentsSeparatedByString:@" "];
    NSMutableArray *capitalized = [NSMutableArray array];
    
    for (NSString *word in words) {
        if ([word length] > 0) {
            NSString *titled = [word stringByReplacingCharactersInRange:NSMakeRange(0, 1)
                                                             withString:[[word substringToIndex:1] uppercaseString]];
            [capitalized addObject:titled];
        }
    }
    
    return [capitalized componentsJoinedByString:@" "];
}

// นับจำนวนคำ
int wordCount(NSString *str) {
    if ([str length] == 0) return 0;
    
    NSArray *words = [[trim(str) componentsSeparatedByCharactersInSet:
                       [NSCharacterSet whitespaceCharacterSet]]
                      filteredArrayUsingPredicate:
                      [NSPredicate predicateWithFormat:@"length > 0"]];
    return (int)[words count];
}

// แทนที่ทุก occurrence
NSString* replaceAll(NSString *str, NSString *target, NSString *replacement) {
    return [str stringByReplacingOccurrencesOfString:target
                                          withString:replacement];
}

// ตรวจสอบว่าเป็นตัวเลขล้วนๆ
BOOL isNumericString(NSString *str) {
    if ([str length] == 0) return NO;
    
    NSCharacterSet *nonDigits = [[NSCharacterSet decimalDigitCharacterSet] invertedSet];
    return [str rangeOfCharacterFromSet:nonDigits].location == NSNotFound;
}

// ย่อยสตริงแบบปลอดภัย
NSString* safeSubstring(NSString *str, NSUInteger start, NSUInteger length) {
    if (str == nil) return @"";
    NSUInteger strLen = [str length];
    if (start >= strLen) return @"";
    
    NSUInteger actualLength = MIN(length, strLen - start);
    return [str substringWithRange:NSMakeRange(start, actualLength)];
}

// Pad string ให้ถึงความยาวที่กำหนด
NSString* padLeft(NSString *str, NSUInteger width, char padChar) {
    NSUInteger strLen = [str length];
    if (strLen >= width) return str;
    
    NSMutableString *padded = [NSMutableString string];
    for (NSUInteger i = 0; i < width - strLen; i++) {
        [padded appendFormat:@"%c", padChar];
    }
    [padded appendString:str];
    return [padded copy];
}

NSString* padRight(NSString *str, NSUInteger width, char padChar) {
    NSUInteger strLen = [str length];
    if (strLen >= width) return str;
    
    NSMutableString *padded = [NSMutableString stringWithString:str];
    for (NSUInteger i = 0; i < width - strLen; i++) {
        [padded appendFormat:@"%c", padChar];
    }
    return [padded copy];
}

int main(int argc, const char * argv[]) {
    @autoreleasepool {
        NSLog(@"=== String Helpers Demo ===\n");
        
        NSString *text = @"  hello world  ";
        NSLog(@"trim('  hello world  ') = '%@'", trim(text));
        
        NSLog(@"toTitleCase('the quick brown fox') = '%@'",
              toTitleCase(@"the quick brown fox"));
        
        NSLog(@"wordCount('hello world foo') = %d",
              wordCount(@"hello world foo"));
        
        NSLog(@"replaceAll = '%@'",
              replaceAll(@"a_b_c_d", @"_", @"-"));
        
        NSLog(@"isNumericString('12345') = %@",
              isNumericString(@"12345") ? @"YES" : @"NO");
        NSLog(@"isNumericString('123a5') = %@",
              isNumericString(@"123a5") ? @"YES" : @"NO");
        
        NSLog(@"safeSubstring('Hello World', 6, 5) = '%@'",
              safeSubstring(@"Hello World", 6, 5));
        
        NSLog(@"padLeft('42', 5, '0') = '%@'",
              padLeft(@"42", 5, '0'));
        NSLog(@"padRight('yes', 10, '.') = '%@'",
              padRight(@"yes", 10, '.'));
    }
    return 0;
}
```

---

## 6.14 แบบฝึกหัดพร้อมเฉลย

### แบบฝึกหัดที่ 1: ฟังก์ชันแปลงหน่วย

**โจทย์:** สร้างฟังก์ชันแปลงหน่วยอุณหภูมิ

```objc
#import <Foundation/Foundation.h>

double celsiusToFahrenheit(double celsius) {
    return celsius * 9.0/5.0 + 32;
}

double fahrenheitToCelsius(double fahrenheit) {
    return (fahrenheit - 32) * 5.0/9.0;
}

double celsiusToKelvin(double celsius) {
    return celsius + 273.15;
}

double kelvinToCelsius(double kelvin) {
    return kelvin - 273.15;
}

int main(int argc, const char * argv[]) {
    @autoreleasepool {
        double c = 100.0;
        NSLog(@"%.1f°C = %.1f°F = %.2fK",
              c,
              celsiusToFahrenheit(c),
              celsiusToKelvin(c));
        
        double f = 212.0;
        NSLog(@"%.1f°F = %.1f°C",
              f, fahrenheitToCelsius(f));
        
        // ทดสอบ round-trip
        double original = 25.0;
        double converted = celsiusToFahrenheit(original);
        double backToC = fahrenheitToCelsius(converted);
        NSLog(@"%.1f°C → %.1f°F → %.1f°C", original, converted, backToC);
    }
    return 0;
}
```

---

### แบบฝึกหัดที่ 2: Sorting Algorithms

**โจทย์:** เขียน Selection Sort และ Insertion Sort

```objc
#import <Foundation/Foundation.h>

void selectionSort(int arr[], int n) {
    for (int i = 0; i < n - 1; i++) {
        int minIdx = i;
        for (int j = i + 1; j < n; j++) {
            if (arr[j] < arr[minIdx]) {
                minIdx = j;
            }
        }
        if (minIdx != i) {
            int temp = arr[i];
            arr[i] = arr[minIdx];
            arr[minIdx] = temp;
        }
    }
}

void insertionSort(int arr[], int n) {
    for (int i = 1; i < n; i++) {
        int key = arr[i];
        int j = i - 1;
        
        while (j >= 0 && arr[j] > key) {
            arr[j + 1] = arr[j];
            j--;
        }
        arr[j + 1] = key;
    }
}

void printArray(int arr[], int n) {
    NSMutableString *s = [NSMutableString stringWithString:@"["];
    for (int i = 0; i < n; i++) {
        [s appendFormat:@"%d%@", arr[i], i < n-1 ? @", " : @""];
    }
    [s appendString:@"]"];
    NSLog(@"%@", s);
}

int main(int argc, const char * argv[]) {
    @autoreleasepool {
        int data1[] = {64, 25, 12, 22, 11};
        int data2[] = {64, 25, 12, 22, 11};
        int n = 5;
        
        NSLog(@"ก่อนเรียง:"); printArray(data1, n);
        selectionSort(data1, n);
        NSLog(@"Selection Sort:"); printArray(data1, n);
        
        NSLog(@"\nก่อนเรียง:"); printArray(data2, n);
        insertionSort(data2, n);
        NSLog(@"Insertion Sort:"); printArray(data2, n);
    }
    return 0;
}
```

---

### แบบฝึกหัดที่ 3: String Utilities

**โจทย์:** เขียนฟังก์ชัน reverse string และ check palindrome

```objc
#import <Foundation/Foundation.h>

NSString* reverseString(NSString *str) {
    NSMutableString *reversed = [NSMutableString string];
    NSInteger len = [str length];
    for (NSInteger i = len - 1; i >= 0; i--) {
        [reversed appendFormat:@"%c", [str characterAtIndex:i]];
    }
    return [reversed copy];
}

BOOL isPalindromeString(NSString *str) {
    NSString *cleaned = [[str lowercaseString] 
                         stringByTrimmingCharactersInSet:
                         [NSCharacterSet whitespaceCharacterSet]];
    return [cleaned isEqualToString:reverseString(cleaned)];
}

int main(int argc, const char * argv[]) {
    @autoreleasepool {
        NSArray *testStrings = @[@"racecar", @"hello", @"madam", @"level", @"world"];
        
        for (NSString *s in testStrings) {
            NSLog(@"'%@' → reverse='%@', palindrome=%@",
                  s, reverseString(s),
                  isPalindromeString(s) ? @"YES" : @"NO");
        }
    }
    return 0;
}
```

---

### แบบฝึกหัดที่ 4: Function Composition

**โจทย์:** สร้างระบบ pipeline ที่ใช้ function pointers

```objc
#import <Foundation/Foundation.h>

typedef double (*Transform)(double);

double double_it(double x) { return x * 2; }
double add_ten(double x) { return x + 10; }
double square_it(double x) { return x * x; }
double negate_it(double x) { return -x; }

double applyPipeline(double input, Transform *transforms, int count) {
    double result = input;
    for (int i = 0; i < count; i++) {
        result = transforms[i](result);
    }
    return result;
}

int main(int argc, const char * argv[]) {
    @autoreleasepool {
        Transform pipeline[] = {double_it, add_ten, square_it};
        int pipelineSize = 3;
        
        double input = 3.0;
        NSLog(@"input = %.1f", input);
        NSLog(@"double_it → add_ten → square_it");
        NSLog(@"result = %.1f", applyPipeline(input, pipeline, pipelineSize));
        // (3 * 2 + 10)^2 = 16^2 = 256
        
        Transform pipeline2[] = {add_ten, double_it, negate_it};
        NSLog(@"\nadd_ten → double_it → negate_it");
        NSLog(@"result = %.1f", applyPipeline(input, pipeline2, 3));
        // -((3+10)*2) = -26
    }
    return 0;
}
```

---

### แบบฝึกหัดที่ 5: Recursive สร้าง Permutations

**โจทย์:** หา permutations ทั้งหมดของ string

```objc
#import <Foundation/Foundation.h>

void swap(char *a, char *b) {
    char t = *a; *a = *b; *b = t;
}

int permCount = 0;

void generatePermutations(char *str, int start, int end) {
    if (start == end) {
        permCount++;
        NSLog(@"%d: %s", permCount, str);
        return;
    }
    
    for (int i = start; i <= end; i++) {
        swap(&str[start], &str[i]);
        generatePermutations(str, start + 1, end);
        swap(&str[start], &str[i]); // backtrack
    }
}

int main(int argc, const char * argv[]) {
    @autoreleasepool {
        char str[] = "ABC";
        int n = strlen(str);
        
        permCount = 0;
        NSLog(@"=== Permutations of '%s' ===", str);
        generatePermutations(str, 0, n - 1);
        NSLog(@"รวม %d permutations", permCount);
    }
    return 0;
}
```

---

### แบบฝึกหัดที่ 6: Memoization ด้วย NSDictionary

**โจทย์:** ใช้ NSDictionary cache ผลลัพธ์ Fibonacci

```objc
#import <Foundation/Foundation.h>

NSMutableDictionary *fibCache;

long fibMemo(int n) {
    if (n <= 1) return n;
    
    NSNumber *key = @(n);
    NSNumber *cached = fibCache[key];
    
    if (cached) {
        return [cached longValue];
    }
    
    long result = fibMemo(n - 1) + fibMemo(n - 2);
    fibCache[key] = @(result);
    return result;
}

int main(int argc, const char * argv[]) {
    @autoreleasepool {
        fibCache = [NSMutableDictionary dictionary];
        
        NSLog(@"=== Fibonacci ด้วย Memoization ===");
        for (int i = 0; i <= 30; i++) {
            NSLog(@"F(%2d) = %ld", i, fibMemo(i));
        }
        
        NSLog(@"\nขนาด Cache: %lu entries", (unsigned long)[fibCache count]);
    }
    return 0;
}
```

---

### แบบฝึกหัดที่ 7: Calculator ด้วย Function Pointers

**โจทย์:** สร้าง scientific calculator ด้วย function pointers

```objc
#import <Foundation/Foundation.h>

typedef double (*UnaryOp)(double);
typedef double (*BinaryOp)(double, double);

double op_sin(double x) { return sin(x * M_PI / 180); }
double op_cos(double x) { return cos(x * M_PI / 180); }
double op_tan(double x) { return tan(x * M_PI / 180); }
double op_sqrt(double x) { return sqrt(x); }
double op_log(double x) { return log10(x); }
double op_ln(double x) { return log(x); }
double op_abs(double x) { return fabs(x); }

double op_power(double base, double exp) { return pow(base, exp); }
double op_mod(double a, double b) { return fmod(a, b); }
double op_hypot(double a, double b) { return hypot(a, b); }

typedef struct {
    const char *name;
    UnaryOp func;
} UnaryFunction;

int main(int argc, const char * argv[]) {
    @autoreleasepool {
        UnaryFunction unaryFuncs[] = {
            {"sin", op_sin},
            {"cos", op_cos},
            {"tan", op_tan},
            {"sqrt", op_sqrt},
            {"log", op_log},
            {"abs", op_abs}
        };
        int numFuncs = 6;
        
        double testValues[] = {30, 45, 60, 90};
        
        NSLog(@"=== Scientific Calculator ===");
        NSLog(@"%10s | %8s | %8s | %8s | %8s",
              "Function", "30°", "45°", "60°", "90°");
        NSLog(@"------------------------------------------------------");
        
        for (int i = 0; i < numFuncs; i++) {
            NSMutableString *row = [NSMutableString stringWithFormat:@"%10s", unaryFuncs[i].name];
            for (int j = 0; j < 4; j++) {
                [row appendFormat:@" | %8.4f", unaryFuncs[i].func(testValues[j])];
            }
            NSLog(@"%@", row);
        }
        
        NSLog(@"\n--- Binary Operations ---");
        NSLog(@"pow(2, 10) = %.0f", op_power(2, 10));
        NSLog(@"mod(17, 5) = %.0f", op_mod(17, 5));
        NSLog(@"hypot(3, 4) = %.2f", op_hypot(3, 4));
    }
    return 0;
}
```

---

### แบบฝึกหัดที่ 8: Merge Sort

**โจทย์:** Implement Merge Sort ด้วย recursive function

```objc
#import <Foundation/Foundation.h>

void merge(int arr[], int left, int mid, int right) {
    int n1 = mid - left + 1;
    int n2 = right - mid;
    
    int L[n1], R[n2];
    
    for (int i = 0; i < n1; i++) L[i] = arr[left + i];
    for (int j = 0; j < n2; j++) R[j] = arr[mid + 1 + j];
    
    int i = 0, j = 0, k = left;
    
    while (i < n1 && j < n2) {
        if (L[i] <= R[j]) {
            arr[k++] = L[i++];
        } else {
            arr[k++] = R[j++];
        }
    }
    
    while (i < n1) arr[k++] = L[i++];
    while (j < n2) arr[k++] = R[j++];
}

void mergeSort(int arr[], int left, int right) {
    if (left < right) {
        int mid = left + (right - left) / 2;
        mergeSort(arr, left, mid);
        mergeSort(arr, mid + 1, right);
        merge(arr, left, mid, right);
    }
}

int main(int argc, const char * argv[]) {
    @autoreleasepool {
        int arr[] = {38, 27, 43, 3, 9, 82, 10, 56, 71, 15};
        int n = sizeof(arr) / sizeof(arr[0]);
        
        NSLog(@"ก่อนเรียง:");
        NSMutableString *before = [NSMutableString stringWithString:@"["];
        for (int i = 0; i < n; i++) {
            [before appendFormat:@"%d%@", arr[i], i < n-1 ? @", " : @""];
        }
        [before appendString:@"]"];
        NSLog(@"%@", before);
        
        mergeSort(arr, 0, n - 1);
        
        NSLog(@"หลังเรียง (Merge Sort):");
        NSMutableString *after = [NSMutableString stringWithString:@"["];
        for (int i = 0; i < n; i++) {
            [after appendFormat:@"%d%@", arr[i], i < n-1 ? @", " : @""];
        }
        [after appendString:@"]"];
        NSLog(@"%@", after);
    }
    return 0;
}
```

---

### แบบฝึกหัดที่ 9: ฟังก์ชัน Validation

**โจทย์:** สร้าง validation functions สำหรับข้อมูลต่างๆ

```objc
#import <Foundation/Foundation.h>

typedef struct {
    BOOL valid;
    NSString *errorMessage;
} ValidationResult;

ValidationResult validateEmail(NSString *email) {
    if (!email || [email length] == 0)
        return (ValidationResult){NO, @"อีเมลต้องไม่ว่าง"};
    
    if ([email rangeOfString:@"@"].location == NSNotFound)
        return (ValidationResult){NO, @"อีเมลต้องมี @"};
    
    NSArray *parts = [email componentsSeparatedByString:@"@"];
    if ([parts count] != 2)
        return (ValidationResult){NO, @"รูปแบบอีเมลไม่ถูกต้อง"};
    
    if ([parts[0] length] == 0)
        return (ValidationResult){NO, @"ต้องมีชื่อก่อน @"};
    
    NSString *domain = parts[1];
    if ([domain rangeOfString:@"."].location == NSNotFound)
        return (ValidationResult){NO, @"domain ต้องมี ."};
    
    return (ValidationResult){YES, nil};
}

ValidationResult validateThaiID(NSString *idNumber) {
    if ([idNumber length] != 13)
        return (ValidationResult){NO, @"เลขบัตรต้องมี 13 หลัก"};
    
    // ตรวจสอบว่าเป็นตัวเลขทั้งหมด
    NSCharacterSet *nonDigits = [[NSCharacterSet decimalDigitCharacterSet] invertedSet];
    if ([idNumber rangeOfCharacterFromSet:nonDigits].location != NSNotFound)
        return (ValidationResult){NO, @"เลขบัตรต้องเป็นตัวเลขเท่านั้น"};
    
    // ตรวจสอบ checksum ของบัตรประชาชนไทย
    int sum = 0;
    for (int i = 0; i < 12; i++) {
        sum += [[idNumber substringWithRange:NSMakeRange(i, 1)] intValue] * (13 - i);
    }
    int checkDigit = (11 - (sum % 11)) % 10;
    int lastDigit = [[idNumber substringFromIndex:12] intValue];
    
    if (checkDigit != lastDigit)
        return (ValidationResult){NO, @"เลขบัตรไม่ถูกต้อง (checksum ผิด)"};
    
    return (ValidationResult){YES, nil};
}

int main(int argc, const char * argv[]) {
    @autoreleasepool {
        NSLog(@"=== Email Validation ===");
        NSArray *emails = @[
            @"user@example.com",
            @"invalid-email",
            @"@nodomain.com",
            @"user@",
            @"test@test.co.th"
        ];
        
        for (NSString *email in emails) {
            ValidationResult result = validateEmail(email);
            if (result.valid) {
                NSLog(@"✓ '%@' - ถูกต้อง", email);
            } else {
                NSLog(@"✗ '%@' - %@", email, result.errorMessage);
            }
        }
        
        NSLog(@"\n=== Thai ID Validation ===");
        // หมายเหตุ: ตัวอย่างเลขสมมติ
        NSArray *ids = @[@"1234567890123", @"1234", @"12345678901a3"];
        
        for (NSString *id in ids) {
            ValidationResult result = validateThaiID(id);
            if (result.valid) {
                NSLog(@"✓ '%@' - ถูกต้อง", id);
            } else {
                NSLog(@"✗ '%@' - %@", id, result.errorMessage);
            }
        }
    }
    return 0;
}
```

---

### แบบฝึกหัดที่ 10: Higher-Order Functions

**โจทย์:** สร้าง map, filter, reduce ด้วย function pointers

```objc
#import <Foundation/Foundation.h>

typedef int (*MapFunction)(int);
typedef BOOL (*FilterFunction)(int);
typedef int (*ReduceFunction)(int, int);

// Map: แปลงทุก element
NSArray* mapArray(NSArray *arr, MapFunction transform) {
    NSMutableArray *result = [NSMutableArray arrayWithCapacity:[arr count]];
    for (NSNumber *n in arr) {
        [result addObject:@(transform([n intValue]))];
    }
    return [result copy];
}

// Filter: คัดเลือก element
NSArray* filterArray(NSArray *arr, FilterFunction predicate) {
    NSMutableArray *result = [NSMutableArray array];
    for (NSNumber *n in arr) {
        if (predicate([n intValue])) {
            [result addObject:n];
        }
    }
    return [result copy];
}

// Reduce: รวม elements เป็นค่าเดียว
int reduceArray(NSArray *arr, int initialValue, ReduceFunction combine) {
    int accumulator = initialValue;
    for (NSNumber *n in arr) {
        accumulator = combine(accumulator, [n intValue]);
    }
    return accumulator;
}

// ฟังก์ชันสำหรับ map
int double_value(int x) { return x * 2; }
int square_value(int x) { return x * x; }
int add_one(int x) { return x + 1; }

// ฟังก์ชันสำหรับ filter
BOOL is_even(int x) { return x % 2 == 0; }
BOOL is_positive(int x) { return x > 0; }
BOOL is_gt_10(int x) { return x > 10; }

// ฟังก์ชันสำหรับ reduce
int sum_two(int a, int b) { return a + b; }
int multiply_two(int a, int b) { return a * b; }
int max_two(int a, int b) { return a > b ? a : b; }

int main(int argc, const char * argv[]) {
    @autoreleasepool {
        NSArray *numbers = @[@1, @2, @3, @4, @5, @6, @7, @8, @9, @10];
        NSLog(@"ข้อมูลต้นฉบับ: %@", numbers);
        
        NSLog(@"\n=== Map ===");
        NSLog(@"map(double_value): %@", mapArray(numbers, double_value));
        NSLog(@"map(square_value): %@", mapArray(numbers, square_value));
        NSLog(@"map(add_one): %@", mapArray(numbers, add_one));
        
        NSLog(@"\n=== Filter ===");
        NSLog(@"filter(is_even): %@", filterArray(numbers, is_even));
        NSLog(@"filter(is_gt_10 after map): %@",
              filterArray(mapArray(numbers, double_value), is_gt_10));
        
        NSLog(@"\n=== Reduce ===");
        NSLog(@"reduce(sum, 0) = %d", reduceArray(numbers, 0, sum_two));
        NSLog(@"reduce(product, 1) = %d", reduceArray(numbers, 1, multiply_two));
        NSLog(@"reduce(max, 0) = %d", reduceArray(numbers, 0, max_two));
        
        NSLog(@"\n=== Pipeline: filter even → square → sum ===");
        NSArray *evens = filterArray(numbers, is_even);
        NSArray *squared = mapArray(evens, square_value);
        int result = reduceArray(squared, 0, sum_two);
        NSLog(@"Evens: %@", evens);
        NSLog(@"Squared: %@", squared);
        NSLog(@"Sum: %d", result);
        // 2^2 + 4^2 + 6^2 + 8^2 + 10^2 = 4+16+36+64+100 = 220
    }
    return 0;
}
```

---

## สรุปบทที่ 6

ในบทนี้เราได้เรียนรู้:

1. **Function declaration & definition** - การประกาศและนิยามฟังก์ชัน
2. **Return types & parameters** - ประเภทค่าที่คืนและรับ
3. **void functions** - ฟังก์ชันที่ไม่คืนค่า
4. **Function prototypes** - การประกาศฟังก์ชันล่วงหน้า
5. **Pass by value vs pointer** - ความแตกต่างของการส่งค่า
6. **Recursive functions** - ฟังก์ชันที่เรียกตัวเอง
7. **Static functions** - ฟังก์ชันที่มองเห็นเฉพาะใน file
8. **Inline functions** - การแนะนำ compiler ให้ inline
9. **Multiple return values** - คืนหลายค่าผ่าน pointer หรือ struct
10. **Function pointers** - ตัวแปรที่ชี้ไปยังฟังก์ชัน
11. **Variadic functions** - ฟังก์ชันที่รับ arguments จำนวนไม่แน่นอน

### เคล็ดลับสำคัญ

- ฟังก์ชันควรทำงานเพียงอย่างเดียว (Single Responsibility)
- ใช้ **Guard clauses** เพื่อ validate input ก่อนทำงาน
- **static** ฟังก์ชันช่วยซ่อน implementation details
- **Function pointers** ทำให้โค้ดยืดหยุ่นและ testable
- ใช้ **struct** แทนการ return หลายค่าผ่าน pointer หลายตัว
- **Recursive** functions ต้องมี base case เสมอ

---

*บทต่อไป: ส่วนที่ 07 - Arrays และ Pointers*
