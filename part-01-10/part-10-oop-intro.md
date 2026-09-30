# ตอนที่ 10: บทนำสู่การเขียนโปรแกรมเชิงวัตถุ (OOP)

## บทนำ

Object-Oriented Programming (OOP) หรือการเขียนโปรแกรมเชิงวัตถุ เป็นแนวคิดการเขียนโปรแกรมที่จัดรหัสโดยใช้ "objects" เป็นศูนย์กลาง Objective-C เป็นภาษาที่สนับสนุน OOP อย่างสมบูรณ์ บทนี้จะแนะนำแนวคิด OOP และวิธีใช้งานใน Objective-C

---

## 10.1 แนวคิด OOP

### OOP คืออะไร?

```
Procedural Programming (แบบเดิม):
- โปรแกรม = ชุดคำสั่งที่รันตามลำดับ
- ข้อมูลและฟังก์ชันแยกกัน
- ยากต่อการ reuse และ maintain

Object-Oriented Programming:
- โปรแกรม = กลุ่มของ objects ที่ทำงานร่วมกัน
- ข้อมูล (data) และพฤติกรรม (behavior) อยู่ด้วยกัน
- ง่ายต่อการ reuse, maintain และ extend
```

### 4 หลักการ OOP

```
1. Encapsulation (การห่อหุ้ม):
   - ซ่อน implementation details ไว้ภายใน
   - เปิดเผยเฉพาะ interface ที่จำเป็น
   - ตัวอย่าง: bank account ซ่อน balance, expose deposit/withdraw

2. Inheritance (การสืบทอด):
   - Class ลูกได้รับ properties/methods จาก class แม่
   - ลดการ duplicate code
   - ตัวอย่าง: Cat, Dog สืบทอดจาก Animal

3. Polymorphism (ความหลากหลายรูปแบบ):
   - Object ต่างชนิดตอบสนองต่อ message เดียวกันต่างกัน
   - ตัวอย่าง: animal.makeSound() แตกต่างกันใน Cat, Dog

4. Abstraction (การนามธรรม):
   - ซ่อนความซับซ้อน แสดงเฉพาะสิ่งจำเป็น
   - ตัวอย่าง: คุณขับรถโดยไม่รู้ engine internals
```

```objc
#import <Foundation/Foundation.h>

// ตัวอย่าง OOP concept ง่ายๆ

// Procedural approach (แบบเดิม)
void printEmployeeInfo_procedural(NSString *name, int age, double salary) {
    NSLog(@"Employee: %@, Age: %d, Salary: %.2f", name, age, salary);
}

double calculateRaise_procedural(double salary, double percent) {
    return salary * (1 + percent / 100);
}

// OOP approach
// (ดูตัวอย่างเต็มในหัวข้อถัดไป)

int main(int argc, const char * argv[]) {
    @autoreleasepool {
        
        NSLog(@"=== OOP vs Procedural ===");
        
        // Procedural
        NSLog(@"\nProcedural approach:");
        printEmployeeInfo_procedural(@"สมชาย", 25, 30000.0);
        double newSalary = calculateRaise_procedural(30000.0, 10);
        NSLog(@"New salary: %.2f", newSalary);
        
        // OOP (จะแสดงในหัวข้อถัดไป)
        NSLog(@"\nOOP approach จะแสดงด้านล่าง...");
    }
    return 0;
}
```

---

## 10.2 Classes vs Objects

### ความแตกต่าง

```
Class = แบบพิมพ์เขียว (blueprint)
Object = สิ่งที่สร้างจาก class (instance)

เหมือน:
- Class 'Car' = แบบรถ (พิมพ์เขียว)
- Object = รถจริงๆ ที่สร้างขึ้น (Toyota Corolla, Honda Civic, ...)

- Class 'Human' = แนวคิด "มนุษย์"
- Object = คนจริงๆ (สมชาย, มณี, ...)
```

```objc
#import <Foundation/Foundation.h>

// Class definition
@interface Circle : NSObject

// Properties (ข้อมูล)
@property (nonatomic, assign) double radius;
@property (nonatomic, assign) NSString *color;

// Methods (พฤติกรรม)
- (double)area;
- (double)circumference;
- (void)describe;

// Class method (factory)
+ (instancetype)circleWithRadius:(double)radius;

@end

@implementation Circle

- (double)area {
    return M_PI * _radius * _radius;
}

- (double)circumference {
    return 2 * M_PI * _radius;
}

- (void)describe {
    NSLog(@"วงกลม: รัศมี=%.2f, พื้นที่=%.2f, เส้นรอบวง=%.2f",
          _radius, [self area], [self circumference]);
}

+ (instancetype)circleWithRadius:(double)radius {
    Circle *c = [[Circle alloc] init];
    c.radius = radius;
    return c;
}

@end

int main(int argc, const char * argv[]) {
    @autoreleasepool {
        
        NSLog(@"=== Classes vs Objects ===");
        
        // สร้าง objects หลาย instance จาก class เดียวกัน
        Circle *c1 = [Circle circleWithRadius:5.0];
        Circle *c2 = [Circle circleWithRadius:10.0];
        Circle *c3 = [[Circle alloc] init];
        c3.radius = 3.0;
        
        NSLog(@"\nc1:");
        [c1 describe];
        
        NSLog(@"\nc2:");
        [c2 describe];
        
        NSLog(@"\nc3:");
        [c3 describe];
        
        // Objects ต่างกันโดยสิ้นเชิง
        NSLog(@"\nc1 radius: %.2f", c1.radius);
        NSLog(@"c2 radius: %.2f", c2.radius);
        NSLog(@"เป็น object เดียวกัน: %@", (c1 == c2) ? @"YES" : @"NO");
        
        // ตรวจสอบ class
        NSLog(@"\nc1 class: %@", [c1 class]);
        NSLog(@"c1 isKindOfClass Circle: %@",
              [c1 isKindOfClass:[Circle class]] ? @"YES" : @"NO");
        NSLog(@"c1 isKindOfClass NSObject: %@",
              [c1 isKindOfClass:[NSObject class]] ? @"YES" : @"NO");
    }
    return 0;
}
```

---

## 10.3 @interface และ @implementation

### โครงสร้างพื้นฐาน

