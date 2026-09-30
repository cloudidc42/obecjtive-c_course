# Part 01: บทนำสู่ Objective-C (Introduction to Objective-C)

---

## สารบัญ (Table of Contents)

1. ประวัติและที่มาของ Objective-C
2. ทำไมต้องเรียน Objective-C
3. การติดตั้ง Development Environment
4. โครงสร้างโปรแกรม Objective-C แรก
5. การคอมไพล์และรันโปรแกรม
6. ความแตกต่างจาก C และ C++
7. ไฟล์ .h และ .m
8. การใช้ #import และ #include
9. NSLog สำหรับ Output
10. Foundation Framework เบื้องต้น
11. ตัวอย่างโปรแกรม Hello World แบบสมบูรณ์
12. แบบฝึกหัดพร้อมเฉลย

---

## 1. ประวัติและที่มาของ Objective-C

### กำเนิดของ Objective-C

Objective-C ถูกสร้างขึ้นในช่วงต้นทศวรรษ 1980 โดย **Brad Cox** และ **Tom Love** ผ่านบริษัท Productivity Products International (PPI) ซึ่งต่อมาเปลี่ยนชื่อเป็น Stepstone

**Brad Cox** นักวิทยาศาสตร์คอมพิวเตอร์ชาวอเมริกัน ได้รับแรงบันดาลใจจากภาษา Smalltalk ซึ่งเป็นภาษาแบบ Object-Oriented ที่พัฒนาโดย Xerox PARC เขาต้องการนำแนวคิด Object-Oriented Programming (OOP) มาผสมผสานกับภาษา C ที่มีประสิทธิภาพสูง

### Timeline ที่สำคัญ

```
ปี 1981 - Brad Cox และ Tom Love เริ่มพัฒนา Objective-C
ปี 1983 - เผยแพร่ Objective-C เวอร์ชันแรกอย่างเป็นทางการ
ปี 1984 - Steve Jobs ออกจาก Apple และก่อตั้ง NeXT Computer
ปี 1988 - NeXT ซื้อสิทธิ์ใช้งาน Objective-C จาก Stepstone
ปี 1989 - NeXT เปิดตัว NeXTSTEP OS ที่ใช้ Objective-C เป็น primary language
ปี 1994 - NeXT ปล่อย OpenStep specification
ปี 1996 - Apple ซื้อกิจการ NeXT ด้วยมูลค่า 429 ล้านดอลลาร์
ปี 1997 - Steve Jobs กลับมา Apple พร้อม Objective-C และ NeXTSTEP
ปี 2001 - Mac OS X เปิดตัว ใช้ Objective-C เป็นภาษาหลัก
ปี 2007 - iPhone SDK เปิดตัว นักพัฒนาใช้ Objective-C เขียน iOS apps
ปี 2008 - App Store เปิดตัว Objective-C เป็น king ของ iOS development
ปี 2011 - Objective-C 2.0 พร้อม ARC (Automatic Reference Counting)
ปี 2014 - Apple เปิดตัว Swift ภาษาใหม่แทน Objective-C
ปี 2015 - Swift กลายเป็น open source
ปี 2023 - ยังคงมี Objective-C ใน legacy codebase จำนวนมาก
```

### NeXT Computer และบทบาทสำคัญ

เมื่อ Steve Jobs ออกจาก Apple ในปี 1985 เขาก่อตั้ง **NeXT Computer** และเลือก Objective-C เป็นภาษาหลักสำหรับการพัฒนา NeXTSTEP OS ระบบปฏิบัติการนี้ถือว่าล้ำสมัยมากในยุคนั้น:

- มี **Interface Builder** ซึ่งเป็นต้นแบบของ Xcode Interface Builder ปัจจุบัน
- มี **AppKit** framework สำหรับสร้าง GUI
- มี **Foundation Kit** สำหรับ data structures และ utilities
- รองรับ **Display PostScript** สำหรับ high-quality graphics

เมื่อ Apple ซื้อ NeXT ในปี 1996 พวกเขาได้รับ:
1. Objective-C runtime และ compiler
2. NeXTSTEP frameworks ที่กลายเป็น Cocoa และ Cocoa Touch
3. Steve Jobs กลับมาเป็น CEO ของ Apple
4. วิสัยทัศน์ใหม่สำหรับ Mac OS X

### Smalltalk Influence

Objective-C ได้รับอิทธิพลอย่างมากจาก Smalltalk โดยเฉพาะในเรื่อง:

- **Message Passing**: การส่ง message ไปยัง object แทนการเรียก method โดยตรง
- **Dynamic Dispatch**: การตัดสินใจว่าจะเรียก method ไหนในระหว่าง runtime
- **Introspection**: ความสามารถในการตรวจสอบ class และ method ใน runtime

```objc
// Smalltalk style message passing ใน Objective-C
// [receiver message]
[myObject doSomething];
[myObject setName:@"Hello" age:25];

// เทียบกับ C++ style
myObject.doSomething();
myObject.setName("Hello", 25);
```

---

## 2. ทำไมต้องเรียน Objective-C

### เหตุผลที่ 1: Legacy Code มหาศาล

ปัจจุบันยังมี Objective-C code จำนวนมหาศาลในโลก iOS/macOS development:

- แอปพลิเคชันที่เปิดตัวก่อนปี 2014 ส่วนใหญ่เขียนด้วย Objective-C
- บริษัทใหญ่ๆ เช่น Facebook, Twitter, LinkedIn มี codebase Objective-C ขนาดใหญ่
- Framework และ library จำนวนมากยังคงเป็น Objective-C
- iOS SDK ยังมี Objective-C header files อยู่

### เหตุผลที่ 2: เข้าใจรากฐานของ Swift

Swift ถูกออกแบบมาให้ทำงานร่วมกับ Objective-C ได้อย่างสมบูรณ์:

```swift
// Swift code สามารถเรียกใช้ Objective-C class ได้โดยตรง
import Foundation

let str = NSString(string: "Hello from Swift!")
let array = NSMutableArray()
array.add("item1")
array.add("item2")
```

การเข้าใจ Objective-C ช่วยให้:
- อ่านและแก้ไข legacy code ได้
- ใช้ Objective-C frameworks ใน Swift projects
- เข้าใจ bridging header
- Debug issues ที่เกิดจาก ObjC runtime

### เหตุผลที่ 3: Runtime Flexibility

Objective-C มี dynamic runtime ที่ทรงพลัง:

```objc
// Method Swizzling - เปลี่ยน implementation ของ method ใน runtime
#import <objc/runtime.h>

void swizzleMethod(Class class, SEL originalSel, SEL swizzledSel) {
    Method originalMethod = class_getInstanceMethod(class, originalSel);
    Method swizzledMethod = class_getInstanceMethod(class, swizzledSel);
    method_exchangeImplementations(originalMethod, swizzledMethod);
}
```

### เหตุผลที่ 4: ตลาดงาน

แม้ Swift จะเป็น primary language แต่ยังมีความต้องการ Objective-C:
- บริษัทที่มี legacy codebase
- Maintaining เดิม apps
- Cross-platform libraries ที่ต้องการ ObjC interface

### เหตุผลที่ 5: เข้าใจ Memory Management ลึกซึ้ง

Objective-C สอนให้เข้าใจ memory management อย่างลึกซึ้ง:

```objc
// Manual Reference Counting (MRC) - เข้าใจ retain/release
NSObject *obj = [[NSObject alloc] init]; // retain count = 1
[obj retain];  // retain count = 2
[obj release]; // retain count = 1
[obj release]; // retain count = 0, object deallocated

// ARC ทำสิ่งนี้อัตโนมัติ แต่ knowing how it works helps debugging
```

---

## 3. การติดตั้ง Development Environment

