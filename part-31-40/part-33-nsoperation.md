# ตอนที่ 33: NSOperation และ NSOperationQueue

## บทนำ

ในการพัฒนาแอปพลิเคชัน iOS และ macOS การจัดการงานแบบ concurrent (ทำงานพร้อมกัน) เป็นสิ่งสำคัญมาก เพราะช่วยให้แอปทำงานได้อย่างราบรื่นโดยไม่ค้าง `NSOperation` และ `NSOperationQueue` เป็น framework ระดับสูงที่ Apple จัดเตรียมไว้สำหรับการจัดการงานแบบ concurrent โดยสร้างมาบน GCD (Grand Central Dispatch) อีกชั้นหนึ่ง

## ทำไมต้องใช้ NSOperation?

เมื่อเปรียบเทียบกับ GCD แล้ว NSOperation มีข้อได้เปรียบหลายประการ:

1. **Dependencies** - กำหนดความสัมพันธ์ระหว่าง operations ได้ง่าย
2. **Cancellation** - ยกเลิก operation ที่กำลังทำงานหรือรอทำงานได้
3. **State Tracking** - ตรวจสอบสถานะของ operation ได้ (isReady, isExecuting, isFinished, isCancelled)
4. **Priority** - กำหนดลำดับความสำคัญของ operation ได้
5. **Maximum Concurrent Operations** - ควบคุมจำนวน operation ที่ทำงานพร้อมกันได้
6. **KVO Compliance** - รองรับ Key-Value Observing เพื่อติดตามการเปลี่ยนแปลงสถานะ

---

## 33.1 NSOperation พื้นฐาน

`NSOperation` เป็น abstract class ที่แทน unit of work หนึ่งชิ้น คุณไม่สามารถใช้ NSOperation โดยตรงได้ แต่ต้องใช้ผ่าน subclass เช่น `NSBlockOperation` หรือสร้าง subclass เอง

### สถานะของ NSOperation

```objc
// NSOperation มีสถานะต่างๆ ดังนี้:
// isReady      - พร้อมทำงาน (dependencies ครบแล้ว)
// isExecuting  - กำลังทำงาน
// isFinished   - ทำงานเสร็จแล้ว
// isCancelled  - ถูกยกเลิก

NSOperation *op = [[NSOperation alloc] init];
NSLog(@"isReady: %d", op.isReady);
NSLog(@"isExecuting: %d", op.isExecuting);
NSLog(@"isFinished: %d", op.isFinished);
NSLog(@"isCancelled: %d", op.isCancelled);
```

### Properties สำคัญของ NSOperation

```objc
// completionBlock - block ที่จะถูกเรียกเมื่อ operation เสร็จสิ้น
op.completionBlock = ^{
    NSLog(@"Operation เสร็จแล้ว!");
};

// qualityOfService - กำหนด QoS class
op.qualityOfService = NSQualityOfServiceUserInitiated;

// name - ชื่อของ operation (ช่วยในการ debug)
op.name = @"MyDataProcessingOperation";

// queuePriority - ลำดับความสำคัญใน queue
op.queuePriority = NSOperationQueuePriorityNormal;
```

---

## 33.2 NSBlockOperation

`NSBlockOperation` เป็น concrete subclass ของ NSOperation ที่รองรับการใช้งาน blocks ทำให้ใช้งานได้ง่ายและสะดวกที่สุด

### การสร้าง NSBlockOperation แบบพื้นฐาน

```objc
#import <Foundation/Foundation.h>

int main() {
    @autoreleasepool {
        // สร้าง block operation แบบง่ายที่สุด
        NSBlockOperation *op = [NSBlockOperation blockOperationWithBlock:^{
            NSLog(@"ทำงานใน block operation");
            NSLog(@"Thread: %@", [NSThread currentThread]);
        }];
        
        // เพิ่ม completion block
        op.completionBlock = ^{
            NSLog(@"Block operation เสร็จสิ้น");
        };
        
        // start() ทำงานใน thread ปัจจุบัน (synchronous)
        [op start];
        
        return 0;
    }
}
```

### การเพิ่มหลาย Blocks ใน NSBlockOperation

```objc
// NSBlockOperation สามารถมีหลาย blocks ที่ทำงานพร้อมกันได้
NSBlockOperation *multiBlockOp = [[NSBlockOperation alloc] init];

[multiBlockOp addExecutionBlock:^{
    NSLog(@"Block 1 กำลังทำงาน - Thread: %@", [NSThread currentThread]);
    [NSThread sleepForTimeInterval:1.0];
    NSLog(@"Block 1 เสร็จแล้ว");
}];

[multiBlockOp addExecutionBlock:^{
    NSLog(@"Block 2 กำลังทำงาน - Thread: %@", [NSThread currentThread]);
    [NSThread sleepForTimeInterval:0.5];
    NSLog(@"Block 2 เสร็จแล้ว");
}];

[multiBlockOp addExecutionBlock:^{
    NSLog(@"Block 3 กำลังทำงาน - Thread: %@", [NSThread currentThread]);
    [NSThread sleepForTimeInterval:0.7];
    NSLog(@"Block 3 เสร็จแล้ว");
}];

// Completion block จะถูกเรียกเมื่อทุก blocks เสร็จสิ้น
multiBlockOp.completionBlock = ^{
    NSLog(@"ทุก blocks เสร็จสิ้น!");
};

[multiBlockOp start];
```

### ตัวอย่าง NSBlockOperation กับ NSOperationQueue

```objc
NSOperationQueue *queue = [[NSOperationQueue alloc] init];
queue.maxConcurrentOperationCount = 2; // จำกัด 2 operations พร้อมกัน

// สร้าง operations หลายอัน
for (int i = 1; i <= 5; i++) {
    NSBlockOperation *op = [NSBlockOperation blockOperationWithBlock:^{
        NSLog(@"Operation %d เริ่มทำงาน", i);
        [NSThread sleepForTimeInterval:1.0];
        NSLog(@"Operation %d เสร็จแล้ว", i);
    }];
    
    op.name = [NSString stringWithFormat:@"Operation-%d", i];
    [queue addOperation:op];
}

// รอให้ทุก operations เสร็จ
[queue waitUntilAllOperationsAreFinished];
NSLog(@"ทุก operations เสร็จสิ้น");
```

---

## 33.3 Custom NSOperation Subclass

สำหรับงานที่ซับซ้อนขึ้น เราควรสร้าง NSOperation subclass เอง วิธีนี้ช่วยให้โค้ดสะอาดและนำกลับมาใช้ใหม่ได้

