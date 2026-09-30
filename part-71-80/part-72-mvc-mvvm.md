# Part 72: MVC, MVVM, MVP Architecture ใน Objective-C

## บทนำ

Architecture Pattern คือแนวทางการจัดโครงสร้างของโค้ดในระดับสูง ซึ่งแตกต่างจาก Design Patterns ที่แก้ปัญหาระดับ class ในขณะที่ Architecture Patterns กำหนดว่าแต่ละส่วนของแอปควรรับผิดชอบอะไรและสื่อสารกันอย่างไร

การเลือก architecture ที่เหมาะสมส่งผลต่อ:
- ความสามารถในการทดสอบ (Testability)
- ความสามารถในการบำรุงรักษา (Maintainability)
- ความยืดหยุ่น (Flexibility)
- ความง่ายในการทำงานเป็นทีม (Team Scalability)

---

## 72.1 MVC - Model View Controller

MVC คือ architecture ดั้งเดิมของ iOS ที่ Apple สนับสนุนตั้งแต่ต้น ประกอบด้วย 3 ส่วนหลัก:

### ส่วนประกอบ

```
┌─────────┐     Updates      ┌──────────┐
│  Model  │ ────────────────> │   View   │
│         │                  │          │
│  Data   │ <──────────────── │    UI    │
│  Logic  │   User Actions   │          │
└────┬────┘                  └────┬─────┘
     │                            │
     │         ┌──────────┐       │
     └──────── │Controller│ ──────┘
               │          │
               │  Handles │
               │  Logic   │
               └──────────┘
```

### Model Layer

```objc
// User.h - Model
@interface User : NSObject

@property (nonatomic, strong) NSString *userID;
@property (nonatomic, strong) NSString *name;
@property (nonatomic, strong) NSString *email;
@property (nonatomic, strong) NSDate *createdAt;

- (BOOL)isValidEmail;
- (NSDictionary *)toDictionary;
+ (instancetype)userFromDictionary:(NSDictionary *)dict;

@end

// User.m
@implementation User

- (BOOL)isValidEmail {
    NSString *regex = @"[A-Z0-9a-z._%+-]+@[A-Za-z0-9.-]+\\.[A-Za-z]{2,}";
    NSPredicate *predicate = [NSPredicate predicateWithFormat:@"SELF MATCHES %@", regex];
    return [predicate evaluateWithObject:self.email];
}

- (NSDictionary *)toDictionary {
    return @{
        @"userID": self.userID ?: @"",
        @"name": self.name ?: @"",
        @"email": self.email ?: @""
    };
}

+ (instancetype)userFromDictionary:(NSDictionary *)dict {
    User *user = [[User alloc] init];
    user.userID = dict[@"userID"];
    user.name = dict[@"name"];
    user.email = dict[@"email"];
    return user;
}

@end
```

### View Layer

```objc
// UserProfileView.h
@protocol UserProfileViewDelegate <NSObject>
- (void)userProfileViewDidTapEdit:(UIView *)view;
- (void)userProfileViewDidTapSave:(UIView *)view;
@end

@interface UserProfileView : UIView

@property (nonatomic, weak) id<UserProfileViewDelegate> delegate;

// Outlets
@property (nonatomic, strong) UIImageView *avatarImageView;
@property (nonatomic, strong) UILabel *nameLabel;
@property (nonatomic, strong) UILabel *emailLabel;
@property (nonatomic, strong) UIButton *editButton;
@property (nonatomic, strong) UIButton *saveButton;

- (void)configure;

@end

// UserProfileView.m
@implementation UserProfileView

- (instancetype)initWithFrame:(CGRect)frame {
    self = [super initWithFrame:frame];
    if (self) {
        [self configure];
    }
    return self;
}

- (void)configure {
    self.backgroundColor = [UIColor systemBackgroundColor];
    
    // Avatar
    self.avatarImageView = [[UIImageView alloc] init];
    self.avatarImageView.contentMode = UIViewContentModeScaleAspectFill;
    self.avatarImageView.clipsToBounds = YES;
    self.avatarImageView.layer.cornerRadius = 40;
    self.avatarImageView.layer.borderWidth = 2;
    self.avatarImageView.layer.borderColor = [UIColor systemBlueColor].CGColor;
    self.avatarImageView.translatesAutoresizingMaskIntoConstraints = NO;
    
    // Name Label
    self.nameLabel = [[UILabel alloc] init];
    self.nameLabel.font = [UIFont boldSystemFontOfSize:20];
    self.nameLabel.textAlignment = NSTextAlignmentCenter;
    self.nameLabel.translatesAutoresizingMaskIntoConstraints = NO;
    
    // Email Label
    self.emailLabel = [[UILabel alloc] init];
    self.emailLabel.font = [UIFont systemFontOfSize:14];
    self.emailLabel.textColor = [UIColor secondaryLabelColor];
    self.emailLabel.textAlignment = NSTextAlignmentCenter;
    self.emailLabel.translatesAutoresizingMaskIntoConstraints = NO;
    
    // Edit Button
    self.editButton = [UIButton buttonWithType:UIButtonTypeSystem];
    [self.editButton setTitle:@"แก้ไข" forState:UIControlStateNormal];
    [self.editButton addTarget:self action:@selector(editTapped) forControlEvents:UIControlEventTouchUpInside];
    self.editButton.translatesAutoresizingMaskIntoConstraints = NO;
    
    [self addSubview:self.avatarImageView];
    [self addSubview:self.nameLabel];
    [self addSubview:self.emailLabel];
    [self addSubview:self.editButton];
    
    [self setupConstraints];
}

- (void)setupConstraints {
    [NSLayoutConstraint activateConstraints:@[
        [self.avatarImageView.centerXAnchor constraintEqualToAnchor:self.centerXAnchor],
        [self.avatarImageView.topAnchor constraintEqualToAnchor:self.topAnchor constant:20],
        [self.avatarImageView.widthAnchor constraintEqualToConstant:80],
        [self.avatarImageView.heightAnchor constraintEqualToConstant:80],
        
        [self.nameLabel.topAnchor constraintEqualToAnchor:self.avatarImageView.bottomAnchor constant:12],
        [self.nameLabel.centerXAnchor constraintEqualToAnchor:self.centerXAnchor],
        
        [self.emailLabel.topAnchor constraintEqualToAnchor:self.nameLabel.bottomAnchor constant:4],
        [self.emailLabel.centerXAnchor constraintEqualToAnchor:self.centerXAnchor],
        
        [self.editButton.topAnchor constraintEqualToAnchor:self.emailLabel.bottomAnchor constant:16],
        [self.editButton.centerXAnchor constraintEqualToAnchor:self.centerXAnchor]
    ]];
}

- (void)editTapped {
    [self.delegate userProfileViewDidTapEdit:self];
}

@end
```

### Controller Layer

