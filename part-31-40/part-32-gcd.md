# Part 32: Grand Central Dispatch (GCD)

## บทนำ: ทำไมต้องใช้ Concurrency?

ในการพัฒนาแอปพลิเคชัน iOS/macOS สมัยใหม่ การทำงานแบบ concurrent (พร้อมกัน) เป็นสิ่งจำเป็นอย่างยิ่ง เพราะ:

- **UI ต้องตอบสนองเสมอ**: ผู้ใช้คาดหวังว่าแอปจะไม่ค้างขณะโหลดข้อมูล
- **CPU มีหลาย core**: การใช้งาน core เดียวเป็นการสิ้นเปลืองทรัพยากร
- **I/O operations ใช้เวลานาน**: การอ่านไฟล์, network request ไม่ควรบล็อก main thread

**Grand Central Dispatch (GCD)** หรือ **libdispatch** คือ framework ระดับ system ของ Apple ที่ช่วยจัดการ concurrency ได้อย่างมีประสิทธิภาพ โดยซ่อนความซับซ้อนของการจัดการ thread ไว้ภายใน

---

## 32.1 พื้นฐาน Concurrency: Threads และ Queues

### Thread คืออะไร?

Thread คือหน่วยการประมวลผลที่เล็กที่สุดใน process หนึ่ง process สามารถมีได้หลาย thread ที่ทำงานพร้อมกัน

```objc
// ตัวอย่างการสร้าง Thread แบบดั้งเดิม (ไม่แนะนำ - ใช้ GCD แทน)
#import <Foundation/Foundation.h>

@interface ManualThread : NSObject
- (void)doSomeWork;
@end

@implementation ManualThread
- (void)doSomeWork {
    NSLog(@"Working on thread: %@", [NSThread currentThread]);
}
@end

int main(int argc, const char * argv[]) {
    @autoreleasepool {
        ManualThread *worker = [[ManualThread alloc] init];
        
        // วิธีดั้งเดิม - สร้าง NSThread โดยตรง
        NSThread *thread = [[NSThread alloc] initWithTarget:worker
                                                   selector:@selector(doSomeWork)
                                                     object:nil];
        [thread start];
        
        // GCD จัดการ thread pool ให้อัตโนมัติ - ดีกว่ามาก!
        dispatch_async(dispatch_get_global_queue(DISPATCH_QUEUE_PRIORITY_DEFAULT, 0), ^{
            NSLog(@"GCD manages threads automatically!");
        });
        
        [NSThread sleepForTimeInterval:1.0];
    }
    return 0;
}
```

### Queue คืออะไร?

Queue (คิว) ใน GCD คือโครงสร้างข้อมูลแบบ FIFO (First In, First Out) ที่รับ **blocks** (งาน) เข้ามาแล้วประมวลผลตามลำดับ

มี Queue 2 ประเภทหลัก:
1. **Serial Queue**: ทำงานทีละ task ตามลำดับ
2. **Concurrent Queue**: ทำงานหลาย task พร้อมกันได้

```
Serial Queue:
[Task1] --> [Task2] --> [Task3] --> [Task4]
 ทำก่อน    รอ Task1   รอ Task2   รอ Task3

Concurrent Queue:
[Task1] --|
[Task2] --|--> ทำงานพร้อมกัน
[Task3] --|
[Task4] --|
```

---

## 32.2 dispatch_queue_t: หัวใจของ GCD

`dispatch_queue_t` คือ type ของ queue object ใน GCD

```objc
#import <Foundation/Foundation.h>

int main(int argc, const char * argv[]) {
    @autoreleasepool {
        // 1. Main Queue - serial queue บน main thread
        dispatch_queue_t mainQueue = dispatch_get_main_queue();
        
        // 2. Global Queue - concurrent queue ที่ระบบสร้างให้
        dispatch_queue_t globalQueue = dispatch_get_global_queue(DISPATCH_QUEUE_PRIORITY_DEFAULT, 0);
        
        // 3. Custom Serial Queue
        dispatch_queue_t serialQueue = dispatch_queue_create("com.myapp.serial", DISPATCH_QUEUE_SERIAL);
        
        // 4. Custom Concurrent Queue
        dispatch_queue_t concurrentQueue = dispatch_queue_create("com.myapp.concurrent", DISPATCH_QUEUE_CONCURRENT);
        
        NSLog(@"Main Queue: %@", mainQueue);
        NSLog(@"Global Queue: %@", globalQueue);
        NSLog(@"Serial Queue: %@", serialQueue);
        NSLog(@"Concurrent Queue: %@", concurrentQueue);
        
        // ส่งงานเข้า queue แบบ async
        dispatch_async(serialQueue, ^{
            NSLog(@"Task on serial queue, thread: %@", [NSThread currentThread]);
        });
        
        dispatch_async(concurrentQueue, ^{
            NSLog(@"Task on concurrent queue, thread: %@", [NSThread currentThread]);
        });
        
        [NSThread sleepForTimeInterval:1.0];
    }
    return 0;
}
```

---

## 32.3 Main Queue: dispatch_get_main_queue()

Main queue เป็น **serial queue** พิเศษที่ทำงานบน main thread เสมอ ใช้สำหรับการอัพเดท UI

```objc
#import <Foundation/Foundation.h>

// จำลอง Network Request
void simulateNetworkRequest(void (^completion)(NSData *data, NSError *error)) {
    dispatch_async(dispatch_get_global_queue(DISPATCH_QUEUE_PRIORITY_DEFAULT, 0), ^{
        // จำลองการรอ network
        [NSThread sleepForTimeInterval:2.0];
        
        NSString *responseString = @"{'status': 'success', 'data': 'Hello from server'}";
        NSData *data = [responseString dataUsingEncoding:NSUTF8StringEncoding];
        
        // เรียก completion block บน background thread
        completion(data, nil);
    });
}

int main(int argc, const char * argv[]) {
    @autoreleasepool {
        NSLog(@"Starting network request...");
        NSLog(@"Main thread: %@", [NSThread mainThread]);
        
        simulateNetworkRequest(^(NSData *data, NSError *error) {
            // ตอนนี้อยู่บน background thread
            NSLog(@"Got data on thread: %@", [NSThread currentThread]);
            NSLog(@"Is main thread: %@", [NSThread isMainThread] ? @"YES" : @"NO");
            
            // ต้องอัพเดท UI บน main thread เท่านั้น!
            dispatch_async(dispatch_get_main_queue(), ^{
                // ตอนนี้อยู่บน main thread
                NSLog(@"Updating UI on main thread: %@", [NSThread currentThread]);
                NSLog(@"Is main thread: %@", [NSThread isMainThread] ? @"YES" : @"NO");
                
                // [self.label setText:@"Data loaded!"];  // อัพเดท UI ที่นี่
                NSString *text = [[NSString alloc] initWithData:data encoding:NSUTF8StringEncoding];
                NSLog(@"UI Updated with: %@", text);
            });
        });
        
        // รอให้งานเสร็จ
        [NSThread sleepForTimeInterval:3.0];
        NSLog(@"Done!");
    }
    return 0;
}
```

### ข้อควรระวัง: อย่า dispatch_sync บน main thread จาก main thread!

```objc
// WRONG! จะเกิด DEADLOCK!
dispatch_sync(dispatch_get_main_queue(), ^{
    NSLog(@"This will DEADLOCK!");
});

// CORRECT: ใช้ dispatch_async แทน
dispatch_async(dispatch_get_main_queue(), ^{
    NSLog(@"This is safe!");
});
```

---

## 32.4 Global Queues และ QOS Classes

GCD มี global concurrent queues หลายระดับตาม **Quality of Service (QoS)**:

| QoS Class | ลำดับความสำคัญ | ใช้สำหรับ |
|-----------|----------------|-----------|
| `QOS_CLASS_USER_INTERACTIVE` | สูงสุด | Animation, event handling, UI updates |
| `QOS_CLASS_USER_INITIATED` | สูง | งานที่ user รอผล เช่น เปิดไฟล์ |
| `QOS_CLASS_DEFAULT` | ปานกลาง | งานทั่วไป |
| `QOS_CLASS_UTILITY` | ต่ำ | งานที่ใช้เวลานาน เช่น download |
| `QOS_CLASS_BACKGROUND` | ต่ำสุด | Backup, sync, indexing |

```objc
#import <Foundation/Foundation.h>

void demonstrateQoSClasses(void) {
    // User Interactive - งานที่ต้องทำทันทีสำหรับ UI
    dispatch_queue_t userInteractiveQueue = dispatch_get_global_queue(QOS_CLASS_USER_INTERACTIVE, 0);
    
    // User Initiated - user รอผลลัพธ์
    dispatch_queue_t userInitiatedQueue = dispatch_get_global_queue(QOS_CLASS_USER_INITIATED, 0);
    
    // Default
    dispatch_queue_t defaultQueue = dispatch_get_global_queue(QOS_CLASS_DEFAULT, 0);
    
    // Utility - งานที่ใช้เวลาแต่ user รับรู้ progress
    dispatch_queue_t utilityQueue = dispatch_get_global_queue(QOS_CLASS_UTILITY, 0);
    
    // Background - งานที่ user ไม่รับรู้โดยตรง
    dispatch_queue_t backgroundQueue = dispatch_get_global_queue(QOS_CLASS_BACKGROUND, 0);
    
    // ส่งงานตาม priority
    dispatch_async(backgroundQueue, ^{
        NSLog(@"Background: syncing data...");
        [NSThread sleepForTimeInterval:0.5];
        NSLog(@"Background: sync complete");
    });
    
    dispatch_async(utilityQueue, ^{
        NSLog(@"Utility: downloading file...");
        [NSThread sleepForTimeInterval:0.3];
        NSLog(@"Utility: download complete");
    });
    
    dispatch_async(userInitiatedQueue, ^{
        NSLog(@"UserInitiated: loading document...");
        [NSThread sleepForTimeInterval:0.1];
        NSLog(@"UserInitiated: document loaded");
    });
    
    dispatch_async(userInteractiveQueue, ^{
        NSLog(@"UserInteractive: updating animation...");
        NSLog(@"UserInteractive: animation updated");
    });
}

int main(int argc, const char * argv[]) {
    @autoreleasepool {
        demonstrateQoSClasses();
        [NSThread sleepForTimeInterval:2.0];
    }
    return 0;
}
```

### การใช้ Priority แบบเก่า (ยังรองรับแต่ไม่แนะนำ)

```objc
// แบบเก่า - ใช้ DISPATCH_QUEUE_PRIORITY
dispatch_get_global_queue(DISPATCH_QUEUE_PRIORITY_HIGH, 0);
dispatch_get_global_queue(DISPATCH_QUEUE_PRIORITY_DEFAULT, 0);
dispatch_get_global_queue(DISPATCH_QUEUE_PRIORITY_LOW, 0);
dispatch_get_global_queue(DISPATCH_QUEUE_PRIORITY_BACKGROUND, 0);

// แบบใหม่ - ใช้ QoS (แนะนำ)
dispatch_get_global_queue(QOS_CLASS_USER_INITIATED, 0);
dispatch_get_global_queue(QOS_CLASS_DEFAULT, 0);
dispatch_get_global_queue(QOS_CLASS_UTILITY, 0);
dispatch_get_global_queue(QOS_CLASS_BACKGROUND, 0);
```

---

## 32.5 การสร้าง Custom Serial Queue

Serial queue ทำงานทีละ task ตามลำดับ เหมาะสำหรับปกป้อง shared resource

