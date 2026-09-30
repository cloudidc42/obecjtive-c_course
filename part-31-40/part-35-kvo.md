# ตอนที่ 35: Key-Value Observing (KVO)

## บทนำ

Key-Value Observing (KVO) เป็นกลไกใน Objective-C ที่ช่วยให้ object หนึ่งสามารถ "สังเกต" (observe) การเปลี่ยนแปลงของ property ใน object อื่นได้ เมื่อ property มีการเปลี่ยนแปลง ระบบจะแจ้งเตือน observer โดยอัตโนมัติ

KVO สร้างอยู่บน KVC (Key-Value Coding) และเป็นรูปแบบหนึ่งของ Observer Pattern ที่ใช้กันอย่างแพร่หลายใน Cocoa framework

## KVO คืออะไร?

```objc
// สมมุติ Person มี property 'score'
// เราต้องการรู้ทุกครั้งที่ score เปลี่ยน

// โดยไม่ใช้ KVO:
// person.score = 100;
// -> ต้องมีการเรียก update UI แยกต่างหาก

// ใช้ KVO:
// ลงทะเบียน observer ครั้งเดียว
[person addObserver:self 
         forKeyPath:@"score" 
            options:NSKeyValueObservingOptionNew 
            context:NULL];

// ทุกครั้งที่ person.score เปลี่ยน จะเรียก observeValueForKeyPath:... อัตโนมัติ
```

---

## 35.1 addObserver:forKeyPath:options:context:

### พารามิเตอร์ทั้งหมด

```objc
[object addObserver:observer
         forKeyPath:keyPath
            options:options
            context:context];
```

- **object** - object ที่เราต้องการ observe
- **observer** - object ที่จะได้รับ notification (ต้อง implement `observeValueForKeyPath:`)
- **keyPath** - ชื่อ property ที่ต้องการ observe (รองรับ key paths)
- **options** - กำหนดข้อมูลใน change dictionary
- **context** - pointer ที่ใช้แยกแยะ observers (สำคัญมาก)

### NSKeyValueObservingOptions

```objc
// NSKeyValueObservingOptionNew - รับค่าใหม่หลัง change
NSKeyValueObservingOptionNew        = 0x01

// NSKeyValueObservingOptionOld - รับค่าเก่าก่อน change  
NSKeyValueObservingOptionOld        = 0x02

// NSKeyValueObservingOptionInitial - เรียก observeValueForKeyPath ทันทีที่ register
NSKeyValueObservingOptionInitial    = 0x04

// NSKeyValueObservingOptionPrior - เรียกก่อนและหลัง change
NSKeyValueObservingOptionPrior      = 0x08
```

### ตัวอย่างพื้นฐาน

```objc
#import <Foundation/Foundation.h>

@interface Counter : NSObject

@property (nonatomic, assign) NSInteger count;
@property (nonatomic, strong) NSString *status;

@end

@implementation Counter
@end
```

```objc
@interface CounterObserver : NSObject

- (void)startObservingCounter:(Counter *)counter;
- (void)stopObservingCounter:(Counter *)counter;

@end

@implementation CounterObserver

static void *CounterObserverContext = &CounterObserverContext;

- (void)startObservingCounter:(Counter *)counter {
    // ลงทะเบียน observe ทั้ง count และ status
    [counter addObserver:self
              forKeyPath:@"count"
                 options:NSKeyValueObservingOptionNew | NSKeyValueObservingOptionOld
                 context:CounterObserverContext];
    
    [counter addObserver:self
              forKeyPath:@"status"
                 options:NSKeyValueObservingOptionNew
                 context:CounterObserverContext];
}

- (void)stopObservingCounter:(Counter *)counter {
    [counter removeObserver:self forKeyPath:@"count" context:CounterObserverContext];
    [counter removeObserver:self forKeyPath:@"status" context:CounterObserverContext];
}

- (void)observeValueForKeyPath:(NSString *)keyPath
                      ofObject:(id)object
                        change:(NSDictionary<NSKeyValueChangeKey, id> *)change
                       context:(void *)context {
    
    // ตรวจสอบว่าเป็น notification ของเราหรือไม่
    if (context != CounterObserverContext) {
        [super observeValueForKeyPath:keyPath ofObject:object change:change context:context];
        return;
    }
    
    if ([keyPath isEqualToString:@"count"]) {
        NSInteger newCount = [change[NSKeyValueChangeNewKey] integerValue];
        NSInteger oldCount = [change[NSKeyValueChangeOldKey] integerValue];
        NSLog(@"Count เปลี่ยนจาก %ld เป็น %ld", (long)oldCount, (long)newCount);
        
    } else if ([keyPath isEqualToString:@"status"]) {
        NSString *newStatus = change[NSKeyValueChangeNewKey];
        NSLog(@"Status เปลี่ยนเป็น: %@", newStatus);
    }
}

@end
```

```objc
// การใช้งาน
Counter *counter = [[Counter alloc] init];
CounterObserver *observer = [[CounterObserver alloc] init];

[observer startObservingCounter:counter];

counter.count = 1;    // จะ trigger notification
counter.count = 5;    // จะ trigger notification
counter.status = @"active"; // จะ trigger notification
counter.count = 10;   // จะ trigger notification

[observer stopObservingCounter:counter];
```

---

## 35.2 observeValueForKeyPath:ofObject:change:context:

### Change Dictionary Keys

```objc
// ใน change dictionary มี keys ต่อไปนี้:
// NSKeyValueChangeKindKey       - ประเภทของการเปลี่ยนแปลง
// NSKeyValueChangeNewKey        - ค่าใหม่ (ถ้าใช้ NSKeyValueObservingOptionNew)
// NSKeyValueChangeOldKey        - ค่าเก่า (ถ้าใช้ NSKeyValueObservingOptionOld)
// NSKeyValueChangeIndexesKey    - indices ที่เปลี่ยน (สำหรับ array)
// NSKeyValueChangeNotificationIsPriorKey - เป็น prior notification หรือไม่
```

