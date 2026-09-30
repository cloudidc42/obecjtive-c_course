# ส่วนที่ 17: Class Extensions ใน Objective-C

## บทนำ

Class Extension (บางครั้งเรียกว่า "anonymous category" หรือ "continuation class") เป็นฟีเจอร์ใน Objective-C ที่ช่วยให้เราสามารถประกาศ private interface ของ class ได้ Extension ต่างจาก Category ตรงที่มีชื่อว่าง `()` และต้องอยู่ในไฟล์ .m เดียวกับ class implementation

---

## 17.1 Class Extensions vs Categories

### ความแตกต่างสำคัญ

```
Class Extension:              Category:
@interface Foo ()             @interface Foo (BarCategory)

- ชื่อว่าง ()                - มีชื่อ (CategoryName)
- ใน .m file (private)        - มักอยู่ใน .h file (public)
- เพิ่ม ivars ได้             - ไม่สามารถเพิ่ม ivars
- เพิ่ม private methods       - เพิ่ม public methods
- compile พร้อม class         - load ที่ runtime
- implementation ใน class imp - implementation ใน category imp
- ใช้สำหรับ private interface  - ใช้สำหรับ extend existing class
```

### Syntax เปรียบเทียบ

```objc
// ---- Category ----
// ไฟล์: NSString+Utils.h (header ที่แชร์ได้)
@interface NSString (Utils)
- (NSString *)reversed;
@end

// ไฟล์: NSString+Utils.m
@implementation NSString (Utils)
- (NSString *)reversed {
    // ...
}
@end

// ---- Extension ----
// ไฟล์: MyClass.m (private - ไม่แชร์ใน header)
@interface MyClass ()  // ว่างเปล่า = extension
@property (nonatomic, strong) NSString *privateData;
- (void)privateMethod;
@end

@implementation MyClass
// สามารถ implement privateMethod ได้ที่นี่
- (void)privateMethod {
    // ...
}
@end
```

---

## 17.2 @interface ClassName ()

### การประกาศ Extension

```objc
// ไฟล์: UserManager.h (Public Interface)
@interface UserManager : NSObject

// สิ่งที่ public เห็นได้
@property (nonatomic, readonly) NSArray *users;
@property (nonatomic, readonly) NSUInteger userCount;

- (instancetype)init;
- (void)addUser:(NSDictionary *)userInfo;
- (void)removeUserWithID:(NSString *)userID;
- (NSDictionary *)findUserByID:(NSString *)userID;

@end

// ไฟล์: UserManager.m
// Extension ประกาศก่อน @implementation
@interface UserManager ()

// Private properties
@property (nonatomic, strong) NSMutableArray *mutableUsers;
@property (nonatomic, strong) NSMutableDictionary *userIndex; // สำหรับค้นหาเร็ว
@property (nonatomic, strong) NSDateFormatter *dateFormatter;
@property (nonatomic, assign) BOOL isDirty; // มีการเปลี่ยนแปลงที่ยังไม่ save

// Private methods
- (NSString *)generateUserID;
- (BOOL)validateUserInfo:(NSDictionary *)userInfo;
- (void)rebuildIndex;
- (void)saveChanges;

@end

@implementation UserManager

- (instancetype)init {
    self = [super init];
    if (self) {
        _mutableUsers = [NSMutableArray array];
        _userIndex = [NSMutableDictionary dictionary];
        _dateFormatter = [[NSDateFormatter alloc] init];
        _dateFormatter.dateFormat = @"yyyy-MM-dd HH:mm:ss";
        _isDirty = NO;
    }
    return self;
}

// Public property (readonly public, readwrite private via extension)
- (NSArray *)users {
    return [_mutableUsers copy];
}

- (NSUInteger)userCount {
    return [_mutableUsers count];
}

- (void)addUser:(NSDictionary *)userInfo {
    if (![self validateUserInfo:userInfo]) {
        NSLog(@"Invalid user info");
        return;
    }
    
    NSMutableDictionary *user = [userInfo mutableCopy];
    user[@"id"] = [self generateUserID];
    user[@"createdAt"] = [_dateFormatter stringFromDate:[NSDate date]];
    
    [_mutableUsers addObject:user];
    _userIndex[user[@"id"]] = user;
    _isDirty = YES;
    
    [self saveChanges]; // private method
}

- (void)removeUserWithID:(NSString *)userID {
    NSDictionary *user = [self findUserByID:userID];
    if (user) {
        [_mutableUsers removeObject:user];
        [_userIndex removeObjectForKey:userID];
        _isDirty = YES;
        [self saveChanges];
    }
}

- (NSDictionary *)findUserByID:(NSString *)userID {
    return _userIndex[userID]; // O(1) lookup
}

#pragma mark - Private Methods (from extension)

- (NSString *)generateUserID {
    return [[NSUUID UUID] UUIDString];
}

- (BOOL)validateUserInfo:(NSDictionary *)userInfo {
    return userInfo[@"name"] != nil && userInfo[@"email"] != nil;
}

- (void)rebuildIndex {
    [_userIndex removeAllObjects];
    for (NSDictionary *user in _mutableUsers) {
        if (user[@"id"]) {
            _userIndex[user[@"id"]] = user;
        }
    }
}

- (void)saveChanges {
    if (!_isDirty) return;
    NSLog(@"Saving %lu users...", (unsigned long)[_mutableUsers count]);
    // บันทึกข้อมูล (จำลอง)
    _isDirty = NO;
}

@end
```

---

## 17.3 การเพิ่ม Private Properties ผ่าน Extension

### Private Properties เป็น Implementation Detail

