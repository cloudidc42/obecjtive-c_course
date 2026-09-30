# ตอนที่ 34: Key-Value Coding (KVC)

## บทนำ

Key-Value Coding (KVC) เป็นกลไกใน Objective-C และ Swift ที่ช่วยให้เราเข้าถึงและแก้ไข properties ของ object โดยใช้ string key แทนที่จะเรียก accessor methods โดยตรง มันเป็นส่วนหนึ่งของ Foundation framework และเป็นรากฐานของ KVO (Key-Value Observing), Core Data, Bindings และเทคโนโลยีอื่นๆ อีกมากมายใน Cocoa

## KVC คืออะไร?

KVC ช่วยให้เราเขียนโค้ดที่ยืดหยุ่นกว่า โดยเข้าถึง property ผ่าน string key:

```objc
// วิธีปกติ
NSString *name = person.name;
person.name = @"Alice";

// วิธี KVC
NSString *name = [person valueForKey:@"name"];
[person setValue:@"Alice" forKey:@"name"];
```

แม้ดูเหมือนยากขึ้น แต่ KVC มีประโยชน์มากเมื่อ key เป็น dynamic (ไม่รู้จนกว่า runtime)

---

## 34.1 valueForKey: และ setValue:forKey:

### valueForKey:

```objc
#import <Foundation/Foundation.h>

@interface Person : NSObject

@property (nonatomic, strong) NSString *name;
@property (nonatomic, assign) NSInteger age;
@property (nonatomic, strong) NSString *email;
@property (nonatomic, strong) NSString *city;

@end

@implementation Person
@end
```

```objc
// สร้าง person object
Person *person = [[Person alloc] init];
person.name = @"สมชาย ใจดี";
person.age = 30;
person.email = @"somchai@example.com";
person.city = @"Bangkok";

// เข้าถึง properties ด้วย KVC
NSString *name = [person valueForKey:@"name"];
NSNumber *age = [person valueForKey:@"age"];   // primitive ถูก wrapped ใน NSNumber
NSString *email = [person valueForKey:@"email"];

NSLog(@"ชื่อ: %@", name);
NSLog(@"อายุ: %@", age);
NSLog(@"อีเมล: %@", email);
```

### setValue:forKey:

```objc
// กำหนดค่าด้วย KVC
[person setValue:@"สมหญิง ใจดี" forKey:@"name"];
[person setValue:@25 forKey:@"age"];  // NSNumber สำหรับ primitive
[person setValue:@"somying@example.com" forKey:@"email"];

NSLog(@"ชื่อใหม่: %@", person.name);
NSLog(@"อายุใหม่: %ld", (long)person.age);
```

### Dynamic Property Access

KVC มีประโยชน์มากเมื่อต้องการ access property แบบ dynamic:

```objc
// สร้างฟังก์ชันที่ update properties แบบ dynamic
void updatePersonProperties(Person *person, NSDictionary *updates) {
    for (NSString *key in updates) {
        @try {
            [person setValue:updates[key] forKey:key];
            NSLog(@"อัปเดต %@ = %@", key, updates[key]);
        } @catch (NSException *exception) {
            NSLog(@"ไม่สามารถอัปเดต key '%@': %@", key, exception.reason);
        }
    }
}

// การใช้งาน
Person *person = [[Person alloc] init];
NSDictionary *updates = @{
    @"name": @"Alice Smith",
    @"age": @28,
    @"email": @"alice@example.com",
    @"city": @"Chiang Mai",
};
updatePersonProperties(person, updates);
```

### การจัดการ nil และ undefined keys

```objc
// setValue:forKey: กับ nil
// สำหรับ object properties: ค่าจะเป็น nil
// สำหรับ primitive properties: จะเรียก setNilValueForKey:

@interface Person : NSObject
@property (nonatomic, assign) NSInteger age;
- (void)setNilValueForKey:(NSString *)key; // override เพื่อจัดการ
@end

@implementation Person

- (void)setNilValueForKey:(NSString *)key {
    if ([key isEqualToString:@"age"]) {
        self.age = 0; // ค่า default สำหรับ nil
    } else {
        [super setNilValueForKey:key];
    }
}

@end

// undefined key
@interface Person : NSObject
- (id)valueForUndefinedKey:(NSString *)key;
- (void)setValue:(id)value forUndefinedKey:(NSString *)key;
@end

@implementation Person

- (id)valueForUndefinedKey:(NSString *)key {
    NSLog(@"พยายามอ่าน key ที่ไม่มีอยู่: %@", key);
    return nil; // คืนค่า nil แทน throw exception
}

- (void)setValue:(id)value forUndefinedKey:(NSString *)key {
    NSLog(@"พยายามกำหนดค่า key ที่ไม่มีอยู่: %@ = %@", key, value);
    // ไม่ทำอะไร แทน throw exception
}

@end
```