### NSKeyValueChange (ประเภทการเปลี่ยนแปลง)

```objc
typedef NS_ENUM(NSUInteger, NSKeyValueChange) {
    NSKeyValueChangeSetting     = 1, // กำหนดค่าใหม่
    NSKeyValueChangeInsertion   = 2, // เพิ่มใน collection
    NSKeyValueChangeRemoval     = 3, // ลบจาก collection
    NSKeyValueChangeReplacement = 4, // แทนที่ใน collection
};
```

### ตัวอย่างครบถ้วน

```objc
- (void)observeValueForKeyPath:(NSString *)keyPath
                      ofObject:(id)object
                        change:(NSDictionary<NSKeyValueChangeKey, id> *)change
                       context:(void *)context {
    
    if (context != MyObserverContext) {
        // ส่งต่อให้ super จัดการ notifications ที่ไม่ใช่ของเรา
        [super observeValueForKeyPath:keyPath ofObject:object change:change context:context];
        return;
    }
    
    // ตรวจสอบว่าเป็น prior notification หรือไม่
    BOOL isPrior = [change[NSKeyValueChangeNotificationIsPriorKey] boolValue];
    if (isPrior) {
        NSLog(@"กำลังจะเปลี่ยน: %@", keyPath);
        return;
    }
    
    // ดึงประเภทการเปลี่ยนแปลง
    NSKeyValueChange changeKind = [change[NSKeyValueChangeKindKey] unsignedIntegerValue];
    
    switch (changeKind) {
        case NSKeyValueChangeSetting:
            NSLog(@"Property %@ ถูกกำหนดค่าใหม่", keyPath);
            NSLog(@"  เก่า: %@", change[NSKeyValueChangeOldKey]);
            NSLog(@"  ใหม่: %@", change[NSKeyValueChangeNewKey]);
            break;
            
        case NSKeyValueChangeInsertion:
            NSLog(@"เพิ่มสมาชิกใน %@ ที่ index: %@", keyPath, change[NSKeyValueChangeIndexesKey]);
            NSLog(@"  ค่าที่เพิ่ม: %@", change[NSKeyValueChangeNewKey]);
            break;
            
        case NSKeyValueChangeRemoval:
            NSLog(@"ลบสมาชิกออกจาก %@ ที่ index: %@", keyPath, change[NSKeyValueChangeIndexesKey]);
            NSLog(@"  ค่าที่ลบ: %@", change[NSKeyValueChangeOldKey]);
            break;
            
        case NSKeyValueChangeReplacement:
            NSLog(@"แทนที่สมาชิกใน %@ ที่ index: %@", keyPath, change[NSKeyValueChangeIndexesKey]);
            NSLog(@"  เก่า: %@", change[NSKeyValueChangeOldKey]);
            NSLog(@"  ใหม่: %@", change[NSKeyValueChangeNewKey]);
            break;
    }
}
```

---

## 35.3 removeObserver:forKeyPath:

การลบ observer เป็นสิ่งสำคัญมาก ถ้าลืมลบจะเกิด crash หรือ memory leak

### วิธีที่ถูกต้อง

```objc
@interface ViewController : NSObject

@property (nonatomic, strong) Counter *counter;

@end

@implementation ViewController

static void *ViewControllerContext = &ViewControllerContext;

- (void)viewDidLoad {
    self.counter = [[Counter alloc] init];
    
    [self.counter addObserver:self
                   forKeyPath:@"count"
                      options:NSKeyValueObservingOptionNew
                      context:ViewControllerContext];
}

- (void)dealloc {
    // MUST remove observer ก่อน dealloc!
    [self.counter removeObserver:self 
                      forKeyPath:@"count" 
                         context:ViewControllerContext];
}

- (void)observeValueForKeyPath:(NSString *)keyPath
                      ofObject:(id)object
                        change:(NSDictionary *)change
                       context:(void *)context {
    if (context == ViewControllerContext) {
        // จัดการ notification
    } else {
        [super observeValueForKeyPath:keyPath ofObject:object change:change context:context];
    }
}

@end
```

### การลบพร้อม Error Handling

```objc
// ถ้าลบ observer ที่ไม่ได้ register ไว้จะเกิด exception
// ใช้ @try/@catch เพื่อป้องกัน (แต่ควรใช้ context แทน)

- (void)safeRemoveObserver {
    @try {
        [self.counter removeObserver:self forKeyPath:@"count"];
    } @catch (NSException *exception) {
        NSLog(@"ไม่สามารถลบ observer: %@", exception.reason);
    }
}
```

---

## 35.4 NSKeyValueObservingOptions อย่างละเอียด

### NSKeyValueObservingOptionInitial

```objc
// เรียก observeValueForKeyPath: ทันทีหลัง register
// ประโยชน์: ทำ initial setup ใน observeValueForKeyPath: เพียงที่เดียว

[model addObserver:self
        forKeyPath:@"score"
           options:NSKeyValueObservingOptionNew | NSKeyValueObservingOptionInitial
           context:MyContext];

// จะเรียก observeValueForKeyPath ทันทีพร้อมค่าปัจจุบัน
// ทำให้ observer รู้สถานะเริ่มต้นโดยไม่ต้องอ่านค่าแยก
```

### NSKeyValueObservingOptionPrior

```objc
// เรียก observeValueForKeyPath สองครั้งต่อการเปลี่ยนแปลงหนึ่งครั้ง:
// 1. ก่อนเปลี่ยน (isPrior = YES)
// 2. หลังเปลี่ยน (isPrior = NO)

[model addObserver:self
        forKeyPath:@"items"
           options:NSKeyValueObservingOptionNew | NSKeyValueObservingOptionOld | NSKeyValueObservingOptionPrior
           context:MyContext];

- (void)observeValueForKeyPath:(NSString *)keyPath
                      ofObject:(id)object
                        change:(NSDictionary *)change
                       context:(void *)context {
    
    BOOL isPrior = [change[NSKeyValueChangeNotificationIsPriorKey] boolValue];
    
    if (isPrior) {
        NSLog(@"กำลังจะเปลี่ยน items: %@", change[NSKeyValueChangeOldKey]);
    } else {
        NSLog(@"items เปลี่ยนแล้ว: %@", change[NSKeyValueChangeNewKey]);
    }
}
```