```objc
// UserProfileViewController.h
@interface UserProfileViewController : UIViewController

@property (nonatomic, strong) NSString *userID;

@end

// UserProfileViewController.m
#import "UserProfileViewController.h"
#import "User.h"
#import "UserProfileView.h"
#import "UserService.h"

@interface UserProfileViewController () <UserProfileViewDelegate>

@property (nonatomic, strong) UserProfileView *profileView;
@property (nonatomic, strong) User *currentUser;
@property (nonatomic, strong) UserService *userService;

@end

@implementation UserProfileViewController

- (void)viewDidLoad {
    [super viewDidLoad];
    
    self.userService = [[UserService alloc] init];
    
    self.profileView = [[UserProfileView alloc] initWithFrame:self.view.bounds];
    self.profileView.delegate = self;
    [self.view addSubview:self.profileView];
    
    [self loadUserData];
}

- (void)loadUserData {
    [self.userService fetchUserWithID:self.userID
                           completion:^(User *user, NSError *error) {
        if (error) {
            [self showError:error.localizedDescription];
            return;
        }
        
        self.currentUser = user;
        [self updateViewWithUser:user];
    }];
}

- (void)updateViewWithUser:(User *)user {
    dispatch_async(dispatch_get_main_queue(), ^{
        self.profileView.nameLabel.text = user.name;
        self.profileView.emailLabel.text = user.email;
        self.title = user.name;
    });
}

- (void)showError:(NSString *)message {
    dispatch_async(dispatch_get_main_queue(), ^{
        UIAlertController *alert = [UIAlertController alertControllerWithTitle:@"ข้อผิดพลาด"
                                                                        message:message
                                                                 preferredStyle:UIAlertControllerStyleAlert];
        [alert addAction:[UIAlertAction actionWithTitle:@"ตกลง" style:UIAlertActionStyleDefault handler:nil]];
        [self presentViewController:alert animated:YES completion:nil];
    });
}

#pragma mark - UserProfileViewDelegate

- (void)userProfileViewDidTapEdit:(UIView *)view {
    NSLog(@"Edit button tapped");
    // แสดง edit screen
}

- (void)userProfileViewDidTapSave:(UIView *)view {
    NSLog(@"Save button tapped");
    [self saveUserChanges];
}

- (void)saveUserChanges {
    [self.userService updateUser:self.currentUser
                      completion:^(BOOL success, NSError *error) {
        if (success) {
            NSLog(@"User updated successfully");
        } else {
            [self showError:error.localizedDescription];
        }
    }];
}

@end
```

---

## 72.2 ปัญหา Massive View Controller (MVC)

ใน iOS จริงๆ MVC มักกลายเป็น "Massive View Controller" เพราะ:

```objc
// ❌ ตัวอย่าง Massive View Controller ที่ไม่ดี
@interface BadViewController : UIViewController <UITableViewDelegate,
                                                  UITableViewDataSource,
                                                  UISearchBarDelegate,
                                                  UIImagePickerControllerDelegate,
                                                  NSFetchedResultsControllerDelegate,
                                                  CLLocationManagerDelegate,
                                                  AVAudioPlayerDelegate>

// - เป็นทั้ง data source และ delegate
// - จัดการ business logic
// - จัดการ network calls
// - จัดการ database
// - จัดการ location
// - จัดการ media
// ทำให้ไฟล์มี 2,000+ บรรทัด

@end
```

### แนวทางแก้ไข MVC

```objc
// แยก concerns ออกจาก ViewController

// 1. Data Source แยกออก
@interface UserTableViewDataSource : NSObject <UITableViewDataSource>

@property (nonatomic, strong) NSArray<User *> *users;

@end

@implementation UserTableViewDataSource

- (NSInteger)tableView:(UITableView *)tableView numberOfRowsInSection:(NSInteger)section {
    return self.users.count;
}

- (UITableViewCell *)tableView:(UITableView *)tableView cellForRowAtIndexPath:(NSIndexPath *)indexPath {
    UITableViewCell *cell = [tableView dequeueReusableCellWithIdentifier:@"UserCell" forIndexPath:indexPath];
    User *user = self.users[indexPath.row];
    cell.textLabel.text = user.name;
    cell.detailTextLabel.text = user.email;
    return cell;
}

@end

// 2. Network Service แยกออก
@interface UserService : NSObject

- (void)fetchUsersWithCompletion:(void(^)(NSArray<User *> *users, NSError *error))completion;
- (void)fetchUserWithID:(NSString *)userID completion:(void(^)(User *user, NSError *error))completion;
- (void)updateUser:(User *)user completion:(void(^)(BOOL success, NSError *error))completion;

@end

@implementation UserService

- (void)fetchUsersWithCompletion:(void(^)(NSArray<User *> *users, NSError *error))completion {
    NSURL *url = [NSURL URLWithString:@"https://api.example.com/users"];
    
    [[[NSURLSession sharedSession] dataTaskWithURL:url
        completionHandler:^(NSData *data, NSURLResponse *response, NSError *error) {
        
        if (error) {
            completion(nil, error);
            return;
        }
        
        NSArray *json = [NSJSONSerialization JSONObjectWithData:data options:0 error:nil];
        NSMutableArray<User *> *users = [NSMutableArray array];
        
        for (NSDictionary *dict in json) {
            [users addObject:[User userFromDictionary:dict]];
        }
        
        completion([users copy], nil);
    }] resume];
}

- (void)fetchUserWithID:(NSString *)userID completion:(void(^)(User *user, NSError *error))completion {
    // ดึงข้อมูล user
    User *mockUser = [[User alloc] init];
    mockUser.userID = userID;
    mockUser.name = @"John Doe";
    mockUser.email = @"john@example.com";
    completion(mockUser, nil);
}

- (void)updateUser:(User *)user completion:(void(^)(BOOL success, NSError *error))completion {
    completion(YES, nil);
}

@end
```

---

## 72.3 MVVM - Model View ViewModel

MVVM แก้ปัญหา Massive View Controller โดยแยก presentation logic ออกมาเป็น ViewModel

```
┌─────────┐    Binds    ┌────────────┐    Notifies    ┌─────────┐
│  View   │ ─────────> │ ViewModel  │ ─────────────> │  Model  │
│(VC+View)│ <───────── │            │ <───────────── │  Data   │
│         │  Updates   │ Presentation│    Updates    │  Logic  │
└─────────┘            │    Logic   │               └─────────┘
                        └────────────┘
```

### UserViewModel

```objc
// UserViewModel.h
@interface UserViewModel : NSObject

// Observable properties
@property (nonatomic, strong, readonly) NSString *displayName;
@property (nonatomic, strong, readonly) NSString *displayEmail;
@property (nonatomic, strong, readonly) NSString *memberSinceText;
@property (nonatomic, strong, readonly) UIColor *statusColor;
@property (nonatomic, assign, readonly) BOOL isLoading;
@property (nonatomic, assign, readonly) BOOL hasError;
@property (nonatomic, strong, readonly) NSString *errorMessage;

// Callbacks (Data Binding)
@property (nonatomic, copy) void(^onDataChanged)(void);
@property (nonatomic, copy) void(^onLoadingChanged)(BOOL isLoading);
@property (nonatomic, copy) void(^onError)(NSString *message);

// Initializer
- (instancetype)initWithUserID:(NSString *)userID;

// Commands
- (void)loadUser;
- (void)saveChanges;
- (void)updateName:(NSString *)name;
- (void)updateEmail:(NSString *)email;

// Validation
- (BOOL)isNameValid;
- (BOOL)isEmailValid;
- (BOOL)canSave;

@end
```

