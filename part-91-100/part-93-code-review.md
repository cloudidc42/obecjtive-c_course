# ส่วนที่ 93: Code Review และ Refactoring ใน Objective-C

## บทนำ

การ Code Review และ Refactoring เป็นทักษะที่สำคัญมากสำหรับนักพัฒนา iOS มืออาชีพ การรีวิวโค้ดช่วยให้ทีมค้นพบบั๊ก ปรับปรุงคุณภาพ และแบ่งปันความรู้ ส่วน Refactoring ช่วยให้โค้ดที่มีอยู่แล้วมีโครงสร้างที่ดีขึ้นโดยไม่เปลี่ยนพฤติกรรมภายนอก

---

## 93.1 Code Review Best Practices

### หลักการพื้นฐานของ Code Review

Code Review ที่ดีควรโฟกัสที่การปรับปรุงคุณภาพโค้ด ไม่ใช่การวิจารณ์ผู้เขียน ต่อไปนี้คือหลักการสำคัญ:

**1. ตรวจสอบความถูกต้องของตรรกะ (Correctness)**
- โค้ดทำในสิ่งที่ตั้งใจหรือเปล่า?
- มี edge cases ที่ไม่ได้จัดการหรือเปล่า?
- การจัดการ error ถูกต้องหรือเปล่า?

**2. ตรวจสอบความสามารถในการอ่าน (Readability)**
- ชื่อตัวแปร เมธอด และคลาสสื่อความหมายชัดเจนหรือเปล่า?
- โค้ดอธิบายตัวเองได้หรือเปล่า?
- มี comment ในที่ที่จำเป็นหรือเปล่า?

**3. ตรวจสอบประสิทธิภาพ (Performance)**
- มี memory leak หรือเปล่า?
- มีการใช้ algorithm ที่เหมาะสมหรือเปล่า?
- มีการทำงานที่ไม่จำเป็นบน main thread หรือเปล่า?

**4. ตรวจสอบความปลอดภัย (Security)**
- มีการตรวจสอบ input หรือเปล่า?
- ข้อมูลสำคัญถูกจัดเก็บอย่างปลอดภัยหรือเปล่า?

### วิธีให้ Feedback ที่ดี

```
// ตัวอย่าง Feedback ที่ไม่ดี:
// "โค้ดนี้แย่มาก"
// "ทำไมถึงเขียนแบบนี้?"

// ตัวอย่าง Feedback ที่ดี:
// "เมธอดนี้ยาวเกินไป (100 บรรทัด) อาจแยกออกเป็นเมธอดย่อยๆ ได้
//  เพื่อให้อ่านง่ายขึ้น เช่น extractValidationLogic และ processData"
// "พิจารณาใช้ NSCache แทน NSDictionary เพราะจัดการ memory ได้ดีกว่า"
```

### Checklist สำหรับ Code Review

```objc
// ตรวจสอบ Memory Management
// ✅ ใช้ weak reference ที่เหมาะสม (เพื่อป้องกัน retain cycle)
// ✅ ไม่มี strong reference cycle ใน block
// ✅ delegate เป็น weak property

// ตัวอย่างที่ถูกต้อง:
@interface UserViewController : UIViewController
@property (nonatomic, weak) id<UserViewControllerDelegate> delegate;
@end

// ตัวอย่างที่ถูกต้องสำหรับ block:
__weak typeof(self) weakSelf = self;
[self.networkManager fetchDataWithCompletion:^(NSData *data, NSError *error) {
    __strong typeof(weakSelf) strongSelf = weakSelf;
    if (!strongSelf) return;
    [strongSelf handleData:data error:error];
}];
```

---

## 93.2 Code Smells ใน Objective-C

### 1. God Class (คลาสที่ทำทุกอย่าง)

God Class คือคลาสที่มีความรับผิดชอบมากเกินไป เป็นหนึ่งในปัญหาที่พบบ่อยที่สุดใน iOS development

```objc
// ตัวอย่าง God Class - แบบไม่ดี
@interface AppManager : NSObject

// User management
- (void)loginUser:(NSString *)username password:(NSString *)password;
- (void)logoutUser;
- (User *)currentUser;

// Network
- (void)fetchDataFromURL:(NSURL *)url completion:(void(^)(NSData *))completion;
- (void)uploadImage:(UIImage *)image completion:(void(^)(BOOL))completion;

// Database
- (void)saveUserToDatabase:(User *)user;
- (void)deleteUserFromDatabase:(User *)user;
- (NSArray *)fetchAllUsers;

// UI Management
- (void)showAlert:(NSString *)message;
- (void)showLoadingIndicator;
- (void)hideLoadingIndicator;
- (void)navigateToViewController:(UIViewController *)vc;

// Analytics
- (void)trackEvent:(NSString *)eventName;
- (void)setUserProperty:(NSString *)value forKey:(NSString *)key;

// Push Notifications
- (void)registerForPushNotifications;
- (void)handlePushNotification:(NSDictionary *)userInfo;

// ... และอีกหลายสิบเมธอด

@end
```

**วิธีแก้ไข**: แยกความรับผิดชอบออกเป็นคลาสต่างๆ ตาม Single Responsibility Principle

```objc
// หลัง Refactoring - แบบดี
@interface AuthManager : NSObject
- (void)loginUser:(NSString *)username password:(NSString *)password completion:(void(^)(User *, NSError *))completion;
- (void)logoutUser;
- (User *)currentUser;
@end

@interface NetworkManager : NSObject
- (void)fetchDataFromURL:(NSURL *)url completion:(void(^)(NSData *, NSError *))completion;
- (void)uploadImage:(UIImage *)image completion:(void(^)(BOOL, NSError *))completion;
@end

@interface DatabaseManager : NSObject
- (void)saveUser:(User *)user;
- (void)deleteUser:(User *)user;
- (NSArray<User *> *)fetchAllUsers;
@end

@interface AnalyticsManager : NSObject
- (void)trackEvent:(NSString *)eventName properties:(NSDictionary *)properties;
- (void)setUserProperty:(NSString *)value forKey:(NSString *)key;
@end
```

### 2. Massive View Controller (MVC = Massive View Controller)

ปัญหาคลาสสิกของ iOS development คือ ViewController ที่มีโค้ดมากเกินไป

