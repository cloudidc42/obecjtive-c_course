# ตอนที่ 92: โปรเจกต์สมบูรณ์ - E-Commerce App

## บทนำ

ในบทนี้เราจะสร้าง E-Commerce App แบบสมบูรณ์ด้วย Objective-C ครอบคลุมฟีเจอร์หลักของแอปขายสินค้าออนไลน์ เช่น รายการสินค้า, รายละเอียดสินค้า, ตะกร้าสินค้า, และการค้นหา โดยใช้ MVVM architecture, Core Data, และ UICollectionView

---

## 92.1 App Architecture (MVVM)

### โครงสร้างโปรเจกต์

```
ECommerceApp/
├── AppDelegate.h / .m
├── Models/
│   ├── ECProduct.h / .m
│   ├── ECCartItem.h / .m
│   └── ECCategory.h / .m
├── ViewModels/
│   ├── ECProductListViewModel.h / .m
│   ├── ECProductDetailViewModel.h / .m
│   └── ECCartViewModel.h / .m
├── Views/
│   ├── Cells/
│   │   ├── ECProductCell.h / .m
│   │   └── ECCartCell.h / .m
│   └── ECPriceLabel.h / .m
├── ViewControllers/
│   ├── ECProductListViewController.h / .m
│   ├── ECProductDetailViewController.h / .m
│   └── ECCartViewController.h / .m
├── Networking/
│   └── ECNetworkManager.h / .m
├── Persistence/
│   └── ECCartStore.h / .m
└── Resources/
    └── ECommerceApp.xcdatamodeld
```

### หลักการ MVVM

- **Model**: ข้อมูลสินค้า (`ECProduct`, `ECCartItem`)
- **ViewModel**: Logic การแสดงผล, การ format ราคา, การจัดการ state
- **View/ViewController**: แสดงผลและรับ input จาก user
- **NetworkManager**: ดึงข้อมูลจาก API
- **CartStore**: บันทึกตะกร้าสินค้าด้วย Core Data

---

## 92.2 Product Model + JSON Parsing

### ECProduct.h

```objc
#import <Foundation/Foundation.h>

NS_ASSUME_NONNULL_BEGIN

@interface ECProduct : NSObject

@property (nonatomic, assign) NSInteger productId;
@property (nonatomic, copy)   NSString *title;
@property (nonatomic, copy)   NSString *productDescription;
@property (nonatomic, assign) double price;
@property (nonatomic, copy)   NSString *category;
@property (nonatomic, copy)   NSString *imageURL;
@property (nonatomic, assign) double rating;
@property (nonatomic, assign) NSInteger reviewCount;

+ (instancetype)productFromDictionary:(NSDictionary *)dict;
- (NSString *)formattedPrice;

@end

NS_ASSUME_NONNULL_END
```

### ECProduct.m

```objc
#import "ECProduct.h"

@implementation ECProduct

+ (instancetype)productFromDictionary:(NSDictionary *)dict {
    ECProduct *product = [[ECProduct alloc] init];
    product.productId  = [dict[@"id"] integerValue];
    product.title      = dict[@"title"] ?: @"";
    product.productDescription = dict[@"description"] ?: @"";
    product.price      = [dict[@"price"] doubleValue];
    product.category   = dict[@"category"] ?: @"";
    product.imageURL   = dict[@"image"] ?: @"";

    NSDictionary *ratingDict = dict[@"rating"];
    if ([ratingDict isKindOfClass:[NSDictionary class]]) {
        product.rating      = [ratingDict[@"rate"] doubleValue];
        product.reviewCount = [ratingDict[@"count"] integerValue];
    }
    return product;
}

- (NSString *)formattedPrice {
    return [NSString stringWithFormat:@"$%.2f", self.price];
}

- (NSString *)description {
    return [NSString stringWithFormat:@"<ECProduct id=%ld title=%@>",
            (long)self.productId, self.title];
}

@end
```

### ECCartItem.h / .m

```objc
// ECCartItem.h
@interface ECCartItem : NSObject
@property (nonatomic, strong) ECProduct *product;
@property (nonatomic, assign) NSInteger quantity;
- (double)subtotal;
@end

// ECCartItem.m
@implementation ECCartItem
- (double)subtotal {
    return self.product.price * self.quantity;
}
@end
```

---

## 92.3 NetworkManager สำหรับ Product API

### ECNetworkManager.h

```objc
#import <Foundation/Foundation.h>
#import "ECProduct.h"

NS_ASSUME_NONNULL_BEGIN

typedef void(^ECProductsCompletion)(NSArray<ECProduct *> * _Nullable products,
                                    NSError * _Nullable error);
typedef void(^ECProductCompletion)(ECProduct * _Nullable product,
                                   NSError * _Nullable error);

@interface ECNetworkManager : NSObject

@property (nonatomic, copy) NSString *baseURL;

+ (instancetype)sharedManager;

- (void)fetchProductsWithCompletion:(ECProductsCompletion)completion;
- (void)fetchProductsInCategory:(NSString *)category
                     completion:(ECProductsCompletion)completion;
- (void)fetchProductById:(NSInteger)productId
              completion:(ECProductCompletion)completion;
- (void)searchProducts:(NSString *)query
            completion:(ECProductsCompletion)completion;

@end

NS_ASSUME_NONNULL_END
```

### ECNetworkManager.m

