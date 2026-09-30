# Part 32: Grand Central Dispatch (GCD) ใน Objective-C

## บทนำ

**Grand Central Dispatch (GCD)** หรือ **libdispatch** เป็น C-level API ของ Apple สำหรับการทำ **concurrency** (การทำงานหลายอย่างพร้อมกัน) GCD ช่วยจัดการ thread pool โดยอัตโนมัติ ทำให้เราไม่ต้องสร้างและจัดการ threads เอง

GCD ถูกออกแบบมาเพื่อ:
- ใช้ CPU cores หลาย core ให้มีประสิทธิภาพสูงสุด
- ลด overhead จากการสร้าง thread
- ป้องกัน race condition และ deadlock ด้วย patterns ที่ถูกต้อง
- ทำงานร่วมกับ Blocks ได้อย่างสมบูรณ์

---

## 32.1 พื้นฐาน Concurrency

### 32.1.1 Thread คืออะไร?

```
Process (App)
├── Main Thread (UI Thread)
│   - รัน UI code ทั้งหมด
│   - ห้ามทำงานหนักที่นี่ (จะ freeze UI)
│
├── Background Thread 1
│   - ทำงาน Network call
│
├── Background Thread 2
│   - ทำงาน image processing
│
└── Background Thread N
    - ทำงาน database queries
```

```objc
#import <Foundation/Foundation.h>

int main(int argc, const char * argv[]) {
    @autoreleasepool {
        // ตรวจสอบ thread ปัจจุบัน
        NSThread *currentThread = [NSThread currentThread];
        NSLog(@"Main thread: %@", currentThread);
        NSLog(@"Is main thread: %@", [NSThread isMainThread] ? @"YES" : @"NO");
        NSLog(@"Thread number: %@", currentThread.name ?: @"(no name)");
        
        // สร้าง NSThread แบบ manual (ไม่แนะนำ - ใช้ GCD แทน)
        NSThread *manualThread = [[NSThread alloc] initWithBlock:^{
            NSLog(@"Manual thread: %@", [NSThread currentThread]);
            NSLog(@"Is main thread: %@", [NSThread isMainThread] ? @"YES" : @"NO");
        }];
        manualThread.name = @"MyManualThread";
        [manualThread start];
        
        [NSThread sleepForTimeInterval:0.1]; // รอให้ thread ทำงานเสร็จ
    }
    return 0;
}
```

### 32.1.2 Serial vs Concurrent Execution

```
Serial Queue (ทำทีละงาน):
Time: ──[Task A]──[Task B]──[Task C]──▶

Concurrent Queue (ทำพร้อมกัน):
Time: ──[Task A    ]──▶
         ──[Task B]──▶
              ──[Task C        ]──▶
```

---

## 32.2 dispatch_queue_t

Queue คือคิวของ tasks (blocks) ที่รอดำเนินการ GCD จัดการ thread pool ให้เรา

### 32.2.1 ประเภทของ Queue

```objc
#import <Foundation/Foundation.h>

int main(int argc, const char * argv[]) {
    @autoreleasepool {
        // 1. Main Queue - serial, รัน tasks บน main thread
        dispatch_queue_t mainQ = dispatch_get_main_queue();
        
        // 2. Global Queues - concurrent, มี 4 priority levels (QoS)
        dispatch_queue_t highQ    = dispatch_get_global_queue(QOS_CLASS_USER_INTERACTIVE, 0);
        dispatch_queue_t medQ     = dispatch_get_global_queue(QOS_CLASS_USER_INITIATED, 0);
        dispatch_queue_t defaultQ = dispatch_get_global_queue(QOS_CLASS_DEFAULT, 0);
        dispatch_queue_t lowQ     = dispatch_get_global_queue(QOS_CLASS_UTILITY, 0);
        dispatch_queue_t bgQ      = dispatch_get_global_queue(QOS_CLASS_BACKGROUND, 0);
        
        // 3. Custom Serial Queue - ทำทีละงาน
        dispatch_queue_t mySerialQ = dispatch_queue_create("com.myapp.serial",
                                                           DISPATCH_QUEUE_SERIAL);
        
        // 4. Custom Concurrent Queue - ทำหลายงานพร้อมกัน
        dispatch_queue_t myConcurrentQ = dispatch_queue_create("com.myapp.concurrent",
                                                                DISPATCH_QUEUE_CONCURRENT);
        
        NSLog(@"Main queue:          %@", mainQ);
        NSLog(@"High priority:       %@", highQ);
        NSLog(@"Default priority:    %@", defaultQ);
        NSLog(@"My serial queue:     %@", mySerialQ);
        NSLog(@"My concurrent queue: %@", myConcurrentQ);
    }
    return 0;
}
```

### 32.2.2 QoS (Quality of Service) Classes

| QoS Class                    | ใช้สำหรับ                                  | ตัวอย่าง                          |
|-----------------------------|------------------------------------------|----------------------------------|
| `QOS_CLASS_USER_INTERACTIVE` | งาน UI ที่ต้องการทันที                    | animation, gesture response      |
| `QOS_CLASS_USER_INITIATED`  | งานที่ user เริ่มและรอผล                  | เปิดเอกสาร, sort list            |
| `QOS_CLASS_DEFAULT`         | งานทั่วไป                                | background operations            |
| `QOS_CLASS_UTILITY`         | งานที่ใช้เวลานาน แต่ user รับทราบ        | download ไฟล์, import data       |
| `QOS_CLASS_BACKGROUND`      | งานที่ user ไม่รู้ตัว                    | sync, backup, indexing           |

---

## 32.3 dispatch_async vs dispatch_sync

### 32.3.1 dispatch_async (ไม่รอ)

```objc
#import <Foundation/Foundation.h>

int main(int argc, const char * argv[]) {
    @autoreleasepool {
        dispatch_queue_t queue = dispatch_get_global_queue(QOS_CLASS_DEFAULT, 0);
        
        NSLog(@"ก่อน dispatch_async");
        
        // dispatch_async: submit และ return ทันที ไม่รอให้ block เสร็จ
        dispatch_async(queue, ^{
            [NSThread sleepForTimeInterval:1.0]; // จำลองงานหนัก
            NSLog(@"Block เสร็จ! thread: %@", [NSThread currentThread]);
        });
        
        NSLog(@"หลัง dispatch_async (กลับมาทันที)");
        
        // รอให้ background task เสร็จ (สำหรับตัวอย่างเท่านั้น)
        [NSThread sleepForTimeInterval:2.0];
        NSLog(@"จบโปรแกรม");
    }
    return 0;
}
```

**ผลลัพธ์:**
```
ก่อน dispatch_async
หลัง dispatch_async (กลับมาทันที)
Block เสร็จ! thread: <NSThread: ...>{number = 3, name = (null)}
จบโปรแกรม
```

### 32.3.2 dispatch_sync (รอ)

```objc
#import <Foundation/Foundation.h>

int main(int argc, const char * argv[]) {
    @autoreleasepool {
        dispatch_queue_t queue = dispatch_get_global_queue(QOS_CLASS_DEFAULT, 0);
        
        NSLog(@"ก่อน dispatch_sync");
        
        // dispatch_sync: submit และ รอ จนกว่า block จะเสร็จ
        dispatch_sync(queue, ^{
            [NSThread sleepForTimeInterval:1.0];
            NSLog(@"Block เสร็จ! thread: %@", [NSThread currentThread]);
        });
        
        NSLog(@"หลัง dispatch_sync (หลังจาก block เสร็จ)");
    }
    return 0;
}
```

**ผลลัพธ์:**
```
ก่อน dispatch_sync
Block เสร็จ! thread: <NSThread: ...>{number = 3, name = (null)}
หลัง dispatch_sync (หลังจาก block เสร็จ)
```

### 32.3.3 ข้อควรระวัง: Deadlock

```objc
// ❌ DEADLOCK - อย่าทำ!
dispatch_queue_t serialQ = dispatch_queue_create("com.myapp.serial", DISPATCH_QUEUE_SERIAL);

dispatch_async(serialQ, ^{
    NSLog(@"Task 1 started");
    
    // dispatch_sync บน queue เดียวกัน = DEADLOCK!
    dispatch_sync(serialQ, ^{
        NSLog(@"Task 2 - จะไม่มีทางถึงที่นี่!");
    });
    
    NSLog(@"Task 1 จะไม่มีทางมาถึงที่นี่ด้วย!");
});

// ✅ ถูกต้อง - ใช้ async แทน
dispatch_async(serialQ, ^{
    NSLog(@"Task 1 started");
    
    dispatch_async(serialQ, ^{ // async ไม่ block!
        NSLog(@"Task 2 รันหลัง Task 1");
    });
    
    NSLog(@"Task 1 เสร็จ");
});
```

---

## 32.4 Main Queue

Main queue เป็น serial queue ที่รัน tasks บน main (UI) thread:

```objc
#import <Foundation/Foundation.h>

// Pattern ทั่วไป: ทำงานหนักใน background แล้ว update UI บน main thread
void processDataAndUpdateUI(void) {
    dispatch_queue_t backgroundQueue = dispatch_get_global_queue(QOS_CLASS_USER_INITIATED, 0);
    dispatch_queue_t mainQueue = dispatch_get_main_queue();
    
    NSLog(@"เริ่มต้น - thread: %@", [NSThread currentThread]);
    
    dispatch_async(backgroundQueue, ^{
        NSLog(@"Background: กำลังประมวลผล - thread: %@", [NSThread currentThread]);
        
        // จำลองงานหนัก
        [NSThread sleepForTimeInterval:0.5];
        NSString *result = @"ผลลัพธ์จาก background processing";
        
        // กลับมาที่ main thread สำหรับ UI update
        dispatch_async(mainQueue, ^{
            NSLog(@"Main: Update UI ด้วย '%@' - thread: %@",
                  result, [NSThread currentThread]);
            NSLog(@"Is main thread: %@", [NSThread isMainThread] ? @"YES" : @"NO");
        });
    });
}

int main(int argc, const char * argv[]) {
    @autoreleasepool {
        processDataAndUpdateUI();
        [[NSRunLoop mainRunLoop] runUntilDate:[NSDate dateWithTimeIntervalSinceNow:2.0]];
    }
    return 0;
}
```

