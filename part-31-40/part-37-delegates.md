# Part 37: Delegate Pattern ใน Objective-C

## บทนำ

Delegate Pattern เป็นหนึ่งในรูปแบบการออกแบบ (Design Pattern) ที่สำคัญที่สุดใน Objective-C และ iOS development โดยเฉพาะ Apple ใช้ pattern นี้อย่างแพร่หลายในทุก Framework เช่น UIKit, Foundation และอื่นๆ

ในบทนี้เราจะเรียนรู้:
- Delegate Pattern คืออะไรและทำงานอย่างไร
- การสร้าง Protocol สำหรับ Delegation
- การ implement ระบบ Delegate ตั้งแต่ต้น
- ตัวอย่างจากชีวิตจริงในการพัฒนา iOS
- การเปรียบเทียบกับ Block callbacks
- Best practices และข้อควรระวัง

---

## 37.1 Delegate Pattern คืออะไร?

### แนวคิดพื้นฐาน

Delegate Pattern เป็นรูปแบบที่ให้ object หนึ่ง (delegator) มอบหมายความรับผิดชอบบางส่วนให้กับอีก object หนึ่ง (delegate)

**เปรียบเทียบในชีวิตจริง:**
- นายจ้าง (delegator) มอบหมายงานให้พนักงาน (delegate)
- พนักงานทำงานตามที่ได้รับมอบหมาย และรายงานผลกลับมา
- นายจ้างไม่จำเป็นต้องรู้ว่าพนักงานทำงานอย่างไร เพียงแต่รู้ว่าผลลัพธ์จะออกมาในรูปแบบใด

### ส่วนประกอบของ Delegate Pattern

1. **Protocol** - กำหนด interface ที่ delegate ต้องปฏิบัติตาม
2. **Delegator** - object ที่มี delegate property และเรียกใช้ method ผ่าน delegate
3. **Delegate** - object ที่ adopt protocol และ implement methods

```
┌─────────────────┐         ┌──────────────────────┐
│   Delegator     │         │      Protocol         │
│                 │─────────▶  (เงื่อนไขที่ต้องทำ)  │
│  delegate: id   │         └──────────────────────┘
└────────┬────────┘                    ▲
         │                             │
         │ sends messages              │ conforms to
         ▼                             │
┌─────────────────┐         ┌──────────────────────┐
│   Delegate      │─────────▶    Delegate Object    │
│ (any object     │  adopts  │  (implements methods) │
│  conforming to  │          └──────────────────────┘
│  protocol)      │
└─────────────────┘
```

---

## 37.2 Protocol-Based Delegation

### การสร้าง Protocol

```objc
// MyButtonDelegate.h
@protocol MyButtonDelegate <NSObject>

// Required methods - delegate MUST implement these
@required
- (void)buttonWasTapped:(UIButton *)button;

// Optional methods - delegate MAY implement these
@optional
- (void)buttonWasLongPressed:(UIButton *)button;
- (BOOL)buttonShouldHighlight:(UIButton *)button;

@end
```

### Protocol Inheritance

Protocol สามารถ inherit จาก Protocol อื่นได้:

```objc
// BaseDelegate.h
@protocol BaseDelegate <NSObject>
- (void)operationDidStart:(id)sender;
- (void)operationDidFinish:(id)sender;
@end

// ExtendedDelegate.h
@protocol ExtendedDelegate <BaseDelegate>
- (void)operationDidFail:(id)sender withError:(NSError *)error;
- (void)operationDidUpdateProgress:(id)sender progress:(float)progress;
@end
```

### การ Adopt Protocol

```objc
// ViewController.h
#import <UIKit/UIKit.h>
#import "MyButtonDelegate.h"

@interface ViewController : UIViewController <MyButtonDelegate>

@end

// ViewController.m
@implementation ViewController

// Implement required method
- (void)buttonWasTapped:(UIButton *)button {
    NSLog(@"Button was tapped!");
    // Handle button tap
}

// Implement optional method
- (void)buttonWasLongPressed:(UIButton *)button {
    NSLog(@"Button was long pressed!");
}

// Implement optional method
- (BOOL)buttonShouldHighlight:(UIButton *)button {
    return YES;
}

@end
```

---

## 37.3 การ Implement Delegate System ตั้งแต่ต้น

### ตัวอย่าง: CustomButton with Delegate

#### Step 1: สร้าง Protocol

```objc
// CustomButtonDelegate.h
#import <Foundation/Foundation.h>

@class CustomButton;

@protocol CustomButtonDelegate <NSObject>

@required
- (void)customButton:(CustomButton *)button didTapWithTitle:(NSString *)title;

@optional
- (void)customButton:(CustomButton *)button didLongPressWithTitle:(NSString *)title;
- (BOOL)customButtonShouldBeEnabled:(CustomButton *)button;
- (NSString *)customButtonTitleForNormalState:(CustomButton *)button;

@end
```

#### Step 2: สร้าง CustomButton Class