```objc
#import <Foundation/Foundation.h>

// ตัวอย่าง: Counter ที่ thread-safe ด้วย Serial Queue
@interface ThreadSafeCounter : NSObject {
    NSInteger _count;
    dispatch_queue_t _queue;
}

- (void)increment;
- (void)decrement;
- (NSInteger)count;

@end

@implementation ThreadSafeCounter

- (instancetype)init {
    self = [super init];
    if (self) {
        _count = 0;
        // สร้าง serial queue เฉพาะสำหรับ counter นี้
        _queue = dispatch_queue_create("com.myapp.counter.queue", DISPATCH_QUEUE_SERIAL);
    }
    return self;
}

- (void)increment {
    // ทุก operation ต้องผ่าน queue เพื่อความ thread-safe
    dispatch_async(_queue, ^{
        self->_count++;
        NSLog(@"Incremented to: %ld on thread: %@", (long)self->_count, [NSThread currentThread]);
    });
}

- (void)decrement {
    dispatch_async(_queue, ^{
        self->_count--;
        NSLog(@"Decremented to: %ld on thread: %@", (long)self->_count, [NSThread currentThread]);
    });
}

- (NSInteger)count {
    // ใช้ sync เพื่อรอผลค่าก่อน return
    __block NSInteger result;
    dispatch_sync(_queue, ^{
        result = self->_count;
    });
    return result;
}

@end

int main(int argc, const char * argv[]) {
    @autoreleasepool {
        ThreadSafeCounter *counter = [[ThreadSafeCounter alloc] init];
        
        // เรียกจากหลาย thread พร้อมกัน
        dispatch_queue_t concurrent = dispatch_get_global_queue(QOS_CLASS_DEFAULT, 0);
        
        for (int i = 0; i < 5; i++) {
            dispatch_async(concurrent, ^{
                [counter increment];
            });
        }
        
        for (int i = 0; i < 2; i++) {
            dispatch_async(concurrent, ^{
                [counter decrement];
            });
        }
        
        [NSThread sleepForTimeInterval:1.0];
        NSLog(@"Final count: %ld", (long)[counter count]);
    }
    return 0;
}
```

---

## 32.6 การสร้าง Custom Concurrent Queue

Concurrent queue อนุญาตให้หลาย task ทำงานพร้อมกัน แต่ต้องระวัง race condition

```objc
#import <Foundation/Foundation.h>

int main(int argc, const char * argv[]) {
    @autoreleasepool {
        // สร้าง concurrent queue พร้อม QoS attribute
        dispatch_queue_attr_t attr = dispatch_queue_attr_make_with_qos_class(
            DISPATCH_QUEUE_CONCURRENT,
            QOS_CLASS_USER_INITIATED,
            0
        );
        dispatch_queue_t imageProcessingQueue = dispatch_queue_create("com.myapp.imageProcessing", attr);
        
        NSArray *imageNames = @[@"photo1.jpg", @"photo2.jpg", @"photo3.jpg", 
                                @"photo4.jpg", @"photo5.jpg"];
        
        NSLog(@"Starting image processing...");
        NSDate *startTime = [NSDate date];
        
        // ประมวลผลภาพทั้งหมดพร้อมกัน
        for (NSString *imageName in imageNames) {
            dispatch_async(imageProcessingQueue, ^{
                // จำลองการประมวลผลภาพ
                NSLog(@"Processing %@ on thread %@", imageName, [NSThread currentThread]);
                [NSThread sleepForTimeInterval:0.5]; // จำลองงาน
                NSLog(@"Finished processing %@", imageName);
            });
        }
        
        [NSThread sleepForTimeInterval:2.0];
        
        NSTimeInterval elapsed = [[NSDate date] timeIntervalSinceDate:startTime];
        NSLog(@"Total time: %.2f seconds", elapsed);
        // เนื่องจากทำงานพร้อมกัน จะใช้เวลาน้อยกว่า 5 * 0.5 = 2.5 วินาที
    }
    return 0;
}
```

---

## 32.7 dispatch_async vs dispatch_sync

### dispatch_async: ส่งงานแล้วไปทำอย่างอื่นต่อได้ทันที

```objc
#import <Foundation/Foundation.h>

int main(int argc, const char * argv[]) {
    @autoreleasepool {
        dispatch_queue_t queue = dispatch_queue_create("com.myapp.demo", DISPATCH_QUEUE_SERIAL);
        
        NSLog(@"=== dispatch_async Demo ===");
        NSLog(@"Before async - Thread: %@", [NSThread currentThread]);
        
        dispatch_async(queue, ^{
            NSLog(@"Inside async block - Thread: %@", [NSThread currentThread]);
            [NSThread sleepForTimeInterval:1.0];
            NSLog(@"Async block completed");
        });
        
        // บรรทัดนี้ทำงานทันทีโดยไม่รอ block ข้างบน
        NSLog(@"After async - continues immediately!");
        
        [NSThread sleepForTimeInterval:2.0];
    }
    return 0;
}
```

### dispatch_sync: รอจนงานเสร็จก่อนทำต่อ

```objc
#import <Foundation/Foundation.h>

int main(int argc, const char * argv[]) {
    @autoreleasepool {
        dispatch_queue_t queue = dispatch_queue_create("com.myapp.demo", DISPATCH_QUEUE_SERIAL);
        
        NSLog(@"=== dispatch_sync Demo ===");
        NSLog(@"Before sync - Thread: %@", [NSThread currentThread]);
        
        dispatch_sync(queue, ^{
            NSLog(@"Inside sync block - Thread: %@", [NSThread currentThread]);
            [NSThread sleepForTimeInterval:1.0];
            NSLog(@"Sync block completed");
        });
        
        // บรรทัดนี้จะทำงานก็ต่อเมื่อ block ข้างบนเสร็จแล้วเท่านั้น
        NSLog(@"After sync - waited for completion");
    }
    return 0;
}
```

### เมื่อไรใช้อะไร?

```objc
#import <Foundation/Foundation.h>

@interface DataProcessor : NSObject
@property (nonatomic, strong) NSMutableArray *processedData;
@property (nonatomic, strong) dispatch_queue_t processingQueue;
@end

@implementation DataProcessor

- (instancetype)init {
    self = [super init];
    if (self) {
        _processedData = [NSMutableArray array];
        _processingQueue = dispatch_queue_create("com.myapp.dataprocessing", DISPATCH_QUEUE_SERIAL);
    }
    return self;
}

// ใช้ async เมื่อไม่ต้องการผลทันที
- (void)processDataAsync:(NSData *)data completion:(void(^)(NSString *result))completion {
    dispatch_async(_processingQueue, ^{
        // ประมวลผลข้อมูล
        [NSThread sleepForTimeInterval:0.5]; // จำลองงาน
        NSString *result = [NSString stringWithFormat:@"Processed %lu bytes", (unsigned long)data.length];
        
        // ส่งผลลัพธ์กลับบน main thread
        dispatch_async(dispatch_get_main_queue(), ^{
            completion(result);
        });
    });
}

// ใช้ sync เมื่อต้องการผลทันที (ใช้ระวัง deadlock!)
- (NSArray *)getCurrentDataSync {
    __block NSArray *result;
    dispatch_sync(_processingQueue, ^{
        result = [self->_processedData copy];
    });
    return result;
}

@end

int main(int argc, const char * argv[]) {
    @autoreleasepool {
        DataProcessor *processor = [[DataProcessor alloc] init];
        NSData *testData = [@"Hello World" dataUsingEncoding:NSUTF8StringEncoding];
        
        [processor processDataAsync:testData completion:^(NSString *result) {
            NSLog(@"Async result: %@", result);
        }];
        
        NSArray *currentData = [processor getCurrentDataSync];
        NSLog(@"Sync result: %@", currentData);
        
        [NSThread sleepForTimeInterval:1.0];
    }
    return 0;
}
```

---

## 32.8 DEADLOCK: สาเหตุและการหลีกเลี่ยง

**Deadlock** เกิดขึ้นเมื่อ thread รอ thread อื่นที่กำลังรอตัวมันเองอยู่

### กรณีที่ 1: dispatch_sync บน Serial Queue เดิม

```objc
#import <Foundation/Foundation.h>

void demonstrateDeadlock(void) {
    dispatch_queue_t serialQueue = dispatch_queue_create("com.myapp.serial", DISPATCH_QUEUE_SERIAL);
    
    dispatch_async(serialQueue, ^{
        NSLog(@"Outer block executing...");
        
        // DEADLOCK! 
        // serialQueue กำลังรัน outer block อยู่
        // แต่ dispatch_sync รอให้ serialQueue ว่างก่อน
        // ทำให้เกิดการรอกันเป็นวงกลม (deadlock)
        
        // dispatch_sync(serialQueue, ^{  // <-- อย่าทำแบบนี้!
        //     NSLog(@"Inner block - will NEVER execute");
        // });
        
        // วิธีแก้: ใช้ dispatch_async แทน
        dispatch_async(serialQueue, ^{
            NSLog(@"Inner block - this is safe with async");
        });
        
        NSLog(@"Outer block continues...");
    });
}

// กรณีที่ 2: Main Thread Deadlock
void mainThreadDeadlock(void) {
    // อย่าเรียกจาก main thread!
    // dispatch_sync(dispatch_get_main_queue(), ^{
    //     NSLog(@"DEADLOCK!");  // main thread รอตัวเอง
    // });
    
    // วิธีที่ถูกต้อง: ตรวจสอบก่อน
    if ([NSThread isMainThread]) {
        NSLog(@"Already on main thread, execute directly");
        // ทำงานโดยตรงโดยไม่ต้อง dispatch
    } else {
        dispatch_async(dispatch_get_main_queue(), ^{
            NSLog(@"Dispatched to main thread safely");
        });
    }
}

// Helper function ที่ปลอดภัย
void dispatchOnMain(dispatch_block_t block) {
    if ([NSThread isMainThread]) {
        block(); // ทำงานโดยตรงถ้าอยู่บน main thread แล้ว
    } else {
        dispatch_async(dispatch_get_main_queue(), block);
    }
}

int main(int argc, const char * argv[]) {
    @autoreleasepool {
        demonstrateDeadlock();
        mainThreadDeadlock();
        
        dispatchOnMain(^{
            NSLog(@"Safe dispatch to main: %@", [NSThread isMainThread] ? @"main" : @"background");
        });
        
        [NSThread sleepForTimeInterval:1.0];
    }
    return 0;
}
```

---

## 32.9 dispatch_after: การทำงานแบบ Delayed

```objc
#import <Foundation/Foundation.h>

int main(int argc, const char * argv[]) {
    @autoreleasepool {
        NSLog(@"Starting delayed tasks...");
        
        // ทำงานหลังจาก 2 วินาที
        dispatch_time_t delay2s = dispatch_time(DISPATCH_TIME_NOW, (int64_t)(2.0 * NSEC_PER_SEC));
        dispatch_after(delay2s, dispatch_get_main_queue(), ^{
            NSLog(@"This runs after 2 seconds on main queue");
        });
        
        // ทำงานหลังจาก 1 วินาทีบน background queue
        dispatch_time_t delay1s = dispatch_time(DISPATCH_TIME_NOW, (int64_t)(1.0 * NSEC_PER_SEC));
        dispatch_after(delay1s, dispatch_get_global_queue(QOS_CLASS_DEFAULT, 0), ^{
            NSLog(@"This runs after 1 second on background queue");
        });
        
        // ทำงานหลังจาก 500ms
        dispatch_time_t delay500ms = dispatch_time(DISPATCH_TIME_NOW, (int64_t)(0.5 * NSEC_PER_SEC));
        dispatch_after(delay500ms, dispatch_get_main_queue(), ^{
            NSLog(@"This runs after 500ms");
        });
        
        // ตัวอย่างจริง: แสดง tooltip แล้วซ่อนหลัง 3 วินาที
        NSLog(@"Showing tooltip...");
        dispatch_time_t hideDelay = dispatch_time(DISPATCH_TIME_NOW, (int64_t)(3.0 * NSEC_PER_SEC));
        dispatch_after(hideDelay, dispatch_get_main_queue(), ^{
            NSLog(@"Hiding tooltip after 3 seconds");
            // [self.tooltipView setHidden:YES];
        });
        
        [NSThread sleepForTimeInterval:4.0];
    }
    return 0;
}
```

### dispatch_after กับ Cancellation

```objc
#import <Foundation/Foundation.h>

// dispatch_after ไม่มีการยกเลิกโดยตรง ต้องใช้ flag
@interface DelayedTask : NSObject
@property (nonatomic, assign) BOOL cancelled;
- (void)scheduleAfter:(NSTimeInterval)delay task:(void(^)(void))task;
- (void)cancel;
@end

@implementation DelayedTask

- (void)scheduleAfter:(NSTimeInterval)delay task:(void(^)(void))task {
    self.cancelled = NO;
    
    dispatch_time_t when = dispatch_time(DISPATCH_TIME_NOW, (int64_t)(delay * NSEC_PER_SEC));
    dispatch_after(when, dispatch_get_main_queue(), ^{
        if (!self.cancelled) {
            task();
        } else {
            NSLog(@"Task was cancelled, skipping...");
        }
    });
}

- (void)cancel {
    self.cancelled = YES;
}

@end

int main(int argc, const char * argv[]) {
    @autoreleasepool {
        DelayedTask *task = [[DelayedTask alloc] init];
        
        [task scheduleAfter:3.0 task:^{
            NSLog(@"This task might or might not run");
        }];
        
        // ยกเลิกก่อนเวลา
        dispatch_after(dispatch_time(DISPATCH_TIME_NOW, (int64_t)(1.5 * NSEC_PER_SEC)),
                       dispatch_get_main_queue(), ^{
            [task cancel];
            NSLog(@"Task cancelled!");
        });
        
        [NSThread sleepForTimeInterval:4.0];
    }
    return 0;
}
```