### Option 1: Xcode บน macOS (แนะนำ)

**ความต้องการของระบบ:**
- macOS 12 Monterey หรือสูงกว่า
- พื้นที่ว่างบน disk อย่างน้อย 15 GB
- RAM อย่างน้อย 8 GB (แนะนำ 16 GB)

**ขั้นตอนการติดตั้ง Xcode:**

1. เปิด **App Store** บน macOS
2. ค้นหา "Xcode"
3. กด **Get** หรือ **Install**
4. รอการดาวน์โหลด (ประมาณ 10-12 GB)
5. เปิด Xcode ครั้งแรกและยอมรับ Terms of Service
6. Xcode จะติดตั้ง Command Line Tools อัตโนมัติ

**ตรวจสอบการติดตั้ง:**

```bash
# เปิด Terminal และรันคำสั่งนี้
xcode-select --version
# ควรแสดง: xcode-select version 2395 (หรือสูงกว่า)

gcc --version
# ควรแสดง: Apple clang version ...

clang --version
# ควรแสดง: Apple clang version ...
```

### Option 2: Command Line Tools เท่านั้น

สำหรับผู้ที่ต้องการแค่ compiler ไม่ต้องการ full IDE:

```bash
# ติดตั้ง Command Line Tools
xcode-select --install

# หน้าต่างจะปรากฏขึ้น กด Install และรอ
```

### Option 3: GNUstep บน Linux/Windows

สำหรับผู้ใช้ Linux หรือ Windows:

```bash
# Ubuntu/Debian
sudo apt-get update
sudo apt-get install gnustep gnustep-devel gobjc

# ตรวจสอบ
gnustep-config --version
```

**การคอมไพล์บน GNUstep:**

```bash
# Compile Objective-C file
gcc -o hello hello.m $(gnustep-config --objc-flags) $(gnustep-config --objc-libs)

# หรือใช้ make
make -f GNUmakefile
```

### การสร้าง Project ใหม่ใน Xcode

1. เปิด Xcode
2. เลือก **File > New > Project** (หรือ Cmd+Shift+N)
3. เลือก **macOS > Command Line Tool**
4. กด **Next**
5. ตั้งชื่อ Product Name: "HelloWorld"
6. Organization Identifier: "com.yourname"
7. Language: **Objective-C**
8. กด **Next** และเลือก folder ที่ต้องการบันทึก

### การสร้าง Single File ใน Terminal

```bash
# สร้าง file ใหม่
touch hello.m

# เปิดด้วย text editor
nano hello.m
# หรือ
vim hello.m
# หรือ
open -a TextEdit hello.m
```

---

## 4. โครงสร้างโปรแกรม Objective-C แรก

### Hello World - เวอร์ชันพื้นฐาน

```objc
// File: hello.m
// โปรแกรม Hello World พื้นฐานที่สุด

#import <Foundation/Foundation.h>

int main(int argc, const char * argv[]) {
    @autoreleasepool {
        NSLog(@"Hello, World!");
    }
    return 0;
}
```

### อธิบายทีละบรรทัด

```objc
// 1. #import <Foundation/Foundation.h>
// - #import คือคำสั่งนำเข้า Framework
// - Foundation/Foundation.h คือ header file หลักของ Foundation Framework
// - Foundation ให้ NSLog, NSString, NSArray, NSDictionary และอื่นๆ
// - ใช้ <> สำหรับ system frameworks, ใช้ "" สำหรับ local files

#import <Foundation/Foundation.h>

// 2. int main(int argc, const char * argv[])
// - จุดเริ่มต้นของทุกโปรแกรม C/Objective-C
// - argc = argument count (จำนวน command-line arguments)
// - argv = argument values (array ของ argument strings)
// - return type int หมายถึงส่งค่ากลับ integer ให้ OS
// - return 0 หมายถึงโปรแกรมสำเร็จ

int main(int argc, const char * argv[]) {
    
    // 3. @autoreleasepool { ... }
    // - เป็น Objective-C construct สำหรับ memory management
    // - objects ที่ถูก autorelease จะถูก deallocated เมื่อออกจาก block นี้
    // - จำเป็นสำหรับ NSLog และ Foundation objects
    // - ARC จัดการ memory ให้อัตโนมัติ แต่ยังต้องมี autoreleasepool
    
    @autoreleasepool {
        
        // 4. NSLog(@"Hello, World!");
        // - NSLog คือ function สำหรับ print ข้อความออกสู่ console
        // - @ นำหน้า string literal สร้าง NSString object
        // - "Hello, World!" คือ string ที่ต้องการแสดง
        // - NSLog เพิ่ม timestamp และ process name ให้อัตโนมัติ
        
        NSLog(@"Hello, World!");
    }
    
    // 5. return 0;
    // - ส่งค่า 0 กลับให้ OS
    // - 0 หมายถึงสำเร็จ, ค่าอื่นหมายถึงมี error
    
    return 0;
}
```

### ผลลัพธ์ของโปรแกรม

```
2024-01-15 10:30:45.123 HelloWorld[1234:56789] Hello, World!
```

**อธิบาย output format:**
- `2024-01-15 10:30:45.123` = timestamp
- `HelloWorld` = ชื่อโปรแกรม
- `[1234:56789]` = process ID และ thread ID
- `Hello, World!` = ข้อความที่แสดง

### โครงสร้างขยาย - การใช้ Variables

```objc
#import <Foundation/Foundation.h>

int main(int argc, const char * argv[]) {
    @autoreleasepool {
        // ประกาศตัวแปรและแสดงผล
        NSString *name = @"สมชาย";
        int age = 25;
        double height = 175.5;
        
        NSLog(@"ชื่อ: %@", name);
        NSLog(@"อายุ: %d ปี", age);
        NSLog(@"ส่วนสูง: %.1f ซม.", height);
        
        // การรวม string
        NSString *greeting = [NSString stringWithFormat:@"สวัสดี, %@!", name];
        NSLog(@"%@", greeting);
    }
    return 0;
}
```

**ผลลัพธ์:**
```
2024-01-15 10:30:45.123 HelloWorld[1234:56789] ชื่อ: สมชาย
2024-01-15 10:30:45.124 HelloWorld[1234:56789] อายุ: 25 ปี
2024-01-15 10:30:45.124 HelloWorld[1234:56789] ส่วนสูง: 175.5 ซม.
2024-01-15 10:30:45.124 HelloWorld[1234:56789] สวัสดี, สมชาย!
```

---

## 5. การคอมไพล์และรันโปรแกรม

### วิธีที่ 1: ผ่าน Xcode

1. เปิด project ใน Xcode
2. กด **Cmd+B** เพื่อ Build
3. กด **Cmd+R** เพื่อ Run
4. ดู output ใน **Debug area** (ด้านล่างของ Xcode)

### วิธีที่ 2: ผ่าน Command Line

```bash
# สร้างไฟล์ hello.m ก่อน
cat > hello.m << 'EOF'
#import <Foundation/Foundation.h>

int main(int argc, const char * argv[]) {
    @autoreleasepool {
        NSLog(@"Hello, World!");
    }
    return 0;
}
EOF

# คอมไพล์ด้วย clang
clang -framework Foundation hello.m -o hello

# รันโปรแกรม
./hello
```

**Output:**
```
2024-01-15 10:30:45.123 hello[1234:56789] Hello, World!
```

### ทำความเข้าใจ Compiler Flags