```objc
// UserViewModel.m
@interface UserViewModel ()

@property (nonatomic, strong) User *model;
@property (nonatomic, strong) UserService *userService;
@property (nonatomic, strong) NSString *userID;

// Writable internal properties
@property (nonatomic, strong) NSString *pendingName;
@property (nonatomic, strong) NSString *pendingEmail;

// State
@property (nonatomic, assign) BOOL isLoading;
@property (nonatomic, assign) BOOL hasError;
@property (nonatomic, strong) NSString *errorMessage;

@end

@implementation UserViewModel

- (instancetype)initWithUserID:(NSString *)userID {
    self = [super init];
    if (self) {
        _userID = userID;
        _userService = [[UserService alloc] init];
    }
    return self;
}

#pragma mark - Public Properties (Computed)

- (NSString *)displayName {
    return self.pendingName ?: self.model.name ?: @"ไม่ระบุชื่อ";
}

- (NSString *)displayEmail {
    return self.pendingEmail ?: self.model.email ?: @"ไม่ระบุอีเมล";
}

- (NSString *)memberSinceText {
    if (!self.model.createdAt) return @"";
    
    NSDateFormatter *formatter = [[NSDateFormatter alloc] init];
    formatter.dateStyle = NSDateFormatterMediumStyle;
    formatter.locale = [NSLocale localeWithLocaleIdentifier:@"th_TH"];
    
    return [NSString stringWithFormat:@"สมาชิกตั้งแต่ %@", [formatter stringFromDate:self.model.createdAt]];
}

- (UIColor *)statusColor {
    if (self.hasError) return [UIColor systemRedColor];
    if (self.isLoading) return [UIColor systemOrangeColor];
    return [UIColor systemGreenColor];
}

#pragma mark - Commands

- (void)loadUser {
    [self setLoading:YES];
    
    [self.userService fetchUserWithID:self.userID
                           completion:^(User *user, NSError *error) {
        dispatch_async(dispatch_get_main_queue(), ^{
            [self setLoading:NO];
            
            if (error) {
                [self setError:error.localizedDescription];
            } else {
                self.model = user;
                self.pendingName = nil;
                self.pendingEmail = nil;
                [self notifyDataChanged];
            }
        });
    }];
}

- (void)saveChanges {
    if (![self canSave]) return;
    
    // อัพเดต model
    if (self.pendingName) self.model.name = self.pendingName;
    if (self.pendingEmail) self.model.email = self.pendingEmail;
    
    [self setLoading:YES];
    
    [self.userService updateUser:self.model
                      completion:^(BOOL success, NSError *error) {
        dispatch_async(dispatch_get_main_queue(), ^{
            [self setLoading:NO];
            
            if (success) {
                self.pendingName = nil;
                self.pendingEmail = nil;
                [self notifyDataChanged];
            } else {
                [self setError:error.localizedDescription];
            }
        });
    }];
}

- (void)updateName:(NSString *)name {
    self.pendingName = name;
    [self notifyDataChanged];
}

- (void)updateEmail:(NSString *)email {
    self.pendingEmail = email;
    [self notifyDataChanged];
}

#pragma mark - Validation

- (BOOL)isNameValid {
    NSString *name = self.displayName;
    return name.length >= 2 && name.length <= 50;
}

- (BOOL)isEmailValid {
    NSString *email = self.displayEmail;
    NSString *regex = @"[A-Z0-9a-z._%+-]+@[A-Za-z0-9.-]+\\.[A-Za-z]{2,}";
    NSPredicate *predicate = [NSPredicate predicateWithFormat:@"SELF MATCHES %@", regex];
    return [predicate evaluateWithObject:email];
}

- (BOOL)canSave {
    return [self isNameValid] && [self isEmailValid] && !self.isLoading;
}

#pragma mark - Private

- (void)setLoading:(BOOL)loading {
    _isLoading = loading;
    if (self.onLoadingChanged) {
        self.onLoadingChanged(loading);
    }
    [self notifyDataChanged];
}

- (void)setError:(NSString *)message {
    _hasError = YES;
    _errorMessage = message;
    if (self.onError) {
        self.onError(message);
    }
}

- (void)notifyDataChanged {
    if (self.onDataChanged) {
        self.onDataChanged();
    }
}

@end
```

### MVVM ViewController