---

## 32.10 dispatch_once: Thread-Safe Singleton Pattern

`dispatch_once` รับประกันว่า block จะถูก execute เพียงครั้งเดียวตลอด lifetime ของ process แม้จะเรียกจากหลาย thread พร้อมกัน

```objc
#import <Foundation/Foundation.h>

// Singleton ที่ thread-safe ด้วย dispatch_once
@interface DatabaseManager : NSObject

@property (nonatomic, strong) NSString *databasePath;
@property (nonatomic, assign) BOOL isConnected;

+ (instancetype)sharedManager;
- (void)connect;
- (void)disconnect;
- (NSArray *)executeQuery:(NSString *)query;

@end

@implementation DatabaseManager

+ (instancetype)sharedManager {
    static DatabaseManager *instance = nil;
    static dispatch_once_t onceToken;
    
    dispatch_once(&onceToken, ^{
        instance = [[self alloc] init];
        NSLog(@"DatabaseManager created on thread: %@", [NSThread currentThread]);
    });
    
    return instance;
}

- (instancetype)init {
    self = [super init];
    if (self) {
        _databasePath = @"/var/db/myapp.sqlite";
        _isConnected = NO;
        NSLog(@"DatabaseManager initialized");
    }
    return self;
}

- (void)connect {
    _isConnected = YES;
    NSLog(@"Connected to database at: %@", _databasePath);
}

- (void)disconnect {
    _isConnected = NO;
    NSLog(@"Disconnected from database");
}

- (NSArray *)executeQuery:(NSString *)query {
    if (!_isConnected) {
        NSLog(@"Not connected! Please connect first.");
        return nil;
    }
    NSLog(@"Executing: %@", query);
    return @[@"Row1", @"Row2", @"Row3"]; // ผลลัพธ์จำลอง
}

@end

int main(int argc, const char * argv[]) {
    @autoreleasepool {
        // เรียกจากหลาย thread พร้อมกัน - ควรได้ instance เดียวกัน
        dispatch_queue_t concurrent = dispatch_get_global_queue(QOS_CLASS_DEFAULT, 0);
        
        for (int i = 0; i < 5; i++) {
            dispatch_async(concurrent, ^{
                DatabaseManager *mgr = [DatabaseManager sharedManager];
                NSLog(@"Got instance: %p on thread: %@", (void *)mgr, [NSThread currentThread]);
            });
        }
        
        [NSThread sleepForTimeInterval:0.5];
        
        // ทุก call จะได้ instance เดียวกัน
        DatabaseManager *db1 = [DatabaseManager sharedManager];
        DatabaseManager *db2 = [DatabaseManager sharedManager];
        
        NSLog(@"db1 == db2: %@", (db1 == db2) ? @"YES (same instance)" : @"NO (different instances)");
        
        [db1 connect];
        NSArray *results = [db1 executeQuery:@"SELECT * FROM users"];
        NSLog(@"Results: %@", results);
        [db1 disconnect];
    }
    return 0;
}
```

### dispatch_once สำหรับ Lazy Initialization

```objc
#import <Foundation/Foundation.h>

@interface AppConfig : NSObject

+ (NSDictionary *)defaultSettings;
+ (NSDateFormatter *)sharedDateFormatter;
+ (NSNumberFormatter *)sharedCurrencyFormatter;

@end

@implementation AppConfig

+ (NSDictionary *)defaultSettings {
    static NSDictionary *settings = nil;
    static dispatch_once_t onceToken;
    
    dispatch_once(&onceToken, ^{
        settings = @{
            @"theme": @"light",
            @"language": @"th",
            @"maxRetries": @3,
            @"timeout": @30.0,
            @"apiBaseURL": @"https://api.example.com/v1"
        };
        NSLog(@"Default settings initialized");
    });
    
    return settings;
}

+ (NSDateFormatter *)sharedDateFormatter {
    static NSDateFormatter *formatter = nil;
    static dispatch_once_t onceToken;
    
    dispatch_once(&onceToken, ^{
        formatter = [[NSDateFormatter alloc] init];
        [formatter setDateFormat:@"yyyy-MM-dd HH:mm:ss"];
        [formatter setLocale:[NSLocale localeWithLocaleIdentifier:@"th_TH"]];
        NSLog(@"DateFormatter initialized");
    });
    
    return formatter;
}

+ (NSNumberFormatter *)sharedCurrencyFormatter {
    static NSNumberFormatter *formatter = nil;
    static dispatch_once_t onceToken;
    
    dispatch_once(&onceToken, ^{
        formatter = [[NSNumberFormatter alloc] init];
        [formatter setNumberStyle:NSNumberFormatterCurrencyStyle];
        [formatter setLocale:[NSLocale localeWithLocaleIdentifier:@"th_TH"]];
        NSLog(@"CurrencyFormatter initialized");
    });
    
    return formatter;
}

@end

int main(int argc, const char * argv[]) {
    @autoreleasepool {
        // เรียกหลายครั้ง แต่ initialize เพียงครั้งเดียว
        NSDictionary *settings1 = [AppConfig defaultSettings];
        NSDictionary *settings2 = [AppConfig defaultSettings];
        
        NSLog(@"Settings same object: %@", (settings1 == settings2) ? @"YES" : @"NO");
        NSLog(@"Theme: %@", settings1[@"theme"]);
        
        NSDateFormatter *df = [AppConfig sharedDateFormatter];
        NSLog(@"Current date: %@", [df stringFromDate:[NSDate date]]);
        
        NSNumberFormatter *cf = [AppConfig sharedCurrencyFormatter];
        NSLog(@"Price: %@", [cf stringFromNumber:@1299.99]);
    }
    return 0;
}
```

---

## 32.11 dispatch_group: รอหลาย Tasks พร้อมกัน

`dispatch_group` ใช้เมื่อต้องการรอให้หลาย async tasks เสร็จก่อนทำงานต่อ

```objc
#import <Foundation/Foundation.h>

// ตัวอย่าง: โหลดข้อมูลจาก 3 API พร้อมกัน แล้วรวมผล
int main(int argc, const char * argv[]) {
    @autoreleasepool {
        dispatch_group_t group = dispatch_group_create();
        dispatch_queue_t queue = dispatch_get_global_queue(QOS_CLASS_USER_INITIATED, 0);
        
        __block NSString *userData = nil;
        __block NSString *productsData = nil;
        __block NSString *ordersData = nil;
        
        NSLog(@"Starting parallel data fetching...");
        NSDate *start = [NSDate date];
        
        // Task 1: โหลดข้อมูล User
        dispatch_group_async(group, queue, ^{
            NSLog(@"Fetching user data...");
            [NSThread sleepForTimeInterval:1.0]; // จำลอง network
            userData = @"{'id': 1, 'name': 'สมชาย', 'email': 'somchai@example.com'}";
            NSLog(@"User data fetched");
        });
        
        // Task 2: โหลดข้อมูล Products
        dispatch_group_async(group, queue, ^{
            NSLog(@"Fetching products data...");
            [NSThread sleepForTimeInterval:1.5]; // จำลอง network
            productsData = @"[{'id': 1, 'name': 'iPhone', 'price': 35000}]";
            NSLog(@"Products data fetched");
        });
        
        // Task 3: โหลดข้อมูล Orders
        dispatch_group_async(group, queue, ^{
            NSLog(@"Fetching orders data...");
            [NSThread sleepForTimeInterval:0.8]; // จำลอง network
            ordersData = @"[{'id': 101, 'total': 35000, 'status': 'shipped'}]";
            NSLog(@"Orders data fetched");
        });
        
        // รอทุก task เสร็จแล้วทำงานต่อ
        dispatch_group_notify(group, dispatch_get_main_queue(), ^{
            NSTimeInterval elapsed = [[NSDate date] timeIntervalSinceDate:start];
            NSLog(@"\n=== All data fetched in %.2f seconds ===", elapsed);
            NSLog(@"User: %@", userData);
            NSLog(@"Products: %@", productsData);
            NSLog(@"Orders: %@", ordersData);
            // อัพเดท UI ที่นี่
        });
        
        // รอให้ semaphore/group จบก่อนออกจาก main
        dispatch_group_wait(group, DISPATCH_TIME_FOREVER);
        [NSThread sleepForTimeInterval:0.5];
    }
    return 0;
}
```

---

## 32.12 dispatch_group_enter / dispatch_group_leave

ใช้เมื่อ tasks ภายใน group มี async operation ซ้อนอยู่อีกชั้น (เช่น network callbacks)

```objc
#import <Foundation/Foundation.h>

// จำลอง async network request
void fetchURL(NSString *url, void(^completion)(NSString *result, NSError *error)) {
    dispatch_async(dispatch_get_global_queue(QOS_CLASS_DEFAULT, 0), ^{
        [NSThread sleepForTimeInterval:arc4random_uniform(2) + 0.5];
        
        if (arc4random_uniform(10) < 9) { // 90% success rate
            NSString *result = [NSString stringWithFormat:@"Response from %@", url];
            completion(result, nil);
        } else {
            NSError *error = [NSError errorWithDomain:@"NetworkError"
                                                 code:503
                                             userInfo:@{NSLocalizedDescriptionKey: @"Service Unavailable"}];
            completion(nil, error);
        }
    });
}

int main(int argc, const char * argv[]) {
    @autoreleasepool {
        dispatch_group_t group = dispatch_group_create();
        dispatch_queue_t resultQueue = dispatch_queue_create("com.myapp.results", DISPATCH_QUEUE_SERIAL);
        
        NSMutableArray *results = [NSMutableArray array];
        NSMutableArray *errors = [NSMutableArray array];
        
        NSArray *urls = @[
            @"https://api.example.com/users",
            @"https://api.example.com/products",
            @"https://api.example.com/orders",
            @"https://api.example.com/categories"
        ];
        
        NSLog(@"Starting requests to %lu URLs", (unsigned long)urls.count);
        
        for (NSString *url in urls) {
            // บอก group ว่ากำลังจะมีงานเพิ่ม
            dispatch_group_enter(group);
            
            fetchURL(url, ^(NSString *result, NSError *error) {
                // เก็บผลลัพธ์บน serial queue เพื่อความ thread-safe
                dispatch_async(resultQueue, ^{
                    if (result) {
                        [results addObject:result];
                        NSLog(@"Got result from: %@", url);
                    } else {
                        [errors addObject:error];
                        NSLog(@"Error from %@: %@", url, error.localizedDescription);
                    }
                    
                    // บอก group ว่างานชิ้นนี้เสร็จแล้ว
                    dispatch_group_leave(group);
                });
            });
        }
        
        // เรียกเมื่อ enter/leave สมดุลกัน (ทุก task เสร็จ)
        dispatch_group_notify(group, dispatch_get_main_queue(), ^{
            NSLog(@"\n=== All requests completed ===");
            NSLog(@"Successful: %lu", (unsigned long)results.count);
            NSLog(@"Failed: %lu", (unsigned long)errors.count);
            
            for (NSString *result in results) {
                NSLog(@"  ✓ %@", result);
            }
            for (NSError *error in errors) {
                NSLog(@"  ✗ %@", error.localizedDescription);
            }
        });
        
        // รอ group จบ (สำหรับ command-line app)
        dispatch_group_wait(group, dispatch_time(DISPATCH_TIME_NOW, 10 * NSEC_PER_SEC));
        [NSThread sleepForTimeInterval:0.5];
    }
    return 0;
}
```

---

## 32.13 dispatch_group_notify vs dispatch_group_wait