```bash
# -framework Foundation: ใช้ Foundation framework
clang -framework Foundation hello.m -o hello

# -Wall: แสดง warning ทั้งหมด
clang -framework Foundation -Wall hello.m -o hello

# -Wextra: แสดง extra warnings
clang -framework Foundation -Wall -Wextra hello.m -o hello

# -g: เพิ่ม debug information
clang -framework Foundation -g hello.m -o hello

# -O2: optimize code (level 2)
clang -framework Foundation -O2 hello.m -o hello

# -fobjc-arc: เปิดใช้ ARC (Automatic Reference Counting)
clang -framework Foundation -fobjc-arc hello.m -o hello
```

### Makefile สำหรับโปรเจค Objective-C

```makefile
# Makefile
CC = clang
CFLAGS = -Wall -Wextra -fobjc-arc
FRAMEWORKS = -framework Foundation
TARGET = hello
SOURCES = hello.m

all: $(TARGET)

$(TARGET): $(SOURCES)
	$(CC) $(CFLAGS) $(FRAMEWORKS) $(SOURCES) -o $(TARGET)

clean:
	rm -f $(TARGET)

run: $(TARGET)
	./$(TARGET)

.PHONY: all clean run
```

**การใช้ Makefile:**
```bash
make          # คอมไพล์
make run      # คอมไพล์และรัน
make clean    # ลบไฟล์ที่คอมไพล์แล้ว
```

### การ Debug ด้วย LLDB

```bash
# คอมไพล์พร้อม debug info
clang -framework Foundation -g hello.m -o hello

# เริ่ม LLDB
lldb hello

# ใน LLDB console
(lldb) breakpoint set --name main     # ตั้ง breakpoint ที่ main
(lldb) run                            # รันโปรแกรม
(lldb) step                           # step through code
(lldb) print argc                     # แสดงค่า variable
(lldb) continue                       # continue จนจบ
(lldb) quit                           # ออก
```

---

## 6. ความแตกต่างจาก C และ C++

### Objective-C vs C

| Feature | C | Objective-C |
|---------|---|-------------|
| Object-Oriented | ไม่มี | มี (class, object, message) |
| String Literals | "hello" | @"hello" (NSString) |
| Memory Management | Manual | ARC หรือ Manual |
| Nil Handling | NULL pointer crash | nil safe |
| Dynamic Typing | ไม่มี | id type |
| Categories | ไม่มี | มี (extend existing class) |
| Protocols | ไม่มี | มี (เหมือน interface) |

```c
// C - ไม่มี OOP
#include <stdio.h>

typedef struct {
    char name[50];
    int age;
} Person;

void greet(Person *p) {
    printf("Hello, %s! You are %d years old.\n", p->name, p->age);
}

int main() {
    Person p = {"Somchai", 25};
    greet(&p);
    return 0;
}
```

```objc
// Objective-C - มี OOP
#import <Foundation/Foundation.h>

@interface Person : NSObject
@property (nonatomic, strong) NSString *name;
@property (nonatomic, assign) NSInteger age;
- (void)greet;
@end

@implementation Person
- (void)greet {
    NSLog(@"Hello, %@! You are %ld years old.", self.name, (long)self.age);
}
@end

int main(int argc, const char * argv[]) {
    @autoreleasepool {
        Person *p = [[Person alloc] init];
        p.name = @"Somchai";
        p.age = 25;
        [p greet];
    }
    return 0;
}
```

### Objective-C vs C++

| Feature | C++ | Objective-C |
|---------|-----|-------------|
| Syntax สำหรับ method call | obj.method() | [obj method] |
| Multiple inheritance | มี | ไม่มี (ใช้ Protocols แทน) |
| Templates | มี | ไม่มี (ใช้ id และ generics) |
| Runtime | Static dispatch | Dynamic dispatch |
| Exception handling | try/catch | @try/@catch |
| String | std::string | NSString |
| Boolean | bool | BOOL |

```cpp
// C++ style
#include <iostream>
#include <string>

class Animal {
public:
    std::string name;
    
    Animal(std::string n) : name(n) {}
    
    virtual void speak() {
        std::cout << name << " makes a sound." << std::endl;
    }
};

class Dog : public Animal {
public:
    Dog(std::string n) : Animal(n) {}
    
    void speak() override {
        std::cout << name << " says: Woof!" << std::endl;
    }
};

int main() {
    Dog *d = new Dog("Rex");
    d->speak();
    delete d;
    return 0;
}
```

```objc
// Objective-C style
#import <Foundation/Foundation.h>

@interface Animal : NSObject
@property (nonatomic, strong) NSString *name;
- (instancetype)initWithName:(NSString *)name;
- (void)speak;
@end

@implementation Animal
- (instancetype)initWithName:(NSString *)name {
    self = [super init];
    if (self) {
        _name = name;
    }
    return self;
}

- (void)speak {
    NSLog(@"%@ makes a sound.", self.name);
}
@end

@interface Dog : Animal
@end

@implementation Dog
- (void)speak {
    NSLog(@"%@ says: Woof!", self.name);
}
@end

int main(int argc, const char * argv[]) {
    @autoreleasepool {
        Dog *d = [[Dog alloc] initWithName:@"Rex"];
        [d speak];
    }
    return 0;
}
```

### Message Passing vs Function Call

นี่คือความแตกต่างที่สำคัญที่สุดระหว่าง Objective-C กับภาษาอื่น:

```objc
// Objective-C: Message Passing
// Syntax: [receiver message]
// Syntax: [receiver messageWithArg:value]

NSString *str = @"Hello";
NSUInteger length = [str length];      // ส่ง message "length" ไปยัง str
NSString *upper = [str uppercaseString]; // ส่ง message "uppercaseString"

// Method กับ Arguments หลายตัว
NSString *result = [str stringByReplacingOccurrencesOfString:@"Hello" 
                                                   withString:@"Goodbye"];

// Nested messages (อ่านจากใน -> นอก)
NSString *trimmed = [[str stringByTrimmingCharactersInSet:
                      [NSCharacterSet whitespaceCharacterSet]] 
                     uppercaseString];
```

```objc
// ความปลอดภัยของ nil
NSString *nilString = nil;
NSUInteger nilLength = [nilString length]; // ไม่ crash! ส่งคืน 0
NSLog(@"Length: %lu", (unsigned long)nilLength); // Output: Length: 0

// ใน C++ การเรียก method บน null pointer จะ crash
// SomeClass *ptr = nullptr;
// ptr->someMethod(); // CRASH!
```

---

## 7. ไฟล์ .h (Header) และ .m (Implementation)

### บทบาทของแต่ละไฟล์

**Header File (.h):**
- ประกาศ interface ของ class (สิ่งที่ class นั้นทำได้)
- ประกาศ properties, methods, instance variables
- ไฟล์ที่ผู้อื่นจะ #import เพื่อใช้ class ของคุณ
- เหมือน "สัญญา" ว่า class นั้นมีอะไรให้ใช้

**Implementation File (.m):**
- เขียน code จริงของ methods ที่ประกาศใน .h
- มี implementation details ที่ไม่จำเป็นต้องเปิดเผย
- ชื่อไฟล์ควรตรงกับ .h (เช่น Person.h และ Person.m)

### ตัวอย่าง: Class Person

**Person.h:**
```objc
// Person.h
// Header file - ประกาศ interface

#import <Foundation/Foundation.h>

// ประกาศ class Person ที่สืบทอดจาก NSObject
@interface Person : NSObject

// Properties - สมบัติของ Person
@property (nonatomic, strong) NSString *firstName;   // ชื่อ
@property (nonatomic, strong) NSString *lastName;    // นามสกุล
@property (nonatomic, assign) NSInteger age;          // อายุ
@property (nonatomic, strong) NSString *email;        // อีเมล

// Instance Methods - method ที่เรียกใช้กับ object
- (instancetype)initWithFirstName:(NSString *)firstName 
                         lastName:(NSString *)lastName 
                              age:(NSInteger)age;

- (NSString *)fullName;          // คืนค่าชื่อเต็ม
- (BOOL)isAdult;                 // ตรวจสอบว่าเป็นผู้ใหญ่หรือไม่
- (void)printInfo;               // แสดงข้อมูล

// Class Methods - method ที่เรียกใช้กับ class
+ (Person *)personWithFirstName:(NSString *)firstName 
                       lastName:(NSString *)lastName 
                            age:(NSInteger)age;

+ (NSInteger)minimumAdultAge;    // อายุขั้นต่ำของผู้ใหญ่

@end
```