### ตัวอย่าง: Initial + New + Old

```objc
@interface ScoreBoard : NSObject
@property (nonatomic, assign) NSInteger playerScore;
@property (nonatomic, strong) NSString *playerName;
@end

@implementation ScoreBoard
@end

@interface ScoreBoardUI : NSObject
- (void)connectToScoreBoard:(ScoreBoard *)board;
- (void)disconnectFromScoreBoard:(ScoreBoard *)board;
@end

static void *ScoreBoardUIContext = &ScoreBoardUIContext;

@implementation ScoreBoardUI

- (void)connectToScoreBoard:(ScoreBoard *)board {
    // NSKeyValueObservingOptionInitial: อัปเดต UI ทันที
    // NSKeyValueObservingOptionNew: รับค่าใหม่ทุกครั้ง
    // NSKeyValueObservingOptionOld: รับค่าเก่า (สำหรับ animation)
    [board addObserver:self
            forKeyPath:@"playerScore"
               options:NSKeyValueObservingOptionNew | 
                       NSKeyValueObservingOptionOld | 
                       NSKeyValueObservingOptionInitial
               context:ScoreBoardUIContext];
    
    [board addObserver:self
            forKeyPath:@"playerName"
               options:NSKeyValueObservingOptionNew | NSKeyValueObservingOptionInitial
               context:ScoreBoardUIContext];
}

- (void)disconnectFromScoreBoard:(ScoreBoard *)board {
    [board removeObserver:self forKeyPath:@"playerScore" context:ScoreBoardUIContext];
    [board removeObserver:self forKeyPath:@"playerName" context:ScoreBoardUIContext];
}

- (void)observeValueForKeyPath:(NSString *)keyPath
                      ofObject:(id)object
                        change:(NSDictionary *)change
                       context:(void *)context {
    
    if (context != ScoreBoardUIContext) {
        [super observeValueForKeyPath:keyPath ofObject:object change:change context:context];
        return;
    }
    
    if ([keyPath isEqualToString:@"playerScore"]) {
        NSInteger newScore = [change[NSKeyValueChangeNewKey] integerValue];
        NSInteger oldScore = [change[NSKeyValueChangeOldKey] integerValue];
        
        if (change[NSKeyValueChangeOldKey]) {
            NSInteger diff = newScore - oldScore;
            NSString *arrow = diff > 0 ? @"▲" : @"▼";
            NSLog(@"Score: %ld %@ %ld (เปลี่ยน %+ld)", (long)oldScore, arrow, (long)newScore, (long)diff);
        } else {
            // Initial call - ไม่มีค่าเก่า
            NSLog(@"Score เริ่มต้น: %ld", (long)newScore);
        }
        
    } else if ([keyPath isEqualToString:@"playerName"]) {
        NSString *name = change[NSKeyValueChangeNewKey];
        NSLog(@"Player: %@", name);
    }
}

@end

// การใช้งาน
ScoreBoard *board = [[ScoreBoard alloc] init];
board.playerName = @"Alice";
board.playerScore = 100;

ScoreBoardUI *ui = [[ScoreBoardUI alloc] init];
[ui connectToScoreBoard:board]; // จะ print initial values ทันที

board.playerScore = 150; // จะ print: Score: 100 ▲ 150 (เปลี่ยน +50)
board.playerScore = 120; // จะ print: Score: 150 ▼ 120 (เปลี่ยน -30)

[ui disconnectFromScoreBoard:board];
```

---

## 35.5 KVO Compliance: Automatic vs Manual

### Automatic KVO Compliance

Objective-C properties ที่ synthesize ด้วย `@synthesize` หรือ `@property` มาพร้อม automatic KVO support โดยระบบจะ override setter ให้อัตโนมัติ:

```objc
@interface Person : NSObject

@property (nonatomic, strong) NSString *name; // ✅ Auto KVO compliant
@property (nonatomic, assign) NSInteger age;  // ✅ Auto KVO compliant

@end

@implementation Person
// ไม่ต้องทำอะไรเพิ่ม - setter จะ trigger KVO อัตโนมัติ
@end

// เมื่อ set ค่า ผ่าน setter:
person.name = @"Alice"; // -> trigger KVO
[person setValue:@"Alice" forKey:@"name"]; // -> trigger KVO
```

### การปิด Automatic KVO

```objc
@implementation Person

// Override นี้เพื่อปิด automatic KVO สำหรับบาง properties
+ (BOOL)automaticallyNotifiesObserversForKey:(NSString *)key {
    if ([key isEqualToString:@"name"]) {
        return NO; // ปิด auto KVO สำหรับ name
    }
    return [super automaticallyNotifiesObserversForKey:key];
}

@end
```

---

## 35.6 willChangeValueForKey: / didChangeValueForKey:

เมื่อปิด automatic KVO หรือเมื่อ property ไม่ได้ถูก set ผ่าน setter ตามปกติ เราต้องเรียก manual notifications เอง

### การใช้งาน Manual KVO

```objc
@interface DataModel : NSObject {
    NSInteger _rawCount; // instance variable ไม่ใช่ property
}

- (void)incrementCount;
- (NSInteger)count;

@end

@implementation DataModel

+ (BOOL)automaticallyNotifiesObserversForKey:(NSString *)key {
    if ([key isEqualToString:@"count"]) {
        return NO; // ต้องทำ manual
    }
    return [super automaticallyNotifiesObserversForKey:key];
}

- (NSInteger)count {
    return _rawCount;
}

- (void)incrementCount {
    // ต้องเรียก will/did เอง
    [self willChangeValueForKey:@"count"];
    _rawCount++; // เปลี่ยนค่าโดยตรง ไม่ผ่าน setter
    [self didChangeValueForKey:@"count"];
}

@end
```