```objc
#import <Foundation/Foundation.h>

int main(int argc, const char * argv[]) {
    @autoreleasepool {
        dispatch_group_t group = dispatch_group_create();
        dispatch_queue_t queue = dispatch_get_global_queue(QOS_CLASS_DEFAULT, 0);
        
        for (int i = 1; i <= 3; i++) {
            dispatch_group_async(group, queue, ^{
                NSLog(@"Task %d starting", i);
                [NSThread sleepForTimeInterval:i * 0.5];
                NSLog(@"Task %d done", i);
            });
        }
        
        // วิธีที่ 1: dispatch_group_notify (async - ไม่บล็อก)
        dispatch_group_notify(group, dispatch_get_main_queue(), ^{
            NSLog(@"[notify] All tasks completed! (non-blocking)");
        });
        
        NSLog(@"After notify setup - code continues here immediately");
        
        // วิธีที่ 2: dispatch_group_wait (sync - บล็อก thread ปัจจุบัน)
        // แต่ต้องสร้าง group ใหม่เพราะ group เดิมถูก notify แล้ว
        dispatch_group_t group2 = dispatch_group_create();
        
        for (int i = 1; i <= 3; i++) {
            dispatch_group_async(group2, queue, ^{
                NSLog(@"Group2 Task %d", i);
                [NSThread sleepForTimeInterval:0.3];
            });
        }
        
        // Timeout: รอสูงสุด 5 วินาที
        dispatch_time_t timeout = dispatch_time(DISPATCH_TIME_NOW, 5 * NSEC_PER_SEC);
        long result = dispatch_group_wait(group2, timeout);
        
        if (result == 0) {
            NSLog(@"[wait] All group2 tasks completed within timeout");
        } else {
            NSLog(@"[wait] Timeout! Some tasks still running");
        }
        
        [NSThread sleepForTimeInterval:2.5];
    }
    return 0;
}
```

---

## 32.14 dispatch_barrier_async: Reader-Writer Pattern

`dispatch_barrier_async` ใช้กับ concurrent queue เพื่อสร้าง "barrier" - งานทั้งหมดก่อนหน้าต้องเสร็จก่อน และงานหลัง barrier จะรอ barrier เสร็จก่อน

เหมาะมากสำหรับ **Reader-Writer pattern**: อ่านได้หลายคนพร้อมกัน แต่เขียนต้องได้คนเดียว

```objc
#import <Foundation/Foundation.h>

// Thread-safe Dictionary ด้วย dispatch_barrier
@interface ThreadSafeDictionary : NSObject {
    NSMutableDictionary *_dictionary;
    dispatch_queue_t _queue;
}

- (void)setObject:(id)object forKey:(NSString *)key;
- (id)objectForKey:(NSString *)key;
- (void)removeObjectForKey:(NSString *)key;
- (NSDictionary *)allObjects;

@end

@implementation ThreadSafeDictionary

- (instancetype)init {
    self = [super init];
    if (self) {
        _dictionary = [NSMutableDictionary dictionary];
        // ต้องใช้ CONCURRENT queue สำหรับ barrier pattern
        _queue = dispatch_queue_create("com.myapp.threadsafedict", DISPATCH_QUEUE_CONCURRENT);
    }
    return self;
}

// Write operation - ใช้ barrier เพื่อ exclusive access
- (void)setObject:(id)object forKey:(NSString *)key {
    dispatch_barrier_async(_queue, ^{
        // ระหว่างนี้: readers ทั้งหมดหยุด, เขียนคนเดียว
        self->_dictionary[key] = object;
        NSLog(@"Written: %@ = %@ on thread %@", key, object, [NSThread currentThread]);
    });
}

// Read operation - หลาย readers พร้อมกันได้
- (id)objectForKey:(NSString *)key {
    __block id result;
    dispatch_sync(_queue, ^{
        // หลาย reads ทำงานพร้อมกันได้
        result = self->_dictionary[key];
    });
    return result;
}

- (void)removeObjectForKey:(NSString *)key {
    dispatch_barrier_async(_queue, ^{
        [self->_dictionary removeObjectForKey:key];
        NSLog(@"Removed key: %@", key);
    });
}

- (NSDictionary *)allObjects {
    __block NSDictionary *result;
    dispatch_sync(_queue, ^{
        result = [self->_dictionary copy];
    });
    return result;
}

@end

int main(int argc, const char * argv[]) {
    @autoreleasepool {
        ThreadSafeDictionary *dict = [[ThreadSafeDictionary alloc] init];
        dispatch_queue_t concurrentQ = dispatch_get_global_queue(QOS_CLASS_DEFAULT, 0);
        
        // หลาย writers
        for (int i = 0; i < 5; i++) {
            dispatch_async(concurrentQ, ^{
                NSString *key = [NSString stringWithFormat:@"key%d", i];
                NSString *value = [NSString stringWithFormat:@"value%d", i];
                [dict setObject:value forKey:key];
            });
        }
        
        // หลาย readers พร้อมกัน
        for (int i = 0; i < 10; i++) {
            dispatch_async(concurrentQ, ^{
                NSString *key = [NSString stringWithFormat:@"key%d", i % 5];
                id value = [dict objectForKey:key];
                NSLog(@"Read: %@ = %@", key, value);
            });
        }
        
        [NSThread sleepForTimeInterval:1.0];
        
        NSDictionary *all = [dict allObjects];
        NSLog(@"\nFinal dictionary: %@", all);
    }
    return 0;
}
```

### dispatch_barrier_sync vs dispatch_barrier_async

```objc
// async barrier: ส่งงานแล้วไปทำต่อ (ส่วนใหญ่ใช้แบบนี้)
dispatch_barrier_async(concurrentQueue, ^{
    // write operation
});

// sync barrier: รอจน barrier เสร็จ (ใช้เมื่อต้องการผล)
dispatch_barrier_sync(concurrentQueue, ^{
    // write operation ที่ต้องรอจบก่อน
});
```

---

## 32.15 dispatch_semaphore: จำกัด Concurrent Access

`dispatch_semaphore` ใช้ควบคุมจำนวน tasks ที่ทำงานพร้อมกันได้ หรือใช้เป็น lock

```objc
#import <Foundation/Foundation.h>

// ตัวอย่าง: จำกัด concurrent network requests เป็น 3 พร้อมกัน
int main(int argc, const char * argv[]) {
    @autoreleasepool {
        // สร้าง semaphore ที่อนุญาตให้ทำงานได้สูงสุด 3 tasks พร้อมกัน
        dispatch_semaphore_t semaphore = dispatch_semaphore_create(3);
        dispatch_queue_t queue = dispatch_get_global_queue(QOS_CLASS_DEFAULT, 0);
        
        NSLog(@"Starting 10 concurrent tasks, limited to 3 at a time...");
        NSDate *start = [NSDate date];
        
        for (int i = 1; i <= 10; i++) {
            dispatch_async(queue, ^{
                // รอจนมี slot ว่าง (semaphore value > 0)
                dispatch_semaphore_wait(semaphore, DISPATCH_TIME_FOREVER);
                
                NSLog(@"Task %d started (active tasks ≤ 3)", i);
                [NSThread sleepForTimeInterval:1.0]; // จำลองงาน
                NSLog(@"Task %d completed", i);
                
                // คืน slot ให้ task อื่น
                dispatch_semaphore_signal(semaphore);
            });
        }
        
        // รอทุก task เสร็จ
        [NSThread sleepForTimeInterval:5.0];
        
        NSTimeInterval elapsed = [[NSDate date] timeIntervalSinceDate:start];
        NSLog(@"All tasks done in %.2f seconds", elapsed);
        // ถ้าทำทีละ 3 กลุ่ม: กลุ่ม1(1s) + กลุ่ม2(1s) + กลุ่ม3(1s) + กลุ่ม4(1s) ≈ 4s
    }
    return 0;
}
```

### Semaphore เป็น Lock (Binary Semaphore)

```objc
#import <Foundation/Foundation.h>

@interface FileWriter : NSObject {
    dispatch_semaphore_t _lock;
    NSMutableString *_buffer;
}

- (void)write:(NSString *)text;
- (NSString *)read;

@end

@implementation FileWriter

- (instancetype)init {
    self = [super init];
    if (self) {
        // Semaphore = 1 ทำหน้าที่เป็น mutex lock
        _lock = dispatch_semaphore_create(1);
        _buffer = [NSMutableString string];
    }
    return self;
}

- (void)write:(NSString *)text {
    dispatch_semaphore_wait(_lock, DISPATCH_TIME_FOREVER); // lock
    [_buffer appendString:text];
    [_buffer appendString:@"\n"];
    dispatch_semaphore_signal(_lock); // unlock
}

- (NSString *)read {
    dispatch_semaphore_wait(_lock, DISPATCH_TIME_FOREVER); // lock
    NSString *result = [_buffer copy];
    dispatch_semaphore_signal(_lock); // unlock
    return result;
}

@end

// ตัวอย่าง: ใช้ semaphore เปลี่ยน async เป็น sync (ระวัง deadlock!)
NSData* synchronousFetch(NSURL *url) {
    __block NSData *result = nil;
    dispatch_semaphore_t done = dispatch_semaphore_create(0);
    
    // Async task
    dispatch_async(dispatch_get_global_queue(QOS_CLASS_DEFAULT, 0), ^{
        // จำลอง async fetch
        [NSThread sleepForTimeInterval:1.0];
        result = [@"Fetched data" dataUsingEncoding:NSUTF8StringEncoding];
        
        dispatch_semaphore_signal(done); // บอกว่าเสร็จแล้ว
    });
    
    dispatch_semaphore_wait(done, DISPATCH_TIME_FOREVER); // รอ
    return result;
}

int main(int argc, const char * argv[]) {
    @autoreleasepool {
        FileWriter *writer = [[FileWriter alloc] init];
        dispatch_queue_t concurrent = dispatch_get_global_queue(QOS_CLASS_DEFAULT, 0);
        
        for (int i = 0; i < 5; i++) {
            dispatch_async(concurrent, ^{
                [writer write:[NSString stringWithFormat:@"Line %d from thread %@", i, [NSThread currentThread]]];
            });
        }
        
        [NSThread sleepForTimeInterval:0.5];
        NSLog(@"File contents:\n%@", [writer read]);
        
        // ตัวอย่าง synchronous fetch
        NSLog(@"\nFetching synchronously...");
        NSURL *url = [NSURL URLWithString:@"https://example.com"];
        NSData *data = synchronousFetch(url);
        NSLog(@"Got: %@", [[NSString alloc] initWithData:data encoding:NSUTF8StringEncoding]);
    }
    return 0;
}
```

---

## 32.16 dispatch_source: Timer และ File Monitoring

`dispatch_source` เป็น mechanism สำหรับ monitor kernel/system events เช่น timer, file changes, signals

### Timer ด้วย dispatch_source

```objc
#import <Foundation/Foundation.h>

@interface Timer : NSObject {
    dispatch_source_t _timerSource;
    NSInteger _count;
}

- (void)startWithInterval:(NSTimeInterval)interval;
- (void)stop;

@end

@implementation Timer

- (void)startWithInterval:(NSTimeInterval)interval {
    _count = 0;
    dispatch_queue_t queue = dispatch_get_global_queue(QOS_CLASS_DEFAULT, 0);
    
    // สร้าง timer source
    _timerSource = dispatch_source_create(DISPATCH_SOURCE_TYPE_TIMER, 0, 0, queue);
    
    // ตั้งค่า timer: เริ่มหลัง 0s, ยิงทุก interval วินาที, leeway 0.1s
    dispatch_source_set_timer(_timerSource,
                              dispatch_time(DISPATCH_TIME_NOW, 0),
                              interval * NSEC_PER_SEC,
                              0.1 * NSEC_PER_SEC);
    
    // กำหนด event handler
    dispatch_source_set_event_handler(_timerSource, ^{
        self->_count++;
        NSLog(@"Timer fired! Count: %ld, Thread: %@",
              (long)self->_count, [NSThread currentThread]);
        
        // หยุดหลัง 5 ครั้ง
        if (self->_count >= 5) {
            [self stop];
        }
    });
    
    // กำหนด cancel handler
    dispatch_source_set_cancel_handler(_timerSource, ^{
        NSLog(@"Timer cancelled after %ld fires", (long)self->_count);
    });
    
    // เริ่ม timer
    dispatch_resume(_timerSource);
    NSLog(@"Timer started with interval: %.1f seconds", interval);
}

- (void)stop {
    if (_timerSource) {
        dispatch_source_cancel(_timerSource);
        _timerSource = nil;
    }
}

@end

int main(int argc, const char * argv[]) {
    @autoreleasepool {
        Timer *timer = [[Timer alloc] init];
        [timer startWithInterval:0.5]; // ยิงทุก 0.5 วินาที
        
        [NSThread sleepForTimeInterval:4.0];
        NSLog(@"Main: done");
    }
    return 0;
}
```