**Person.m:**
```objc
// Person.m
// Implementation file - เขียน code จริง

#import "Person.h"

@implementation Person

// Designated Initializer
- (instancetype)initWithFirstName:(NSString *)firstName 
                         lastName:(NSString *)lastName 
                              age:(NSInteger)age {
    // เรียก super's init ก่อน
    self = [super init];
    if (self) {
        // ตั้งค่า properties
        _firstName = firstName;  // ใช้ _propertyName เพื่อ access backing store โดยตรง
        _lastName = lastName;
        _age = age;
    }
    return self;
}

// คืนค่าชื่อเต็ม
- (NSString *)fullName {
    return [NSString stringWithFormat:@"%@ %@", self.firstName, self.lastName];
}

// ตรวจสอบว่าเป็นผู้ใหญ่หรือไม่
- (BOOL)isAdult {
    return self.age >= [Person minimumAdultAge];
}

// แสดงข้อมูล
- (void)printInfo {
    NSLog(@"ชื่อ: %@", [self fullName]);
    NSLog(@"อายุ: %ld ปี", (long)self.age);
    NSLog(@"สถานะ: %@", [self isAdult] ? @"ผู้ใหญ่" : @"เยาวชน");
    if (self.email) {
        NSLog(@"อีเมล: %@", self.email);
    }
}

// Class method - สร้าง Person object
+ (Person *)personWithFirstName:(NSString *)firstName 
                       lastName:(NSString *)lastName 
                            age:(NSInteger)age {
    return [[self alloc] initWithFirstName:firstName 
                                  lastName:lastName 
                                       age:age];
}

// Class method - อายุขั้นต่ำของผู้ใหญ่
+ (NSInteger)minimumAdultAge {
    return 18;
}

// Override description สำหรับ NSLog(@"%@", person)
- (NSString *)description {
    return [NSString stringWithFormat:@"Person(name: %@, age: %ld)", 
            [self fullName], (long)self.age];
}

@end
```

**main.m:**
```objc
// main.m
// ไฟล์หลักที่รันโปรแกรม

#import <Foundation/Foundation.h>
#import "Person.h"  // import Person class

int main(int argc, const char * argv[]) {
    @autoreleasepool {
        
        // สร้าง Person objects หลายวิธี
        
        // วิธีที่ 1: ใช้ alloc/init
        Person *person1 = [[Person alloc] initWithFirstName:@"สมชาย" 
                                                   lastName:@"ใจดี" 
                                                        age:25];
        
        // วิธีที่ 2: ใช้ class method (convenience constructor)
        Person *person2 = [Person personWithFirstName:@"สมหญิง" 
                                            lastName:@"งามดี" 
                                                 age:17];
        person2.email = @"somying@example.com";
        
        // แสดงข้อมูล
        NSLog(@"=== ข้อมูลบุคคล ===");
        [person1 printInfo];
        
        NSLog(@"");
        NSLog(@"=== ข้อมูลบุคคล ===");
        [person2 printInfo];
        
        // ใช้ description
        NSLog(@"\nPerson 1: %@", person1);
        NSLog(@"Person 2: %@", person2);
        
        // ตรวจสอบ isAdult
        NSLog(@"\n%@ เป็นผู้ใหญ่: %@", 
              person1.firstName, 
              [person1 isAdult] ? @"YES" : @"NO");
        NSLog(@"%@ เป็นผู้ใหญ่: %@", 
              person2.firstName, 
              [person2 isAdult] ? @"YES" : @"NO");
    }
    return 0;
}
```

**ผลลัพธ์:**
```
=== ข้อมูลบุคคล ===
ชื่อ: สมชาย ใจดี
อายุ: 25 ปี
สถานะ: ผู้ใหญ่

=== ข้อมูลบุคคล ===
ชื่อ: สมหญิง งามดี
อายุ: 17 ปี
สถานะ: เยาวชน
อีเมล: somying@example.com

Person 1: Person(name: สมชาย ใจดี, age: 25)
Person 2: Person(name: สมหญิง งามดี, age: 17)

สมชาย เป็นผู้ใหญ่: YES
สมหญิง เป็นผู้ใหญ่: NO
```

---

## 8. การใช้ #import และ #include

### ความแตกต่างระหว่าง #import และ #include

```objc
// #include (C style) - อาจ include ไฟล์เดิมซ้ำหลายครั้ง
// ต้องใช้ include guards เพื่อป้องกัน
#ifndef MYFILE_H
#define MYFILE_H
// ... code ...
#endif

// #import (Objective-C style) - include แต่ละไฟล์เพียงครั้งเดียวอัตโนมัติ
// ไม่ต้องใช้ include guards
#import "MyFile.h"  // ปลอดภัย import ซ้ำกี่ครั้งก็ได้
```

### รูปแบบการ import

```objc
// System/Framework headers - ใช้ < >
#import <Foundation/Foundation.h>    // Foundation Framework
#import <UIKit/UIKit.h>              // UIKit Framework (iOS)
#import <AppKit/AppKit.h>            // AppKit Framework (macOS)
#import <CoreData/CoreData.h>        // Core Data Framework
#import <CoreLocation/CoreLocation.h> // Core Location Framework

// Local/Project headers - ใช้ " "
#import "Person.h"          // ไฟล์ในโปรเจคเดียวกัน
#import "Models/User.h"     // ไฟล์ใน subfolder
#import "Helpers/Utils.h"   // ไฟล์ utility

// Umbrella header (import หลายไฟล์ในครั้งเดียว)
// สร้างไฟล์ MyProject.h ที่ import ทุกอย่าง
// MyProject.h:
// #import "Person.h"
// #import "Animal.h"
// #import "Vehicle.h"
// จากนั้นใน main.m import แค่:
// #import "MyProject.h"
```

### Forward Declaration (@class)

เมื่อ header file สองไฟล์อ้างถึงกันเอง (circular dependency):

```objc
// Person.h
#import <Foundation/Foundation.h>

// ใช้ @class แทน #import เพื่อหลีกเลี่ยง circular dependency
@class Company;  // แจ้งว่า Company เป็น class โดยไม่ต้อง import

@interface Person : NSObject
@property (nonatomic, strong) NSString *name;
@property (nonatomic, weak) Company *employer;  // ใช้ weak เพื่อหลีกเลี่ยง retain cycle
@end
```

```objc
// Company.h
#import <Foundation/Foundation.h>
@class Person;  // forward declaration

@interface Company : NSObject
@property (nonatomic, strong) NSString *name;
@property (nonatomic, strong) NSMutableArray<Person *> *employees;
@end
```

```objc
// Person.m - ที่นี่ import ไฟล์จริงได้
#import "Person.h"
#import "Company.h"  // OK ที่จะ import ใน .m file

@implementation Person
// implementation...
@end
```

### Modules (@import)

Objective-C รองรับ module system แบบใหม่:

```objc
// แบบเก่า
#import <Foundation/Foundation.h>
#import <UIKit/UIKit.h>

// แบบใหม่ (Modules) - เร็วกว่าและแนะนำให้ใช้
@import Foundation;
@import UIKit;
@import CoreData;
@import CoreLocation;
```