### 32.4.1 ตรวจสอบว่าอยู่บน Main Thread

```objc
#import <Foundation/Foundation.h>

// ฟังก์ชัน helper: รัน block บน main thread เสมอ
void runOnMain(dispatch_block_t block) {
    if ([NSThread isMainThread]) {
        block(); // อยู่บน main thread แล้ว รันทันที
    } else {
        dispatch_async(dispatch_get_main_queue(), block);
    }
}

// ตัวอย่างการใช้งาน
void updateLabel(NSString *text) {
    runOnMain(^{
        // อัพเดท UI ที่นี่
        NSLog(@"[Main Thread] Updating label: %@", text);
    });
}

int main(int argc, const char * argv[]) {
    @autoreleasepool {
        // เรียกจาก main thread
        updateLabel(@"Hello from main");
        
        // เรียกจาก background thread
        dispatch_async(dispatch_get_global_queue(QOS_CLASS_DEFAULT, 0), ^{
            NSLog(@"[Background] Calling updateLabel from background");
            updateLabel(@"Hello from background");
        });
        
        [[NSRunLoop mainRunLoop] runUntilDate:[NSDate dateWithTimeIntervalSinceNow:1.0]];
    }
    return 0;
}
```

---

## 32.5 Serial vs Concurrent Queues

### 32.5.1 Serial Queue

```objc
#import <Foundation/Foundation.h>

int main(int argc, const char * argv[]) {
    @autoreleasepool {
        // Serial queue: ทำทีละงาน ตามลำดับ
        dispatch_queue_t serialQ = dispatch_queue_create("com.myapp.serial",
                                                         DISPATCH_QUEUE_SERIAL);
        
        NSLog(@"Dispatching tasks to serial queue...");
        
        for (NSInteger i = 1; i <= 5; i++) {
            dispatch_async(serialQ, ^{
                NSLog(@"Task %ld start - thread: %@",
                      (long)i, [NSThread currentThread]);
                [NSThread sleepForTimeInterval:0.1]; // simulate work
                NSLog(@"Task %ld end", (long)i);
            });
        }
        
        // รอให้ tasks เสร็จ
        dispatch_sync(serialQ, ^{
            NSLog(@"All serial tasks done");
        });
    }
    return 0;
}
```

**ผลลัพธ์:** (ลำดับแน่นอน)
```
Task 1 start
Task 1 end
Task 2 start
Task 2 end
... (ตามลำดับ)
```

### 32.5.2 Concurrent Queue

```objc
#import <Foundation/Foundation.h>

int main(int argc, const char * argv[]) {
    @autoreleasepool {
        // Concurrent queue: ทำหลายงานพร้อมกัน
        dispatch_queue_t concurrentQ = dispatch_queue_create("com.myapp.concurrent",
                                                              DISPATCH_QUEUE_CONCURRENT);
        
        NSLog(@"Dispatching tasks to concurrent queue...");
        
        for (NSInteger i = 1; i <= 5; i++) {
            dispatch_async(concurrentQ, ^{
                NSLog(@"Task %ld start - thread: %@",
                      (long)i, [NSThread currentThread]);
                [NSThread sleepForTimeInterval:0.1 * i]; // ระยะเวลาต่างกัน
                NSLog(@"Task %ld end", (long)i);
            });
        }
        
        [NSThread sleepForTimeInterval:2.0]; // รอให้ tasks เสร็จ
    }
    return 0;
}
```

**ผลลัพธ์:** (ลำดับไม่แน่นอน - tasks รันพร้อมกัน)
```
Task 1 start
Task 2 start
Task 3 start
...
Task 1 end
Task 2 end
... (ไม่ตามลำดับ)
```

### 32.5.3 เมื่อไรใช้ Serial vs Concurrent

```objc
/*
 Serial Queue ใช้เมื่อ:
 ✅ ต้องการ thread safety (ป้องกัน race condition)
 ✅ Tasks ต้องทำตามลำดับ
 ✅ ทำ synchronization สำหรับ shared resource
 ✅ ป้องกัน multiple writes พร้อมกัน
 
 Concurrent Queue ใช้เมื่อ:
 ✅ Tasks เป็น independent กัน (ไม่ต้องการลำดับ)
 ✅ ต้องการ maximum performance
 ✅ Read operations บน shared data
 ✅ Tasks ใช้เวลานานและรอผลไม่จำเป็น
*/

// ตัวอย่าง: Serial queue สำหรับ thread-safe counter
@interface ThreadSafeCounter : NSObject
@property (nonatomic, assign, readonly) NSInteger count;
- (void)increment;
- (void)decrement;
@end

@implementation ThreadSafeCounter {
    dispatch_queue_t _queue;
    NSInteger _count;
}

- (instancetype)init {
    self = [super init];
    if (self) {
        _queue = dispatch_queue_create("com.counter.serial", DISPATCH_QUEUE_SERIAL);
        _count = 0;
    }
    return self;
}

- (NSInteger)count {
    // อ่านต้อง sync เพื่อความถูกต้อง
    __block NSInteger val;
    dispatch_sync(_queue, ^{
        val = _count;
    });
    return val;
}

- (void)increment {
    dispatch_async(_queue, ^{
        self->_count++;
    });
}

- (void)decrement {
    dispatch_async(_queue, ^{
        self->_count--;
    });
}

@end
```

---

## 32.6 dispatch_after

รัน block หลังจาก delay ที่กำหนด:

```objc
#import <Foundation/Foundation.h>

int main(int argc, const char * argv[]) {
    @autoreleasepool {
        NSLog(@"เริ่มต้น: %@", [NSDate date]);
        
        // รัน block หลังจาก 2 วินาที บน main queue
        dispatch_after(dispatch_time(DISPATCH_TIME_NOW, (int64_t)(2.0 * NSEC_PER_SEC)),
                       dispatch_get_main_queue(), ^{
            NSLog(@"หลัง 2 วินาที: %@", [NSDate date]);
        });
        
        // รัน block หลังจาก 1 วินาที บน background queue
        dispatch_queue_t bgQ = dispatch_get_global_queue(QOS_CLASS_DEFAULT, 0);
        dispatch_after(dispatch_time(DISPATCH_TIME_NOW, (int64_t)(1.0 * NSEC_PER_SEC)),
                       bgQ, ^{
            NSLog(@"Background หลัง 1 วินาที - thread: %@", [NSThread currentThread]);
        });
        
        // milliseconds
        dispatch_after(dispatch_time(DISPATCH_TIME_NOW, (int64_t)(500 * NSEC_PER_MSEC)),
                       dispatch_get_main_queue(), ^{
            NSLog(@"หลัง 500ms");
        });
        
        [[NSRunLoop mainRunLoop] runUntilDate:[NSDate dateWithTimeIntervalSinceNow:3.0]];
    }
    return 0;
}
```

### 32.6.1 Cancellable delay (iOS 8+)

```objc
#import <Foundation/Foundation.h>

int main(int argc, const char * argv[]) {
    @autoreleasepool {
        // dispatch_block_t สามารถ cancel ได้
        __block BOOL cancelled = NO;
        
        dispatch_block_t delayedTask = dispatch_block_create(0, ^{
            if (!cancelled) {
                NSLog(@"Delayed task executed!");
            }
        });
        
        dispatch_after(dispatch_time(DISPATCH_TIME_NOW, (int64_t)(2.0 * NSEC_PER_SEC)),
                       dispatch_get_main_queue(),
                       delayedTask);
        
        // ยกเลิกหลัง 0.5 วินาที
        dispatch_after(dispatch_time(DISPATCH_TIME_NOW, (int64_t)(0.5 * NSEC_PER_SEC)),
                       dispatch_get_main_queue(), ^{
            NSLog(@"Cancelling delayed task...");
            dispatch_block_cancel(delayedTask);
            cancelled = YES; // ป้องกัน race condition
        });
        
        [[NSRunLoop mainRunLoop] runUntilDate:[NSDate dateWithTimeIntervalSinceNow:3.0]];
    }
    return 0;
}
```

---

## 32.7 dispatch_once (Singleton Pattern)

`dispatch_once` รับประกันว่า block จะรันเพียงครั้งเดียวตลอด lifetime ของโปรแกรม thread-safe:

```objc
#import <Foundation/Foundation.h>

// Singleton pattern ด้วย dispatch_once
@interface DatabaseManager : NSObject

+ (instancetype)sharedInstance;
- (void)query:(NSString *)sql;
- (BOOL)connect;

@end

@implementation DatabaseManager {
    BOOL _connected;
}

+ (instancetype)sharedInstance {
    static DatabaseManager *instance;
    static dispatch_once_t onceToken;
    
    dispatch_once(&onceToken, ^{
        instance = [[self alloc] initPrivate];
        NSLog(@"DatabaseManager instance created (once!)");
    });
    
    return instance;
}

// ซ่อน init ปกติ
- (instancetype)init {
    NSAssert(NO, @"Use +sharedInstance instead");
    return nil;
}

- (instancetype)initPrivate {
    self = [super init];
    if (self) {
        _connected = NO;
    }
    return self;
}

- (BOOL)connect {
    if (_connected) {
        NSLog(@"Already connected");
        return YES;
    }
    NSLog(@"Connecting to database...");
    _connected = YES;
    return YES;
}

- (void)query:(NSString *)sql {
    if (!_connected) {
        NSLog(@"Error: Not connected!");
        return;
    }
    NSLog(@"Query: %@", sql);
}

@end

// Lazy initialization ด้วย dispatch_once
NSArray* getDefaultSettings(void) {
    static NSArray *settings;
    static dispatch_once_t onceToken;
    
    dispatch_once(&onceToken, ^{
        NSLog(@"Creating default settings...");
        settings = @[
            @{@"key": @"theme",    @"value": @"light"},
            @{@"key": @"language", @"value": @"th"},
            @{@"key": @"fontSize", @"value": @16},
        ];
    });
    
    return settings;
}

int main(int argc, const char * argv[]) {
    @autoreleasepool {
        // Singleton - ได้ instance เดียวกันทุกครั้ง
        DatabaseManager *db1 = [DatabaseManager sharedInstance];
        DatabaseManager *db2 = [DatabaseManager sharedInstance];
        DatabaseManager *db3 = [DatabaseManager sharedInstance];
        
        NSLog(@"Same instance? %@", (db1 == db2 && db2 == db3) ? @"YES" : @"NO");
        
        [db1 connect];
        [db2 query:@"SELECT * FROM users"]; // ใช้ instance เดิม
        
        // Test thread safety - เรียกจาก multiple threads พร้อมกัน
        dispatch_queue_t concurrentQ = dispatch_get_global_queue(QOS_CLASS_DEFAULT, 0);
        for (int i = 0; i < 5; i++) {
            dispatch_async(concurrentQ, ^{
                DatabaseManager *db = [DatabaseManager sharedInstance];
                [db query:[NSString stringWithFormat:@"SELECT %d", i]];
            });
        }
        
        // Lazy init test
        NSArray *s1 = getDefaultSettings();
        NSArray *s2 = getDefaultSettings(); // ไม่สร้างใหม่
        NSLog(@"Same settings? %@", (s1 == s2) ? @"YES" : @"NO");
        
        [NSThread sleepForTimeInterval:0.5];
    }
    return 0;
}
```

---

## 32.8 dispatch_group

ใช้สำหรับรอให้งานหลาย ๆ ชิ้นที่รันพร้อมกันเสร็จทั้งหมดก่อน:

```objc
#import <Foundation/Foundation.h>

// จำลอง async data loading
void loadDataAsync(NSString *source,
                   NSTimeInterval delay,
                   void (^completion)(NSString *result)) {
    dispatch_queue_t bgQ = dispatch_get_global_queue(QOS_CLASS_DEFAULT, 0);
    dispatch_async(bgQ, ^{
        [NSThread sleepForTimeInterval:delay];
        NSString *result = [NSString stringWithFormat:@"Data from %@", source];
        completion(result);
    });
}

int main(int argc, const char * argv[]) {
    @autoreleasepool {
        dispatch_group_t group = dispatch_group_create();
        __block NSMutableArray *results = [NSMutableArray array];
        dispatch_queue_t serialQ = dispatch_queue_create("com.results.serial", DISPATCH_QUEUE_SERIAL);
        
        NSLog(@"เริ่มโหลดข้อมูลพร้อมกัน...");
        
        // เข้า group
        dispatch_group_enter(group);
        loadDataAsync(@"Database", 1.0, ^(NSString *result) {
            dispatch_async(serialQ, ^{
                [results addObject:result];
                NSLog(@"โหลด DB เสร็จ: %@", result);
            });
            dispatch_group_leave(group); // ออกจาก group
        });
        
        dispatch_group_enter(group);
        loadDataAsync(@"API", 0.5, ^(NSString *result) {
            dispatch_async(serialQ, ^{
                [results addObject:result];
                NSLog(@"โหลด API เสร็จ: %@", result);
            });
            dispatch_group_leave(group);
        });
        
        dispatch_group_enter(group);
        loadDataAsync(@"Cache", 0.2, ^(NSString *result) {
            dispatch_async(serialQ, ^{
                [results addObject:result];
                NSLog(@"โหลด Cache เสร็จ: %@", result);
            });
            dispatch_group_leave(group);
        });
        
        // รอให้ทุก task เสร็จ แล้วทำ completion
        dispatch_group_notify(group, dispatch_get_main_queue(), ^{
            NSLog(@"\nทุกอย่างโหลดเสร็จแล้ว!");
            NSLog(@"Results: %@", results);
        });
        
        [[NSRunLoop mainRunLoop] runUntilDate:[NSDate dateWithTimeIntervalSinceNow:3.0]];
    }
    return 0;
}
```

### 32.8.1 dispatch_group_async

```objc
#import <Foundation/Foundation.h>

int main(int argc, const char * argv[]) {
    @autoreleasepool {
        dispatch_group_t group = dispatch_group_create();
        dispatch_queue_t concurrentQ = dispatch_get_global_queue(QOS_CLASS_DEFAULT, 0);
        __block NSMutableArray *results = [NSMutableArray array];
        NSLock *lock = [[NSLock alloc] init]; // สำหรับ thread-safe access
        
        // dispatch_group_async: submit task เข้า group และ queue พร้อมกัน
        for (NSInteger i = 1; i <= 5; i++) {
            dispatch_group_async(group, concurrentQ, ^{
                // งานที่ทำใน background
                NSLog(@"Task %ld running on thread: %@",
                      (long)i, [NSThread currentThread]);
                [NSThread sleepForTimeInterval:0.1 * i];
                
                NSString *result = [NSString stringWithFormat:@"Result %ld", (long)i];
                
                [lock lock];
                [results addObject:result];
                [lock unlock];
            });
        }
        
        // dispatch_group_wait: รอแบบ sync จนกว่าทุก task เสร็จหรือ timeout
        dispatch_time_t timeout = dispatch_time(DISPATCH_TIME_NOW,
                                                (int64_t)(5.0 * NSEC_PER_SEC));
        long result = dispatch_group_wait(group, timeout);
        
        if (result == 0) {
            NSLog(@"ทุก task เสร็จแล้ว!");
        } else {
            NSLog(@"Timeout!");
        }
        
        NSLog(@"Results: %@", results);
    }
    return 0;
}
```

### 32.8.2 Group สำหรับ parallel download

```objc
#import <Foundation/Foundation.h>

typedef void (^DownloadCompletion)(NSDictionary *results, NSError *error);

void downloadAllResources(NSArray *urls, DownloadCompletion completion) {
    dispatch_group_t group = dispatch_group_create();
    dispatch_queue_t concurrentQ = dispatch_get_global_queue(QOS_CLASS_USER_INITIATED, 0);
    
    __block NSMutableDictionary *results = [NSMutableDictionary dictionary];
    __block NSError *firstError = nil;
    dispatch_queue_t resultsQ = dispatch_queue_create("com.results", DISPATCH_QUEUE_SERIAL);
    
    for (NSString *url in urls) {
        dispatch_group_enter(group);
        
        // จำลอง download
        dispatch_async(concurrentQ, ^{
            [NSThread sleepForTimeInterval:arc4random_uniform(10) * 0.1 + 0.1];
            
            NSData *mockData = nil;
            NSError *error = nil;
            
            if ([url containsString:@"fail"]) {
                error = [NSError errorWithDomain:@"com.net" code:404
                                       userInfo:@{NSLocalizedDescriptionKey: @"Not found"}];
            } else {
                mockData = [url dataUsingEncoding:NSUTF8StringEncoding];
            }
            
            dispatch_async(resultsQ, ^{
                if (error && !firstError) {
                    firstError = error;
                }
                if (mockData) {
                    results[url] = mockData;
                }
                dispatch_group_leave(group);
            });
        });
    }
    
    dispatch_group_notify(group, dispatch_get_main_queue(), ^{
        completion([results copy], firstError);
    });
}

int main(int argc, const char * argv[]) {
    @autoreleasepool {
        NSArray *urls = @[
            @"https://example.com/image1.jpg",
            @"https://example.com/data.json",
            @"https://example.com/style.css",
        ];
        
        downloadAllResources(urls, ^(NSDictionary *results, NSError *error) {
            if (error) {
                NSLog(@"Error: %@", error.localizedDescription);
            }
            NSLog(@"Downloaded %lu resources", (unsigned long)results.count);
            for (NSString *url in results) {
                NSData *data = results[url];
                NSLog(@"  %@: %lu bytes", url, (unsigned long)data.length);
            }
        });
        
        [[NSRunLoop mainRunLoop] runUntilDate:[NSDate dateWithTimeIntervalSinceNow:5.0]];
    }
    return 0;
}
```

---

## 32.9 dispatch_barrier_async (Reader-Writer Pattern)

`dispatch_barrier_async` ใช้สร้าง "fence" ใน concurrent queue - รอให้ tasks ก่อนหน้าเสร็จทั้งหมด แล้วรัน barrier task เพียงลำพัง จากนั้นถึงจะให้ tasks ถัดไปรันต่อ