### Non-Concurrent Operation (แบบพื้นฐาน)

```objc
// DataProcessingOperation.h
#import <Foundation/Foundation.h>

@interface DataProcessingOperation : NSOperation

@property (nonatomic, strong) NSData *inputData;
@property (nonatomic, strong) NSData *outputData;
@property (nonatomic, strong) NSError *error;

- (instancetype)initWithData:(NSData *)data;

@end
```

```objc
// DataProcessingOperation.m
#import "DataProcessingOperation.h"

@implementation DataProcessingOperation

- (instancetype)initWithData:(NSData *)data {
    self = [super init];
    if (self) {
        _inputData = data;
    }
    return self;
}

// Override main สำหรับ non-concurrent operation
- (void)main {
    // ตรวจสอบว่าถูกยกเลิกหรือยัง
    if (self.isCancelled) {
        NSLog(@"Operation ถูกยกเลิกก่อนเริ่มทำงาน");
        return;
    }
    
    NSLog(@"เริ่มประมวลผลข้อมูล...");
    
    // จำลองการประมวลผลข้อมูล
    [NSThread sleepForTimeInterval:2.0];
    
    // ตรวจสอบอีกครั้งระหว่างทำงาน
    if (self.isCancelled) {
        NSLog(@"Operation ถูกยกเลิกระหว่างทำงาน");
        return;
    }
    
    // ประมวลผลข้อมูล (ตัวอย่าง: แปลงเป็น uppercase string)
    NSString *inputString = [[NSString alloc] initWithData:self.inputData 
                                                  encoding:NSUTF8StringEncoding];
    NSString *outputString = [inputString uppercaseString];
    self.outputData = [outputString dataUsingEncoding:NSUTF8StringEncoding];
    
    NSLog(@"ประมวลผลเสร็จแล้ว: %@", outputString);
}

@end
```

### การใช้งาน Custom Operation

```objc
// สร้างและใช้งาน custom operation
NSData *testData = [@"hello world from objective-c" dataUsingEncoding:NSUTF8StringEncoding];
DataProcessingOperation *processOp = [[DataProcessingOperation alloc] initWithData:testData];

processOp.completionBlock = ^{
    if (processOp.error) {
        NSLog(@"เกิดข้อผิดพลาด: %@", processOp.error.localizedDescription);
    } else {
        NSString *result = [[NSString alloc] initWithData:processOp.outputData 
                                                 encoding:NSUTF8StringEncoding];
        NSLog(@"ผลลัพธ์: %@", result);
    }
};

NSOperationQueue *queue = [[NSOperationQueue alloc] init];
[queue addOperation:processOp];
[queue waitUntilAllOperationsAreFinished];
```

### Concurrent Operation (แบบขั้นสูง)

สำหรับงานที่ทำงานแบบ asynchronous จริงๆ (เช่น network requests) ต้องสร้าง concurrent operation:

```objc
// AsyncNetworkOperation.h
#import <Foundation/Foundation.h>

@interface AsyncNetworkOperation : NSOperation

@property (nonatomic, strong) NSURL *url;
@property (nonatomic, strong) NSData *responseData;
@property (nonatomic, strong) NSError *error;

- (instancetype)initWithURL:(NSURL *)url;

@end
```

```objc
// AsyncNetworkOperation.m
#import "AsyncNetworkOperation.h"

@interface AsyncNetworkOperation ()

// Internal state variables
@property (nonatomic, assign, getter=isExecuting) BOOL executing;
@property (nonatomic, assign, getter=isFinished) BOOL finished;

@end

@implementation AsyncNetworkOperation

@synthesize executing = _executing;
@synthesize finished = _finished;

- (instancetype)initWithURL:(NSURL *)url {
    self = [super init];
    if (self) {
        _url = url;
        _executing = NO;
        _finished = NO;
    }
    return self;
}

// Concurrent operation ต้อง override isConcurrent/isAsynchronous
- (BOOL)isConcurrent {
    return YES;
}

- (BOOL)isAsynchronous {
    return YES;
}

// ต้อง override isExecuting และ isFinished
- (BOOL)isExecuting {
    return _executing;
}

- (BOOL)isFinished {
    return _finished;
}

- (void)start {
    // ตรวจสอบ cancellation ก่อนเริ่ม
    if (self.isCancelled) {
        [self willChangeValueForKey:@"isFinished"];
        _finished = YES;
        [self didChangeValueForKey:@"isFinished"];
        return;
    }
    
    // อัปเดตสถานะ isExecuting
    [self willChangeValueForKey:@"isExecuting"];
    _executing = YES;
    [self didChangeValueForKey:@"isExecuting"];
    
    // เริ่มทำงาน asynchronous
    [self main];
}

- (void)main {
    NSURLSession *session = [NSURLSession sharedSession];
    NSURLSessionDataTask *task = [session dataTaskWithURL:self.url 
                                       completionHandler:^(NSData *data, NSURLResponse *response, NSError *error) {
        
        if (self.isCancelled) {
            [self completeOperation];
            return;
        }
        
        if (error) {
            self.error = error;
            NSLog(@"เกิดข้อผิดพลาด: %@", error.localizedDescription);
        } else {
            self.responseData = data;
            NSLog(@"ได้รับข้อมูล %lu bytes", (unsigned long)data.length);
        }
        
        [self completeOperation];
    }];
    
    [task resume];
}

- (void)completeOperation {
    [self willChangeValueForKey:@"isFinished"];
    [self willChangeValueForKey:@"isExecuting"];
    
    _executing = NO;
    _finished = YES;
    
    [self didChangeValueForKey:@"isExecuting"];
    [self didChangeValueForKey:@"isFinished"];
}

- (void)cancel {
    [super cancel];
    // ยกเลิก network task ถ้ายังทำงานอยู่
    // (ในตัวอย่างนี้เราไม่ได้เก็บ task reference ไว้ แต่ในการใช้งานจริงควรทำ)
}

@end
```

---

## 33.4 NSOperationQueue

`NSOperationQueue` เป็น queue สำหรับจัดการและรัน NSOperation objects มันจัดการ threading ให้เราโดยอัตโนมัติ

### การสร้างและกำหนดค่า NSOperationQueue