### Batch Updates

```objc
@implementation DataModel

- (void)performBatchUpdate:(void (^)(void))updateBlock {
    // แจ้งว่ากำลังจะเปลี่ยน (trigger prior)
    [self willChangeValueForKey:@"items"];
    [self willChangeValueForKey:@"count"];
    
    if (updateBlock) {
        updateBlock();
    }
    
    // แจ้งว่าเปลี่ยนแล้ว (trigger notification)
    [self didChangeValueForKey:@"count"];
    [self didChangeValueForKey:@"items"];
}

@end
```

### KVO กับ Mutable Array

```objc
@interface Library : NSObject

@property (nonatomic, strong) NSMutableArray *books;

- (void)addBook:(NSString *)book;
- (void)removeBook:(NSString *)book;
- (void)replaceBookAtIndex:(NSUInteger)index withBook:(NSString *)newBook;

@end

@implementation Library

- (instancetype)init {
    self = [super init];
    if (self) {
        _books = [NSMutableArray array];
    }
    return self;
}

// ❌ วิธีนี้ไม่ trigger KVO
- (void)addBookWrong:(NSString *)book {
    [self.books addObject:book]; // KVO ไม่รู้ว่า array เปลี่ยน
}

// ✅ วิธีที่ถูกต้อง: ใช้ mutableArrayValueForKey:
- (NSMutableArray *)mutableBooksProxy {
    return [self mutableArrayValueForKey:@"books"];
}

// วิธีแรก: ใช้ mutableArrayValueForKey:
- (void)addBookCorrect1:(NSString *)book {
    NSMutableArray *proxy = [self mutableArrayValueForKey:@"books"];
    [proxy addObject:book]; // จะ trigger KVO อัตโนมัติ
}

// วิธีที่สอง: Manual notification
- (void)addBook:(NSString *)book {
    NSUInteger index = self.books.count;
    
    // แจ้ง KVO ว่าจะเพิ่มสมาชิกที่ index นี้
    [self willChange:NSKeyValueChangeInsertion
     valuesAtIndexes:[NSIndexSet indexSetWithIndex:index]
              forKey:@"books"];
    
    [self.books addObject:book];
    
    [self didChange:NSKeyValueChangeInsertion
    valuesAtIndexes:[NSIndexSet indexSetWithIndex:index]
             forKey:@"books"];
}

- (void)removeBook:(NSString *)book {
    NSUInteger index = [self.books indexOfObject:book];
    if (index == NSNotFound) return;
    
    [self willChange:NSKeyValueChangeRemoval
     valuesAtIndexes:[NSIndexSet indexSetWithIndex:index]
              forKey:@"books"];
    
    [self.books removeObjectAtIndex:index];
    
    [self didChange:NSKeyValueChangeRemoval
    valuesAtIndexes:[NSIndexSet indexSetWithIndex:index]
             forKey:@"books"];
}

- (void)replaceBookAtIndex:(NSUInteger)index withBook:(NSString *)newBook {
    [self willChange:NSKeyValueChangeReplacement
     valuesAtIndexes:[NSIndexSet indexSetWithIndex:index]
              forKey:@"books"];
    
    [self.books replaceObjectAtIndex:index withObject:newBook];
    
    [self didChange:NSKeyValueChangeReplacement
    valuesAtIndexes:[NSIndexSet indexSetWithIndex:index]
             forKey:@"books"];
}

@end
```

---

## 35.7 Context Parameter

Context pointer เป็นสิ่งสำคัญในการเขียน KVO ที่ถูกต้อง โดยเฉพาะเมื่อมี inheritance

### ทำไม Context จำเป็น?

```objc
// ❌ ไม่ดี: ไม่ใช้ context
// ถ้า super class ก็ observe "name" เหมือนกัน จะแยกไม่ออก
- (void)observeValueForKeyPath:(NSString *)keyPath
                      ofObject:(id)object
                        change:(NSDictionary *)change
                       context:(void *)context {
    if ([keyPath isEqualToString:@"name"]) {
        // ไม่รู้ว่าเป็น notification ของเราหรือของ super
    }
}

// ✅ ดี: ใช้ context
static void *MyClassContext = &MyClassContext; // unique pointer

- (void)observeValueForKeyPath:(NSString *)keyPath
                      ofObject:(id)object
                        change:(NSDictionary *)change
                       context:(void *)context {
    if (context == MyClassContext) {
        // notification ของเรา
    } else {
        // ส่งต่อให้ super
        [super observeValueForKeyPath:keyPath ofObject:object change:change context:context];
    }
}
```

### Context กับ Inheritance

```objc
// BaseClass.m
@implementation BaseClass

static void *BaseClassContext = &BaseClassContext;

- (void)setup {
    [self.model addObserver:self
                 forKeyPath:@"name"
                    options:NSKeyValueObservingOptionNew
                    context:BaseClassContext];
}

- (void)observeValueForKeyPath:(NSString *)keyPath
                      ofObject:(id)object
                        change:(NSDictionary *)change
                       context:(void *)context {
    if (context == BaseClassContext) {
        // จัดการของ BaseClass
        NSLog(@"BaseClass: name = %@", change[NSKeyValueChangeNewKey]);
    } else {
        [super observeValueForKeyPath:keyPath ofObject:object change:change context:context];
    }
}

@end

// SubClass.m
@implementation SubClass

static void *SubClassContext = &SubClassContext;

- (void)setup {
    [super setup]; // BaseClass ก็ observe "name" เหมือนกัน
    
    [self.model addObserver:self
                 forKeyPath:@"name" // observe "name" เหมือนกัน
                    options:NSKeyValueObservingOptionNew | NSKeyValueObservingOptionOld
                    context:SubClassContext]; // context ต่างกัน
}

- (void)observeValueForKeyPath:(NSString *)keyPath
                      ofObject:(id)object
                        change:(NSDictionary *)change
                       context:(void *)context {
    if (context == SubClassContext) {
        // จัดการของ SubClass
        NSLog(@"SubClass: name เปลี่ยนจาก %@ เป็น %@",
              change[NSKeyValueChangeOldKey], change[NSKeyValueChangeNewKey]);
    } else {
        // ส่งต่อให้ BaseClass จัดการ
        [super observeValueForKeyPath:keyPath ofObject:object change:change context:context];
    }
}

@end
```