```objc
// ===== Header File: Person.h =====
#import <Foundation/Foundation.h>

@interface Person : NSObject  // Person สืบทอดจาก NSObject

// Instance variables (ไม่แนะนำในปัจจุบัน ใช้ property แทน)
// {
//     NSString *_name;
//     int _age;
// }

// Properties
@property (nonatomic, strong) NSString *name;
@property (nonatomic, assign) NSInteger age;
@property (nonatomic, strong) NSString *email;

// Instance methods (-)
- (instancetype)initWithName:(NSString *)name age:(NSInteger)age;
- (void)greet;
- (BOOL)isAdult;
- (NSString *)description;

// Class methods (+)
+ (instancetype)personWithName:(NSString *)name age:(NSInteger)age;
+ (NSString *)defaultGreeting;

@end


// ===== Implementation File: Person.m =====

@implementation Person

// Class method: factory method
+ (instancetype)personWithName:(NSString *)name age:(NSInteger)age {
    return [[self alloc] initWithName:name age:age];
}

+ (NSString *)defaultGreeting {
    return @"สวัสดี";
}

// Designated initializer
- (instancetype)initWithName:(NSString *)name age:(NSInteger)age {
    self = [super init];  // เรียก super init
    if (self) {
        _name = name;
        _age = age;
    }
    return self;
}

// Convenience initializer
- (instancetype)init {
    return [self initWithName:@"Unknown" age:0];
}

- (void)greet {
    NSLog(@"%@, ผมชื่อ %@, อายุ %ld ปี",
          [[self class] defaultGreeting],
          _name, (long)_age);
}

- (BOOL)isAdult {
    return _age >= 18;
}

// Override description (เหมือน toString())
- (NSString *)description {
    return [NSString stringWithFormat:@"Person{name='%@', age=%ld}",
            _name, (long)_age];
}

@end
```

### ตัวอย่างการใช้งาน

```objc
#import <Foundation/Foundation.h>

@interface Person : NSObject
@property (nonatomic, strong) NSString *name;
@property (nonatomic, assign) NSInteger age;
@property (nonatomic, strong) NSString *email;
- (instancetype)initWithName:(NSString *)name age:(NSInteger)age;
- (void)greet;
- (BOOL)isAdult;
@end

@implementation Person
+ (instancetype)personWithName:(NSString *)name age:(NSInteger)age {
    return [[self alloc] initWithName:name age:age];
}
- (instancetype)initWithName:(NSString *)name age:(NSInteger)age {
    self = [super init];
    if (self) { _name = name; _age = age; }
    return self;
}
- (void)greet {
    NSLog(@"สวัสดี ฉันชื่อ %@", _name);
}
- (BOOL)isAdult { return _age >= 18; }
- (NSString *)description {
    return [NSString stringWithFormat:@"Person{%@, %ld}", _name, (long)_age];
}
@end

int main(int argc, const char * argv[]) {
    @autoreleasepool {
        
        NSLog(@"=== @interface & @implementation ===");
        
        // สร้าง Person หลายวิธี
        Person *p1 = [[Person alloc] initWithName:@"สมชาย" age:25];
        Person *p2 = [Person personWithName:@"มณี" age:17];
        
        // ใช้ methods
        [p1 greet];
        [p2 greet];
        
        // Properties
        NSLog(@"\n%@ เป็นผู้ใหญ่: %@",
              p1.name, [p1 isAdult] ? @"ใช่" : @"ไม่ใช่");
        NSLog(@"%@ เป็นผู้ใหญ่: %@",
              p2.name, [p2 isAdult] ? @"ใช่" : @"ไม่ใช่");
        
        // description
        NSLog(@"\n%@", p1);  // เรียก description อัตโนมัติ
        NSLog(@"%@", p2);
        
        // แก้ไข property
        p1.age = 26;
        p1.email = @"somchai@example.com";
        NSLog(@"\nอีเมล: %@", p1.email);
        
        // Array ของ Person
        NSArray *people = @[p1, p2,
                            [Person personWithName:@"อนุชา" age:30]];
        
        NSLog(@"\n=== รายชื่อทั้งหมด ===");
        for (Person *p in people) {
            NSLog(@"%@ (ผู้ใหญ่: %@)",
                  p, [p isAdult] ? @"ใช่" : @"ไม่ใช่");
        }
    }
    return 0;
}
```

---

## 10.4 Instance Variables และ Properties

```objc
#import <Foundation/Foundation.h>

@interface BankAccount : NSObject {
    // Instance variables (private โดย default)
    double _balance;         // underscore prefix เป็น convention
    NSString *_accountNumber;
    NSMutableArray *_transactions;
}

// Properties (สร้าง getter/setter อัตโนมัติ)
@property (nonatomic, readonly) double balance;         // readonly
@property (nonatomic, copy) NSString *ownerName;
@property (nonatomic, readonly) NSString *accountNumber;

// Methods
- (instancetype)initWithOwner:(NSString *)name initialBalance:(double)balance;
- (BOOL)deposit:(double)amount;
- (BOOL)withdraw:(double)amount;
- (void)printStatement;

@end

@implementation BankAccount

// Synthesize สร้าง getter/setter ให้
// @synthesize balance = _balance;  // ปัจจุบัน auto-synthesize

- (instancetype)initWithOwner:(NSString *)name initialBalance:(double)balance {
    self = [super init];
    if (self) {
        _ownerName = [name copy];
        _balance = balance;
        _accountNumber = [NSString stringWithFormat:@"ACC%06d", arc4random_uniform(999999)];
        _transactions = [NSMutableArray array];
        
        if (balance > 0) {
            [_transactions addObject:@{
                @"type": @"INIT",
                @"amount": @(balance),
                @"balance": @(balance)
            }];
        }
    }
    return self;
}

- (BOOL)deposit:(double)amount {
    if (amount <= 0) {
        NSLog(@"จำนวนเงินต้องมากกว่า 0");
        return NO;
    }
    
    _balance += amount;
    [_transactions addObject:@{
        @"type": @"DEPOSIT",
        @"amount": @(amount),
        @"balance": @(_balance)
    }];
    return YES;
}

- (BOOL)withdraw:(double)amount {
    if (amount <= 0) {
        NSLog(@"จำนวนเงินต้องมากกว่า 0");
        return NO;
    }
    if (amount > _balance) {
        NSLog(@"ยอดเงินไม่พอ (คงเหลือ: %.2f)", _balance);
        return NO;
    }
    
    _balance -= amount;
    [_transactions addObject:@{
        @"type": @"WITHDRAW",
        @"amount": @(amount),
        @"balance": @(_balance)
    }];
    return YES;
}

- (void)printStatement {
    NSLog(@"\n=== Statement ===");
    NSLog(@"บัญชีเลขที่: %@", _accountNumber);
    NSLog(@"ชื่อ: %@", _ownerName);
    NSLog(@"ธุรกรรม:");
    
    for (NSDictionary *tx in _transactions) {
        NSLog(@"  [%@] ฿%.2f (คงเหลือ: ฿%.2f)",
              tx[@"type"],
              [tx[@"amount"] doubleValue],
              [tx[@"balance"] doubleValue]);
    }
    NSLog(@"ยอดคงเหลือ: ฿%.2f", _balance);
    NSLog(@"=================");
}

@end

int main(int argc, const char * argv[]) {
    @autoreleasepool {
        
        NSLog(@"=== Bank Account (Encapsulation) ===");
        
        BankAccount *account = [[BankAccount alloc] initWithOwner:@"สมชาย ใจดี"
                                                    initialBalance:10000];
        
        [account deposit:5000];
        [account withdraw:2000];
        [account deposit:3000];
        [account withdraw:15000];  // ยอดไม่พอ
        [account withdraw:8000];
        
        [account printStatement];
        
        // balance เป็น readonly - อ่านได้ แต่แก้ไขตรงๆ ไม่ได้
        NSLog(@"\nคงเหลือ: ฿%.2f", account.balance);
        // account.balance = 999999;  // Error! readonly property
    }
    return 0;
}
```