```objc
// สร้าง operation queue
NSOperationQueue *myQueue = [[NSOperationQueue alloc] init];

// ตั้งชื่อ queue (ช่วยในการ debug)
myQueue.name = @"com.myapp.dataProcessingQueue";

// กำหนดจำนวน operations ที่ทำงานพร้อมกัน
myQueue.maxConcurrentOperationCount = 3;

// กำหนด QoS
myQueue.qualityOfService = NSQualityOfServiceUserInitiated;

// ตรวจสอบจำนวน operations ที่รออยู่
NSLog(@"Operations รออยู่: %lu", (unsigned long)myQueue.operationCount);
```

### Main Queue vs Background Queue

```objc
// Main Queue - สำหรับอัปเดต UI
NSOperationQueue *mainQueue = [NSOperationQueue mainQueue];

// Background Queue - สำหรับงานหนัก
NSOperationQueue *backgroundQueue = [[NSOperationQueue alloc] init];
backgroundQueue.qualityOfService = NSQualityOfServiceBackground;

// ตัวอย่างการใช้งาน: ประมวลผลในพื้นหลัง แล้วอัปเดต UI ใน main thread
NSBlockOperation *backgroundWork = [NSBlockOperation blockOperationWithBlock:^{
    // ทำงานหนักในพื้นหลัง
    NSLog(@"กำลังประมวลผลข้อมูล...");
    [NSThread sleepForTimeInterval:2.0];
    NSLog(@"ประมวลผลเสร็จ");
}];

backgroundWork.completionBlock = ^{
    // อัปเดต UI ใน main thread
    [[NSOperationQueue mainQueue] addOperationWithBlock:^{
        NSLog(@"อัปเดต UI ใน main thread: %@", [NSThread isMainThread] ? @"YES" : @"NO");
        // [self.label setText:@"เสร็จแล้ว!"];
    }];
};

[backgroundQueue addOperation:backgroundWork];
```

### การเพิ่ม Operations ลง Queue

```objc
NSOperationQueue *queue = [[NSOperationQueue alloc] init];

// วิธีที่ 1: เพิ่มทีละอัน
NSBlockOperation *op1 = [NSBlockOperation blockOperationWithBlock:^{
    NSLog(@"Operation 1");
}];
[queue addOperation:op1];

// วิธีที่ 2: เพิ่มหลายอันพร้อมกัน
NSOperation *op2 = [[NSBlockOperation alloc] init];
NSOperation *op3 = [[NSBlockOperation alloc] init];
[queue addOperations:@[op2, op3] waitUntilFinished:NO];

// วิธีที่ 3: เพิ่ม block โดยตรง
[queue addOperationWithBlock:^{
    NSLog(@"Operation inline");
}];

// วิธีที่ 4: เพิ่มพร้อมรอให้เสร็จ
[queue addOperations:@[op1, op2] waitUntilFinished:YES];
NSLog(@"Operations เสร็จแล้ว");
```

### หยุดและหยุดชั่วคราว Queue

```objc
NSOperationQueue *queue = [[NSOperationQueue alloc] init];

// หยุดชั่วคราว (suspend)
queue.suspended = YES;
NSLog(@"Queue หยุดชั่วคราว: %d", queue.isSuspended);

// เพิ่ม operations ในขณะที่ suspend - จะรอจนกว่าจะ resume
[queue addOperationWithBlock:^{
    NSLog(@"Operation นี้จะรอจนกว่า queue จะ resume");
}];

// Resume
queue.suspended = NO;
NSLog(@"Queue resume แล้ว");

// ยกเลิกทุก operations ที่รออยู่ (แต่ไม่หยุด operations ที่กำลังทำงาน)
[queue cancelAllOperations];

// รอให้ทุก operations เสร็จ
[queue waitUntilAllOperationsAreFinished];
```

---

## 33.5 Dependencies ระหว่าง Operations

หนึ่งในฟีเจอร์เด่นของ NSOperation คือการกำหนด dependencies ทำให้เราควบคุมลำดับการทำงานได้

### การกำหนด Dependencies พื้นฐาน

```objc
NSOperationQueue *queue = [[NSOperationQueue alloc] init];

NSBlockOperation *downloadOp = [NSBlockOperation blockOperationWithBlock:^{
    NSLog(@"1. ดาวน์โหลดข้อมูล...");
    [NSThread sleepForTimeInterval:1.0];
    NSLog(@"1. ดาวน์โหลดเสร็จ");
}];
downloadOp.name = @"DownloadOperation";

NSBlockOperation *parseOp = [NSBlockOperation blockOperationWithBlock:^{
    NSLog(@"2. Parse ข้อมูล...");
    [NSThread sleepForTimeInterval:0.5];
    NSLog(@"2. Parse เสร็จ");
}];
parseOp.name = @"ParseOperation";

NSBlockOperation *saveOp = [NSBlockOperation blockOperationWithBlock:^{
    NSLog(@"3. บันทึกข้อมูล...");
    [NSThread sleepForTimeInterval:0.3];
    NSLog(@"3. บันทึกเสร็จ");
}];
saveOp.name = @"SaveOperation";

NSBlockOperation *notifyOp = [NSBlockOperation blockOperationWithBlock:^{
    NSLog(@"4. แจ้งเตือน UI...");
    NSLog(@"4. แจ้งเตือนเสร็จ");
}];
notifyOp.name = @"NotifyOperation";

// กำหนด dependencies
// parseOp จะเริ่มได้ก็ต่อเมื่อ downloadOp เสร็จ
[parseOp addDependency:downloadOp];

// saveOp จะเริ่มได้ก็ต่อเมื่อ parseOp เสร็จ
[saveOp addDependency:parseOp];

// notifyOp จะเริ่มได้ก็ต่อเมื่อ saveOp เสร็จ
[notifyOp addDependency:saveOp];

// เพิ่มทุก operations ลง queue (ไม่ต้องเรียงลำดับ เพราะมี dependencies)
[queue addOperations:@[notifyOp, saveOp, parseOp, downloadOp] 
  waitUntilFinished:YES];

NSLog(@"ทุกขั้นตอนเสร็จสิ้น");
```

### Dependencies แบบ Fan-out และ Fan-in

