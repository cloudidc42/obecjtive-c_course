# Part 70: In-App Purchase ใน Objective-C

## บทนำ

In-App Purchase (IAP) คือระบบที่ช่วยให้นักพัฒนาสามารถขายสินค้าและบริการดิจิทัลภายในแอปพลิเคชัน iOS ได้ โดยใช้ StoreKit framework ที่ Apple จัดเตรียมไว้ ระบบ IAP รองรับหลายประเภทสินค้า ตั้งแต่การซื้อครั้งเดียวไปจนถึง subscription

ในบทนี้เราจะเรียนรู้:
- StoreKit framework และการตั้งค่า
- ประเภทของ In-App Purchase
- การดึงข้อมูลสินค้า (SKProductsRequest)
- การซื้อสินค้า (SKPaymentQueue)
- การจัดการ Transaction
- Receipt Validation
- การ Restore Purchases
- Subscription Management
- Sandbox Testing

---

## 70.1 StoreKit Framework

### การ Import

```objc
#import <StoreKit/StoreKit.h>
```

### ประเภทของ In-App Purchase

**1. Consumable (สินค้าที่ใช้แล้วหมด)**
- เหรียญในเกม, ชีวิตในเกม, Credits
- ซื้อได้หลายครั้ง
- ไม่สามารถ restore ได้

**2. Non-Consumable (สินค้าที่ซื้อครั้งเดียว)**
- ปลดล็อก premium features, ลบโฆษณา
- ซื้อได้ครั้งเดียว แต่ restore ได้
- ถ้าลบแอปแล้วติดตั้งใหม่ ต้อง restore

**3. Auto-Renewable Subscription**
- Apple Music, Netflix-style
- ต่ออายุอัตโนมัติ จนกว่าจะยกเลิก

**4. Non-Renewing Subscription**
- ใช้ได้ช่วงเวลาหนึ่ง แต่ไม่ต่ออัตโนมัติ

---

## 70.2 การตั้งค่าใน App Store Connect

ก่อนเริ่มเขียนโค้ด ต้องตั้งค่าใน App Store Connect ก่อน:

1. เข้าสู่ App Store Connect
2. เลือกแอปของคุณ
3. ไปที่ "In-App Purchases" หรือ "Subscriptions"
4. สร้าง Product ID (เช่น `com.yourcompany.yourapp.premium`)
5. กำหนดราคาและรายละเอียด

---

## 70.3 การตั้งค่า StoreKit Manager

### IAPManager.h

```objc
#import <Foundation/Foundation.h>
#import <StoreKit/StoreKit.h>

// Notification names
extern NSString * const IAPManagerPurchaseSuccessNotification;
extern NSString * const IAPManagerPurchaseFailedNotification;
extern NSString * const IAPManagerRestoreSuccessNotification;
extern NSString * const IAPManagerRestoreFailedNotification;

// UserInfo keys
extern NSString * const IAPManagerProductIDKey;
extern NSString * const IAPManagerErrorKey;

@interface IAPManager : NSObject <SKProductsRequestDelegate, SKPaymentTransactionObserver>

@property (nonatomic, strong, readonly) NSArray<SKProduct *> *availableProducts;
@property (nonatomic, assign, readonly) BOOL isLoading;

// Singleton
+ (instancetype)sharedManager;

// Products
- (void)fetchProductsWithIdentifiers:(NSSet<NSString *> *)identifiers
                          completion:(void(^)(NSArray<SKProduct *> *products, NSError *error))completion;

// Purchasing
- (BOOL)canMakePurchases;
- (void)purchaseProduct:(SKProduct *)product;
- (void)purchaseProductWithIdentifier:(NSString *)productID;
- (void)restorePurchases;

// Status
- (BOOL)isProductPurchased:(NSString *)productID;

@end
```

### IAPManager.m