---

## 10.5 Methods - Instance และ Class Methods

```objc
#import <Foundation/Foundation.h>

@interface Calculator : NSObject

@property (nonatomic, assign) double result;
@property (nonatomic, strong) NSMutableArray *history;

// Class methods (factory & utilities)
+ (instancetype)calculator;
+ (double)add:(double)a to:(double)b;
+ (double)factorial:(NSInteger)n;

// Instance methods (operations)
- (Calculator *)add:(double)value;
- (Calculator *)subtract:(double)value;
- (Calculator *)multiply:(double)value;
- (Calculator *)divide:(double)value;
- (void)reset;
- (void)printHistory;

@end

@implementation Calculator

+ (instancetype)calculator {
    return [[self alloc] init];
}

+ (double)add:(double)a to:(double)b {
    return a + b;
}

+ (double)factorial:(NSInteger)n {
    if (n <= 1) return 1;
    return n * [Calculator factorial:n - 1];
}

- (instancetype)init {
    self = [super init];
    if (self) {
        _result = 0;
        _history = [NSMutableArray array];
        [_history addObject:@"เริ่มต้น: 0"];
    }
    return self;
}

// Method chaining pattern - return self
- (Calculator *)add:(double)value {
    _result += value;
    [_history addObject:[NSString stringWithFormat:@"+ %.2f = %.2f", value, _result]];
    return self;
}

- (Calculator *)subtract:(double)value {
    _result -= value;
    [_history addObject:[NSString stringWithFormat:@"- %.2f = %.2f", value, _result]];
    return self;
}

- (Calculator *)multiply:(double)value {
    _result *= value;
    [_history addObject:[NSString stringWithFormat:@"× %.2f = %.2f", value, _result]];
    return self;
}

- (Calculator *)divide:(double)value {
    if (value == 0) {
        NSLog(@"ไม่สามารถหารด้วย 0 ได้");
        return self;
    }
    _result /= value;
    [_history addObject:[NSString stringWithFormat:@"÷ %.2f = %.2f", value, _result]];
    return self;
}

- (void)reset {
    _result = 0;
    [_history addObject:@"รีเซ็ต: 0"];
}

- (void)printHistory {
    NSLog(@"\n=== ประวัติการคำนวณ ===");
    for (NSString *entry in _history) {
        NSLog(@"  %@", entry);
    }
}

@end

int main(int argc, const char * argv[]) {
    @autoreleasepool {
        
        NSLog(@"=== Calculator - Instance & Class Methods ===");
        
        // Class methods (ไม่ต้องสร้าง instance)
        NSLog(@"\n=== Class Methods ===");
        double sum = [Calculator add:10.5 to:20.3];
        NSLog(@"10.5 + 20.3 = %.2f", sum);
        
        NSLog(@"5! = %.0f", [Calculator factorial:5]);
        NSLog(@"10! = %.0f", [Calculator factorial:10]);
        
        // Instance methods (ต้องสร้าง instance)
        NSLog(@"\n=== Instance Methods ===");
        Calculator *calc = [Calculator calculator];
        
        // Method chaining
        [[[[calc add:100] add:50] subtract:30] multiply:2];
        NSLog(@"ผลลัพธ์: %.2f", calc.result);
        
        [calc printHistory];
        
        // อีก instance
        Calculator *calc2 = [Calculator calculator];
        [[[calc2 add:1000] divide:4] subtract:100];
        NSLog(@"\nCalc2 result: %.2f", calc2.result);
    }
    return 0;
}
```

---

## 10.6 self Keyword

```objc
#import <Foundation/Foundation.h>

@interface Rectangle : NSObject

@property (nonatomic, assign) double width;
@property (nonatomic, assign) double height;

- (instancetype)initWithWidth:(double)width height:(double)height;
- (double)area;
- (double)perimeter;
- (BOOL)isSquare;
- (Rectangle *)scaled:(double)factor;
- (void)describe;

@end

@implementation Rectangle

- (instancetype)initWithWidth:(double)width height:(double)height {
    self = [super init];  // self คือ instance ที่กำลังสร้าง
    if (self) {
        // self.width เรียก setter (ผ่าน property)
        self.width = width;
        self.height = height;
        // หรือ access ivar โดยตรง: _width = width;
    }
    return self;
}

- (double)area {
    return self.width * self.height;  // self = ตัวเอง
}

- (double)perimeter {
    return 2 * (self.width + self.height);
}

- (BOOL)isSquare {
    return self.width == self.height;
}

- (Rectangle *)scaled:(double)factor {
    // สร้าง Rectangle ใหม่ที่ scale แล้ว
    return [[Rectangle alloc] initWithWidth:self.width * factor
                                     height:self.height * factor];
}

- (void)describe {
    // เรียก method อื่นบน self
    NSLog(@"Rectangle: %.2f × %.2f, area=%.2f, perimeter=%.2f, square=%@",
          self.width, self.height,
          [self area],
          [self perimeter],
          [self isSquare] ? @"YES" : @"NO");
}

@end

int main(int argc, const char * argv[]) {
    @autoreleasepool {
        
        NSLog(@"=== self Keyword ===");
        
        Rectangle *r1 = [[Rectangle alloc] initWithWidth:5.0 height:3.0];
        Rectangle *r2 = [[Rectangle alloc] initWithWidth:4.0 height:4.0];
        
        [r1 describe];
        [r2 describe];
        
        Rectangle *r3 = [r1 scaled:2.0];
        NSLog(@"\nScaled r1 by 2:");
        [r3 describe];
        
        // self ใน class method คือ class เอง
        // [Rectangle someClassMethod] - self คือ Rectangle class
    }
    return 0;
}
```

---

## 10.7 init และ dealloc