```objc
#import "ECNetworkManager.h"

@interface ECNetworkManager ()
@property (nonatomic, strong) NSURLSession *session;
@end

@implementation ECNetworkManager

+ (instancetype)sharedManager {
    static ECNetworkManager *shared = nil;
    static dispatch_once_t onceToken;
    dispatch_once(&onceToken, ^{
        shared = [[ECNetworkManager alloc] init];
    });
    return shared;
}

- (instancetype)init {
    self = [super init];
    if (self) {
        _baseURL = @"https://fakestoreapi.com";
        NSURLSessionConfiguration *config =
            [NSURLSessionConfiguration defaultSessionConfiguration];
        config.timeoutIntervalForRequest = 30.0;
        _session = [NSURLSession sessionWithConfiguration:config];
    }
    return self;
}

- (void)fetchProductsWithCompletion:(ECProductsCompletion)completion {
    NSURL *url = [NSURL URLWithString:[self.baseURL stringByAppendingString:@"/products"]];
    [self fetchProductsFromURL:url completion:completion];
}

- (void)fetchProductsInCategory:(NSString *)category
                     completion:(ECProductsCompletion)completion {
    NSString *encoded = [category stringByAddingPercentEncodingWithAllowedCharacters:
                         [NSCharacterSet URLPathAllowedCharacterSet]];
    NSString *path = [NSString stringWithFormat:@"/products/category/%@", encoded];
    NSURL *url = [NSURL URLWithString:[self.baseURL stringByAppendingString:path]];
    [self fetchProductsFromURL:url completion:completion];
}

- (void)fetchProductById:(NSInteger)productId
              completion:(ECProductCompletion)completion {
    NSString *path = [NSString stringWithFormat:@"/products/%ld", (long)productId];
    NSURL *url = [NSURL URLWithString:[self.baseURL stringByAppendingString:path]];
    NSURLSessionDataTask *task =
        [self.session dataTaskWithURL:url
                    completionHandler:^(NSData *data, NSURLResponse *response, NSError *error) {
        if (error) { completion(nil, error); return; }
        NSError *jsonError;
        NSDictionary *dict = [NSJSONSerialization JSONObjectWithData:data
                                                             options:0
                                                               error:&jsonError];
        if (jsonError) { completion(nil, jsonError); return; }
        completion([ECProduct productFromDictionary:dict], nil);
    }];
    [task resume];
}

- (void)searchProducts:(NSString *)query
            completion:(ECProductsCompletion)completion {
    // FakeStore API ไม่มี search endpoint — กรองฝั่ง client
    [self fetchProductsWithCompletion:^(NSArray<ECProduct *> *products, NSError *error) {
        if (error || !products) { completion(nil, error); return; }
        NSPredicate *pred = [NSPredicate predicateWithFormat:
                             @"title CONTAINS[cd] %@ OR category CONTAINS[cd] %@",
                             query, query];
        completion([products filteredArrayUsingPredicate:pred], nil);
    }];
}

// MARK: - Private

- (void)fetchProductsFromURL:(NSURL *)url completion:(ECProductsCompletion)completion {
    NSURLSessionDataTask *task =
        [self.session dataTaskWithURL:url
                    completionHandler:^(NSData *data, NSURLResponse *response, NSError *error) {
        if (error) { completion(nil, error); return; }
        NSError *jsonError;
        NSArray *array = [NSJSONSerialization JSONObjectWithData:data
                                                         options:0
                                                           error:&jsonError];
        if (jsonError) { completion(nil, jsonError); return; }
        NSMutableArray<ECProduct *> *products = [NSMutableArray array];
        for (NSDictionary *dict in array) {
            [products addObject:[ECProduct productFromDictionary:dict]];
        }
        completion([products copy], nil);
    }];
    [task resume];
}

@end
```

---

## 92.4 ProductListViewController + UICollectionView

### ECProductListViewController.h

```objc
#import <UIKit/UIKit.h>

@interface ECProductListViewController : UIViewController
@end
```

### ECProductListViewController.m