```objc
NSOperationQueue *queue = [[NSOperationQueue alloc] init];
queue.maxConcurrentOperationCount = NSOperationQueueDefaultMaxConcurrentOperationCount;

// Fan-out: operation หนึ่งเสร็จแล้วทำหลาย operations พร้อมกัน
NSBlockOperation *fetchUserOp = [NSBlockOperation blockOperationWithBlock:^{
    NSLog(@"ดึงข้อมูล user...");
    [NSThread sleepForTimeInterval:1.0];
}];

NSBlockOperation *processProfileOp = [NSBlockOperation blockOperationWithBlock:^{
    NSLog(@"ประมวลผล profile...");
    [NSThread sleepForTimeInterval:0.8];
}];

NSBlockOperation *processFriendsOp = [NSBlockOperation blockOperationWithBlock:^{
    NSLog(@"ประมวลผล friends list...");
    [NSThread sleepForTimeInterval:0.6];
}];

NSBlockOperation *processPostsOp = [NSBlockOperation blockOperationWithBlock:^{
    NSLog(@"ประมวลผล posts...");
    [NSThread sleepForTimeInterval:1.2];
}];

// Fan-in: รอทุก operations เสร็จก่อนทำ final operation
NSBlockOperation *updateUIop = [NSBlockOperation blockOperationWithBlock:^{
    NSLog(@"อัปเดต UI ด้วยข้อมูลทั้งหมด");
}];

// processProfile, processFriends, processPosts รอ fetchUser เสร็จก่อน
[processProfileOp addDependency:fetchUserOp];
[processFriendsOp addDependency:fetchUserOp];
[processPostsOp addDependency:fetchUserOp];

// updateUI รอทั้ง 3 process เสร็จ (fan-in)
[updateUIop addDependency:processProfileOp];
[updateUIop addDependency:processFriendsOp];
[updateUIop addDependency:processPostsOp];

[queue addOperations:@[fetchUserOp, processProfileOp, processFriendsOp, 
                       processPostsOp, updateUIop] 
  waitUntilFinished:YES];
```

### การส่งข้อมูลระหว่าง Operations ที่มี Dependencies

```objc
// สร้าง class ที่แชร์ข้อมูลระหว่าง operations
@interface SharedDataStore : NSObject
@property (nonatomic, strong) NSArray *downloadedItems;
@property (nonatomic, strong) NSArray *processedItems;
@end

@implementation SharedDataStore
@end

// ใช้งาน
SharedDataStore *dataStore = [[SharedDataStore alloc] init];

NSBlockOperation *downloadOp = [NSBlockOperation blockOperationWithBlock:^{
    NSLog(@"ดาวน์โหลดข้อมูล...");
    [NSThread sleepForTimeInterval:0.5];
    // จำลองข้อมูลที่ดาวน์โหลดมา
    dataStore.downloadedItems = @[@"item1", @"item2", @"item3"];
    NSLog(@"ดาวน์โหลดเสร็จ: %lu รายการ", (unsigned long)dataStore.downloadedItems.count);
}];

NSBlockOperation *processOp = [NSBlockOperation blockOperationWithBlock:^{
    NSLog(@"ประมวลผลข้อมูล %lu รายการ...", (unsigned long)dataStore.downloadedItems.count);
    NSMutableArray *processed = [NSMutableArray array];
    for (NSString *item in dataStore.downloadedItems) {
        [processed addObject:[item uppercaseString]];
    }
    dataStore.processedItems = processed;
    NSLog(@"ประมวลผลเสร็จ: %@", dataStore.processedItems);
}];

[processOp addDependency:downloadOp];

NSOperationQueue *queue = [[NSOperationQueue alloc] init];
[queue addOperations:@[downloadOp, processOp] waitUntilFinished:YES];
```

---

## 33.6 Operation Priorities

NSOperation รองรับการกำหนดลำดับความสำคัญสองระดับ: `queuePriority` และ `qualityOfService`

### queuePriority

```objc
typedef enum : NSInteger {
    NSOperationQueuePriorityVeryLow     = -8,
    NSOperationQueuePriorityLow         = -4,
    NSOperationQueuePriorityNormal      =  0,  // ค่าเริ่มต้น
    NSOperationQueuePriorityHigh        =  4,
    NSOperationQueuePriorityVeryHigh    =  8,
} NSOperationQueuePriority;
```

```objc
NSOperationQueue *queue = [[NSOperationQueue alloc] init];
queue.maxConcurrentOperationCount = 1; // ทำทีละอัน เพื่อเห็นผลของ priority

NSBlockOperation *lowPriorityOp = [NSBlockOperation blockOperationWithBlock:^{
    NSLog(@"Low priority operation ทำงาน");
}];
lowPriorityOp.queuePriority = NSOperationQueuePriorityLow;

NSBlockOperation *normalPriorityOp = [NSBlockOperation blockOperationWithBlock:^{
    NSLog(@"Normal priority operation ทำงาน");
}];
normalPriorityOp.queuePriority = NSOperationQueuePriorityNormal;

NSBlockOperation *highPriorityOp = [NSBlockOperation blockOperationWithBlock:^{
    NSLog(@"High priority operation ทำงาน");
}];
highPriorityOp.queuePriority = NSOperationQueuePriorityHigh;

NSBlockOperation *veryHighPriorityOp = [NSBlockOperation blockOperationWithBlock:^{
    NSLog(@"Very high priority operation ทำงาน");
}];
veryHighPriorityOp.queuePriority = NSOperationQueuePriorityVeryHigh;

// เพิ่มใน queue แบบสุ่ม แต่ high priority จะทำงานก่อน
[queue addOperation:lowPriorityOp];
[queue addOperation:normalPriorityOp];
[queue addOperation:highPriorityOp];
[queue addOperation:veryHighPriorityOp];

[queue waitUntilAllOperationsAreFinished];
// ผลลัพธ์: veryHigh -> high -> normal -> low
```

### qualityOfService (QoS)

```objc
typedef enum : NSInteger {
    NSQualityOfServiceUserInteractive = 0x21,  // งาน UI ที่ต้องตอบสนองทันที
    NSQualityOfServiceUserInitiated   = 0x19,  // งานที่ user รอผล
    NSQualityOfServiceDefault         = -1,    // ค่าเริ่มต้น
    NSQualityOfServiceUtility         = 0x11,  // งาน background ที่ user รู้ตัว
    NSQualityOfServiceBackground      = 0x09,  // งาน background ที่ user ไม่รู้ตัว
} NSQualityOfService;
```

```objc
// Operation ที่ user รอผล (เช่น search)
NSBlockOperation *searchOp = [NSBlockOperation blockOperationWithBlock:^{
    NSLog(@"ค้นหาข้อมูล...");
}];
searchOp.qualityOfService = NSQualityOfServiceUserInitiated;

// Operation ที่ทำใน background (เช่น sync ข้อมูล)
NSBlockOperation *syncOp = [NSBlockOperation blockOperationWithBlock:^{
    NSLog(@"Sync ข้อมูล...");
}];
syncOp.qualityOfService = NSQualityOfServiceBackground;
```