```objc
// UserProfileViewController (MVVM).m
@interface UserProfileViewControllerMVVM : UIViewController

@property (nonatomic, strong) NSString *userID;

@end

@interface UserProfileViewControllerMVVM ()

@property (nonatomic, strong) UserViewModel *viewModel;

// UI Elements
@property (nonatomic, strong) UILabel *nameLabel;
@property (nonatomic, strong) UILabel *emailLabel;
@property (nonatomic, strong) UILabel *memberSinceLabel;
@property (nonatomic, strong) UITextField *nameTextField;
@property (nonatomic, strong) UITextField *emailTextField;
@property (nonatomic, strong) UIButton *saveButton;
@property (nonatomic, strong) UIActivityIndicatorView *loadingIndicator;
@property (nonatomic, strong) UIView *statusDot;

@end

@implementation UserProfileViewControllerMVVM

- (void)viewDidLoad {
    [super viewDidLoad];
    
    [self setupUI];
    [self setupViewModel];
    [self bindViewModel];
    [self.viewModel loadUser];
}

- (void)setupViewModel {
    self.viewModel = [[UserViewModel alloc] initWithUserID:self.userID];
}

- (void)bindViewModel {
    // Data binding ด้วย blocks
    __weak typeof(self) weakSelf = self;
    
    self.viewModel.onDataChanged = ^{
        [weakSelf updateUI];
    };
    
    self.viewModel.onLoadingChanged = ^(BOOL isLoading) {
        if (isLoading) {
            [weakSelf.loadingIndicator startAnimating];
            weakSelf.saveButton.enabled = NO;
        } else {
            [weakSelf.loadingIndicator stopAnimating];
            weakSelf.saveButton.enabled = weakSelf.viewModel.canSave;
        }
    };
    
    self.viewModel.onError = ^(NSString *message) {
        UIAlertController *alert = [UIAlertController alertControllerWithTitle:@"ข้อผิดพลาด"
                                                                        message:message
                                                                 preferredStyle:UIAlertControllerStyleAlert];
        [alert addAction:[UIAlertAction actionWithTitle:@"ตกลง" style:UIAlertActionStyleDefault handler:nil]];
        [weakSelf presentViewController:alert animated:YES completion:nil];
    };
}

- (void)updateUI {
    self.nameLabel.text = self.viewModel.displayName;
    self.emailLabel.text = self.viewModel.displayEmail;
    self.memberSinceLabel.text = self.viewModel.memberSinceText;
    self.statusDot.backgroundColor = self.viewModel.statusColor;
    self.saveButton.enabled = self.viewModel.canSave;
    
    if (!self.nameTextField.isFirstResponder) {
        self.nameTextField.text = self.viewModel.displayName;
    }
    if (!self.emailTextField.isFirstResponder) {
        self.emailTextField.text = self.viewModel.displayEmail;
    }
}

- (void)setupUI {
    self.view.backgroundColor = [UIColor systemBackgroundColor];
    
    // Status Dot
    self.statusDot = [[UIView alloc] initWithFrame:CGRectMake(0, 0, 12, 12)];
    self.statusDot.layer.cornerRadius = 6;
    self.statusDot.translatesAutoresizingMaskIntoConstraints = NO;
    
    // Name Label
    self.nameLabel = [[UILabel alloc] init];
    self.nameLabel.font = [UIFont boldSystemFontOfSize:24];
    self.nameLabel.translatesAutoresizingMaskIntoConstraints = NO;
    
    // Email Label
    self.emailLabel = [[UILabel alloc] init];
    self.emailLabel.font = [UIFont systemFontOfSize:16];
    self.emailLabel.textColor = [UIColor secondaryLabelColor];
    self.emailLabel.translatesAutoresizingMaskIntoConstraints = NO;
    
    // Member Since Label
    self.memberSinceLabel = [[UILabel alloc] init];
    self.memberSinceLabel.font = [UIFont systemFontOfSize:14];
    self.memberSinceLabel.textColor = [UIColor tertiaryLabelColor];
    self.memberSinceLabel.translatesAutoresizingMaskIntoConstraints = NO;
    
    // Text Fields
    self.nameTextField = [self createTextField:@"ชื่อ"];
    [self.nameTextField addTarget:self action:@selector(nameChanged:) forControlEvents:UIControlEventEditingChanged];
    
    self.emailTextField = [self createTextField:@"อีเมล"];
    self.emailTextField.keyboardType = UIKeyboardTypeEmailAddress;
    [self.emailTextField addTarget:self action:@selector(emailChanged:) forControlEvents:UIControlEventEditingChanged];
    
    // Save Button
    self.saveButton = [UIButton buttonWithType:UIButtonTypeSystem];
    [self.saveButton setTitle:@"บันทึก" forState:UIControlStateNormal];
    [self.saveButton addTarget:self action:@selector(saveTapped) forControlEvents:UIControlEventTouchUpInside];
    self.saveButton.translatesAutoresizingMaskIntoConstraints = NO;
    
    // Loading Indicator
    self.loadingIndicator = [[UIActivityIndicatorView alloc] initWithActivityIndicatorStyle:UIActivityIndicatorViewStyleMedium];
    self.loadingIndicator.hidesWhenStopped = YES;
    self.loadingIndicator.translatesAutoresizingMaskIntoConstraints = NO;
    
    [self.view addSubview:self.statusDot];
    [self.view addSubview:self.nameLabel];
    [self.view addSubview:self.emailLabel];
    [self.view addSubview:self.memberSinceLabel];
    [self.view addSubview:self.nameTextField];
    [self.view addSubview:self.emailTextField];
    [self.view addSubview:self.saveButton];
    [self.view addSubview:self.loadingIndicator];
    
    [NSLayoutConstraint activateConstraints:@[
        [self.statusDot.topAnchor constraintEqualToAnchor:self.view.safeAreaLayoutGuide.topAnchor constant:20],
        [self.statusDot.trailingAnchor constraintEqualToAnchor:self.view.trailingAnchor constant:-20],
        [self.statusDot.widthAnchor constraintEqualToConstant:12],
        [self.statusDot.heightAnchor constraintEqualToConstant:12],
        
        [self.nameLabel.topAnchor constraintEqualToAnchor:self.view.safeAreaLayoutGuide.topAnchor constant:20],
        [self.nameLabel.leadingAnchor constraintEqualToAnchor:self.view.leadingAnchor constant:20],
        
        [self.emailLabel.topAnchor constraintEqualToAnchor:self.nameLabel.bottomAnchor constant:4],
        [self.emailLabel.leadingAnchor constraintEqualToAnchor:self.view.leadingAnchor constant:20],
        
        [self.memberSinceLabel.topAnchor constraintEqualToAnchor:self.emailLabel.bottomAnchor constant:4],
        [self.memberSinceLabel.leadingAnchor constraintEqualToAnchor:self.view.leadingAnchor constant:20],
        
        [self.nameTextField.topAnchor constraintEqualToAnchor:self.memberSinceLabel.bottomAnchor constant:24],
        [self.nameTextField.leadingAnchor constraintEqualToAnchor:self.view.leadingAnchor constant:20],
        [self.nameTextField.trailingAnchor constraintEqualToAnchor:self.view.trailingAnchor constant:-20],
        [self.nameTextField.heightAnchor constraintEqualToConstant:44],
        
        [self.emailTextField.topAnchor constraintEqualToAnchor:self.nameTextField.bottomAnchor constant:12],
        [self.emailTextField.leadingAnchor constraintEqualToAnchor:self.view.leadingAnchor constant:20],
        [self.emailTextField.trailingAnchor constraintEqualToAnchor:self.view.trailingAnchor constant:-20],
        [self.emailTextField.heightAnchor constraintEqualToConstant:44],
        
        [self.saveButton.topAnchor constraintEqualToAnchor:self.emailTextField.bottomAnchor constant:24],
        [self.saveButton.centerXAnchor constraintEqualToAnchor:self.view.centerXAnchor],
        
        [self.loadingIndicator.centerXAnchor constraintEqualToAnchor:self.view.centerXAnchor],
        [self.loadingIndicator.centerYAnchor constraintEqualToAnchor:self.view.centerYAnchor]
    ]];
}

- (UITextField *)createTextField:(NSString *)placeholder {
    UITextField *field = [[UITextField alloc] init];
    field.placeholder = placeholder;
    field.borderStyle = UITextBorderStyleRoundedRect;
    field.translatesAutoresizingMaskIntoConstraints = NO;
    return field;
}

- (void)nameChanged:(UITextField *)sender {
    [self.viewModel updateName:sender.text];
}

- (void)emailChanged:(UITextField *)sender {
    [self.viewModel updateEmail:sender.text];
}

- (void)saveTapped {
    [self.view endEditing:YES];
    [self.viewModel saveChanges];
}

@end
```

---

## 72.4 Data Binding ด้วย KVO

```objc
// KVO-based ViewModel
@interface KVOUserViewModel : NSObject

@property (nonatomic, strong) NSString *name;
@property (nonatomic, strong) NSString *email;
@property (nonatomic, assign) BOOL isLoading;
@property (nonatomic, assign) BOOL isValid;

@end

@implementation KVOUserViewModel

- (void)setName:(NSString *)name {
    _name = name;
    [self validate];
}

- (void)setEmail:(NSString *)email {
    _email = email;
    [self validate];
}

- (void)validate {
    BOOL nameValid = self.name.length >= 2;
    BOOL emailValid = [self.email containsString:@"@"];
    self.isValid = nameValid && emailValid;
}

@end

// ViewController ที่ใช้ KVO
@interface KVOViewController : UIViewController

@end

@interface KVOViewController ()
@property (nonatomic, strong) KVOUserViewModel *viewModel;
@property (nonatomic, strong) UIButton *saveButton;
@end

@implementation KVOViewController

- (void)viewDidLoad {
    [super viewDidLoad];
    
    self.viewModel = [[KVOUserViewModel alloc] init];
    
    // Observer ด้วย KVO
    [self.viewModel addObserver:self
                     forKeyPath:@"isValid"
                        options:NSKeyValueObservingOptionNew
                        context:nil];
    
    [self.viewModel addObserver:self
                     forKeyPath:@"isLoading"
                        options:NSKeyValueObservingOptionNew
                        context:nil];
}

- (void)observeValueForKeyPath:(NSString *)keyPath
                      ofObject:(id)object
                        change:(NSDictionary *)change
                       context:(void *)context {
    
    if ([keyPath isEqualToString:@"isValid"]) {
        BOOL isValid = [change[NSKeyValueChangeNewKey] boolValue];
        dispatch_async(dispatch_get_main_queue(), ^{
            self.saveButton.enabled = isValid;
            self.saveButton.alpha = isValid ? 1.0 : 0.5;
        });
    }
    else if ([keyPath isEqualToString:@"isLoading"]) {
        BOOL isLoading = [change[NSKeyValueChangeNewKey] boolValue];
        dispatch_async(dispatch_get_main_queue(), ^{
            self.saveButton.enabled = !isLoading;
        });
    }
}

- (void)dealloc {
    [self.viewModel removeObserver:self forKeyPath:@"isValid"];
    [self.viewModel removeObserver:self forKeyPath:@"isLoading"];
}

@end
```