```objc
#import "ECProductListViewController.h"
#import "ECNetworkManager.h"
#import "ECProductCell.h"
#import "ECProductDetailViewController.h"
#import "ECCartViewController.h"

static NSString * const kProductCellID = @"ECProductCell";

@interface ECProductListViewController ()
    <UICollectionViewDataSource, UICollectionViewDelegate,
     UICollectionViewDelegateFlowLayout, UISearchResultsUpdating>

@property (nonatomic, strong) UICollectionView        *collectionView;
@property (nonatomic, strong) UIRefreshControl        *refreshControl;
@property (nonatomic, strong) UISearchController      *searchController;
@property (nonatomic, strong) NSArray<ECProduct *>    *allProducts;
@property (nonatomic, strong) NSArray<ECProduct *>    *displayProducts;
@property (nonatomic, assign) BOOL                     isLoading;
@property (nonatomic, assign) NSInteger                currentPage;
@property (nonatomic, assign) NSInteger                pageSize;
@end

@implementation ECProductListViewController

- (void)viewDidLoad {
    [super viewDidLoad];
    self.title = @"สินค้าทั้งหมด";
    self.pageSize = 10;
    self.currentPage = 0;
    [self setupNavigationBar];
    [self setupSearchController];
    [self setupCollectionView];
    [self loadProducts];
}

- (void)setupNavigationBar {
    UIBarButtonItem *cartBtn = [[UIBarButtonItem alloc]
        initWithImage:[UIImage systemImageNamed:@"cart"]
                style:UIBarButtonItemStylePlain
               target:self
               action:@selector(openCart)];
    self.navigationItem.rightBarButtonItem = cartBtn;
}

- (void)setupSearchController {
    self.searchController = [[UISearchController alloc] initWithSearchResultsController:nil];
    self.searchController.searchResultsUpdater = self;
    self.searchController.obscuresBackgroundDuringPresentation = NO;
    self.searchController.searchBar.placeholder = @"ค้นหาสินค้า...";
    self.navigationItem.searchController = self.searchController;
    self.navigationItem.hidesSearchBarWhenScrolling = NO;
}

- (void)setupCollectionView {
    UICollectionViewFlowLayout *layout = [[UICollectionViewFlowLayout alloc] init];
    layout.minimumInteritemSpacing = 12;
    layout.minimumLineSpacing = 16;
    layout.sectionInset = UIEdgeInsetsMake(16, 16, 16, 16);

    self.collectionView = [[UICollectionView alloc] initWithFrame:self.view.bounds
                                             collectionViewLayout:layout];
    self.collectionView.autoresizingMask =
        UIViewAutoresizingFlexibleWidth | UIViewAutoresizingFlexibleHeight;
    self.collectionView.backgroundColor = [UIColor systemGroupedBackgroundColor];
    self.collectionView.dataSource = self;
    self.collectionView.delegate   = self;

    [self.collectionView registerClass:[ECProductCell class]
            forCellWithReuseIdentifier:kProductCellID];
    [self.view addSubview:self.collectionView];

    self.refreshControl = [[UIRefreshControl alloc] init];
    [self.refreshControl addTarget:self action:@selector(refreshProducts)
                  forControlEvents:UIControlEventValueChanged];
    self.collectionView.refreshControl = self.refreshControl;
}

- (void)loadProducts {
    if (self.isLoading) return;
    self.isLoading = YES;
    [[ECNetworkManager sharedManager] fetchProductsWithCompletion:^(NSArray *products, NSError *error) {
        dispatch_async(dispatch_get_main_queue(), ^{
            self.isLoading = NO;
            [self.refreshControl endRefreshing];
            if (error) { [self showError:error]; return; }
            self.allProducts     = products;
            self.displayProducts = products;
            [self.collectionView reloadData];
        });
    }];
}

- (void)refreshProducts {
    self.currentPage = 0;
    [self loadProducts];
}

- (void)openCart {
    ECCartViewController *vc = [[ECCartViewController alloc] init];
    [self.navigationController pushViewController:vc animated:YES];
}

// MARK: - UICollectionViewDataSource

- (NSInteger)collectionView:(UICollectionView *)cv numberOfItemsInSection:(NSInteger)section {
    return self.displayProducts.count;
}

- (UICollectionViewCell *)collectionView:(UICollectionView *)cv
                  cellForItemAtIndexPath:(NSIndexPath *)indexPath {
    ECProductCell *cell = [cv dequeueReusableCellWithReuseIdentifier:kProductCellID
                                                        forIndexPath:indexPath];
    [cell configureWithProduct:self.displayProducts[indexPath.item]];
    return cell;
}

// MARK: - UICollectionViewDelegate

- (void)collectionView:(UICollectionView *)cv didSelectItemAtIndexPath:(NSIndexPath *)indexPath {
    ECProductDetailViewController *vc = [[ECProductDetailViewController alloc] init];
    vc.product = self.displayProducts[indexPath.item];
    [self.navigationController pushViewController:vc animated:YES];
}

// MARK: - FlowLayout sizing

- (CGSize)collectionView:(UICollectionView *)cv
                  layout:(UICollectionViewLayout *)layout
  sizeForItemAtIndexPath:(NSIndexPath *)indexPath {
    CGFloat width = (cv.bounds.size.width - 44) / 2;
    return CGSizeMake(width, width * 1.45);
}

// MARK: - UISearchResultsUpdating

- (void)updateSearchResultsForSearchController:(UISearchController *)sc {
    NSString *query = sc.searchBar.text;
    if (query.length == 0) {
        self.displayProducts = self.allProducts;
    } else {
        NSPredicate *pred = [NSPredicate predicateWithFormat:
                             @"title CONTAINS[cd] %@ OR category CONTAINS[cd] %@",
                             query, query];
        self.displayProducts = [self.allProducts filteredArrayUsingPredicate:pred];
    }
    [self.collectionView reloadData];
}

- (void)showError:(NSError *)error {
    UIAlertController *alert = [UIAlertController
        alertControllerWithTitle:@"เกิดข้อผิดพลาด"
                         message:error.localizedDescription
                  preferredStyle:UIAlertControllerStyleAlert];
    [alert addAction:[UIAlertAction actionWithTitle:@"ตกลง" style:UIAlertActionStyleDefault handler:nil]];
    [self presentViewController:alert animated:YES completion:nil];
}

@end
```

### ECProductCell.m (ย่อ)

```objc
@implementation ECProductCell

- (instancetype)initWithFrame:(CGRect)frame {
    self = [super initWithFrame:frame];
    if (self) {
        self.contentView.backgroundColor = [UIColor secondarySystemBackgroundColor];
        self.contentView.layer.cornerRadius = 12;
        self.contentView.clipsToBounds = YES;
        [self buildUI];
    }
    return self;
}

- (void)buildUI {
    _imageView = [[UIImageView alloc] init];
    _imageView.contentMode = UIViewContentModeScaleAspectFit;
    _imageView.translatesAutoresizingMaskIntoConstraints = NO;

    _titleLabel = [[UILabel alloc] init];
    _titleLabel.font = [UIFont systemFontOfSize:13 weight:UIFontWeightMedium];
    _titleLabel.numberOfLines = 2;
    _titleLabel.translatesAutoresizingMaskIntoConstraints = NO;

    _priceLabel = [[UILabel alloc] init];
    _priceLabel.font = [UIFont systemFontOfSize:15 weight:UIFontWeightBold];
    _priceLabel.textColor = [UIColor systemBlueColor];
    _priceLabel.translatesAutoresizingMaskIntoConstraints = NO;

    [self.contentView addSubview:_imageView];
    [self.contentView addSubview:_titleLabel];
    [self.contentView addSubview:_priceLabel];
    // Auto Layout constraints...
}

- (void)configureWithProduct:(ECProduct *)product {
    _titleLabel.text = product.title;
    _priceLabel.text = product.formattedPrice;
    // Load image with URLSession or SDWebImage
    [self loadImageFromURL:product.imageURL];
}

@end
```