```objc
// ตัวอย่าง Massive View Controller - แบบไม่ดี (2000+ บรรทัด)
@implementation UserListViewController

// มี properties มากมาย
@synthesize users = _users;
@synthesize filteredUsers = _filteredUsers;
@synthesize searchText = _searchText;
// ... อีก 20+ properties

- (void)viewDidLoad {
    [super viewDidLoad];
    // setup UI (100 บรรทัด)
    // setup networking (80 บรรทัด)  
    // setup database (60 บรรทัด)
    // setup notifications (40 บรรทัด)
}

// UITableViewDataSource methods (200 บรรทัด)
// UITableViewDelegate methods (150 บรรทัด)
// UISearchBarDelegate methods (100 บรรทัด)
// Networking code (300 บรรทัด)
// Data parsing code (200 บรรทัด)
// Business logic (400 บรรทัด)
// ... etc

@end
```

**วิธีแก้ไข**: แยก responsibility ออกไปด้วย patterns ต่างๆ

```objc
// UserListDataSource.h - แยก DataSource ออกไป
@interface UserListDataSource : NSObject <UITableViewDataSource>
@property (nonatomic, copy) NSArray<User *> *users;
- (instancetype)initWithTableView:(UITableView *)tableView;
@end

// UserListDataSource.m
@implementation UserListDataSource

- (NSInteger)tableView:(UITableView *)tableView numberOfRowsInSection:(NSInteger)section {
    return self.users.count;
}

- (UITableViewCell *)tableView:(UITableView *)tableView cellForRowAtIndexPath:(NSIndexPath *)indexPath {
    UserCell *cell = [tableView dequeueReusableCellWithIdentifier:@"UserCell" forIndexPath:indexPath];
    User *user = self.users[indexPath.row];
    [cell configureWithUser:user];
    return cell;
}

@end

// UserListViewModel.h - แยก Business Logic ออกไป
@interface UserListViewModel : NSObject
@property (nonatomic, readonly) NSArray<User *> *users;
@property (nonatomic, copy) void (^usersDidUpdate)(void);
@property (nonatomic, copy) void (^errorDidOccur)(NSError *error);

- (void)fetchUsers;
- (void)searchUsersWithQuery:(NSString *)query;
- (void)deleteUserAtIndex:(NSInteger)index;
@end

// UserListViewModel.m
@implementation UserListViewModel {
    NSArray<User *> *_allUsers;
    UserRepository *_repository;
}

- (instancetype)init {
    self = [super init];
    if (self) {
        _repository = [[UserRepository alloc] init];
    }
    return self;
}

- (void)fetchUsers {
    [_repository fetchAllUsersWithCompletion:^(NSArray<User *> *users, NSError *error) {
        if (error) {
            if (self.errorDidOccur) {
                self.errorDidOccur(error);
            }
            return;
        }
        _allUsers = users;
        _users = users;
        if (self.usersDidUpdate) {
            self.usersDidUpdate();
        }
    }];
}

- (void)searchUsersWithQuery:(NSString *)query {
    if (query.length == 0) {
        _users = _allUsers;
    } else {
        _users = [_allUsers filteredArrayUsingPredicate:
            [NSPredicate predicateWithFormat:@"name CONTAINS[cd] %@", query]];
    }
    if (self.usersDidUpdate) {
        self.usersDidUpdate();
    }
}

@end

// UserListViewController.m - ตอนนี้สั้นและอ่านง่ายกว่าเดิมมาก
@implementation UserListViewController {
    UserListViewModel *_viewModel;
    UserListDataSource *_dataSource;
}

- (void)viewDidLoad {
    [super viewDidLoad];
    [self setupViewModel];
    [self setupDataSource];
    [_viewModel fetchUsers];
}

- (void)setupViewModel {
    _viewModel = [[UserListViewModel alloc] init];
    
    __weak typeof(self) weakSelf = self;
    _viewModel.usersDidUpdate = ^{
        weakSelf.dataSource.users = weakSelf.viewModel.users;
        [weakSelf.tableView reloadData];
    };
    _viewModel.errorDidOccur = ^(NSError *error) {
        [weakSelf showErrorAlert:error];
    };
}

- (void)setupDataSource {
    _dataSource = [[UserListDataSource alloc] initWithTableView:self.tableView];
    self.tableView.dataSource = _dataSource;
}

@end
```

### 3. Deep Nesting (การซ้อนกันลึกเกินไป)

```objc
// แบบไม่ดี - Pyramid of Doom
- (void)processUserData:(NSDictionary *)data {
    if (data) {
        if (data[@"user"]) {
            NSDictionary *userDict = data[@"user"];
            if (userDict[@"name"]) {
                NSString *name = userDict[@"name"];
                if (name.length > 0) {
                    if (userDict[@"email"]) {
                        NSString *email = userDict[@"email"];
                        if ([email containsString:@"@"]) {
                            // ในที่สุดก็ถึงตรงนี้!
                            [self createUserWithName:name email:email];
                        } else {
                            NSLog(@"Invalid email");
                        }
                    } else {
                        NSLog(@"No email");
                    }
                } else {
                    NSLog(@"Empty name");
                }
            } else {
                NSLog(@"No name");
            }
        } else {
            NSLog(@"No user data");
        }
    } else {
        NSLog(@"No data");
    }
}
```

**วิธีแก้ไข**: ใช้ Early Return (Guard Clause)

```objc
// แบบดี - Early Return
- (void)processUserData:(NSDictionary *)data {
    if (!data) {
        NSLog(@"No data");
        return;
    }
    
    NSDictionary *userDict = data[@"user"];
    if (!userDict) {
        NSLog(@"No user data");
        return;
    }
    
    NSString *name = userDict[@"name"];
    if (name.length == 0) {
        NSLog(@"Empty or missing name");
        return;
    }
    
    NSString *email = userDict[@"email"];
    if (![email containsString:@"@"]) {
        NSLog(@"Invalid or missing email");
        return;
    }
    
    [self createUserWithName:name email:email];
}
```

### 4. Long Method (เมธอดที่ยาวเกินไป)

```objc
// แบบไม่ดี - เมธอดที่ยาวมาก
- (void)viewDidLoad {
    [super viewDidLoad];
    
    // Setup Navigation Bar (20 บรรทัด)
    self.navigationItem.title = @"Users";
    UIBarButtonItem *addButton = [[UIBarButtonItem alloc] 
        initWithBarButtonSystemItem:UIBarButtonSystemItemAdd 
        target:self 
        action:@selector(addUser)];
    self.navigationItem.rightBarButtonItem = addButton;
    // ...
    
    // Setup Table View (30 บรรทัด)
    self.tableView.rowHeight = UITableViewAutomaticDimension;
    self.tableView.estimatedRowHeight = 60;
    // ...
    
    // Register Cells (15 บรรทัด)
    [self.tableView registerNib:[UINib nibWithNibName:@"UserCell" bundle:nil] 
        forCellReuseIdentifier:@"UserCell"];
    // ...
    
    // Setup Search Bar (25 บรรทัด)
    UISearchController *searchController = [[UISearchController alloc] initWithSearchResultsController:nil];
    // ...
    
    // Load Data (10 บรรทัด)
    [self fetchUsers];
}
```