---

## 34.2 valueForKeyPath: (Nested Properties)

Key Path ช่วยให้เข้าถึง nested properties ได้สะดวก โดยใช้จุด (.) เป็นตัวคั่น

### Nested Object Access

```objc
@interface Address : NSObject
@property (nonatomic, strong) NSString *street;
@property (nonatomic, strong) NSString *city;
@property (nonatomic, strong) NSString *country;
@property (nonatomic, assign) NSInteger postalCode;
@end

@implementation Address
@end

@interface Person : NSObject
@property (nonatomic, strong) NSString *name;
@property (nonatomic, assign) NSInteger age;
@property (nonatomic, strong) Address *address;
@end

@implementation Person
@end
```

```objc
// สร้าง objects
Address *address = [[Address alloc] init];
address.street = @"123 ถ.สุขุมวิท";
address.city = @"Bangkok";
address.country = @"Thailand";
address.postalCode = 10110;

Person *person = [[Person alloc] init];
person.name = @"สมชาย ใจดี";
person.age = 30;
person.address = address;

// เข้าถึง nested properties ด้วย key path
NSString *city = [person valueForKeyPath:@"address.city"];
NSString *country = [person valueForKeyPath:@"address.country"];
NSNumber *postalCode = [person valueForKeyPath:@"address.postalCode"];

NSLog(@"เมือง: %@", city);           // Bangkok
NSLog(@"ประเทศ: %@", country);       // Thailand
NSLog(@"รหัสไปรษณีย์: %@", postalCode); // 10110

// กำหนดค่าด้วย key path
[person setValue:@"Chiang Mai" forKeyPath:@"address.city"];
NSLog(@"เมืองใหม่: %@", person.address.city); // Chiang Mai
```

### Deep Nested Access

```objc
@interface Company : NSObject
@property (nonatomic, strong) NSString *name;
@property (nonatomic, strong) Person *ceo;
@end

@implementation Company
@end

Company *company = [[Company alloc] init];
company.name = @"บริษัท ABC จำกัด";
company.ceo = person;

// เข้าถึง nested 3 ระดับ
NSString *ceoCityName = [company valueForKeyPath:@"ceo.address.city"];
NSLog(@"เมืองของ CEO: %@", ceoCityName);
```

### Key Path กับ Array

```objc
// เมื่อ key path ผ่าน collection
@interface Department : NSObject
@property (nonatomic, strong) NSString *name;
@property (nonatomic, strong) NSArray *employees; // NSArray of Person
@end

@implementation Department
@end

Person *alice = [[Person alloc] init];
alice.name = @"Alice";
alice.age = 25;

Person *bob = [[Person alloc] init];
bob.name = @"Bob";
bob.age = 30;

Person *charlie = [[Person alloc] init];
charlie.name = @"Charlie";
charlie.age = 35;

Department *dept = [[Department alloc] init];
dept.name = @"Engineering";
dept.employees = @[alice, bob, charlie];

// valueForKeyPath บน array จะ map property ทุก object
NSArray *employeeNames = [dept valueForKeyPath:@"employees.name"];
NSLog(@"พนักงาน: %@", employeeNames);
// Output: ("Alice", "Bob", "Charlie")

NSArray *employeeAges = [dept valueForKeyPath:@"employees.age"];
NSLog(@"อายุ: %@", employeeAges);
// Output: (25, 30, 35)
```

---

## 34.3 KVC Collection Operators

Collection operators เป็นฟีเจอร์ที่ทรงพลังของ KVC ที่ช่วยคำนวณค่าสถิติจาก collections

### โครงสร้าง Key Path ของ Collection Operator

```
[left key path].[collection operator].[right key path]
เช่น: employees.@sum.salary
```

### @sum - ผลรวม

```objc
@interface Employee : NSObject

@property (nonatomic, strong) NSString *name;
@property (nonatomic, assign) CGFloat salary;
@property (nonatomic, assign) NSInteger yearsOfService;
@property (nonatomic, strong) NSString *department;

@end

@implementation Employee
@end

// สร้างข้อมูล employees
NSMutableArray *employees = [NSMutableArray array];

NSDictionary *empData[] = {
    @{@"name": @"Alice",   @"salary": @75000, @"years": @3, @"dept": @"Engineering"},
    @{@"name": @"Bob",     @"salary": @85000, @"years": @5, @"dept": @"Engineering"},
    @{@"name": @"Charlie", @"salary": @65000, @"years": @2, @"dept": @"Marketing"},
    @{@"name": @"Diana",   @"salary": @90000, @"years": @7, @"dept": @"Management"},
    @{@"name": @"Eve",     @"salary": @70000, @"years": @4, @"dept": @"Marketing"},
};

int empCount = sizeof(empData) / sizeof(empData[0]);
for (int i = 0; i < empCount; i++) {
    Employee *emp = [[Employee alloc] init];
    emp.name = empData[i][@"name"];
    emp.salary = [empData[i][@"salary"] floatValue];
    emp.yearsOfService = [empData[i][@"years"] integerValue];
    emp.department = empData[i][@"dept"];
    [employees addObject:emp];
}

// คำนวณผลรวมเงินเดือน
NSNumber *totalSalary = [employees valueForKeyPath:@"@sum.salary"];
NSLog(@"เงินเดือนรวม: %.2f บาท", [totalSalary floatValue]);
// Output: เงินเดือนรวม: 385000.00 บาท
```