```objc
#import "IAPManager.h"

NSString * const IAPManagerPurchaseSuccessNotification = @"IAPManagerPurchaseSuccessNotification";
NSString * const IAPManagerPurchaseFailedNotification  = @"IAPManagerPurchaseFailedNotification";
NSString * const IAPManagerRestoreSuccessNotification  = @"IAPManagerRestoreSuccessNotification";
NSString * const IAPManagerRestoreFailedNotification   = @"IAPManagerRestoreFailedNotification";

NSString * const IAPManagerProductIDKey = @"productID";
NSString * const IAPManagerErrorKey     = @"error";

@interface IAPManager ()

@property (nonatomic, strong) NSMutableArray<SKProduct *> *mutableProducts;
@property (nonatomic, strong) NSMutableSet<NSString *> *purchasedProductIDs;
@property (nonatomic, strong) SKProductsRequest *productsRequest;
@property (nonatomic, copy) void(^fetchCompletion)(NSArray<SKProduct *> *products, NSError *error);
@property (nonatomic, assign) BOOL isLoading;

@end

@implementation IAPManager

+ (instancetype)sharedManager {
    static IAPManager *instance = nil;
    static dispatch_once_t onceToken;
    dispatch_once(&onceToken, ^{
        instance = [[IAPManager alloc] init];
    });
    return instance;
}

- (instancetype)init {
    self = [super init];
    if (self) {
        _mutableProducts = [NSMutableArray array];
        _purchasedProductIDs = [NSMutableSet set];
        
        // โหลด purchased products จาก UserDefaults
        [self loadPurchasedProducts];
        
        // เพิ่มตัวเองเป็น transaction observer
        [[SKPaymentQueue defaultQueue] addTransactionObserver:self];
    }
    return self;
}

- (void)dealloc {
    [[SKPaymentQueue defaultQueue] removeTransactionObserver:self];
}

- (NSArray<SKProduct *> *)availableProducts {
    return [self.mutableProducts copy];
}

#pragma mark - Products

- (void)fetchProductsWithIdentifiers:(NSSet<NSString *> *)identifiers
                          completion:(void(^)(NSArray<SKProduct *> *products, NSError *error))completion {
    self.fetchCompletion = completion;
    self.isLoading = YES;
    
    self.productsRequest = [[SKProductsRequest alloc] initWithProductIdentifiers:identifiers];
    self.productsRequest.delegate = self;
    [self.productsRequest start];
}

#pragma mark - SKProductsRequestDelegate

- (void)productsRequest:(SKProductsRequest *)request
     didReceiveResponse:(SKProductsResponse *)response {
    
    self.isLoading = NO;
    [self.mutableProducts removeAllObjects];
    [self.mutableProducts addObjectsFromArray:response.products];
    
    // Log invalid product identifiers
    if (response.invalidProductIdentifiers.count > 0) {
        NSLog(@"Invalid Product IDs: %@", response.invalidProductIdentifiers);
    }
    
    NSLog(@"Found %lu valid products", (unsigned long)response.products.count);
    for (SKProduct *product in response.products) {
        NSLog(@"Product: %@ - %@", product.productIdentifier, product.localizedTitle);
    }
    
    // เรียก completion handler
    if (self.fetchCompletion) {
        dispatch_async(dispatch_get_main_queue(), ^{
            self.fetchCompletion(response.products, nil);
            self.fetchCompletion = nil;
        });
    }
}

- (void)request:(SKRequest *)request didFailWithError:(NSError *)error {
    self.isLoading = NO;
    NSLog(@"Products request failed: %@", error.localizedDescription);
    
    if (self.fetchCompletion) {
        dispatch_async(dispatch_get_main_queue(), ^{
            self.fetchCompletion(nil, error);
            self.fetchCompletion = nil;
        });
    }
}

#pragma mark - Purchasing

- (BOOL)canMakePurchases {
    return [SKPaymentQueue canMakePayments];
}

- (void)purchaseProduct:(SKProduct *)product {
    if (![self canMakePurchases]) {
        NSLog(@"Purchases are disabled on this device");
        return;
    }
    
    SKPayment *payment = [SKPayment paymentWithProduct:product];
    [[SKPaymentQueue defaultQueue] addPayment:payment];
}

- (void)purchaseProductWithIdentifier:(NSString *)productID {
    SKProduct *product = [self productForIdentifier:productID];
    if (product) {
        [self purchaseProduct:product];
    } else {
        NSLog(@"Product not found for ID: %@", productID);
    }
}

- (void)restorePurchases {
    [[SKPaymentQueue defaultQueue] restoreCompletedTransactions];
}

#pragma mark - SKPaymentTransactionObserver

- (void)paymentQueue:(SKPaymentQueue *)queue
 updatedTransactions:(NSArray<SKPaymentTransaction *> *)transactions {
    
    for (SKPaymentTransaction *transaction in transactions) {
        switch (transaction.transactionState) {
            case SKPaymentTransactionStatePurchased:
                [self handlePurchasedTransaction:transaction];
                break;
                
            case SKPaymentTransactionStateFailed:
                [self handleFailedTransaction:transaction];
                break;
                
            case SKPaymentTransactionStateRestored:
                [self handleRestoredTransaction:transaction];
                break;
                
            case SKPaymentTransactionStateDeferred:
                [self handleDeferredTransaction:transaction];
                break;
                
            case SKPaymentTransactionStatePurchasing:
                NSLog(@"Transaction purchasing: %@", transaction.payment.productIdentifier);
                break;
        }
    }
}

- (void)handlePurchasedTransaction:(SKPaymentTransaction *)transaction {
    NSString *productID = transaction.payment.productIdentifier;
    NSLog(@"Purchase successful: %@", productID);
    
    // บันทึกการซื้อ
    [self savePurchaseForProductID:productID];
    
    // Deliver the purchased content
    [self deliverPurchaseForProductID:productID];
    
    // Finish the transaction - ต้องเรียกเสมอ
    [[SKPaymentQueue defaultQueue] finishTransaction:transaction];
    
    // แจ้ง notification
    dispatch_async(dispatch_get_main_queue(), ^{
        [[NSNotificationCenter defaultCenter] 
            postNotificationName:IAPManagerPurchaseSuccessNotification
            object:self
            userInfo:@{IAPManagerProductIDKey: productID}];
    });
}

- (void)handleFailedTransaction:(SKPaymentTransaction *)transaction {
    NSString *productID = transaction.payment.productIdentifier;
    
    if (transaction.error.code != SKErrorPaymentCancelled) {
        NSLog(@"Transaction failed: %@ - %@", productID, transaction.error.localizedDescription);
        
        dispatch_async(dispatch_get_main_queue(), ^{
            [[NSNotificationCenter defaultCenter]
                postNotificationName:IAPManagerPurchaseFailedNotification
                object:self
                userInfo:@{
                    IAPManagerProductIDKey: productID,
                    IAPManagerErrorKey: transaction.error
                }];
        });
    } else {
        NSLog(@"Transaction cancelled: %@", productID);
    }
    
    [[SKPaymentQueue defaultQueue] finishTransaction:transaction];
}

- (void)handleRestoredTransaction:(SKPaymentTransaction *)transaction {
    NSString *productID = transaction.originalTransaction.payment.productIdentifier;
    NSLog(@"Transaction restored: %@", productID);
    
    [self savePurchaseForProductID:productID];
    [self deliverPurchaseForProductID:productID];
    
    [[SKPaymentQueue defaultQueue] finishTransaction:transaction];
}

- (void)handleDeferredTransaction:(SKPaymentTransaction *)transaction {
    NSLog(@"Transaction deferred (Ask to Buy): %@", transaction.payment.productIdentifier);
    // แสดง UI บอกผู้ใช้ว่ารอการอนุมัติจากผู้ปกครอง
}

- (void)paymentQueueRestoreCompletedTransactionsFinished:(SKPaymentQueue *)queue {
    NSLog(@"Restore completed successfully");
    dispatch_async(dispatch_get_main_queue(), ^{
        [[NSNotificationCenter defaultCenter]
            postNotificationName:IAPManagerRestoreSuccessNotification
            object:self
            userInfo:nil];
    });
}

- (void)paymentQueue:(SKPaymentQueue *)queue
restoreCompletedTransactionsFailedWithError:(NSError *)error {
    NSLog(@"Restore failed: %@", error.localizedDescription);
    dispatch_async(dispatch_get_main_queue(), ^{
        [[NSNotificationCenter defaultCenter]
            postNotificationName:IAPManagerRestoreFailedNotification
            object:self
            userInfo:@{IAPManagerErrorKey: error}];
    });
}

#pragma mark - Private Helpers

- (SKProduct *)productForIdentifier:(NSString *)productID {
    for (SKProduct *product in self.mutableProducts) {
        if ([product.productIdentifier isEqualToString:productID]) {
            return product;
        }
    }
    return nil;
}

- (void)deliverPurchaseForProductID:(NSString *)productID {
    // Override this to deliver purchased content
    // หรือใช้ Notification ให้ parts อื่นของแอปจัดการ
    NSLog(@"Delivering content for: %@", productID);
}

- (void)savePurchaseForProductID:(NSString *)productID {
    [self.purchasedProductIDs addObject:productID];
    
    NSArray *savedIDs = [self.purchasedProductIDs allObjects];
    [[NSUserDefaults standardUserDefaults] setObject:savedIDs forKey:@"PurchasedProductIDs"];
    [[NSUserDefaults standardUserDefaults] synchronize];
}

- (void)loadPurchasedProducts {
    NSArray *savedIDs = [[NSUserDefaults standardUserDefaults] objectForKey:@"PurchasedProductIDs"];
    if (savedIDs) {
        [self.purchasedProductIDs addObjectsFromArray:savedIDs];
    }
}

- (BOOL)isProductPurchased:(NSString *)productID {
    return [self.purchasedProductIDs containsObject:productID];
}

@end
```