```objc
#import <Foundation/Foundation.h>

// Thread-safe data store ด้วย barrier
@interface SafeCache : NSObject

- (void)setValue:(id)value forKey:(NSString *)key;
- (id)valueForKey:(NSString *)key;
- (void)removeValueForKey:(NSString *)key;
- (NSDictionary *)allValues;

@end

@implementation SafeCache {
    NSMutableDictionary *_store;
    dispatch_queue_t _queue;
}

- (instancetype)init {
    self = [super init];
    if (self) {
        _store = [NSMutableDictionary dictionary];
        // ต้องใช้ concurrent queue กับ barrier
        _queue = dispatch_queue_create("com.safecache.queue",
                                       DISPATCH_QUEUE_CONCURRENT);
    }
    return self;
}

// READ: ใช้ sync concurrent - หลาย readers ได้พร้อมกัน
- (id)valueForKey:(NSString *)key {
    __block id value;
    dispatch_sync(_queue, ^{
        value = self->_store[key];
    });
    return value;
}

// WRITE: ใช้ barrier async - exclusive access
- (void)setValue:(id)value forKey:(NSString *)key {
    dispatch_barrier_async(_queue, ^{
        self->_store[key] = value;
    });
}

- (void)removeValueForKey:(NSString *)key {
    dispatch_barrier_async(_queue, ^{
        [self->_store removeObjectForKey:key];
    });
}

- (NSDictionary *)allValues {
    __block NSDictionary *snapshot;
    dispatch_sync(_queue, ^{
        snapshot = [self->_store copy];
    });
    return snapshot;
}

@end

int main(int argc, const char * argv[]) {
    @autoreleasepool {
        SafeCache *cache = [[SafeCache alloc] init];
        dispatch_queue_t concurrentQ = dispatch_get_global_queue(QOS_CLASS_DEFAULT, 0);
        
        // จำลอง concurrent reads and writes
        for (NSInteger i = 0; i < 10; i++) {
            // Writer
            if (i % 3 == 0) {
                dispatch_async(concurrentQ, ^{
                    [cache setValue:[NSString stringWithFormat:@"Value %ld", (long)i]
                             forKey:[NSString stringWithFormat:@"key%ld", (long)i]];
                    NSLog(@"Write key%ld", (long)i);
                });
            }
            // Reader
            else {
                dispatch_async(concurrentQ, ^{
                    NSString *val = [cache valueForKey:@"key0"];
                    NSLog(@"Read key0: %@", val ?: @"(nil)");
                });
            }
        }
        
        [NSThread sleepForTimeInterval:1.0];
        NSLog(@"Final cache: %@", [cache allValues]);
    }
    return 0;
}
```

---

## 32.10 Semaphore

Semaphore ใช้จำกัดจำนวน concurrent tasks หรือทำ synchronization:

```objc
#import <Foundation/Foundation.h>

int main(int argc, const char * argv[]) {
    @autoreleasepool {
        // Semaphore ที่จำกัดให้รันได้แค่ 2 tasks พร้อมกัน
        dispatch_semaphore_t semaphore = dispatch_semaphore_create(2);
        dispatch_queue_t concurrentQ = dispatch_get_global_queue(QOS_CLASS_DEFAULT, 0);
        
        NSLog(@"เริ่มต้น - จำกัด 2 tasks พร้อมกัน");
        
        for (NSInteger i = 1; i <= 6; i++) {
            dispatch_async(concurrentQ, ^{
                dispatch_semaphore_wait(semaphore, DISPATCH_TIME_FOREVER); // รอถ้า full
                
                NSLog(@"Task %ld เริ่ม - thread: %@",
                      (long)i, [NSThread currentThread]);
                [NSThread sleepForTimeInterval:0.5]; // simulate work
                NSLog(@"Task %ld เสร็จ", (long)i);
                
                dispatch_semaphore_signal(semaphore); // คืน slot
            });
        }
        
        [NSThread sleepForTimeInterval:3.0];
        NSLog(@"ทุก tasks เสร็จแล้ว");
    }
    return 0;
}
```

### 32.10.1 Semaphore สำหรับ sync async operations

```objc
#import <Foundation/Foundation.h>

// แปลง async API เป็น sync ด้วย semaphore (สำหรับ testing เท่านั้น!)
NSData* synchronousDownload(NSString *url) {
    __block NSData *result = nil;
    dispatch_semaphore_t sem = dispatch_semaphore_create(0);
    
    // จำลอง async download
    dispatch_queue_t bgQ = dispatch_get_global_queue(QOS_CLASS_DEFAULT, 0);
    dispatch_async(bgQ, ^{
        [NSThread sleepForTimeInterval:0.5];
        result = [url dataUsingEncoding:NSUTF8StringEncoding];
        dispatch_semaphore_signal(sem); // signal เมื่อเสร็จ
    });
    
    dispatch_semaphore_wait(sem, DISPATCH_TIME_FOREVER); // รอ signal
    return result;
}

// Rate limiting: จำกัด API calls
@interface RateLimiter : NSObject
@property (nonatomic, assign) NSInteger maxConcurrent;
- (void)executeTask:(void (^)(void))task;
@end

@implementation RateLimiter {
    dispatch_semaphore_t _semaphore;
    dispatch_queue_t _queue;
}

- (instancetype)initWithMaxConcurrent:(NSInteger)max {
    self = [super init];
    if (self) {
        _maxConcurrent = max;
        _semaphore = dispatch_semaphore_create(max);
        _queue = dispatch_get_global_queue(QOS_CLASS_DEFAULT, 0);
    }
    return self;
}

- (void)executeTask:(void (^)(void))task {
    dispatch_async(_queue, ^{
        dispatch_semaphore_wait(self->_semaphore, DISPATCH_TIME_FOREVER);
        task();
        dispatch_semaphore_signal(self->_semaphore);
    });
}

@end

int main(int argc, const char * argv[]) {
    @autoreleasepool {
        // Sync download
        NSData *data = synchronousDownload(@"https://example.com/data");
        NSLog(@"Downloaded: %lu bytes", (unsigned long)data.length);
        
        // Rate limiter
        RateLimiter *limiter = [[RateLimiter alloc] initWithMaxConcurrent:2];
        
        for (NSInteger i = 1; i <= 5; i++) {
            [limiter executeTask:^{
                NSLog(@"Task %ld running (max 2 at once)", (long)i);
                [NSThread sleepForTimeInterval:0.3];
                NSLog(@"Task %ld done", (long)i);
            }];
        }
        
        [NSThread sleepForTimeInterval:2.0];
    }
    return 0;
}
```

---

## 32.11 Combining GCD กับ Completion Blocks

```objc
#import <Foundation/Foundation.h>

// Service class ที่ใช้ GCD + completion blocks
@interface ImageService : NSObject

typedef void (^ImageCompletion)(UIImage *image, NSError *error);
// (ใช้ NSData แทน UIImage เพราะไม่มี UIKit ใน Command Line Tool)
typedef void (^DataCompletion)(NSData *data, NSError *error);

- (void)downloadImageAtURL:(NSString *)url
                completion:(DataCompletion)completion;

- (void)processImageData:(NSData *)data
              completion:(DataCompletion)completion;

- (void)downloadAndProcessURL:(NSString *)url
                   completion:(DataCompletion)completion;

@end

@implementation ImageService

- (void)downloadImageAtURL:(NSString *)url completion:(DataCompletion)completion {
    dispatch_queue_t bgQ = dispatch_get_global_queue(QOS_CLASS_USER_INITIATED, 0);
    
    dispatch_async(bgQ, ^{
        NSLog(@"[Download] Starting: %@", url);
        [NSThread sleepForTimeInterval:0.5]; // simulate network
        
        if ([url containsString:@"invalid"]) {
            NSError *error = [NSError errorWithDomain:@"com.image" code:404
                                             userInfo:@{NSLocalizedDescriptionKey: @"URL ไม่ถูกต้อง"}];
            dispatch_async(dispatch_get_main_queue(), ^{
                completion(nil, error);
            });
            return;
        }
        
        NSData *mockImageData = [url dataUsingEncoding:NSUTF8StringEncoding];
        NSLog(@"[Download] Complete: %lu bytes", (unsigned long)mockImageData.length);
        
        dispatch_async(dispatch_get_main_queue(), ^{
            completion(mockImageData, nil);
        });
    });
}

- (void)processImageData:(NSData *)data completion:(DataCompletion)completion {
    dispatch_queue_t bgQ = dispatch_get_global_queue(QOS_CLASS_USER_INITIATED, 0);
    
    dispatch_async(bgQ, ^{
        NSLog(@"[Process] Processing %lu bytes", (unsigned long)data.length);
        [NSThread sleepForTimeInterval:0.3]; // simulate processing
        
        // จำลอง processing (ใน reality จะ resize, compress etc.)
        NSMutableData *processed = [NSMutableData dataWithData:data];
        [processed appendBytes:"_processed" length:10];
        
        NSLog(@"[Process] Done: %lu bytes", (unsigned long)processed.length);
        
        dispatch_async(dispatch_get_main_queue(), ^{
            completion([processed copy], nil);
        });
    });
}

- (void)downloadAndProcessURL:(NSString *)url completion:(DataCompletion)completion {
    // Chain async operations
    [self downloadImageAtURL:url completion:^(NSData *data, NSError *error) {
        if (error) {
            completion(nil, error);
            return;
        }
        
        [self processImageData:data completion:^(NSData *processed, NSError *procError) {
            completion(processed, procError);
        }];
    }];
}

@end

int main(int argc, const char * argv[]) {
    @autoreleasepool {
        ImageService *service = [[ImageService alloc] init];
        
        NSLog(@"=== Download & Process ===");
        [service downloadAndProcessURL:@"https://example.com/photo.jpg"
                            completion:^(NSData *data, NSError *error) {
            if (data) {
                NSLog(@"Final result: %lu bytes", (unsigned long)data.length);
            } else {
                NSLog(@"Error: %@", error.localizedDescription);
            }
        }];
        
        NSLog(@"=== Invalid URL ===");
        [service downloadAndProcessURL:@"invalid://bad-url"
                            completion:^(NSData *data, NSError *error) {
            if (error) {
                NSLog(@"Expected error: %@", error.localizedDescription);
            }
        }];
        
        [[NSRunLoop mainRunLoop] runUntilDate:[NSDate dateWithTimeIntervalSinceNow:3.0]];
    }
    return 0;
}
```