```objc
// CustomButton.h
#import <UIKit/UIKit.h>
#import "CustomButtonDelegate.h"

@interface CustomButton : UIView

// Delegate property - ใช้ weak เสมอเพื่อป้องกัน retain cycle
@property (nonatomic, weak) id<CustomButtonDelegate> delegate;

@property (nonatomic, copy) NSString *title;
@property (nonatomic, strong) UIColor *backgroundColor;
@property (nonatomic, assign) BOOL enabled;

- (instancetype)initWithTitle:(NSString *)title;
- (void)simulateTap;
- (void)simulateLongPress;

@end

// CustomButton.m
#import "CustomButton.h"

@interface CustomButton ()
@property (nonatomic, strong) UILabel *titleLabel;
@property (nonatomic, strong) UITapGestureRecognizer *tapGesture;
@property (nonatomic, strong) UILongPressGestureRecognizer *longPressGesture;
@end

@implementation CustomButton

- (instancetype)initWithTitle:(NSString *)title {
    self = [super initWithFrame:CGRectMake(0, 0, 200, 50)];
    if (self) {
        _title = [title copy];
        _enabled = YES;
        [self setupUI];
        [self setupGestures];
    }
    return self;
}

- (void)setupUI {
    self.backgroundColor = [UIColor systemBlueColor];
    self.layer.cornerRadius = 8.0;
    
    self.titleLabel = [[UILabel alloc] initWithFrame:self.bounds];
    self.titleLabel.textAlignment = NSTextAlignmentCenter;
    self.titleLabel.textColor = [UIColor whiteColor];
    self.titleLabel.text = self.title;
    [self addSubview:self.titleLabel];
}

- (void)setupGestures {
    self.tapGesture = [[UITapGestureRecognizer alloc] initWithTarget:self
                                                             action:@selector(handleTap:)];
    [self addGestureRecognizer:self.tapGesture];
    
    self.longPressGesture = [[UILongPressGestureRecognizer alloc] initWithTarget:self
                                                                         action:@selector(handleLongPress:)];
    [self addGestureRecognizer:self.longPressGesture];
}

- (void)handleTap:(UITapGestureRecognizer *)gesture {
    if (!self.enabled) return;
    
    // ตรวจสอบว่า delegate implement method นี้หรือไม่
    if ([self.delegate respondsToSelector:@selector(customButton:didTapWithTitle:)]) {
        [self.delegate customButton:self didTapWithTitle:self.title];
    }
}

- (void)handleLongPress:(UILongPressGestureRecognizer *)gesture {
    if (!self.enabled) return;
    if (gesture.state != UIGestureRecognizerStateBegan) return;
    
    // ตรวจสอบ optional method
    if ([self.delegate respondsToSelector:@selector(customButton:didLongPressWithTitle:)]) {
        [self.delegate customButton:self didLongPressWithTitle:self.title];
    }
}

- (void)simulateTap {
    [self handleTap:nil];
}

- (void)simulateLongPress {
    [self handleLongPress:nil];
}

// Override title setter เพื่อ update UI
- (void)setTitle:(NSString *)title {
    _title = [title copy];
    
    // ถาม delegate ว่าต้องการเปลี่ยน title หรือไม่
    if ([self.delegate respondsToSelector:@selector(customButtonTitleForNormalState:)]) {
        NSString *delegateTitle = [self.delegate customButtonTitleForNormalState:self];
        if (delegateTitle) {
            _title = [delegateTitle copy];
        }
    }
    
    self.titleLabel.text = _title;
}

- (void)setEnabled:(BOOL)enabled {
    // ถาม delegate ก่อนว่าควร enable หรือไม่
    if ([self.delegate respondsToSelector:@selector(customButtonShouldBeEnabled:)]) {
        _enabled = [self.delegate customButtonShouldBeEnabled:self];
    } else {
        _enabled = enabled;
    }
    
    self.alpha = _enabled ? 1.0 : 0.5;
    self.userInteractionEnabled = _enabled;
}

@end
```

#### Step 3: ใช้งาน CustomButton

```objc
// ViewController.m
#import "ViewController.h"
#import "CustomButton.h"

@interface ViewController () <CustomButtonDelegate>
@property (nonatomic, strong) CustomButton *myButton;
@property (nonatomic, assign) NSInteger tapCount;
@end

@implementation ViewController

- (void)viewDidLoad {
    [super viewDidLoad];
    
    self.myButton = [[CustomButton alloc] initWithTitle:@"กดที่นี่"];
    self.myButton.center = self.view.center;
    self.myButton.delegate = self;  // ตั้งค่า delegate
    [self.view addSubview:self.myButton];
}

#pragma mark - CustomButtonDelegate (Required)

- (void)customButton:(CustomButton *)button didTapWithTitle:(NSString *)title {
    self.tapCount++;
    NSLog(@"Button '%@' tapped! Count: %ld", title, (long)self.tapCount);
    
    if (self.tapCount >= 5) {
        button.enabled = NO;
        NSLog(@"Button disabled after 5 taps");
    }
}

#pragma mark - CustomButtonDelegate (Optional)

- (void)customButton:(CustomButton *)button didLongPressWithTitle:(NSString *)title {
    NSLog(@"Long press on '%@'", title);
    self.tapCount = 0;
    button.enabled = YES;
    NSLog(@"Count reset!");
}

- (BOOL)customButtonShouldBeEnabled:(CustomButton *)button {
    return self.tapCount < 5;
}

- (NSString *)customButtonTitleForNormalState:(CustomButton *)button {
    return [NSString stringWithFormat:@"กดแล้ว %ld ครั้ง", (long)self.tapCount];
}

@end
```

---

## 37.4 UITableViewDelegate เป็นตัวอย่าง Pattern

UITableViewDelegate เป็นตัวอย่างคลาสสิกของ Delegate Pattern ใน iOS:

```objc
// ตัวอย่างการ implement UITableViewDelegate
@interface ViewController () <UITableViewDelegate, UITableViewDataSource>
@property (nonatomic, strong) UITableView *tableView;
@property (nonatomic, strong) NSArray *items;
@end

@implementation ViewController

- (void)viewDidLoad {
    [super viewDidLoad];
    
    self.items = @[@"รายการที่ 1", @"รายการที่ 2", @"รายการที่ 3", @"รายการที่ 4", @"รายการที่ 5"];
    
    self.tableView = [[UITableView alloc] initWithFrame:self.view.bounds style:UITableViewStylePlain];
    self.tableView.delegate = self;        // ตั้งค่า delegate
    self.tableView.dataSource = self;      // ตั้งค่า dataSource
    [self.view addSubview:self.tableView];
}

#pragma mark - UITableViewDataSource

- (NSInteger)tableView:(UITableView *)tableView numberOfRowsInSection:(NSInteger)section {
    return self.items.count;
}

- (UITableViewCell *)tableView:(UITableView *)tableView cellForRowAtIndexPath:(NSIndexPath *)indexPath {
    UITableViewCell *cell = [tableView dequeueReusableCellWithIdentifier:@"Cell"];
    if (!cell) {
        cell = [[UITableViewCell alloc] initWithStyle:UITableViewCellStyleDefault
                                      reuseIdentifier:@"Cell"];
    }
    cell.textLabel.text = self.items[indexPath.row];
    return cell;
}

#pragma mark - UITableViewDelegate

- (void)tableView:(UITableView *)tableView didSelectRowAtIndexPath:(NSIndexPath *)indexPath {
    NSString *item = self.items[indexPath.row];
    NSLog(@"Selected: %@", item);
    [tableView deselectRowAtIndexPath:indexPath animated:YES];
}

- (CGFloat)tableView:(UITableView *)tableView heightForRowAtIndexPath:(NSIndexPath *)indexPath {
    return 60.0;
}

- (void)tableView:(UITableView *)tableView willDisplayCell:(UITableViewCell *)cell forRowAtIndexPath:(NSIndexPath *)indexPath {
    // เรียกก่อนแสดง cell
    cell.backgroundColor = indexPath.row % 2 == 0 ? [UIColor whiteColor] : [UIColor systemGray6Color];
}

- (BOOL)tableView:(UITableView *)tableView canEditRowAtIndexPath:(NSIndexPath *)indexPath {
    return YES;
}

- (void)tableView:(UITableView *)tableView commitEditingStyle:(UITableViewCellEditingStyle)editingStyle forRowAtIndexPath:(NSIndexPath *)indexPath {
    if (editingStyle == UITableViewCellEditingStyleDelete) {
        NSMutableArray *mutableItems = [self.items mutableCopy];
        [mutableItems removeObjectAtIndex:indexPath.row];
        self.items = [mutableItems copy];
        [tableView deleteRowsAtIndexPaths:@[indexPath] withRowAnimation:UITableViewRowAnimationFade];
    }
}

@end
```

---

## 37.5 Multiple Delegate Patterns

บางครั้ง object อาจต้องมี delegate หลายตัว หรือ delegate สำหรับวัตถุประสงค์ต่างกัน:

### Pattern 1: Separate Protocols

```objc
// DataManagerDataDelegate.h
@class DataManager;

@protocol DataManagerDataDelegate <NSObject>
@required
- (void)dataManager:(DataManager *)manager didLoadData:(NSArray *)data;
- (void)dataManager:(DataManager *)manager didFailWithError:(NSError *)error;
@end

// DataManagerProgressDelegate.h
@class DataManager;

@protocol DataManagerProgressDelegate <NSObject>
@optional
- (void)dataManager:(DataManager *)manager didUpdateProgress:(float)progress;
- (void)dataManagerDidStartLoading:(DataManager *)manager;
- (void)dataManagerDidFinishLoading:(DataManager *)manager;
@end

// DataManager.h
@interface DataManager : NSObject

@property (nonatomic, weak) id<DataManagerDataDelegate> dataDelegate;
@property (nonatomic, weak) id<DataManagerProgressDelegate> progressDelegate;

- (void)loadDataFromURL:(NSURL *)url;

@end

// DataManager.m
@implementation DataManager

- (void)loadDataFromURL:(NSURL *)url {
    // แจ้ง progress delegate
    if ([self.progressDelegate respondsToSelector:@selector(dataManagerDidStartLoading:)]) {
        [self.progressDelegate dataManagerDidStartLoading:self];
    }
    
    // Simulate loading
    dispatch_async(dispatch_get_global_queue(DISPATCH_QUEUE_PRIORITY_DEFAULT, 0), ^{
        // Simulate progress
        for (int i = 1; i <= 10; i++) {
            dispatch_async(dispatch_get_main_queue(), ^{
                float progress = i / 10.0f;
                if ([self.progressDelegate respondsToSelector:@selector(dataManager:didUpdateProgress:)]) {
                    [self.progressDelegate dataManager:self didUpdateProgress:progress];
                }
            });
            [NSThread sleepForTimeInterval:0.1];
        }
        
        // Simulate result
        NSArray *mockData = @[@"Item 1", @"Item 2", @"Item 3"];
        
        dispatch_async(dispatch_get_main_queue(), ^{
            if ([self.progressDelegate respondsToSelector:@selector(dataManagerDidFinishLoading:)]) {
                [self.progressDelegate dataManagerDidFinishLoading:self];
            }
            
            if ([self.dataDelegate respondsToSelector:@selector(dataManager:didLoadData:)]) {
                [self.dataDelegate dataManager:self didLoadData:mockData];
            }
        });
    });
}

@end
```

### Pattern 2: Multiple Observers (Notification-like)

```objc
// EventBroadcaster.h
@protocol EventObserver <NSObject>
- (void)eventOccurred:(NSString *)eventName userInfo:(NSDictionary *)userInfo;
@end

@interface EventBroadcaster : NSObject

- (void)addObserver:(id<EventObserver>)observer;
- (void)removeObserver:(id<EventObserver>)observer;
- (void)broadcastEvent:(NSString *)eventName userInfo:(NSDictionary *)userInfo;

@end

// EventBroadcaster.m
@interface EventBroadcaster ()
@property (nonatomic, strong) NSMutableArray *observers;
@end

@implementation EventBroadcaster

- (instancetype)init {
    self = [super init];
    if (self) {
        _observers = [NSMutableArray array];
    }
    return self;
}

- (void)addObserver:(id<EventObserver>)observer {
    if (![self.observers containsObject:observer]) {
        [self.observers addObject:observer];
    }
}

- (void)removeObserver:(id<EventObserver>)observer {
    [self.observers removeObject:observer];
}

- (void)broadcastEvent:(NSString *)eventName userInfo:(NSDictionary *)userInfo {
    for (id<EventObserver> observer in [self.observers copy]) {
        if ([observer respondsToSelector:@selector(eventOccurred:userInfo:)]) {
            [observer eventOccurred:eventName userInfo:userInfo];
        }
    }
}

@end
```

---

## 37.6 Weak Delegate References

### ทำไมต้องใช้ `weak`?

การใช้ `weak` reference สำหรับ delegate เป็นสิ่งสำคัญมากเพื่อป้องกัน **Retain Cycle**:

```
ปัญหา Retain Cycle:
┌─────────────────┐         ┌──────────────────────┐
│   ViewController │         │    CustomButton       │
│                  │◀────────│  delegate (strong)    │
│                  │─────────▶                       │
│  button (strong) │         │                       │
└─────────────────┘         └──────────────────────┘
   ทั้งสองต่างถือ reference กันและกัน → ไม่มีใครถูก deallocate!
```