### File Monitoring ด้วย dispatch_source

```objc
#import <Foundation/Foundation.h>
#import <fcntl.h>

@interface FileMonitor : NSObject {
    dispatch_source_t _source;
    int _fileDescriptor;
    NSString *_filePath;
}

- (instancetype)initWithPath:(NSString *)path;
- (void)startMonitoring;
- (void)stopMonitoring;

@end

@implementation FileMonitor

- (instancetype)initWithPath:(NSString *)path {
    self = [super init];
    if (self) {
        _filePath = [path copy];
    }
    return self;
}

- (void)startMonitoring {
    // เปิดไฟล์สำหรับ monitoring
    _fileDescriptor = open([_filePath fileSystemRepresentation], O_EVTONLY);
    
    if (_fileDescriptor < 0) {
        NSLog(@"Failed to open file: %@", _filePath);
        return;
    }
    
    dispatch_queue_t queue = dispatch_get_global_queue(QOS_CLASS_DEFAULT, 0);
    
    // สร้าง vnode source สำหรับ monitor file events
    _source = dispatch_source_create(DISPATCH_SOURCE_TYPE_VNODE,
                                      _fileDescriptor,
                                      DISPATCH_VNODE_DELETE | DISPATCH_VNODE_WRITE | DISPATCH_VNODE_EXTEND,
                                      queue);
    
    dispatch_source_set_event_handler(_source, ^{
        unsigned long flags = dispatch_source_get_data(self->_source);
        
        if (flags & DISPATCH_VNODE_DELETE) {
            NSLog(@"File deleted: %@", self->_filePath);
            [self stopMonitoring];
        }
        if (flags & DISPATCH_VNODE_WRITE) {
            NSLog(@"File modified: %@", self->_filePath);
        }
        if (flags & DISPATCH_VNODE_EXTEND) {
            NSLog(@"File extended: %@", self->_filePath);
        }
    });
    
    dispatch_source_set_cancel_handler(_source, ^{
        close(self->_fileDescriptor);
        NSLog(@"Stopped monitoring: %@", self->_filePath);
    });
    
    dispatch_resume(_source);
    NSLog(@"Monitoring file: %@", _filePath);
}

- (void)stopMonitoring {
    if (_source) {
        dispatch_source_cancel(_source);
        _source = nil;
    }
}

@end

int main(int argc, const char * argv[]) {
    @autoreleasepool {
        NSString *tempFile = @"/tmp/test_monitor.txt";
        
        // สร้างไฟล์ทดสอบ
        [@"Initial content" writeToFile:tempFile
                             atomically:YES
                               encoding:NSUTF8StringEncoding
                                  error:nil];
        
        FileMonitor *monitor = [[FileMonitor alloc] initWithPath:tempFile];
        [monitor startMonitoring];
        
        // จำลองการแก้ไขไฟล์
        dispatch_after(dispatch_time(DISPATCH_TIME_NOW, 1 * NSEC_PER_SEC),
                       dispatch_get_global_queue(QOS_CLASS_DEFAULT, 0), ^{
            [@"Modified content" writeToFile:tempFile
                                  atomically:YES
                                    encoding:NSUTF8StringEncoding
                                       error:nil];
        });
        
        [NSThread sleepForTimeInterval:2.0];
        [monitor stopMonitoring];
    }
    return 0;
}
```

---

## 32.17 Combining GCD กับ Completion Blocks

Pattern ที่ใช้บ่อยที่สุดในการพัฒนา iOS: ทำงานบน background แล้ว callback บน main thread

```objc
#import <Foundation/Foundation.h>

// ====================================================
// Network Layer ด้วย GCD และ Completion Blocks
// ====================================================

typedef void(^NetworkCompletion)(id responseObject, NSError *error);
typedef void(^ProgressBlock)(float progress);

@interface NetworkManager : NSObject

+ (instancetype)sharedManager;
- (void)GET:(NSString *)urlString
 completion:(NetworkCompletion)completion;
- (void)downloadFile:(NSString *)urlString
            progress:(ProgressBlock)progress
          completion:(void(^)(NSURL *localURL, NSError *error))completion;

@end

@implementation NetworkManager

+ (instancetype)sharedManager {
    static NetworkManager *instance = nil;
    static dispatch_once_t onceToken;
    dispatch_once(&onceToken, ^{ instance = [[self alloc] init]; });
    return instance;
}

- (void)GET:(NSString *)urlString completion:(NetworkCompletion)completion {
    dispatch_async(dispatch_get_global_queue(QOS_CLASS_USER_INITIATED, 0), ^{
        NSLog(@"[Network] GET %@ on thread: %@", urlString, [NSThread currentThread]);
        
        // จำลอง network delay
        [NSThread sleepForTimeInterval:1.0 + (arc4random_uniform(10) * 0.1)];
        
        // จำลอง response
        NSDictionary *response = @{
            @"status": @"success",
            @"url": urlString,
            @"timestamp": [NSDate date].description
        };
        
        // ส่งผลกลับบน main thread เสมอ!
        dispatch_async(dispatch_get_main_queue(), ^{
            NSLog(@"[Network] Response received on main thread");
            if (completion) {
                completion(response, nil);
            }
        });
    });
}

- (void)downloadFile:(NSString *)urlString
            progress:(ProgressBlock)progress
          completion:(void(^)(NSURL *localURL, NSError *error))completion {
    
    dispatch_async(dispatch_get_global_queue(QOS_CLASS_UTILITY, 0), ^{
        NSLog(@"[Download] Starting download: %@", urlString);
        
        // จำลองการ download พร้อม progress
        for (int i = 1; i <= 10; i++) {
            [NSThread sleepForTimeInterval:0.3];
            float prog = i / 10.0f;
            
            dispatch_async(dispatch_get_main_queue(), ^{
                if (progress) {
                    progress(prog);
                }
                NSLog(@"[Download] Progress: %.0f%%", prog * 100);
            });
        }
        
        // จำลอง saved file
        NSURL *savedURL = [NSURL fileURLWithPath:@"/tmp/downloaded_file.dat"];
        
        dispatch_async(dispatch_get_main_queue(), ^{
            if (completion) {
                completion(savedURL, nil);
            }
        });
    });
}

@end

// ====================================================
// การใช้งาน
// ====================================================

int main(int argc, const char * argv[]) {
    @autoreleasepool {
        NetworkManager *network = [NetworkManager sharedManager];
        
        // Simple GET request
        NSLog(@"Making GET request...");
        [network GET:@"https://api.example.com/users"
          completion:^(id responseObject, NSError *error) {
            if (error) {
                NSLog(@"Error: %@", error.localizedDescription);
            } else {
                NSLog(@"Response: %@", responseObject);
            }
        }];
        
        // Download with progress
        NSLog(@"\nStarting download...");
        [network downloadFile:@"https://example.com/bigfile.zip"
                     progress:^(float progress) {
                         // อัพเดท progress bar
                     }
                   completion:^(NSURL *localURL, NSError *error) {
                       NSLog(@"Downloaded to: %@", localURL);
                   }];
        
        [NSThread sleepForTimeInterval:6.0];
    }
    return 0;
}
```

---

## 32.18 UI Update Pattern: Background Thread → Main Thread

```objc
#import <Foundation/Foundation.h>

// จำลอง UIViewController
@interface DataViewController : NSObject

@property (nonatomic, strong) NSArray *tableData;
@property (nonatomic, assign) BOOL isLoading;

- (void)loadDataFromAPI;
- (void)processImageInBackground:(NSString *)imagePath;

@end

@implementation DataViewController

- (void)loadDataFromAPI {
    self.isLoading = YES;
    NSLog(@"[VC] Started loading, isLoading = YES");
    
    // จำลอง show loading indicator
    // [self.activityIndicator startAnimating];
    
    dispatch_async(dispatch_get_global_queue(QOS_CLASS_USER_INITIATED, 0), ^{
        // ====== Background Thread ======
        NSLog(@"[VC] Fetching data on thread: %@", [NSThread currentThread]);
        NSLog(@"[VC] Is main thread: %@", [NSThread isMainThread] ? @"YES" : @"NO");
        
        [NSThread sleepForTimeInterval:2.0]; // จำลอง network request
        
        NSArray *newData = @[
            @{@"id": @1, @"name": @"สมชาย", @"age": @25},
            @{@"id": @2, @"name": @"สมหญิง", @"age": @30},
            @{@"id": @3, @"name": @"สมศักดิ์", @"age": @22}
        ];
        
        // ====== กลับ Main Thread สำหรับ UI ======
        dispatch_async(dispatch_get_main_queue(), ^{
            NSLog(@"[VC] Updating UI on main thread: %@", [NSThread currentThread]);
            
            self.tableData = newData;
            self.isLoading = NO;
            
            // [self.tableView reloadData];
            // [self.activityIndicator stopAnimating];
            
            NSLog(@"[VC] UI Updated! tableData count: %lu", (unsigned long)self.tableData.count);
            NSLog(@"[VC] isLoading = %@", self.isLoading ? @"YES" : @"NO");
        });
    });
}

- (void)processImageInBackground:(NSString *)imagePath {
    NSLog(@"[VC] Starting image processing for: %@", imagePath);
    
    dispatch_async(dispatch_get_global_queue(QOS_CLASS_USER_INITIATED, 0), ^{
        // จำลองการประมวลผลภาพ (Heavy computation)
        NSLog(@"[VC] Processing image on background thread...");
        [NSThread sleepForTimeInterval:1.5];
        
        // ผลลัพธ์การประมวลผล
        NSString *processedImagePath = [imagePath stringByAppendingString:@"_processed"];
        
        // อัพเดท UI บน main thread
        dispatch_async(dispatch_get_main_queue(), ^{
            NSLog(@"[VC] Image processed: %@", processedImagePath);
            // self.imageView.image = processedImage;
            // [self showNotification:@"Image processed successfully"];
        });
    });
}

@end

int main(int argc, const char * argv[]) {
    @autoreleasepool {
        DataViewController *vc = [[DataViewController alloc] init];
        
        [vc loadDataFromAPI];
        [vc processImageInBackground:@"/photos/profile.jpg"];
        
        [NSThread sleepForTimeInterval:4.0];
    }
    return 0;
}
```

---

## 32.19 Pattern จริง: Image Processing Pipeline