**วิธีแก้ไข**: Extract Method

```objc
// แบบดี - แยกเป็นเมธอดย่อย
- (void)viewDidLoad {
    [super viewDidLoad];
    [self setupNavigationBar];
    [self setupTableView];
    [self registerCells];
    [self setupSearchController];
    [self fetchUsers];
}

- (void)setupNavigationBar {
    self.navigationItem.title = @"Users";
    UIBarButtonItem *addButton = [[UIBarButtonItem alloc] 
        initWithBarButtonSystemItem:UIBarButtonSystemItemAdd 
        target:self 
        action:@selector(addUser)];
    self.navigationItem.rightBarButtonItem = addButton;
}

- (void)setupTableView {
    self.tableView.rowHeight = UITableViewAutomaticDimension;
    self.tableView.estimatedRowHeight = 60;
    self.tableView.separatorStyle = UITableViewCellSeparatorStyleSingleLine;
}

- (void)registerCells {
    [self.tableView registerNib:[UINib nibWithNibName:@"UserCell" bundle:nil] 
        forCellReuseIdentifier:@"UserCell"];
}

- (void)setupSearchController {
    UISearchController *searchController = [[UISearchController alloc] initWithSearchResultsController:nil];
    searchController.searchResultsUpdater = self;
    searchController.obscuresBackgroundDuringPresentation = NO;
    self.navigationItem.searchController = searchController;
}
```

### 5. Duplicate Code (โค้ดที่ซ้ำกัน)

```objc
// แบบไม่ดี - โค้ดซ้ำ
- (void)showSuccessAlert {
    UIAlertController *alert = [UIAlertController alertControllerWithTitle:@"สำเร็จ"
                                                                   message:@"บันทึกข้อมูลเรียบร้อย"
                                                            preferredStyle:UIAlertControllerStyleAlert];
    UIAlertAction *action = [UIAlertAction actionWithTitle:@"ตกลง"
                                                     style:UIAlertActionStyleDefault
                                                   handler:nil];
    [alert addAction:action];
    [self presentViewController:alert animated:YES completion:nil];
}

- (void)showErrorAlert {
    UIAlertController *alert = [UIAlertController alertControllerWithTitle:@"ผิดพลาด"
                                                                   message:@"เกิดข้อผิดพลาด"
                                                            preferredStyle:UIAlertControllerStyleAlert];
    UIAlertAction *action = [UIAlertAction actionWithTitle:@"ตกลง"
                                                     style:UIAlertActionStyleDefault
                                                   handler:nil];
    [alert addAction:action];
    [self presentViewController:alert animated:YES completion:nil];
}
```

**วิธีแก้ไข**: Extract Method พร้อม Parameters

```objc
// แบบดี - ใช้เมธอดร่วมกัน
- (void)showAlertWithTitle:(NSString *)title message:(NSString *)message {
    UIAlertController *alert = [UIAlertController alertControllerWithTitle:title
                                                                   message:message
                                                            preferredStyle:UIAlertControllerStyleAlert];
    UIAlertAction *action = [UIAlertAction actionWithTitle:@"ตกลง"
                                                     style:UIAlertActionStyleDefault
                                                   handler:nil];
    [alert addAction:action];
    [self presentViewController:alert animated:YES completion:nil];
}

- (void)showSuccessAlert {
    [self showAlertWithTitle:@"สำเร็จ" message:@"บันทึกข้อมูลเรียบร้อย"];
}

- (void)showErrorAlertWithMessage:(NSString *)message {
    [self showAlertWithTitle:@"ผิดพลาด" message:message];
}
```

---

## 93.3 Refactoring Techniques

### Extract Method

การแยกส่วนของโค้ดออกมาเป็นเมธอดใหม่

```objc
// ก่อน Refactoring
- (CGFloat)calculateTotalPrice {
    CGFloat subtotal = 0;
    for (Product *product in self.cartItems) {
        subtotal += product.price * product.quantity;
    }
    
    // Calculate discount
    CGFloat discount = 0;
    if (subtotal > 1000) {
        discount = subtotal * 0.1; // 10% discount
    } else if (subtotal > 500) {
        discount = subtotal * 0.05; // 5% discount
    }
    
    // Calculate tax
    CGFloat tax = (subtotal - discount) * 0.07; // 7% VAT
    
    return subtotal - discount + tax;
}
```

```objc
// หลัง Refactoring - Extract Method
- (CGFloat)calculateSubtotal {
    CGFloat subtotal = 0;
    for (Product *product in self.cartItems) {
        subtotal += product.price * product.quantity;
    }
    return subtotal;
}

- (CGFloat)calculateDiscountForSubtotal:(CGFloat)subtotal {
    if (subtotal > 1000) {
        return subtotal * 0.1;
    } else if (subtotal > 500) {
        return subtotal * 0.05;
    }
    return 0;
}

- (CGFloat)calculateTaxForAmount:(CGFloat)amount {
    return amount * 0.07; // 7% VAT
}

- (CGFloat)calculateTotalPrice {
    CGFloat subtotal = [self calculateSubtotal];
    CGFloat discount = [self calculateDiscountForSubtotal:subtotal];
    CGFloat taxableAmount = subtotal - discount;
    CGFloat tax = [self calculateTaxForAmount:taxableAmount];
    return taxableAmount + tax;
}
```

### Extract Class

การแยกส่วนของคลาสออกมาเป็นคลาสใหม่

```objc
// ก่อน Refactoring - Person มีข้อมูลที่อยู่รวมอยู่ด้วย
@interface Person : NSObject
@property (nonatomic, copy) NSString *firstName;
@property (nonatomic, copy) NSString *lastName;
@property (nonatomic, copy) NSString *street;
@property (nonatomic, copy) NSString *city;
@property (nonatomic, copy) NSString *province;
@property (nonatomic, copy) NSString *postalCode;

- (NSString *)fullName;
- (NSString *)fullAddress;
@end
```