```objc
// ไฟล์: ShoppingCart.h (Public)
@interface ShoppingCart : NSObject

// Public API
@property (nonatomic, readonly) NSArray *items;
@property (nonatomic, readonly) double totalPrice;
@property (nonatomic, readonly) NSInteger itemCount;
@property (nonatomic, copy) NSString *currency;

- (void)addItem:(NSDictionary *)item;
- (void)removeItemAtIndex:(NSInteger)index;
- (void)updateQuantity:(NSInteger)quantity atIndex:(NSInteger)index;
- (void)clear;
- (NSDictionary *)generateReceipt;

@end

// ไฟล์: ShoppingCart.m
@interface ShoppingCart ()

// Private storage
@property (nonatomic, strong) NSMutableArray *mutableItems;
@property (nonatomic, assign) double cachedTotal;
@property (nonatomic, assign) BOOL totalIsDirty;

// Private state
@property (nonatomic, strong) NSString *cartID;
@property (nonatomic, strong) NSDate *createdAt;
@property (nonatomic, strong) NSMutableDictionary *discounts;

// Internal flags
@property (nonatomic, assign) BOOL isProcessing;

@end

@implementation ShoppingCart

- (instancetype)init {
    self = [super init];
    if (self) {
        _mutableItems = [NSMutableArray array];
        _currency = @"THB";
        _cartID = [[NSUUID UUID] UUIDString];
        _createdAt = [NSDate date];
        _discounts = [NSMutableDictionary dictionary];
        _totalIsDirty = YES;
        _cachedTotal = 0;
    }
    return self;
}

// Public readonly - computed from private mutable array
- (NSArray *)items {
    return [_mutableItems copy];
}

- (NSInteger)itemCount {
    return [_mutableItems count];
}

- (double)totalPrice {
    if (_totalIsDirty) {
        [self recalculateTotal]; // private method
    }
    return _cachedTotal;
}

- (void)addItem:(NSDictionary *)item {
    [_mutableItems addObject:[item copy]];
    _totalIsDirty = YES;
}

- (void)removeItemAtIndex:(NSInteger)index {
    if (index >= 0 && index < (NSInteger)[_mutableItems count]) {
        [_mutableItems removeObjectAtIndex:index];
        _totalIsDirty = YES;
    }
}

- (void)updateQuantity:(NSInteger)quantity atIndex:(NSInteger)index {
    if (index >= 0 && index < (NSInteger)[_mutableItems count]) {
        NSMutableDictionary *item = [_mutableItems[index] mutableCopy];
        item[@"quantity"] = @(quantity);
        _mutableItems[index] = item;
        _totalIsDirty = YES;
    }
}

- (void)clear {
    [_mutableItems removeAllObjects];
    [_discounts removeAllObjects];
    _totalIsDirty = YES;
}

- (NSDictionary *)generateReceipt {
    return @{
        @"cartID": _cartID,
        @"items": [_mutableItems copy],
        @"total": @(self.totalPrice),
        @"currency": _currency,
        @"generatedAt": [NSDate date]
    };
}

#pragma mark - Private Methods

- (void)recalculateTotal {
    double total = 0;
    for (NSDictionary *item in _mutableItems) {
        double price = [item[@"price"] doubleValue];
        NSInteger qty = [item[@"quantity"] integerValue] ?: 1;
        total += price * qty;
    }
    
    // Apply discounts
    for (NSString *code in _discounts) {
        double discountPct = [_discounts[code] doubleValue];
        total *= (1.0 - discountPct);
    }
    
    _cachedTotal = total;
    _totalIsDirty = NO;
}

@end
```

---

## 17.4 การเพิ่ม Private Methods

### Private Methods ที่ไม่ต้องการให้ภายนอกเห็น