---

## 32.12 UI Update Pattern (สำคัญมาก!)

ใน iOS development กฎเหล็กคือ: **ทุก UI operation ต้องรันบน Main Thread เท่านั้น**

```objc
#import <Foundation/Foundation.h>

// Pattern สำหรับ network call พร้อม UI update
@interface DataLoader : NSObject

typedef void (^LoadCompletion)(NSDictionary *data, NSError *error);

- (void)loadDataFromURL:(NSString *)url
             completion:(LoadCompletion)completion;

@end

@implementation DataLoader

- (void)loadDataFromURL:(NSString *)url completion:(LoadCompletion)completion {
    // ทำงานใน background thread
    dispatch_async(dispatch_get_global_queue(QOS_CLASS_USER_INITIATED, 0), ^{
        NSLog(@"Network: Loading %@", url);
        [NSThread sleepForTimeInterval:0.5]; // simulate network
        
        // จำลอง response
        NSDictionary *data = @{
            @"status": @"ok",
            @"count": @42,
            @"items": @[@"item1", @"item2", @"item3"]
        };
        
        // ✅ คืน result บน main thread
        dispatch_async(dispatch_get_main_queue(), ^{
            NSLog(@"[Main Thread] Calling completion - isMain: %@",
                  [NSThread isMainThread] ? @"YES" : @"NO");
            completion(data, nil);
        });
    });
}

@end

// ViewController simulation
@interface ViewController : NSObject

@property (nonatomic, strong) NSArray *items;
@property (nonatomic, strong) DataLoader *loader;

- (void)viewDidLoad;
- (void)reloadData;
- (void)updateUI:(NSArray *)items;

@end

@implementation ViewController

- (instancetype)init {
    self = [super init];
    if (self) {
        _loader = [[DataLoader alloc] init];
        _items = @[];
    }
    return self;
}

- (void)viewDidLoad {
    NSLog(@"viewDidLoad - isMain: %@", [NSThread isMainThread] ? @"YES" : @"NO");
    [self reloadData];
}

- (void)reloadData {
    NSLog(@"Loading data...");
    
    __weak typeof(self) weakSelf = self; // ป้องกัน retain cycle
    
    [self.loader loadDataFromURL:@"https://api.example.com/items"
                     completion:^(NSDictionary *data, NSError *error) {
        // เรียกบน main thread อยู่แล้ว (ตาม DataLoader implementation)
        NSLog(@"Completion on main: %@", [NSThread isMainThread] ? @"YES" : @"NO");
        
        __strong typeof(weakSelf) strongSelf = weakSelf;
        if (!strongSelf) return;
        
        if (error) {
            NSLog(@"Error: %@", error.localizedDescription);
            return;
        }
        
        [strongSelf updateUI:data[@"items"]];
    }];
}

- (void)updateUI:(NSArray *)items {
    NSAssert([NSThread isMainThread], @"Must be on main thread!");
    self.items = items;
    NSLog(@"UI Updated with %lu items", (unsigned long)items.count);
}

@end

int main(int argc, const char * argv[]) {
    @autoreleasepool {
        ViewController *vc = [[ViewController alloc] init];
        [vc viewDidLoad];
        [[NSRunLoop mainRunLoop] runUntilDate:[NSDate dateWithTimeIntervalSinceNow:2.0]];
    }
    return 0;
}
```

---

## 32.13 ตัวอย่างจริง: Image Processing Pipeline

```objc
#import <Foundation/Foundation.h>

typedef NS_ENUM(NSInteger, ProcessingStatus) {
    ProcessingStatusPending,
    ProcessingStatusDownloading,
    ProcessingStatusProcessing,
    ProcessingStatusDone,
    ProcessingStatusFailed,
};

@interface ImageTask : NSObject
@property (nonatomic, strong) NSString *url;
@property (nonatomic, assign) ProcessingStatus status;
@property (nonatomic, strong) NSData *result;
@property (nonatomic, strong) NSError *error;
@end

@implementation ImageTask
- (instancetype)initWithURL:(NSString *)url {
    self = [super init];
    if (self) {
        _url = url;
        _status = ProcessingStatusPending;
    }
    return self;
}
@end

@interface ImageProcessor : NSObject

typedef void (^ProgressCallback)(NSUInteger completed, NSUInteger total);
typedef void (^AllCompleteCallback)(NSArray<ImageTask *> *tasks);

- (void)processBatch:(NSArray<NSString *> *)urls
            progress:(ProgressCallback)progress
          completion:(AllCompleteCallback)completion;

@end

@implementation ImageProcessor

- (void)processBatch:(NSArray<NSString *> *)urls
            progress:(ProgressCallback)progress
          completion:(AllCompleteCallback)completion {
    
    NSMutableArray<ImageTask *> *tasks = [NSMutableArray array];
    for (NSString *url in urls) {
        [tasks addObject:[[ImageTask alloc] initWithURL:url]];
    }
    
    dispatch_group_t group = dispatch_group_create();
    dispatch_queue_t processQueue = dispatch_queue_create("com.image.process",
                                                           DISPATCH_QUEUE_CONCURRENT);
    
    __block NSUInteger completedCount = 0;
    dispatch_queue_t countQueue = dispatch_queue_create("com.count.serial",
                                                         DISPATCH_QUEUE_SERIAL);
    NSUInteger totalCount = tasks.count;
    
    for (ImageTask *task in tasks) {
        dispatch_group_enter(group);
        
        dispatch_async(processQueue, ^{
            // Step 1: Download
            task.status = ProcessingStatusDownloading;
            NSLog(@"[Download] %@", task.url);
            [NSThread sleepForTimeInterval:0.1 + (arc4random_uniform(5) * 0.1)];
            
            // จำลอง download failure
            if ([task.url containsString:@"broken"]) {
                task.status = ProcessingStatusFailed;
                task.error = [NSError errorWithDomain:@"com.image" code:404
                                             userInfo:@{NSLocalizedDescriptionKey: @"Download failed"}];
                dispatch_async(countQueue, ^{
                    completedCount++;
                    if (progress) {
                        dispatch_async(dispatch_get_main_queue(), ^{
                            progress(completedCount, totalCount);
                        });
                    }
                    dispatch_group_leave(group);
                });
                return;
            }
            
            NSData *mockImageData = [task.url dataUsingEncoding:NSUTF8StringEncoding];
            
            // Step 2: Process
            task.status = ProcessingStatusProcessing;
            NSLog(@"[Process] %@", task.url);
            [NSThread sleepForTimeInterval:0.1];
            
            NSMutableData *processed = [NSMutableData dataWithData:mockImageData];
            [processed appendBytes:"_processed" length:10];
            
            task.result = [processed copy];
            task.status = ProcessingStatusDone;
            
            dispatch_async(countQueue, ^{
                completedCount++;
                if (progress) {
                    dispatch_async(dispatch_get_main_queue(), ^{
                        progress(completedCount, totalCount);
                    });
                }
                dispatch_group_leave(group);
            });
        });
    }
    
    dispatch_group_notify(group, dispatch_get_main_queue(), ^{
        completion([tasks copy]);
    });
}

@end

int main(int argc, const char * argv[]) {
    @autoreleasepool {
        ImageProcessor *processor = [[ImageProcessor alloc] init];
        
        NSArray *urls = @[
            @"https://cdn.example.com/image1.jpg",
            @"https://cdn.example.com/image2.png",
            @"https://cdn.broken.com/image3.jpg",  // จะ fail
            @"https://cdn.example.com/image4.gif",
            @"https://cdn.example.com/image5.webp",
        ];
        
        NSLog(@"Processing %lu images...", (unsigned long)urls.count);
        
        [processor processBatch:urls
                       progress:^(NSUInteger completed, NSUInteger total) {
            NSLog(@"Progress: %lu/%lu (%.0f%%)",
                  (unsigned long)completed,
                  (unsigned long)total,
                  (double)completed/total * 100);
        } completion:^(NSArray<ImageTask *> *tasks) {
            NSUInteger success = 0, failed = 0;
            for (ImageTask *task in tasks) {
                if (task.status == ProcessingStatusDone) {
                    success++;
                    NSLog(@"✓ %@: %lu bytes",
                          task.url, (unsigned long)task.result.length);
                } else {
                    failed++;
                    NSLog(@"✗ %@: %@",
                          task.url, task.error.localizedDescription);
                }
            }
            NSLog(@"\nSummary: %lu succeeded, %lu failed", (unsigned long)success, (unsigned long)failed);
        }];
        
        [[NSRunLoop mainRunLoop] runUntilDate:[NSDate dateWithTimeIntervalSinceNow:5.0]];
    }
    return 0;
}
```

---

## 32.14 ตัวอย่างจริง: Network Call Simulation