```objc
#import <Foundation/Foundation.h>

@interface Vehicle : NSObject

@property (nonatomic, copy) NSString *brand;
@property (nonatomic, copy) NSString *model;
@property (nonatomic, assign) NSInteger year;
@property (nonatomic, assign) double fuelLevel;
@property (nonatomic, assign) double odometer;

// Designated initializer
- (instancetype)initWithBrand:(NSString *)brand
                        model:(NSString *)model
                         year:(NSInteger)year;

// Convenience initializers
- (instancetype)initWithBrand:(NSString *)brand model:(NSString *)model;

// Methods
- (BOOL)refuel:(double)liters;
- (BOOL)drive:(double)km;
- (void)status;

@end

@implementation Vehicle

// Designated initializer - initializer หลักที่ทุกอย่างผ่านมา
- (instancetype)initWithBrand:(NSString *)brand
                        model:(NSString *)model
                         year:(NSInteger)year {
    self = [super init];  // เรียก NSObject's init
    if (self) {
        _brand = [brand copy];
        _model = [model copy];
        _year = year;
        _fuelLevel = 0;
        _odometer = 0;
        NSLog(@"[+] สร้าง %@ %@ (%ld)", _brand, _model, (long)_year);
    }
    return self;
}

// Convenience initializer - เรียก designated initializer
- (instancetype)initWithBrand:(NSString *)brand model:(NSString *)model {
    return [self initWithBrand:brand model:model year:2024];
}

// Default init - เรียก designated initializer ด้วย default values
- (instancetype)init {
    return [self initWithBrand:@"Unknown" model:@"Unknown" year:0];
}

- (void)dealloc {
    NSLog(@"[-] ทำลาย %@ %@", _brand, _model);
    // ARC จัดการ release properties อัตโนมัติ
    // ไม่ต้องเรียก [super dealloc]
}

- (BOOL)refuel:(double)liters {
    if (liters <= 0) return NO;
    double maxFuel = 50.0;  // ถังน้ำมัน 50 ลิตร
    _fuelLevel = MIN(_fuelLevel + liters, maxFuel);
    NSLog(@"เติมน้ำมัน %.1f ลิตร (รวม: %.1f ลิตร)", liters, _fuelLevel);
    return YES;
}

- (BOOL)drive:(double)km {
    double fuelNeeded = km * 0.1;  // 10 กม./ลิตร
    if (fuelNeeded > _fuelLevel) {
        NSLog(@"น้ำมันไม่พอ! ต้องการ %.1f ลิตร มี %.1f ลิตร",
              fuelNeeded, _fuelLevel);
        return NO;
    }
    _fuelLevel -= fuelNeeded;
    _odometer += km;
    NSLog(@"ขับ %.1f กม. (เหลือน้ำมัน: %.1f ลิตร, ระยะทางรวม: %.1f กม.)",
          km, _fuelLevel, _odometer);
    return YES;
}

- (void)status {
    NSLog(@"--- สถานะรถ ---");
    NSLog(@"รุ่น: %@ %@ (%ld)", _brand, _model, (long)_year);
    NSLog(@"น้ำมัน: %.1f ลิตร", _fuelLevel);
    NSLog(@"ระยะทาง: %.1f กม.", _odometer);
}

@end

int main(int argc, const char * argv[]) {
    @autoreleasepool {
        
        NSLog(@"=== init & dealloc ===\n");
        
        // สร้าง vehicles
        Vehicle *v1 = [[Vehicle alloc] initWithBrand:@"Toyota" model:@"Corolla" year:2020];
        Vehicle *v2 = [[Vehicle alloc] initWithBrand:@"Honda" model:@"Civic"];
        
        NSLog(@"");
        
        // ใช้งาน v1
        [v1 refuel:30.0];
        [v1 drive:100];
        [v1 drive:200];
        [v1 drive:100];  // น้ำมันไม่พอ
        [v1 status];
        
        NSLog(@"");
        
        // ใช้งาน v2
        [v2 refuel:40.0];
        [v2 drive:50];
        [v2 status];
        
        NSLog(@"\n--- จบ scope, objects จะถูก deallocate ---");
    }
    // v1 และ v2 ถูก deallocate ที่นี่
    NSLog(@"หลัง @autoreleasepool");
    return 0;
}
```

---

## 10.8 Encapsulation

```objc
#import <Foundation/Foundation.h>

// ตัวอย่าง Encapsulation ที่ดี
@interface Temperature : NSObject

// Public interface
- (instancetype)initWithCelsius:(double)celsius;
- (instancetype)initWithFahrenheit:(double)fahrenheit;
- (instancetype)initWithKelvin:(double)kelvin;

@property (nonatomic, readonly) double celsius;
@property (nonatomic, readonly) double fahrenheit;
@property (nonatomic, readonly) double kelvin;

- (NSString *)formatted;
- (BOOL)isBoiling;
- (BOOL)isFreezing;

@end

@implementation Temperature {
    double _celsius;  // เก็บเป็น celsius ภายใน
}

- (instancetype)initWithCelsius:(double)celsius {
    self = [super init];
    if (self) {
        _celsius = celsius;
    }
    return self;
}

- (instancetype)initWithFahrenheit:(double)fahrenheit {
    return [self initWithCelsius:(fahrenheit - 32) * 5.0 / 9.0];
}

- (instancetype)initWithKelvin:(double)kelvin {
    return [self initWithCelsius:kelvin - 273.15];
}

// Computed properties - คำนวณจาก _celsius
- (double)celsius {
    return _celsius;
}

- (double)fahrenheit {
    return _celsius * 9.0 / 5.0 + 32;
}

- (double)kelvin {
    return _celsius + 273.15;
}

- (NSString *)formatted {
    return [NSString stringWithFormat:@"%.2f°C / %.2f°F / %.2fK",
            self.celsius, self.fahrenheit, self.kelvin];
}

- (BOOL)isBoiling {
    return _celsius >= 100;
}

- (BOOL)isFreezing {
    return _celsius <= 0;
}

@end

int main(int argc, const char * argv[]) {
    @autoreleasepool {
        
        NSLog(@"=== Encapsulation ===");
        
        Temperature *bodyTemp = [[Temperature alloc] initWithCelsius:36.5];
        Temperature *boiling = [[Temperature alloc] initWithFahrenheit:212];
        Temperature *freezing = [[Temperature alloc] initWithKelvin:273.15];
        Temperature *room = [[Temperature alloc] initWithCelsius:25];
        
        NSLog(@"อุณหภูมิร่างกาย: %@", [bodyTemp formatted]);
        NSLog(@"จุดเดือด: %@", [boiling formatted]);
        NSLog(@"จุดเยือกแข็ง: %@", [freezing formatted]);
        NSLog(@"ห้อง: %@", [room formatted]);
        
        NSLog(@"\nน้ำเดือดหรือไม่ (boiling): %@",
              [boiling isBoiling] ? @"ใช่" : @"ไม่ใช่");
        NSLog(@"น้ำแข็งหรือไม่ (freezing): %@",
              [freezing isFreezing] ? @"ใช่" : @"ไม่ใช่");
    }
    return 0;
}
```

---

## 10.9 ตัวอย่างจริง: Person Class