```objc
// หลัง Refactoring - แยก Address ออกมา
@interface Address : NSObject
@property (nonatomic, copy) NSString *street;
@property (nonatomic, copy) NSString *city;
@property (nonatomic, copy) NSString *province;
@property (nonatomic, copy) NSString *postalCode;

- (NSString *)formattedAddress;
@end

@implementation Address
- (NSString *)formattedAddress {
    return [NSString stringWithFormat:@"%@ %@ %@ %@", 
            self.street, self.city, self.province, self.postalCode];
}
@end

@interface Person : NSObject
@property (nonatomic, copy) NSString *firstName;
@property (nonatomic, copy) NSString *lastName;
@property (nonatomic, strong) Address *address;

- (NSString *)fullName;
@end

@implementation Person
- (NSString *)fullName {
    return [NSString stringWithFormat:@"%@ %@", self.firstName, self.lastName];
}
@end
```

### Replace Conditional with Polymorphism

แทนที่ switch/if-else ด้วย polymorphism

```objc
// ก่อน Refactoring - ใช้ switch statement
typedef NS_ENUM(NSInteger, ShapeType) {
    ShapeTypeCircle,
    ShapeTypeRectangle,
    ShapeTypeTriangle
};

@interface Shape : NSObject
@property (nonatomic, assign) ShapeType type;
@property (nonatomic, assign) CGFloat width;
@property (nonatomic, assign) CGFloat height;
@property (nonatomic, assign) CGFloat radius;
@end

- (CGFloat)calculateArea:(Shape *)shape {
    switch (shape.type) {
        case ShapeTypeCircle:
            return M_PI * shape.radius * shape.radius;
        case ShapeTypeRectangle:
            return shape.width * shape.height;
        case ShapeTypeTriangle:
            return 0.5 * shape.width * shape.height;
        default:
            return 0;
    }
}
```

```objc
// หลัง Refactoring - ใช้ Polymorphism
@interface Shape : NSObject
- (CGFloat)area;
- (CGFloat)perimeter;
- (NSString *)description;
@end

@interface Circle : Shape
@property (nonatomic, assign) CGFloat radius;
- (instancetype)initWithRadius:(CGFloat)radius;
@end

@implementation Circle
- (instancetype)initWithRadius:(CGFloat)radius {
    self = [super init];
    if (self) {
        _radius = radius;
    }
    return self;
}

- (CGFloat)area {
    return M_PI * _radius * _radius;
}

- (CGFloat)perimeter {
    return 2 * M_PI * _radius;
}
@end

@interface Rectangle : Shape
@property (nonatomic, assign) CGFloat width;
@property (nonatomic, assign) CGFloat height;
- (instancetype)initWithWidth:(CGFloat)width height:(CGFloat)height;
@end

@implementation Rectangle
- (instancetype)initWithWidth:(CGFloat)width height:(CGFloat)height {
    self = [super init];
    if (self) {
        _width = width;
        _height = height;
    }
    return self;
}

- (CGFloat)area {
    return _width * _height;
}

- (CGFloat)perimeter {
    return 2 * (_width + _height);
}
@end

@interface Triangle : Shape
@property (nonatomic, assign) CGFloat base;
@property (nonatomic, assign) CGFloat height;
- (instancetype)initWithBase:(CGFloat)base height:(CGFloat)height;
@end

@implementation Triangle
- (CGFloat)area {
    return 0.5 * _base * _height;
}
@end

// การใช้งาน - ไม่ต้องใช้ switch อีกต่อไป
- (CGFloat)totalAreaOfShapes:(NSArray<Shape *> *)shapes {
    CGFloat total = 0;
    for (Shape *shape in shapes) {
        total += [shape area]; // polymorphism จัดการให้
    }
    return total;
}
```

### Null Object Pattern

แทนที่การ check nil ด้วย Null Object

```objc
// ก่อน Refactoring - ต้อง check nil ทุกที่
- (void)displayUserInfo:(User *)user {
    if (user) {
        self.nameLabel.text = user.name;
        self.emailLabel.text = user.email;
        self.profileImageView.image = user.profileImage;
        self.followersLabel.text = [NSString stringWithFormat:@"%ld followers", user.followersCount];
    } else {
        self.nameLabel.text = @"Guest";
        self.emailLabel.text = @"Not logged in";
        self.profileImageView.image = [UIImage imageNamed:@"default_avatar"];
        self.followersLabel.text = @"";
    }
}
```

```objc
// หลัง Refactoring - Null Object Pattern
@protocol UserProtocol <NSObject>
@property (nonatomic, readonly) NSString *name;
@property (nonatomic, readonly) NSString *email;
@property (nonatomic, readonly) UIImage *profileImage;
@property (nonatomic, readonly) NSInteger followersCount;
@property (nonatomic, readonly) BOOL isAuthenticated;
@end

@interface User : NSObject <UserProtocol>
@property (nonatomic, copy) NSString *name;
@property (nonatomic, copy) NSString *email;
@property (nonatomic, strong) UIImage *profileImage;
@property (nonatomic, assign) NSInteger followersCount;
@end

@implementation User
- (BOOL)isAuthenticated { return YES; }
@end

// NullUser - ส่งคืนค่า default แทน nil
@interface NullUser : NSObject <UserProtocol>
+ (instancetype)sharedInstance;
@end

@implementation NullUser

+ (instancetype)sharedInstance {
    static NullUser *instance = nil;
    static dispatch_once_t onceToken;
    dispatch_once(&onceToken, ^{
        instance = [[NullUser alloc] init];
    });
    return instance;
}

- (NSString *)name { return @"Guest"; }
- (NSString *)email { return @"Not logged in"; }
- (UIImage *)profileImage { return [UIImage imageNamed:@"default_avatar"]; }
- (NSInteger)followersCount { return 0; }
- (BOOL)isAuthenticated { return NO; }

@end

// การใช้งาน - ไม่ต้อง check nil
- (void)displayUserInfo:(id<UserProtocol>)user {
    self.nameLabel.text = user.name;
    self.emailLabel.text = user.email;
    self.profileImageView.image = user.profileImage;
    if (user.followersCount > 0) {
        self.followersLabel.text = [NSString stringWithFormat:@"%ld followers", user.followersCount];
    } else {
        self.followersLabel.text = @"";
    }
}

// UserManager ส่งคืน NullUser แทน nil
- (id<UserProtocol>)currentUser {
    if (self.isLoggedIn) {
        return self.authenticatedUser;
    }
    return [NullUser sharedInstance];
}
```

### Decompose Conditional

แยก conditional logic ที่ซับซ้อนออกมาให้อ่านง่ายขึ้น