```objc
#import <Foundation/Foundation.h>

// HTTP Method enum
typedef NS_ENUM(NSInteger, HTTPMethod) {
    HTTPMethodGET,
    HTTPMethodPOST,
    HTTPMethodPUT,
    HTTPMethodDELETE,
};

NSString* httpMethodString(HTTPMethod method) {
    switch (method) {
        case HTTPMethodGET:    return @"GET";
        case HTTPMethodPOST:   return @"POST";
        case HTTPMethodPUT:    return @"PUT";
        case HTTPMethodDELETE: return @"DELETE";
    }
    return @"UNKNOWN";
}

// Mock API Client
@interface MockAPIClient : NSObject

typedef void (^APICompletion)(NSDictionary *response, NSError *error);

- (void)request:(NSString *)path
         method:(HTTPMethod)method
           body:(NSDictionary *)body
     completion:(APICompletion)completion;

@end

@implementation MockAPIClient {
    NSMutableDictionary *_database;
    dispatch_queue_t _networkQueue;
}

- (instancetype)init {
    self = [super init];
    if (self) {
        _database = [NSMutableDictionary dictionary];
        _networkQueue = dispatch_queue_create("com.api.network", DISPATCH_QUEUE_CONCURRENT);
        
        // seed data
        _database[@"users"] = [@{
            @"1": @{@"id": @1, @"name": @"Alice", @"email": @"alice@example.com"},
            @"2": @{@"id": @2, @"name": @"Bob",   @"email": @"bob@example.com"},
        } mutableCopy];
    }
    return self;
}

- (void)request:(NSString *)path
         method:(HTTPMethod)method
           body:(NSDictionary *)body
     completion:(APICompletion)completion {
    
    dispatch_async(_networkQueue, ^{
        // Simulate network latency
        [NSThread sleepForTimeInterval:0.1 + arc4random_uniform(3) * 0.05];
        
        NSLog(@"[API] %@ %@", httpMethodString(method), path);
        
        NSDictionary *response = nil;
        NSError *error = nil;
        
        // Simplified routing
        if ([path hasPrefix:@"/users"]) {
            NSArray *parts = [path componentsSeparatedByString:@"/"];
            NSString *userID = parts.count > 2 ? parts[2] : nil;
            
            NSMutableDictionary *users = self->_database[@"users"];
            
            switch (method) {
                case HTTPMethodGET:
                    if (userID.length > 0) {
                        NSDictionary *user = users[userID];
                        if (user) {
                            response = @{@"status": @200, @"data": user};
                        } else {
                            error = [NSError errorWithDomain:@"com.api" code:404
                                                    userInfo:@{NSLocalizedDescriptionKey: @"User not found"}];
                        }
                    } else {
                        response = @{@"status": @200, @"data": [users allValues]};
                    }
                    break;
                    
                case HTTPMethodPOST:
                    if (body) {
                        NSString *newID = [NSString stringWithFormat:@"%lu",
                                          (unsigned long)users.count + 1];
                        NSMutableDictionary *newUser = [body mutableCopy];
                        newUser[@"id"] = @(newID.integerValue);
                        dispatch_barrier_async(self->_networkQueue, ^{
                            self->_database[@"users"][newID] = newUser;
                        });
                        response = @{@"status": @201, @"data": newUser};
                    }
                    break;
                    
                case HTTPMethodDELETE:
                    if (userID && users[userID]) {
                        dispatch_barrier_async(self->_networkQueue, ^{
                            [self->_database[@"users"] removeObjectForKey:userID];
                        });
                        response = @{@"status": @200, @"message": @"Deleted"};
                    } else {
                        error = [NSError errorWithDomain:@"com.api" code:404
                                               userInfo:@{NSLocalizedDescriptionKey: @"User not found"}];
                    }
                    break;
                    
                default:
                    error = [NSError errorWithDomain:@"com.api" code:405
                                           userInfo:@{NSLocalizedDescriptionKey: @"Method not allowed"}];
            }
        } else {
            error = [NSError errorWithDomain:@"com.api" code:404
                                   userInfo:@{NSLocalizedDescriptionKey: @"Route not found"}];
        }
        
        // คืนผลลัพธ์บน main thread
        dispatch_async(dispatch_get_main_queue(), ^{
            completion(response, error);
        });
    });
}

@end

int main(int argc, const char * argv[]) {
    @autoreleasepool {
        MockAPIClient *api = [[MockAPIClient alloc] init];
        dispatch_group_t group = dispatch_group_create();
        
        // GET all users
        dispatch_group_enter(group);
        [api request:@"/users" method:HTTPMethodGET body:nil
          completion:^(NSDictionary *response, NSError *error) {
            if (response) {
                NSLog(@"GET /users - Status: %@ - %lu users",
                      response[@"status"],
                      (unsigned long)[response[@"data"] count]);
            }
            dispatch_group_leave(group);
        }];
        
        // GET specific user
        dispatch_group_enter(group);
        [api request:@"/users/1" method:HTTPMethodGET body:nil
          completion:^(NSDictionary *response, NSError *error) {
            if (response) {
                NSLog(@"GET /users/1 - %@", response[@"data"][@"name"]);
            }
            dispatch_group_leave(group);
        }];
        
        // GET non-existent user
        dispatch_group_enter(group);
        [api request:@"/users/999" method:HTTPMethodGET body:nil
          completion:^(NSDictionary *response, NSError *error) {
            if (error) {
                NSLog(@"GET /users/999 - Error: %@", error.localizedDescription);
            }
            dispatch_group_leave(group);
        }];
        
        // POST new user
        dispatch_group_enter(group);
        [api request:@"/users" method:HTTPMethodPOST
               body:@{@"name": @"Charlie", @"email": @"charlie@example.com"}
         completion:^(NSDictionary *response, NSError *error) {
            if (response) {
                NSLog(@"POST /users - Created: %@", response[@"data"]);
            }
            dispatch_group_leave(group);
        }];
        
        dispatch_group_notify(group, dispatch_get_main_queue(), ^{
            NSLog(@"\nAll API calls completed!");
        });
        
        [[NSRunLoop mainRunLoop] runUntilDate:[NSDate dateWithTimeIntervalSinceNow:3.0]];
    }
    return 0;
}
```

---

## 32.15 Common Patterns และ Best Practices

```objc
#import <Foundation/Foundation.h>

// Pattern 1: Background task + Main thread update
void fetchAndDisplay(NSString *url, void (^updateUI)(NSString *content)) {
    dispatch_async(dispatch_get_global_queue(QOS_CLASS_USER_INITIATED, 0), ^{
        // Background work
        NSString *content = [NSString stringWithFormat:@"Content from %@", url];
        [NSThread sleepForTimeInterval:0.3];
        
        // Main thread update
        dispatch_async(dispatch_get_main_queue(), ^{
            updateUI(content);
        });
    });
}

// Pattern 2: Thread-safe property
@interface SafeValue : NSObject
@property (nonatomic, strong) id value;
@end

@implementation SafeValue {
    dispatch_queue_t _queue;
    id _value;
}

- (instancetype)init {
    self = [super init];
    if (self) {
        _queue = dispatch_queue_create("com.safevalue", DISPATCH_QUEUE_CONCURRENT);
    }
    return self;
}

- (id)value {
    __block id v;
    dispatch_sync(_queue, ^{ v = self->_value; });
    return v;
}

- (void)setValue:(id)value {
    dispatch_barrier_async(_queue, ^{ self->_value = value; });
}

@end

// Pattern 3: Throttle (จำกัดความถี่)
@interface Throttle : NSObject
- (instancetype)initWithDelay:(NSTimeInterval)delay;
- (void)call:(void (^)(void))block;
@end

@implementation Throttle {
    NSTimeInterval _delay;
    NSDate *_lastCallTime;
}

- (instancetype)initWithDelay:(NSTimeInterval)delay {
    self = [super init];
    if (self) {
        _delay = delay;
        _lastCallTime = [NSDate distantPast];
    }
    return self;
}

- (void)call:(void (^)(void))block {
    NSDate *now = [NSDate date];
    if ([now timeIntervalSinceDate:_lastCallTime] >= _delay) {
        _lastCallTime = now;
        block();
    }
}

@end

int main(int argc, const char * argv[]) {
    @autoreleasepool {
        // Test Pattern 1
        fetchAndDisplay(@"https://example.com", ^(NSString *content) {
            NSLog(@"UI Updated: %@", content);
            NSLog(@"On main thread: %@", [NSThread isMainThread] ? @"YES" : @"NO");
        });
        
        // Test Pattern 2
        SafeValue *sv = [[SafeValue alloc] init];
        dispatch_queue_t concQ = dispatch_get_global_queue(QOS_CLASS_DEFAULT, 0);
        
        for (int i = 0; i < 3; i++) {
            dispatch_async(concQ, ^{
                sv.value = @(i);
                NSLog(@"Set value: %d", i);
            });
        }
        
        // Test Pattern 3
        Throttle *throttle = [[Throttle alloc] initWithDelay:0.5];
        
        for (int i = 0; i < 10; i++) {
            [NSThread sleepForTimeInterval:0.1];
            [throttle call:^{
                NSLog(@"Throttled call at i=%d", i);
            }];
        }
        
        [[NSRunLoop mainRunLoop] runUntilDate:[NSDate dateWithTimeIntervalSinceNow:2.0]];
    }
    return 0;
}
```

---

## แบบฝึกหัด (Practice Exercises)

### แบบฝึกหัดที่ 1: dispatch_async พื้นฐาน
เขียนโปรแกรมที่รัน 5 tasks พร้อมกันใน global queue และ log เมื่อแต่ละ task เสร็จ

```objc
dispatch_queue_t queue = dispatch_get_global_queue(QOS_CLASS_DEFAULT, 0);

for (NSInteger i = 1; i <= 5; i++) {
    dispatch_async(queue, ^{
        NSLog(@"Task %ld started on thread %@",
              (long)i, [NSThread currentThread]);
        [NSThread sleepForTimeInterval:0.1 * i];
        NSLog(@"Task %ld completed", (long)i);
    });
}

[NSThread sleepForTimeInterval:2.0];
NSLog(@"All tasks submitted");
```