```objc
#import <Foundation/Foundation.h>

@interface Address : NSObject
@property (nonatomic, copy) NSString *street;
@property (nonatomic, copy) NSString *city;
@property (nonatomic, copy) NSString *province;
@property (nonatomic, copy) NSString *zipCode;

- (instancetype)initWithStreet:(NSString *)street
                          city:(NSString *)city
                      province:(NSString *)province
                       zipCode:(NSString *)zipCode;
- (NSString *)fullAddress;
@end

@implementation Address
- (instancetype)initWithStreet:(NSString *)street
                          city:(NSString *)city
                      province:(NSString *)province
                       zipCode:(NSString *)zipCode {
    self = [super init];
    if (self) {
        _street = [street copy];
        _city = [city copy];
        _province = [province copy];
        _zipCode = [zipCode copy];
    }
    return self;
}

- (NSString *)fullAddress {
    return [NSString stringWithFormat:@"%@ %@ %@ %@",
            _street, _city, _province, _zipCode];
}

- (NSString *)description {
    return [self fullAddress];
}
@end

@interface Person : NSObject

@property (nonatomic, copy) NSString *firstName;
@property (nonatomic, copy) NSString *lastName;
@property (nonatomic, assign) NSInteger age;
@property (nonatomic, copy) NSString *email;
@property (nonatomic, copy) NSString *phone;
@property (nonatomic, strong) Address *address;
@property (nonatomic, strong) NSMutableArray *hobbies;

// Computed properties
@property (nonatomic, readonly) NSString *fullName;
@property (nonatomic, readonly) BOOL isAdult;

// Class methods
+ (instancetype)personWithFirstName:(NSString *)firstName
                           lastName:(NSString *)lastName
                                age:(NSInteger)age;

// Instance methods
- (void)addHobby:(NSString *)hobby;
- (void)removeHobby:(NSString *)hobby;
- (NSString *)introduce;
- (void)printInfo;

@end

@implementation Person

+ (instancetype)personWithFirstName:(NSString *)firstName
                           lastName:(NSString *)lastName
                                age:(NSInteger)age {
    Person *p = [[Person alloc] init];
    p.firstName = firstName;
    p.lastName = lastName;
    p.age = age;
    return p;
}

- (instancetype)init {
    self = [super init];
    if (self) {
        _hobbies = [NSMutableArray array];
    }
    return self;
}

- (NSString *)fullName {
    return [NSString stringWithFormat:@"%@ %@", _firstName, _lastName];
}

- (BOOL)isAdult {
    return _age >= 18;
}

- (void)addHobby:(NSString *)hobby {
    if (![_hobbies containsObject:hobby]) {
        [_hobbies addObject:hobby];
    }
}

- (void)removeHobby:(NSString *)hobby {
    [_hobbies removeObject:hobby];
}

- (NSString *)introduce {
    NSMutableString *intro = [NSMutableString string];
    [intro appendFormat:@"สวัสดี ผม/หนูชื่อ %@\n", self.fullName];
    [intro appendFormat:@"อายุ %ld ปี\n", (long)_age];
    
    if (_email) {
        [intro appendFormat:@"อีเมล: %@\n", _email];
    }
    if (_phone) {
        [intro appendFormat:@"โทร: %@\n", _phone];
    }
    if (_address) {
        [intro appendFormat:@"ที่อยู่: %@\n", _address.fullAddress];
    }
    if (_hobbies.count > 0) {
        [intro appendFormat:@"งานอดิเรก: %@\n",
         [_hobbies componentsJoinedByString:@", "]];
    }
    return intro;
}

- (void)printInfo {
    NSLog(@"\n%@", [self introduce]);
}

- (NSString *)description {
    return [NSString stringWithFormat:@"Person{%@, อายุ %ld}",
            self.fullName, (long)_age];
}

@end

int main(int argc, const char * argv[]) {
    @autoreleasepool {
        
        NSLog(@"=== Person Class Example ===");
        
        // สร้าง person
        Person *p1 = [Person personWithFirstName:@"สมชาย"
                                        lastName:@"ใจดี"
                                             age:28];
        p1.email = @"somchai@example.com";
        p1.phone = @"081-234-5678";
        p1.address = [[Address alloc] initWithStreet:@"123 ถนนสุขุมวิท"
                                               city:@"กรุงเทพ"
                                           province:@"กรุงเทพมหานคร"
                                            zipCode:@"10110"];
        
        [p1 addHobby:@"อ่านหนังสือ"];
        [p1 addHobby:@"เล่นกีตาร์"];
        [p1 addHobby:@"ว่ายน้ำ"];
        
        [p1 printInfo];
        
        Person *p2 = [Person personWithFirstName:@"มณี"
                                        lastName:@"สวยงาม"
                                             age:16];
        [p2 addHobby:@"วาดรูป"];
        
        NSLog(@"\n%@ เป็นผู้ใหญ่: %@",
              p2.fullName, p2.isAdult ? @"ใช่" : @"ไม่ใช่");
        
        // NSLog array
        NSArray *people = @[p1, p2];
        NSLog(@"\nรายชื่อทั้งหมด:");
        for (Person *p in people) {
            NSLog(@"  - %@", p);
        }
    }
    return 0;
}
```

---

## 10.10 ตัวอย่างจริง: Car Class