### Context เป็น Pointer ไปยัง Object

```objc
// บางครั้งใช้ context เป็น pointer ไปยัง identifier string
static NSString *kNameContext = @"NameObserverContext";
static NSString *kAgeContext = @"AgeObserverContext";

[person addObserver:self
         forKeyPath:@"name"
            options:NSKeyValueObservingOptionNew
            context:(__bridge void *)kNameContext];

[person addObserver:self
         forKeyPath:@"age"
            options:NSKeyValueObservingOptionNew
            context:(__bridge void *)kAgeContext];

- (void)observeValueForKeyPath:(NSString *)keyPath
                      ofObject:(id)object
                        change:(NSDictionary *)change
                       context:(void *)context {
    NSString *contextString = (__bridge NSString *)context;
    
    if ([contextString isEqualToString:@"NameObserverContext"]) {
        NSLog(@"Name notification: %@", change[NSKeyValueChangeNewKey]);
    } else if ([contextString isEqualToString:@"AgeObserverContext"]) {
        NSLog(@"Age notification: %@", change[NSKeyValueChangeNewKey]);
    }
}
```

---

## 35.8 KVO กับ Computed Properties

```objc
@interface Rectangle : NSObject

@property (nonatomic, assign) CGFloat width;
@property (nonatomic, assign) CGFloat height;

// Computed property - ขึ้นอยู่กับ width และ height
@property (nonatomic, readonly) CGFloat area;

@end

@implementation Rectangle

- (CGFloat)area {
    return self.width * self.height;
}

// บอก KVO ว่า "area" ขึ้นอยู่กับ "width" และ "height"
+ (NSSet *)keyPathsForValuesAffectingArea {
    return [NSSet setWithObjects:@"width", @"height", nil];
}

@end

// เมื่อ width หรือ height เปลี่ยน KVO จะ notify สำหรับ "area" ด้วย

Rectangle *rect = [[Rectangle alloc] init];

[rect addObserver:self
       forKeyPath:@"area"
          options:NSKeyValueObservingOptionNew
          context:nil];

rect.width = 10;  // จะ trigger notification สำหรับ "area"
rect.height = 5;  // จะ trigger notification สำหรับ "area"

// Output: area = 50
// Output: area = 50 (ถ้า width เปลี่ยนก่อน height)
```

### keyPathsForValuesAffecting<Key>

```objc
// Pattern: + (NSSet *)keyPathsForValuesAffecting<Key>
// ใช้สำหรับ computed properties

@interface ShoppingCart : NSObject

@property (nonatomic, strong) NSArray *items; // each item มี price
@property (nonatomic, readonly) CGFloat totalPrice;
@property (nonatomic, readonly) NSString *summaryText;

@end

@implementation ShoppingCart

- (CGFloat)totalPrice {
    return [[self.items valueForKeyPath:@"@sum.price"] floatValue];
}

- (NSString *)summaryText {
    return [NSString stringWithFormat:@"%lu รายการ รวม %.2f บาท", 
            (unsigned long)self.items.count, self.totalPrice];
}

// totalPrice ขึ้นอยู่กับ items
+ (NSSet *)keyPathsForValuesAffectingTotalPrice {
    return [NSSet setWithObject:@"items"];
}

// summaryText ขึ้นอยู่กับ items และ totalPrice
+ (NSSet *)keyPathsForValuesAffectingSummaryText {
    return [NSSet setWithObjects:@"items", @"totalPrice", nil];
}

@end
```

---

## 35.9 KVO กับ Dependent Key Paths

```objc
@interface UserProfile : NSObject

@property (nonatomic, strong) NSString *firstName;
@property (nonatomic, strong) NSString *lastName;
@property (nonatomic, readonly) NSString *fullName;

@end

@implementation UserProfile

- (NSString *)fullName {
    return [NSString stringWithFormat:@"%@ %@", self.firstName, self.lastName];
}

+ (NSSet *)keyPathsForValuesAffectingFullName {
    return [NSSet setWithObjects:@"firstName", @"lastName", nil];
}

@end

// Observer จะได้รับ notification เมื่อ fullName "เปลี่ยน"
// นั่นคือเมื่อ firstName หรือ lastName เปลี่ยน
UserProfile *profile = [[UserProfile alloc] init];

[profile addObserver:self
          forKeyPath:@"fullName"
             options:NSKeyValueObservingOptionNew
             context:MyContext];

profile.firstName = @"Alice"; // -> notification: fullName = "Alice (null)"
profile.lastName = @"Smith";  // -> notification: fullName = "Alice Smith"
```

---

## 35.10 KVO vs NSNotificationCenter

```
┌─────────────────────────────────────────────────────────────────────────┐
│                    KVO vs NSNotificationCenter                           │
├──────────────────────┬───────────────────────┬──────────────────────── │
│ Feature              │ KVO                    │ NSNotificationCenter    │
├──────────────────────┼───────────────────────┼──────────────────────── │
│ Target               │ Property ของ object   │ Event ทั่วไป            │
│ Setup                │ addObserver:forKeyPath │ addObserver:selector    │
│ เพิ่ม info เอง       │ ❌ (เปลี่ยนค่าเท่านั้น) │ ✅ userInfo dict        │
│ Many-to-one          │ ✅ หลาย observers      │ ✅ หลาย observers       │
│ One-to-many          │ ✅ หลาย key paths      │ ✅ หลาย events          │
│ Thread Safety        │ ⚠️ ต้องระวัง           │ ⚠️ ต้องระวัง            │
│ Loose Coupling       │ 🔶 (รู้จัก object)     │ ✅ ไม่รู้จัก sender     │
│ Memory Management    │ ⚠️ ต้อง remove เอง    │ ⚠️ ต้อง remove เอง     │
│ Built on             │ KVC                   │ ไม่เกี่ยวกับ KVC        │
└──────────────────────┴───────────────────────┴──────────────────────────┘
```