```objc
// ❌ อย่าทำแบบนี้ - จะเกิด retain cycle!
@interface CustomButton : UIView
@property (nonatomic, strong) id<CustomButtonDelegate> delegate;  // strong!
@end

// ✅ ทำแบบนี้ถูกต้อง - ใช้ weak
@interface CustomButton : UIView
@property (nonatomic, weak) id<CustomButtonDelegate> delegate;  // weak!
@end
```

### ตัวอย่างปัญหาที่ชัดเจน

```objc
@implementation ViewController

- (void)viewDidLoad {
    [super viewDidLoad];
    
    CustomButton *button = [[CustomButton alloc] initWithTitle:@"Test"];
    
    // ✅ ถูกต้อง: ViewController ถือ button (strong)
    // button ถือ delegate (weak) ชี้ไป ViewController
    // เมื่อ ViewController deallocate, button's delegate = nil อัตโนมัติ
    button.delegate = self;  // self คือ ViewController
    
    [self.view addSubview:button];
}

- (void)dealloc {
    NSLog(@"ViewController deallocated!");
    // ถ้าใช้ weak delegate, delegate ใน button จะเป็น nil อัตโนมัติ
}

@end
```

### การตรวจสอบ weak delegate

```objc
// ในฝั่ง delegator
- (void)doSomethingAndNotifyDelegate {
    // ตรวจสอบว่า delegate ยังมีอยู่ (ไม่ถูก deallocate)
    id<MyDelegate> strongDelegate = self.delegate;  // capture เป็น strong ชั่วคราว
    
    if (strongDelegate != nil) {
        if ([strongDelegate respondsToSelector:@selector(someMethod)]) {
            [strongDelegate someMethod];
        }
    }
}
```

---

## 37.7 Optional Delegate Methods

### การตรวจสอบ Optional Methods

```objc
// ก่อนเรียก optional method ต้องตรวจสอบก่อนเสมอ
- (void)notifyDelegate {
    // respondsToSelector: ตรวจสอบว่า object มี method นี้หรือไม่
    if ([self.delegate respondsToSelector:@selector(optionalMethod:)]) {
        [self.delegate optionalMethod:self];
    }
}
```

### ตัวอย่างที่ครบถ้วน

```objc
// NetworkManagerDelegate.h
@class NetworkManager;

@protocol NetworkManagerDelegate <NSObject>

// Required
@required
- (void)networkManager:(NetworkManager *)manager didReceiveData:(NSData *)data;
- (void)networkManager:(NetworkManager *)manager didFailWithError:(NSError *)error;

// Optional
@optional
- (void)networkManager:(NetworkManager *)manager didUpdateProgress:(double)progress;
- (void)networkManagerWillStartRequest:(NetworkManager *)manager;
- (void)networkManagerDidFinishRequest:(NetworkManager *)manager;
- (BOOL)networkManager:(NetworkManager *)manager shouldRetryForError:(NSError *)error;
- (NSURLRequest *)networkManager:(NetworkManager *)manager willSendRequest:(NSURLRequest *)request;

@end

// NetworkManager.m
@implementation NetworkManager

- (void)startRequest:(NSURLRequest *)request {
    // Optional: แจ้ง delegate ก่อนเริ่ม
    if ([self.delegate respondsToSelector:@selector(networkManagerWillStartRequest:)]) {
        [self.delegate networkManagerWillStartRequest:self];
    }
    
    // Optional: อนุญาตให้ delegate modify request
    NSURLRequest *finalRequest = request;
    if ([self.delegate respondsToSelector:@selector(networkManager:willSendRequest:)]) {
        finalRequest = [self.delegate networkManager:self willSendRequest:request];
    }
    
    // ทำการ request จริงๆ
    NSURLSessionDataTask *task = [[NSURLSession sharedSession] dataTaskWithRequest:finalRequest
                                  completionHandler:^(NSData *data, NSURLResponse *response, NSError *error) {
        dispatch_async(dispatch_get_main_queue(), ^{
            if (error) {
                // Optional: ถาม delegate ว่าควร retry หรือไม่
                BOOL shouldRetry = NO;
                if ([self.delegate respondsToSelector:@selector(networkManager:shouldRetryForError:)]) {
                    shouldRetry = [self.delegate networkManager:self shouldRetryForError:error];
                }
                
                if (!shouldRetry) {
                    // Required: แจ้ง error
                    [self.delegate networkManager:self didFailWithError:error];
                }
            } else {
                // Required: ส่งข้อมูลกลับ
                [self.delegate networkManager:self didReceiveData:data];
                
                // Optional: แจ้งว่าเสร็จแล้ว
                if ([self.delegate respondsToSelector:@selector(networkManagerDidFinishRequest:)]) {
                    [self.delegate networkManagerDidFinishRequest:self];
                }
            }
        });
    }];
    
    [task resume];
}

@end
```

---

## 37.8 Delegate vs Block Callback เปรียบเทียบ

### Delegate Approach

```objc
// Protocol definition
@protocol DownloadManagerDelegate <NSObject>
@required
- (void)downloadManager:(DownloadManager *)manager didFinishDownloading:(NSData *)data;
- (void)downloadManager:(DownloadManager *)manager didFailWithError:(NSError *)error;
@optional
- (void)downloadManager:(DownloadManager *)manager didUpdateProgress:(float)progress;
@end

// Usage
@interface ViewController () <DownloadManagerDelegate>
@property (nonatomic, strong) DownloadManager *downloadManager;
@end

@implementation ViewController

- (void)startDownload {
    self.downloadManager = [[DownloadManager alloc] init];
    self.downloadManager.delegate = self;
    [self.downloadManager downloadURL:[NSURL URLWithString:@"https://example.com/file"]];
}

- (void)downloadManager:(DownloadManager *)manager didFinishDownloading:(NSData *)data {
    NSLog(@"Download complete: %lu bytes", (unsigned long)data.length);
}

- (void)downloadManager:(DownloadManager *)manager didFailWithError:(NSError *)error {
    NSLog(@"Download failed: %@", error.localizedDescription);
}

@end
```