```objc
// ไฟล์: ImageProcessor.h (Public)
@interface ImageProcessor : NSObject

- (UIImage *)processImage:(UIImage *)image withOptions:(NSDictionary *)options;
- (UIImage *)applyFilter:(NSString *)filterName toImage:(UIImage *)image;
- (NSData *)compressImage:(UIImage *)image quality:(float)quality;

@end

// ไฟล์: ImageProcessor.m
@interface ImageProcessor ()

// Private methods - ไม่ให้ผู้ใช้เรียกโดยตรง
- (UIImage *)applyBrightnessAdjustment:(float)brightness toImage:(UIImage *)image;
- (UIImage *)applyContrastAdjustment:(float)contrast toImage:(UIImage *)image;
- (UIImage *)applySaturationAdjustment:(float)saturation toImage:(UIImage *)image;
- (UIImage *)resizeImage:(UIImage *)image toSize:(CGSize)size;
- (UIImage *)cropImage:(UIImage *)image toRect:(CGRect)rect;
- (CIFilter *)filterNamed:(NSString *)name;
- (UIImage *)renderCIContext:(CIContext *)context image:(CIImage *)image;
- (BOOL)validateImageFormat:(UIImage *)image;

@end

@implementation ImageProcessor

- (UIImage *)processImage:(UIImage *)image withOptions:(NSDictionary *)options {
    if (![self validateImageFormat:image]) {
        NSLog(@"Invalid image format");
        return nil;
    }
    
    UIImage *result = image;
    
    // Apply options in order
    if (options[@"brightness"]) {
        result = [self applyBrightnessAdjustment:[options[@"brightness"] floatValue] 
                                         toImage:result];
    }
    
    if (options[@"contrast"]) {
        result = [self applyContrastAdjustment:[options[@"contrast"] floatValue] 
                                       toImage:result];
    }
    
    if (options[@"resize"]) {
        CGSize size = CGSizeFromString(options[@"resize"]);
        result = [self resizeImage:result toSize:size];
    }
    
    return result;
}

- (UIImage *)applyFilter:(NSString *)filterName toImage:(UIImage *)image {
    CIFilter *filter = [self filterNamed:filterName];
    if (!filter) return image;
    
    CIImage *ciImage = [CIImage imageWithCGImage:image.CGImage];
    [filter setValue:ciImage forKey:kCIInputImageKey];
    
    CIContext *context = [CIContext context];
    return [self renderCIContext:context image:[filter outputImage]];
}

- (NSData *)compressImage:(UIImage *)image quality:(float)quality {
    return UIImageJPEGRepresentation(image, quality);
}

#pragma mark - Private Implementation

- (UIImage *)applyBrightnessAdjustment:(float)brightness toImage:(UIImage *)image {
    CIFilter *filter = [CIFilter filterWithName:@"CIColorControls"];
    CIImage *ciImage = [CIImage imageWithCGImage:image.CGImage];
    [filter setValue:ciImage forKey:kCIInputImageKey];
    [filter setValue:@(brightness) forKey:kCIInputBrightnessKey];
    
    CIContext *context = [CIContext context];
    return [self renderCIContext:context image:[filter outputImage]];
}

- (UIImage *)applyContrastAdjustment:(float)contrast toImage:(UIImage *)image {
    CIFilter *filter = [CIFilter filterWithName:@"CIColorControls"];
    CIImage *ciImage = [CIImage imageWithCGImage:image.CGImage];
    [filter setValue:ciImage forKey:kCIInputImageKey];
    [filter setValue:@(contrast) forKey:kCIInputContrastKey];
    
    CIContext *context = [CIContext context];
    return [self renderCIContext:context image:[filter outputImage]];
}

- (UIImage *)applySaturationAdjustment:(float)saturation toImage:(UIImage *)image {
    CIFilter *filter = [CIFilter filterWithName:@"CIColorControls"];
    CIImage *ciImage = [CIImage imageWithCGImage:image.CGImage];
    [filter setValue:ciImage forKey:kCIInputImageKey];
    [filter setValue:@(saturation) forKey:kCIInputSaturationKey];
    
    CIContext *context = [CIContext context];
    return [self renderCIContext:context image:[filter outputImage]];
}

- (UIImage *)resizeImage:(UIImage *)image toSize:(CGSize)size {
    UIGraphicsBeginImageContextWithOptions(size, NO, 0.0);
    [image drawInRect:CGRectMake(0, 0, size.width, size.height)];
    UIImage *resized = UIGraphicsGetImageFromCurrentImageContext();
    UIGraphicsEndImageContext();
    return resized;
}

- (UIImage *)cropImage:(UIImage *)image toRect:(CGRect)rect {
    CGImageRef cgImage = CGImageCreateWithImageInRect(image.CGImage, rect);
    UIImage *cropped = [UIImage imageWithCGImage:cgImage];
    CGImageRelease(cgImage);
    return cropped;
}

- (CIFilter *)filterNamed:(NSString *)name {
    NSDictionary *filterMap = @{
        @"sepia": @"CISepiaTone",
        @"blur": @"CIGaussianBlur",
        @"sharpen": @"CISharpenLuminance",
        @"vignette": @"CIVignette"
    };
    
    NSString *ciName = filterMap[name.lowercaseString];
    if (!ciName) return nil;
    
    return [CIFilter filterWithName:ciName];
}

- (UIImage *)renderCIContext:(CIContext *)context image:(CIImage *)image {
    CGImageRef cgImage = [context createCGImage:image fromRect:[image extent]];
    UIImage *result = [UIImage imageWithCGImage:cgImage];
    CGImageRelease(cgImage);
    return result;
}

- (BOOL)validateImageFormat:(UIImage *)image {
    return image != nil && image.CGImage != nil;
}

@end
```

---

## 17.5 Redeclaring Public readonly Properties as readwrite

### Pattern ที่สำคัญมาก!

```objc
// ไฟล์: BankAccount.h (Public - readonly)
@interface BankAccount : NSObject

// Public เห็นเป็น readonly
@property (nonatomic, readonly) NSString *accountNumber;
@property (nonatomic, readonly) double balance;
@property (nonatomic, readonly) NSString *ownerName;
@property (nonatomic, readonly) NSDate *openedDate;
@property (nonatomic, readonly) BOOL isActive;
@property (nonatomic, readonly) NSArray *transactions;

- (instancetype)initWithOwner:(NSString *)name;
- (BOOL)deposit:(double)amount;
- (BOOL)withdraw:(double)amount;
- (BOOL)transfer:(double)amount toAccount:(BankAccount *)destination;
- (NSDictionary *)statement;

@end

// ไฟล์: BankAccount.m
@interface BankAccount ()

// Redeclare as readwrite ใน extension (private writeable)
@property (nonatomic, readwrite, copy) NSString *accountNumber;
@property (nonatomic, readwrite, assign) double balance;
@property (nonatomic, readwrite, copy) NSString *ownerName;
@property (nonatomic, readwrite, strong) NSDate *openedDate;
@property (nonatomic, readwrite, assign) BOOL isActive;
@property (nonatomic, readwrite, strong) NSMutableArray *mutableTransactions;

// Additional private properties
@property (nonatomic, strong) NSString *internalNotes;
@property (nonatomic, assign) NSInteger failedTransactionCount;
@property (nonatomic, strong) NSDate *lastActivityDate;

// Private methods
- (void)recordTransaction:(NSDictionary *)transaction;
- (BOOL)checkSufficientFunds:(double)amount;
- (void)updateLastActivity;

@end

@implementation BankAccount

- (instancetype)initWithOwner:(NSString *)name {
    self = [super init];
    if (self) {
        _accountNumber = [NSString stringWithFormat:@"ACC-%@", 
                         [[[NSUUID UUID] UUIDString] substringToIndex:8]];
        _ownerName = [name copy];
        _balance = 0.0;
        _openedDate = [NSDate date];
        _isActive = YES;
        _mutableTransactions = [NSMutableArray array];
        _failedTransactionCount = 0;
    }
    return self;
}

// Public transactions (readonly copy)
- (NSArray *)transactions {
    return [_mutableTransactions copy];
}

- (BOOL)deposit:(double)amount {
    if (!_isActive || amount <= 0) return NO;
    
    _balance += amount; // สามารถ set ได้เพราะ extension redeclare เป็น readwrite
    
    [self recordTransaction:@{
        @"type": @"deposit",
        @"amount": @(amount),
        @"balanceAfter": @(_balance),
        @"date": [NSDate date]
    }];
    
    [self updateLastActivity];
    return YES;
}

- (BOOL)withdraw:(double)amount {
    if (!_isActive || amount <= 0) return NO;
    
    if (![self checkSufficientFunds:amount]) {
        _failedTransactionCount++;
        NSLog(@"Insufficient funds. Balance: %.2f, Requested: %.2f", _balance, amount);
        return NO;
    }
    
    _balance -= amount;
    
    [self recordTransaction:@{
        @"type": @"withdrawal",
        @"amount": @(amount),
        @"balanceAfter": @(_balance),
        @"date": [NSDate date]
    }];
    
    [self updateLastActivity];
    return YES;
}

- (BOOL)transfer:(double)amount toAccount:(BankAccount *)destination {
    if (![self withdraw:amount]) return NO;
    return [destination deposit:amount];
}

- (NSDictionary *)statement {
    return @{
        @"accountNumber": _accountNumber,
        @"owner": _ownerName,
        @"balance": @(_balance),
        @"transactions": [_mutableTransactions copy],
        @"openedDate": _openedDate,
        @"lastActivity": _lastActivityDate ?: [NSNull null]
    };
}

#pragma mark - Private Methods

- (void)recordTransaction:(NSDictionary *)transaction {
    [_mutableTransactions addObject:transaction];
}

- (BOOL)checkSufficientFunds:(double)amount {
    return _balance >= amount;
}

- (void)updateLastActivity {
    _lastActivityDate = [NSDate date]; // private readwrite
}

@end

// การใช้งาน
int main(int argc, const char * argv[]) {
    @autoreleasepool {
        BankAccount *account = [[BankAccount alloc] initWithOwner:@"John Doe"];
        
        // Public readonly - ดูได้แต่ set ไม่ได้
        NSLog(@"Account: %@", account.accountNumber);
        NSLog(@"Owner: %@", account.ownerName);
        NSLog(@"Balance: %.2f", account.balance);
        
        // account.balance = 1000000; // ❌ ERROR: readonly property!
        
        // ใช้ public methods แทน
        [account deposit:50000.0];
        [account withdraw:15000.0];
        
        NSLog(@"Balance after transactions: %.2f", account.balance);
        NSLog(@"Transaction count: %lu", (unsigned long)[account.transactions count]);
    }
    return 0;
}
```