```objc
// ก่อน Refactoring
- (CGFloat)calculateShippingCost:(Order *)order {
    CGFloat shippingCost = 0;
    
    if (order.totalAmount >= 1000 && 
        [order.customer.membershipLevel isEqualToString:@"premium"] && 
        order.shippingMethod == ShippingMethodStandard &&
        order.items.count <= 5 &&
        ![order.shippingAddress.province isEqualToString:@"เชียงใหม่"] &&
        ![order.shippingAddress.province isEqualToString:@"ภูเก็ต"]) {
        shippingCost = 0; // Free shipping
    } else if (order.shippingMethod == ShippingMethodExpress) {
        shippingCost = 200;
    } else {
        shippingCost = 50;
    }
    
    return shippingCost;
}
```

```objc
// หลัง Refactoring - Decompose Conditional
- (BOOL)qualifiesForFreeShipping:(Order *)order {
    BOOL hasMinimumAmount = order.totalAmount >= 1000;
    BOOL isPremiumMember = [order.customer.membershipLevel isEqualToString:@"premium"];
    BOOL isStandardShipping = order.shippingMethod == ShippingMethodStandard;
    BOOL isSmallOrder = order.items.count <= 5;
    BOOL isMainlandAddress = [self isMainlandAddress:order.shippingAddress];
    
    return hasMinimumAmount && isPremiumMember && isStandardShipping && 
           isSmallOrder && isMainlandAddress;
}

- (BOOL)isMainlandAddress:(Address *)address {
    NSArray *remoteProvinces = @[@"เชียงใหม่", @"ภูเก็ต"];
    return ![remoteProvinces containsObject:address.province];
}

- (CGFloat)calculateShippingCost:(Order *)order {
    if ([self qualifiesForFreeShipping:order]) {
        return 0;
    }
    
    if (order.shippingMethod == ShippingMethodExpress) {
        return 200;
    }
    
    return 50;
}
```

---

## 93.4 Before/After Refactoring Examples

### ตัวอย่างที่ 1: Network Request Handler

```objc
// ก่อน Refactoring - โค้ดยุ่งเหยิง
- (void)loadUserProfile {
    NSURL *url = [NSURL URLWithString:@"https://api.example.com/user/profile"];
    NSURLRequest *request = [NSURLRequest requestWithURL:url];
    
    [NSURLConnection sendAsynchronousRequest:request 
                                       queue:[NSOperationQueue mainQueue] 
                           completionHandler:^(NSURLResponse *response, NSData *data, NSError *connectionError) {
        if (connectionError) {
            UIAlertController *alert = [UIAlertController alertControllerWithTitle:@"Error" message:connectionError.localizedDescription preferredStyle:UIAlertControllerStyleAlert];
            [alert addAction:[UIAlertAction actionWithTitle:@"OK" style:UIAlertActionStyleDefault handler:nil]];
            [self presentViewController:alert animated:YES completion:nil];
            return;
        }
        
        NSError *jsonError;
        NSDictionary *json = [NSJSONSerialization JSONObjectWithData:data options:0 error:&jsonError];
        if (jsonError) {
            UIAlertController *alert = [UIAlertController alertControllerWithTitle:@"Error" message:jsonError.localizedDescription preferredStyle:UIAlertControllerStyleAlert];
            [alert addAction:[UIAlertAction actionWithTitle:@"OK" style:UIAlertActionStyleDefault handler:nil]];
            [self presentViewController:alert animated:YES completion:nil];
            return;
        }
        
        self.nameLabel.text = json[@"name"];
        self.emailLabel.text = json[@"email"];
        self.ageLabel.text = [NSString stringWithFormat:@"%@", json[@"age"]];
    }];
}
```

```objc
// หลัง Refactoring - Clean Architecture

// UserProfileService.h
@interface UserProfileService : NSObject
- (void)fetchUserProfileWithCompletion:(void(^)(UserProfile *profile, NSError *error))completion;
@end

// UserProfile.h
@interface UserProfile : NSObject
@property (nonatomic, copy) NSString *name;
@property (nonatomic, copy) NSString *email;
@property (nonatomic, assign) NSInteger age;

+ (instancetype)profileFromDictionary:(NSDictionary *)dictionary;
@end

// UserProfile.m
@implementation UserProfile
+ (instancetype)profileFromDictionary:(NSDictionary *)dictionary {
    UserProfile *profile = [[UserProfile alloc] init];
    profile.name = dictionary[@"name"];
    profile.email = dictionary[@"email"];
    profile.age = [dictionary[@"age"] integerValue];
    return profile;
}
@end

// UserProfileService.m
@implementation UserProfileService

- (void)fetchUserProfileWithCompletion:(void(^)(UserProfile *profile, NSError *error))completion {
    NSURL *url = [NSURL URLWithString:@"https://api.example.com/user/profile"];
    NSURLSessionDataTask *task = [[NSURLSession sharedSession] dataTaskWithURL:url 
                                                            completionHandler:^(NSData *data, NSURLResponse *response, NSError *error) {
        if (error) {
            dispatch_async(dispatch_get_main_queue(), ^{
                completion(nil, error);
            });
            return;
        }
        
        NSError *parseError;
        NSDictionary *json = [NSJSONSerialization JSONObjectWithData:data options:0 error:&parseError];
        if (parseError) {
            dispatch_async(dispatch_get_main_queue(), ^{
                completion(nil, parseError);
            });
            return;
        }
        
        UserProfile *profile = [UserProfile profileFromDictionary:json];
        dispatch_async(dispatch_get_main_queue(), ^{
            completion(profile, nil);
        });
    }];
    [task resume];
}

@end

// UserProfileViewController.m - ตอนนี้สะอาดมาก
@implementation UserProfileViewController {
    UserProfileService *_service;
}

- (void)viewDidLoad {
    [super viewDidLoad];
    _service = [[UserProfileService alloc] init];
    [self loadUserProfile];
}

- (void)loadUserProfile {
    [self showLoadingIndicator];
    
    __weak typeof(self) weakSelf = self;
    [_service fetchUserProfileWithCompletion:^(UserProfile *profile, NSError *error) {
        [weakSelf hideLoadingIndicator];
        
        if (error) {
            [weakSelf showErrorAlertWithMessage:error.localizedDescription];
            return;
        }
        
        [weakSelf updateUIWithProfile:profile];
    }];
}

- (void)updateUIWithProfile:(UserProfile *)profile {
    self.nameLabel.text = profile.name;
    self.emailLabel.text = profile.email;
    self.ageLabel.text = [NSString stringWithFormat:@"%ld ปี", profile.age];
}

@end
```

### ตัวอย่างที่ 2: Form Validation