### Block Callback Approach

```objc
// Block-based API
@interface DownloadManagerWithBlocks : NSObject

@property (nonatomic, copy) void(^progressBlock)(float progress);
@property (nonatomic, copy) void(^completionBlock)(NSData *data, NSError *error);

- (void)downloadURL:(NSURL *)url
           progress:(void(^)(float progress))progressBlock
         completion:(void(^)(NSData *data, NSError *error))completionBlock;

@end

// Usage
- (void)startDownloadWithBlocks {
    DownloadManagerWithBlocks *manager = [[DownloadManagerWithBlocks alloc] init];
    
    [manager downloadURL:[NSURL URLWithString:@"https://example.com/file"]
                progress:^(float progress) {
        NSLog(@"Progress: %.0f%%", progress * 100);
    }
              completion:^(NSData *data, NSError *error) {
        if (error) {
            NSLog(@"Error: %@", error.localizedDescription);
        } else {
            NSLog(@"Complete: %lu bytes", (unsigned long)data.length);
        }
    }];
}
```

### ตารางเปรียบเทียบ

| ด้าน | Delegate | Block |
|------|----------|-------|
| Readability | ชัดเจนสำหรับ multiple callbacks | ดีสำหรับ 1-2 callbacks |
| Reusability | Protocol สามารถนำไปใช้ใหม่ได้ | แต่ละ block เป็น instance |
| Multiple methods | จัดการได้ดีมาก | อาจยาวและซับซ้อน |
| Memory | ต้องระวัง retain cycle ผ่าน weak | ต้องระวัง strong capture |
| Testing | Mock delegate ทำได้ง่าย | Mock ทำได้แต่ยากกว่า |
| Single callback | อาจ verbose เกินไป | กระชับและสะดวก |
| Customization | Delegate อาจ customize behavior ได้ | จำกัดกว่า |

### เมื่อใดควรใช้ Delegate vs Block

```objc
// ✅ ใช้ Delegate เมื่อ:
// 1. มี callbacks หลายตัวที่เกี่ยวข้องกัน
// 2. Delegate ต้องการข้อมูลจาก delegator (two-way communication)
// 3. Protocol สามารถนำไปใช้ใหม่ได้หลายที่
// 4. เป็น behavior ที่กำหนดได้ (customizable behavior)

// ✅ ใช้ Block เมื่อ:
// 1. Operation เดียวที่มี completion callback
// 2. Callback ต้องการ capture local variables
// 3. One-off operations (เช่น animation completion)
// 4. API ที่ใช้งานง่ายและกระชับ
```

---

## 37.9 เมื่อใดควรใช้ Delegates

### Use Cases ที่เหมาะสม

```objc
// 1. UI Component customization
@protocol TableViewCellDelegate <NSObject>
- (void)cell:(UITableViewCell *)cell didTapDeleteButton:(NSIndexPath *)indexPath;
- (void)cell:(UITableViewCell *)cell didToggleSwitch:(BOOL)isOn;
@end

// 2. Navigation and flow control
@protocol FormViewControllerDelegate <NSObject>
- (void)formViewController:(FormViewController *)controller didSubmitForm:(NSDictionary *)formData;
- (void)formViewControllerDidCancel:(FormViewController *)controller;
@end

// 3. Data source pattern
@protocol ChartDataSource <NSObject>
@required
- (NSInteger)numberOfDataPointsInChart:(Chart *)chart;
- (CGFloat)chart:(Chart *)chart valueForDataPointAtIndex:(NSInteger)index;
@optional
- (NSString *)chart:(Chart *)chart labelForDataPointAtIndex:(NSInteger)index;
- (UIColor *)chart:(Chart *)chart colorForDataPointAtIndex:(NSInteger)index;
@end

// 4. Service callbacks
@protocol LocationManagerDelegate <NSObject>
@required
- (void)locationManager:(LocationManager *)manager didUpdateLocation:(CLLocation *)location;
- (void)locationManager:(LocationManager *)manager didFailWithError:(NSError *)error;
@optional
- (void)locationManager:(LocationManager *)manager didChangeAuthorizationStatus:(CLAuthorizationStatus)status;
@end
```

---

## 37.10 Building: NetworkManager with Delegate

### ตัวอย่างที่สมบูรณ์: NetworkManager