---

## 17.6 Extensions ใน .m Files

### Best Practice: Extension อยู่ใน .m เสมอ

```objc
// ==========================================
// ProductManager.h - Public Interface
// ==========================================
#import <Foundation/Foundation.h>

@class Product;

@interface ProductManager : NSObject

+ (instancetype)sharedManager;

// Public methods only
- (void)fetchProducts:(void(^)(NSArray *products, NSError *error))completion;
- (void)saveProduct:(Product *)product completion:(void(^)(BOOL success))completion;
- (Product *)productWithID:(NSString *)productID;
- (NSArray *)searchProducts:(NSString *)query;

@end

// ==========================================
// ProductManager.m - Implementation + Extension
// ==========================================
#import "ProductManager.h"
#import "Product.h"

// Extension ใน .m file - เป็น private
@interface ProductManager ()

// Private singleton
+ (instancetype)_sharedInstance;

// Private storage
@property (nonatomic, strong) NSMutableDictionary<NSString *, Product *> *productCache;
@property (nonatomic, strong) NSURLSession *urlSession;
@property (nonatomic, strong) NSString *baseURL;
@property (nonatomic, strong) dispatch_queue_t processingQueue;

// Private state
@property (nonatomic, assign) BOOL isFetching;
@property (nonatomic, strong) NSDate *lastFetchDate;
@property (nonatomic, assign) NSInteger requestCount;

// Private methods
- (NSURL *)urlForEndpoint:(NSString *)endpoint;
- (void)processProductData:(NSArray *)data;
- (Product *)parseProduct:(NSDictionary *)dict;
- (void)cacheProduct:(Product *)product;
- (BOOL)isCacheValid;
- (void)clearExpiredCache;

@end

@implementation ProductManager

+ (instancetype)sharedManager {
    static ProductManager *instance = nil;
    static dispatch_once_t token;
    dispatch_once(&token, ^{
        instance = [[self alloc] init];
    });
    return instance;
}

- (instancetype)init {
    self = [super init];
    if (self) {
        _productCache = [NSMutableDictionary dictionary];
        _urlSession = [NSURLSession sharedSession];
        _baseURL = @"https://api.example.com/v1";
        _processingQueue = dispatch_queue_create("com.app.products", DISPATCH_QUEUE_SERIAL);
        _isFetching = NO;
        _requestCount = 0;
    }
    return self;
}

- (void)fetchProducts:(void(^)(NSArray *products, NSError *error))completion {
    if (_isFetching) {
        NSLog(@"Already fetching, skipping...");
        return;
    }
    
    if ([self isCacheValid]) {
        NSLog(@"Using cached data");
        NSArray *products = [_productCache allValues];
        if (completion) completion(products, nil);
        return;
    }
    
    _isFetching = YES;
    _requestCount++;
    
    NSURL *url = [self urlForEndpoint:@"/products"];
    
    [[_urlSession dataTaskWithURL:url 
                completionHandler:^(NSData *data, NSURLResponse *response, NSError *error) {
        
        self->_isFetching = NO;
        
        if (error) {
            dispatch_async(dispatch_get_main_queue(), ^{
                if (completion) completion(nil, error);
            });
            return;
        }
        
        dispatch_async(self->_processingQueue, ^{
            NSArray *jsonArray = [NSJSONSerialization JSONObjectWithData:data 
                                                                options:0 
                                                                  error:nil];
            [self processProductData:jsonArray];
            
            dispatch_async(dispatch_get_main_queue(), ^{
                NSArray *products = [self->_productCache allValues];
                if (completion) completion(products, nil);
            });
        });
    }] resume];
}

- (void)saveProduct:(Product *)product completion:(void(^)(BOOL success))completion {
    [self cacheProduct:product]; // private
    // ส่งไปยัง server...
    if (completion) completion(YES);
}

- (Product *)productWithID:(NSString *)productID {
    return _productCache[productID];
}

- (NSArray *)searchProducts:(NSString *)query {
    NSPredicate *pred = [NSPredicate predicateWithFormat:@"name CONTAINS[cd] %@", query];
    return [[_productCache allValues] filteredArrayUsingPredicate:pred];
}

#pragma mark - Private Methods (from extension)

- (NSURL *)urlForEndpoint:(NSString *)endpoint {
    return [NSURL URLWithString:[_baseURL stringByAppendingString:endpoint]];
}

- (void)processProductData:(NSArray *)data {
    for (NSDictionary *dict in data) {
        Product *product = [self parseProduct:dict];
        if (product) {
            [self cacheProduct:product];
        }
    }
    _lastFetchDate = [NSDate date];
}

- (Product *)parseProduct:(NSDictionary *)dict {
    // สร้าง Product จาก dictionary
    // (implementation จะขึ้นอยู่กับ Product class)
    return nil; // placeholder
}

- (void)cacheProduct:(Product *)product {
    if (product.productID) {
        _productCache[product.productID] = product;
    }
}

- (BOOL)isCacheValid {
    if (!_lastFetchDate) return NO;
    
    NSTimeInterval elapsed = [[NSDate date] timeIntervalSinceDate:_lastFetchDate];
    return elapsed < 300; // cache valid for 5 minutes
}

- (void)clearExpiredCache {
    _lastFetchDate = nil;
    [_productCache removeAllObjects];
}

@end
```