### แบบฝึกหัดที่ 2: Serial Queue สำหรับ Thread Safety
สร้าง thread-safe array ที่รับการ read/write จาก multiple threads

```objc
@interface ThreadSafeArray : NSObject
- (void)addObject:(id)object;
- (id)objectAtIndex:(NSUInteger)index;
- (NSUInteger)count;
- (NSArray *)allObjects;
@end

@implementation ThreadSafeArray {
    NSMutableArray *_array;
    dispatch_queue_t _queue;
}

- (instancetype)init {
    self = [super init];
    if (self) {
        _array = [NSMutableArray array];
        _queue = dispatch_queue_create("com.thread.safe.array", DISPATCH_QUEUE_SERIAL);
    }
    return self;
}

- (void)addObject:(id)object {
    dispatch_async(_queue, ^{
        [self->_array addObject:object];
    });
}

- (id)objectAtIndex:(NSUInteger)index {
    __block id obj;
    dispatch_sync(_queue, ^{
        obj = index < self->_array.count ? self->_array[index] : nil;
    });
    return obj;
}

- (NSUInteger)count {
    __block NSUInteger c;
    dispatch_sync(_queue, ^{ c = self->_array.count; });
    return c;
}

- (NSArray *)allObjects {
    __block NSArray *copy;
    dispatch_sync(_queue, ^{ copy = [self->_array copy]; });
    return copy;
}

@end

// ทดสอบ
ThreadSafeArray *arr = [[ThreadSafeArray alloc] init];
dispatch_queue_t concQ = dispatch_get_global_queue(QOS_CLASS_DEFAULT, 0);

for (int i = 0; i < 100; i++) {
    dispatch_async(concQ, ^{
        [arr addObject:@(i)];
    });
}

[NSThread sleepForTimeInterval:1.0];
NSLog(@"Count: %lu", (unsigned long)arr.count);
```

### แบบฝึกหัดที่ 3: dispatch_group สำหรับ parallel loading
โหลดข้อมูลจาก 3 "sources" พร้อมกัน แล้วรวมผลลัพธ์

```objc
void loadFromSource(NSString *source, NSTimeInterval delay,
                    void (^completion)(NSDictionary *)) {
    dispatch_async(dispatch_get_global_queue(QOS_CLASS_DEFAULT, 0), ^{
        [NSThread sleepForTimeInterval:delay];
        completion(@{@"source": source, @"timestamp": [NSDate date]});
    });
}

dispatch_group_t group = dispatch_group_create();
__block NSDictionary *dataA, *dataB, *dataC;

dispatch_group_enter(group);
loadFromSource(@"DatabaseA", 0.3, ^(NSDictionary *data) {
    dataA = data;
    dispatch_group_leave(group);
});

dispatch_group_enter(group);
loadFromSource(@"DatabaseB", 0.5, ^(NSDictionary *data) {
    dataB = data;
    dispatch_group_leave(group);
});

dispatch_group_enter(group);
loadFromSource(@"Cache", 0.1, ^(NSDictionary *data) {
    dataC = data;
    dispatch_group_leave(group);
});

dispatch_group_notify(group, dispatch_get_main_queue(), ^{
    NSLog(@"All data loaded:");
    NSLog(@"  A: %@", dataA[@"source"]);
    NSLog(@"  B: %@", dataB[@"source"]);
    NSLog(@"  C: %@", dataC[@"source"]);
});
```

### แบบฝึกหัดที่ 4: dispatch_once สำหรับ Singleton
สร้าง singleton class ที่ thread-safe สำหรับ app configuration

```objc
@interface AppConfig : NSObject
@property (nonatomic, strong, readonly) NSString *apiBaseURL;
@property (nonatomic, strong, readonly) NSString *apiKey;
@property (nonatomic, assign, readonly) NSInteger timeout;
+ (instancetype)shared;
@end

@implementation AppConfig

+ (instancetype)shared {
    static AppConfig *config;
    static dispatch_once_t token;
    dispatch_once(&token, ^{
        config = [[self alloc] initOnce];
    });
    return config;
}

- (instancetype)initOnce {
    self = [super init];
    if (self) {
        _apiBaseURL = @"https://api.myapp.com/v1";
        _apiKey     = @"abc123secret";
        _timeout    = 30;
    }
    return self;
}

- (instancetype)init { NSAssert(NO, @"Use +shared"); return nil; }

@end

// ทดสอบ
AppConfig *c1 = [AppConfig shared];
AppConfig *c2 = [AppConfig shared];
NSLog(@"Same instance: %@", c1 == c2 ? @"YES" : @"NO");
NSLog(@"Base URL: %@", [AppConfig shared].apiBaseURL);
```

### แบบฝึกหัดที่ 5: dispatch_after สำหรับ delayed actions
สร้าง countdown timer ที่แสดงตัวเลขทุกวินาที

```objc
void countdown(NSInteger from, void (^onTick)(NSInteger remaining),
               void (^onDone)(void)) {
    if (from <= 0) {
        onDone();
        return;
    }
    onTick(from);
    dispatch_after(dispatch_time(DISPATCH_TIME_NOW, NSEC_PER_SEC),
                   dispatch_get_main_queue(), ^{
        countdown(from - 1, onTick, onDone);
    });
}

countdown(5,
    ^(NSInteger remaining) { NSLog(@"นับถอยหลัง: %ld", (long)remaining); },
    ^{ NSLog(@"BOOM! 🎆"); }
);
```

### แบบฝึกหัดที่ 6: Semaphore สำหรับ Rate Limiting
สร้าง function ที่จำกัดจำนวน concurrent API calls

```objc
// Rate limiter: จำกัด N concurrent operations
dispatch_semaphore_t rateLimiter = dispatch_semaphore_create(3); // max 3
dispatch_queue_t apiQueue = dispatch_get_global_queue(QOS_CLASS_DEFAULT, 0);

NSMutableArray *results = [NSMutableArray array];
NSLock *lock = [[NSLock alloc] init];

for (NSInteger i = 1; i <= 10; i++) {
    dispatch_async(apiQueue, ^{
        dispatch_semaphore_wait(rateLimiter, DISPATCH_TIME_FOREVER);
        
        NSLog(@"API Call %ld starting (max 3 at once)", (long)i);
        [NSThread sleepForTimeInterval:0.2];
        NSLog(@"API Call %ld done", (long)i);
        
        [lock lock];
        [results addObject:@(i)];
        [lock unlock];
        
        dispatch_semaphore_signal(rateLimiter);
    });
}

[NSThread sleepForTimeInterval:2.0];
NSLog(@"Completed: %lu calls", (unsigned long)results.count);
```

### แบบฝึกหัดที่ 7: dispatch_barrier สำหรับ Read-Write Lock
สร้าง thread-safe cache ด้วย concurrent reads และ exclusive writes

```objc
@interface ReadWriteCache : NSObject
- (void)setObject:(id)obj forKey:(NSString *)key;
- (id)objectForKey:(NSString *)key;
- (void)removeObjectForKey:(NSString *)key;
@end

@implementation ReadWriteCache {
    NSMutableDictionary *_dict;
    dispatch_queue_t _queue;
}

- (instancetype)init {
    self = [super init];
    if (self) {
        _dict = [NSMutableDictionary dictionary];
        _queue = dispatch_queue_create("com.cache.rw", DISPATCH_QUEUE_CONCURRENT);
    }
    return self;
}

// Multiple readers OK
- (id)objectForKey:(NSString *)key {
    __block id obj;
    dispatch_sync(_queue, ^{ obj = self->_dict[key]; }); // concurrent read
    return obj;
}

// Exclusive writer
- (void)setObject:(id)obj forKey:(NSString *)key {
    dispatch_barrier_async(_queue, ^{ // exclusive write
        self->_dict[key] = obj;
    });
}

- (void)removeObjectForKey:(NSString *)key {
    dispatch_barrier_async(_queue, ^{
        [self->_dict removeObjectForKey:key];
    });
}

@end

// ทดสอบ
ReadWriteCache *cache = [[ReadWriteCache alloc] init];
dispatch_queue_t q = dispatch_get_global_queue(QOS_CLASS_DEFAULT, 0);

// Write
dispatch_async(q, ^{ [cache setObject:@"Hello" forKey:@"greeting"]; });

// Many concurrent reads
for (int i = 0; i < 5; i++) {
    dispatch_async(q, ^{
        NSString *val = [cache objectForKey:@"greeting"];
        NSLog(@"Read: %@", val);
    });
}
```

### แบบฝึกหัดที่ 8: Serial Queue สำหรับ File Operations
สร้าง file manager ที่ serialize การอ่าน/เขียนไฟล์

```objc
@interface SafeFileManager : NSObject
- (void)writeData:(NSData *)data toPath:(NSString *)path
       completion:(void (^)(BOOL success))completion;
- (void)readDataFromPath:(NSString *)path
             completion:(void (^)(NSData *data))completion;
@end

@implementation SafeFileManager {
    dispatch_queue_t _fileQueue;
}

- (instancetype)init {
    self = [super init];
    if (self) {
        _fileQueue = dispatch_queue_create("com.file.serial", DISPATCH_QUEUE_SERIAL);
    }
    return self;
}

- (void)writeData:(NSData *)data toPath:(NSString *)path
       completion:(void (^)(BOOL))completion {
    dispatch_async(_fileQueue, ^{
        BOOL ok = [data writeToFile:path atomically:YES];
        NSLog(@"Write to %@: %@", path, ok ? @"OK" : @"FAIL");
        if (completion) {
            dispatch_async(dispatch_get_main_queue(), ^{
                completion(ok);
            });
        }
    });
}

- (void)readDataFromPath:(NSString *)path
             completion:(void (^)(NSData *))completion {
    dispatch_async(_fileQueue, ^{
        NSData *data = [NSData dataWithContentsOfFile:path];
        if (completion) {
            dispatch_async(dispatch_get_main_queue(), ^{
                completion(data);
            });
        }
    });
}

@end

// ทดสอบ
SafeFileManager *fm = [[SafeFileManager alloc] init];
NSString *tempPath = [NSTemporaryDirectory() stringByAppendingPathComponent:@"test.dat"];

NSData *testData = [@"Hello from GCD!" dataUsingEncoding:NSUTF8StringEncoding];
[fm writeData:testData toPath:tempPath completion:^(BOOL success) {
    NSLog(@"Write completed: %@", success ? @"OK" : @"FAIL");
    
    [fm readDataFromPath:tempPath completion:^(NSData *data) {
        NSString *content = [[NSString alloc] initWithData:data encoding:NSUTF8StringEncoding];
        NSLog(@"Read back: %@", content);
    }];
}];
```