ใน Xcode โดย default จะเปิด **Enable Modules** ไว้แล้ว ทำให้ `#import <Foundation/Foundation.h>` ถูก convert เป็น `@import Foundation` อัตโนมัติ

---

## 9. NSLog สำหรับ Output

### Format Specifiers พื้นฐาน

```objc
#import <Foundation/Foundation.h>

int main(int argc, const char * argv[]) {
    @autoreleasepool {
        
        // %@ - NSObject (NSString, NSArray, NSDictionary, custom objects)
        NSString *name = @"สมชาย";
        NSLog(@"ชื่อ: %@", name);
        
        // %d, %i - int
        int count = 42;
        NSLog(@"จำนวน: %d", count);
        
        // %ld - long
        long bigNum = 1234567890L;
        NSLog(@"ตัวเลขใหญ่: %ld", bigNum);
        
        // %lu - unsigned long
        NSUInteger uCount = 100;
        NSLog(@"จำนวนไม่มีเครื่องหมาย: %lu", (unsigned long)uCount);
        
        // %f - float/double
        float price = 9.99f;
        NSLog(@"ราคา: %f", price);
        
        // %.2f - 2 decimal places
        double pi = 3.14159265358979;
        NSLog(@"Pi: %.2f", pi);        // 3.14
        NSLog(@"Pi: %.4f", pi);        // 3.1416
        NSLog(@"Pi: %.8f", pi);        // 3.14159265
        
        // %e - scientific notation
        double small = 0.000001234;
        NSLog(@"Scientific: %e", small);  // 1.234000e-06
        
        // %c - char
        char letter = 'A';
        NSLog(@"ตัวอักษร: %c", letter);
        
        // %s - C string
        const char *cStr = "C String";
        NSLog(@"C String: %s", cStr);
        
        // %p - pointer address
        int x = 10;
        NSLog(@"Address of x: %p", &x);
        
        // %% - literal % sign
        NSLog(@"Discount: 50%%");
        
        // %x, %X - hexadecimal
        int hex = 255;
        NSLog(@"Hex lower: %x", hex);   // ff
        NSLog(@"Hex upper: %X", hex);   // FF
        
        // %o - octal
        NSLog(@"Octal: %o", hex);       // 377
        
        // %u - unsigned int
        unsigned int uInt = 4294967295U;
        NSLog(@"Unsigned: %u", uInt);
        
        // BOOL
        BOOL isTrue = YES;
        NSLog(@"Boolean: %@", isTrue ? @"YES" : @"NO");
        // หรือ
        NSLog(@"Boolean int: %d", isTrue);  // แสดงเป็น 1 หรือ 0
        
    }
    return 0;
}
```

### การจัดรูปแบบขั้นสูง

```objc
#import <Foundation/Foundation.h>

int main(int argc, const char * argv[]) {
    @autoreleasepool {
        
        // Width formatting
        int num = 42;
        NSLog(@"|%10d|", num);     // |        42|  (right aligned, width 10)
        NSLog(@"|%-10d|", num);    // |42        |  (left aligned, width 10)
        NSLog(@"|%010d|", num);    // |0000000042|  (zero padded, width 10)
        
        // Float precision
        double value = 1234.5678;
        NSLog(@"|%15.2f|", value);   // |        1234.57|
        NSLog(@"|%-15.2f|", value);  // |1234.57        |
        
        // Multiple values
        NSString *product = @"กาแฟ";
        int quantity = 3;
        double price = 45.0;
        double total = quantity * price;
        
        NSLog(@"%-20s %5s %8s %10s", "สินค้า", "จำนวน", "ราคา", "รวม");
        NSLog(@"%-20@ %5d %8.2f %10.2f", product, quantity, price, total);
        
        // Arrays and Dictionaries - %@ ใช้ description method
        NSArray *fruits = @[@"แอปเปิ้ล", @"กล้วย", @"ส้ม"];
        NSLog(@"ผลไม้: %@", fruits);
        // Output: ผลไม้: (
        //     แอปเปิ้ล,
        //     กล้วย,
        //     ส้ม
        // )
        
        NSDictionary *scores = @{@"คณิตศาสตร์": @95, @"ภาษาไทย": @88};
        NSLog(@"คะแนน: %@", scores);
        
    }
    return 0;
}
```

### NSLog vs printf

```objc
#import <Foundation/Foundation.h>
#include <stdio.h>

int main(int argc, const char * argv[]) {
    @autoreleasepool {
        
        // NSLog - มี timestamp, process info, thread info
        NSLog(@"Hello from NSLog");
        // 2024-01-15 10:30:45.123 program[1234:56789] Hello from NSLog
        
        // printf - แบบ C, ไม่มี timestamp, ไม่มี newline อัตโนมัติ
        printf("Hello from printf\n");
        // Hello from printf
        
        // NSLog เหมาะสำหรับ debugging
        // printf เหมาะสำหรับ user-facing output
        
        // หมายเหตุ: NSLog ไม่มีประสิทธิภาพสำหรับ production
        // ใน production ควรใช้ os_log
        
        // os_log (iOS 10+, macOS 10.12+) - แนะนำสำหรับ production
        // #import <os/log.h>
        // os_log(OS_LOG_DEFAULT, "Hello from os_log");
        
    }
    return 0;
}
```

---

## 10. Foundation Framework เบื้องต้น

### Foundation คืออะไร

Foundation Framework เป็น base framework ของ iOS/macOS development ประกอบด้วย:

- **Data Types**: NSString, NSNumber, NSDate, NSData
- **Collections**: NSArray, NSDictionary, NSSet
- **Mutable Collections**: NSMutableArray, NSMutableDictionary, NSMutableSet
- **File System**: NSFileManager, NSFileHandle
- **Networking**: NSURLSession, NSURL, NSURLRequest
- **Threading**: NSThread, NSOperationQueue, GCD
- **Utilities**: NSNotificationCenter, NSUserDefaults, NSBundle

### NS Prefix

NS ย่อมาจาก **NeXTSTEP** - เป็น prefix ที่ใช้ใน Foundation และ AppKit/UIKit

```objc
// ตัวอย่าง NS classes
NSString      // String
NSNumber      // Number (int, float, double, BOOL)
NSArray       // Immutable array
NSMutableArray // Mutable array
NSDictionary  // Immutable dictionary (key-value)
NSMutableDictionary // Mutable dictionary
NSSet         // Immutable set
NSDate        // Date and time
NSData        // Raw bytes
NSURL         // URL
NSError       // Error information
NSLog         // Logging function (ไม่ใช่ class)
```

### NSString - String ใน Objective-C