---

## 70.4 การแสดงราคาสินค้า

```objc
// การ format ราคาตาม locale ของผู้ใช้
- (NSString *)formattedPriceForProduct:(SKProduct *)product {
    NSNumberFormatter *formatter = [[NSNumberFormatter alloc] init];
    formatter.numberStyle = NSNumberFormatterCurrencyStyle;
    formatter.locale = product.priceLocale;
    return [formatter stringFromNumber:product.price];
}

// การแสดงข้อมูลสินค้า
- (void)displayProductInfo:(SKProduct *)product {
    NSString *price = [self formattedPriceForProduct:product];
    
    NSLog(@"ชื่อสินค้า: %@", product.localizedTitle);
    NSLog(@"รายละเอียด: %@", product.localizedDescription);
    NSLog(@"ราคา: %@", price);
    NSLog(@"Product ID: %@", product.productIdentifier);
    
    // สำหรับ Subscription
    if (@available(iOS 11.2, *)) {
        SKProductSubscriptionPeriod *period = product.subscriptionPeriod;
        if (period) {
            NSString *periodString = [self descriptionForPeriod:period];
            NSLog(@"ระยะเวลา subscription: %@", periodString);
        }
        
        SKProductDiscount *introDiscount = product.introductoryPrice;
        if (introDiscount) {
            NSNumberFormatter *f = [[NSNumberFormatter alloc] init];
            f.numberStyle = NSNumberFormatterCurrencyStyle;
            f.locale = product.priceLocale;
            NSString *introPrice = [f stringFromNumber:introDiscount.price];
            NSLog(@"ราคาแนะนำ: %@", introPrice);
        }
    }
}

- (NSString *)descriptionForPeriod:(SKProductSubscriptionPeriod *)period API_AVAILABLE(ios(11.2)) {
    NSString *unit;
    switch (period.unit) {
        case SKProductPeriodUnitDay:
            unit = period.numberOfUnits == 1 ? @"วัน" : [NSString stringWithFormat:@"%lu วัน", (unsigned long)period.numberOfUnits];
            break;
        case SKProductPeriodUnitWeek:
            unit = period.numberOfUnits == 1 ? @"สัปดาห์" : [NSString stringWithFormat:@"%lu สัปดาห์", (unsigned long)period.numberOfUnits];
            break;
        case SKProductPeriodUnitMonth:
            unit = period.numberOfUnits == 1 ? @"เดือน" : [NSString stringWithFormat:@"%lu เดือน", (unsigned long)period.numberOfUnits];
            break;
        case SKProductPeriodUnitYear:
            unit = period.numberOfUnits == 1 ? @"ปี" : [NSString stringWithFormat:@"%lu ปี", (unsigned long)period.numberOfUnits];
            break;
    }
    return unit;
}
```

---

## 70.5 Store View Controller