```objc
#import <Foundation/Foundation.h>

typedef NS_ENUM(NSInteger, FuelType) {
    FuelTypeGasoline,
    FuelTypeDiesel,
    FuelTypeElectric,
    FuelTypeHybrid
};

typedef NS_ENUM(NSInteger, TransmissionType) {
    TransmissionManual,
    TransmissionAutomatic,
    TransmissionCVT
};

@interface Car : NSObject

// Properties
@property (nonatomic, copy) NSString *make;
@property (nonatomic, copy) NSString *model;
@property (nonatomic, assign) NSInteger year;
@property (nonatomic, copy) NSString *color;
@property (nonatomic, assign) FuelType fuelType;
@property (nonatomic, assign) TransmissionType transmission;
@property (nonatomic, assign) double engineSize;    // cc
@property (nonatomic, assign) NSInteger horsepower;
@property (nonatomic, assign) double price;

// State
@property (nonatomic, readonly) BOOL isRunning;
@property (nonatomic, readonly) double currentSpeed; // km/h
@property (nonatomic, readonly) double fuelLevel;    // liters
@property (nonatomic, readonly) double totalDistance;

// Class methods
+ (instancetype)carWithMake:(NSString *)make
                      model:(NSString *)model
                       year:(NSInteger)year;
+ (NSString *)fuelTypeName:(FuelType)type;

// Operations
- (BOOL)startEngine;
- (void)stopEngine;
- (void)accelerate:(double)kmh;
- (void)brake:(double)kmh;
- (BOOL)refuel:(double)liters;
- (NSString *)specs;
- (void)printStatus;

@end

@implementation Car {
    BOOL _isRunning;
    double _currentSpeed;
    double _fuelLevel;
    double _totalDistance;
    double _fuelCapacity;
    double _fuelEfficiency;  // km/L
}

+ (instancetype)carWithMake:(NSString *)make
                      model:(NSString *)model
                       year:(NSInteger)year {
    Car *car = [[Car alloc] init];
    car.make = make;
    car.model = model;
    car.year = year;
    return car;
}

+ (NSString *)fuelTypeName:(FuelType)type {
    switch (type) {
        case FuelTypeGasoline: return @"น้ำมันเบนซิน";
        case FuelTypeDiesel:   return @"ดีเซล";
        case FuelTypeElectric: return @"ไฟฟ้า";
        case FuelTypeHybrid:   return @"ไฮบริด";
    }
}

- (instancetype)init {
    self = [super init];
    if (self) {
        _isRunning = NO;
        _currentSpeed = 0;
        _fuelLevel = 0;
        _totalDistance = 0;
        _fuelCapacity = 50;
        _fuelEfficiency = 15;  // 15 km/L default
        _color = @"ขาว";
        _fuelType = FuelTypeGasoline;
        _transmission = TransmissionAutomatic;
    }
    return self;
}

- (BOOL)isRunning { return _isRunning; }
- (double)currentSpeed { return _currentSpeed; }
- (double)fuelLevel { return _fuelLevel; }
- (double)totalDistance { return _totalDistance; }

- (BOOL)startEngine {
    if (_isRunning) {
        NSLog(@"เครื่องยนต์ทำงานอยู่แล้ว");
        return NO;
    }
    if (_fuelLevel <= 0 && _fuelType != FuelTypeElectric) {
        NSLog(@"น้ำมันหมด ไม่สามารถสตาร์ทได้");
        return NO;
    }
    _isRunning = YES;
    NSLog(@"🔑 เครื่องยนต์ %@ %@ (%ld) ติดแล้ว",
          _make, _model, (long)_year);
    return YES;
}

- (void)stopEngine {
    if (!_isRunning) {
        NSLog(@"เครื่องยนต์ไม่ได้ทำงาน");
        return;
    }
    _isRunning = NO;
    _currentSpeed = 0;
    NSLog(@"🛑 ดับเครื่องยนต์");
}

- (void)accelerate:(double)kmh {
    if (!_isRunning) {
        NSLog(@"ต้องสตาร์ทเครื่องยนต์ก่อน");
        return;
    }
    double newSpeed = _currentSpeed + kmh;
    double fuelUsed = kmh / _fuelEfficiency * 0.1;
    
    if (_fuelLevel < fuelUsed) {
        NSLog(@"น้ำมันไม่พอ");
        return;
    }
    
    _currentSpeed = MIN(newSpeed, 200);  // max 200 km/h
    _fuelLevel -= fuelUsed;
    _totalDistance += kmh * 0.5;  // simulation
    
    NSLog(@"⬆️ เพิ่มความเร็ว %.0f km/h (ความเร็วปัจจุบัน: %.0f km/h)",
          kmh, _currentSpeed);
}

- (void)brake:(double)kmh {
    _currentSpeed = MAX(0, _currentSpeed - kmh);
    NSLog(@"⬇️ ลดความเร็ว %.0f km/h (ความเร็วปัจจุบัน: %.0f km/h)",
          kmh, _currentSpeed);
}

- (BOOL)refuel:(double)liters {
    if (_isRunning) {
        NSLog(@"ดับเครื่องก่อนเติมน้ำมัน");
        return NO;
    }
    double added = MIN(liters, _fuelCapacity - _fuelLevel);
    _fuelLevel += added;
    NSLog(@"⛽ เติมน้ำมัน %.1f ลิตร (รวม: %.1f/%.1f ลิตร)",
          added, _fuelLevel, _fuelCapacity);
    return YES;
}

- (NSString *)specs {
    return [NSString stringWithFormat:
            @"%ld %@ %@\n"
            @"  สี: %@\n"
            @"  เครื่องยนต์: %.0f cc, %ld แรงม้า\n"
            @"  เชื้อเพลิง: %@\n"
            @"  เกียร์: %@\n"
            @"  ราคา: ฿%.0f",
            (long)_year, _make, _model,
            _color,
            _engineSize, (long)_horsepower,
            [Car fuelTypeName:_fuelType],
            _transmission == TransmissionManual ? @"Manual" :
            _transmission == TransmissionAutomatic ? @"Automatic" : @"CVT",
            _price];
}

- (void)printStatus {
    NSLog(@"\n=== สถานะรถ ===");
    NSLog(@"รุ่น: %ld %@ %@", (long)_year, _make, _model);
    NSLog(@"เครื่องยนต์: %@", _isRunning ? @"ทำงาน" : @"ดับ");
    NSLog(@"ความเร็ว: %.0f km/h", _currentSpeed);
    NSLog(@"น้ำมัน: %.1f/%.1f ลิตร (%.0f%%)",
          _fuelLevel, _fuelCapacity,
          (_fuelLevel / _fuelCapacity) * 100);
    NSLog(@"ระยะทางรวม: %.1f km", _totalDistance);
}

- (NSString *)description {
    return [NSString stringWithFormat:@"%ld %@ %@", (long)_year, _make, _model];
}

@end

int main(int argc, const char * argv[]) {
    @autoreleasepool {
        
        NSLog(@"=== Car Class Example ===\n");
        
        Car *car = [Car carWithMake:@"Toyota" model:@"Fortuner" year:2023];
        car.color = @"ดำ";
        car.engineSize = 2800;
        car.horsepower = 204;
        car.fuelType = FuelTypeDiesel;
        car.transmission = TransmissionAutomatic;
        car.price = 1490000;
        
        NSLog(@"=== สเปค ===\n%@\n", [car specs]);
        
        // Simulation
        [car refuel:45];
        [car startEngine];
        [car accelerate:60];
        [car accelerate:40];
        [car brake:30];
        [car accelerate:20];
        [car brake:90];  // brake หมด
        [car stopEngine];
        [car printStatus];
    }
    return 0;
}
```

---

## 10.11 Inheritance - การสืบทอด (Preview)