```objc
#import <Foundation/Foundation.h>

int main(int argc, const char * argv[]) {
    @autoreleasepool {
        
        // สร้าง NSString
        NSString *str1 = @"Hello, World!";
        NSString *str2 = [NSString stringWithFormat:@"ยินดีต้อนรับ %@", @"สมชาย"];
        NSString *str3 = [[NSString alloc] initWithString:@"Allocated string"];
        
        // ความยาว
        NSUInteger length = [str1 length];
        NSLog(@"ความยาว: %lu", (unsigned long)length);
        
        // เปรียบเทียบ (ใช้ isEqualToString ไม่ใช่ ==)
        NSString *a = @"Hello";
        NSString *b = @"Hello";
        if ([a isEqualToString:b]) {
            NSLog(@"เหมือนกัน");
        }
        
        // เชื่อมต่อ string
        NSString *firstName = @"สมชาย";
        NSString *lastName = @"ใจดี";
        NSString *fullName = [firstName stringByAppendingString:@" "];
        fullName = [fullName stringByAppendingString:lastName];
        NSLog(@"ชื่อเต็ม: %@", fullName);
        
        // แปลงเป็น uppercase/lowercase
        NSLog(@"Upper: %@", [str1 uppercaseString]);
        NSLog(@"Lower: %@", [str1 lowercaseString]);
        
        // ตรวจสอบ substring
        if ([str1 containsString:@"World"]) {
            NSLog(@"มี 'World' ใน string");
        }
        
        // ค้นหา range
        NSRange range = [str1 rangeOfString:@"World"];
        if (range.location != NSNotFound) {
            NSLog(@"พบ 'World' ที่ตำแหน่ง: %lu", (unsigned long)range.location);
        }
        
        // แทนที่ substring
        NSString *replaced = [str1 stringByReplacingOccurrencesOfString:@"World" 
                                                             withString:@"Objective-C"];
        NSLog(@"แทนที่แล้ว: %@", replaced);
        
        // ตัด whitespace
        NSString *padded = @"  Hello  ";
        NSString *trimmed = [padded stringByTrimmingCharactersInSet:
                             [NSCharacterSet whitespaceCharacterSet]];
        NSLog(@"ตัดแล้ว: '%@'", trimmed);
        
        // Split string
        NSString *csv = @"แอปเปิ้ล,กล้วย,ส้ม,มะม่วง";
        NSArray *fruits = [csv componentsSeparatedByString:@","];
        NSLog(@"ผลไม้: %@", fruits);
        
        // แปลงเป็น Number
        NSString *numStr = @"42";
        NSInteger num = [numStr integerValue];
        NSLog(@"ตัวเลข: %ld", (long)num);
        
        double dNum = [@"3.14" doubleValue];
        NSLog(@"Double: %.2f", dNum);
        
    }
    return 0;
}
```

### NSArray - Array ใน Objective-C

```objc
#import <Foundation/Foundation.h>

int main(int argc, const char * argv[]) {
    @autoreleasepool {
        
        // สร้าง NSArray (Immutable)
        NSArray *fruits = @[@"แอปเปิ้ล", @"กล้วย", @"ส้ม"];
        NSArray *numbers = @[@1, @2, @3, @4, @5];
        
        // ความยาว
        NSLog(@"จำนวน: %lu", (unsigned long)fruits.count);
        
        // เข้าถึง element
        NSLog(@"ตัวแรก: %@", fruits[0]);           // Subscript syntax
        NSLog(@"ตัวแรก: %@", [fruits objectAtIndex:0]); // Traditional
        NSLog(@"ตัวสุดท้าย: %@", fruits.lastObject);
        
        // วนซ้ำ
        for (NSString *fruit in fruits) {
            NSLog(@"- %@", fruit);
        }
        
        // ตรวจสอบ
        if ([fruits containsObject:@"กล้วย"]) {
            NSLog(@"มีกล้วย!");
        }
        
        // หา index
        NSUInteger index = [fruits indexOfObject:@"ส้ม"];
        NSLog(@"ส้มอยู่ที่ index: %lu", (unsigned long)index);
        
        // NSMutableArray (เพิ่ม/ลบ element ได้)
        NSMutableArray *mutableFruits = [NSMutableArray arrayWithArray:fruits];
        [mutableFruits addObject:@"มะม่วง"];
        [mutableFruits insertObject:@"องุ่น" atIndex:1];
        [mutableFruits removeObject:@"กล้วย"];
        [mutableFruits removeObjectAtIndex:0];
        
        NSLog(@"Mutable fruits: %@", mutableFruits);
        
        // Sort
        NSArray *sorted = [fruits sortedArrayUsingSelector:@selector(compare:)];
        NSLog(@"เรียงแล้ว: %@", sorted);
        
    }
    return 0;
}
```

### NSDictionary - Dictionary ใน Objective-C

```objc
#import <Foundation/Foundation.h>

int main(int argc, const char * argv[]) {
    @autoreleasepool {
        
        // สร้าง NSDictionary
        NSDictionary *person = @{
            @"name": @"สมชาย ใจดี",
            @"age": @25,
            @"city": @"กรุงเทพฯ"
        };
        
        // เข้าถึงค่า
        NSLog(@"ชื่อ: %@", person[@"name"]);
        NSLog(@"อายุ: %@", person[@"age"]);
        
        // ตรวจสอบ key
        if (person[@"email"] == nil) {
            NSLog(@"ไม่มี email");
        }
        
        // วนซ้ำ
        for (NSString *key in person) {
            NSLog(@"%@: %@", key, person[key]);
        }
        
        // NSMutableDictionary
        NSMutableDictionary *settings = [NSMutableDictionary dictionary];
        settings[@"theme"] = @"dark";
        settings[@"fontSize"] = @14;
        settings[@"language"] = @"th";
        
        // เพิ่ม/แก้ไข
        settings[@"theme"] = @"light";  // แก้ไข
        
        // ลบ
        [settings removeObjectForKey:@"language"];
        
        NSLog(@"Settings: %@", settings);
        
    }
    return 0;
}
```

---

## 11. ตัวอย่างโปรแกรม Hello World แบบสมบูรณ์

ต่อไปนี้เป็นโปรแกรมตัวอย่างที่สมบูรณ์พร้อม comments อธิบายทุก concept:

```objc
// =============================================================================
// File: main.m
// Project: HelloWorldComplete
// Description: โปรแกรม Hello World แบบสมบูรณ์พร้อม Foundation Framework
// =============================================================================

// นำเข้า Foundation Framework ซึ่งให้ NSLog, NSString, NSArray ฯลฯ
#import <Foundation/Foundation.h>

// ==============================
// การประกาศ Class (Interface)
// ==============================

/**
 * Class Greeter - จัดการการทักทาย
 * สืบทอดจาก NSObject ซึ่งเป็น base class ของทุก Objective-C object
 */
@interface Greeter : NSObject

// Property - ชื่อของผู้ใช้
@property (nonatomic, strong) NSString *userName;

// Designated initializer - วิธีสร้าง Greeter ที่แนะนำ
- (instancetype)initWithName:(NSString *)name;

// Methods
- (void)greetInEnglish;
- (void)greetInThai;
- (void)greetInJapanese;
- (void)greetAllLanguages;
- (NSString *)personalizedMessage;

// Class method - factory method
+ (Greeter *)greeterWithName:(NSString *)name;

@end

// ==============================
// การ Implement Class
// ==============================

@implementation Greeter

/**
 * Designated Initializer
 * เป็น initializer หลักที่ทำงานจริง
 */
- (instancetype)initWithName:(NSString *)name {
    // เรียก super init ก่อนเสมอ - เป็น convention ของ Objective-C
    self = [super init];
    if (self) {
        // ใช้ _propertyName เพื่อ set backing variable โดยตรง
        // ไม่ใช้ self.propertyName ใน init เพราะอาจมีปัญหากับ subclass
        _userName = name ? name : @"Guest";  // Default to "Guest" if nil
    }
    return self;
}

/**
 * ทักทายภาษาอังกฤษ
 */
- (void)greetInEnglish {
    NSLog(@"Hello, %@! Welcome to Objective-C programming!", self.userName);
}

/**
 * ทักทายภาษาไทย
 */
- (void)greetInThai {
    NSLog(@"สวัสดีครับ/ค่ะ คุณ%@! ยินดีต้อนรับสู่การเขียนโปรแกรม Objective-C!", 
          self.userName);
}

/**
 * ทักทายภาษาญี่ปุ่น
 */
- (void)greetInJapanese {
    NSLog(@"こんにちは、%@さん！Objective-Cプログラミングへようこそ！", self.userName);
}

/**
 * ทักทายทุกภาษา
 */
- (void)greetAllLanguages {
    NSLog(@"===== การทักทาย =====");
    [self greetInEnglish];
    [self greetInThai];
    [self greetInJapanese];
    NSLog(@"====================");
}

/**
 * สร้างข้อความส่วนตัว
 * ใช้ NSDate เพื่อหาเวลาปัจจุบัน
 */
- (NSString *)personalizedMessage {
    // ดึง current date/time
    NSDate *now = [NSDate date];
    NSCalendar *calendar = [NSCalendar currentCalendar];
    NSInteger hour = [calendar component:NSCalendarUnitHour fromDate:now];
    
    // เลือกคำทักทายตามเวลา
    NSString *timeGreeting;
    if (hour >= 5 && hour < 12) {
        timeGreeting = @"Good morning / สวัสดีตอนเช้า";
    } else if (hour >= 12 && hour < 17) {
        timeGreeting = @"Good afternoon / สวัสดีตอนบ่าย";
    } else if (hour >= 17 && hour < 21) {
        timeGreeting = @"Good evening / สวัสดีตอนเย็น";
    } else {
        timeGreeting = @"Good night / ราตรีสวัสดิ์";
    }
    
    return [NSString stringWithFormat:@"%@, คุณ%@!", timeGreeting, self.userName];
}

/**
 * Factory method - วิธี create Greeter แบบสั้น
 */
+ (Greeter *)greeterWithName:(NSString *)name {
    // สร้างและส่งคืน Greeter object
    return [[self alloc] initWithName:name];
}

/**
 * Override description สำหรับการแสดงผลด้วย NSLog(@"%@", greeter)
 */
- (NSString *)description {
    return [NSString stringWithFormat:@"<Greeter: userName=%@>", self.userName];
}

@end

// ==============================
// Main Function
// ==============================

int main(int argc, const char * argv[]) {
    
    // @autoreleasepool - จัดการ memory ของ autorelease objects
    @autoreleasepool {
        
        // ===== ส่วนที่ 1: โปรแกรม Hello World พื้นฐาน =====
        NSLog(@"");
        NSLog(@"========================================");
        NSLog(@"   ยินดีต้อนรับสู่ Objective-C!");
        NSLog(@"========================================");
        NSLog(@"");
        
        // NSLog พื้นฐาน
        NSLog(@"Hello, World!");
        NSLog(@"สวัสดีโลก!");
        
        // ===== ส่วนที่ 2: การใช้ Variables =====
        NSLog(@"\n----- Variables -----");
        
        NSString *language = @"Objective-C";
        NSInteger year = 1983;
        NSString *creator = @"Brad Cox";
        
        NSLog(@"ภาษา: %@", language);
        NSLog(@"ปีที่สร้าง: %ld", (long)year);
        NSLog(@"ผู้สร้าง: %@", creator);
        NSLog(@"%@ ถูกสร้างในปี %ld โดย %@", language, (long)year, creator);
        
        // ===== ส่วนที่ 3: การใช้ Class Greeter =====
        NSLog(@"\n----- Using Greeter Class -----");
        
        // สร้าง Greeter objects
        Greeter *greeter1 = [[Greeter alloc] initWithName:@"สมชาย"];
        Greeter *greeter2 = [Greeter greeterWithName:@"Alice"];
        Greeter *greeter3 = [Greeter greeterWithName:nil]; // จะใช้ "Guest"
        
        // ทดสอบ description
        NSLog(@"greeter1: %@", greeter1);
        
        // ทักทายหลายภาษา
        [greeter1 greetAllLanguages];
        
        NSLog(@"");
        [greeter2 greetInEnglish];
        
        NSLog(@"");
        NSLog(@"%@", [greeter1 personalizedMessage]);
        NSLog(@"%@", [greeter3 personalizedMessage]);
        
        // ===== ส่วนที่ 4: NSArray =====
        NSLog(@"\n----- NSArray -----");
        
        NSArray *programmingLanguages = @[
            @"Objective-C",
            @"Swift",
            @"C",
            @"C++",
            @"Java",
            @"Python"
        ];
        
        NSLog(@"ภาษาโปรแกรมมิ่ง %lu ภาษา:", (unsigned long)programmingLanguages.count);
        for (NSUInteger i = 0; i < programmingLanguages.count; i++) {
            NSLog(@"  %lu. %@", (unsigned long)(i + 1), programmingLanguages[i]);
        }
        
        // Fast enumeration (แนะนำ)
        NSLog(@"\nด้วย Fast Enumeration:");
        for (NSString *lang in programmingLanguages) {
            NSLog(@"  - %@", lang);
        }
        
        // ===== ส่วนที่ 5: NSDictionary =====
        NSLog(@"\n----- NSDictionary -----");
        
        NSDictionary *info = @{
            @"ชื่อโปรแกรม": @"Hello World Complete",
            @"เวอร์ชัน": @"1.0.0",
            @"ผู้เขียน": @"Objective-C Student",
            @"ภาษา": @"Objective-C",
            @"ปีที่เขียน": @2024
        };
        
        NSLog(@"ข้อมูลโปรแกรม:");
        for (NSString *key in info) {
            NSLog(@"  %@: %@", key, info[key]);
        }
        
        // ===== ส่วนที่ 6: NSNumber =====
        NSLog(@"\n----- NSNumber -----");
        
        NSNumber *intNum = @42;
        NSNumber *floatNum = @3.14f;
        NSNumber *doubleNum = @3.14159265358979;
        NSNumber *boolNum = @YES;
        
        NSLog(@"Integer: %@", intNum);
        NSLog(@"Float: %@", floatNum);
        NSLog(@"Double: %@", doubleNum);
        NSLog(@"Boolean: %@", boolNum);
        
        // แปลงเป็น primitive
        int iVal = [intNum intValue];
        float fVal = [floatNum floatValue];
        double dVal = [doubleNum doubleValue];
        BOOL bVal = [boolNum boolValue];
        
        NSLog(@"As primitives: %d, %.2f, %.4f, %@", 
              iVal, fVal, dVal, bVal ? @"YES" : @"NO");
        
        // ===== ส่วนที่ 7: Command Line Arguments =====
        NSLog(@"\n----- Command Line Arguments -----");
        NSLog(@"จำนวน arguments: %d", argc);
        for (int i = 0; i < argc; i++) {
            NSLog(@"  argv[%d]: %s", i, argv[i]);
        }
        
        // ===== สรุป =====
        NSLog(@"\n========================================");
        NSLog(@"   โปรแกรมทำงานเสร็จสิ้นสมบูรณ์!");
        NSLog(@"========================================");
        
    } // สิ้นสุด @autoreleasepool - memory ถูก release ที่นี่
    
    return 0; // ส่งคืน 0 = สำเร็จ
}
```

---

## 12. แบบฝึกหัดพร้อมเฉลย

### แบบฝึกหัดที่ 1: Hello with Variables

**โจทย์:** สร้างโปรแกรมที่แสดงข้อมูลส่วนตัวของคุณ (ชื่อ, อายุ, เมืองที่อยู่, ภาษาโปรดที่ชอบเขียน)

**เฉลย:**
```objc
#import <Foundation/Foundation.h>

int main(int argc, const char * argv[]) {
    @autoreleasepool {
        // ข้อมูลส่วนตัว
        NSString *name = @"สมชาย ใจดี";
        NSInteger age = 22;
        NSString *city = @"เชียงใหม่";
        NSString *favLanguage = @"Objective-C";
        
        // แสดงผล
        NSLog(@"===== ข้อมูลส่วนตัว =====");
        NSLog(@"ชื่อ     : %@", name);
        NSLog(@"อายุ     : %ld ปี", (long)age);
        NSLog(@"เมือง    : %@", city);
        NSLog(@"ภาษาโปรด : %@", favLanguage);
        NSLog(@"=========================");
        NSLog(@"\nสวัสดี! ฉันชื่อ%@ อายุ%ldปี อยู่ที่%@ ชอบเขียน%@", 
              name, (long)age, city, favLanguage);
    }
    return 0;
}
```