---

## 33.7 Cancellation

การยกเลิก operation เป็นสิ่งสำคัญสำหรับการจัดการทรัพยากรและ user experience ที่ดี

### การยกเลิก Operation เดี่ยว

```objc
// สร้าง operation ที่รองรับ cancellation
NSBlockOperation *longRunningOp = [NSBlockOperation blockOperationWithBlock:^{
    for (int i = 0; i < 100; i++) {
        // ตรวจสอบ cancellation ทุกรอบ
        // ใช้ weakSelf เพื่อหลีกเลี่ยง retain cycle
        // ในตัวอย่างนี้เราใช้ currentOperation แทน
        NSOperation *currentOp = [NSOperationQueue currentQueue].operations.firstObject;
        if (currentOp.isCancelled) {
            NSLog(@"Operation ถูกยกเลิกที่ step %d", i);
            return;
        }
        
        NSLog(@"ทำงาน step %d", i);
        [NSThread sleepForTimeInterval:0.1];
    }
    NSLog(@"ทำงานเสร็จทั้งหมด");
}];

NSOperationQueue *queue = [[NSOperationQueue alloc] init];
[queue addOperation:longRunningOp];

// ยกเลิกหลังจาก 0.5 วินาที
[NSThread sleepForTimeInterval:0.5];
[longRunningOp cancel];
NSLog(@"ส่งคำสั่งยกเลิก");

[queue waitUntilAllOperationsAreFinished];
```

### Custom Operation ที่รองรับ Cancellation

```objc
@interface CancellableOperation : NSOperation

@property (nonatomic, strong) NSString *taskName;
@property (nonatomic, copy) void (^progressBlock)(float progress);

- (instancetype)initWithTaskName:(NSString *)name;

@end

@implementation CancellableOperation

- (instancetype)initWithTaskName:(NSString *)name {
    self = [super init];
    if (self) {
        _taskName = name;
    }
    return self;
}

- (void)main {
    NSLog(@"เริ่ม: %@", self.taskName);
    
    // จำลองงานที่ใช้เวลานาน
    int totalSteps = 20;
    for (int step = 0; step < totalSteps; step++) {
        // ตรวจสอบ cancellation ก่อนทุก step
        if (self.isCancelled) {
            NSLog(@"ยกเลิก %@ ที่ step %d/%d", self.taskName, step, totalSteps);
            return;
        }
        
        // รายงาน progress
        float progress = (float)(step + 1) / totalSteps;
        if (self.progressBlock) {
            self.progressBlock(progress);
        }
        
        [NSThread sleepForTimeInterval:0.2];
    }
    
    NSLog(@"เสร็จ: %@", self.taskName);
}

@end
```

### ยกเลิกทุก Operations ใน Queue

```objc
NSOperationQueue *queue = [[NSOperationQueue alloc] init];
queue.maxConcurrentOperationCount = 2;

// เพิ่ม operations จำนวนมาก
for (int i = 0; i < 10; i++) {
    CancellableOperation *op = [[CancellableOperation alloc] initWithTaskName:
                                [NSString stringWithFormat:@"Task-%d", i]];
    [queue addOperation:op];
}

NSLog(@"Operations ใน queue: %lu", (unsigned long)queue.operationCount);

// ยกเลิกทุกอย่าง
[NSThread sleepForTimeInterval:0.5];
[queue cancelAllOperations];
NSLog(@"ยกเลิกทุก operations แล้ว");

[queue waitUntilAllOperationsAreFinished];
NSLog(@"เสร็จสิ้น");
```

---

## 33.8 Max Concurrent Operations

การควบคุมจำนวน operations ที่ทำงานพร้อมกันเป็นสิ่งสำคัญสำหรับการจัดการทรัพยากร

```objc
NSOperationQueue *queue = [[NSOperationQueue alloc] init];

// ค่าเริ่มต้น: NSOperationQueueDefaultMaxConcurrentOperationCount (-1)
// ระบบจะกำหนดจำนวนที่เหมาะสมเอง
NSLog(@"ค่าเริ่มต้น maxConcurrentOperationCount: %ld", 
      (long)queue.maxConcurrentOperationCount);

// กำหนดให้ทำทีละอัน (serial queue)
queue.maxConcurrentOperationCount = 1;

// กำหนดให้ทำพร้อมกันสูงสุด 4 operations
queue.maxConcurrentOperationCount = 4;

// ตัวอย่างการใช้งานจริง: จำกัด network connections
NSOperationQueue *networkQueue = [[NSOperationQueue alloc] init];
networkQueue.name = @"com.myapp.networkQueue";
networkQueue.maxConcurrentOperationCount = 3; // จำกัดเพื่อไม่ให้ server รับภาระมากเกินไป

// ตัวอย่าง: ดาวน์โหลดรูปภาพ 10 รูปพร้อมกันสูงสุด 3 รูป
NSArray *imageURLs = @[
    @"https://example.com/image1.jpg",
    @"https://example.com/image2.jpg",
    @"https://example.com/image3.jpg",
    @"https://example.com/image4.jpg",
    @"https://example.com/image5.jpg",
];

for (NSString *urlString in imageURLs) {
    NSBlockOperation *downloadOp = [NSBlockOperation blockOperationWithBlock:^{
        NSLog(@"ดาวน์โหลด: %@", urlString);
        [NSThread sleepForTimeInterval:1.0]; // จำลองการดาวน์โหลด
        NSLog(@"เสร็จ: %@", urlString);
    }];
    [networkQueue addOperation:downloadOp];
}

[networkQueue waitUntilAllOperationsAreFinished];
```

---

## 33.9 เปรียบเทียบ NSOperation กับ GCD