```objc
// ก่อน Refactoring
- (BOOL)validateForm {
    BOOL isValid = YES;
    
    if (self.nameTextField.text.length == 0) {
        self.nameErrorLabel.text = @"กรุณากรอกชื่อ";
        self.nameErrorLabel.hidden = NO;
        isValid = NO;
    } else {
        self.nameErrorLabel.hidden = YES;
    }
    
    if (self.emailTextField.text.length == 0) {
        self.emailErrorLabel.text = @"กรุณากรอกอีเมล";
        self.emailErrorLabel.hidden = NO;
        isValid = NO;
    } else if (![self.emailTextField.text containsString:@"@"]) {
        self.emailErrorLabel.text = @"รูปแบบอีเมลไม่ถูกต้อง";
        self.emailErrorLabel.hidden = NO;
        isValid = NO;
    } else {
        self.emailErrorLabel.hidden = YES;
    }
    
    if (self.passwordTextField.text.length < 8) {
        self.passwordErrorLabel.text = @"รหัสผ่านต้องมีอย่างน้อย 8 ตัวอักษร";
        self.passwordErrorLabel.hidden = NO;
        isValid = NO;
    } else {
        self.passwordErrorLabel.hidden = YES;
    }
    
    return isValid;
}
```

```objc
// หลัง Refactoring - แยก Validator ออกมา

// FormValidator.h
@interface ValidationResult : NSObject
@property (nonatomic, assign) BOOL isValid;
@property (nonatomic, copy) NSString *errorMessage;
+ (instancetype)validResult;
+ (instancetype)invalidResultWithMessage:(NSString *)message;
@end

@interface FormValidator : NSObject
- (ValidationResult *)validateName:(NSString *)name;
- (ValidationResult *)validateEmail:(NSString *)email;
- (ValidationResult *)validatePassword:(NSString *)password;
@end

// FormValidator.m
@implementation ValidationResult
+ (instancetype)validResult {
    ValidationResult *result = [[ValidationResult alloc] init];
    result.isValid = YES;
    return result;
}

+ (instancetype)invalidResultWithMessage:(NSString *)message {
    ValidationResult *result = [[ValidationResult alloc] init];
    result.isValid = NO;
    result.errorMessage = message;
    return result;
}
@end

@implementation FormValidator

- (ValidationResult *)validateName:(NSString *)name {
    if (name.length == 0) {
        return [ValidationResult invalidResultWithMessage:@"กรุณากรอกชื่อ"];
    }
    return [ValidationResult validResult];
}

- (ValidationResult *)validateEmail:(NSString *)email {
    if (email.length == 0) {
        return [ValidationResult invalidResultWithMessage:@"กรุณากรอกอีเมล"];
    }
    
    NSString *emailRegex = @"[A-Z0-9a-z._%+-]+@[A-Za-z0-9.-]+\\.[A-Za-z]{2,}";
    NSPredicate *emailPredicate = [NSPredicate predicateWithFormat:@"SELF MATCHES %@", emailRegex];
    if (![emailPredicate evaluateWithObject:email]) {
        return [ValidationResult invalidResultWithMessage:@"รูปแบบอีเมลไม่ถูกต้อง"];
    }
    
    return [ValidationResult validResult];
}

- (ValidationResult *)validatePassword:(NSString *)password {
    if (password.length < 8) {
        return [ValidationResult invalidResultWithMessage:@"รหัสผ่านต้องมีอย่างน้อย 8 ตัวอักษร"];
    }
    return [ValidationResult validResult];
}

@end

// ViewController - สะอาดและทดสอบง่าย
@implementation RegisterViewController {
    FormValidator *_validator;
}

- (void)viewDidLoad {
    [super viewDidLoad];
    _validator = [[FormValidator alloc] init];
}

- (BOOL)validateForm {
    NSArray *validations = @[
        @{@"result": [_validator validateName:self.nameTextField.text], @"label": self.nameErrorLabel},
        @{@"result": [_validator validateEmail:self.emailTextField.text], @"label": self.emailErrorLabel},
        @{@"result": [_validator validatePassword:self.passwordTextField.text], @"label": self.passwordErrorLabel}
    ];
    
    BOOL allValid = YES;
    for (NSDictionary *validation in validations) {
        ValidationResult *result = validation[@"result"];
        UILabel *errorLabel = validation[@"label"];
        
        if (!result.isValid) {
            errorLabel.text = result.errorMessage;
            errorLabel.hidden = NO;
            allValid = NO;
        } else {
            errorLabel.hidden = YES;
        }
    }
    
    return allValid;
}

@end
```

---

## 93.5 Static Analysis Tools

### Clang Static Analyzer

Clang Analyzer เป็น tool ที่ built-in มากับ Xcode สามารถค้นหาปัญหาได้โดยอัตโนมัติ

**วิธีเปิดใช้งาน Clang Analyzer:**
1. ใน Xcode ไปที่ Product → Analyze (⌘⇧B)
2. หรือเปิดใน Build Settings → Enable Static Analyzer → YES

```objc
// ตัวอย่างที่ Clang Analyzer จะตรวจพบ

// Memory Leak
- (void)example1 {
    NSMutableArray *array = [[NSMutableArray alloc] init];
    // Clang จะแจ้งเตือน: Object leaked
    // ควร autorelease หรือเป็น local variable
}

// Use after free (ใน manual reference counting)
- (void)example2 {
    NSObject *obj = [[NSObject alloc] init];
    [obj release];
    [obj description]; // Clang: Use-after-release
}

// Null dereference
- (void)example3 {
    NSString *str = nil;
    NSInteger length = [str length]; // Clang: Message sent to nil
    // ไม่ใช่ crash ใน ObjC แต่อาจเป็นบั๊กด้านตรรกะ
}

// Dead store
- (void)example4 {
    NSInteger value = 5;
    value = 10; // Clang: Value stored to 'value' is never read
    NSLog(@"Done");
}
```

### OCLint

OCLint เป็น tool สำหรับตรวจสอบ code quality

**การติดตั้ง OCLint ผ่าน Homebrew:**
```bash
brew install oclint
```

**การใช้งาน OCLint กับ Xcode project:**
```bash
# สร้าง compilation database
xcodebuild -project MyApp.xcodeproj \
           -scheme MyApp \
           -configuration Debug \
           clean build | xcpretty -r json-compilation-database

# รัน OCLint
oclint-json-compilation-database \
    -e Pods \
    -- \
    -report-type html \
    -o oclint_report.html \
    -rc LONG_LINE=120 \
    -rc LONG_METHOD=50 \
    -rc NCSS_METHOD=40
```