```objc
// NetworkManagerDelegate.h
#import <Foundation/Foundation.h>

@class NetworkManager;

typedef NS_ENUM(NSInteger, NetworkRequestType) {
    NetworkRequestTypeGET,
    NetworkRequestTypePOST,
    NetworkRequestTypePUT,
    NetworkRequestTypeDELETE
};

@protocol NetworkManagerDelegate <NSObject>

@required
- (void)networkManager:(NetworkManager *)manager
    didCompleteRequest:(NSURLRequest *)request
          responseData:(NSData *)data
          statusCode:(NSInteger)statusCode;

- (void)networkManager:(NetworkManager *)manager
    didFailRequest:(NSURLRequest *)request
         withError:(NSError *)error;

@optional
- (void)networkManager:(NetworkManager *)manager
    didBeginRequest:(NSURLRequest *)request;

- (void)networkManager:(NetworkManager *)manager
    didReceiveHeaders:(NSDictionary *)headers
           forRequest:(NSURLRequest *)request;

- (BOOL)networkManager:(NetworkManager *)manager
    shouldFollowRedirect:(NSURLRequest *)request
         toNewRequest:(NSURLRequest *)newRequest;

- (void)networkManager:(NetworkManager *)manager
    didUpdateDownloadProgress:(double)progress
               forRequest:(NSURLRequest *)request;

@end

// NetworkManager.h
#import <Foundation/Foundation.h>
#import "NetworkManagerDelegate.h"

@interface NetworkManager : NSObject

@property (nonatomic, weak) id<NetworkManagerDelegate> delegate;
@property (nonatomic, assign) NSTimeInterval timeoutInterval;
@property (nonatomic, strong) NSDictionary *defaultHeaders;

+ (instancetype)sharedManager;

- (void)performRequest:(NSURLRequest *)request;
- (void)performGETRequestToURL:(NSURL *)url parameters:(NSDictionary *)params;
- (void)performPOSTRequestToURL:(NSURL *)url body:(NSDictionary *)body;
- (void)cancelAllRequests;

@end

// NetworkManager.m
#import "NetworkManager.h"

@interface NetworkManager () <NSURLSessionDelegate, NSURLSessionDataDelegate>
@property (nonatomic, strong) NSURLSession *session;
@property (nonatomic, strong) NSMutableSet *activeTasks;
@end

@implementation NetworkManager

+ (instancetype)sharedManager {
    static NetworkManager *instance = nil;
    static dispatch_once_t onceToken;
    dispatch_once(&onceToken, ^{
        instance = [[NetworkManager alloc] init];
    });
    return instance;
}

- (instancetype)init {
    self = [super init];
    if (self) {
        NSURLSessionConfiguration *config = [NSURLSessionConfiguration defaultSessionConfiguration];
        _session = [NSURLSession sessionWithConfiguration:config delegate:self delegateQueue:nil];
        _activeTasks = [NSMutableSet set];
        _timeoutInterval = 30.0;
        _defaultHeaders = @{
            @"Content-Type": @"application/json",
            @"Accept": @"application/json"
        };
    }
    return self;
}

- (void)performGETRequestToURL:(NSURL *)url parameters:(NSDictionary *)params {
    NSURLComponents *components = [NSURLComponents componentsWithURL:url resolvingAgainstBaseURL:NO];
    
    if (params.count > 0) {
        NSMutableArray *queryItems = [NSMutableArray array];
        [params enumerateKeysAndObjectsUsingBlock:^(NSString *key, id value, BOOL *stop) {
            NSURLQueryItem *item = [NSURLQueryItem queryItemWithName:key value:[value description]];
            [queryItems addObject:item];
        }];
        components.queryItems = queryItems;
    }
    
    NSMutableURLRequest *request = [NSMutableURLRequest requestWithURL:components.URL];
    request.HTTPMethod = @"GET";
    request.timeoutInterval = self.timeoutInterval;
    
    [self.defaultHeaders enumerateKeysAndObjectsUsingBlock:^(NSString *key, NSString *value, BOOL *stop) {
        [request setValue:value forHTTPHeaderField:key];
    }];
    
    [self performRequest:request];
}

- (void)performPOSTRequestToURL:(NSURL *)url body:(NSDictionary *)body {
    NSMutableURLRequest *request = [NSMutableURLRequest requestWithURL:url];
    request.HTTPMethod = @"POST";
    request.timeoutInterval = self.timeoutInterval;
    
    [self.defaultHeaders enumerateKeysAndObjectsUsingBlock:^(NSString *key, NSString *value, BOOL *stop) {
        [request setValue:value forHTTPHeaderField:key];
    }];
    
    if (body) {
        NSError *error = nil;
        NSData *bodyData = [NSJSONSerialization dataWithJSONObject:body options:0 error:&error];
        if (!error) {
            request.HTTPBody = bodyData;
        }
    }
    
    [self performRequest:request];
}

- (void)performRequest:(NSURLRequest *)request {
    // แจ้ง delegate ว่าเริ่ม request
    dispatch_async(dispatch_get_main_queue(), ^{
        if ([self.delegate respondsToSelector:@selector(networkManager:didBeginRequest:)]) {
            [self.delegate networkManager:self didBeginRequest:request];
        }
    });
    
    NSURLSessionDataTask *task = [self.session dataTaskWithRequest:request
                                                completionHandler:^(NSData *data, NSURLResponse *response, NSError *error) {
        [self.activeTasks removeObject:self];
        
        dispatch_async(dispatch_get_main_queue(), ^{
            if (error) {
                if ([self.delegate respondsToSelector:@selector(networkManager:didFailRequest:withError:)]) {
                    [self.delegate networkManager:self didFailRequest:request withError:error];
                }
                return;
            }
            
            NSHTTPURLResponse *httpResponse = (NSHTTPURLResponse *)response;
            NSInteger statusCode = httpResponse.statusCode;
            
            // แจ้ง headers ถ้า delegate ต้องการ
            if ([self.delegate respondsToSelector:@selector(networkManager:didReceiveHeaders:forRequest:)]) {
                [self.delegate networkManager:self didReceiveHeaders:httpResponse.allHeaderFields forRequest:request];
            }
            
            if ([self.delegate respondsToSelector:@selector(networkManager:didCompleteRequest:responseData:statusCode:)]) {
                [self.delegate networkManager:self didCompleteRequest:request responseData:data statusCode:statusCode];
            }
        });
    }];
    
    [self.activeTasks addObject:task];
    [task resume];
}

- (void)cancelAllRequests {
    [self.session invalidateAndCancel];
}

@end
```

### การใช้งาน NetworkManager

```objc
// APIClient.h
#import "NetworkManager.h"

@protocol APIClientDelegate <NSObject>
- (void)apiClient:(id)client didFetchUsers:(NSArray *)users;
- (void)apiClient:(id)client didFailWithError:(NSError *)error;
@end

@interface APIClient : NSObject <NetworkManagerDelegate>

@property (nonatomic, weak) id<APIClientDelegate> delegate;
- (void)fetchUsers;

@end

// APIClient.m
@implementation APIClient

- (void)fetchUsers {
    NetworkManager *manager = [NetworkManager sharedManager];
    manager.delegate = self;
    
    NSURL *url = [NSURL URLWithString:@"https://jsonplaceholder.typicode.com/users"];
    [manager performGETRequestToURL:url parameters:nil];
}

#pragma mark - NetworkManagerDelegate

- (void)networkManager:(NetworkManager *)manager
    didCompleteRequest:(NSURLRequest *)request
          responseData:(NSData *)data
            statusCode:(NSInteger)statusCode {
    
    if (statusCode == 200) {
        NSError *error = nil;
        NSArray *users = [NSJSONSerialization JSONObjectWithData:data options:0 error:&error];
        
        if (error) {
            [self.delegate apiClient:self didFailWithError:error];
        } else {
            [self.delegate apiClient:self didFetchUsers:users];
        }
    } else {
        NSError *error = [NSError errorWithDomain:@"APIError"
                                             code:statusCode
                                         userInfo:@{NSLocalizedDescriptionKey: @"HTTP Error"}];
        [self.delegate apiClient:self didFailWithError:error];
    }
}

- (void)networkManager:(NetworkManager *)manager
    didFailRequest:(NSURLRequest *)request
         withError:(NSError *)error {
    [self.delegate apiClient:self didFailWithError:error];
}

@end
```