### เมื่อไหรควรใช้ KVO?

```objc
// ใช้ KVO เมื่อ:
// 1. ต้องการ observe property เฉพาะของ object ที่รู้จัก
// 2. ต้องการค่าเก่าและใหม่ของ property
// 3. ต้องการ observe property ของ object อื่น (ไม่ใช่ตัวเอง)

// ตัวอย่าง: View ต้องการรู้เมื่อ model.isLoading เปลี่ยน
[model addObserver:view forKeyPath:@"isLoading" options:... context:...];

// ใช้ NSNotificationCenter เมื่อ:
// 1. ต้องการส่ง notification ที่มีข้อมูลเพิ่มเติม
// 2. ต้องการ broadcast ไปยัง observers ที่ไม่รู้จักกัน
// 3. สำหรับ app-wide events (app foreground/background)

// ตัวอย่าง: แจ้ง logout event ไปทั่วทั้ง app
[[NSNotificationCenter defaultCenter] postNotificationName:@"UserDidLogout" object:nil];
```

---

## 35.11 Memory Management กับ KVO

### ปัญหาที่พบบ่อย

```objc
// ❌ ปัญหา: Observer ถูก dealloc ก่อนที่จะ remove
@implementation ViewController

- (void)viewDidLoad {
    [model addObserver:self forKeyPath:@"data" options:... context:...];
}

// ลืม implement dealloc! -> crash เมื่อ model พยายาม notify observer ที่ถูก dealloc แล้ว

@end
```

```objc
// ✅ แก้ไข: ต้อง remove ใน dealloc เสมอ
@implementation ViewController

static void *VCContext = &VCContext;

- (void)viewDidLoad {
    [model addObserver:self 
            forKeyPath:@"data" 
               options:NSKeyValueObservingOptionNew
               context:VCContext];
}

- (void)dealloc {
    [model removeObserver:self forKeyPath:@"data" context:VCContext];
}

@end
```

### Safe Observer Pattern

```objc
// สร้าง helper class สำหรับ safe KVO
@interface KVOObserver : NSObject

+ (instancetype)observeObject:(id)object
                      keyPath:(NSString *)keyPath
                      options:(NSKeyValueObservingOptions)options
                      handler:(void (^)(NSDictionary *change))handler;

@end

@interface KVOObserver ()

@property (nonatomic, weak) id observedObject;
@property (nonatomic, copy) NSString *keyPath;
@property (nonatomic, copy) void (^handler)(NSDictionary *change);

@end

@implementation KVOObserver

+ (instancetype)observeObject:(id)object
                      keyPath:(NSString *)keyPath
                      options:(NSKeyValueObservingOptions)options
                      handler:(void (^)(NSDictionary *))handler {
    KVOObserver *observer = [[KVOObserver alloc] init];
    observer.observedObject = object;
    observer.keyPath = keyPath;
    observer.handler = handler;
    
    [object addObserver:observer
             forKeyPath:keyPath
                options:options
                context:NULL];
    
    return observer;
}

- (void)observeValueForKeyPath:(NSString *)keyPath
                      ofObject:(id)object
                        change:(NSDictionary *)change
                       context:(void *)context {
    if (self.handler) {
        self.handler(change);
    }
}

- (void)dealloc {
    if (self.observedObject) {
        [self.observedObject removeObserver:self forKeyPath:self.keyPath];
    }
}

@end

// การใช้งาน
Counter *counter = [[Counter alloc] init];

// เก็บ observer ไว้ใน property
// เมื่อ observer ถูก dealloc จะ remove ตัวเองอัตโนมัติ
KVOObserver *obs = [KVOObserver observeObject:counter
                                      keyPath:@"count"
                                      options:NSKeyValueObservingOptionNew
                                      handler:^(NSDictionary *change) {
    NSLog(@"Count: %@", change[NSKeyValueChangeNewKey]);
}];

counter.count = 5;   // -> Count: 5
counter.count = 10;  // -> Count: 10

// obs ออก scope -> dealloc -> remove observer อัตโนมัติ
```

---

## 35.12 KVO Pitfalls ที่พบบ่อย

### 1. ลืม Remove Observer

```objc
// ❌ Crash: EXC_BAD_ACCESS หรือ NSInternalInconsistencyException
@interface ViewController : NSObject
@property (nonatomic, strong) Model *model;
@end

@implementation ViewController
- (void)init {
    self.model = [[Model alloc] init];
    [self.model addObserver:self forKeyPath:@"data" options:... context:...];
}
// ลืม dealloc! 
@end
```

### 2. Remove Observer ที่ไม่ได้ Register

```objc
// ❌ Exception: Cannot remove an observer that has not been registered
[model removeObserver:self forKeyPath:@"data"]; // ไม่เคย add

// ✅ วิธีแก้: ใช้ context หรือ @try/@catch
@try {
    [model removeObserver:self forKeyPath:@"data" context:MyContext];
} @catch (NSException *e) {
    // ไม่เป็นไร
}
```

### 3. Threading Issues

```objc
// ❌ KVO notification มาใน background thread
// แต่อัปเดต UI ใน background = crash

- (void)observeValueForKeyPath:(NSString *)keyPath
                      ofObject:(id)object
                        change:(NSDictionary *)change
                       context:(void *)context {
    // ❌ อัปเดต UI ตรงนี้โดยไม่ตรวจ thread
    // self.label.text = change[NSKeyValueChangeNewKey]; // CRASH!
    
    // ✅ ตรวจสอบ thread
    dispatch_async(dispatch_get_main_queue(), ^{
        self.label.text = change[NSKeyValueChangeNewKey]; // OK
    });
}
```

### 4. Circular Observations