---

## 92.5 ProductDetailViewController

### ECProductDetailViewController.m

```objc
#import "ECProductDetailViewController.h"
#import "ECCartStore.h"

@interface ECProductDetailViewController ()
@property (nonatomic, strong) UIScrollView  *scrollView;
@property (nonatomic, strong) UIImageView   *productImageView;
@property (nonatomic, strong) UILabel       *titleLabel;
@property (nonatomic, strong) UILabel       *priceLabel;
@property (nonatomic, strong) UILabel       *categoryLabel;
@property (nonatomic, strong) UILabel       *descriptionLabel;
@property (nonatomic, strong) UILabel       *ratingLabel;
@property (nonatomic, strong) UIStepper     *quantityStepper;
@property (nonatomic, strong) UILabel       *quantityLabel;
@property (nonatomic, strong) UIButton      *addToCartButton;
@end

@implementation ECProductDetailViewController

- (void)viewDidLoad {
    [super viewDidLoad];
    self.view.backgroundColor = [UIColor systemBackgroundColor];
    self.title = @"รายละเอียดสินค้า";
    [self setupScrollView];
    [self setupViews];
    [self populateData];
    [self setupAddToCartButton];
}

- (void)setupScrollView {
    self.scrollView = [[UIScrollView alloc] initWithFrame:self.view.bounds];
    self.scrollView.autoresizingMask =
        UIViewAutoresizingFlexibleWidth | UIViewAutoresizingFlexibleHeight;
    [self.view addSubview:self.scrollView];
}

- (void)setupViews {
    self.productImageView = [[UIImageView alloc] init];
    self.productImageView.contentMode = UIViewContentModeScaleAspectFit;
    self.productImageView.translatesAutoresizingMaskIntoConstraints = NO;

    self.titleLabel = [self makeLabelFont:[UIFont boldSystemFontOfSize:18] lines:0];
    self.priceLabel = [self makeLabelFont:[UIFont systemFontOfSize:22
                                                           weight:UIFontWeightBold] lines:1];
    self.priceLabel.textColor = [UIColor systemBlueColor];
    self.categoryLabel = [self makeLabelFont:[UIFont systemFontOfSize:13] lines:1];
    self.categoryLabel.textColor = [UIColor secondaryLabelColor];
    self.ratingLabel   = [self makeLabelFont:[UIFont systemFontOfSize:14] lines:1];
    self.descriptionLabel = [self makeLabelFont:[UIFont systemFontOfSize:15] lines:0];

    UIStackView *stack = [[UIStackView alloc] initWithArrangedSubviews:@[
        self.productImageView, self.categoryLabel, self.titleLabel,
        self.priceLabel, self.ratingLabel, self.descriptionLabel
    ]];
    stack.axis = UILayoutConstraintAxisVertical;
    stack.spacing = 12;
    stack.translatesAutoresizingMaskIntoConstraints = NO;
    [self.scrollView addSubview:stack];

    [NSLayoutConstraint activateConstraints:@[
        [self.productImageView.heightAnchor constraintEqualToConstant:260],
        [stack.topAnchor constraintEqualToSystemSpacingBelowAnchor:self.scrollView.topAnchor multiplier:1],
        [stack.leadingAnchor constraintEqualToAnchor:self.view.leadingAnchor constant:20],
        [stack.trailingAnchor constraintEqualToAnchor:self.view.trailingAnchor constant:-20],
        [stack.bottomAnchor constraintEqualToAnchor:self.scrollView.bottomAnchor constant:-100]
    ]];
}

- (void)populateData {
    self.titleLabel.text       = self.product.title;
    self.priceLabel.text       = self.product.formattedPrice;
    self.categoryLabel.text    = [NSString stringWithFormat:@"หมวดหมู่: %@", self.product.category];
    self.descriptionLabel.text = self.product.productDescription;
    self.ratingLabel.text      = [NSString stringWithFormat:@"★ %.1f (%ld รีวิว)",
                                   self.product.rating, (long)self.product.reviewCount];
}

- (void)setupAddToCartButton {
    self.quantityStepper = [[UIStepper alloc] init];
    self.quantityStepper.minimumValue = 1;
    self.quantityStepper.maximumValue = 99;
    self.quantityStepper.value = 1;
    [self.quantityStepper addTarget:self action:@selector(stepperChanged:)
                   forControlEvents:UIControlEventValueChanged];

    self.quantityLabel = [[UILabel alloc] init];
    self.quantityLabel.text = @"จำนวน: 1";
    self.quantityLabel.font = [UIFont systemFontOfSize:16];

    self.addToCartButton = [UIButton buttonWithType:UIButtonTypeSystem];
    [self.addToCartButton setTitle:@"เพิ่มลงตะกร้า" forState:UIControlStateNormal];
    self.addToCartButton.backgroundColor = [UIColor systemBlueColor];
    [self.addToCartButton setTitleColor:[UIColor whiteColor] forState:UIControlStateNormal];
    self.addToCartButton.layer.cornerRadius = 12;
    [self.addToCartButton addTarget:self action:@selector(addToCart)
                   forControlEvents:UIControlEventTouchUpInside];

    UIStackView *bottomBar = [[UIStackView alloc] initWithArrangedSubviews:@[
        self.quantityLabel, self.quantityStepper, self.addToCartButton
    ]];
    bottomBar.spacing = 12;
    bottomBar.translatesAutoresizingMaskIntoConstraints = NO;
    [self.view addSubview:bottomBar];

    [NSLayoutConstraint activateConstraints:@[
        [bottomBar.leadingAnchor constraintEqualToAnchor:self.view.leadingAnchor constant:20],
        [bottomBar.trailingAnchor constraintEqualToAnchor:self.view.trailingAnchor constant:-20],
        [bottomBar.bottomAnchor constraintEqualToSystemSpacingBelowAnchor:
            self.view.safeAreaLayoutGuide.bottomAnchor multiplier:1]
    ]];
}

- (void)stepperChanged:(UIStepper *)stepper {
    self.quantityLabel.text = [NSString stringWithFormat:@"จำนวน: %d", (int)stepper.value];
}

- (void)addToCart {
    NSInteger qty = (NSInteger)self.quantityStepper.value;
    [[ECCartStore sharedStore] addProduct:self.product quantity:qty];

    UIAlertController *alert = [UIAlertController
        alertControllerWithTitle:@"เพิ่มสำเร็จ"
                         message:[NSString stringWithFormat:@"เพิ่ม %@ x%ld ลงตะกร้าแล้ว",
                                   self.product.title, (long)qty]
                  preferredStyle:UIAlertControllerStyleAlert];
    [alert addAction:[UIAlertAction actionWithTitle:@"ตกลง" style:UIAlertActionStyleDefault handler:nil]];
    [self presentViewController:alert animated:YES completion:nil];
}

- (UILabel *)makeLabelFont:(UIFont *)font lines:(NSInteger)lines {
    UILabel *label = [[UILabel alloc] init];
    label.font = font;
    label.numberOfLines = lines;
    label.translatesAutoresizingMaskIntoConstraints = NO;
    return label;
}

@end
```