---

## 17.7 ตารางเปรียบเทียบ Extensions vs Categories

```
╔══════════════════════════╦═══════════════════════╦═══════════════════════╗
║ Feature                  ║ Class Extension       ║ Category              ║
╠══════════════════════════╬═══════════════════════╬═══════════════════════╣
║ Syntax                   ║ @interface Foo ()     ║ @interface Foo (Bar)  ║
║ Location                 ║ .m file               ║ .h + .m files         ║
║ Visibility               ║ Private               ║ Public                ║
║ Add ivars/properties     ║ YES                   ║ NO (need assoc obj)   ║
║ Add required @property   ║ YES                   ║ NO                    ║
║ Override superclass      ║ YES (via super)       ║ YES (dangerous)       ║
║ Compiled with class      ║ YES                   ║ NO (runtime)          ║
║ Purpose                  ║ Private interface     ║ Extend existing class ║
║ Multiple definitions     ║ Only one              ║ Multiple allowed      ║
║ Import needed            ║ NO (same file)        ║ YES                   ║
║ Apply to any class       ║ Only your own classes ║ Any class             ║
╚══════════════════════════╩═══════════════════════╩═══════════════════════╝
```

---

## 17.8 Pattern: Public Interface ใน .h, Private Extension ใน .m

### Real-World Example: Authentication Manager