```objc
// ❌ A observe B, B observe A -> infinite loop
// Object A observes B.value
// Object B observes A.value
// เมื่อ A.value เปลี่ยน -> แจ้ง B -> B.value เปลี่ยน -> แจ้ง A -> วนซ้ำ

// ✅ วิธีแก้: ใช้ flag
@implementation MyObject {
    BOOL _isUpdating;
}

- (void)observeValueForKeyPath:... {
    if (_isUpdating) return;
    _isUpdating = YES;
    // อัปเดตค่า
    _isUpdating = NO;
}
```

### 5. Add Observer หลายครั้ง

```objc
// ❌ Add observer สองครั้ง = notification สองครั้ง
[model addObserver:self forKeyPath:@"data" options:... context:...]; // ครั้งที่ 1
[model addObserver:self forKeyPath:@"data" options:... context:...]; // ครั้งที่ 2
// จะต้อง remove สองครั้งด้วย!

// ✅ ตรวจสอบก่อน add (แต่ไม่มี built-in method ตรวจ)
// วิธีที่ดีที่สุด: add ใน setup method ที่เรียกครั้งเดียวเท่านั้น
```

---

## 35.13 ตัวอย่างจริง: Model-View Binding

```objc
// Model
@interface UserModel : NSObject

@property (nonatomic, strong) NSString *username;
@property (nonatomic, assign) BOOL isOnline;
@property (nonatomic, assign) NSInteger messageCount;
@property (nonatomic, readonly) NSString *displayStatus;

@end

@implementation UserModel

- (NSString *)displayStatus {
    if (self.isOnline) {
        return [NSString stringWithFormat:@"Online - %ld ข้อความ", (long)self.messageCount];
    }
    return @"Offline";
}

+ (NSSet *)keyPathsForValuesAffectingDisplayStatus {
    return [NSSet setWithObjects:@"isOnline", @"messageCount", nil];
}

@end

// View (Console-based UI)
@interface UserView : NSObject

@property (nonatomic, strong) UserModel *model;

- (instancetype)initWithModel:(UserModel *)model;
- (void)render;

@end

static void *UserViewContext = &UserViewContext;

@implementation UserView

- (instancetype)initWithModel:(UserModel *)model {
    self = [super init];
    if (self) {
        _model = model;
        [self setupObservations];
    }
    return self;
}

- (void)setupObservations {
    [self.model addObserver:self
                 forKeyPath:@"username"
                    options:NSKeyValueObservingOptionNew | NSKeyValueObservingOptionInitial
                    context:UserViewContext];
    
    [self.model addObserver:self
                 forKeyPath:@"displayStatus"
                    options:NSKeyValueObservingOptionNew | NSKeyValueObservingOptionInitial
                    context:UserViewContext];
}

- (void)dealloc {
    [self.model removeObserver:self forKeyPath:@"username" context:UserViewContext];
    [self.model removeObserver:self forKeyPath:@"displayStatus" context:UserViewContext];
}

- (void)observeValueForKeyPath:(NSString *)keyPath
                      ofObject:(id)object
                        change:(NSDictionary *)change
                       context:(void *)context {
    if (context != UserViewContext) {
        [super observeValueForKeyPath:keyPath ofObject:object change:change context:context];
        return;
    }
    
    // อัปเดต UI เมื่อ model เปลี่ยน
    dispatch_async(dispatch_get_main_queue(), ^{
        [self render];
    });
}

- (void)render {
    NSLog(@"╔══════════════════════════╗");
    NSLog(@"║  User: %-18s ║", [self.model.username UTF8String] ?: "Unknown");
    NSLog(@"║  Status: %-16s ║", [self.model.displayStatus UTF8String]);
    NSLog(@"╚══════════════════════════╝");
}

@end

// การใช้งาน
UserModel *user = [[UserModel alloc] init];
UserView *view = [[UserView alloc] initWithModel:user];

// ทุกครั้งที่ model เปลี่ยน view จะอัปเดตอัตโนมัติ
user.username = @"alice123";
user.isOnline = YES;
user.messageCount = 5;
user.messageCount = 8;
user.isOnline = NO;
```

---

## 35.14 ตัวอย่างจริง: Progress Tracker