---

## 72.5 MVP - Model View Presenter

MVP แตกต่างจาก MVVM ตรงที่ Presenter รู้จัก View ผ่าน Protocol

```
┌─────────────────────────────────┐
│    View (ViewController)         │
│  - Passive (แค่แสดงผล)          │
│  - Delegate events to Presenter │
└──────────┬──────────────────────┘
           │ implements ViewProtocol
           ▼
┌──────────────────────────────────┐
│    Presenter                      │
│  - Handles all logic             │
│  - Knows about View (via protocol)│
│  - Knows about Model             │
└──────────┬──────────────────────┘
           │ uses
           ▼
┌──────────────────────────────────┐
│    Model                          │
│  - Business Logic                │
│  - Data                          │
└──────────────────────────────────┘
```

### View Protocol

```objc
// UserListViewProtocol.h
@protocol UserListViewProtocol <NSObject>

- (void)showLoading;
- (void)hideLoading;
- (void)displayUsers:(NSArray<User *> *)users;
- (void)displayError:(NSString *)message;
- (void)navigateToUserDetail:(User *)user;

@end
```

### Presenter

```objc
// UserListPresenter.h
@interface UserListPresenter : NSObject

@property (nonatomic, weak) id<UserListViewProtocol> view;

- (instancetype)initWithView:(id<UserListViewProtocol>)view;

- (void)viewDidLoad;
- (void)userSelectedAtIndex:(NSInteger)index;
- (void)refreshRequested;
- (void)searchQueryChanged:(NSString *)query;

@end

// UserListPresenter.m
@interface UserListPresenter ()

@property (nonatomic, strong) UserService *userService;
@property (nonatomic, strong) NSArray<User *> *allUsers;
@property (nonatomic, strong) NSArray<User *> *filteredUsers;

@end

@implementation UserListPresenter

- (instancetype)initWithView:(id<UserListViewProtocol>)view {
    self = [super init];
    if (self) {
        _view = view;
        _userService = [[UserService alloc] init];
    }
    return self;
}

- (void)viewDidLoad {
    [self loadUsers];
}

- (void)loadUsers {
    [self.view showLoading];
    
    [self.userService fetchUsersWithCompletion:^(NSArray<User *> *users, NSError *error) {
        dispatch_async(dispatch_get_main_queue(), ^{
            [self.view hideLoading];
            
            if (error) {
                [self.view displayError:error.localizedDescription];
            } else {
                self.allUsers = users;
                self.filteredUsers = users;
                [self.view displayUsers:self.filteredUsers];
            }
        });
    }];
}

- (void)userSelectedAtIndex:(NSInteger)index {
    if (index < 0 || index >= (NSInteger)self.filteredUsers.count) return;
    
    User *user = self.filteredUsers[index];
    [self.view navigateToUserDetail:user];
}

- (void)refreshRequested {
    [self loadUsers];
}

- (void)searchQueryChanged:(NSString *)query {
    if (query.length == 0) {
        self.filteredUsers = self.allUsers;
    } else {
        NSPredicate *predicate = [NSPredicate predicateWithFormat:
            @"name CONTAINS[cd] %@ OR email CONTAINS[cd] %@", query, query];
        self.filteredUsers = [self.allUsers filteredArrayUsingPredicate:predicate];
    }
    
    [self.view displayUsers:self.filteredUsers];
}

@end
```

### View (ViewController implements Protocol)

```objc
// UserListViewController.h
@interface UserListViewController : UIViewController <UserListViewProtocol>
@end

// UserListViewController.m
@interface UserListViewController () <UITableViewDelegate, UITableViewDataSource, UISearchBarDelegate>

@property (nonatomic, strong) UserListPresenter *presenter;
@property (nonatomic, strong) UITableView *tableView;
@property (nonatomic, strong) UISearchBar *searchBar;
@property (nonatomic, strong) UIActivityIndicatorView *loadingIndicator;
@property (nonatomic, strong) NSArray<User *> *users;

@end

@implementation UserListViewController

- (void)viewDidLoad {
    [super viewDidLoad];
    
    [self setupUI];
    
    // Presenter รู้จัก View ผ่าน Protocol
    self.presenter = [[UserListPresenter alloc] initWithView:self];
    [self.presenter viewDidLoad];
}

- (void)setupUI {
    self.title = @"ผู้ใช้งาน";
    
    // Search Bar
    self.searchBar = [[UISearchBar alloc] init];
    self.searchBar.delegate = self;
    self.searchBar.placeholder = @"ค้นหาผู้ใช้...";
    self.navigationItem.titleView = self.searchBar;
    
    // Table View
    self.tableView = [[UITableView alloc] initWithFrame:self.view.bounds style:UITableViewStylePlain];
    self.tableView.delegate = self;
    self.tableView.dataSource = self;
    self.tableView.autoresizingMask = UIViewAutoresizingFlexibleWidth | UIViewAutoresizingFlexibleHeight;
    [self.tableView registerClass:[UITableViewCell class] forCellReuseIdentifier:@"UserCell"];
    
    // Refresh Control
    UIRefreshControl *refreshControl = [[UIRefreshControl alloc] init];
    [refreshControl addTarget:self action:@selector(refreshPulled) forControlEvents:UIControlEventValueChanged];
    self.tableView.refreshControl = refreshControl;
    
    [self.view addSubview:self.tableView];
    
    // Loading
    self.loadingIndicator = [[UIActivityIndicatorView alloc] initWithActivityIndicatorStyle:UIActivityIndicatorViewStyleLarge];
    self.loadingIndicator.center = self.view.center;
    self.loadingIndicator.hidesWhenStopped = YES;
    [self.view addSubview:self.loadingIndicator];
}

#pragma mark - UserListViewProtocol

- (void)showLoading {
    [self.loadingIndicator startAnimating];
    self.tableView.userInteractionEnabled = NO;
}

- (void)hideLoading {
    [self.loadingIndicator stopAnimating];
    [self.tableView.refreshControl endRefreshing];
    self.tableView.userInteractionEnabled = YES;
}

- (void)displayUsers:(NSArray<User *> *)users {
    self.users = users;
    [self.tableView reloadData];
}

- (void)displayError:(NSString *)message {
    UIAlertController *alert = [UIAlertController alertControllerWithTitle:@"ข้อผิดพลาด"
                                                                    message:message
                                                             preferredStyle:UIAlertControllerStyleAlert];
    [alert addAction:[UIAlertAction actionWithTitle:@"ลองใหม่"
                                             style:UIAlertActionStyleDefault
                                           handler:^(UIAlertAction *action) {
        [self.presenter refreshRequested];
    }]];
    [alert addAction:[UIAlertAction actionWithTitle:@"ยกเลิก"
                                             style:UIAlertActionStyleCancel
                                           handler:nil]];
    [self presentViewController:alert animated:YES completion:nil];
}

- (void)navigateToUserDetail:(User *)user {
    // ไปหน้า detail
    NSLog(@"Navigate to user: %@", user.name);
}

#pragma mark - UITableViewDataSource

- (NSInteger)tableView:(UITableView *)tableView numberOfRowsInSection:(NSInteger)section {
    return self.users.count;
}

- (UITableViewCell *)tableView:(UITableView *)tableView cellForRowAtIndexPath:(NSIndexPath *)indexPath {
    UITableViewCell *cell = [tableView dequeueReusableCellWithIdentifier:@"UserCell" forIndexPath:indexPath];
    User *user = self.users[indexPath.row];
    cell.textLabel.text = user.name;
    cell.detailTextLabel.text = user.email;
    cell.accessoryType = UITableViewCellAccessoryDisclosureIndicator;
    return cell;
}

#pragma mark - UITableViewDelegate

- (void)tableView:(UITableView *)tableView didSelectRowAtIndexPath:(NSIndexPath *)indexPath {
    [tableView deselectRowAtIndexPath:indexPath animated:YES];
    [self.presenter userSelectedAtIndex:indexPath.row];
}

#pragma mark - UISearchBarDelegate

- (void)searchBar:(UISearchBar *)searchBar textDidChange:(NSString *)searchText {
    [self.presenter searchQueryChanged:searchText];
}

#pragma mark - Actions

- (void)refreshPulled {
    [self.presenter refreshRequested];
}

@end
```