### แบบฝึกหัดที่ 9: Producer-Consumer Pattern
สร้าง producer-consumer ด้วย serial queue เป็น buffer

```objc
@interface WorkQueue : NSObject
- (void)produce:(id)item;
- (void)startConsuming:(void (^)(id item))consumer;
- (void)stop;
@end

@implementation WorkQueue {
    NSMutableArray *_items;
    dispatch_queue_t _serialQ;
    dispatch_semaphore_t _itemAvailable;
    BOOL _running;
}

- (instancetype)init {
    self = [super init];
    if (self) {
        _items = [NSMutableArray array];
        _serialQ = dispatch_queue_create("com.workqueue.serial", DISPATCH_QUEUE_SERIAL);
        _itemAvailable = dispatch_semaphore_create(0);
        _running = NO;
    }
    return self;
}

- (void)produce:(id)item {
    dispatch_async(_serialQ, ^{
        [self->_items addObject:item];
        NSLog(@"Produced: %@ (queue size: %lu)", item, (unsigned long)self->_items.count);
        dispatch_semaphore_signal(self->_itemAvailable);
    });
}

- (void)startConsuming:(void (^)(id))consumer {
    _running = YES;
    dispatch_queue_t consumerQ = dispatch_queue_create("com.consumer", DISPATCH_QUEUE_SERIAL);
    
    dispatch_async(consumerQ, ^{
        while (self->_running) {
            dispatch_semaphore_wait(self->_itemAvailable, dispatch_time(DISPATCH_TIME_NOW,
                                                                          100 * NSEC_PER_MSEC));
            
            __block id item = nil;
            dispatch_sync(self->_serialQ, ^{
                if (self->_items.count > 0) {
                    item = self->_items[0];
                    [self->_items removeObjectAtIndex:0];
                }
            });
            
            if (item) {
                NSLog(@"Consuming: %@", item);
                consumer(item);
            }
        }
    });
}

- (void)stop {
    _running = NO;
    dispatch_semaphore_signal(_itemAvailable); // unblock consumer
}

@end

// ทดสอบ
WorkQueue *wq = [[WorkQueue alloc] init];
[wq startConsuming:^(id item) {
    [NSThread sleepForTimeInterval:0.1]; // simulate processing
}];

// Produce items
for (int i = 1; i <= 5; i++) {
    dispatch_after(dispatch_time(DISPATCH_TIME_NOW, (int64_t)(i * 0.2 * NSEC_PER_SEC)),
                   dispatch_get_main_queue(), ^{
        [wq produce:[NSString stringWithFormat:@"Item %d", i]];
    });
}

[[NSRunLoop mainRunLoop] runUntilDate:[NSDate dateWithTimeIntervalSinceNow:3.0]];
[wq stop];
```

### แบบฝึกหัดที่ 10: Timeout Pattern
เพิ่ม timeout ให้ async operation

```objc
typedef void (^TimeoutCompletion)(id result, BOOL timedOut);

void withTimeout(NSTimeInterval timeout,
                 void (^operation)(void (^complete)(id)),
                 TimeoutCompletion completion) {
    
    __block BOOL completed = NO;
    dispatch_queue_t q = dispatch_queue_create("com.timeout.serial", DISPATCH_QUEUE_SERIAL);
    
    // Timeout timer
    dispatch_after(dispatch_time(DISPATCH_TIME_NOW, (int64_t)(timeout * NSEC_PER_SEC)),
                   q, ^{
        if (!completed) {
            completed = YES;
            dispatch_async(dispatch_get_main_queue(), ^{
                completion(nil, YES); // timed out
            });
        }
    });
    
    // Run operation
    operation(^(id result) {
        dispatch_async(q, ^{
            if (!completed) {
                completed = YES;
                dispatch_async(dispatch_get_main_queue(), ^{
                    completion(result, NO); // completed
                });
            }
        });
    });
}

// ทดสอบ - operation ที่เสร็จก่อน timeout
withTimeout(1.0, ^(void (^complete)(id)) {
    dispatch_async(dispatch_get_global_queue(QOS_CLASS_DEFAULT, 0), ^{
        [NSThread sleepForTimeInterval:0.3]; // เสร็จก่อน 1 วินาที
        complete(@"Success result!");
    });
}, ^(id result, BOOL timedOut) {
    if (timedOut) {
        NSLog(@"TIMEOUT!");
    } else {
        NSLog(@"Completed: %@", result);
    }
});

// operation ที่ timeout
withTimeout(0.5, ^(void (^complete)(id)) {
    dispatch_async(dispatch_get_global_queue(QOS_CLASS_DEFAULT, 0), ^{
        [NSThread sleepForTimeInterval:2.0]; // ช้ากว่า 0.5 วินาที
        complete(@"Too late");
    });
}, ^(id result, BOOL timedOut) {
    if (timedOut) {
        NSLog(@"TIMEOUT! (expected)");
    } else {
        NSLog(@"Completed: %@", result);
    }
});
```

---

## 32.16 สรุป: GCD Cheat Sheet

```objc
// ===== Queue Types =====

// Main queue (serial, main thread)
dispatch_get_main_queue()

// Global queues (concurrent)
dispatch_get_global_queue(QOS_CLASS_USER_INTERACTIVE, 0) // สูงสุด
dispatch_get_global_queue(QOS_CLASS_USER_INITIATED, 0)
dispatch_get_global_queue(QOS_CLASS_DEFAULT, 0)
dispatch_get_global_queue(QOS_CLASS_UTILITY, 0)
dispatch_get_global_queue(QOS_CLASS_BACKGROUND, 0)      // ต่ำสุด

// Custom queues
dispatch_queue_create("label", DISPATCH_QUEUE_SERIAL)
dispatch_queue_create("label", DISPATCH_QUEUE_CONCURRENT)

// ===== Dispatching =====

// Async (ไม่รอ)
dispatch_async(queue, ^{ /* work */ });

// Sync (รอ)
dispatch_sync(queue, ^{ /* work */ });

// After delay
dispatch_after(dispatch_time(DISPATCH_TIME_NOW, delayInNanoseconds), queue, ^{ });
// delay = (int64_t)(seconds * NSEC_PER_SEC)
// delay = (int64_t)(ms * NSEC_PER_MSEC)

// Once (thread-safe singleton)
static dispatch_once_t token;
dispatch_once(&token, ^{ /* initialize */ });

// ===== Groups =====
dispatch_group_t group = dispatch_group_create();

// With group_async
dispatch_group_async(group, queue, ^{ /* task */ });

// With enter/leave
dispatch_group_enter(group);
// ... async operation ...
dispatch_group_leave(group);

// Wait (sync)
dispatch_group_wait(group, DISPATCH_TIME_FOREVER);

// Notify (async callback)
dispatch_group_notify(group, queue, ^{ /* all done */ });

// ===== Barrier =====
dispatch_barrier_async(concurrentQueue, ^{ /* exclusive write */ });

// ===== Semaphore =====
dispatch_semaphore_t sem = dispatch_semaphore_create(N);
dispatch_semaphore_wait(sem, DISPATCH_TIME_FOREVER); // -1
dispatch_semaphore_signal(sem); // +1
```

---

## สรุป

ในบทนี้เราได้เรียนรู้:

1. **Concurrency Basics** - threads, queues, serial vs concurrent
2. **dispatch_queue_t** - ชนิดของ queue และการสร้าง custom queue
3. **Main Queue** - ทำไมต้อง update UI บน main thread
4. **Global Queues** - QoS classes และการเลือกใช้
5. **dispatch_async / dispatch_sync** - ความแตกต่างและข้อควรระวัง Deadlock
6. **dispatch_after** - delayed execution
7. **dispatch_once** - thread-safe singleton pattern
8. **dispatch_group** - รอ multiple async tasks
9. **dispatch_barrier_async** - reader-writer pattern
10. **Semaphores** - จำกัด concurrency และ synchronization
11. **GCD + Completion Blocks** - pattern สำหรับ async programming
12. **UI Update Pattern** - background work + main thread update
13. **Real-world examples** - image processing, API calls

GCD เป็น foundation สำคัญของ iOS/macOS development ทุก app ที่ดีต้องใช้ GCD อย่างถูกต้องเพื่อ performance ที่ดีและ UI ที่ responsive

---

## อ่านเพิ่มเติม

- [Apple's Concurrency Programming Guide](https://developer.apple.com/library/archive/documentation/General/Conceptual/ConcurrencyProgrammingGuide/)
- [libdispatch Documentation](https://developer.apple.com/documentation/dispatch)
- [WWDC - Modernizing Grand Central Dispatch Usage](https://developer.apple.com/videos/play/wwdc2017/706/)