```objc
// StoreViewController.h
@interface StoreViewController : UIViewController

@end

// StoreViewController.m
@interface StoreViewController () <UITableViewDelegate, UITableViewDataSource>

@property (nonatomic, strong) UITableView *tableView;
@property (nonatomic, strong) NSArray<SKProduct *> *products;
@property (nonatomic, strong) UIActivityIndicatorView *loadingIndicator;

@end

@implementation StoreViewController

static NSString * const kProductIDPremium    = @"com.yourapp.premium";
static NSString * const kProductIDCoins100   = @"com.yourapp.coins.100";
static NSString * const kProductIDCoins500   = @"com.yourapp.coins.500";
static NSString * const kProductIDMonthly    = @"com.yourapp.subscription.monthly";
static NSString * const kProductIDYearly     = @"com.yourapp.subscription.yearly";

- (void)viewDidLoad {
    [super viewDidLoad];
    
    self.title = @"Store";
    self.view.backgroundColor = [UIColor systemBackgroundColor];
    
    [self setupTableView];
    [self setupLoadingIndicator];
    [self setupNotifications];
    [self fetchProducts];
    
    // ปุ่ม Restore
    UIBarButtonItem *restoreButton = [[UIBarButtonItem alloc]
        initWithTitle:@"Restore"
        style:UIBarButtonItemStylePlain
        target:self
        action:@selector(restorePurchases)];
    self.navigationItem.rightBarButtonItem = restoreButton;
}

- (void)setupTableView {
    self.tableView = [[UITableView alloc] initWithFrame:self.view.bounds style:UITableViewStyleGrouped];
    self.tableView.delegate = self;
    self.tableView.dataSource = self;
    self.tableView.autoresizingMask = UIViewAutoresizingFlexibleWidth | UIViewAutoresizingFlexibleHeight;
    [self.view addSubview:self.tableView];
}

- (void)setupLoadingIndicator {
    self.loadingIndicator = [[UIActivityIndicatorView alloc] initWithActivityIndicatorStyle:UIActivityIndicatorViewStyleLarge];
    self.loadingIndicator.center = self.view.center;
    self.loadingIndicator.hidesWhenStopped = YES;
    [self.view addSubview:self.loadingIndicator];
}

- (void)setupNotifications {
    [[NSNotificationCenter defaultCenter] addObserver:self
        selector:@selector(purchaseSuccess:)
        name:IAPManagerPurchaseSuccessNotification
        object:nil];
    
    [[NSNotificationCenter defaultCenter] addObserver:self
        selector:@selector(purchaseFailed:)
        name:IAPManagerPurchaseFailedNotification
        object:nil];
    
    [[NSNotificationCenter defaultCenter] addObserver:self
        selector:@selector(restoreSuccess)
        name:IAPManagerRestoreSuccessNotification
        object:nil];
}

- (void)fetchProducts {
    [self.loadingIndicator startAnimating];
    
    NSSet *productIDs = [NSSet setWithObjects:
        kProductIDPremium,
        kProductIDCoins100,
        kProductIDCoins500,
        kProductIDMonthly,
        kProductIDYearly,
        nil];
    
    [[IAPManager sharedManager] fetchProductsWithIdentifiers:productIDs
                                                  completion:^(NSArray<SKProduct *> *products, NSError *error) {
        [self.loadingIndicator stopAnimating];
        
        if (error) {
            [self showAlert:@"เกิดข้อผิดพลาด" message:error.localizedDescription];
        } else {
            // เรียงสินค้าตาม Product ID
            self.products = [products sortedArrayUsingComparator:^NSComparisonResult(SKProduct *p1, SKProduct *p2) {
                return [p1.productIdentifier compare:p2.productIdentifier];
            }];
            [self.tableView reloadData];
        }
    }];
}

#pragma mark - UITableViewDataSource

- (NSInteger)numberOfSectionsInTableView:(UITableView *)tableView {
    return 3; // Consumable, Non-Consumable, Subscription
}

- (NSString *)tableView:(UITableView *)tableView titleForHeaderInSection:(NSInteger)section {
    switch (section) {
        case 0: return @"เหรียญ (Consumable)";
        case 1: return @"Premium (Non-Consumable)";
        case 2: return @"Subscription";
        default: return nil;
    }
}

- (NSInteger)tableView:(UITableView *)tableView numberOfRowsInSection:(NSInteger)section {
    switch (section) {
        case 0: return 2; // Coins 100 & 500
        case 1: return 1; // Premium
        case 2: return 2; // Monthly & Yearly
        default: return 0;
    }
}

- (UITableViewCell *)tableView:(UITableView *)tableView cellForRowAtIndexPath:(NSIndexPath *)indexPath {
    UITableViewCell *cell = [tableView dequeueReusableCellWithIdentifier:@"ProductCell"];
    if (!cell) {
        cell = [[UITableViewCell alloc] initWithStyle:UITableViewCellStyleSubtitle reuseIdentifier:@"ProductCell"];
    }
    
    NSString *productID = [self productIDForIndexPath:indexPath];
    SKProduct *product = [self productForID:productID];
    
    if (product) {
        NSNumberFormatter *formatter = [[NSNumberFormatter alloc] init];
        formatter.numberStyle = NSNumberFormatterCurrencyStyle;
        formatter.locale = product.priceLocale;
        NSString *price = [formatter stringFromNumber:product.price];
        
        cell.textLabel.text = product.localizedTitle;
        cell.detailTextLabel.text = [NSString stringWithFormat:@"%@ - %@", price, product.localizedDescription];
        
        // ตรวจสอบว่าซื้อแล้วหรือไม่
        BOOL isPurchased = [[IAPManager sharedManager] isProductPurchased:productID];
        if (isPurchased && indexPath.section != 0) { // ไม่แสดงสำหรับ consumable
            cell.accessoryType = UITableViewCellAccessoryCheckmark;
            cell.textLabel.textColor = [UIColor secondaryLabelColor];
        } else {
            cell.accessoryType = UITableViewCellAccessoryNone;
            cell.textLabel.textColor = [UIColor labelColor];
        }
    } else {
        cell.textLabel.text = @"กำลังโหลด...";
        cell.detailTextLabel.text = productID;
    }
    
    return cell;
}

- (void)tableView:(UITableView *)tableView didSelectRowAtIndexPath:(NSIndexPath *)indexPath {
    [tableView deselectRowAtIndexPath:indexPath animated:YES];
    
    NSString *productID = [self productIDForIndexPath:indexPath];
    
    // ถ้าซื้อแล้ว (non-consumable) ไม่ต้องซื้ออีก
    if (indexPath.section != 0 && [[IAPManager sharedManager] isProductPurchased:productID]) {
        [self showAlert:@"ซื้อแล้ว" message:@"คุณได้ซื้อสินค้านี้แล้ว"];
        return;
    }
    
    SKProduct *product = [self productForID:productID];
    if (!product) {
        [self showAlert:@"ข้อผิดพลาด" message:@"ไม่พบสินค้า"];
        return;
    }
    
    // แสดง confirmation
    NSNumberFormatter *formatter = [[NSNumberFormatter alloc] init];
    formatter.numberStyle = NSNumberFormatterCurrencyStyle;
    formatter.locale = product.priceLocale;
    NSString *price = [formatter stringFromNumber:product.price];
    
    NSString *message = [NSString stringWithFormat:@"ราคา: %@\n%@", price, product.localizedDescription];
    UIAlertController *alert = [UIAlertController alertControllerWithTitle:product.localizedTitle
                                                                    message:message
                                                             preferredStyle:UIAlertControllerStyleAlert];
    
    [alert addAction:[UIAlertAction actionWithTitle:@"ซื้อ"
                                             style:UIAlertActionStyleDefault
                                           handler:^(UIAlertAction *action) {
        [[IAPManager sharedManager] purchaseProduct:product];
    }]];
    
    [alert addAction:[UIAlertAction actionWithTitle:@"ยกเลิก"
                                             style:UIAlertActionStyleCancel
                                           handler:nil]];
    
    [self presentViewController:alert animated:YES completion:nil];
}

#pragma mark - Helpers

- (NSString *)productIDForIndexPath:(NSIndexPath *)indexPath {
    if (indexPath.section == 0) {
        return indexPath.row == 0 ? kProductIDCoins100 : kProductIDCoins500;
    } else if (indexPath.section == 1) {
        return kProductIDPremium;
    } else {
        return indexPath.row == 0 ? kProductIDMonthly : kProductIDYearly;
    }
}

- (SKProduct *)productForID:(NSString *)productID {
    for (SKProduct *p in self.products) {
        if ([p.productIdentifier isEqualToString:productID]) return p;
    }
    return nil;
}

- (void)showAlert:(NSString *)title message:(NSString *)message {
    UIAlertController *alert = [UIAlertController alertControllerWithTitle:title
                                                                    message:message
                                                             preferredStyle:UIAlertControllerStyleAlert];
    [alert addAction:[UIAlertAction actionWithTitle:@"ตกลง" style:UIAlertActionStyleDefault handler:nil]];
    [self presentViewController:alert animated:YES completion:nil];
}

#pragma mark - Notifications

- (void)purchaseSuccess:(NSNotification *)notification {
    NSString *productID = notification.userInfo[IAPManagerProductIDKey];
    [self showAlert:@"ซื้อสำเร็จ" message:[NSString stringWithFormat:@"ขอบคุณสำหรับการซื้อ %@", productID]];
    [self.tableView reloadData];
}

- (void)purchaseFailed:(NSNotification *)notification {
    NSError *error = notification.userInfo[IAPManagerErrorKey];
    [self showAlert:@"ซื้อไม่สำเร็จ" message:error.localizedDescription];
}

- (void)restoreSuccess {
    [self showAlert:@"Restore สำเร็จ" message:@"ได้คืนสินค้าที่ซื้อไว้แล้ว"];
    [self.tableView reloadData];
}

- (void)restorePurchases {
    [[IAPManager sharedManager] restorePurchases];
}

- (void)dealloc {
    [[NSNotificationCenter defaultCenter] removeObserver:self];
}

@end
```