```objc
#import <Foundation/Foundation.h>

typedef NS_ENUM(NSInteger, FilterType) {
    FilterTypeGrayscale,
    FilterTypeSepia,
    FilterTypeBrightness,
    FilterTypeContrast
};

@interface ImageProcessor : NSObject

- (void)applyFilters:(NSArray<NSNumber *> *)filters
         toImagePath:(NSString *)imagePath
          completion:(void(^)(NSString *outputPath, NSError *error))completion;

@end

@implementation ImageProcessor

- (void)applyFilters:(NSArray<NSNumber *> *)filters
         toImagePath:(NSString *)imagePath
          completion:(void(^)(NSString *outputPath, NSError *error))completion {
    
    // ใช้ concurrent queue สำหรับประมวลผล
    dispatch_queue_t processingQueue = dispatch_queue_create(
        "com.myapp.imageprocessing",
        dispatch_queue_attr_make_with_qos_class(DISPATCH_QUEUE_CONCURRENT, QOS_CLASS_USER_INITIATED, 0)
    );
    
    dispatch_async(processingQueue, ^{
        NSLog(@"[ImageProcessor] Starting pipeline for: %@", imagePath);
        NSLog(@"[ImageProcessor] Thread: %@", [NSThread currentThread]);
        
        NSMutableString *currentPath = [NSMutableString stringWithString:imagePath];
        
        // Apply filters sequentially
        for (NSNumber *filterNum in filters) {
            FilterType filter = (FilterType)[filterNum integerValue];
            NSString *filterName = @"unknown";
            
            switch (filter) {
                case FilterTypeGrayscale: filterName = @"grayscale"; break;
                case FilterTypeSepia: filterName = @"sepia"; break;
                case FilterTypeBrightness: filterName = @"brightness"; break;
                case FilterTypeContrast: filterName = @"contrast"; break;
            }
            
            NSLog(@"[ImageProcessor] Applying %@ filter...", filterName);
            [NSThread sleepForTimeInterval:0.3]; // จำลองการ apply filter
            
            [currentPath appendFormat:@"_%@", filterName];
        }
        
        NSString *outputPath = [currentPath stringByAppendingString:@".jpg"];
        NSLog(@"[ImageProcessor] Pipeline complete: %@", outputPath);
        
        // ส่งผลกลับบน main thread
        dispatch_async(dispatch_get_main_queue(), ^{
            completion(outputPath, nil);
        });
    });
}

@end

// ===== Batch Image Processing =====
void processBatchImages(NSArray *imagePaths, void(^allDone)(NSArray *results)) {
    dispatch_group_t group = dispatch_group_create();
    dispatch_queue_t queue = dispatch_get_global_queue(QOS_CLASS_USER_INITIATED, 0);
    dispatch_queue_t resultQueue = dispatch_queue_create("com.myapp.results", DISPATCH_QUEUE_SERIAL);
    
    NSMutableArray *results = [NSMutableArray array];
    
    for (NSString *imagePath in imagePaths) {
        dispatch_group_enter(group);
        
        dispatch_async(queue, ^{
            NSLog(@"Processing: %@", imagePath);
            [NSThread sleepForTimeInterval:0.5];
            
            NSString *result = [imagePath stringByAppendingString:@"_processed"];
            
            dispatch_async(resultQueue, ^{
                [results addObject:result];
                dispatch_group_leave(group);
            });
        });
    }
    
    dispatch_group_notify(group, dispatch_get_main_queue(), ^{
        NSLog(@"All images processed!");
        allDone([results copy]);
    });
}

int main(int argc, const char * argv[]) {
    @autoreleasepool {
        ImageProcessor *processor = [[ImageProcessor alloc] init];
        
        NSArray *filters = @[
            @(FilterTypeGrayscale),
            @(FilterTypeContrast),
            @(FilterTypeSepia)
        ];
        
        [processor applyFilters:filters
                    toImagePath:@"/photos/original.jpg"
                     completion:^(NSString *outputPath, NSError *error) {
            if (error) {
                NSLog(@"Error: %@", error);
            } else {
                NSLog(@"Final image: %@", outputPath);
            }
        }];
        
        // Batch processing
        NSArray *images = @[@"/photos/img1.jpg", @"/photos/img2.jpg", 
                            @"/photos/img3.jpg", @"/photos/img4.jpg"];
        
        processBatchImages(images, ^(NSArray *results) {
            NSLog(@"Batch complete! %lu images processed", (unsigned long)results.count);
            for (NSString *path in results) {
                NSLog(@"  - %@", path);
            }
        });
        
        [NSThread sleepForTimeInterval:4.0];
    }
    return 0;
}
```

---

## 32.20 Pattern: Network Request ที่สมจริง

```objc
#import <Foundation/Foundation.h>

// Model
@interface User : NSObject
@property (nonatomic, strong) NSNumber *userId;
@property (nonatomic, strong) NSString *name;
@property (nonatomic, strong) NSString *email;
@property (nonatomic, strong) NSArray *posts;
+ (instancetype)userFromDictionary:(NSDictionary *)dict;
@end

@implementation User
+ (instancetype)userFromDictionary:(NSDictionary *)dict {
    User *user = [[User alloc] init];
    user.userId = dict[@"id"];
    user.name = dict[@"name"];
    user.email = dict[@"email"];
    return user;
}
@end

// Service
@interface UserService : NSObject

- (void)fetchUser:(NSInteger)userId
       completion:(void(^)(User *user, NSError *error))completion;

- (void)fetchPostsForUser:(NSInteger)userId
              completion:(void(^)(NSArray *posts, NSError *error))completion;

- (void)fetchUserWithPosts:(NSInteger)userId
               completion:(void(^)(User *user, NSError *error))completion;

@end

@implementation UserService

- (void)fetchUser:(NSInteger)userId
       completion:(void(^)(User *user, NSError *error))completion {
    
    dispatch_async(dispatch_get_global_queue(QOS_CLASS_USER_INITIATED, 0), ^{
        [NSThread sleepForTimeInterval:0.8];
        
        NSDictionary *userData = @{
            @"id": @(userId),
            @"name": [NSString stringWithFormat:@"User%ld", (long)userId],
            @"email": [NSString stringWithFormat:@"user%ld@example.com", (long)userId]
        };
        
        User *user = [User userFromDictionary:userData];
        
        dispatch_async(dispatch_get_main_queue(), ^{
            completion(user, nil);
        });
    });
}

- (void)fetchPostsForUser:(NSInteger)userId
              completion:(void(^)(NSArray *posts, NSError *error))completion {
    
    dispatch_async(dispatch_get_global_queue(QOS_CLASS_USER_INITIATED, 0), ^{
        [NSThread sleepForTimeInterval:1.2];
        
        NSMutableArray *posts = [NSMutableArray array];
        for (int i = 1; i <= 5; i++) {
            [posts addObject:@{
                @"id": @(i),
                @"title": [NSString stringWithFormat:@"Post %d by User%ld", i, (long)userId],
                @"content": @"Lorem ipsum dolor sit amet..."
            }];
        }
        
        dispatch_async(dispatch_get_main_queue(), ^{
            completion([posts copy], nil);
        });
    });
}

// Fetch User และ Posts พร้อมกัน แล้วรวมผล
- (void)fetchUserWithPosts:(NSInteger)userId
               completion:(void(^)(User *user, NSError *error))completion {
    
    dispatch_group_t group = dispatch_group_create();
    __block User *fetchedUser = nil;
    __block NSArray *fetchedPosts = nil;
    __block NSError *firstError = nil;
    
    // Fetch User
    dispatch_group_enter(group);
    [self fetchUser:userId completion:^(User *user, NSError *error) {
        if (error) {
            firstError = error;
        } else {
            fetchedUser = user;
        }
        dispatch_group_leave(group);
    }];
    
    // Fetch Posts (parallel)
    dispatch_group_enter(group);
    [self fetchPostsForUser:userId completion:^(NSArray *posts, NSError *error) {
        if (error) {
            firstError = error;
        } else {
            fetchedPosts = posts;
        }
        dispatch_group_leave(group);
    }];
    
    // เมื่อทั้งสองเสร็จ
    dispatch_group_notify(group, dispatch_get_main_queue(), ^{
        if (firstError) {
            completion(nil, firstError);
            return;
        }
        
        fetchedUser.posts = fetchedPosts;
        NSLog(@"User %@ has %lu posts", fetchedUser.name, (unsigned long)fetchedUser.posts.count);
        completion(fetchedUser, nil);
    });
}

@end

int main(int argc, const char * argv[]) {
    @autoreleasepool {
        UserService *service = [[UserService alloc] init];
        
        NSLog(@"Fetching user with posts...");
        NSDate *start = [NSDate date];
        
        [service fetchUserWithPosts:42 completion:^(User *user, NSError *error) {
            if (error) {
                NSLog(@"Error: %@", error);
            } else {
                NSLog(@"Got user: %@ (%@)", user.name, user.email);
                NSLog(@"Posts count: %lu", (unsigned long)user.posts.count);
                NSLog(@"Fetched in: %.2f seconds", [[NSDate date] timeIntervalSinceDate:start]);
                // เนื่องจาก parallel: ใช้เวลาประมาณ max(0.8, 1.2) = 1.2s ไม่ใช่ 0.8+1.2=2.0s
            }
        }];
        
        [NSThread sleepForTimeInterval:3.0];
    }
    return 0;
}
```

---

## 32.21 Deadlock Scenarios และวิธีหลีกเลี่ยง

### Scenario 1: Main Thread Sync to Main Thread

```objc
// DEADLOCK!
void scenario1_deadlock(void) {
    // ถ้าเรียกบน main thread:
    // dispatch_sync(dispatch_get_main_queue(), ^{
    //     NSLog(@"Never executes - DEADLOCK!");
    // });
    
    // FIX: ใช้ async หรือตรวจสอบก่อน
    if (![NSThread isMainThread]) {
        dispatch_sync(dispatch_get_main_queue(), ^{
            NSLog(@"Safe: already checked we're not on main thread");
        });
    } else {
        NSLog(@"Already on main thread, no dispatch needed");
    }
}
```

### Scenario 2: Nested Sync on Same Serial Queue

```objc
// DEADLOCK!
void scenario2_deadlock(void) {
    dispatch_queue_t serial = dispatch_queue_create("com.test", DISPATCH_QUEUE_SERIAL);
    
    dispatch_async(serial, ^{
        NSLog(@"Outer block");
        
        // DEADLOCK! serial queue กำลังรันอยู่ แต่ sync รอให้ serial ว่าง
        // dispatch_sync(serial, ^{
        //     NSLog(@"Never executes - DEADLOCK!");
        // });
        
        // FIX: ใช้ async หรือ queue อื่น
        dispatch_async(serial, ^{
            NSLog(@"Inner block - safe with async");
        });
    });
}
```

### Scenario 3: Classic Lock Ordering Deadlock

```objc
#import <Foundation/Foundation.h>

// DEADLOCK: Thread A lock1 รอ lock2, Thread B lock2 รอ lock1
void scenario3_deadlock_demo(void) {
    dispatch_semaphore_t lock1 = dispatch_semaphore_create(1);
    dispatch_semaphore_t lock2 = dispatch_semaphore_create(1);
    
    dispatch_queue_t q = dispatch_get_global_queue(QOS_CLASS_DEFAULT, 0);
    
    // Thread A: lock1 แล้ว lock2
    dispatch_async(q, ^{
        dispatch_semaphore_wait(lock1, DISPATCH_TIME_FOREVER);
        NSLog(@"Thread A: got lock1");
        [NSThread sleepForTimeInterval:0.1]; // ให้ Thread B ได้ lock2 ก่อน
        
        // DEADLOCK! Thread B ถือ lock2 อยู่
        // dispatch_semaphore_wait(lock2, DISPATCH_TIME_FOREVER);
        
        // FIX: ใช้ lock ordering ที่ consistent เสมอ
        // หรือใช้ trylock กับ timeout
        long result = dispatch_semaphore_wait(lock2, dispatch_time(DISPATCH_TIME_NOW, 2 * NSEC_PER_SEC));
        if (result == 0) {
            NSLog(@"Thread A: got lock2 too");
            dispatch_semaphore_signal(lock2);
        } else {
            NSLog(@"Thread A: timeout waiting for lock2");
        }
        dispatch_semaphore_signal(lock1);
    });
    
    // Thread B: lock2 แล้ว lock1 (순序ตรงข้าม!)
    dispatch_async(q, ^{
        dispatch_semaphore_wait(lock2, DISPATCH_TIME_FOREVER);
        NSLog(@"Thread B: got lock2");
        [NSThread sleepForTimeInterval:0.1];
        
        // FIX: เปลี่ยนเป็น lock1 ก่อน lock2 ทั้งสองที่ (consistent ordering)
        long result = dispatch_semaphore_wait(lock1, dispatch_time(DISPATCH_TIME_NOW, 2 * NSEC_PER_SEC));
        if (result == 0) {
            NSLog(@"Thread B: got lock1 too");
            dispatch_semaphore_signal(lock1);
        } else {
            NSLog(@"Thread B: timeout waiting for lock1");
        }
        dispatch_semaphore_signal(lock2);
    });
}

int main(int argc, const char * argv[]) {
    @autoreleasepool {
        scenario1_deadlock();
        scenario2_deadlock();
        scenario3_deadlock_demo();
        [NSThread sleepForTimeInterval:2.0];
    }
    return 0;
}
```

---

## 32.22 Performance Comparison: Serial vs Concurrent