**กฎสำคัญของ OCLint:**
```
- CyclomaticComplexity: ซับซ้อนเกินไป (default: > 10)
- LongClass: คลาสยาวเกินไป (default: > 1000 บรรทัด)
- LongMethod: เมธอดยาวเกินไป (default: > 50 บรรทัด)
- HighNPathComplexity: มีเส้นทางการทำงานมากเกินไป (default: > 200)
- TooManyMethods: มีเมธอดมากเกินไป (default: > 30)
- TooManyParameters: มี parameter มากเกินไป (default: > 5)
```

### SwiftLint สำหรับโปรเจกต์ผสม

```bash
# ติดตั้ง
brew install swiftlint

# .swiftlint.yml
disabled_rules:
  - colon
  - comma
  - control_statement

opt_in_rules:
  - empty_count
  - missing_docs

included:
  - Source

excluded:
  - Pods
  - R.generated.swift

line_length:
  warning: 120
  error: 200
  
type_body_length:
  warning: 300
  error: 400

file_length:
  warning: 500
  error: 1000
```

---

## 93.6 Code Style Guide

### Naming Conventions สำหรับ Objective-C

#### Classes
```objc
// ใช้ PascalCase และ prefix 2-3 ตัวอักษร
@interface TKUserProfileViewController : UIViewController  // TK = TeamName
@interface TKAuthManager : NSObject
@interface TKNetworkConstants : NSObject

// ผิด
@interface user_profile_vc : UIViewController  // ห้ามใช้ snake_case
@interface UserProfileVC : UIViewController     // ห้ามใช้ VC ย่อ
```

#### Methods
```objc
// ใช้ camelCase และควรอ่านเป็นประโยคได้
- (void)fetchUserWithID:(NSString *)userID completion:(void(^)(User *, NSError *))completion;
- (BOOL)isValidEmail:(NSString *)email;
- (void)configureWithUser:(User *)user;

// ผิด
- (void)fetch_user:(NSString *)id;  // ห้ามใช้ snake_case
- (void)doIt;                        // ชื่อไม่สื่อความหมาย
- (void)setU:(User *)u;             // ชื่อสั้นเกินไป
```

#### Properties
```objc
@interface User : NSObject

// ใช้ camelCase
@property (nonatomic, copy) NSString *firstName;
@property (nonatomic, copy) NSString *lastName;
@property (nonatomic, strong) NSDate *birthDate;
@property (nonatomic, assign) NSInteger followersCount;
@property (nonatomic, assign, getter=isActive) BOOL active;

// BOOL properties ควรเริ่มด้วย is/has/can
@property (nonatomic, assign) BOOL isLoggedIn;
@property (nonatomic, assign) BOOL hasProfileImage;
@property (nonatomic, assign) BOOL canEdit;

@end
```

#### Constants
```objc
// ใช้ static const สำหรับ type-safe constants
static NSString * const TKAPIBaseURL = @"https://api.example.com";
static CGFloat const TKDefaultCellHeight = 60.0;
static NSInteger const TKMaxRetryCount = 3;

// ห้ามใช้ #define สำหรับ typed constants
#define API_BASE_URL @"https://api.example.com"  // ผิด - ไม่มี type safety
```

#### Enumerations
```objc
// ใช้ NS_ENUM หรือ NS_OPTIONS
typedef NS_ENUM(NSInteger, TKUserRole) {
    TKUserRoleGuest = 0,
    TKUserRoleUser,
    TKUserRoleAdmin,
    TKUserRoleSuperAdmin
};

typedef NS_OPTIONS(NSUInteger, TKPermissions) {
    TKPermissionsNone    = 0,
    TKPermissionsRead    = 1 << 0,  // 1
    TKPermissionsWrite   = 1 << 1,  // 2
    TKPermissionsDelete  = 1 << 2,  // 4
    TKPermissionsAll     = TKPermissionsRead | TKPermissionsWrite | TKPermissionsDelete
};
```

### Header File Organization

```objc
// MyViewController.h - ตัวอย่าง header ที่จัดระเบียบดี

#import <UIKit/UIKit.h>  // Framework imports

@class User;              // Forward declarations
@protocol MyViewControllerDelegate;

NS_ASSUME_NONNULL_BEGIN   // Nullability annotations

// Protocol definition
@protocol MyViewControllerDelegate <NSObject>
@required
- (void)viewController:(MyViewController *)vc didSelectUser:(User *)user;
@optional
- (void)viewControllerDidCancel:(MyViewController *)vc;
@end

// Main interface
@interface MyViewController : UIViewController

// Designated initializer
- (instancetype)initWithUsers:(NSArray<User *> *)users NS_DESIGNATED_INITIALIZER;
- (instancetype)init NS_UNAVAILABLE;

// Public properties
@property (nonatomic, weak, nullable) id<MyViewControllerDelegate> delegate;
@property (nonatomic, readonly) NSArray<User *> *users;

// Public methods
- (void)reload;

@end

NS_ASSUME_NONNULL_END
```

### Implementation File Organization

```objc
// MyViewController.m - การจัดระเบียบ implementation

#import "MyViewController.h"

// Private imports
#import "UserCell.h"
#import "UserViewModel.h"

// Private constants
static NSString * const kCellIdentifier = @"UserCell";
static CGFloat const kRowHeight = 60.0;

// Private category (extension)
@interface MyViewController () <UITableViewDataSource, UITableViewDelegate>
// Private properties
@property (nonatomic, strong) UITableView *tableView;
@property (nonatomic, strong) UserViewModel *viewModel;
@end

@implementation MyViewController

#pragma mark - Lifecycle

- (instancetype)initWithUsers:(NSArray<User *> *)users {
    self = [super init];
    if (self) {
        _users = [users copy];
        _viewModel = [[UserViewModel alloc] initWithUsers:users];
    }
    return self;
}

- (void)viewDidLoad {
    [super viewDidLoad];
    [self setupTableView];
    [self bindViewModel];
}

#pragma mark - Setup

- (void)setupTableView {
    // ...
}

- (void)bindViewModel {
    // ...
}

#pragma mark - UITableViewDataSource

- (NSInteger)tableView:(UITableView *)tableView numberOfRowsInSection:(NSInteger)section {
    return self.viewModel.users.count;
}

- (UITableViewCell *)tableView:(UITableView *)tableView cellForRowAtIndexPath:(NSIndexPath *)indexPath {
    // ...
}

#pragma mark - UITableViewDelegate

- (void)tableView:(UITableView *)tableView didSelectRowAtIndexPath:(NSIndexPath *)indexPath {
    // ...
}

#pragma mark - Actions

- (void)addButtonTapped {
    // ...
}

#pragma mark - Private Methods

- (void)reload {
    [self.tableView reloadData];
}

@end
```