### @avg - ค่าเฉลี่ย

```objc
NSNumber *avgSalary = [employees valueForKeyPath:@"@avg.salary"];
NSNumber *avgYears = [employees valueForKeyPath:@"@avg.yearsOfService"];

NSLog(@"เงินเดือนเฉลี่ย: %.2f บาท", [avgSalary floatValue]);
NSLog(@"ปีทำงานเฉลี่ย: %.1f ปี", [avgYears floatValue]);
```

### @max - ค่าสูงสุด

```objc
NSNumber *maxSalary = [employees valueForKeyPath:@"@max.salary"];
NSNumber *maxYears = [employees valueForKeyPath:@"@max.yearsOfService"];

NSLog(@"เงินเดือนสูงสุด: %.2f บาท", [maxSalary floatValue]);
NSLog(@"ปีทำงานสูงสุด: %ld ปี", (long)[maxYears integerValue]);
```

### @min - ค่าต่ำสุด

```objc
NSNumber *minSalary = [employees valueForKeyPath:@"@min.salary"];
NSNumber *minYears = [employees valueForKeyPath:@"@min.yearsOfService"];

NSLog(@"เงินเดือนต่ำสุด: %.2f บาท", [minSalary floatValue]);
NSLog(@"ปีทำงานต่ำสุด: %ld ปี", (long)[minYears integerValue]);
```

### @count - จำนวน

```objc
NSNumber *count = [employees valueForKeyPath:@"@count"];
NSLog(@"จำนวนพนักงาน: %@ คน", count);

// @count ไม่ต้องการ right key path
// แต่ถ้าใส่ก็ได้ (จะถูกละเว้น)
NSNumber *count2 = [employees valueForKeyPath:@"@count.name"];
NSLog(@"จำนวน: %@", count2); // เหมือนกัน
```

### ใช้ทุก operators พร้อมกัน

```objc
// สรุปสถิติเงินเดือน
NSLog(@"=== สถิติเงินเดือน ===");
NSLog(@"จำนวนพนักงาน: %@", [employees valueForKeyPath:@"@count"]);
NSLog(@"เงินเดือนรวม: %.2f", [[employees valueForKeyPath:@"@sum.salary"] floatValue]);
NSLog(@"เงินเดือนเฉลี่ย: %.2f", [[employees valueForKeyPath:@"@avg.salary"] floatValue]);
NSLog(@"เงินเดือนสูงสุด: %.2f", [[employees valueForKeyPath:@"@max.salary"] floatValue]);
NSLog(@"เงินเดือนต่ำสุด: %.2f", [[employees valueForKeyPath:@"@min.salary"] floatValue]);
```

---

## 34.4 Array Operators

Array operators ส่งคืน NSArray ของผลลัพธ์

### @distinctUnionOfObjects - ค่าที่ไม่ซ้ำ

```objc
// ดึง departments ที่ไม่ซ้ำกัน
NSArray *uniqueDepts = [employees valueForKeyPath:@"@distinctUnionOfObjects.department"];
NSLog(@"แผนกต่างๆ: %@", uniqueDepts);
// Output: ("Engineering", "Marketing", "Management")

// เปรียบเทียบกับ @unionOfObjects ที่ไม่ลบค่าซ้ำ
NSArray *allDepts = [employees valueForKeyPath:@"@unionOfObjects.department"];
NSLog(@"แผนกทั้งหมด (มีซ้ำ): %@", allDepts);
// Output: ("Engineering", "Engineering", "Marketing", "Management", "Marketing")
```

### @unionOfObjects - ค่าทั้งหมด (รวมซ้ำ)

```objc
// ดึงชื่อทั้งหมด
NSArray *allNames = [employees valueForKeyPath:@"@unionOfObjects.name"];
NSLog(@"พนักงานทั้งหมด: %@", allNames);
// Output: ("Alice", "Bob", "Charlie", "Diana", "Eve")
```

---

## 34.5 Set Operators (สำหรับ Array ของ Arrays)

Set operators ทำงานกับ collection ของ collections