```objc
#import <Foundation/Foundation.h>

void measurePerformance(NSString *label, void(^block)(void)) {
    NSDate *start = [NSDate date];
    block();
    NSTimeInterval elapsed = [[NSDate date] timeIntervalSinceDate:start];
    NSLog(@"[%@] Time: %.3f seconds", label, elapsed);
}

int main(int argc, const char * argv[]) {
    @autoreleasepool {
        const int TASK_COUNT = 8;
        const NSTimeInterval TASK_DURATION = 0.5;
        
        // ====== Serial Execution ======
        measurePerformance(@"Serial", ^{
            dispatch_queue_t serial = dispatch_queue_create("com.test.serial", DISPATCH_QUEUE_SERIAL);
            dispatch_group_t group = dispatch_group_create();
            
            for (int i = 0; i < TASK_COUNT; i++) {
                dispatch_group_async(group, serial, ^{
                    [NSThread sleepForTimeInterval:TASK_DURATION];
                });
            }
            dispatch_group_wait(group, DISPATCH_TIME_FOREVER);
        });
        // คาดว่า: 8 * 0.5 = 4.0 วินาที
        
        // ====== Concurrent Execution ======
        measurePerformance(@"Concurrent", ^{
            dispatch_queue_t concurrent = dispatch_queue_create("com.test.concurrent", DISPATCH_QUEUE_CONCURRENT);
            dispatch_group_t group = dispatch_group_create();
            
            for (int i = 0; i < TASK_COUNT; i++) {
                dispatch_group_async(group, concurrent, ^{
                    [NSThread sleepForTimeInterval:TASK_DURATION];
                });
            }
            dispatch_group_wait(group, DISPATCH_TIME_FOREVER);
        });
        // คาดว่า: ~0.5 วินาที (ทำพร้อมกันทั้งหมด, ขึ้นกับจำนวน CPU cores)
        
        // ====== Global Queue ======
        measurePerformance(@"Global Queue", ^{
            dispatch_queue_t global = dispatch_get_global_queue(QOS_CLASS_USER_INITIATED, 0);
            dispatch_group_t group = dispatch_group_create();
            
            for (int i = 0; i < TASK_COUNT; i++) {
                dispatch_group_async(group, global, ^{
                    [NSThread sleepForTimeInterval:TASK_DURATION];
                });
            }
            dispatch_group_wait(group, DISPATCH_TIME_FOREVER);
        });
        
        NSLog(@"\nSystem Info:");
        NSLog(@"Active processors: %ld", (long)[[NSProcessInfo processInfo] activeProcessorCount]);
        NSLog(@"Physical memory: %.2f GB", 
              [[NSProcessInfo processInfo] physicalMemory] / 1e9);
    }
    return 0;
}
```

---

## 32.23 Advanced Pattern: Operation Pipeline ด้วย GCD

```objc
#import <Foundation/Foundation.h>

// Pipeline: Fetch -> Parse -> Transform -> Cache -> Display
@interface DataPipeline : NSObject

@property (nonatomic, strong) dispatch_queue_t fetchQueue;
@property (nonatomic, strong) dispatch_queue_t parseQueue;
@property (nonatomic, strong) dispatch_queue_t cacheQueue;

- (instancetype)init;
- (void)processURL:(NSString *)url
        completion:(void(^)(NSDictionary *result, NSError *error))completion;

@end

@implementation DataPipeline

- (instancetype)init {
    self = [super init];
    if (self) {
        _fetchQueue = dispatch_queue_create("com.pipeline.fetch",
            dispatch_queue_attr_make_with_qos_class(DISPATCH_QUEUE_CONCURRENT, QOS_CLASS_USER_INITIATED, 0));
        _parseQueue = dispatch_queue_create("com.pipeline.parse",
            dispatch_queue_attr_make_with_qos_class(DISPATCH_QUEUE_CONCURRENT, QOS_CLASS_USER_INITIATED, 0));
        _cacheQueue = dispatch_queue_create("com.pipeline.cache",
            dispatch_queue_attr_make_with_qos_class(DISPATCH_QUEUE_SERIAL, QOS_CLASS_UTILITY, 0));
    }
    return self;
}

- (void)processURL:(NSString *)url
        completion:(void(^)(NSDictionary *result, NSError *error))completion {
    
    // Step 1: Fetch
    dispatch_async(_fetchQueue, ^{
        NSLog(@"[Pipeline] Fetching: %@", url);
        [NSThread sleepForTimeInterval:0.8];
        NSString *rawData = [NSString stringWithFormat:@"{\"url\":\"%@\",\"data\":\"raw content\"}", url];
        
        // Step 2: Parse
        dispatch_async(self->_parseQueue, ^{
            NSLog(@"[Pipeline] Parsing data...");
            [NSThread sleepForTimeInterval:0.3];
            
            // จำลอง JSON parsing
            NSDictionary *parsed = @{
                @"url": url,
                @"content": rawData,
                @"parsed": @YES,
                @"timestamp": [NSDate date]
            };
            
            // Step 3: Cache
            dispatch_async(self->_cacheQueue, ^{
                NSLog(@"[Pipeline] Caching result...");
                [NSThread sleepForTimeInterval:0.1];
                // บันทึก cache
                
                // Step 4: Notify completion on main thread
                dispatch_async(dispatch_get_main_queue(), ^{
                    NSLog(@"[Pipeline] Complete!");
                    completion(parsed, nil);
                });
            });
        });
    });
}

@end

int main(int argc, const char * argv[]) {
    @autoreleasepool {
        DataPipeline *pipeline = [[DataPipeline alloc] init];
        
        NSArray *urls = @[
            @"https://api.example.com/data1",
            @"https://api.example.com/data2",
            @"https://api.example.com/data3"
        ];
        
        dispatch_group_t allDone = dispatch_group_create();
        
        for (NSString *url in urls) {
            dispatch_group_enter(allDone);
            
            [pipeline processURL:url completion:^(NSDictionary *result, NSError *error) {
                NSLog(@"Result for %@: parsed=%@", result[@"url"], result[@"parsed"]);
                dispatch_group_leave(allDone);
            }];
        }
        
        dispatch_group_notify(allDone, dispatch_get_main_queue(), ^{
            NSLog(@"\nAll pipelines complete!");
        });
        
        dispatch_group_wait(allDone, DISPATCH_TIME_FOREVER);
        [NSThread sleepForTimeInterval:0.5];
    }
    return 0;
}
```

---

## 32.24 ตัวอย่างที่ครบครัน: Task Manager ด้วย GCD

```objc
#import <Foundation/Foundation.h>

typedef NS_ENUM(NSInteger, TaskPriority) {
    TaskPriorityHigh,
    TaskPriorityNormal,
    TaskPriorityLow
};

typedef NS_ENUM(NSInteger, TaskState) {
    TaskStatePending,
    TaskStateRunning,
    TaskStateCompleted,
    TaskStateFailed,
    TaskStateCancelled
};

@interface Task : NSObject

@property (nonatomic, strong) NSString *taskID;
@property (nonatomic, strong) NSString *name;
@property (nonatomic, assign) TaskPriority priority;
@property (nonatomic, assign) TaskState state;
@property (nonatomic, strong) void (^work)(void(^progress)(float), void(^done)(NSError *));

+ (instancetype)taskWithName:(NSString *)name
                    priority:(TaskPriority)priority
                        work:(void(^)(void(^progress)(float), void(^done)(NSError *)))work;

@end

@implementation Task

+ (instancetype)taskWithName:(NSString *)name
                    priority:(TaskPriority)priority
                        work:(void(^)(void(^progress)(float), void(^done)(NSError *)))work {
    Task *task = [[Task alloc] init];
    task.taskID = [[NSUUID UUID] UUIDString];
    task.name = name;
    task.priority = priority;
    task.state = TaskStatePending;
    task.work = work;
    return task;
}

@end

@interface TaskManager : NSObject {
    dispatch_queue_t _highPriorityQueue;
    dispatch_queue_t _normalPriorityQueue;
    dispatch_queue_t _lowPriorityQueue;
    dispatch_queue_t _taskListQueue;
    NSMutableDictionary<NSString *, Task *> *_tasks;
    dispatch_semaphore_t _concurrencyLimit;
}

- (instancetype)initWithMaxConcurrency:(NSInteger)maxConcurrency;
- (void)submitTask:(Task *)task;
- (void)cancelTask:(NSString *)taskID;
- (NSDictionary *)taskSummary;

@end

@implementation TaskManager

- (instancetype)initWithMaxConcurrency:(NSInteger)maxConcurrency {
    self = [super init];
    if (self) {
        _highPriorityQueue = dispatch_queue_create("com.tasks.high",
            dispatch_queue_attr_make_with_qos_class(DISPATCH_QUEUE_CONCURRENT, QOS_CLASS_USER_INITIATED, 0));
        _normalPriorityQueue = dispatch_queue_create("com.tasks.normal",
            dispatch_queue_attr_make_with_qos_class(DISPATCH_QUEUE_CONCURRENT, QOS_CLASS_DEFAULT, 0));
        _lowPriorityQueue = dispatch_queue_create("com.tasks.low",
            dispatch_queue_attr_make_with_qos_class(DISPATCH_QUEUE_CONCURRENT, QOS_CLASS_UTILITY, 0));
        _taskListQueue = dispatch_queue_create("com.tasks.list", DISPATCH_QUEUE_SERIAL);
        _tasks = [NSMutableDictionary dictionary];
        _concurrencyLimit = dispatch_semaphore_create(maxConcurrency);
    }
    return self;
}

- (void)submitTask:(Task *)task {
    dispatch_async(_taskListQueue, ^{
        self->_tasks[task.taskID] = task;
    });
    
    dispatch_queue_t queue;
    switch (task.priority) {
        case TaskPriorityHigh: queue = _highPriorityQueue; break;
        case TaskPriorityNormal: queue = _normalPriorityQueue; break;
        case TaskPriorityLow: queue = _lowPriorityQueue; break;
    }
    
    dispatch_async(queue, ^{
        // รอ slot ว่าง
        dispatch_semaphore_wait(self->_concurrencyLimit, DISPATCH_TIME_FOREVER);
        
        dispatch_async(self->_taskListQueue, ^{
            task.state = TaskStateRunning;
        });
        
        NSLog(@"[TaskMgr] Starting task: %@", task.name);
        
        task.work(
            ^(float progress) {
                // Progress callback
                dispatch_async(dispatch_get_main_queue(), ^{
                    NSLog(@"[TaskMgr] %@: %.0f%%", task.name, progress * 100);
                });
            },
            ^(NSError *error) {
                // Done callback
                dispatch_async(self->_taskListQueue, ^{
                    task.state = error ? TaskStateFailed : TaskStateCompleted;
                });
                
                dispatch_async(dispatch_get_main_queue(), ^{
                    if (error) {
                        NSLog(@"[TaskMgr] FAILED: %@ - %@", task.name, error.localizedDescription);
                    } else {
                        NSLog(@"[TaskMgr] COMPLETED: %@", task.name);
                    }
                });
                
                dispatch_semaphore_signal(self->_concurrencyLimit);
            }
        );
    });
}

- (void)cancelTask:(NSString *)taskID {
    dispatch_async(_taskListQueue, ^{
        Task *task = self->_tasks[taskID];
        if (task && task.state == TaskStatePending) {
            task.state = TaskStateCancelled;
            NSLog(@"[TaskMgr] Cancelled: %@", task.name);
        }
    });
}

- (NSDictionary *)taskSummary {
    __block NSMutableDictionary *summary = [NSMutableDictionary dictionary];
    dispatch_sync(_taskListQueue, ^{
        NSInteger pending = 0, running = 0, completed = 0, failed = 0, cancelled = 0;
        for (Task *task in self->_tasks.allValues) {
            switch (task.state) {
                case TaskStatePending: pending++; break;
                case TaskStateRunning: running++; break;
                case TaskStateCompleted: completed++; break;
                case TaskStateFailed: failed++; break;
                case TaskStateCancelled: cancelled++; break;
            }
        }
        summary = [@{
            @"pending": @(pending),
            @"running": @(running),
            @"completed": @(completed),
            @"failed": @(failed),
            @"cancelled": @(cancelled),
            @"total": @(self->_tasks.count)
        } mutableCopy];
    });
    return [summary copy];
}

@end

int main(int argc, const char * argv[]) {
    @autoreleasepool {
        // สร้าง task manager รับงานพร้อมกันสูงสุด 3 งาน
        TaskManager *manager = [[TaskManager alloc] initWithMaxConcurrency:3];
        
        // สร้าง tasks
        for (int i = 1; i <= 6; i++) {
            TaskPriority priority = (i % 3 == 0) ? TaskPriorityHigh :
                                    (i % 3 == 1) ? TaskPriorityNormal : TaskPriorityLow;
            
            Task *task = [Task taskWithName:[NSString stringWithFormat:@"Task-%d", i]
                                   priority:priority
                                       work:^(void (^progress)(float), void (^done)(NSError *)) {
                NSUInteger duration = (arc4random_uniform(3) + 1);
                for (NSUInteger j = 1; j <= duration; j++) {
                    [NSThread sleepForTimeInterval:0.5];
                    progress((float)j / duration);
                }
                done(nil);
            }];
            
            [manager submitTask:task];
        }
        
        [NSThread sleepForTimeInterval:1.0];
        NSDictionary *summary = [manager taskSummary];
        NSLog(@"\n=== Task Summary (mid-run) ===");
        NSLog(@"%@", summary);
        
        [NSThread sleepForTimeInterval:4.0];
        summary = [manager taskSummary];
        NSLog(@"\n=== Final Task Summary ===");
        NSLog(@"%@", summary);
    }
    return 0;
}
```