---

## 70.6 Receipt Validation

Receipt Validation ใช้เพื่อยืนยันว่าการซื้อถูกต้องจริง ป้องกันการทุจริต

### Client-Side Validation (ไม่แนะนำสำหรับ Production)

```objc
- (void)validateReceiptLocally {
    NSURL *receiptURL = [[NSBundle mainBundle] appStoreReceiptURL];
    
    if (![[NSFileManager defaultManager] fileExistsAtPath:receiptURL.path]) {
        NSLog(@"ไม่พบ receipt");
        return;
    }
    
    NSData *receiptData = [NSData dataWithContentsOfURL:receiptURL];
    if (!receiptData) {
        NSLog(@"ไม่สามารถอ่าน receipt ได้");
        return;
    }
    
    NSString *receiptBase64 = [receiptData base64EncodedStringWithOptions:0];
    NSLog(@"Receipt (Base64): %@...", [receiptBase64 substringToIndex:50]);
    
    // ส่งไปยัง server เพื่อ validate
    [self validateReceiptWithServer:receiptBase64];
}
```

### Server-Side Validation (แนะนำ)

```objc
- (void)validateReceiptWithServer:(NSString *)receiptBase64 {
    // URL ของ server ของคุณที่จะ forward ไปยัง Apple
    NSURL *serverURL = [NSURL URLWithString:@"https://yourserver.com/validate-receipt"];
    
    NSMutableURLRequest *request = [NSMutableURLRequest requestWithURL:serverURL];
    request.HTTPMethod = @"POST";
    [request setValue:@"application/json" forHTTPHeaderField:@"Content-Type"];
    
    NSDictionary *body = @{
        @"receipt-data": receiptBase64,
        @"password": @"your-shared-secret" // สำหรับ subscriptions
    };
    
    NSData *bodyData = [NSJSONSerialization dataWithJSONObject:body options:0 error:nil];
    request.HTTPBody = bodyData;
    
    NSURLSession *session = [NSURLSession sharedSession];
    [[session dataTaskWithRequest:request
               completionHandler:^(NSData *data, NSURLResponse *response, NSError *error) {
        if (error) {
            NSLog(@"Network error: %@", error.localizedDescription);
            return;
        }
        
        NSDictionary *result = [NSJSONSerialization JSONObjectWithData:data options:0 error:nil];
        [self processValidationResult:result];
    }] resume];
}

// ถ้า validate กับ Apple โดยตรง (ไม่แนะนำ ควรผ่าน server)
- (void)validateWithAppleDirectly:(NSString *)receiptBase64 isSandbox:(BOOL)isSandbox {
    NSString *urlString = isSandbox
        ? @"https://sandbox.itunes.apple.com/verifyReceipt"
        : @"https://buy.itunes.apple.com/verifyReceipt";
    
    NSURL *url = [NSURL URLWithString:urlString];
    NSMutableURLRequest *request = [NSMutableURLRequest requestWithURL:url];
    request.HTTPMethod = @"POST";
    [request setValue:@"application/json" forHTTPHeaderField:@"Content-Type"];
    
    NSDictionary *body = @{@"receipt-data": receiptBase64};
    request.HTTPBody = [NSJSONSerialization dataWithJSONObject:body options:0 error:nil];
    
    [[[NSURLSession sharedSession] dataTaskWithRequest:request
        completionHandler:^(NSData *data, NSURLResponse *response, NSError *error) {
        
        if (error || !data) return;
        
        NSDictionary *json = [NSJSONSerialization JSONObjectWithData:data options:0 error:nil];
        NSInteger status = [json[@"status"] integerValue];
        
        switch (status) {
            case 0:
                NSLog(@"Receipt valid");
                [self processValidationResult:json];
                break;
            case 21007:
                NSLog(@"Sandbox receipt, trying sandbox URL");
                [self validateWithAppleDirectly:receiptBase64 isSandbox:YES];
                break;
            case 21008:
                NSLog(@"Production receipt, trying production URL");
                [self validateWithAppleDirectly:receiptBase64 isSandbox:NO];
                break;
            default:
                NSLog(@"Invalid receipt, status: %ld", (long)status);
                break;
        }
    }] resume];
}

- (void)processValidationResult:(NSDictionary *)result {
    // Parse receipt information
    NSDictionary *receipt = result[@"receipt"];
    NSArray *inAppPurchases = receipt[@"in_app"];
    
    NSLog(@"In-app purchases: %lu", (unsigned long)inAppPurchases.count);
    
    for (NSDictionary *purchase in inAppPurchases) {
        NSString *productID = purchase[@"product_id"];
        NSString *purchaseDate = purchase[@"purchase_date"];
        NSString *quantity = purchase[@"quantity"];
        
        NSLog(@"Product: %@, Date: %@, Qty: %@", productID, purchaseDate, quantity);
    }
    
    // สำหรับ subscription ตรวจสอบ latest_receipt_info
    NSArray *latestReceiptInfo = result[@"latest_receipt_info"];
    if (latestReceiptInfo) {
        [self processSubscriptionInfo:latestReceiptInfo];
    }
}

- (void)processSubscriptionInfo:(NSArray *)latestReceiptInfo {
    // หา receipt ล่าสุด
    NSDictionary *latestReceipt = [latestReceiptInfo firstObject];
    
    NSString *productID = latestReceipt[@"product_id"];
    NSString *expiresDateMs = latestReceipt[@"expires_date_ms"];
    
    if (expiresDateMs) {
        NSTimeInterval expireTimestamp = [expiresDateMs doubleValue] / 1000.0;
        NSDate *expiryDate = [NSDate dateWithTimeIntervalSince1970:expireTimestamp];
        BOOL isActive = [expiryDate timeIntervalSinceNow] > 0;
        
        NSLog(@"Subscription %@: %@, expires: %@", 
              productID, 
              isActive ? @"Active" : @"Expired",
              expiryDate);
    }
}
```