```objc
// สร้าง teams ที่แต่ละ team มีรายการ employees
NSArray *engineeringTeam = @[alice, bob];
NSArray *marketingTeam = @[charlie, eve];
NSArray *allTeams = @[engineeringTeam, marketingTeam];

// @distinctUnionOfArrays - รวมทุก arrays แล้วลบซ้ำ
NSArray *uniqueMembers = [allTeams valueForKeyPath:@"@distinctUnionOfArrays.self"];
NSLog(@"สมาชิกไม่ซ้ำทุกทีม: %lu คน", (unsigned long)uniqueMembers.count);

// @unionOfArrays - รวมทุก arrays ไม่ลบซ้ำ
NSArray *allMembers = [allTeams valueForKeyPath:@"@unionOfArrays.self"];
NSLog(@"สมาชิกทุกทีม: %lu คน", (unsigned long)allMembers.count);
```

---

## 34.6 KVC Compliance Requirements

เพื่อให้ object รองรับ KVC ต้องทำตามข้อกำหนดต่อไปนี้

### Accessor Methods Pattern

KVC ค้นหา accessor methods ตามลำดับต่อไปนี้:

```objc
// สำหรับ getValue ของ key "name":
// 1. getName    (ถ้าชื่อ key ขึ้นต้นด้วย get)
// 2. name       (ชื่อ key โดยตรง)
// 3. isName     (สำหรับ boolean)
// 4. _name      (instance variable โดยตรง)

@interface Person : NSObject {
    NSString *_name; // instance variable (ถ้าไม่มี accessor)
}

// accessor method ที่ KVC จะหา:
- (NSString *)name;      // ✅ จะพบ
- (NSString *)getName;   // ✅ จะพบ (get prefix)
- (void)setName:(NSString *)name; // ✅ สำหรับ setValue:

@end
```

### ตัวอย่าง Manual KVC Compliance

```objc
@interface Product : NSObject

// สิ่งที่จำเป็น:
// 1. ตั้งชื่อ ivar ตาม convention
// 2. มี getter ที่มีชื่อตรงกับ key
// 3. มี setter ที่มีชื่อ set<Key>:

- (NSString *)productName;
- (void)setProductName:(NSString *)name;
- (NSDecimalNumber *)price;
- (void)setPrice:(NSDecimalNumber *)price;

@end

@implementation Product {
    NSString *_productName;
    NSDecimalNumber *_price;
}

- (NSString *)productName {
    return _productName;
}

- (void)setProductName:(NSString *)name {
    _productName = [name copy];
}

- (NSDecimalNumber *)price {
    return _price;
}

- (void)setPrice:(NSDecimalNumber *)price {
    _price = price;
}

@end

// ทดสอบ
Product *product = [[Product alloc] init];
[product setValue:@"iPhone" forKey:@"productName"];
[product setValue:[NSDecimalNumber decimalNumberWithString:@"39900"] forKey:@"price"];

NSLog(@"ชื่อ: %@", [product valueForKey:@"productName"]);
NSLog(@"ราคา: %@", [product valueForKey:@"price"]);
```

### KVC กับ Instance Variables โดยตรง

```objc
// หาก object ไม่มี accessor methods
// KVC จะค้นหา instance variables โดยตรง

@interface MyObject : NSObject {
@public
    NSString *data; // accessible directly
}
@end

@implementation MyObject
// ไม่มี accessor methods
+ (BOOL)accessInstanceVariablesDirectly {
    return YES; // ค่าเริ่มต้น
    // คืนค่า NO เพื่อ disable การเข้าถึง ivar โดยตรง
}
@end

MyObject *obj = [[MyObject alloc] init];
[obj setValue:@"test data" forKey:@"data"];
NSLog(@"data: %@", [obj valueForKey:@"data"]);
```

---

## 34.7 Custom KVC Validation

KVC รองรับการ validate ค่าก่อนกำหนด ด้วย `validateValue:forKey:error:`

### การ Override Validation Methods