---

## 72.6 VIPER Architecture (ภาพรวม)

VIPER เป็น architecture ที่ละเอียดกว่า MVP มาก ประกอบด้วย 5 ส่วน:

```
View ←→ Presenter ←→ Interactor ←→ Entity
              ↕
            Router
```

- **V**iew - แสดงผล UI
- **I**nteractor - Business Logic
- **P**resenter - ประสาน View กับ Interactor
- **E**ntity - Model/Data
- **R**outer - Navigation

```objc
// VIPER Module Setup

// 1. Entity (Model)
@interface ArticleEntity : NSObject
@property (nonatomic, strong) NSString *articleID;
@property (nonatomic, strong) NSString *title;
@property (nonatomic, strong) NSString *content;
@property (nonatomic, strong) NSDate *publishedAt;
@end

// 2. Interactor
@protocol ArticleInteractorInput <NSObject>
- (void)fetchArticles;
- (void)fetchArticleById:(NSString *)articleID;
@end

@protocol ArticleInteractorOutput <NSObject>
- (void)didFetchArticles:(NSArray<ArticleEntity *> *)articles;
- (void)didFailWithError:(NSError *)error;
@end

@interface ArticleInteractor : NSObject <ArticleInteractorInput>
@property (nonatomic, weak) id<ArticleInteractorOutput> presenter;
@end

@implementation ArticleInteractor
- (void)fetchArticles {
    // Fetch data (network/database)
    // เรียก presenter.didFetchArticles หรือ presenter.didFailWithError
}
- (void)fetchArticleById:(NSString *)articleID { /* fetch specific */ }
@end

// 3. Presenter
@protocol ArticleViewInput <NSObject>
- (void)showArticles:(NSArray *)viewModels;
- (void)showLoading;
- (void)hideLoading;
- (void)showError:(NSString *)message;
@end

@protocol ArticleViewOutput <NSObject>
- (void)viewLoaded;
- (void)selectedArticle:(NSInteger)index;
@end

@interface ArticlePresenter : NSObject <ArticleViewOutput, ArticleInteractorOutput>
@property (nonatomic, weak) id<ArticleViewInput> view;
@property (nonatomic, strong) ArticleInteractor *interactor;
@property (nonatomic, strong) id router; // ArticleRouter
@end

@implementation ArticlePresenter

- (void)viewLoaded {
    [self.view showLoading];
    [self.interactor fetchArticles];
}

- (void)selectedArticle:(NSInteger)index {
    // [self.router navigateToArticleDetail:article];
}

- (void)didFetchArticles:(NSArray<ArticleEntity *> *)articles {
    [self.view hideLoading];
    // Convert entities to view models
    [self.view showArticles:articles];
}

- (void)didFailWithError:(NSError *)error {
    [self.view hideLoading];
    [self.view showError:error.localizedDescription];
}

@end

// 4. Router
@protocol ArticleRouterInput <NSObject>
- (void)navigateToArticleDetail:(ArticleEntity *)article;
@end

@interface ArticleRouter : NSObject <ArticleRouterInput>
@property (nonatomic, weak) UIViewController *viewController;
+ (UIViewController *)createModule;
@end

@implementation ArticleRouter

+ (UIViewController *)createModule {
    // สร้างและประกอบ VIPER module
    ArticleInteractor *interactor = [[ArticleInteractor alloc] init];
    ArticlePresenter *presenter = [[ArticlePresenter alloc] init];
    ArticleRouter *router = [[ArticleRouter alloc] init];
    
    // VC ที่ conform ArticleViewInput
    UIViewController *view = [[UIViewController alloc] init]; // ใช้ VC จริงๆ
    
    // เชื่อมโยงกัน
    presenter.interactor = interactor;
    presenter.router = router;
    interactor.presenter = presenter;
    // presenter.view = view;
    router.viewController = view;
    
    return view;
}

- (void)navigateToArticleDetail:(ArticleEntity *)article {
    // push/present detail VC
}

@end
```

---

## 72.7 Clean Architecture

Clean Architecture แบ่งออกเป็น concentric rings:

```
┌─────────────────────────────────────┐
│  Frameworks & Drivers (UI, DB, Web) │
│  ┌────────────────────────────────┐ │
│  │  Interface Adapters            │ │
│  │  (Controllers, Presenters, GW) │ │
│  │  ┌────────────────────────┐   │ │
│  │  │  Application Business  │   │ │
│  │  │  Rules (Use Cases)     │   │ │
│  │  │  ┌──────────────────┐  │   │ │
│  │  │  │  Enterprise Logic │  │   │ │
│  │  │  │  (Entities)       │  │   │ │
│  │  │  └──────────────────┘  │   │ │
│  │  └────────────────────────┘   │ │
│  └────────────────────────────────┘ │
└─────────────────────────────────────┘
```

**กฎสำคัญ**: Dependency ชี้เข้าข้างใน (Dependency Rule) - Code ชั้นในต้องไม่รู้จักชั้นนอก

```objc
// Use Case (Application Business Rules)
@protocol GetUserUseCase <NSObject>
- (void)executeWithUserID:(NSString *)userID completion:(void(^)(User *, NSError *))completion;
@end

@interface GetUserUseCaseImpl : NSObject <GetUserUseCase>

@property (nonatomic, strong) id<UserRepository> repository;

@end

@implementation GetUserUseCaseImpl

- (void)executeWithUserID:(NSString *)userID completion:(void(^)(User *, NSError *))completion {
    // Business logic
    if (!userID || userID.length == 0) {
        NSError *error = [NSError errorWithDomain:@"AppDomain" 
                                             code:400 
                                         userInfo:@{NSLocalizedDescriptionKey: @"UserID ไม่ถูกต้อง"}];
        completion(nil, error);
        return;
    }
    
    [self.repository findUserById:userID completion:completion];
}

@end

// Repository Protocol (Gateway)
@protocol UserRepository <NSObject>
- (void)findUserById:(NSString *)userID completion:(void(^)(User *, NSError *))completion;
- (void)saveUser:(User *)user completion:(void(^)(BOOL, NSError *))completion;
- (void)getAllUsersWithCompletion:(void(^)(NSArray<User *> *, NSError *))completion;
@end

// Repository Implementation (Infrastructure)
@interface UserRepositoryImpl : NSObject <UserRepository>
@property (nonatomic, strong) id dataSource; // Network or Database
@end

@implementation UserRepositoryImpl

- (void)findUserById:(NSString *)userID completion:(void(^)(User *, NSError *))completion {
    // ติดต่อ data source
    // เป็นชั้นนอก ที่ชั้นในไม่รู้จัก
    completion(nil, nil);
}

- (void)saveUser:(User *)user completion:(void(^)(BOOL, NSError *))completion {
    completion(YES, nil);
}

- (void)getAllUsersWithCompletion:(void(^)(NSArray<User *> *, NSError *))completion {
    completion(@[], nil);
}

@end
```