---

## 70.7 Subscription Management

```objc
// SubscriptionManager.h
@interface SubscriptionManager : NSObject

@property (nonatomic, assign, readonly) BOOL isSubscribed;
@property (nonatomic, strong, readonly) NSDate *expiryDate;
@property (nonatomic, strong, readonly) NSString *activeSubscriptionID;

+ (instancetype)sharedManager;

- (void)checkSubscriptionStatus:(void(^)(BOOL isActive, NSDate *expiryDate))completion;
- (void)updateSubscriptionStatus:(NSDictionary *)receiptInfo;

@end

// SubscriptionManager.m
@implementation SubscriptionManager

+ (instancetype)sharedManager {
    static SubscriptionManager *instance = nil;
    static dispatch_once_t onceToken;
    dispatch_once(&onceToken, ^{
        instance = [[SubscriptionManager alloc] init];
    });
    return instance;
}

- (instancetype)init {
    self = [super init];
    if (self) {
        [self loadStoredSubscriptionData];
    }
    return self;
}

- (void)loadStoredSubscriptionData {
    NSUserDefaults *defaults = [NSUserDefaults standardUserDefaults];
    double expiryTimestamp = [defaults doubleForKey:@"SubscriptionExpiryDate"];
    
    if (expiryTimestamp > 0) {
        _expiryDate = [NSDate dateWithTimeIntervalSince1970:expiryTimestamp];
        _isSubscribed = [_expiryDate timeIntervalSinceNow] > 0;
        _activeSubscriptionID = [defaults stringForKey:@"ActiveSubscriptionID"];
    }
}

- (void)checkSubscriptionStatus:(void(^)(BOOL isActive, NSDate *expiryDate))completion {
    // ตรวจสอบจาก local data ก่อน
    if (self.isSubscribed && self.expiryDate) {
        BOOL isActive = [self.expiryDate timeIntervalSinceNow] > 0;
        if (isActive) {
            completion(YES, self.expiryDate);
            return;
        }
    }
    
    // ถ้าไม่ active ให้ fetch receipt ใหม่
    NSURL *receiptURL = [[NSBundle mainBundle] appStoreReceiptURL];
    NSData *receiptData = [NSData dataWithContentsOfURL:receiptURL];
    
    if (!receiptData) {
        completion(NO, nil);
        return;
    }
    
    // ส่งไปยัง server
    NSString *receiptBase64 = [receiptData base64EncodedStringWithOptions:0];
    // ... (validate with server logic)
    
    // ตัวอย่าง: สมมติได้รับผลลัพธ์
    completion(NO, nil);
}

- (void)updateSubscriptionStatus:(NSDictionary *)receiptInfo {
    NSString *expiresDateMs = receiptInfo[@"expires_date_ms"];
    NSString *productID = receiptInfo[@"product_id"];
    
    if (expiresDateMs) {
        NSTimeInterval expireTimestamp = [expiresDateMs doubleValue] / 1000.0;
        _expiryDate = [NSDate dateWithTimeIntervalSince1970:expireTimestamp];
        _isSubscribed = [_expiryDate timeIntervalSinceNow] > 0;
        _activeSubscriptionID = productID;
        
        // บันทึกลง UserDefaults
        NSUserDefaults *defaults = [NSUserDefaults standardUserDefaults];
        [defaults setDouble:expireTimestamp forKey:@"SubscriptionExpiryDate"];
        [defaults setObject:productID forKey:@"ActiveSubscriptionID"];
        [defaults synchronize];
    }
}

@end
```

---

## 70.8 Transaction Observer ขั้นสูง

```objc
// การจัดการ Promoted IAP (iOS 11+)
- (BOOL)paymentQueue:(SKPaymentQueue *)queue
shouldAddStorePayment:(SKPayment *)payment
          forProduct:(SKProduct *)product API_AVAILABLE(ios(11.0)) {
    
    NSLog(@"User initiated purchase from App Store: %@", product.productIdentifier);
    
    // ตรวจสอบว่าผู้ใช้ login แล้วหรือไม่
    BOOL isLoggedIn = [self checkUserLoginStatus];
    
    if (!isLoggedIn) {
        // บันทึก pending payment
        [self savePendingPayment:payment];
        // แสดง login screen
        [self presentLoginScreen];
        return NO; // ไม่ดำเนินการซื้อตอนนี้
    }
    
    return YES; // ดำเนินการซื้อได้เลย
}

// หลัง login แล้ว ดำเนินการซื้อต่อ
- (void)processPendingPurchase {
    SKPayment *pendingPayment = [self loadPendingPayment];
    if (pendingPayment) {
        [[SKPaymentQueue defaultQueue] addPayment:pendingPayment];
        [self clearPendingPayment];
    }
}

// สำหรับ downloading hosted content
- (void)paymentQueue:(SKPaymentQueue *)queue
  updatedDownloads:(NSArray<SKDownload *> *)downloads {
    
    for (SKDownload *download in downloads) {
        switch (download.state) {
            case SKDownloadStateActive:
                NSLog(@"Downloading: %.0f%%", download.progress * 100);
                break;
                
            case SKDownloadStateFinished:
                NSLog(@"Download finished: %@", download.contentURL);
                [self processDownloadedContent:download];
                [[SKPaymentQueue defaultQueue] finishTransaction:download.transaction];
                break;
                
            case SKDownloadStateFailed:
                NSLog(@"Download failed: %@", download.error.localizedDescription);
                [[SKPaymentQueue defaultQueue] finishTransaction:download.transaction];
                break;
                
            case SKDownloadStatePaused:
                // Resume download
                [[SKPaymentQueue defaultQueue] resumeDownloads:@[download]];
                break;
                
            default:
                break;
        }
    }
}

- (BOOL)checkUserLoginStatus { return NO; } // Placeholder
- (void)savePendingPayment:(SKPayment *)payment { /* save */ }
- (SKPayment *)loadPendingPayment { return nil; } // Placeholder
- (void)clearPendingPayment { /* clear */ }
- (void)presentLoginScreen { /* present */ }
- (void)processDownloadedContent:(SKDownload *)download { /* process */ }
```