```objc
@interface Person : NSObject

@property (nonatomic, strong) NSString *name;
@property (nonatomic, assign) NSInteger age;
@property (nonatomic, strong) NSString *email;

// KVC validation methods (pattern: validate<Key>:error:)
- (BOOL)validateName:(id *)ioValue error:(NSError **)outError;
- (BOOL)validateAge:(id *)ioValue error:(NSError **)outError;
- (BOOL)validateEmail:(id *)ioValue error:(NSError **)outError;

@end

@implementation Person

- (BOOL)validateName:(id *)ioValue error:(NSError **)outError {
    NSString *name = *ioValue;
    
    // ตรวจสอบว่าไม่ว่างเปล่า
    if (!name || [name length] == 0) {
        if (outError) {
            *outError = [NSError errorWithDomain:@"PersonValidationDomain"
                                           code:100
                                       userInfo:@{NSLocalizedDescriptionKey: @"ชื่อต้องไม่ว่างเปล่า"}];
        }
        return NO;
    }
    
    // Trim whitespace
    NSString *trimmed = [name stringByTrimmingCharactersInSet:[NSCharacterSet whitespaceCharacterSet]];
    if (trimmed.length == 0) {
        if (outError) {
            *outError = [NSError errorWithDomain:@"PersonValidationDomain"
                                           code:101
                                       userInfo:@{NSLocalizedDescriptionKey: @"ชื่อต้องไม่ใช่ช่องว่างทั้งหมด"}];
        }
        return NO;
    }
    
    // แก้ไขค่า (modify ioValue) ถ้าต้องการ
    *ioValue = trimmed; // ส่งค่าที่ trim แล้วกลับไป
    return YES;
}

- (BOOL)validateAge:(id *)ioValue error:(NSError **)outError {
    NSNumber *age = *ioValue;
    
    if (!age) {
        if (outError) {
            *outError = [NSError errorWithDomain:@"PersonValidationDomain"
                                           code:200
                                       userInfo:@{NSLocalizedDescriptionKey: @"อายุต้องไม่เป็น nil"}];
        }
        return NO;
    }
    
    NSInteger ageValue = [age integerValue];
    
    if (ageValue < 0 || ageValue > 150) {
        if (outError) {
            *outError = [NSError errorWithDomain:@"PersonValidationDomain"
                                           code:201
                                       userInfo:@{NSLocalizedDescriptionKey: 
                                                  [NSString stringWithFormat:@"อายุต้องอยู่ระหว่าง 0-150 ปี (ได้รับ: %ld)", (long)ageValue]}];
        }
        return NO;
    }
    
    return YES;
}

- (BOOL)validateEmail:(id *)ioValue error:(NSError **)outError {
    NSString *email = *ioValue;
    
    if (!email) return YES; // อีเมลเป็น optional
    
    // ตรวจสอบรูปแบบอีเมลอย่างง่าย
    BOOL hasAtSign = [email containsString:@"@"];
    BOOL hasDot = [email containsString:@"."];
    
    if (!hasAtSign || !hasDot) {
        if (outError) {
            *outError = [NSError errorWithDomain:@"PersonValidationDomain"
                                           code:300
                                       userInfo:@{NSLocalizedDescriptionKey: @"รูปแบบอีเมลไม่ถูกต้อง"}];
        }
        return NO;
    }
    
    return YES;
}

@end
```

### การใช้งาน Validation

```objc
Person *person = [[Person alloc] init];

// Validate ก่อน set
NSError *error = nil;
id nameValue = @"  Alice  "; // มี whitespace

BOOL isValid = [person validateValue:&nameValue forKey:@"name" error:&error];

if (isValid) {
    person.name = nameValue; // nameValue ถูก trim แล้ว = "Alice"
    NSLog(@"ชื่อที่ valid: '%@'", person.name);
} else {
    NSLog(@"Validation ล้มเหลว: %@", error.localizedDescription);
}

// ทดสอบ age validation
NSError *ageError = nil;
id ageValue = @(-5); // อายุติดลบ

BOOL ageValid = [person validateValue:&ageValue forKey:@"age" error:&ageError];
if (!ageValid) {
    NSLog(@"Validation อายุล้มเหลว: %@", ageError.localizedDescription);
}

// validateAndSetValue:forKey: (convenience method ใน NSManagedObject)
// ต้องทำเอง:
void safeSetValue(id object, id value, NSString *key) {
    NSError *err = nil;
    id mutableValue = value;
    if ([object validateValue:&mutableValue forKey:key error:&err]) {
        [object setValue:mutableValue forKey:key];
        NSLog(@"กำหนดค่า %@ สำเร็จ", key);
    } else {
        NSLog(@"Validation ล้มเหลวสำหรับ %@: %@", key, err.localizedDescription);
    }
}
```

---

## 34.8 KVC กับ Dictionaries

KVC ทำงานได้ดีกับ NSDictionary เพราะ NSDictionary ตอบสนองต่อ KVC โดยธรรมชาติ

### NSDictionary กับ KVC

```objc
NSDictionary *personDict = @{
    @"name": @"สมชาย ใจดี",
    @"age": @30,
    @"email": @"somchai@example.com"
};

// valueForKey กับ Dictionary
NSString *name = [personDict valueForKey:@"name"];
NSLog(@"ชื่อ: %@", name);

// objectForKey vs valueForKey
// objectForKey: ทำงานเฉพาะกับ Dictionary
// valueForKey: รองรับ KVC ทั้งหมด (รวม operators)
NSString *name2 = [personDict objectForKey:@"name"]; // เหมือนกัน
```

### dictionaryWithValuesForKeys:

```objc
Person *person = [[Person alloc] init];
person.name = @"Alice";
person.age = 25;
person.email = @"alice@example.com";
person.city = @"Bangkok";

// ดึงหลาย values เป็น Dictionary ในครั้งเดียว
NSDictionary *values = [person dictionaryWithValuesForKeys:@[@"name", @"age", @"email"]];
NSLog(@"Values: %@", values);
// Output: {name: Alice, age: 25, email: alice@example.com}
```

### setValuesForKeysWithDictionary:

```objc
// กำหนดหลาย values จาก Dictionary ในครั้งเดียว
NSDictionary *updates = @{
    @"name": @"Bob Smith",
    @"age": @35,
    @"city": @"Chiang Mai"
};

[person setValuesForKeysWithDictionary:updates];
NSLog(@"ชื่อ: %@", person.name); // Bob Smith
NSLog(@"อายุ: %ld", (long)person.age); // 35
NSLog(@"เมือง: %@", person.city); // Chiang Mai
```

### ตัวอย่าง: Model Initializer จาก Dictionary

```objc
@interface Article : NSObject

@property (nonatomic, strong) NSString *title;
@property (nonatomic, strong) NSString *body;
@property (nonatomic, strong) NSString *authorName;
@property (nonatomic, strong) NSDate *publishedDate;
@property (nonatomic, assign) NSInteger viewCount;

- (instancetype)initWithDictionary:(NSDictionary *)dict;
- (NSDictionary *)toDictionary;

@end

@implementation Article

- (instancetype)initWithDictionary:(NSDictionary *)dict {
    self = [super init];
    if (self) {
        // กำหนดค่าทั้งหมดจาก dictionary ในครั้งเดียว
        NSArray *validKeys = @[@"title", @"body", @"authorName", @"viewCount"];
        NSDictionary *filteredDict = [dict dictionaryWithValuesForKeys:validKeys];
        
        // ใช้ setValuesForKeysWithDictionary:
        [self setValuesForKeysWithDictionary:filteredDict];
        
        // จัดการ publishedDate แยกต่างหาก (ต้องแปลง string -> date)
        if (dict[@"publishedDate"]) {
            NSDateFormatter *formatter = [[NSDateFormatter alloc] init];
            formatter.dateFormat = @"yyyy-MM-dd";
            self.publishedDate = [formatter dateFromString:dict[@"publishedDate"]];
        }
    }
    return self;
}

- (NSDictionary *)toDictionary {
    NSMutableDictionary *dict = [NSMutableDictionary dictionary];
    NSArray *keys = @[@"title", @"body", @"authorName", @"viewCount"];
    
    for (NSString *key in keys) {
        id value = [self valueForKey:key];
        if (value) {
            dict[key] = value;
        }
    }
    
    if (self.publishedDate) {
        NSDateFormatter *formatter = [[NSDateFormatter alloc] init];
        formatter.dateFormat = @"yyyy-MM-dd";
        dict[@"publishedDate"] = [formatter stringFromDate:self.publishedDate];
    }
    
    return [dict copy];
}

@end

// การใช้งาน
NSDictionary *articleData = @{
    @"title": @"Introduction to KVC",
    @"body": @"Key-Value Coding is a powerful mechanism...",
    @"authorName": @"Alice",
    @"viewCount": @1250,
    @"publishedDate": @"2024-01-15"
};

Article *article = [[Article alloc] initWithDictionary:articleData];
NSLog(@"บทความ: %@", article.title);
NSLog(@"โดย: %@", article.authorName);
NSLog(@"ยอดเข้าชม: %ld", (long)article.viewCount);

NSDictionary *exportedDict = [article toDictionary];
NSLog(@"Export: %@", exportedDict);
```

---

## 34.9 KVC กับ NSMutableDictionary

```objc
NSMutableDictionary *config = [NSMutableDictionary dictionary];

// setValue:forKey: บน NSMutableDictionary
[config setValue:@"MyApp" forKey:@"appName"];
[config setValue:@"1.0.0" forKey:@"version"];
[config setValue:@YES forKey:@"debugMode"];
[config setValue:@3000 forKey:@"serverPort"];

NSLog(@"App: %@", [config valueForKey:@"appName"]);
NSLog(@"Version: %@", [config valueForKey:@"version"]);
NSLog(@"Debug: %@", [config valueForKey:@"debugMode"]);

// setValue:forKey: กับ nil บน NSMutableDictionary จะ remove key นั้น
[config setValue:nil forKey:@"debugMode"];
NSLog(@"หลัง remove debugMode: %@", config);
```

---

## 34.10 ตัวอย่างจริง: Generic Data Mapper