```
┌─────────────────────────────────────────────────────────────────┐
│                NSOperation vs GCD                                │
├─────────────────┬──────────────────┬──────────────────────────── │
│ Feature         │ NSOperation       │ GCD                        │
├─────────────────┼──────────────────┼──────────────────────────── │
│ Dependencies    │ ✅ รองรับ          │ ❌ ต้องทำเอง               │
│ Cancellation    │ ✅ รองรับ          │ ❌ ต้องทำเอง               │
│ State Tracking  │ ✅ รองรับ          │ ❌ ไม่รองรับ               │
│ Priority        │ ✅ รองรับ          │ ✅ รองรับผ่าน QoS          │
│ KVO             │ ✅ รองรับ          │ ❌ ไม่รองรับ               │
│ Reusability     │ ✅ สูง (subclass) │ ❌ ต่ำ (inline blocks)     │
│ Overhead        │ 🔶 สูงกว่า        │ ✅ ต่ำกว่า                 │
│ Simplicity      │ 🔶 ซับซ้อนกว่า   │ ✅ ง่ายกว่า                │
│ Built on        │ GCD               │ OS Level                   │
└─────────────────┴──────────────────┴────────────────────────────┘
```

### เมื่อไหรควรใช้ NSOperation?

```objc
// ใช้ NSOperation เมื่อ:
// 1. ต้องการ dependencies ระหว่าง operations
// 2. ต้องการ cancel operations ได้
// 3. ต้องการ track state ของ operation
// 4. ต้องการ reuse code (custom subclass)
// 5. ต้องการ limit concurrent operations ได้ยืดหยุ่น

// ใช้ GCD เมื่อ:
// 1. งานง่ายๆ ที่ต้องการ run async
// 2. One-shot tasks ที่ไม่ต้องการ control
// 3. ต้องการ performance สูงสุด
// 4. ใช้ dispatch_group, dispatch_barrier, dispatch_semaphore

// ตัวอย่าง: งานง่ายๆ ใช้ GCD
dispatch_async(dispatch_get_global_queue(QOS_CLASS_USER_INITIATED, 0), ^{
    // ทำงาน async
    dispatch_async(dispatch_get_main_queue(), ^{
        // อัปเดต UI
    });
});

// ตัวอย่าง: workflow ที่ซับซ้อน ใช้ NSOperation
NSOperationQueue *queue = [[NSOperationQueue alloc] init];
NSBlockOperation *step1 = [NSBlockOperation blockOperationWithBlock:^{ /* ... */ }];
NSBlockOperation *step2 = [NSBlockOperation blockOperationWithBlock:^{ /* ... */ }];
NSBlockOperation *step3 = [NSBlockOperation blockOperationWithBlock:^{ /* ... */ }];
[step2 addDependency:step1];
[step3 addDependency:step2];
[queue addOperations:@[step1, step2, step3] waitUntilFinished:YES];
```

---

## 33.10 ตัวอย่างจริง: Image Downloader

```objc
// ImageDownloadOperation.h
#import <Foundation/Foundation.h>

@interface ImageDownloadOperation : NSOperation

@property (nonatomic, strong) NSURL *imageURL;
@property (nonatomic, strong) NSData *imageData;
@property (nonatomic, strong) NSString *imageName;
@property (nonatomic, strong) NSError *downloadError;

- (instancetype)initWithURL:(NSURL *)url name:(NSString *)name;

@end
```

```objc
// ImageDownloadOperation.m
#import "ImageDownloadOperation.h"

@implementation ImageDownloadOperation

- (instancetype)initWithURL:(NSURL *)url name:(NSString *)name {
    self = [super init];
    if (self) {
        _imageURL = url;
        _imageName = name;
    }
    return self;
}

- (void)main {
    if (self.isCancelled) return;
    
    NSLog(@"เริ่มดาวน์โหลด: %@", self.imageName);
    
    NSError *error = nil;
    NSData *data = [NSData dataWithContentsOfURL:self.imageURL options:0 error:&error];
    
    if (self.isCancelled) return;
    
    if (error) {
        self.downloadError = error;
        NSLog(@"ดาวน์โหลดล้มเหลว %@: %@", self.imageName, error.localizedDescription);
    } else {
        self.imageData = data;
        NSLog(@"ดาวน์โหลดสำเร็จ %@: %lu bytes", self.imageName, (unsigned long)data.length);
    }
}

@end
```

```objc
// ImageProcessingOperation.h
#import <Foundation/Foundation.h>
@class ImageDownloadOperation;

@interface ImageProcessingOperation : NSOperation

@property (nonatomic, weak) ImageDownloadOperation *downloadOperation;
@property (nonatomic, strong) NSData *processedData;

- (instancetype)initWithDownloadOperation:(ImageDownloadOperation *)downloadOp;

@end
```

```objc
// ImageProcessingOperation.m
#import "ImageProcessingOperation.h"
#import "ImageDownloadOperation.h"

@implementation ImageProcessingOperation

- (instancetype)initWithDownloadOperation:(ImageDownloadOperation *)downloadOp {
    self = [super init];
    if (self) {
        _downloadOperation = downloadOp;
        // เพิ่ม dependency โดยอัตโนมัติ
        [self addDependency:downloadOp];
    }
    return self;
}

- (void)main {
    if (self.isCancelled) return;
    
    // ตรวจสอบว่า download สำเร็จหรือไม่
    if (self.downloadOperation.downloadError || !self.downloadOperation.imageData) {
        NSLog(@"ไม่สามารถประมวลผลได้ เพราะ download ล้มเหลว");
        return;
    }
    
    NSLog(@"กำลังประมวลผลรูปภาพ: %@", self.downloadOperation.imageName);
    
    // จำลองการประมวลผล (resize, filter, compress ฯลฯ)
    [NSThread sleepForTimeInterval:0.5];
    
    // ในตัวอย่างนี้ เราเก็บข้อมูลเดิมไว้
    self.processedData = self.downloadOperation.imageData;
    
    NSLog(@"ประมวลผลเสร็จ: %@", self.downloadOperation.imageName);
}

@end
```