---

## 70.9 Sandbox Testing

### การตั้งค่า Sandbox

1. ใน App Store Connect สร้าง Sandbox Tester accounts
2. ในอุปกรณ์ทดสอบ: Settings > App Store > Sandbox Account

### เงื่อนไขของ Sandbox

```objc
// ตรวจสอบว่าเป็น Sandbox หรือ Production
- (BOOL)isSandboxEnvironment {
    NSURL *receiptURL = [[NSBundle mainBundle] appStoreReceiptURL];
    return [receiptURL.lastPathComponent isEqualToString:@"sandboxReceipt"];
}

// Log สำหรับ debug
- (void)logTransactionDetails:(SKPaymentTransaction *)transaction {
    #if DEBUG
    NSLog(@"=== Transaction Details ===");
    NSLog(@"Product ID: %@", transaction.payment.productIdentifier);
    NSLog(@"State: %ld", (long)transaction.transactionState);
    NSLog(@"Transaction ID: %@", transaction.transactionIdentifier);
    NSLog(@"Date: %@", transaction.transactionDate);
    if (transaction.error) {
        NSLog(@"Error: %@", transaction.error.localizedDescription);
        NSLog(@"Error Code: %ld", (long)transaction.error.code);
    }
    NSLog(@"========================");
    #endif
}
```

### ระยะเวลา Subscription ใน Sandbox

| Production  | Sandbox    |
|-------------|------------|
| 1 สัปดาห์  | 3 นาที     |
| 1 เดือน    | 5 นาที     |
| 2 เดือน    | 10 นาที    |
| 3 เดือน    | 15 นาที    |
| 6 เดือน    | 30 นาที    |
| 1 ปี       | 1 ชั่วโมง  |

---

## 70.10 StoreKit 2 (iOS 15+)

ใน iOS 15+ Apple แนะนำ StoreKit 2 ที่ใช้ Swift Concurrency แต่สามารถใช้ใน Objective-C ได้ผ่าน bridging

```objc
// ใน Objective-C ยังคงใช้ StoreKit 1 ได้ปกติ
// แต่ถ้าต้องการใช้ StoreKit 2 ต้องเขียน Swift wrapper

// สร้าง Swift wrapper สำหรับ StoreKit 2
// StoreKit2Manager.swift
/*
import StoreKit

@objc class StoreKit2Manager: NSObject {
    
    @objc static let shared = StoreKit2Manager()
    
    @objc func fetchProducts(ids: [String], completion: @escaping ([Any], Error?) -> Void) {
        Task {
            do {
                let products = try await Product.products(for: ids)
                completion(products, nil)
            } catch {
                completion([], error)
            }
        }
    }
    
    @objc func purchase(productId: String, completion: @escaping (Bool, Error?) -> Void) {
        Task {
            guard let products = try? await Product.products(for: [productId]),
                  let product = products.first else {
                completion(false, nil)
                return
            }
            
            do {
                let result = try await product.purchase()
                switch result {
                case .success(let verification):
                    switch verification {
                    case .verified(let transaction):
                        await transaction.finish()
                        completion(true, nil)
                    case .unverified:
                        completion(false, nil)
                    }
                case .userCancelled, .pending:
                    completion(false, nil)
                @unknown default:
                    completion(false, nil)
                }
            } catch {
                completion(false, error)
            }
        }
    }
}
*/
```

---

## 70.11 Best Practices

### 1. เพิ่ม Observer เร็วที่สุด

```objc
// AppDelegate.m
- (BOOL)application:(UIApplication *)application
didFinishLaunchingWithOptions:(NSDictionary *)launchOptions {
    
    // เพิ่ม transaction observer ทันที ก่อน UI โหลด
    // เพื่อไม่พลาด transactions ที่ค้างอยู่
    [[SKPaymentQueue defaultQueue] addTransactionObserver:[IAPManager sharedManager]];
    
    return YES;
}
```

### 2. จัดการ Interrupted Transactions

```objc
// เมื่อแอปเปิดขึ้นมา ตรวจสอบ pending transactions
- (void)checkPendingTransactions {
    NSArray *transactions = [[SKPaymentQueue defaultQueue] transactions];
    
    for (SKPaymentTransaction *transaction in transactions) {
        if (transaction.transactionState == SKPaymentTransactionStatePurchased ||
            transaction.transactionState == SKPaymentTransactionStateRestored) {
            // มี transaction ที่ยังไม่ได้ finish
            [self processTransaction:transaction];
        }
    }
}

- (void)processTransaction:(SKPaymentTransaction *)transaction {
    NSString *productID = transaction.payment.productIdentifier;
    
    // Deliver content ถ้ายังไม่ได้ทำ
    [[IAPManager sharedManager] savePurchaseForProductID:productID];
    [[IAPManager sharedManager] deliverPurchaseForProductID:productID];
    
    // Finish transaction
    [[SKPaymentQueue defaultQueue] finishTransaction:transaction];
}
```

### 3. Error Handling