```objc
@interface DataMapper : NSObject

// แปลง JSON dictionary เป็น Object ใดๆ ที่ KVC compliant
+ (id)mapDictionary:(NSDictionary *)dict toObject:(id)object;

// แปลง Array of Dictionaries เป็น Array of Objects
+ (NSArray *)mapArray:(NSArray *)array toClass:(Class)objectClass;

// ดึงค่าจาก path ที่ซับซ้อน
+ (id)valueFrom:(id)object atPath:(NSString *)path;

@end

@implementation DataMapper

+ (id)mapDictionary:(NSDictionary *)dict toObject:(id)object {
    for (NSString *key in dict) {
        @try {
            id value = dict[key];
            
            // แปลง NSNull เป็น nil
            if ([value isKindOfClass:[NSNull class]]) {
                value = nil;
            }
            
            // ตรวจสอบ validation ก่อน set
            NSError *error = nil;
            id mutableValue = value;
            if ([object validateValue:&mutableValue forKey:key error:&error]) {
                [object setValue:mutableValue forKey:key];
            } else {
                NSLog(@"Validation ล้มเหลว %@: %@", key, error.localizedDescription);
            }
        } @catch (NSException *e) {
            NSLog(@"ข้ามการ map key '%@': %@", key, e.reason);
        }
    }
    return object;
}

+ (NSArray *)mapArray:(NSArray *)array toClass:(Class)objectClass {
    NSMutableArray *result = [NSMutableArray array];
    for (NSDictionary *dict in array) {
        if ([dict isKindOfClass:[NSDictionary class]]) {
            id obj = [[objectClass alloc] init];
            [self mapDictionary:dict toObject:obj];
            [result addObject:obj];
        }
    }
    return [result copy];
}

+ (id)valueFrom:(id)object atPath:(NSString *)path {
    @try {
        return [object valueForKeyPath:path];
    } @catch (NSException *e) {
        NSLog(@"ไม่สามารถดึงค่าจาก path '%@': %@", path, e.reason);
        return nil;
    }
}

@end
```

```objc
// ตัวอย่าง JSON Response
NSArray *jsonData = @[
    @{@"name": @"Alice",   @"age": @25, @"email": @"alice@example.com"},
    @{@"name": @"Bob",     @"age": @30, @"email": @"bob@example.com"},
    @{@"name": @"Charlie", @"age": @35, @"email": @[NSNull null]}, // null email
];

// แปลงเป็น Array of Person objects
NSArray *persons = [DataMapper mapArray:jsonData toClass:[Person class]];

NSLog(@"จำนวนคน: %lu", (unsigned long)persons.count);
for (Person *p in persons) {
    NSLog(@"  %@, อายุ %ld", p.name, (long)p.age);
}

// ใช้ collection operators
NSNumber *avgAge = [persons valueForKeyPath:@"@avg.age"];
NSLog(@"อายุเฉลี่ย: %@", avgAge);

NSArray *names = [persons valueForKeyPath:@"@unionOfObjects.name"];
NSLog(@"ชื่อทั้งหมด: %@", names);
```

---

## 34.11 KVC Performance Considerations

```objc
// KVC มี overhead มากกว่าการ access โดยตรง
// ใช้ในกรณีที่ต้องการ flexibility เท่านั้น

// ❌ ไม่แนะนำ: ใช้ KVC ใน tight loop
for (int i = 0; i < 100000; i++) {
    NSString *name = [person valueForKey:@"name"]; // ช้ากว่า
}

// ✅ แนะนำ: เข้าถึงโดยตรง
for (int i = 0; i < 100000; i++) {
    NSString *name = person.name; // เร็วกว่า
}

// KVC เหมาะสำหรับ:
// 1. Dynamic property access ที่ key ไม่รู้จนกว่า runtime
// 2. Generic code ที่ทำงานกับ objects หลายประเภท
// 3. Serialization/Deserialization
// 4. Core Data queries
// 5. Interface Builder bindings (macOS)
```

---

## 34.12 ตัวอย่างจริง: Form Validation System