---

## 92.6 ShoppingCart + Core Data Persistence

### Core Data Model (ECommerceApp.xcdatamodeld)

```
Entity: CartItemEntity
  Attributes:
    productId   : Integer 64
    title       : String
    price       : Double
    imageURL    : String
    quantity    : Integer 16
```

### ECCartStore.h

```objc
#import <Foundation/Foundation.h>
#import "ECProduct.h"
#import "ECCartItem.h"

@interface ECCartStore : NSObject

+ (instancetype)sharedStore;

- (void)addProduct:(ECProduct *)product quantity:(NSInteger)qty;
- (void)removeItemAtIndex:(NSInteger)index;
- (void)updateItem:(ECCartItem *)item quantity:(NSInteger)qty;
- (void)clearCart;

- (NSArray<ECCartItem *> *)cartItems;
- (double)totalPrice;
- (NSInteger)totalItemCount;

@end
```

### ECCartStore.m

```objc
#import "ECCartStore.h"
#import <CoreData/CoreData.h>

@interface ECCartStore ()
@property (nonatomic, strong) NSPersistentContainer *container;
@property (nonatomic, strong) NSManagedObjectContext *context;
@end

@implementation ECCartStore

+ (instancetype)sharedStore {
    static ECCartStore *shared = nil;
    static dispatch_once_t onceToken;
    dispatch_once(&onceToken, ^{ shared = [[ECCartStore alloc] init]; });
    return shared;
}

- (instancetype)init {
    self = [super init];
    if (self) {
        _container = [[NSPersistentContainer alloc] initWithName:@"ECommerceApp"];
        [_container loadPersistentStoresWithCompletionHandler:
         ^(NSPersistentStoreDescription *desc, NSError *error) {
            if (error) NSLog(@"Core Data error: %@", error);
        }];
        _context = _container.viewContext;
    }
    return self;
}

- (void)addProduct:(ECProduct *)product quantity:(NSInteger)qty {
    // ตรวจว่ามีสินค้านี้ในตะกร้าแล้วหรือไม่
    NSFetchRequest *req = [NSFetchRequest fetchRequestWithEntityName:@"CartItemEntity"];
    req.predicate = [NSPredicate predicateWithFormat:@"productId == %ld", (long)product.productId];
    NSArray *existing = [self.context executeFetchRequest:req error:nil];

    if (existing.count > 0) {
        NSManagedObject *obj = existing.firstObject;
        NSInteger currentQty = [[obj valueForKey:@"quantity"] integerValue];
        [obj setValue:@(currentQty + qty) forKey:@"quantity"];
    } else {
        NSManagedObject *obj = [NSEntityDescription
            insertNewObjectForEntityForName:@"CartItemEntity"
                     inManagedObjectContext:self.context];
        [obj setValue:@(product.productId) forKey:@"productId"];
        [obj setValue:product.title        forKey:@"title"];
        [obj setValue:@(product.price)     forKey:@"price"];
        [obj setValue:product.imageURL     forKey:@"imageURL"];
        [obj setValue:@(qty)               forKey:@"quantity"];
    }
    [self saveContext];
}

- (void)removeItemAtIndex:(NSInteger)index {
    NSArray *items = [self fetchCartObjects];
    if (index < (NSInteger)items.count) {
        [self.context deleteObject:items[index]];
        [self saveContext];
    }
}

- (void)updateItem:(ECCartItem *)item quantity:(NSInteger)qty {
    NSFetchRequest *req = [NSFetchRequest fetchRequestWithEntityName:@"CartItemEntity"];
    req.predicate = [NSPredicate predicateWithFormat:@"productId == %ld",
                     (long)item.product.productId];
    NSArray *results = [self.context executeFetchRequest:req error:nil];
    if (results.count > 0) {
        [results.firstObject setValue:@(qty) forKey:@"quantity"];
        [self saveContext];
    }
}

- (void)clearCart {
    for (NSManagedObject *obj in [self fetchCartObjects]) {
        [self.context deleteObject:obj];
    }
    [self saveContext];
}

- (NSArray<ECCartItem *> *)cartItems {
    NSMutableArray *items = [NSMutableArray array];
    for (NSManagedObject *obj in [self fetchCartObjects]) {
        ECProduct *product = [[ECProduct alloc] init];
        product.productId = [[obj valueForKey:@"productId"] integerValue];
        product.title     = [obj valueForKey:@"title"];
        product.price     = [[obj valueForKey:@"price"] doubleValue];
        product.imageURL  = [obj valueForKey:@"imageURL"];

        ECCartItem *item  = [[ECCartItem alloc] init];
        item.product  = product;
        item.quantity = [[obj valueForKey:@"quantity"] integerValue];
        [items addObject:item];
    }
    return [items copy];
}

- (double)totalPrice {
    double total = 0;
    for (ECCartItem *item in [self cartItems]) {
        total += item.subtotal;
    }
    return total;
}

- (NSInteger)totalItemCount {
    NSInteger count = 0;
    for (ECCartItem *item in [self cartItems]) {
        count += item.quantity;
    }
    return count;
}

// MARK: - Private

- (NSArray *)fetchCartObjects {
    NSFetchRequest *req = [NSFetchRequest fetchRequestWithEntityName:@"CartItemEntity"];
    req.sortDescriptors = @[[NSSortDescriptor sortDescriptorWithKey:@"title" ascending:YES]];
    return [self.context executeFetchRequest:req error:nil] ?: @[];
}

- (void)saveContext {
    NSError *error;
    if (![self.context save:&error]) {
        NSLog(@"Save failed: %@", error);
    }
}

@end
```