```objc
// การใช้งาน Image Downloader
void runImageDownloader() {
    NSArray *imageInfos = @[
        @{@"name": @"cat", @"url": @"https://picsum.photos/200/200?1"},
        @{@"name": @"dog", @"url": @"https://picsum.photos/200/200?2"},
        @{@"name": @"bird", @"url": @"https://picsum.photos/200/200?3"},
    ];
    
    NSOperationQueue *downloadQueue = [[NSOperationQueue alloc] init];
    downloadQueue.name = @"com.myapp.imageDownloadQueue";
    downloadQueue.maxConcurrentOperationCount = 2;
    
    NSOperationQueue *processQueue = [[NSOperationQueue alloc] init];
    processQueue.name = @"com.myapp.imageProcessQueue";
    processQueue.maxConcurrentOperationCount = 1;
    
    NSMutableArray *allOperations = [NSMutableArray array];
    
    for (NSDictionary *info in imageInfos) {
        NSURL *url = [NSURL URLWithString:info[@"url"]];
        NSString *name = info[@"name"];
        
        // สร้าง download operation
        ImageDownloadOperation *downloadOp = [[ImageDownloadOperation alloc] 
                                              initWithURL:url name:name];
        
        // สร้าง process operation (มี dependency ต่อ download operation อยู่แล้ว)
        ImageProcessingOperation *processOp = [[ImageProcessingOperation alloc] 
                                               initWithDownloadOperation:downloadOp];
        
        processOp.completionBlock = ^{
            if (processOp.processedData) {
                NSLog(@"✅ %@ พร้อมแสดง (%lu bytes)", name, 
                      (unsigned long)processOp.processedData.length);
            } else {
                NSLog(@"❌ %@ ล้มเหลว", name);
            }
        };
        
        [downloadQueue addOperation:downloadOp];
        [processQueue addOperation:processOp];
        
        [allOperations addObject:downloadOp];
        [allOperations addObject:processOp];
    }
    
    // รอทุก operations เสร็จ
    [downloadQueue waitUntilAllOperationsAreFinished];
    [processQueue waitUntilAllOperationsAreFinished];
    
    NSLog(@"ทุกรูปภาพดาวน์โหลดและประมวลผลเสร็จแล้ว");
}
```

---

## 33.11 ตัวอย่างจริง: Data Processing Pipeline

```objc
// DataPipeline.m - ตัวอย่าง pipeline สำหรับประมวลผลข้อมูล
#import <Foundation/Foundation.h>

@interface DataPipeline : NSObject

@property (nonatomic, strong) NSOperationQueue *pipelineQueue;
@property (nonatomic, strong) NSMutableArray *results;

- (void)processDataItems:(NSArray *)items 
              completion:(void (^)(NSArray *results))completion;

@end

@implementation DataPipeline

- (instancetype)init {
    self = [super init];
    if (self) {
        _pipelineQueue = [[NSOperationQueue alloc] init];
        _pipelineQueue.name = @"com.myapp.dataPipeline";
        _pipelineQueue.maxConcurrentOperationCount = 4;
        _results = [NSMutableArray array];
    }
    return self;
}

- (NSOperation *)createValidationOperationForItem:(NSDictionary *)item {
    NSBlockOperation *op = [NSBlockOperation blockOperationWithBlock:^{
        NSString *name = item[@"name"];
        if (!name || name.length == 0) {
            NSLog(@"❌ Validation ล้มเหลว: ไม่มีชื่อ");
        } else {
            NSLog(@"✅ Validation ผ่าน: %@", name);
        }
    }];
    op.name = [NSString stringWithFormat:@"Validate-%@", item[@"id"]];
    return op;
}

- (NSOperation *)createTransformOperationForItem:(NSDictionary *)item {
    NSBlockOperation *op = [NSBlockOperation blockOperationWithBlock:^{
        NSString *name = item[@"name"];
        NSString *transformed = [[name lowercaseString] 
                                  stringByReplacingOccurrencesOfString:@" " withString:@"_"];
        NSLog(@"🔄 Transform: %@ -> %@", name, transformed);
        
        @synchronized(self.results) {
            [self.results addObject:@{
                @"id": item[@"id"],
                @"original": name,
                @"transformed": transformed
            }];
        }
    }];
    op.name = [NSString stringWithFormat:@"Transform-%@", item[@"id"]];
    return op;
}

- (void)processDataItems:(NSArray *)items 
              completion:(void (^)(NSArray *results))completion {
    
    [self.results removeAllObjects];
    NSMutableArray *allOps = [NSMutableArray array];
    
    for (NSDictionary *item in items) {
        NSOperation *validateOp = [self createValidationOperationForItem:item];
        NSOperation *transformOp = [self createTransformOperationForItem:item];
        
        // transform รอ validate เสร็จก่อน
        [transformOp addDependency:validateOp];
        
        [allOps addObject:validateOp];
        [allOps addObject:transformOp];
    }
    
    // Completion operation รอทุกอย่างเสร็จ
    NSBlockOperation *completionOp = [NSBlockOperation blockOperationWithBlock:^{
        NSLog(@"Pipeline เสร็จสิ้น: %lu รายการ", (unsigned long)self.results.count);
        if (completion) {
            completion([self.results copy]);
        }
    }];
    
    for (NSOperation *op in allOps) {
        if ([op.name hasPrefix:@"Transform-"]) {
            [completionOp addDependency:op];
        }
    }
    
    [allOps addObject:completionOp];
    [self.pipelineQueue addOperations:allOps waitUntilFinished:NO];
}

@end

// การใช้งาน
void runDataPipeline() {
    DataPipeline *pipeline = [[DataPipeline alloc] init];
    
    NSArray *items = @[
        @{@"id": @"1", @"name": @"Hello World"},
        @{@"id": @"2", @"name": @"Objective C Programming"},
        @{@"id": @"3", @"name": @"iOS Development"},
        @{@"id": @"4", @"name": @"macOS Application"},
    ];
    
    [pipeline processDataItems:items completion:^(NSArray *results) {
        NSLog(@"ผลลัพธ์ทั้งหมด:");
        for (NSDictionary *result in results) {
            NSLog(@"  %@: %@ -> %@", result[@"id"], result[@"original"], result[@"transformed"]);
        }
    }];
    
    // รอให้เสร็จ
    [pipeline.pipelineQueue waitUntilAllOperationsAreFinished];
}
```

---

## 33.12 NSOperation กับ KVO

NSOperation รองรับ KVO โดยอัตโนมัติ ทำให้เราสามารถ observe สถานะของ operation ได้