---

## 37.11 Building: FormViewController with Delegate

```objc
// FormViewControllerDelegate.h
@class FormViewController;

typedef NS_ENUM(NSInteger, FormFieldType) {
    FormFieldTypeText,
    FormFieldTypeEmail,
    FormFieldTypePassword,
    FormFieldTypeNumber
};

@protocol FormViewControllerDelegate <NSObject>

@required
- (void)formViewController:(FormViewController *)controller
              didSubmitForm:(NSDictionary *)formData;
- (void)formViewControllerDidCancel:(FormViewController *)controller;

@optional
- (BOOL)formViewController:(FormViewController *)controller
       shouldSubmitFormData:(NSDictionary *)formData;
- (void)formViewController:(FormViewController *)controller
        didChangeField:(NSString *)fieldName
                 value:(NSString *)value;

@end

// FormViewController.h
#import <UIKit/UIKit.h>
#import "FormViewControllerDelegate.h"

@interface FormViewController : UIViewController

@property (nonatomic, weak) id<FormViewControllerDelegate> delegate;

- (instancetype)initWithFields:(NSArray<NSDictionary *> *)fields;
- (void)addFieldWithName:(NSString *)name placeholder:(NSString *)placeholder type:(FormFieldType)type;

@end

// FormViewController.m
@interface FormViewController () <UITextFieldDelegate>
@property (nonatomic, strong) NSArray *fieldConfigs;
@property (nonatomic, strong) NSMutableDictionary *formData;
@property (nonatomic, strong) NSMutableArray *textFields;
@end

@implementation FormViewController

- (instancetype)initWithFields:(NSArray *)fields {
    self = [super init];
    if (self) {
        _fieldConfigs = [fields copy];
        _formData = [NSMutableDictionary dictionary];
        _textFields = [NSMutableArray array];
    }
    return self;
}

- (void)viewDidLoad {
    [super viewDidLoad];
    [self setupUI];
}

- (void)setupUI {
    self.view.backgroundColor = [UIColor whiteColor];
    
    UIScrollView *scrollView = [[UIScrollView alloc] initWithFrame:self.view.bounds];
    [self.view addSubview:scrollView];
    
    CGFloat y = 20;
    for (NSDictionary *config in self.fieldConfigs) {
        UITextField *textField = [[UITextField alloc] initWithFrame:CGRectMake(20, y, self.view.bounds.size.width - 40, 44)];
        textField.placeholder = config[@"placeholder"];
        textField.borderStyle = UITextBorderStyleRoundedRect;
        textField.tag = [self.fieldConfigs indexOfObject:config];
        textField.delegate = self;
        
        NSString *type = config[@"type"];
        if ([type isEqualToString:@"password"]) {
            textField.secureTextEntry = YES;
        } else if ([type isEqualToString:@"email"]) {
            textField.keyboardType = UIKeyboardTypeEmailAddress;
        } else if ([type isEqualToString:@"number"]) {
            textField.keyboardType = UIKeyboardTypeNumberPad;
        }
        
        [scrollView addSubview:textField];
        [self.textFields addObject:textField];
        y += 60;
    }
    
    // Submit button
    UIButton *submitButton = [UIButton buttonWithType:UIButtonTypeSystem];
    submitButton.frame = CGRectMake(20, y, self.view.bounds.size.width - 40, 44);
    [submitButton setTitle:@"Submit" forState:UIControlStateNormal];
    [submitButton addTarget:self action:@selector(submitTapped) forControlEvents:UIControlEventTouchUpInside];
    [scrollView addSubview:submitButton];
    
    y += 60;
    
    // Cancel button
    UIButton *cancelButton = [UIButton buttonWithType:UIButtonTypeSystem];
    cancelButton.frame = CGRectMake(20, y, self.view.bounds.size.width - 40, 44);
    [cancelButton setTitle:@"Cancel" forState:UIControlStateNormal];
    [cancelButton addTarget:self action:@selector(cancelTapped) forControlEvents:UIControlEventTouchUpInside];
    [scrollView addSubview:cancelButton];
    
    scrollView.contentSize = CGSizeMake(self.view.bounds.size.width, y + 60);
}

- (void)submitTapped {
    // รวบรวมข้อมูลจาก text fields
    NSMutableDictionary *data = [NSMutableDictionary dictionary];
    for (NSInteger i = 0; i < self.textFields.count; i++) {
        UITextField *field = self.textFields[i];
        NSDictionary *config = self.fieldConfigs[i];
        NSString *name = config[@"name"];
        data[name] = field.text ?: @"";
    }
    
    // ถาม delegate ว่าควร submit หรือไม่
    if ([self.delegate respondsToSelector:@selector(formViewController:shouldSubmitFormData:)]) {
        if (![self.delegate formViewController:self shouldSubmitFormData:data]) {
            return;
        }
    }
    
    // แจ้ง delegate ว่า submit แล้ว
    if ([self.delegate respondsToSelector:@selector(formViewController:didSubmitForm:)]) {
        [self.delegate formViewController:self didSubmitForm:data];
    }
}

- (void)cancelTapped {
    if ([self.delegate respondsToSelector:@selector(formViewControllerDidCancel:)]) {
        [self.delegate formViewControllerDidCancel:self];
    }
}

#pragma mark - UITextFieldDelegate

- (BOOL)textField:(UITextField *)textField shouldChangeCharactersInRange:(NSRange)range replacementString:(NSString *)string {
    NSString *newText = [textField.text stringByReplacingCharactersInRange:range withString:string];
    NSDictionary *config = self.fieldConfigs[textField.tag];
    NSString *fieldName = config[@"name"];
    
    if ([self.delegate respondsToSelector:@selector(formViewController:didChangeField:value:)]) {
        [self.delegate formViewController:self didChangeField:fieldName value:newText];
    }
    
    return YES;
}

@end
```