---

## 72.8 ตัวอย่างการ Refactor จาก MVC เป็น MVVM

### ก่อน Refactor (MVC แบบ Massive)

```objc
// ❌ Before: Massive View Controller
@implementation NewsViewController

- (void)viewDidLoad {
    [super viewDidLoad];
    
    // โหลดข่าว
    NSURL *url = [NSURL URLWithString:@"https://api.news.com/articles"];
    [[[NSURLSession sharedSession] dataTaskWithURL:url completionHandler:^(NSData *data, NSURLResponse *response, NSError *error) {
        NSArray *json = [NSJSONSerialization JSONObjectWithData:data options:0 error:nil];
        
        NSMutableArray *articles = [NSMutableArray array];
        for (NSDictionary *dict in json) {
            NewsArticle *article = [[NewsArticle alloc] init];
            article.title = dict[@"title"];
            article.content = dict[@"body"];
            article.publishDate = [NSDate dateWithTimeIntervalSince1970:[dict[@"timestamp"] doubleValue]];
            [articles addObject:article];
        }
        
        dispatch_async(dispatch_get_main_queue(), ^{
            self.articles = articles;
            [self.tableView reloadData];
        });
    }] resume];
}

- (UITableViewCell *)tableView:(UITableView *)tableView cellForRowAtIndexPath:(NSIndexPath *)indexPath {
    UITableViewCell *cell = [tableView dequeueReusableCellWithIdentifier:@"Cell" forIndexPath:indexPath];
    NewsArticle *article = self.articles[indexPath.row];
    
    // Business logic ใน VC
    NSDateFormatter *formatter = [[NSDateFormatter alloc] init];
    formatter.dateStyle = NSDateFormatterMediumStyle;
    NSString *dateString = [formatter stringFromDate:article.publishDate];
    
    cell.textLabel.text = article.title;
    cell.detailTextLabel.text = dateString;
    
    return cell;
}

@end
```

### หลัง Refactor (MVVM)

```objc
// ✅ After: Separate ViewModel

// NewsArticleViewModel.h
@interface NewsArticleViewModel : NSObject

@property (nonatomic, strong, readonly) NSString *title;
@property (nonatomic, strong, readonly) NSString *formattedDate;
@property (nonatomic, strong, readonly) NSString *summary;
@property (nonatomic, assign, readonly) BOOL isBreakingNews;

- (instancetype)initWithArticle:(NewsArticle *)article;

@end

@implementation NewsArticleViewModel

- (instancetype)initWithArticle:(NewsArticle *)article {
    self = [super init];
    if (self) {
        _title = article.title;
        
        // ย้าย formatting logic มาที่ ViewModel
        NSDateFormatter *formatter = [[NSDateFormatter alloc] init];
        formatter.dateStyle = NSDateFormatterMediumStyle;
        formatter.locale = [NSLocale localeWithLocaleIdentifier:@"th_TH"];
        _formattedDate = [formatter stringFromDate:article.publishDate];
        
        // ตัด content ให้สั้น
        _summary = article.content.length > 100 
            ? [[article.content substringToIndex:100] stringByAppendingString:@"..."]
            : article.content;
        
        // Business logic
        NSTimeInterval age = [[NSDate date] timeIntervalSinceDate:article.publishDate];
        _isBreakingNews = age < 3600; // ข่าวที่โพสต์ใน 1 ชั่วโมง
    }
    return self;
}

@end

// NewsListViewModel.h
@interface NewsListViewModel : NSObject

@property (nonatomic, strong, readonly) NSArray<NewsArticleViewModel *> *articleViewModels;
@property (nonatomic, assign, readonly) BOOL isLoading;

@property (nonatomic, copy) void(^onUpdate)(void);
@property (nonatomic, copy) void(^onError)(NSString *message);

- (void)loadArticles;
- (void)refreshArticles;
- (NSInteger)numberOfArticles;
- (NewsArticleViewModel *)viewModelAtIndex:(NSInteger)index;

@end

@implementation NewsListViewModel

- (void)loadArticles {
    _isLoading = YES;
    if (self.onUpdate) self.onUpdate();
    
    NSURL *url = [NSURL URLWithString:@"https://api.news.com/articles"];
    [[[NSURLSession sharedSession] dataTaskWithURL:url
        completionHandler:^(NSData *data, NSURLResponse *response, NSError *error) {
        
        dispatch_async(dispatch_get_main_queue(), ^{
            _isLoading = NO;
            
            if (error) {
                if (self.onError) self.onError(error.localizedDescription);
                return;
            }
            
            NSArray *json = [NSJSONSerialization JSONObjectWithData:data options:0 error:nil];
            NSMutableArray<NewsArticleViewModel *> *viewModels = [NSMutableArray array];
            
            for (NSDictionary *dict in json) {
                NewsArticle *article = [[NewsArticle alloc] init];
                article.title = dict[@"title"];
                article.content = dict[@"body"];
                article.publishDate = [NSDate dateWithTimeIntervalSince1970:[dict[@"timestamp"] doubleValue]];
                [viewModels addObject:[[NewsArticleViewModel alloc] initWithArticle:article]];
            }
            
            _articleViewModels = [viewModels copy];
            if (self.onUpdate) self.onUpdate();
        });
    }] resume];
}

- (void)refreshArticles {
    [self loadArticles];
}

- (NSInteger)numberOfArticles {
    return self.articleViewModels.count;
}

- (NewsArticleViewModel *)viewModelAtIndex:(NSInteger)index {
    if (index < 0 || index >= (NSInteger)self.articleViewModels.count) return nil;
    return self.articleViewModels[index];
}

@end

// NewsViewController.m (หลัง Refactor - บางมาก!)
@implementation NewsViewControllerRefactored

- (void)viewDidLoad {
    [super viewDidLoad];
    
    self.viewModel = [[NewsListViewModel alloc] init];
    
    __weak typeof(self) weakSelf = self;
    
    self.viewModel.onUpdate = ^{
        [weakSelf.tableView reloadData];
        [weakSelf.tableView.refreshControl endRefreshing];
    };
    
    self.viewModel.onError = ^(NSString *message) {
        UIAlertController *alert = [UIAlertController alertControllerWithTitle:@"Error"
                                                                        message:message
                                                                 preferredStyle:UIAlertControllerStyleAlert];
        [alert addAction:[UIAlertAction actionWithTitle:@"OK" style:UIAlertActionStyleDefault handler:nil]];
        [weakSelf presentViewController:alert animated:YES completion:nil];
    };
    
    [self.viewModel loadArticles];
}

- (NSInteger)tableView:(UITableView *)tableView numberOfRowsInSection:(NSInteger)section {
    return [self.viewModel numberOfArticles];
}

- (UITableViewCell *)tableView:(UITableView *)tableView cellForRowAtIndexPath:(NSIndexPath *)indexPath {
    UITableViewCell *cell = [tableView dequeueReusableCellWithIdentifier:@"Cell" forIndexPath:indexPath];
    NewsArticleViewModel *vm = [self.viewModel viewModelAtIndex:indexPath.row];
    
    cell.textLabel.text = vm.title;
    cell.detailTextLabel.text = vm.formattedDate;
    
    // Breaking news badge
    if (vm.isBreakingNews) {
        cell.textLabel.textColor = [UIColor systemRedColor];
    } else {
        cell.textLabel.textColor = [UIColor labelColor];
    }
    
    return cell;
}

@end
```