```objc
@interface OperationObserver : NSObject

@property (nonatomic, strong) NSOperation *operation;

- (instancetype)initWithOperation:(NSOperation *)operation;
- (void)startObserving;
- (void)stopObserving;

@end

@implementation OperationObserver

- (instancetype)initWithOperation:(NSOperation *)operation {
    self = [super init];
    if (self) {
        _operation = operation;
    }
    return self;
}

- (void)startObserving {
    [self.operation addObserver:self 
                     forKeyPath:@"isExecuting" 
                        options:NSKeyValueObservingOptionNew 
                        context:NULL];
    [self.operation addObserver:self 
                     forKeyPath:@"isFinished" 
                        options:NSKeyValueObservingOptionNew 
                        context:NULL];
    [self.operation addObserver:self 
                     forKeyPath:@"isCancelled" 
                        options:NSKeyValueObservingOptionNew 
                        context:NULL];
}

- (void)stopObserving {
    [self.operation removeObserver:self forKeyPath:@"isExecuting"];
    [self.operation removeObserver:self forKeyPath:@"isFinished"];
    [self.operation removeObserver:self forKeyPath:@"isCancelled"];
}

- (void)observeValueForKeyPath:(NSString *)keyPath 
                      ofObject:(id)object 
                        change:(NSDictionary *)change 
                       context:(void *)context {
    if ([keyPath isEqualToString:@"isExecuting"]) {
        BOOL isExecuting = [change[NSKeyValueChangeNewKey] boolValue];
        NSLog(@"Operation %@: isExecuting = %@", 
              self.operation.name, isExecuting ? @"YES" : @"NO");
    } else if ([keyPath isEqualToString:@"isFinished"]) {
        BOOL isFinished = [change[NSKeyValueChangeNewKey] boolValue];
        NSLog(@"Operation %@: isFinished = %@", 
              self.operation.name, isFinished ? @"YES" : @"NO");
        if (isFinished) {
            [self stopObserving];
        }
    } else if ([keyPath isEqualToString:@"isCancelled"]) {
        BOOL isCancelled = [change[NSKeyValueChangeNewKey] boolValue];
        NSLog(@"Operation %@: isCancelled = %@", 
              self.operation.name, isCancelled ? @"YES" : @"NO");
    }
}

@end
```

---

## 33.13 Best Practices

```objc
// 1. ตรวจสอบ isCancelled เสมอใน main()
- (void)main {
    if (self.isCancelled) return;
    
    // ทำงาน...
    
    if (self.isCancelled) return; // ตรวจสอบระหว่างทาง
    
    // ทำงานต่อ...
}

// 2. ใช้ weak reference ใน completion blocks เพื่อหลีกเลี่ยง retain cycle
__weak typeof(self) weakSelf = self;
op.completionBlock = ^{
    __strong typeof(weakSelf) strongSelf = weakSelf;
    if (strongSelf) {
        [strongSelf handleCompletion];
    }
};

// 3. กำหนดชื่อ queue และ operation ทุกครั้ง (ช่วยใน debug)
queue.name = @"com.myapp.dataProcessingQueue";
op.name = @"ProcessUserData-12345";

// 4. ใช้ QoS ที่เหมาะสม
op.qualityOfService = NSQualityOfServiceUserInitiated; // user กำลังรอ
op.qualityOfService = NSQualityOfServiceBackground;    // background task

// 5. หลีกเลี่ยง circular dependencies
// ❌ Bad: A depends on B, B depends on A
[opA addDependency:opB];
[opB addDependency:opA]; // วนไม่สิ้นสุด!

// ✅ Good: A -> B -> C (linear)
[opB addDependency:opA];
[opC addDependency:opB];
```

---

## แบบฝึกหัด

### แบบฝึกหัดที่ 1: Sequential Download Pipeline
สร้าง pipeline ที่ดาวน์โหลดข้อมูล JSON จาก API แล้ว parse และบันทึกลงไฟล์

```objc
// TODO: สร้าง 3 operations:
// 1. FetchDataOperation - ดาวน์โหลด JSON
// 2. ParseDataOperation - แปลง JSON เป็น objects (depends on 1)
// 3. SaveDataOperation - บันทึก objects ลงไฟล์ (depends on 2)

// ใช้ NSOperationQueue และ dependencies
// รองรับ cancellation ทุก operations
```

### แบบฝึกหัดที่ 2: Priority Queue
สร้าง download manager ที่:
- รองรับ high/normal/low priority downloads
- จำกัด concurrent downloads ที่ 3
- รองรับการ pause/resume queue
- รองรับการ cancel รายการใด
- แสดง progress ของแต่ละรายการ

```objc
@interface DownloadManager : NSObject

- (void)downloadURL:(NSURL *)url 
           priority:(NSOperationQueuePriority)priority
           progress:(void (^)(float progress))progressBlock
         completion:(void (^)(NSData *data, NSError *error))completionBlock;

- (void)pauseAllDownloads;
- (void)resumeAllDownloads;
- (void)cancelAllDownloads;
- (NSArray *)pendingDownloads;

@end
```

### แบบฝึกหัดที่ 3: Batch Image Processor
สร้างโปรแกรมที่:
1. โหลดรายชื่อ URL รูปภาพจากไฟล์
2. ดาวน์โหลดรูปภาพพร้อมกันสูงสุด 3 รูป
3. Compress แต่ละรูป (จำลอง)
4. บันทึกรูปที่ compress แล้วลงโฟลเดอร์
5. สร้าง report สรุปผล

```objc
// ต้องใช้:
// - Custom NSOperation subclass สำหรับแต่ละขั้นตอน
// - NSOperationQueue พร้อม maxConcurrentOperationCount
// - Dependencies ระหว่าง download -> compress -> save
// - Error handling
// - Progress reporting
```

### แบบฝึกหัดที่ 4: Operation Status Monitor
สร้าง UI (console-based) ที่แสดงสถานะของ operations แบบ real-time:

```
[Running]  Download-1   ████████░░  80%
[Waiting]  Download-2   ░░░░░░░░░░   0%  (waiting for: Download-1)
[Done  ]  Process-1   ██████████ 100%
[Cancel]  Save-1       ████░░░░░░  40%
```

---

## สรุป

`NSOperation` และ `NSOperationQueue` เป็นเครื่องมือที่ทรงพลังสำหรับการจัดการงาน concurrent ใน Objective-C มีข้อดีหลักๆ คือ:

- **Dependencies**: ควบคุมลำดับการทำงานได้อย่างชัดเจน
- **Cancellation**: ยกเลิกงานได้เมื่อไม่ต้องการ
- **State Tracking**: รู้สถานะของงานในทุกขณะ  
- **Reusability**: custom subclass สามารถนำกลับมาใช้ใหม่ได้
- **Integration**: ทำงานร่วมกับ KVO, completionBlocks ได้ดี

สำหรับงานที่ซับซ้อน มี dependencies หลายชั้น หรือต้องการ cancel ได้ NSOperation เป็นตัวเลือกที่ดีกว่า GCD แต่สำหรับงานง่ายๆ ที่ต้องการ run async GCD ก็ยังคงเป็นตัวเลือกที่ดีและเรียบง่ายกว่า