---

## สรุป GCD API Reference

```objc
// ======= Queue Creation =======
dispatch_queue_t q1 = dispatch_get_main_queue();
dispatch_queue_t q2 = dispatch_get_global_queue(QOS_CLASS_DEFAULT, 0);
dispatch_queue_t q3 = dispatch_queue_create("label", DISPATCH_QUEUE_SERIAL);
dispatch_queue_t q4 = dispatch_queue_create("label", DISPATCH_QUEUE_CONCURRENT);

// ======= Dispatching =======
dispatch_async(queue, block);          // ส่งงานแบบ async
dispatch_sync(queue, block);           // ส่งงานแบบ sync (ระวัง deadlock)
dispatch_after(when, queue, block);    // ส่งงานแบบ delayed
dispatch_once(&token, block);          // execute เพียงครั้งเดียว

// ======= Groups =======
dispatch_group_t group = dispatch_group_create();
dispatch_group_async(group, queue, block);
dispatch_group_enter(group);
dispatch_group_leave(group);
dispatch_group_notify(group, queue, block);
dispatch_group_wait(group, timeout);

// ======= Barriers =======
dispatch_barrier_async(concurrentQueue, block);
dispatch_barrier_sync(concurrentQueue, block);

// ======= Semaphores =======
dispatch_semaphore_t sem = dispatch_semaphore_create(value);
dispatch_semaphore_wait(sem, timeout);
dispatch_semaphore_signal(sem);

// ======= Sources =======
dispatch_source_t src = dispatch_source_create(type, handle, mask, queue);
dispatch_source_set_event_handler(src, block);
dispatch_source_set_cancel_handler(src, block);
dispatch_resume(src);
dispatch_source_cancel(src);
```

---

## แบบฝึกหัด

### แบบฝึกหัดที่ 1: Basic Queue
สร้าง serial queue ชื่อ `"com.exercise.counter"` แล้วใช้มันทำ counter ที่ thread-safe โดยรับค่า increment จาก 10 threads พร้อมกัน และแสดงผลสุดท้าย

```objc
// TODO: Implement thread-safe counter using serial queue
@interface Exercise1Counter : NSObject
- (void)incrementFromMultipleThreads:(NSInteger)threadCount;
@end
```

### แบบฝึกหัดที่ 2: Concurrent Download Simulator
จำลองการ download 5 files พร้อมกัน โดยแต่ละ file ใช้เวลา random ระหว่าง 1-3 วินาที เมื่อทุกไฟล์ download เสร็จให้แสดงเวลาที่ใช้ทั้งหมด

```objc
// TODO: Use dispatch_group to download files concurrently
void downloadAllFiles(NSArray *fileURLs, void(^completion)(NSTimeInterval elapsed));
```

### แบบฝึกหัดที่ 3: Thread-Safe Cache
สร้าง cache class ที่ thread-safe โดยใช้ `dispatch_barrier_async` สำหรับ write และ `dispatch_sync` สำหรับ read

```objc
@interface ThreadSafeCache : NSObject
- (void)setValue:(id)value forKey:(NSString *)key;
- (id)valueForKey:(NSString *)key;
- (void)removeValueForKey:(NSString *)key;
- (void)clearAll;
@end
```

### แบบฝึกหัดที่ 4: Delayed Retry
เขียน function ที่ลอง fetch URL ซ้ำไม่เกิน 3 ครั้ง โดยรอ 2 วินาทีระหว่างแต่ละครั้ง ถ้า success ก็ return result ถ้าหมดครั้งแล้วยัง fail ให้ return error

```objc
void fetchWithRetry(NSString *url,
                    NSInteger maxRetries,
                    NSTimeInterval retryDelay,
                    void(^completion)(id result, NSError *error));
```

### แบบฝึกหัดที่ 5: Rate Limiter
สร้าง rate limiter ที่จำกัดการเรียก API ไม่เกิน 5 ครั้งต่อวินาที โดยใช้ dispatch_semaphore

```objc
@interface RateLimiter : NSObject
- (instancetype)initWithRequestsPerSecond:(NSInteger)rps;
- (void)executeWhenReady:(dispatch_block_t)block;
@end
```

### แบบฝึกหัดที่ 6: Periodic Timer
สร้าง timer ที่ยิงทุก 1 วินาที แสดง elapsed time และหยุดเองหลัง 10 วินาที โดยใช้ `dispatch_source`

```objc
@interface PeriodicTimer : NSObject
- (void)startWithDuration:(NSTimeInterval)totalDuration;
- (void)stop;
@property (nonatomic, copy) void (^onTick)(NSTimeInterval elapsed);
@property (nonatomic, copy) void (^onComplete)(void);
@end
```

### แบบฝึกหัดที่ 7: Async to Sync Wrapper
เขียน function `synchronousOperation` ที่แปลง async function ใดๆ ให้กลายเป็น synchronous โดยใช้ dispatch_semaphore (ห้ามเรียกจาก main thread)

```objc
id synchronousOperation(void(^asyncOperation)(void(^callback)(id result)));
```

### แบบฝึกหัดที่ 8: Priority Queue Simulator
สร้าง priority queue ที่จัดเรียง tasks ตาม priority โดย High priority ทำก่อน เมื่อ submit task ที่ High priority ขณะที่กำลังประมวล Normal/Low priority อยู่ ระบบควรจัดการ High priority ก่อน

```objc
@interface PriorityTaskQueue : NSObject
- (void)addTask:(void(^)(void))task withPriority:(NSInteger)priority;
@end
```

### แบบฝึกหัดที่ 9: Producer-Consumer
สร้าง producer-consumer pattern โดย:
- Producer: สร้าง items ใหม่ทุก 0.3 วินาที จำนวนสูงสุด 20 items
- Consumer: มี 3 consumers ประมวลผล items พร้อมกัน
- Buffer size: ไม่เกิน 5 items ในคิวพร้อมกัน

```objc
@interface ProducerConsumer : NSObject
- (void)startWithBufferSize:(NSInteger)bufferSize
              numConsumers:(NSInteger)numConsumers;
@end
```

### แบบฝึกหัดที่ 10: Concurrent Image Pipeline
สร้าง image processing pipeline ที่:
1. รับ array ของ image paths
2. Resize ทุกภาพพร้อมกัน (concurrent)
3. Apply filter ทุกภาพพร้อมกัน (concurrent)  
4. Save ทีละภาพ (serial - เพื่อป้องกัน I/O conflict)
5. แจ้งเมื่อทุกภาพเสร็จ พร้อมสถิติเวลา

```objc
@interface ImagePipeline : NSObject
- (void)processImages:(NSArray<NSString *> *)imagePaths
           completion:(void(^)(NSArray<NSString *> *outputPaths,
                               NSDictionary *stats))completion;
@end
```

---

## เฉลยตัวอย่าง: แบบฝึกหัดที่ 3 (Thread-Safe Cache)

```objc
#import <Foundation/Foundation.h>

@interface ThreadSafeCache : NSObject {
    NSMutableDictionary *_cache;
    dispatch_queue_t _queue;
}

- (void)setValue:(id)value forKey:(NSString *)key;
- (id)valueForKey:(NSString *)key;
- (void)removeValueForKey:(NSString *)key;
- (void)clearAll;
- (NSUInteger)count;

@end

@implementation ThreadSafeCache

- (instancetype)init {
    self = [super init];
    if (self) {
        _cache = [NSMutableDictionary dictionary];
        _queue = dispatch_queue_create("com.exercise.cache", DISPATCH_QUEUE_CONCURRENT);
    }
    return self;
}

- (void)setValue:(id)value forKey:(NSString *)key {
    // Write: ต้องการ exclusive access
    dispatch_barrier_async(_queue, ^{
        if (value) {
            self->_cache[key] = value;
        } else {
            [self->_cache removeObjectForKey:key];
        }
    });
}

- (id)valueForKey:(NSString *)key {
    // Read: หลายคนอ่านพร้อมกันได้
    __block id result;
    dispatch_sync(_queue, ^{
        result = self->_cache[key];
    });
    return result;
}

- (void)removeValueForKey:(NSString *)key {
    dispatch_barrier_async(_queue, ^{
        [self->_cache removeObjectForKey:key];
    });
}

- (void)clearAll {
    dispatch_barrier_async(_queue, ^{
        [self->_cache removeAllObjects];
    });
}

- (NSUInteger)count {
    __block NSUInteger count;
    dispatch_sync(_queue, ^{
        count = self->_cache.count;
    });
    return count;
}

@end

int main(int argc, const char * argv[]) {
    @autoreleasepool {
        ThreadSafeCache *cache = [[ThreadSafeCache alloc] init];
        dispatch_queue_t concurrent = dispatch_get_global_queue(QOS_CLASS_DEFAULT, 0);
        dispatch_group_t group = dispatch_group_create();
        
        // หลาย writers
        for (int i = 0; i < 10; i++) {
            dispatch_group_async(group, concurrent, ^{
                NSString *key = [NSString stringWithFormat:@"user_%d", i];
                NSString *value = [NSString stringWithFormat:@"data_%d", i * i];
                [cache setValue:value forKey:key];
            });
        }
        
        // หลาย readers (อาจเกิดพร้อมกับ writers)
        for (int i = 0; i < 20; i++) {
            dispatch_group_async(group, concurrent, ^{
                NSString *key = [NSString stringWithFormat:@"user_%d", i % 10];
                id value = [cache valueForKey:key];
                // value อาจเป็น nil ถ้า writer ยังไม่เสร็จ
                (void)value; // suppress unused warning
            });
        }
        
        dispatch_group_wait(group, DISPATCH_TIME_FOREVER);
        
        NSLog(@"Cache count: %lu", (unsigned long)[cache count]);
        NSLog(@"user_5: %@", [cache valueForKey:@"user_5"]);
        
        [cache clearAll];
        NSLog(@"After clear, count: %lu", (unsigned long)[cache count]);
    }
    return 0;
}
```

---

## สรุปบทเรียน

ใน Part 32 เราได้เรียนรู้ **Grand Central Dispatch (GCD)** ซึ่งเป็นระบบจัดการ concurrency ที่ทรงพลังของ Apple:

### ประเด็นสำคัญที่ต้องจำ

1. **Queue Types**: Main (serial, UI), Global (concurrent, QoS), Custom (serial/concurrent)
2. **async vs sync**: async ไม่รอ, sync รอ — ระวัง deadlock กับ sync!
3. **dispatch_once**: Singleton และ lazy initialization ที่ thread-safe
4. **dispatch_group**: รอหลาย tasks พร้อมกัน (notify = async, wait = sync)
5. **dispatch_barrier**: Reader-Writer pattern บน concurrent queue
6. **dispatch_semaphore**: จำกัด concurrency หรือเป็น mutex lock
7. **dispatch_source**: Timer, file monitoring ด้วย kernel events
8. **UI Rules**: อัพเดท UI บน main thread เสมอ!

### Deadlock Prevention Rules
- อย่า dispatch_sync ไปยัง queue เดิมจาก queue นั้น
- อย่า dispatch_sync ไปยัง main queue จาก main thread
- ใช้ lock ordering ที่ consistent เสมอ
- พิจารณาใช้ timeout กับ semaphore wait

### เมื่อไรใช้อะไร
| สถานการณ์ | เครื่องมือ |
|-----------|-----------|
| UI update จาก background | `dispatch_async(main_queue, ...)` |
| ป้องกัน race condition | Serial queue หรือ barrier |
| รอหลาย async tasks | dispatch_group |
| จำกัด concurrent tasks | dispatch_semaphore |
| Singleton / one-time init | dispatch_once |
| Timer | dispatch_source (TIMER) |
| หน่วงการทำงาน | dispatch_after |

---

**ต่อไป**: Part 33 - NSOperation และ NSOperationQueue สำหรับ concurrency ที่ซับซ้อนยิ่งขึ้น