---

## 93.7 แบบฝึกหัด

### แบบฝึกหัดที่ 1: ระบุ Code Smells

อ่านโค้ดต่อไปนี้และระบุ code smells ที่พบ:

```objc
@implementation DataManager

- (void)processData:(NSData *)data type:(NSString *)type userId:(NSString *)userId 
             token:(NSString *)token callback:(void(^)(id result))callback {
    if (data != nil) {
        if (type != nil) {
            if ([type isEqualToString:@"user"]) {
                if (userId != nil) {
                    if (token != nil) {
                        NSError *error;
                        NSDictionary *dict = [NSJSONSerialization JSONObjectWithData:data 
                                                                             options:0 
                                                                               error:&error];
                        if (!error) {
                            User *user = [[User alloc] init];
                            user.name = dict[@"name"];
                            user.email = dict[@"email"];
                            user.age = [dict[@"age"] integerValue];
                            user.address = dict[@"address"];
                            user.phone = dict[@"phone"];
                            user.avatar = dict[@"avatar"];
                            
                            [self saveUser:user];
                            [self updateLastSyncTime];
                            [self sendAnalyticsEvent:@"user_processed"];
                            [self notifyObservers:user];
                            
                            if (callback) callback(user);
                        } else {
                            if (callback) callback(nil);
                        }
                    } else {
                        if (callback) callback(nil);
                    }
                } else {
                    if (callback) callback(nil);
                }
            } else if ([type isEqualToString:@"product"]) {
                // ... โค้ดซ้ำอีก 50 บรรทัด
            } else if ([type isEqualToString:@"order"]) {
                // ... โค้ดซ้ำอีก 50 บรรทัด
            }
        } else {
            if (callback) callback(nil);
        }
    } else {
        if (callback) callback(nil);
    }
}

@end
```

**คำตอบที่คาดหวัง:**
- Deep Nesting (Pyramid of Doom)
- Long Method
- Too Many Parameters
- Duplicate Code (type handling)
- God Method ที่ทำหลายอย่าง
- ควรใช้ Early Return

### แบบฝึกหัดที่ 2: Refactor โค้ด

ทำการ Refactor โค้ดใน Exercise 1 โดย:
1. ใช้ Early Return เพื่อลด nesting
2. แยก parsing logic ออกมา
3. ใช้ polymorphism แทน type checking
4. ลด parameter ลงโดยใช้ object

```objc
// เฉลยตัวอย่าง
@interface DataProcessRequest : NSObject
@property (nonatomic, strong) NSData *data;
@property (nonatomic, copy) NSString *type;
@property (nonatomic, copy) NSString *userId;
@property (nonatomic, copy) NSString *token;
@end

@protocol DataProcessor <NSObject>
- (id)processData:(NSData *)data error:(NSError **)error;
@end

@interface UserDataProcessor : NSObject <DataProcessor>
@end

@implementation UserDataProcessor
- (id)processData:(NSData *)data error:(NSError **)error {
    NSDictionary *dict = [NSJSONSerialization JSONObjectWithData:data options:0 error:error];
    if (!dict) return nil;
    return [User userFromDictionary:dict];
}
@end

@implementation DataManager

- (void)processRequest:(DataProcessRequest *)request 
            completion:(void(^)(id result, NSError *error))completion {
    if (!request.data || !request.type || !request.userId || !request.token) {
        NSError *error = [NSError errorWithDomain:@"DataManagerError" 
                                             code:400 
                                         userInfo:@{NSLocalizedDescriptionKey: @"Invalid request"}];
        completion(nil, error);
        return;
    }
    
    id<DataProcessor> processor = [self processorForType:request.type];
    if (!processor) {
        NSError *error = [NSError errorWithDomain:@"DataManagerError" 
                                             code:404 
                                         userInfo:@{NSLocalizedDescriptionKey: @"Unknown type"}];
        completion(nil, error);
        return;
    }
    
    NSError *processError;
    id result = [processor processData:request.data error:&processError];
    if (processError) {
        completion(nil, processError);
        return;
    }
    
    [self saveResult:result];
    [self updateLastSyncTime];
    [self sendAnalyticsEvent:[NSString stringWithFormat:@"%@_processed", request.type]];
    completion(result, nil);
}

- (id<DataProcessor>)processorForType:(NSString *)type {
    static NSDictionary *processors = nil;
    static dispatch_once_t onceToken;
    dispatch_once(&onceToken, ^{
        processors = @{
            @"user": [[UserDataProcessor alloc] init],
            @"product": [[ProductDataProcessor alloc] init],
            @"order": [[OrderDataProcessor alloc] init]
        };
    });
    return processors[type];
}

@end
```

### แบบฝึกหัดที่ 3: Setup Clang Analyzer

1. เปิด Xcode project ของคุณ
2. ไปที่ Product → Analyze
3. แก้ไขทุก issue ที่ Analyzer พบ
4. ตั้งค่า Build Settings → Enable Static Analyzer → YES เพื่อให้วิเคราะห์ทุกครั้งที่ build

### แบบฝึกหัดที่ 4: สร้าง Style Guide

สร้าง `.editorconfig` และ naming conventions guide สำหรับโปรเจกต์ของคุณ:

```ini
# .editorconfig
root = true

[*]
charset = utf-8
end_of_line = lf
indent_style = space
indent_size = 4
trim_trailing_whitespace = true
insert_final_newline = true

[*.{h,m,mm}]
indent_size = 4

[*.{yml,yaml,json}]
indent_size = 2

[Makefile]
indent_style = tab
```

---

## สรุป

ในส่วนนี้เราได้เรียนรู้:
- หลักการ Code Review ที่ดี
- Code Smells ที่พบบ่อยใน Objective-C
- Refactoring Techniques หลัก (Extract Method, Extract Class, Replace Conditional with Polymorphism, Null Object Pattern)
- การใช้ Static Analysis Tools (Clang Analyzer, OCLint)
- Code Style Guide และ Naming Conventions

การฝึกทำ Code Review และ Refactoring อย่างสม่ำเสมอจะช่วยให้โค้ดของคุณมีคุณภาพดีขึ้นเรื่อยๆ และทำให้ทีมทำงานร่วมกันได้อย่างมีประสิทธิภาพมากขึ้น

---

*ส่วนถัดไป: Part 94 - Open Source Contribution*