---

## 92.7 CartViewController

### ECCartViewController.m

```objc
#import "ECCartViewController.h"
#import "ECCartStore.h"

static NSString * const kCartCellID = @"CartCell";

@interface ECCartViewController () <UITableViewDataSource, UITableViewDelegate>
@property (nonatomic, strong) UITableView *tableView;
@property (nonatomic, strong) UILabel     *totalLabel;
@property (nonatomic, strong) UIButton    *checkoutButton;
@property (nonatomic, strong) NSArray<ECCartItem *> *cartItems;
@end

@implementation ECCartViewController

- (void)viewDidLoad {
    [super viewDidLoad];
    self.title = @"ตะกร้าสินค้า";
    self.view.backgroundColor = [UIColor systemBackgroundColor];
    [self setupTableView];
    [self setupBottomBar];
    [self reloadCart];

    self.navigationItem.rightBarButtonItem =
        [[UIBarButtonItem alloc] initWithTitle:@"ล้างตะกร้า"
                                         style:UIBarButtonItemStylePlain
                                        target:self
                                        action:@selector(clearCart)];
}

- (void)setupTableView {
    self.tableView = [[UITableView alloc] initWithFrame:self.view.bounds
                                                  style:UITableViewStyleInsetGrouped];
    self.tableView.autoresizingMask =
        UIViewAutoresizingFlexibleWidth | UIViewAutoresizingFlexibleHeight;
    self.tableView.dataSource = self;
    self.tableView.delegate   = self;
    [self.tableView registerClass:[UITableViewCell class]
           forCellReuseIdentifier:kCartCellID];
    [self.view addSubview:self.tableView];
}

- (void)setupBottomBar {
    UIView *bar = [[UIView alloc] init];
    bar.backgroundColor = [UIColor secondarySystemBackgroundColor];
    bar.translatesAutoresizingMaskIntoConstraints = NO;
    [self.view addSubview:bar];

    self.totalLabel = [[UILabel alloc] init];
    self.totalLabel.font = [UIFont boldSystemFontOfSize:18];
    self.totalLabel.translatesAutoresizingMaskIntoConstraints = NO;

    self.checkoutButton = [UIButton buttonWithType:UIButtonTypeSystem];
    [self.checkoutButton setTitle:@"สั่งซื้อ" forState:UIControlStateNormal];
    self.checkoutButton.backgroundColor = [UIColor systemGreenColor];
    [self.checkoutButton setTitleColor:[UIColor whiteColor] forState:UIControlStateNormal];
    self.checkoutButton.layer.cornerRadius = 10;
    self.checkoutButton.translatesAutoresizingMaskIntoConstraints = NO;
    [self.checkoutButton addTarget:self action:@selector(checkout)
                  forControlEvents:UIControlEventTouchUpInside];

    [bar addSubview:self.totalLabel];
    [bar addSubview:self.checkoutButton];

    [NSLayoutConstraint activateConstraints:@[
        [bar.leadingAnchor constraintEqualToAnchor:self.view.leadingAnchor],
        [bar.trailingAnchor constraintEqualToAnchor:self.view.trailingAnchor],
        [bar.bottomAnchor constraintEqualToAnchor:self.view.safeAreaLayoutGuide.bottomAnchor],
        [bar.heightAnchor constraintEqualToConstant:70],
        [self.totalLabel.leadingAnchor constraintEqualToAnchor:bar.leadingAnchor constant:16],
        [self.totalLabel.centerYAnchor constraintEqualToAnchor:bar.centerYAnchor],
        [self.checkoutButton.trailingAnchor constraintEqualToAnchor:bar.trailingAnchor constant:-16],
        [self.checkoutButton.centerYAnchor constraintEqualToAnchor:bar.centerYAnchor],
        [self.checkoutButton.widthAnchor constraintEqualToConstant:100],
        [self.checkoutButton.heightAnchor constraintEqualToConstant:44]
    ]];
}

- (void)reloadCart {
    self.cartItems = [[ECCartStore sharedStore] cartItems];
    double total = [[ECCartStore sharedStore] totalPrice];
    self.totalLabel.text = [NSString stringWithFormat:@"รวม: $%.2f", total];
    [self.tableView reloadData];
}

- (NSInteger)tableView:(UITableView *)tv numberOfRowsInSection:(NSInteger)section {
    return self.cartItems.count;
}

- (UITableViewCell *)tableView:(UITableView *)tv cellForRowAtIndexPath:(NSIndexPath *)ip {
    UITableViewCell *cell = [tv dequeueReusableCellWithIdentifier:kCartCellID forIndexPath:ip];
    ECCartItem *item = self.cartItems[ip.row];
    cell.textLabel.text = item.product.title;
    cell.detailTextLabel.text = [NSString stringWithFormat:@"%@ x%ld = $%.2f",
                                  item.product.formattedPrice,
                                  (long)item.quantity,
                                  item.subtotal];
    return cell;
}

- (void)tableView:(UITableView *)tv
    commitEditingStyle:(UITableViewCellEditingStyle)style
     forRowAtIndexPath:(NSIndexPath *)ip {
    if (style == UITableViewCellEditingStyleDelete) {
        [[ECCartStore sharedStore] removeItemAtIndex:ip.row];
        [self reloadCart];
    }
}

- (void)clearCart {
    [[ECCartStore sharedStore] clearCart];
    [self reloadCart];
}

- (void)checkout {
    UIAlertController *alert = [UIAlertController
        alertControllerWithTitle:@"สั่งซื้อสำเร็จ"
                         message:@"ขอบคุณที่ใช้บริการ!"
                  preferredStyle:UIAlertControllerStyleAlert];
    [alert addAction:[UIAlertAction actionWithTitle:@"ตกลง"
                                              style:UIAlertActionStyleDefault
                                            handler:^(UIAlertAction *a) {
        [[ECCartStore sharedStore] clearCart];
        [self reloadCart];
    }]];
    [self presentViewController:alert animated:YES completion:nil];
}

@end
```