```objc
// ==========================================
// AuthManager.h - เฉพาะ Public API
// ==========================================

typedef NS_ENUM(NSInteger, AuthState) {
    AuthStateLoggedOut,
    AuthStateLoggingIn,
    AuthStateLoggedIn,
    AuthStateRefreshingToken
};

@class AuthManager;

@protocol AuthManagerDelegate <NSObject>
@optional
- (void)authManager:(AuthManager *)manager didChangeState:(AuthState)state;
- (void)authManagerDidLogin:(AuthManager *)manager;
- (void)authManagerDidLogout:(AuthManager *)manager;
- (void)authManager:(AuthManager *)manager didFailWithError:(NSError *)error;
@end

@interface AuthManager : NSObject

+ (instancetype)sharedManager;

@property (nonatomic, readonly) AuthState currentState;
@property (nonatomic, readonly) BOOL isLoggedIn;
@property (nonatomic, readonly, nullable) NSString *currentUserID;
@property (nonatomic, weak, nullable) id<AuthManagerDelegate> delegate;

- (void)loginWithEmail:(NSString *)email 
              password:(NSString *)password
            completion:(void(^)(BOOL success, NSError * _Nullable error))completion;

- (void)logout;
- (void)refreshTokenIfNeeded;

@end

// ==========================================
// AuthManager.m - Implementation
// ==========================================

// Private constants (not exposed in header)
static NSString * const kAuthTokenKey = @"auth_token";
static NSString * const kUserIDKey = @"user_id";
static NSString * const kTokenExpiryKey = @"token_expiry";
static const NSTimeInterval kTokenRefreshThreshold = 300; // 5 minutes

// Private extension - internal state and helpers
@interface AuthManager ()

// Private state - readonly externally but writable here
@property (nonatomic, readwrite, assign) AuthState currentState;
@property (nonatomic, readwrite, copy, nullable) NSString *currentUserID;

// Internal data
@property (nonatomic, strong, nullable) NSString *accessToken;
@property (nonatomic, strong, nullable) NSString *refreshToken;
@property (nonatomic, strong, nullable) NSDate *tokenExpiryDate;
@property (nonatomic, strong) NSUserDefaults *defaults;

// Network
@property (nonatomic, strong) NSURLSession *session;
@property (nonatomic, strong) NSString *apiBaseURL;

// Threading
@property (nonatomic, strong) dispatch_queue_t authQueue;

// Private methods - not in public header
- (void)loadSavedTokens;
- (void)saveTokens;
- (void)clearTokens;
- (BOOL)isTokenValid;
- (BOOL)isTokenExpiringSoon;
- (void)performLogin:(NSDictionary *)credentials 
          completion:(void(^)(NSDictionary *, NSError *))completion;
- (void)handleLoginResponse:(NSDictionary *)response;
- (void)handleLoginError:(NSError *)error;
- (void)setState:(AuthState)state;
- (void)notifyDelegateOfStateChange;

@end

@implementation AuthManager

+ (instancetype)sharedManager {
    static AuthManager *instance = nil;
    static dispatch_once_t onceToken;
    dispatch_once(&onceToken, ^{
        instance = [[self alloc] init];
    });
    return instance;
}

- (instancetype)init {
    self = [super init];
    if (self) {
        _defaults = [NSUserDefaults standardUserDefaults];
        _session = [NSURLSession sharedSession];
        _apiBaseURL = @"https://api.myapp.com";
        _authQueue = dispatch_queue_create("com.myapp.auth", DISPATCH_QUEUE_SERIAL);
        _currentState = AuthStateLoggedOut;
        
        [self loadSavedTokens];
    }
    return self;
}

#pragma mark - Public Methods

- (BOOL)isLoggedIn {
    return _currentState == AuthStateLoggedIn && [self isTokenValid];
}

- (void)loginWithEmail:(NSString *)email 
              password:(NSString *)password
            completion:(void(^)(BOOL success, NSError * _Nullable error))completion {
    
    [self setState:AuthStateLoggingIn];
    
    NSDictionary *credentials = @{
        @"email": email,
        @"password": password
    };
    
    [self performLogin:credentials completion:^(NSDictionary *response, NSError *error) {
        if (error) {
            [self handleLoginError:error];
            if (completion) completion(NO, error);
        } else {
            [self handleLoginResponse:response];
            if (completion) completion(YES, nil);
        }
    }];
}

- (void)logout {
    [self clearTokens];
    [self setState:AuthStateLoggedOut];
    self.currentUserID = nil;
    
    dispatch_async(dispatch_get_main_queue(), ^{
        if ([self->_delegate respondsToSelector:@selector(authManagerDidLogout:)]) {
            [self->_delegate authManagerDidLogout:self];
        }
    });
}

- (void)refreshTokenIfNeeded {
    if ([self isTokenExpiringSoon]) {
        // Refresh token logic
        NSLog(@"Refreshing token...");
    }
}

#pragma mark - Private Methods

- (void)loadSavedTokens {
    _accessToken = [_defaults stringForKey:kAuthTokenKey];
    _currentUserID = [_defaults stringForKey:kUserIDKey];
    
    NSTimeInterval expiry = [_defaults doubleForKey:kTokenExpiryKey];
    if (expiry > 0) {
        _tokenExpiryDate = [NSDate dateWithTimeIntervalSince1970:expiry];
    }
    
    if ([self isTokenValid]) {
        _currentState = AuthStateLoggedIn;
    }
}

- (void)saveTokens {
    if (_accessToken) [_defaults setObject:_accessToken forKey:kAuthTokenKey];
    if (_currentUserID) [_defaults setObject:_currentUserID forKey:kUserIDKey];
    if (_tokenExpiryDate) {
        [_defaults setDouble:[_tokenExpiryDate timeIntervalSince1970] forKey:kTokenExpiryKey];
    }
    [_defaults synchronize];
}

- (void)clearTokens {
    _accessToken = nil;
    _refreshToken = nil;
    _tokenExpiryDate = nil;
    
    [_defaults removeObjectForKey:kAuthTokenKey];
    [_defaults removeObjectForKey:kUserIDKey];
    [_defaults removeObjectForKey:kTokenExpiryKey];
    [_defaults synchronize];
}

- (BOOL)isTokenValid {
    if (!_accessToken || !_tokenExpiryDate) return NO;
    return [_tokenExpiryDate timeIntervalSinceNow] > 0;
}

- (BOOL)isTokenExpiringSoon {
    if (!_tokenExpiryDate) return NO;
    return [_tokenExpiryDate timeIntervalSinceNow] < kTokenRefreshThreshold;
}

- (void)performLogin:(NSDictionary *)credentials 
          completion:(void(^)(NSDictionary *, NSError *))completion {
    // Simulate API call
    dispatch_after(dispatch_time(DISPATCH_TIME_NOW, (int64_t)(0.5 * NSEC_PER_SEC)), 
                   _authQueue, ^{
        // จำลองการ login สำเร็จ
        NSDictionary *mockResponse = @{
            @"access_token": @"eyJhbGciOiJIUzI1NiJ9...",
            @"user_id": @"user123",
            @"expires_in": @3600
        };
        if (completion) completion(mockResponse, nil);
    });
}

- (void)handleLoginResponse:(NSDictionary *)response {
    _accessToken = response[@"access_token"];
    self.currentUserID = response[@"user_id"];
    
    NSTimeInterval expiresIn = [response[@"expires_in"] doubleValue];
    _tokenExpiryDate = [NSDate dateWithTimeIntervalSinceNow:expiresIn];
    
    [self saveTokens];
    [self setState:AuthStateLoggedIn];
    
    dispatch_async(dispatch_get_main_queue(), ^{
        if ([self->_delegate respondsToSelector:@selector(authManagerDidLogin:)]) {
            [self->_delegate authManagerDidLogin:self];
        }
    });
}

- (void)handleLoginError:(NSError *)error {
    [self setState:AuthStateLoggedOut];
    
    dispatch_async(dispatch_get_main_queue(), ^{
        if ([self->_delegate respondsToSelector:@selector(authManager:didFailWithError:)]) {
            [self->_delegate authManager:self didFailWithError:error];
        }
    });
}

- (void)setState:(AuthState)state {
    if (_currentState == state) return;
    
    _currentState = state;
    [self notifyDelegateOfStateChange];
}

- (void)notifyDelegateOfStateChange {
    dispatch_async(dispatch_get_main_queue(), ^{
        if ([self->_delegate respondsToSelector:@selector(authManager:didChangeState:)]) {
            [self->_delegate authManager:self didChangeState:self->_currentState];
        }
    });
}

@end
```

---

## 17.9 Extension กับ Protocol Conformance

### ประกาศ Protocol Conformance ใน Extension (Private)

```objc
// ไฟล์: NetworkViewController.h
@interface NetworkViewController : UIViewController
// ไม่มีการกล่าวถึง NSURLSessionDelegate ใน public header
@end

// ไฟล์: NetworkViewController.m
// ประกาศ protocol conformance ใน extension เพื่อซ่อนรายละเอียด
@interface NetworkViewController () <NSURLSessionDelegate, NSURLSessionDataDelegate>

@property (nonatomic, strong) NSURLSession *session;
@property (nonatomic, strong) NSMutableData *receivedData;

@end

@implementation NetworkViewController

- (void)viewDidLoad {
    [super viewDidLoad];
    
    NSURLSessionConfiguration *config = [NSURLSessionConfiguration defaultSessionConfiguration];
    _session = [NSURLSession sessionWithConfiguration:config 
                                             delegate:self 
                                        delegateQueue:nil];
}

#pragma mark - NSURLSessionDataDelegate (Private)

- (void)URLSession:(NSURLSession *)session 
          dataTask:(NSURLSessionDataTask *)dataTask 
    didReceiveData:(NSData *)data {
    [_receivedData appendData:data];
}

- (void)URLSession:(NSURLSession *)session 
              task:(NSURLSessionTask *)task 
didCompleteWithError:(NSError *)error {
    if (error) {
        NSLog(@"Error: %@", error);
    } else {
        // Process _receivedData
    }
}

@end
```