---

## 37.12 Advanced Patterns

### Pattern: Delegate with Default Implementations

```objc
// DefaultableDelegate.h
@protocol DefaultableDelegate <NSObject>
@optional
- (NSString *)titleForAction;
- (UIColor *)colorForAction;
@end

// สร้าง default implementations ผ่าน category
@interface NSObject (DefaultableDelegateDefaults) <DefaultableDelegate>
@end

@implementation NSObject (DefaultableDelegateDefaults)
- (NSString *)titleForAction {
    return @"Action";  // default value
}

- (UIColor *)colorForAction {
    return [UIColor systemBlueColor];  // default color
}
@end
```

### Pattern: Proxy Delegate

```objc
// DelegateProxy.h - Pattern สำหรับ forward delegate calls
@interface DelegateProxy : NSObject

- (void)addDelegate:(id)delegate;
- (void)removeDelegate:(id)delegate;

@end

@implementation DelegateProxy

- (instancetype)init {
    self = [super init];
    if (self) {
        // ใช้ NSHashTable เพื่อเก็บ weak references
        _delegates = [NSHashTable weakObjectsHashTable];
    }
    return self;
}

- (BOOL)respondsToSelector:(SEL)aSelector {
    if ([super respondsToSelector:aSelector]) return YES;
    
    for (id delegate in self.delegates) {
        if ([delegate respondsToSelector:aSelector]) return YES;
    }
    return NO;
}

- (void)forwardInvocation:(NSInvocation *)invocation {
    for (id delegate in self.delegates) {
        if ([delegate respondsToSelector:invocation.selector]) {
            [invocation invokeWithTarget:delegate];
        }
    }
}

- (NSMethodSignature *)methodSignatureForSelector:(SEL)aSelector {
    NSMethodSignature *sig = [super methodSignatureForSelector:aSelector];
    if (sig) return sig;
    
    for (id delegate in self.delegates) {
        sig = [delegate methodSignatureForSelector:aSelector];
        if (sig) return sig;
    }
    return nil;
}

@end
```

---

## แบบฝึกหัด (Practice Exercises)

### Exercise 1: การสร้าง Protocol พื้นฐาน
สร้าง `AlarmDelegate` protocol ที่มี:
- Required: `alarmDidRing:(Alarm *)alarm`
- Optional: `alarmWillRing:(Alarm *)alarm` และ `alarm:(Alarm *)alarm shouldSnoozeDuration:(NSTimeInterval)duration`

### Exercise 2: CustomSlider with Delegate
สร้าง `CustomSlider` class ที่มี:
- `SliderDelegate` protocol
- Required method: `slider:didChangeValue:`
- Optional methods: `sliderDidBeginDragging:`, `sliderDidEndDragging:`, `slider:shouldUpdateForValue:`

### Exercise 3: Shopping Cart Delegate
สร้างระบบตะกร้าสินค้าที่มี:
- `CartManagerDelegate`
- Methods: `cartManager:didAddItem:`, `cartManager:didRemoveItem:`, `cartManager:totalPriceDidChange:`, `cartManagerShouldAllowCheckout:`

### Exercise 4: Animation Delegate
สร้าง `AnimationController` ที่มี delegate สำหรับ:
- `animationDidStart:`, `animationDidStop:`, `animationDidComplete:`
- `animation:shouldRepeat:afterCompletionCount:`

### Exercise 5: Data Validation Delegate
สร้าง `FormValidator` ที่มี:
- `validateField:withValue:` returns validation result
- `validationDidPass:forField:`
- `validationDidFail:forField:withErrors:`

### Exercise 6: Multiple Delegates
แก้ไข `DataManager` ให้มี 3 delegates:
- `dataDelegate`: สำหรับ data events
- `progressDelegate`: สำหรับ progress updates
- `errorDelegate`: สำหรับ error handling

### Exercise 7: Weak Delegate Verification
สร้าง test ที่แสดงให้เห็นว่า:
- Strong delegate ทำให้เกิด retain cycle
- Weak delegate ป้องกัน retain cycle ได้
ใช้ `NSLog` และ `dealloc` เพื่อยืนยัน

### Exercise 8: UICollectionView Custom Delegate
สร้าง custom `GridView` ที่มี delegate คล้าย `UICollectionViewDelegate`:
- `numberOfItemsInGridView:`
- `gridView:cellForItemAtIndex:`
- `gridView:didSelectItemAtIndex:`
- `gridView:sizeForItemAtIndex:`

### Exercise 9: Modal Presentation Delegate
สร้าง `LoginViewController` ที่มี:
- `LoginViewControllerDelegate`
- Methods: `loginViewController:didLoginWithUsername:`, `loginViewControllerDidCancel:`
- Optional: `loginViewController:didFailWithError:`

### Exercise 10: Chain of Responsibility
สร้าง `RequestHandler` chain ที่ delegate ส่งต่อ request ไปยัง handler ถัดไปถ้าตัวเองไม่สามารถจัดการได้:
- `canHandleRequest:`
- `handleRequest:` หรือ `passRequestToNext:`

---

## สรุป

Delegate Pattern เป็นหัวใจสำคัญของ iOS development:

1. **Protocol** กำหนด contract ที่ delegate ต้องปฏิบัติตาม
2. **@required** สำหรับ methods ที่บังคับ, **@optional** สำหรับ methods ที่เสริม
3. **Weak reference** สำคัญมากเพื่อป้องกัน retain cycle
4. **respondsToSelector:** ต้องตรวจสอบก่อนเรียก optional methods
5. Delegates เหมาะสำหรับ two-way communication และ multiple callbacks
6. Blocks เหมาะสำหรับ simple, one-off callbacks

ในบทถัดไปเราจะเรียนรู้เรื่อง File System Operations ซึ่งใช้ NSFileManager ในการจัดการไฟล์และโฟลเดอร์