```objc
#import <Foundation/Foundation.h>

// Base class
@interface Animal : NSObject
@property (nonatomic, copy) NSString *name;
@property (nonatomic, assign) NSInteger age;

- (instancetype)initWithName:(NSString *)name age:(NSInteger)age;
- (void)makeSound;
- (void)eat:(NSString *)food;
- (NSString *)describe;
@end

@implementation Animal
- (instancetype)initWithName:(NSString *)name age:(NSInteger)age {
    self = [super init];
    if (self) { _name = name; _age = age; }
    return self;
}
- (void)makeSound {
    NSLog(@"%@ ส่งเสียง (generic)", _name);
}
- (void)eat:(NSString *)food {
    NSLog(@"%@ กิน %@", _name, food);
}
- (NSString *)describe {
    return [NSString stringWithFormat:@"%@ (อายุ %ld ปี)", _name, (long)_age];
}
@end

// Subclasses
@interface Dog : Animal
@property (nonatomic, copy) NSString *breed;
- (void)fetch:(NSString *)item;
@end

@implementation Dog
- (void)makeSound {
    NSLog(@"%@: โฮ่ง! โฮ่ง! 🐕", _name);  // Override
}
- (void)fetch:(NSString *)item {
    NSLog(@"%@ วิ่งไปเอา %@", _name, item);
}
@end

@interface Cat : Animal
@property (nonatomic, assign) BOOL isIndoor;
- (void)purr;
@end

@implementation Cat
- (void)makeSound {
    NSLog(@"%@: เมี้ยว~ 🐈", _name);  // Override
}
- (void)purr {
    NSLog(@"%@: *เร่งๆๆ*", _name);
}
@end

int main(int argc, const char * argv[]) {
    @autoreleasepool {
        
        NSLog(@"=== Inheritance Preview ===\n");
        
        // Polymorphism: array ของ Animal references
        NSArray *animals = @[
            [[Dog alloc] initWithName:@"บัดดี้" age:3],
            [[Cat alloc] initWithName:@"วิสกี้" age:2],
            [[Animal alloc] initWithName:@"กระรอก" age:1]
        ];
        
        for (Animal *animal in animals) {
            NSLog(@"%@:", animal.describe);
            [animal makeSound];  // Polymorphism!
            [animal eat:@"อาหาร"];
        }
        
        // ตรวจสอบชนิด
        NSLog(@"\n=== isKindOfClass ===");
        for (Animal *animal in animals) {
            NSLog(@"%@ เป็น Dog: %@", animal.name,
                  [animal isKindOfClass:[Dog class]] ? @"YES" : @"NO");
        }
        
        // Downcast
        Dog *dog = (Dog *)animals[0];
        [dog fetch:@"ลูกบอล"];
        
        Cat *cat = (Cat *)animals[1];
        [cat purr];
    }
    return 0;
}
```

---

## 10.12 UML Class Diagram เบื้องต้น

```
UML Class Diagram แสดงโครงสร้างของ classes และความสัมพันธ์

┌─────────────────────────────┐
│         ClassName           │  ← ชื่อ class
├─────────────────────────────┤
│ + publicProperty: Type      │  ← Properties
│ - privateProperty: Type     │    + = public
│ # protectedProperty: Type   │    - = private
├─────────────────────────────┤  # = protected
│ + classMethod(): ReturnType │  ← Methods
│ - instanceMethod(): Type    │    พร้อม return type
└─────────────────────────────┘

ความสัมพันธ์:
  A ──────── B   Association (มีความสัมพันธ์)
  A ───────▷ B   Inheritance (A สืบทอดจาก B)
  A ─ ─ ─ ▷ B   Realization (A implement B protocol)
  A ◆──────  B   Composition (A เป็นเจ้าของ B)
  A ◇──────  B   Aggregation (A ใช้ B)

ตัวอย่าง Person และ Address:

┌────────────────────────────┐
│           Person           │
├────────────────────────────┤
│ + firstName: NSString      │
│ + lastName: NSString       │
│ + age: NSInteger           │
│ + address: Address         │
│ + hobbies: NSMutableArray  │
├────────────────────────────┤
│ + fullName: NSString       │
│ + isAdult: BOOL            │
│ + greet(): void            │
│ + addHobby(h): void        │
│ + introduce(): NSString    │
└──────────────┬─────────────┘
               │ has-a
               ▼
┌────────────────────────────┐
│           Address          │
├────────────────────────────┤
│ + street: NSString         │
│ + city: NSString           │
│ + province: NSString       │
│ + zipCode: NSString        │
├────────────────────────────┤
│ + fullAddress(): NSString  │
└────────────────────────────┘

Inheritance diagram:
┌────────────────────────────┐
│          NSObject          │
└──────────────┬─────────────┘
               │ inherits
    ┌──────────┴──────────┐
    ▼                     ▼
┌───────────┐      ┌───────────┐
│  Animal   │      │  Vehicle  │
└─────┬─────┘      └─────┬─────┘
      │                  │
  ┌───┴───┐          ┌───┴───┐
  ▼       ▼          ▼       ▼
┌─────┐ ┌─────┐  ┌──────┐ ┌──────┐
│ Dog │ │ Cat │  │  Car │ │Truck │
└─────┘ └─────┘  └──────┘ └──────┘
```

---

## 10.13 สรุปหลักการ OOP ใน Objective-C

```objc
// Encapsulation
@interface BankAccount : NSObject
@property (nonatomic, readonly) double balance;  // readonly = encapsulate
- (void)deposit:(double)amount;                  // interface เท่านั้น
@end

// Inheritance
@interface SavingsAccount : BankAccount  // SavingsAccount สืบทอด BankAccount
@property (nonatomic, assign) double interestRate;
- (void)applyInterest;
@end

// Polymorphism
// BankAccount *accounts[] = {checking, savings, fixed_deposit};
// for each account: [account addInterest];  // ทุก account ทำต่างกัน

// Abstraction
@protocol Printable  // protocol = abstraction
- (void)print;
- (NSString *)toPDF;
@end
```

---

## แบบฝึกหัด

### ข้อที่ 1: Student Class
```objc
// สร้าง class Student ที่มี:
// Properties: name, studentID, GPA, courses (NSMutableArray)
// Methods:
//   - initWithName:studentID:
//   - enrollCourse:
//   - dropCourse:
//   - calculateGPA (จาก array ของ grades)
//   - printTranscript
//   - isHonorsStudent (GPA >= 3.5)
```

### ข้อที่ 2: Rectangle ขั้นสูง
```objc
// สร้าง Rectangle class ที่มี:
// Properties: width, height, color, borderWidth
// Methods:
//   - area, perimeter
//   - isSquare
//   - scale: (คูณทั้ง width และ height)
//   - containsPoint:y: (ตรวจว่า point อยู่ใน rectangle หรือไม่)
//   - intersectsWith: (ตรวจว่า 2 rectangles ตัดกันหรือไม่)
//   - draw (ใช้ ASCII art)
```

### ข้อที่ 3: Library System
```objc
// สร้างระบบห้องสมุดอย่างง่าย:
// Book class: title, author, ISBN, isAvailable
// Library class:
//   - books (NSMutableArray)
//   - addBook:, removeBook:
//   - searchByTitle:, searchByAuthor:
//   - borrowBook: (ตรวจ availability)
//   - returnBook:
//   - availableBooks
//   - printCatalog
```

### ข้อที่ 4: Shape Hierarchy (Preview Polymorphism)
```objc
// Base: Shape (area, perimeter, draw)
// Circle: สืบทอด Shape
// Rectangle: สืบทอด Shape
// Triangle: สืบทอด Shape
// ทดสอบ polymorphism ด้วย array ของ Shape objects
```

### ข้อที่ 5: Inventory System
```objc
// Product class: name, price, quantity, category
// Inventory class:
//   - products (NSMutableArray)
//   - addProduct:, removeProduct:
//   - totalValue (ผลรวม price * quantity)
//   - lowStockProducts (quantity < 5)
//   - searchByCategory:
//   - sortByPrice:ascending:
//   - printReport
```