---

## 17.10 Complex Extension Example: Thread-Safe Cache

```objc
// ==========================================
// ThreadSafeCache.h
// ==========================================

@interface ThreadSafeCache : NSObject

- (instancetype)initWithCapacity:(NSUInteger)capacity;

- (void)setObject:(id)object forKey:(NSString *)key;
- (void)setObject:(id)object forKey:(NSString *)key ttl:(NSTimeInterval)ttl;
- (id)objectForKey:(NSString *)key;
- (void)removeObjectForKey:(NSString *)key;
- (void)removeAllObjects;
- (NSUInteger)count;

@end

// ==========================================
// ThreadSafeCache.m
// ==========================================

@interface CacheEntry : NSObject
@property (nonatomic, strong) id value;
@property (nonatomic, strong) NSDate *expiryDate;
@property (nonatomic, strong) NSDate *createdAt;
@property (nonatomic, assign) NSUInteger accessCount;
@end

@implementation CacheEntry
@end

// ---- Extension ----
@interface ThreadSafeCache ()

@property (nonatomic, strong) NSMutableDictionary<NSString *, CacheEntry *> *storage;
@property (nonatomic, assign) NSUInteger capacity;
@property (nonatomic, strong) dispatch_queue_t concurrentQueue;
@property (nonatomic, strong) NSTimer *cleanupTimer;

// Private methods
- (void)evictIfNeeded;
- (void)removeExpiredEntries;
- (NSString *)leastRecentlyUsedKey;
- (BOOL)isEntryExpired:(CacheEntry *)entry;
- (void)startCleanupTimer;
- (void)stopCleanupTimer;

@end

@implementation ThreadSafeCache

- (instancetype)initWithCapacity:(NSUInteger)capacity {
    self = [super init];
    if (self) {
        _capacity = capacity;
        _storage = [NSMutableDictionary dictionaryWithCapacity:capacity];
        _concurrentQueue = dispatch_queue_create("com.app.cache", 
                                                  DISPATCH_QUEUE_CONCURRENT);
        [self startCleanupTimer];
    }
    return self;
}

- (void)dealloc {
    [self stopCleanupTimer];
}

- (void)setObject:(id)object forKey:(NSString *)key {
    [self setObject:object forKey:key ttl:0]; // 0 = ไม่หมดอายุ
}

- (void)setObject:(id)object forKey:(NSString *)key ttl:(NSTimeInterval)ttl {
    dispatch_barrier_async(_concurrentQueue, ^{
        [self evictIfNeeded];
        
        CacheEntry *entry = [[CacheEntry alloc] init];
        entry.value = object;
        entry.createdAt = [NSDate date];
        entry.accessCount = 0;
        
        if (ttl > 0) {
            entry.expiryDate = [NSDate dateWithTimeIntervalSinceNow:ttl];
        }
        
        self->_storage[key] = entry;
    });
}

- (id)objectForKey:(NSString *)key {
    __block id result = nil;
    
    dispatch_sync(_concurrentQueue, ^{
        CacheEntry *entry = self->_storage[key];
        
        if (entry && ![self isEntryExpired:entry]) {
            entry.accessCount++;
            result = entry.value;
        } else if (entry && [self isEntryExpired:entry]) {
            // Entry expired - remove it
            dispatch_barrier_async(self->_concurrentQueue, ^{
                [self->_storage removeObjectForKey:key];
            });
        }
    });
    
    return result;
}

- (void)removeObjectForKey:(NSString *)key {
    dispatch_barrier_async(_concurrentQueue, ^{
        [self->_storage removeObjectForKey:key];
    });
}

- (void)removeAllObjects {
    dispatch_barrier_async(_concurrentQueue, ^{
        [self->_storage removeAllObjects];
    });
}

- (NSUInteger)count {
    __block NSUInteger count = 0;
    dispatch_sync(_concurrentQueue, ^{
        count = [self->_storage count];
    });
    return count;
}

#pragma mark - Private Methods

- (void)evictIfNeeded {
    // ถ้า cache เต็ม ลบ entry ที่ใช้น้อยที่สุด
    if ([_storage count] >= _capacity) {
        NSString *lruKey = [self leastRecentlyUsedKey];
        if (lruKey) {
            [_storage removeObjectForKey:lruKey];
        }
    }
}

- (void)removeExpiredEntries {
    dispatch_barrier_async(_concurrentQueue, ^{
        NSArray *keys = [self->_storage allKeys];
        for (NSString *key in keys) {
            CacheEntry *entry = self->_storage[key];
            if ([self isEntryExpired:entry]) {
                [self->_storage removeObjectForKey:key];
            }
        }
    });
}

- (NSString *)leastRecentlyUsedKey {
    NSString *lruKey = nil;
    NSUInteger minAccess = NSUIntegerMax;
    
    for (NSString *key in _storage) {
        CacheEntry *entry = _storage[key];
        if (entry.accessCount < minAccess) {
            minAccess = entry.accessCount;
            lruKey = key;
        }
    }
    
    return lruKey;
}

- (BOOL)isEntryExpired:(CacheEntry *)entry {
    if (!entry.expiryDate) return NO;
    return [entry.expiryDate timeIntervalSinceNow] < 0;
}

- (void)startCleanupTimer {
    _cleanupTimer = [NSTimer scheduledTimerWithTimeInterval:60.0
                                                    target:self
                                                  selector:@selector(removeExpiredEntries)
                                                  userInfo:nil
                                                   repeats:YES];
}

- (void)stopCleanupTimer {
    [_cleanupTimer invalidate];
    _cleanupTimer = nil;
}

@end
```