---

## 92.8 Search ด้วย UISearchController

UISearchController ถูก setup ไว้ใน `ECProductListViewController` แล้ว ส่วนนี้แสดง pattern เพิ่มเติมสำหรับ **Scope Bar** (กรองตามหมวดหมู่)

```objc
// เพิ่มใน setupSearchController
self.searchController.searchBar.scopeButtonTitles =
    @[@"ทั้งหมด", @"electronics", @"clothing", @"jewelery"];
self.searchController.searchBar.delegate = self;

// Implement delegate
- (void)searchBar:(UISearchBar *)searchBar selectedScopeButtonIndexDidChange:(NSInteger)idx {
    [self filterProductsWithQuery:searchBar.text scopeIndex:idx];
}

- (void)filterProductsWithQuery:(NSString *)query scopeIndex:(NSInteger)idx {
    NSMutableArray *result = [self.allProducts mutableCopy];

    // กรองตาม scope
    if (idx > 0) {
        NSArray *scopes = @[@"", @"electronics", @"clothing", @"jewelery"];
        NSString *cat = scopes[idx];
        NSPredicate *catPred = [NSPredicate predicateWithFormat:
                                @"category CONTAINS[cd] %@", cat];
        [result filterUsingPredicate:catPred];
    }

    // กรองตาม keyword
    if (query.length > 0) {
        NSPredicate *qPred = [NSPredicate predicateWithFormat:
                              @"title CONTAINS[cd] %@", query];
        [result filterUsingPredicate:qPred];
    }

    self.displayProducts = [result copy];
    [self.collectionView reloadData];
}
```

---

## 92.9 Pull-to-Refresh + Pagination

### Pull-to-Refresh (ใช้ UIRefreshControl)

ถูก setup ไว้ใน `ECProductListViewController` แล้ว ที่สำคัญคือต้องเรียก `[self.refreshControl endRefreshing]` ใน completion block เสมอ

### Pagination ด้วย Prefetching

```objc
// เพิ่ม protocol ใน class declaration
@interface ECProductListViewController ()
    <UICollectionViewDataSourcePrefetching>

// ใน viewDidLoad
self.collectionView.prefetchDataSource = self;
self.pageSize = 10;
self.currentPage = 0;

// เพิ่ม property
@property (nonatomic, assign) BOOL hasMorePages;

// Load page แรก
- (void)loadPage:(NSInteger)page {
    if (self.isLoading || !self.hasMorePages) return;
    self.isLoading = YES;

    [[ECNetworkManager sharedManager] fetchProductsWithCompletion:
     ^(NSArray *products, NSError *error) {
        dispatch_async(dispatch_get_main_queue(), ^{
            self.isLoading = NO;
            [self.refreshControl endRefreshing];
            if (error || !products) return;

            // Simulate pagination (FakeStore ส่งทุก item มาพร้อมกัน)
            NSInteger start = page * self.pageSize;
            NSInteger end   = MIN(start + self.pageSize, (NSInteger)products.count);
            if (start >= (NSInteger)products.count) {
                self.hasMorePages = NO;
                return;
            }
            NSArray *pageItems = [products subarrayWithRange:NSMakeRange(start, end - start)];
            NSMutableArray *all = [self.allProducts mutableCopy] ?: [NSMutableArray array];
            [all addObjectsFromArray:pageItems];
            self.allProducts     = [all copy];
            self.displayProducts = self.allProducts;
            self.currentPage++;
            [self.collectionView reloadData];
        });
    }];
}

// Prefetch delegate — trigger pagination ก่อนถึง item สุดท้าย
- (void)collectionView:(UICollectionView *)cv
       prefetchItemsAtIndexPaths:(NSArray<NSIndexPath *> *)indexPaths {
    NSInteger maxItem = self.displayProducts.count - 1;
    for (NSIndexPath *ip in indexPaths) {
        if (ip.item >= maxItem - 2) {
            [self loadPage:self.currentPage];
            break;
        }
    }
}
```