```objc
@interface DownloadTask : NSObject

@property (nonatomic, strong) NSString *fileName;
@property (nonatomic, assign) CGFloat progress;    // 0.0 - 1.0
@property (nonatomic, assign) BOOL isCompleted;
@property (nonatomic, assign) BOOL isFailed;
@property (nonatomic, readonly) NSString *statusText;

@end

@implementation DownloadTask

- (NSString *)statusText {
    if (self.isFailed) return @"ล้มเหลว";
    if (self.isCompleted) return @"เสร็จแล้ว";
    return [NSString stringWithFormat:@"%.0f%%", self.progress * 100];
}

+ (NSSet *)keyPathsForValuesAffectingStatusText {
    return [NSSet setWithObjects:@"progress", @"isCompleted", @"isFailed", nil];
}

@end

@interface DownloadMonitor : NSObject

@property (nonatomic, strong) NSMutableArray *tasks;

- (void)addTask:(DownloadTask *)task;
- (void)removeTask:(DownloadTask *)task;

@end

static void *MonitorContext = &MonitorContext;

@implementation DownloadMonitor

- (instancetype)init {
    self = [super init];
    if (self) {
        _tasks = [NSMutableArray array];
    }
    return self;
}

- (void)addTask:(DownloadTask *)task {
    [self.tasks addObject:task];
    
    [task addObserver:self
           forKeyPath:@"progress"
              options:NSKeyValueObservingOptionNew | NSKeyValueObservingOptionOld
              context:MonitorContext];
    
    [task addObserver:self
           forKeyPath:@"isCompleted"
              options:NSKeyValueObservingOptionNew
              context:MonitorContext];
    
    [task addObserver:self
           forKeyPath:@"isFailed"
              options:NSKeyValueObservingOptionNew
              context:MonitorContext];
    
    NSLog(@"📥 เพิ่มงาน: %@", task.fileName);
}

- (void)removeTask:(DownloadTask *)task {
    if ([self.tasks containsObject:task]) {
        [task removeObserver:self forKeyPath:@"progress" context:MonitorContext];
        [task removeObserver:self forKeyPath:@"isCompleted" context:MonitorContext];
        [task removeObserver:self forKeyPath:@"isFailed" context:MonitorContext];
        [self.tasks removeObject:task];
        NSLog(@"🗑️  ลบงาน: %@", task.fileName);
    }
}

- (void)observeValueForKeyPath:(NSString *)keyPath
                      ofObject:(id)object
                        change:(NSDictionary *)change
                       context:(void *)context {
    if (context != MonitorContext) {
        [super observeValueForKeyPath:keyPath ofObject:object change:change context:context];
        return;
    }
    
    DownloadTask *task = (DownloadTask *)object;
    
    if ([keyPath isEqualToString:@"progress"]) {
        CGFloat progress = [change[NSKeyValueChangeNewKey] floatValue];
        NSLog(@"  %@: %.0f%%", task.fileName, progress * 100);
        
    } else if ([keyPath isEqualToString:@"isCompleted"]) {
        if ([change[NSKeyValueChangeNewKey] boolValue]) {
            NSLog(@"✅ เสร็จแล้ว: %@", task.fileName);
        }
        
    } else if ([keyPath isEqualToString:@"isFailed"]) {
        if ([change[NSKeyValueChangeNewKey] boolValue]) {
            NSLog(@"❌ ล้มเหลว: %@", task.fileName);
        }
    }
    
    [self printSummary];
}

- (void)printSummary {
    NSInteger completed = [[self.tasks filteredArrayUsingPredicate:
                           [NSPredicate predicateWithFormat:@"isCompleted == YES"]] count];
    NSLog(@"สรุป: %lu/%lu เสร็จแล้ว", (unsigned long)completed, (unsigned long)self.tasks.count);
}

@end

// การใช้งาน
DownloadMonitor *monitor = [[DownloadMonitor alloc] init];

DownloadTask *task1 = [[DownloadTask alloc] init];
task1.fileName = @"video.mp4";

DownloadTask *task2 = [[DownloadTask alloc] init];
task2.fileName = @"image.png";

[monitor addTask:task1];
[monitor addTask:task2];

// จำลองการดาวน์โหลด
for (int i = 1; i <= 10; i++) {
    task1.progress = i * 0.1;
    [NSThread sleepForTimeInterval:0.1];
}
task1.isCompleted = YES;

for (int i = 1; i <= 5; i++) {
    task2.progress = i * 0.1;
    [NSThread sleepForTimeInterval:0.1];
}
task2.isFailed = YES; // จำลองการล้มเหลว
```

---

## แบบฝึกหัด

### แบบฝึกหัดที่ 1: Stock Price Monitor
สร้าง stock price monitor ที่:
- Observe ราคาหุ้นหลายตัว
- แสดง alert เมื่อราคาเกิน/ต่ำกว่า threshold
- Track การเปลี่ยนแปลงราคา (ขึ้น/ลง)

```objc
@interface Stock : NSObject
@property (nonatomic, strong) NSString *symbol; // เช่น "AAPL"
@property (nonatomic, assign) CGFloat price;
@property (nonatomic, assign) CGFloat previousClose;
@property (nonatomic, readonly) CGFloat changePercent;
@end

@interface StockMonitor : NSObject
- (void)watchStock:(Stock *)stock alertAbove:(CGFloat)high alertBelow:(CGFloat)low;
@end
```

### แบบฝึกหัดที่ 2: Form Auto-Save
สร้าง form ที่ auto-save เมื่อข้อมูลเปลี่ยน:

```objc
@interface FormModel : NSObject
@property NSString *title;
@property NSString *body;
@property NSArray *tags;
@property (readonly) NSDate *lastModified;
@end

@interface AutoSaveManager : NSObject
// Observe form properties และ save เมื่อมีการเปลี่ยนแปลง
// debounce 2 วินาที (รอ 2 วินาทีหลังการเปลี่ยนแปลงล่าสุดก่อน save)
@end
```

### แบบฝึกหัดที่ 3: Dependent Properties
สร้าง model ที่มี computed properties ที่ขึ้นอยู่กัน:

```objc
@interface OrderModel : NSObject
@property NSArray *items;        // รายการสั่งซื้อ
@property CGFloat discountRate;  // ส่วนลด 0.0-1.0
@property (readonly) CGFloat subtotal;    // ราคารวมก่อนลด
@property (readonly) CGFloat discount;    // จำนวนเงินที่ลด
@property (readonly) CGFloat total;       // ยอดสุทธิ
@property (readonly) NSString *summary;   // สรุปยอด
@end

// Observe "total" และ "summary" เมื่อ items หรือ discountRate เปลี่ยน
```

---

## สรุป

KVO เป็นกลไกที่ทรงพลังสำหรับ reactive programming ใน Objective-C:

1. **Observation** - observe property ใดๆ ของ KVC-compliant object
2. **Change Notifications** - รับค่าเก่า/ใหม่ โดยอัตโนมัติ
3. **Computed Properties** - `keyPathsForValuesAffecting<Key>` สำหรับ dependent properties
4. **Context** - ใช้ context pointer เพื่อความปลอดภัยใน inheritance
5. **Manual KVO** - `willChangeValueForKey:` / `didChangeValueForKey:` สำหรับ edge cases

ข้อควรระวัง: **ต้อง remove observer เสมอ** ก่อน object ถูก dealloc มิฉะนั้น app จะ crash

KVO เหมาะสำหรับ Model-View binding ที่ view ต้องการอัปเดตอัตโนมัติเมื่อ model เปลี่ยน แต่สำหรับ iOS development สมัยใหม่ Combine framework และ SwiftUI เป็นทางเลือกที่นิยมกว่า อย่างไรก็ตาม KVO ยังคงสำคัญในโค้ด Objective-C ที่ยังใช้งานอยู่