```objc
- (NSString *)localizedErrorForTransaction:(SKPaymentTransaction *)transaction {
    switch (transaction.error.code) {
        case SKErrorUnknown:
            return @"เกิดข้อผิดพลาดที่ไม่ทราบสาเหตุ";
        case SKErrorClientInvalid:
            return @"ไม่ได้รับอนุญาตให้ทำการซื้อ";
        case SKErrorPaymentCancelled:
            return @"การซื้อถูกยกเลิก";
        case SKErrorPaymentInvalid:
            return @"ข้อมูลการชำระเงินไม่ถูกต้อง";
        case SKErrorPaymentNotAllowed:
            return @"อุปกรณ์นี้ไม่อนุญาตการซื้อ";
        case SKErrorStoreProductNotAvailable:
            return @"สินค้าไม่มีในตลาดของคุณ";
        case SKErrorCloudServicePermissionDenied:
            return @"ไม่ได้รับอนุญาตจาก iCloud";
        case SKErrorCloudServiceNetworkConnectionFailed:
            return @"ไม่สามารถเชื่อมต่อกับ iCloud ได้";
        default:
            return transaction.error.localizedDescription;
    }
}
```

---

## 70.12 แบบฝึกหัด

### แบบฝึกหัดที่ 1: Consumable Purchase

สร้างระบบเหรียญในเกม:
- แสดงยอดเหรียญปัจจุบัน
- ปุ่มซื้อเหรียญ 100, 500, 1000
- อัพเดตยอดเมื่อซื้อสำเร็จ
- บันทึกยอดเหรียญ

```objc
// CoinManager.h
@interface CoinManager : NSObject

@property (nonatomic, assign) NSInteger coinBalance;

+ (instancetype)sharedManager;
- (void)addCoins:(NSInteger)amount;
- (BOOL)spendCoins:(NSInteger)amount;

@end

// CoinManager.m
@implementation CoinManager

+ (instancetype)sharedManager {
    static CoinManager *instance = nil;
    static dispatch_once_t onceToken;
    dispatch_once(&onceToken, ^{
        instance = [[CoinManager alloc] init];
    });
    return instance;
}

- (instancetype)init {
    self = [super init];
    if (self) {
        _coinBalance = [[NSUserDefaults standardUserDefaults] integerForKey:@"CoinBalance"];
    }
    return self;
}

- (void)setCoinBalance:(NSInteger)coinBalance {
    _coinBalance = coinBalance;
    [[NSUserDefaults standardUserDefaults] setInteger:coinBalance forKey:@"CoinBalance"];
    [[NSUserDefaults standardUserDefaults] synchronize];
    [[NSNotificationCenter defaultCenter] postNotificationName:@"CoinBalanceChanged" object:nil];
}

- (void)addCoins:(NSInteger)amount {
    self.coinBalance += amount;
    NSLog(@"เพิ่มเหรียญ %ld เหรียญ, ยอดรวม: %ld", (long)amount, (long)self.coinBalance);
}

- (BOOL)spendCoins:(NSInteger)amount {
    if (self.coinBalance >= amount) {
        self.coinBalance -= amount;
        NSLog(@"ใช้เหรียญ %ld เหรียญ, ยอดรวม: %ld", (long)amount, (long)self.coinBalance);
        return YES;
    }
    NSLog(@"เหรียญไม่พอ! ต้องการ: %ld, มี: %ld", (long)amount, (long)self.coinBalance);
    return NO;
}

@end
```

### แบบฝึกหัดที่ 2: Paywall Screen

สร้างหน้า Paywall:
- แสดง features ของ premium
- แสดงราคาแบบ subscription
- ปุ่มเลือก monthly/yearly
- แสดง trial period (ถ้ามี)
- ปุ่ม Restore Purchases

```objc
@implementation PaywallViewController

- (void)viewDidLoad {
    [super viewDidLoad];
    
    [self setupUI];
    [self loadProducts];
    
    // ติดตามสถานะการซื้อ
    [[NSNotificationCenter defaultCenter] addObserver:self
        selector:@selector(handlePurchaseSuccess:)
        name:IAPManagerPurchaseSuccessNotification
        object:nil];
}

- (void)setupUI {
    // Title
    UILabel *titleLabel = [[UILabel alloc] init];
    titleLabel.text = @"ยกระดับเป็น Premium";
    titleLabel.font = [UIFont boldSystemFontOfSize:28];
    titleLabel.textAlignment = NSTextAlignmentCenter;
    
    // Features list
    NSArray *features = @[
        @"✓ ไม่มีโฆษณา",
        @"✓ เนื้อหาพิเศษทั้งหมด",
        @"✓ ดาวน์โหลดเพื่ออ่าน offline",
        @"✓ รองรับ 5 อุปกรณ์",
        @"✓ Priority Support"
    ];
    
    for (NSString *feature in features) {
        UILabel *featureLabel = [[UILabel alloc] init];
        featureLabel.text = feature;
        featureLabel.font = [UIFont systemFontOfSize:16];
        // เพิ่มใน stack view
    }
}

- (void)loadProducts {
    // โหลดสินค้า subscription
    NSSet *ids = [NSSet setWithObjects:
        @"com.yourapp.subscription.monthly",
        @"com.yourapp.subscription.yearly",
        nil];
    
    [[IAPManager sharedManager] fetchProductsWithIdentifiers:ids
        completion:^(NSArray<SKProduct *> *products, NSError *error) {
        // อัพเดต UI ด้วยราคาจริง
        [self updatePriceLabels:products];
    }];
}

- (void)handlePurchaseSuccess:(NSNotification *)notification {
    [self dismissViewControllerAnimated:YES completion:nil];
}

@end
```

### แบบฝึกหัดที่ 3: Purchase History

สร้างหน้าแสดงประวัติการซื้อจาก receipt

---

## สรุป

ในบทนี้เราได้เรียนรู้:

1. **StoreKit Framework** - การ import และตั้งค่า
2. **ประเภท IAP** - Consumable, Non-Consumable, Subscription
3. **SKProductsRequest** - การดึงข้อมูลสินค้าจาก App Store
4. **SKPaymentQueue** - การจัดการ payment queue
5. **Transaction Handling** - การจัดการ states ต่างๆ
6. **Receipt Validation** - การตรวจสอบความถูกต้องของ receipt
7. **Restore Purchases** - การคืนสินค้าที่ซื้อไว้
8. **Subscription Management** - การจัดการ subscription
9. **Sandbox Testing** - การทดสอบใน sandbox environment

การ implement IAP ที่ถูกต้องต้องคำนึงถึง:
- เพิ่ม transaction observer เร็วที่สุด
- จัดการ interrupted transactions
- Validate receipt จาก server (ไม่ใช่ client)
- Handle errors อย่างเหมาะสม
- Test อย่างละเอียดใน Sandbox ก่อน Production