### แบบฝึกหัดที่ 2: Simple Calculator Info

**โจทย์:** สร้างโปรแกรมแสดงผลการคำนวณพื้นฐาน (บวก, ลบ, คูณ, หาร) ของตัวเลขสองตัว

**เฉลย:**
```objc
#import <Foundation/Foundation.h>

int main(int argc, const char * argv[]) {
    @autoreleasepool {
        double a = 15.0;
        double b = 4.0;
        
        NSLog(@"===== เครื่องคิดเลขพื้นฐาน =====");
        NSLog(@"ตัวเลข a = %.1f", a);
        NSLog(@"ตัวเลข b = %.1f", b);
        NSLog(@"--------------------------------");
        NSLog(@"a + b = %.1f", a + b);
        NSLog(@"a - b = %.1f", a - b);
        NSLog(@"a * b = %.1f", a * b);
        NSLog(@"a / b = %.4f", a / b);
        
        // ตรวจสอบการหารด้วยศูนย์
        if (b != 0) {
            NSLog(@"a %% b = %.1f", fmod(a, b));  // modulo สำหรับ double ใช้ fmod
        } else {
            NSLog(@"ไม่สามารถหารด้วยศูนย์ได้!");
        }
        
        NSLog(@"================================");
    }
    return 0;
}
```

### แบบฝึกหัดที่ 3: Student Grade

**โจทย์:** สร้าง Class Student ที่มี:
- Properties: name, score (0-100)
- Method: grade ที่คืนค่าเกรด (A, B, C, D, F)
- Method: printReport ที่แสดงผลคะแนนและเกรด

**เฉลย:**

**Student.h:**
```objc
#import <Foundation/Foundation.h>

@interface Student : NSObject
@property (nonatomic, strong) NSString *name;
@property (nonatomic, assign) NSInteger score;

- (instancetype)initWithName:(NSString *)name score:(NSInteger)score;
- (NSString *)grade;
- (void)printReport;
+ (Student *)studentWithName:(NSString *)name score:(NSInteger)score;
@end
```

**Student.m:**
```objc
#import "Student.h"

@implementation Student

- (instancetype)initWithName:(NSString *)name score:(NSInteger)score {
    self = [super init];
    if (self) {
        _name = name;
        _score = MAX(0, MIN(100, score)); // clamp ระหว่าง 0-100
    }
    return self;
}

- (NSString *)grade {
    if (self.score >= 80) return @"A";
    if (self.score >= 70) return @"B";
    if (self.score >= 60) return @"C";
    if (self.score >= 50) return @"D";
    return @"F";
}

- (void)printReport {
    NSLog(@"===== รายงานผล =====");
    NSLog(@"ชื่อนักเรียน: %@", self.name);
    NSLog(@"คะแนน: %ld / 100", (long)self.score);
    NSLog(@"เกรด: %@", [self grade]);
    
    NSString *status = [self.score >= 50] ? @"ผ่าน" : @"ไม่ผ่าน";
    NSLog(@"สถานะ: %@", status);
    NSLog(@"===================");
}

+ (Student *)studentWithName:(NSString *)name score:(NSInteger)score {
    return [[self alloc] initWithName:name score:score];
}

@end
```

**main.m:**
```objc
#import <Foundation/Foundation.h>
#import "Student.h"

int main(int argc, const char * argv[]) {
    @autoreleasepool {
        NSArray *students = @[
            [Student studentWithName:@"สมชาย" score:85],
            [Student studentWithName:@"สมหญิง" score:72],
            [Student studentWithName:@"วิชัย" score:58],
            [Student studentWithName:@"นิตยา" score:45],
            [Student studentWithName:@"ประเสริฐ" score:91]
        ];
        
        for (Student *student in students) {
            [student printReport];
            NSLog(@"");
        }
        
        // หาคะแนนเฉลี่ย
        NSInteger total = 0;
        for (Student *student in students) {
            total += student.score;
        }
        double average = (double)total / students.count;
        NSLog(@"คะแนนเฉลี่ยทั้งห้อง: %.1f", average);
    }
    return 0;
}
```

### แบบฝึกหัดที่ 4: Foundation Collections

**โจทย์:** สร้างโปรแกรมที่:
1. สร้าง array ของชื่อผลไม้ 5 ชนิด
2. เรียงลำดับตัวอักษร
3. แสดงผลพร้อม index
4. หาผลไม้ที่ขึ้นต้นด้วยตัวอักษร "ก"

**เฉลย:**
```objc
#import <Foundation/Foundation.h>

int main(int argc, const char * argv[]) {
    @autoreleasepool {
        
        // 1. สร้าง array ผลไม้
        NSArray *fruits = @[@"กล้วย", @"มะม่วง", @"แอปเปิ้ล", @"กระท้อน", @"ส้ม"];
        
        NSLog(@"ผลไม้ทั้งหมด:");
        for (NSUInteger i = 0; i < fruits.count; i++) {
            NSLog(@"  %lu. %@", (unsigned long)(i + 1), fruits[i]);
        }
        
        // 2. เรียงลำดับ
        NSArray *sorted = [fruits sortedArrayUsingSelector:@selector(compare:)];
        NSLog(@"\nเรียงลำดับตัวอักษร:");
        for (NSUInteger i = 0; i < sorted.count; i++) {
            NSLog(@"  %lu. %@", (unsigned long)(i + 1), sorted[i]);
        }
        
        // 4. หาผลไม้ที่ขึ้นต้นด้วย "ก"
        NSMutableArray *kFruits = [NSMutableArray array];
        for (NSString *fruit in fruits) {
            if ([fruit hasPrefix:@"ก"]) {
                [kFruits addObject:fruit];
            }
        }
        
        NSLog(@"\nผลไม้ที่ขึ้นต้นด้วย 'ก':");
        for (NSString *fruit in kFruits) {
            NSLog(@"  - %@", fruit);
        }
        
        NSLog(@"\nพบทั้งหมด %lu ชนิด", (unsigned long)kFruits.count);
        
    }
    return 0;
}
```

---

## บทสรุป

ในบทนี้เราได้เรียนรู้:

1. **ประวัติ Objective-C** - เกิดจาก Brad Cox และ Tom Love ปี 1983 ได้รับอิทธิพลจาก Smalltalk
2. **ทำไมต้องเรียน** - Legacy code, Swift foundation, Dynamic runtime
3. **การติดตั้ง** - Xcode เป็น IDE หลัก, สามารถ compile จาก command line ได้
4. **โครงสร้างโปรแกรม** - #import, @autoreleasepool, NSLog
5. **การคอมไพล์** - ใช้ clang พร้อม -framework Foundation flag
6. **ความแตกต่างจาก C/C++** - Message passing, dynamic dispatch, nil safety
7. **ไฟล์ .h/.m** - Header สำหรับ declaration, Implementation สำหรับ code จริง
8. **#import** - ปลอดภัยกว่า #include, รองรับ forward declaration
9. **NSLog** - Output function พร้อม format specifiers
10. **Foundation Framework** - NSString, NSArray, NSDictionary และอื่นๆ

## บทต่อไป

ใน **Part 02** เราจะเรียนรู้เรื่อง:
- ประเภทข้อมูลทั้งหมดใน Objective-C
- การประกาศและใช้งานตัวแปร
- NSString, NSInteger, CGFloat
- Constants และ Type Casting
- nil, NULL, Nil ความแตกต่าง
- BOOL และ bool

---

*เอกสารนี้เป็นส่วนหนึ่งของหลักสูตร Objective-C Programming*
*Part 01 of 100*