### ข้อที่ 6: Todo List
```objc
// TodoItem class: title, description, isCompleted, priority, dueDate
// TodoList class:
//   - items (NSMutableArray)
//   - addItem:, removeItem:, completeItem:
//   - filterByPriority:
//   - overdueTasks
//   - completionRate (เปอร์เซ็นต์ที่ทำเสร็จ)
//   - sortByPriority, sortByDueDate
```

### ข้อที่ 7: ATM Simulation
```objc
// Account class: accountNumber, ownerName, balance, PIN
// ATM class:
//   - accounts (NSMutableDictionary)
//   - authenticate:pin: (return Account or nil)
//   - checkBalance:
//   - deposit:amount:
//   - withdraw:amount:
//   - transfer:toAccount:amount:
//   - transaction history
```

### ข้อที่ 8: School System
```objc
// Student, Teacher, Subject, Grade, Classroom classes
// School class ที่ manage ทั้งหมด
// รายงาน: นักเรียนที่ GPA สูงสุด, วิชาที่คะแนนเฉลี่ยสูงสุด
```

---

## เฉลยตัวอย่าง (ข้อที่ 1: Student Class)

```objc
#import <Foundation/Foundation.h>

@interface Course : NSObject
@property (nonatomic, copy) NSString *name;
@property (nonatomic, copy) NSString *code;
@property (nonatomic, assign) NSInteger credits;
@property (nonatomic, assign) double grade;  // 0.0-4.0

+ (instancetype)courseWithName:(NSString *)name code:(NSString *)code
                       credits:(NSInteger)credits grade:(double)grade;
@end

@implementation Course
+ (instancetype)courseWithName:(NSString *)name code:(NSString *)code
                       credits:(NSInteger)credits grade:(double)grade {
    Course *c = [[Course alloc] init];
    c.name = name; c.code = code; c.credits = credits; c.grade = grade;
    return c;
}
- (NSString *)description {
    return [NSString stringWithFormat:@"%@ (%@): %.1f", _name, _code, _grade];
}
@end

@interface Student : NSObject
@property (nonatomic, copy) NSString *name;
@property (nonatomic, copy) NSString *studentID;
@property (nonatomic, strong) NSMutableArray<Course *> *courses;

- (instancetype)initWithName:(NSString *)name studentID:(NSString *)studentID;
- (void)enrollCourse:(Course *)course;
- (BOOL)dropCourse:(NSString *)courseCode;
- (double)calculateGPA;
- (BOOL)isHonorsStudent;
- (void)printTranscript;
@end

@implementation Student

- (instancetype)initWithName:(NSString *)name studentID:(NSString *)studentID {
    self = [super init];
    if (self) {
        _name = name;
        _studentID = studentID;
        _courses = [NSMutableArray array];
    }
    return self;
}

- (void)enrollCourse:(Course *)course {
    // ตรวจว่าลงทะเบียนแล้ว
    for (Course *c in _courses) {
        if ([c.code isEqualToString:course.code]) {
            NSLog(@"ลงทะเบียน %@ แล้ว", course.code);
            return;
        }
    }
    [_courses addObject:course];
    NSLog(@"ลงทะเบียน %@ สำเร็จ", course.name);
}

- (BOOL)dropCourse:(NSString *)courseCode {
    for (Course *c in _courses) {
        if ([c.code isEqualToString:courseCode]) {
            [_courses removeObject:c];
            NSLog(@"ถอนวิชา %@ สำเร็จ", courseCode);
            return YES;
        }
    }
    NSLog(@"ไม่พบวิชา %@", courseCode);
    return NO;
}

- (double)calculateGPA {
    if (_courses.count == 0) return 0;
    
    double totalPoints = 0;
    NSInteger totalCredits = 0;
    
    for (Course *c in _courses) {
        totalPoints += c.grade * c.credits;
        totalCredits += c.credits;
    }
    
    return totalCredits > 0 ? totalPoints / totalCredits : 0;
}

- (BOOL)isHonorsStudent {
    return [self calculateGPA] >= 3.5;
}

- (void)printTranscript {
    NSLog(@"\n=== ใบแสดงผลการเรียน ===");
    NSLog(@"ชื่อ: %@", _name);
    NSLog(@"รหัสนักศึกษา: %@", _studentID);
    NSLog(@"\nวิชาที่ลงทะเบียน:");
    
    for (Course *c in _courses) {
        NSString *grade = @"";
        if (c.grade >= 3.5) grade = @"A";
        else if (c.grade >= 3.0) grade = @"B+";
        else if (c.grade >= 2.5) grade = @"B";
        else if (c.grade >= 2.0) grade = @"C+";
        else if (c.grade >= 1.5) grade = @"C";
        else grade = @"F";
        
        NSLog(@"  %-20s %@ หน่วยกิต: %ld (เกรด: %@)",
              [c.name UTF8String], grade, (long)c.credits, grade);
    }
    
    NSLog(@"\nGPA: %.2f", [self calculateGPA]);
    NSLog(@"เกียรตินิยม: %@", [self isHonorsStudent] ? @"ได้รับ" : @"ไม่ได้รับ");
}

@end

int main(int argc, const char * argv[]) {
    @autoreleasepool {
        
        Student *stu = [[Student alloc] initWithName:@"สมชาย ใจดี"
                                          studentID:@"6401001"];
        
        [stu enrollCourse:[Course courseWithName:@"Calculus" code:@"MATH101"
                                         credits:3 grade:3.5]];
        [stu enrollCourse:[Course courseWithName:@"Physics" code:@"PHYS101"
                                         credits:3 grade:3.0]];
        [stu enrollCourse:[Course courseWithName:@"Programming" code:@"CS101"
                                         credits:4 grade:4.0]];
        [stu enrollCourse:[Course courseWithName:@"English" code:@"ENG101"
                                         credits:3 grade:3.5]];
        
        [stu printTranscript];
        
        // Drop a course
        NSLog(@"");
        [stu dropCourse:@"PHYS101"];
        [stu printTranscript];
    }
    return 0;
}
```

---

## สรุปบทที่ 10

| แนวคิด | คำอธิบาย | ใน Objective-C |
|--------|---------|----------------|
| Class | แบบพิมพ์เขียวของ object | @interface ... @end |
| Object | instance ของ class | [[ClassName alloc] init] |
| Encapsulation | ซ่อน internals | private ivar, public property |
| Inheritance | สืบทอด | @interface Child : Parent |
| Polymorphism | หลายรูปแบบ | method overriding |
| @interface | ประกาศ class | .h file |
| @implementation | implement class | .m file |
| - method | instance method | - (returnType)methodName |
| + method | class method | + (returnType)methodName |
| self | ตัวเอง | เหมือน this ใน Java/C++ |
| super | parent class | [super method] |
| init | constructor | - (instancetype)init |
| dealloc | destructor | - (void)dealloc |
| property | getter/setter อัตโนมัติ | @property (attrs) Type name |

---

*จบบทที่ 10: Introduction to OOP* | ไปยัง [บทที่ 11: Inheritance & Polymorphism →](../part-11-20/part-11-inheritance.md)