---

## แบบฝึกหัด

### ระดับเริ่มต้น

**แบบฝึกหัดที่ 1**: Library Book System
```objc
// สร้าง LibraryBook class
// Public (.h):
//   - title (readonly)
//   - author (readonly)
//   - isAvailable (readonly)
//   - checkout, returnBook methods

// Private extension (.m):
//   - checkedOutDate (private)
//   - borrowerID (private)
//   - overdueFee calculation
//   - validation methods
```

**แบบฝึกหัดที่ 2**: Password Manager
```objc
// PasswordManager class
// Public:
//   - storePassword:forService:username:
//   - passwordForService:username:
//   - deletePasswordForService:username:

// Private Extension:
//   - encryption/decryption methods
//   - keychain interaction
//   - master password verification
```

**แบบฝึกหัดที่ 3**: Score Tracker
```objc
// GameScoreTracker
// Public:
//   - addScore:forPlayer:
//   - highScore (readonly)
//   - rankings (readonly NSArray)

// Private Extension:
//   - sorting logic
//   - persistence methods
//   - duplicate detection
```

### ระดับกลาง

**แบบฝึกหัดที่ 4**: Event Logger
```objc
// EventLogger
// Public:
//   + sharedLogger
//   - logEvent:withData:
//   - logs (readonly)
//   - clearLogs

// Private Extension:
//   - file writing
//   - log rotation
//   - formatting
//   - thread safety (using dispatch_queue)
```

**แบบฝึกหัดที่ 5**: Network Request Builder
```objc
// RequestBuilder
// Public:
//   + GET:
//   + POST:withBody:
//   - addHeader:value:
//   - build (returns NSURLRequest)

// Private Extension:
//   - URL encoding
//   - authentication token injection
//   - validation
//   - default headers
```

**แบบฝึกหัดที่ 6**: Image Cache ที่มี Disk Storage
```objc
// ImageCache
// Public:
//   - cacheImage:forURL:
//   - imageForURL:
//   - clearMemoryCache
//   - clearDiskCache

// Private Extension:
//   - memory cache (NSCache)
//   - disk cache operations
//   - size management
//   - expiry logic
```

### ระดับสูง

**แบบฝึกหัดที่ 7**: Database Abstraction Layer
```objc
// DatabaseManager
// Public:
//   - executeQuery:parameters:
//   - insertRecord:intoTable:
//   - updateRecord:inTable:where:
//   - deleteFromTable:where:
//   - beginTransaction
//   - commitTransaction
//   - rollbackTransaction

// Private Extension:
//   - connection management
//   - statement preparation and caching
//   - error handling and retry
//   - transaction stack
//   - migration support
```

**แบบฝึกหัดที่ 8**: Dependency Injection Container
```objc
// DIContainer
// Public:
//   - register:factory: (register service)
//   - resolve: (get instance)
//   - registerSingleton:factory:

// Private Extension:
//   - registry storage
//   - circular dependency detection
//   - lifecycle management
//   - lazy initialization
```

**แบบฝึกหัดที่ 9**: State Machine
```objc
// StateMachine<S, E> (generic-like with NSObject)
// Public:
//   - currentState (readonly)
//   - canTransitionTo:
//   - transitionTo:withEvent:
//   - addTransition:from:to:

// Private Extension:
//   - transition table
//   - guard conditions
//   - transition actions
//   - history tracking
//   - delegate notification
```

**แบบฝึกหัดที่ 10 (Challenge)**: Observable Property System
```objc
// สร้าง Observable<T> ที่ใช้ Extension สำหรับ internals
// Public:
//   - value (readonly)
//   - observe:block: (subscribe to changes)
//   - removeObserver:

// Private Extension:
//   - observers storage (NSMapTable weak references)
//   - notification dispatch
//   - thread safety

// Combine กับ Property Wrapper pattern:
// @property (nonatomic, strong) Observable<NSString *> *name;
// - เมื่อ name.value เปลี่ยน observers ทุกตัวจะได้รับแจ้ง
```

---

## 17.11 Extension Best Practices สรุป

### Pattern ที่แนะนำ

```objc
// ✅ Good: แยก public/private อย่างชัดเจน

// MyService.h - เฉพาะ public interface
@interface MyService : NSObject

- (void)publicMethod;
@property (readonly) NSString *publicProperty;

@end

// MyService.m - implementation + private extension
@interface MyService ()
- (void)privateHelper;
@property (nonatomic, strong) NSMutableArray *internalData;
@end

@implementation MyService
// ...
@end
```

```objc
// ✅ Good: readonly public, readwrite private
// .h
@property (nonatomic, readonly) NSInteger score;

// .m extension
@property (nonatomic, readwrite, assign) NSInteger score;
```

```objc
// ✅ Good: Protocol conformance ที่เป็น implementation detail
// .h - ไม่บอกว่า implement protocol ใด
@interface MyViewController : UIViewController
@end

// .m - ประกาศ protocol conformance ที่นี่
@interface MyViewController () <UITableViewDataSource, UITableViewDelegate>
@end
```

---

## สรุปบทเรียน

ในบทนี้เราได้เรียนรู้:
- **Class Extensions คืออะไร**: anonymous category ที่ใช้สำหรับ private interface
- **Syntax**: `@interface ClassName ()`
- **Private Properties**: วิธีเพิ่ม private properties ผ่าน extension
- **Private Methods**: วิธีประกาศ private methods
- **Readonly Redeclaration**: pattern ที่สำคัญมากสำหรับ encapsulation
- **Extensions ใน .m**: ทำไมต้องอยู่ใน .m
- **Extensions vs Categories**: ความแตกต่างที่สำคัญ
- **Real-World Patterns**: AuthManager, Cache, Network layer

> **Best Practice**: ใช้ Extension เสมอเพื่อซ่อน implementation details ใน .m file ออกจาก public .h file สิ่งนี้ทำให้ API สะอาด เปลี่ยน internal implementation ได้โดยไม่กระทบผู้ใช้ และป้องกันไม่ให้ code ภายนอกพึ่งพา internal state