---

## 72.9 การเลือก Architecture ที่เหมาะสม

### เปรียบเทียบ

| Architecture | Complexity | Testability | Team Size |
|-------------|-----------|-------------|-----------|
| MVC         | ต่ำ       | ปานกลาง    | เล็ก     |
| MVVM        | ปานกลาง   | สูง         | กลาง-ใหญ่|
| MVP         | ปานกลาง   | สูงมาก      | กลาง-ใหญ่|
| VIPER       | สูงมาก    | สูงที่สุด   | ใหญ่     |
| Clean       | สูงมาก    | สูงที่สุด   | ใหญ่     |

### คำแนะนำ

```objc
// สำหรับแอปขนาดเล็ก-กลาง: MVVM
// ง่ายพอ, testable, flexible

// สำหรับแอปขนาดใหญ่ หรือทีมใหญ่: VIPER หรือ Clean Architecture
// แยก responsibilities ชัดเจน, test ได้ทุกส่วน

// ไม่ว่าจะเลือก architecture ไหน ควรมีหลักการ:
// 1. Single Responsibility - แต่ละ class รับผิดชอบเรื่องเดียว
// 2. Dependency Inversion - ขึ้นกับ abstraction ไม่ใช่ concrete
// 3. Separation of Concerns - แยกความกังวลต่างๆ ออกจากกัน
// 4. Testability - ออกแบบให้ test ได้
```

---

## 72.10 Unit Testing ใน MVVM

```objc
// UserViewModelTests.m
#import <XCTest/XCTest.h>
#import "UserViewModel.h"
#import "MockUserService.h"

@interface UserViewModelTests : XCTestCase

@property (nonatomic, strong) UserViewModel *sut; // System Under Test
@property (nonatomic, strong) MockUserService *mockService;

@end

@implementation UserViewModelTests

- (void)setUp {
    [super setUp];
    
    self.mockService = [[MockUserService alloc] init];
    // Inject mock dependency
    self.sut = [[UserViewModel alloc] initWithUserID:@"user_123"];
    // self.sut.userService = self.mockService;
}

- (void)tearDown {
    self.sut = nil;
    self.mockService = nil;
    [super tearDown];
}

- (void)testDisplayNameReturnsDefaultWhenNoUser {
    // Given: ViewModel ที่ยังไม่โหลดข้อมูล
    
    // When: ดึง displayName
    NSString *name = self.sut.displayName;
    
    // Then: ควรคืนค่า default
    XCTAssertEqualObjects(name, @"ไม่ระบุชื่อ");
}

- (void)testIsEmailValidWithCorrectEmail {
    // Given
    [self.sut updateEmail:@"test@example.com"];
    
    // When
    BOOL isValid = [self.sut isEmailValid];
    
    // Then
    XCTAssertTrue(isValid);
}

- (void)testIsEmailValidWithIncorrectEmail {
    [self.sut updateEmail:@"not-an-email"];
    XCTAssertFalse([self.sut isEmailValid]);
}

- (void)testCanSaveWhenValidData {
    [self.sut updateName:@"John Doe"];
    [self.sut updateEmail:@"john@example.com"];
    XCTAssertTrue([self.sut canSave]);
}

- (void)testCannotSaveWhenInvalidData {
    [self.sut updateName:@"J"]; // too short
    [self.sut updateEmail:@"not-valid"];
    XCTAssertFalse([self.sut canSave]);
}

- (void)testOnDataChangedCalledWhenNameUpdated {
    XCTestExpectation *expectation = [self expectationWithDescription:@"onDataChanged called"];
    
    self.sut.onDataChanged = ^{
        [expectation fulfill];
    };
    
    [self.sut updateName:@"New Name"];
    
    [self waitForExpectationsWithTimeout:1.0 handler:nil];
}

@end

// Mock Service
@interface MockUserService : NSObject

@property (nonatomic, strong) User *stubbedUser;
@property (nonatomic, strong) NSError *stubbedError;
@property (nonatomic, assign) BOOL fetchWasCalled;

@end

@implementation MockUserService

- (void)fetchUserWithID:(NSString *)userID completion:(void(^)(User *user, NSError *error))completion {
    self.fetchWasCalled = YES;
    completion(self.stubbedUser, self.stubbedError);
}

- (void)updateUser:(User *)user completion:(void(^)(BOOL success, NSError *error))completion {
    completion(YES, nil);
}

@end
```

---

## แบบฝึกหัด

### แบบฝึกหัดที่ 1: Refactor MVC เป็น MVVM

ให้ refactor `PhotoGalleryViewController` (MVC) เป็น MVVM:
- สร้าง `PhotoViewModel` สำหรับแต่ละรูป
- สร้าง `PhotoGalleryViewModel` จัดการ list
- แยก network layer ออก
- เขียน unit tests

### แบบฝึกหัดที่ 2: Shopping Cart MVP

สร้าง Shopping Cart ด้วย MVP:
- `CartViewProtocol` กำหนด interface ของ View
- `CartPresenter` จัดการ logic
- `CartViewController` implement protocol
- `ProductRepository` จัดการข้อมูล

### แบบฝึกหัดที่ 3: Todo App ด้วย Clean Architecture

สร้าง Todo App ด้วย Clean Architecture:
- Entities: `TodoItem`
- Use Cases: `GetTodos`, `AddTodo`, `CompleteTodo`, `DeleteTodo`
- Repositories: `TodoRepository` protocol + implementation
- ViewModels: `TodoListViewModel`, `TodoItemViewModel`
- Unit tests สำหรับ Use Cases

### แบบฝึกหัดที่ 4: Data Binding Library

สร้าง simple reactive binding ด้วย blocks:
```objc
@interface Observable<T> : NSObject
@property (nonatomic, strong) T value;
- (void)bind:(void(^)(T value))observer;
@end
```

---

## สรุป

ในบทนี้เราได้เรียนรู้:

1. **MVC** - Architecture ดั้งเดิมของ iOS ง่ายแต่อาจกลายเป็น Massive VC
2. **Massive VC Problem** - ปัญหาเมื่อ VC รับหน้าที่มากเกินไป
3. **MVVM** - แยก presentation logic เป็น ViewModel ทดสอบง่ายกว่า
4. **Data Binding** - ด้วย KVO หรือ Blocks สำหรับ reactive UI
5. **MVP** - Presenter รู้จัก View ผ่าน Protocol แยกได้ชัดเจนกว่า
6. **VIPER** - Architecture ที่ละเอียดสำหรับแอปขนาดใหญ่
7. **Clean Architecture** - แยก business logic ออกจาก framework อย่างสมบูรณ์
8. **Refactoring** - วิธีการ refactor จาก MVC เป็น MVVM
9. **Testing** - Unit testing ใน MVVM

**หลักการสำคัญ**:
- ไม่มี architecture ที่ดีที่สุดสำหรับทุกสถานการณ์
- เลือก architecture ให้เหมาะกับขนาดแอปและทีม
- ที่สำคัญที่สุดคือ ความสม่ำเสมอ (Consistency) ในทั้งโปรเจกต์
- เริ่มง่ายๆ แล้วค่อย refactor เมื่อความต้องการเพิ่มขึ้น