---

## 92.10 แบบฝึกหัด

### แบบฝึกหัดที่ 1: Wishlist ด้วย NSUserDefaults

เพิ่มฟีเจอร์ Wishlist ให้กับแอป โดยใช้ `NSUserDefaults` บันทึก product ID และแสดง heart icon บน `ECProductCell`

```objc
// ECWishlistManager.h
@interface ECWishlistManager : NSObject
+ (instancetype)sharedManager;
- (void)toggleProductId:(NSInteger)productId;
- (BOOL)isProductInWishlist:(NSInteger)productId;
- (NSArray<NSNumber *> *)wishlistedIds;
@end

// ECWishlistManager.m
@implementation ECWishlistManager
static NSString * const kWishlistKey = @"wishlist_ids";

+ (instancetype)sharedManager {
    static ECWishlistManager *shared = nil;
    static dispatch_once_t t;
    dispatch_once(&t, ^{ shared = [[ECWishlistManager alloc] init]; });
    return shared;
}

- (void)toggleProductId:(NSInteger)productId {
    NSMutableArray *ids = [[[NSUserDefaults standardUserDefaults]
                            arrayForKey:kWishlistKey] mutableCopy]
                          ?: [NSMutableArray array];
    NSNumber *num = @(productId);
    if ([ids containsObject:num]) [ids removeObject:num];
    else [ids addObject:num];
    [[NSUserDefaults standardUserDefaults] setObject:[ids copy] forKey:kWishlistKey];
}

- (BOOL)isProductInWishlist:(NSInteger)productId {
    NSArray *ids = [[NSUserDefaults standardUserDefaults] arrayForKey:kWishlistKey];
    return [ids containsObject:@(productId)];
}

- (NSArray<NSNumber *> *)wishlistedIds {
    return [[NSUserDefaults standardUserDefaults] arrayForKey:kWishlistKey] ?: @[];
}
@end
```

**งานที่ต้องทำ**: เพิ่ม heart button บน `ECProductCell`, แสดง filled heart ถ้าอยู่ใน wishlist, สร้าง `ECWishlistViewController` แสดงสินค้าที่ถูก wishlist

---

### แบบฝึกหัดที่ 2: Filter & Sort Panel

สร้าง Bottom Sheet สำหรับ filter และ sort สินค้า

```objc
// ECFilterViewController.h
typedef void(^ECFilterCompletion)(NSString * _Nullable category,
                                  NSString * _Nullable sortKey,
                                  BOOL ascending);

@interface ECFilterViewController : UIViewController
@property (nonatomic, copy) ECFilterCompletion onApply;
@end

// ECFilterViewController.m — สิ่งที่ต้องทำ:
// 1. UIPickerView สำหรับเลือก category
//    (Electronics, Clothing, Jewelery, ทั้งหมด)
// 2. UISegmentedControl สำหรับเลือก sort:
//    ราคา ต่ำ→สูง / ราคา สูง→ต่ำ / ชื่อ A→Z / Rating
// 3. ปุ่ม "นำไปใช้" เรียก onApply block
// 4. ใน ECProductListViewController apply sort:
//    NSSortDescriptor *sort = [NSSortDescriptor
//        sortDescriptorWithKey:sortKey ascending:ascending];
//    self.displayProducts = [self.displayProducts
//        sortedArrayUsingDescriptors:@[sort]];
```

---

### แบบฝึกหัดที่ 3: Order History ด้วย Core Data

บันทึกประวัติการสั่งซื้อลงใน Core Data

```objc
// เพิ่ม Entity ใน xcdatamodeld:
// Entity: OrderEntity
//   orderId     : UUID (String)
//   orderDate   : Date
//   totalAmount : Double
//   itemsJSON   : String  (JSON array ของ cart items)

// ECOrderStore.h
@interface ECOrderStore : NSObject
+ (instancetype)sharedStore;
- (void)saveOrderFromCartItems:(NSArray<ECCartItem *> *)items total:(double)total;
- (NSArray<NSDictionary *> *)allOrders;
@end

// ใน ECCartViewController checkout:
- (void)checkout {
    NSArray *items = [[ECCartStore sharedStore] cartItems];
    double total   = [[ECCartStore sharedStore] totalPrice];
    [[ECOrderStore sharedStore] saveOrderFromCartItems:items total:total];
    [[ECCartStore sharedStore] clearCart];
    // แสดง confirmation แล้ว pop ไป ECOrderHistoryViewController
}

// ECOrderHistoryViewController แสดง:
// - วันที่สั่ง
// - ยอดรวม
// - รายการสินค้าในออเดอร์
```

---

## สรุปบทที่ 92

| ส่วนประกอบ | เทคโนโลยีที่ใช้ |
|---|---|
| Architecture | MVVM + Coordinator |
| Network | NSURLSession + JSON |
| UI List | UICollectionView + FlowLayout |
| Search | UISearchController + NSPredicate |
| Pagination | Prefetch API |
| Cart Storage | Core Data |
| Wishlist | NSUserDefaults |

E-Commerce App นี้ครอบคลุม pattern ที่ใช้จริงในแอปสมัยใหม่ ฝึกทำแบบฝึกหัดทั้ง 3 ข้อเพื่อเสริมทักษะ Core Data, UX design, และ business logic ก่อนไปบทถัดไป

---

*ตอนที่ 93: Code Review & Refactoring Best Practices →*