```objc
@interface FormValidator : NSObject

@property (nonatomic, strong) NSMutableDictionary *rules; // key -> validation block
@property (nonatomic, strong) NSMutableDictionary *errorMessages;

- (void)addRule:(BOOL (^)(id value))rule forKey:(NSString *)key errorMessage:(NSString *)message;
- (NSDictionary *)validateObject:(id)object withKeys:(NSArray *)keys;

@end

@implementation FormValidator

- (instancetype)init {
    self = [super init];
    if (self) {
        _rules = [NSMutableDictionary dictionary];
        _errorMessages = [NSMutableDictionary dictionary];
    }
    return self;
}

- (void)addRule:(BOOL (^)(id))rule forKey:(NSString *)key errorMessage:(NSString *)message {
    self.rules[key] = [rule copy];
    self.errorMessages[key] = message;
}

- (NSDictionary *)validateObject:(id)object withKeys:(NSArray *)keys {
    NSMutableDictionary *errors = [NSMutableDictionary dictionary];
    
    for (NSString *key in keys) {
        @try {
            id value = [object valueForKey:key]; // ใช้ KVC!
            
            BOOL (^rule)(id) = self.rules[key];
            if (rule && !rule(value)) {
                errors[key] = self.errorMessages[key] ?: @"Invalid value";
            }
        } @catch (NSException *e) {
            errors[key] = @"Key not found";
        }
    }
    
    return [errors copy];
}

@end

// การใช้งาน
FormValidator *validator = [[FormValidator alloc] init];

// เพิ่ม rules
[validator addRule:^BOOL(id value) {
    return value != nil && [value length] >= 2;
} forKey:@"name" errorMessage:@"ชื่อต้องมีอย่างน้อย 2 ตัวอักษร"];

[validator addRule:^BOOL(id value) {
    if (!value) return NO;
    NSInteger age = [value integerValue];
    return age >= 18 && age <= 100;
} forKey:@"age" errorMessage:@"อายุต้องระหว่าง 18-100 ปี"];

[validator addRule:^BOOL(id value) {
    if (!value) return YES; // optional
    return [value containsString:@"@"];
} forKey:@"email" errorMessage:@"อีเมลไม่ถูกต้อง"];

// Validate
Person *person = [[Person alloc] init];
person.name = @"A"; // สั้นเกินไป
person.age = 15;    // น้อยเกินไป
person.email = @"invalid-email"; // ไม่มี @

NSDictionary *errors = [validator validateObject:person 
                                        withKeys:@[@"name", @"age", @"email"]];

if (errors.count > 0) {
    NSLog(@"พบข้อผิดพลาดใน form:");
    for (NSString *key in errors) {
        NSLog(@"  %@: %@", key, errors[key]);
    }
} else {
    NSLog(@"Form valid!");
}
```

---

## แบบฝึกหัด

### แบบฝึกหัดที่ 1: KVC-Based Configuration Manager
สร้าง ConfigurationManager ที่:
- เก็บ configuration ใน dictionary
- รองรับ nested keys ด้วย key paths
- Validate ค่าก่อน set
- รองรับ default values

```objc
ConfigurationManager *config = [[ConfigurationManager alloc] init];
[config setValue:@"MyApp" forKeyPath:@"app.name"];
[config setValue:@3000 forKeyPath:@"server.port"];
[config setValue:@YES forKeyPath:@"features.darkMode"];

NSString *appName = [config valueForKeyPath:@"app.name"];
```

### แบบฝึกหัดที่ 2: Report Generator
สร้าง Report Generator ที่ใช้ KVC collection operators วิเคราะห์ข้อมูล:

```objc
// ข้อมูลนักเรียน
NSArray *students = [...]; // นักเรียนหลายคน

// คำนวณโดยใช้ KVC operators:
// - คะแนนเฉลี่ยของทุกวิชา
// - คะแนนสูงสุดและต่ำสุด
// - จำนวนนักเรียนที่สอบผ่าน/ตก
// - รายชื่อนักเรียนที่เรียนดีที่สุด
```

### แบบฝึกหัดที่ 3: Generic Object Comparison
สร้างฟังก์ชันที่เปรียบเทียบสอง objects โดยใช้ KVC:

```objc
NSDictionary *differences = [ObjectComparator 
                              compareObject:obj1 
                              withObject:obj2 
                              forKeys:@[@"name", @"age", @"email"]];
// Result: {age: {old: 25, new: 30}, email: {old: ..., new: ...}}
```

### แบบฝึกหัดที่ 4: Data Export System
สร้างระบบ export ที่ใช้ KVC แปลงข้อมูล:

```objc
// Export เป็น CSV
NSString *csv = [DataExporter exportObjects:employees 
                                    withKeys:@[@"name", @"salary", @"department"]
                                    format:DataExportFormatCSV];

// Export เป็น JSON
NSString *json = [DataExporter exportObjects:employees 
                                     withKeys:@[@"name", @"salary", @"department"]
                                      format:DataExportFormatJSON];
```

---

## สรุป

Key-Value Coding เป็นกลไกที่ทรงพลังใน Objective-C ที่ช่วยให้:

1. **Dynamic Access** - เข้าถึง properties โดยใช้ string key ที่รู้ตอน runtime
2. **Nested Access** - ใช้ key paths เพื่อเข้าถึง nested properties
3. **Collection Operators** - คำนวณสถิติจาก collections ได้อย่างกระชับ
4. **Bulk Operations** - `dictionaryWithValuesForKeys:` และ `setValuesForKeysWithDictionary:` 
5. **Validation** - validate ค่าก่อน set

KVC เป็นรากฐานของ KVO, Core Data, และ Bindings ดังนั้นการเข้าใจ KVC อย่างถ่องแท้จะทำให้ใช้เทคโนโลยีเหล่านี้ได้อย่างมีประสิทธิภาพมากขึ้น
