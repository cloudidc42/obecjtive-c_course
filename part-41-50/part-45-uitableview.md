# Part 45: UITableView

## บทนำ

UITableView เป็น UI Component ที่ใช้บ่อยที่สุดใน iOS development สำหรับแสดงรายการข้อมูลแบบเลื่อนได้ในแนวตั้ง เป็นพื้นฐานสำคัญที่ developer ทุกคนต้องเข้าใจ ตั้งแต่แอป Contacts, Settings, Mail ล้วนใช้ UITableView ทั้งสิ้น

ในบทนี้เราจะเรียนรู้:
- การตั้งค่า UITableView
- Delegate และ DataSource patterns
- การกำหนดจำนวน sections และ rows
- การสร้าง cells และ reuse
- Custom cells
- Section headers/footers
- Editing mode
- Search, Pull-to-refresh
- และอื่นๆ อีกมากมาย

---

## 1. UITableView Setup

### 1.1 การสร้าง UITableView แบบ Programmatic

```objc
// ViewController.h
#import <UIKit/UIKit.h>

@interface ViewController : UIViewController <UITableViewDelegate, UITableViewDataSource>

@property (nonatomic, strong) UITableView *tableView;
@property (nonatomic, strong) NSArray<NSString *> *data;

@end
```

```objc
// ViewController.m
#import "ViewController.h"

@implementation ViewController

- (void)viewDidLoad {
    [super viewDidLoad];
    
    // สร้างข้อมูลตัวอย่าง
    self.data = @[@"Apple", @"Banana", @"Cherry", @"Date", @"Elderberry",
                  @"Fig", @"Grape", @"Honeydew", @"Kiwi", @"Lemon"];
    
    // สร้าง UITableView
    self.tableView = [[UITableView alloc] initWithFrame:self.view.bounds 
                                                  style:UITableViewStylePlain];
    
    // กำหนด delegate และ dataSource
    self.tableView.delegate = self;
    self.tableView.dataSource = self;
    
    // Auto-resize เมื่อหมุนจอ
    self.tableView.autoresizingMask = UIViewAutoresizingFlexibleWidth | 
                                      UIViewAutoresizingFlexibleHeight;
    
    [self.view addSubview:self.tableView];
}

@end
```

### 1.2 การใช้ UITableViewController

UITableViewController เป็น subclass ของ UIViewController ที่มี UITableView อยู่แล้ว สะดวกกว่าการสร้างเอง

```objc
// FruitTableViewController.h
#import <UIKit/UIKit.h>

@interface FruitTableViewController : UITableViewController

@property (nonatomic, strong) NSMutableArray<NSString *> *fruits;

@end
```

```objc
// FruitTableViewController.m
#import "FruitTableViewController.h"

@implementation FruitTableViewController

- (void)viewDidLoad {
    [super viewDidLoad];
    
    self.title = @"Fruits";
    
    self.fruits = [NSMutableArray arrayWithArray:@[
        @"Apple", @"Banana", @"Cherry", @"Date", @"Elderberry",
        @"Fig", @"Grape", @"Honeydew", @"Kiwi", @"Lemon"
    ]];
    
    // Register cell class
    [self.tableView registerClass:[UITableViewCell class] 
           forCellReuseIdentifier:@"FruitCell"];
}

@end
```

### 1.3 UITableView Styles

UITableView มี 2 styles หลัก:

```objc
// Plain style - rows ต่อเนื่องกัน
UITableView *plainTable = [[UITableView alloc] initWithFrame:CGRectZero 
                                                       style:UITableViewStylePlain];

// Grouped style - rows แบ่งเป็น groups
UITableView *groupedTable = [[UITableView alloc] initWithFrame:CGRectZero 
                                                         style:UITableViewStyleGrouped];

// Inset Grouped style (iOS 13+) - groups มีขอบโค้ง
UITableView *insetGroupedTable = [[UITableView alloc] initWithFrame:CGRectZero 
                                                              style:UITableViewStyleInsetGrouped];
```

---

## 2. UITableViewDataSource Protocol

DataSource คือ protocol ที่บอก UITableView ว่ามีข้อมูลอะไรบ้าง มี 2 methods ที่บังคับต้องทำ:

### 2.1 จำนวน Rows

```objc
// จำนวน rows ใน section นั้นๆ (บังคับ)
- (NSInteger)tableView:(UITableView *)tableView numberOfRowsInSection:(NSInteger)section {
    return self.fruits.count;
}
```

### 2.2 สร้าง Cell

```objc
// สร้าง cell สำหรับแต่ละ row (บังคับ)
- (UITableViewCell *)tableView:(UITableView *)tableView 
         cellForRowAtIndexPath:(NSIndexPath *)indexPath {
    
    UITableViewCell *cell = [tableView dequeueReusableCellWithIdentifier:@"FruitCell" 
                                                            forIndexPath:indexPath];
    
    cell.textLabel.text = self.fruits[indexPath.row];
    
    return cell;
}
```

### 2.3 จำนวน Sections

```objc
// จำนวน sections (optional, default = 1)
- (NSInteger)numberOfSectionsInTableView:(UITableView *)tableView {
    return 3; // มี 3 sections
}
```

---

## 3. UITableViewDelegate Protocol

Delegate จัดการ interactions และ appearance ของ table view

### 3.1 การเลือก Row

```objc
// เมื่อผู้ใช้แตะ row
- (void)tableView:(UITableView *)tableView didSelectRowAtIndexPath:(NSIndexPath *)indexPath {
    // ยกเลิกการเลือก (เพื่อ animation)
    [tableView deselectRowAtIndexPath:indexPath animated:YES];
    
    NSString *fruit = self.fruits[indexPath.row];
    NSLog(@"เลือก: %@", fruit);
    
    // แสดง alert
    UIAlertController *alert = [UIAlertController alertControllerWithTitle:@"เลือกแล้ว"
                                                                   message:fruit
                                                            preferredStyle:UIAlertControllerStyleAlert];
    [alert addAction:[UIAlertAction actionWithTitle:@"OK" 
                                             style:UIAlertActionStyleDefault 
                                           handler:nil]];
    [self presentViewController:alert animated:YES completion:nil];
}

// เมื่อ deselect row
- (void)tableView:(UITableView *)tableView didDeselectRowAtIndexPath:(NSIndexPath *)indexPath {
    NSLog(@"Deselected row: %ld", (long)indexPath.row);
}
```

### 3.2 ความสูง Row

```objc
// กำหนดความสูงของแต่ละ row
- (CGFloat)tableView:(UITableView *)tableView heightForRowAtIndexPath:(NSIndexPath *)indexPath {
    return 60.0;
}

// ความสูง estimated (สำหรับ performance)
- (CGFloat)tableView:(UITableView *)tableView estimatedHeightForRowAtIndexPath:(NSIndexPath *)indexPath {
    return 44.0; // ค่า default
}
```

---

## 4. Cell Reuse (dequeueReusableCell)

Cell reuse เป็น pattern สำคัญที่ช่วยให้ UITableView ทำงานได้อย่างมีประสิทธิภาพ แทนที่จะสร้าง cell ใหม่ทุกครั้ง จะ reuse cells ที่ scroll ออกไปจากหน้าจอแล้ว

### 4.1 Register Cell

```objc
// Register โดยใช้ Class
[self.tableView registerClass:[UITableViewCell class] 
       forCellReuseIdentifier:@"MyCell"];

// Register โดยใช้ NIB
UINib *cellNib = [UINib nibWithNibName:@"CustomCell" bundle:nil];
[self.tableView registerNib:cellNib 
     forCellReuseIdentifier:@"CustomCell"];
```

### 4.2 Dequeue Cell

```objc
- (UITableViewCell *)tableView:(UITableView *)tableView 
         cellForRowAtIndexPath:(NSIndexPath *)indexPath {
    
    // วิธีที่ 1: dequeueReusableCellWithIdentifier:forIndexPath: (แนะนำ)
    // ต้องลง register ก่อน - จะไม่ return nil
    UITableViewCell *cell = [tableView dequeueReusableCellWithIdentifier:@"MyCell" 
                                                            forIndexPath:indexPath];
    
    // วิธีที่ 2: dequeueReusableCellWithIdentifier: (แบบเก่า)
    // อาจ return nil ถ้าไม่มี cell ใน reuse pool
    UITableViewCell *cell2 = [tableView dequeueReusableCellWithIdentifier:@"MyCell"];
    if (cell2 == nil) {
        cell2 = [[UITableViewCell alloc] initWithStyle:UITableViewCellStyleDefault 
                                       reuseIdentifier:@"MyCell"];
    }
    
    // กำหนด content ให้ cell
    cell.textLabel.text = self.data[indexPath.row];
    
    return cell;
}
```

---

## 5. Cell Styles

UITableViewCell มี 4 styles พื้นฐาน:

```objc
// Style 1: Default/Basic - มีแค่ textLabel
UITableViewCell *basicCell = [[UITableViewCell alloc] 
    initWithStyle:UITableViewCellStyleDefault 
    reuseIdentifier:@"BasicCell"];
basicCell.textLabel.text = @"ข้อความหลัก";

// Style 2: Subtitle - มี textLabel และ detailTextLabel อยู่ด้านล่าง
UITableViewCell *subtitleCell = [[UITableViewCell alloc] 
    initWithStyle:UITableViewCellStyleSubtitle 
    reuseIdentifier:@"SubtitleCell"];
subtitleCell.textLabel.text = @"ข้อความหลัก";
subtitleCell.detailTextLabel.text = @"ข้อความรอง";

// Style 3: Value1/Right Detail - textLabel ซ้าย, detailTextLabel ขวา (สีน้ำเงิน)
UITableViewCell *value1Cell = [[UITableViewCell alloc] 
    initWithStyle:UITableViewCellStyleValue1 
    reuseIdentifier:@"Value1Cell"];
value1Cell.textLabel.text = @"ชื่อ";
value1Cell.detailTextLabel.text = @"ค่า";

// Style 4: Value2/Left Detail - detailTextLabel ซ้าย (เล็กกว่า), textLabel ขวา
UITableViewCell *value2Cell = [[UITableViewCell alloc] 
    initWithStyle:UITableViewCellStyleValue2 
    reuseIdentifier:@"Value2Cell"];
value2Cell.textLabel.text = @"ข้อมูล";
value2Cell.detailTextLabel.text = @"หัวข้อ";
```

### 5.1 Accessory Types

```objc
// Checkmark ด้านขวา
cell.accessoryType = UITableViewCellAccessoryCheckmark;

// ลูกศรชี้ขวา (ไปหน้าถัดไป)
cell.accessoryType = UITableViewCellAccessoryDisclosureIndicator;

// ปุ่ม detail (i) พร้อมลูกศร
cell.accessoryType = UITableViewCellAccessoryDetailDisclosureButton;

// ปุ่ม detail (i) อย่างเดียว
cell.accessoryType = UITableViewCellAccessoryDetailButton;

// ไม่มี accessory
cell.accessoryType = UITableViewCellAccessoryNone;
```

---

## 6. Custom UITableViewCell

### 6.1 สร้าง Custom Cell Class

```objc
// ContactCell.h
#import <UIKit/UIKit.h>

@interface ContactCell : UITableViewCell

@property (nonatomic, strong) UIImageView *avatarImageView;
@property (nonatomic, strong) UILabel *nameLabel;
@property (nonatomic, strong) UILabel *phoneLabel;
@property (nonatomic, strong) UILabel *emailLabel;

@end
```

```objc
// ContactCell.m
#import "ContactCell.h"

@implementation ContactCell

- (instancetype)initWithStyle:(UITableViewCellStyle)style 
              reuseIdentifier:(NSString *)reuseIdentifier {
    
    self = [super initWithStyle:style reuseIdentifier:reuseIdentifier];
    if (self) {
        [self setupUI];
    }
    return self;
}

- (void)setupUI {
    // Avatar image
    self.avatarImageView = [[UIImageView alloc] init];
    self.avatarImageView.translatesAutoresizingMaskIntoConstraints = NO;
    self.avatarImageView.layer.cornerRadius = 25;
    self.avatarImageView.clipsToBounds = YES;
    self.avatarImageView.backgroundColor = [UIColor systemGray4Color];
    [self.contentView addSubview:self.avatarImageView];
    
    // Name label
    self.nameLabel = [[UILabel alloc] init];
    self.nameLabel.translatesAutoresizingMaskIntoConstraints = NO;
    self.nameLabel.font = [UIFont boldSystemFontOfSize:16];
    [self.contentView addSubview:self.nameLabel];
    
    // Phone label
    self.phoneLabel = [[UILabel alloc] init];
    self.phoneLabel.translatesAutoresizingMaskIntoConstraints = NO;
    self.phoneLabel.font = [UIFont systemFontOfSize:14];
    self.phoneLabel.textColor = [UIColor systemGray2Color];
    [self.contentView addSubview:self.phoneLabel];
    
    // Email label
    self.emailLabel = [[UILabel alloc] init];
    self.emailLabel.translatesAutoresizingMaskIntoConstraints = NO;
    self.emailLabel.font = [UIFont systemFontOfSize:13];
    self.emailLabel.textColor = [UIColor systemBlueColor];
    [self.contentView addSubview:self.emailLabel];
    
    // Auto Layout constraints
    [NSLayoutConstraint activateConstraints:@[
        // Avatar
        [self.avatarImageView.leadingAnchor constraintEqualToAnchor:self.contentView.leadingAnchor constant:12],
        [self.avatarImageView.centerYAnchor constraintEqualToAnchor:self.contentView.centerYAnchor],
        [self.avatarImageView.widthAnchor constraintEqualToConstant:50],
        [self.avatarImageView.heightAnchor constraintEqualToConstant:50],
        
        // Name
        [self.nameLabel.leadingAnchor constraintEqualToAnchor:self.avatarImageView.trailingAnchor constant:12],
        [self.nameLabel.topAnchor constraintEqualToAnchor:self.contentView.topAnchor constant:10],
        [self.nameLabel.trailingAnchor constraintEqualToAnchor:self.contentView.trailingAnchor constant:-12],
        
        // Phone
        [self.phoneLabel.leadingAnchor constraintEqualToAnchor:self.nameLabel.leadingAnchor],
        [self.phoneLabel.topAnchor constraintEqualToAnchor:self.nameLabel.bottomAnchor constant:4],
        [self.phoneLabel.trailingAnchor constraintEqualToAnchor:self.nameLabel.trailingAnchor],
        
        // Email
        [self.emailLabel.leadingAnchor constraintEqualToAnchor:self.nameLabel.leadingAnchor],
        [self.emailLabel.topAnchor constraintEqualToAnchor:self.phoneLabel.bottomAnchor constant:4],
        [self.emailLabel.trailingAnchor constraintEqualToAnchor:self.nameLabel.trailingAnchor],
        [self.emailLabel.bottomAnchor constraintEqualToAnchor:self.contentView.bottomAnchor constant:-10],
    ]];
}

// เรียก method นี้เพื่อกำหนดข้อมูลให้ cell
- (void)configureWithName:(NSString *)name 
                    phone:(NSString *)phone 
                    email:(NSString *)email {
    self.nameLabel.text = name;
    self.phoneLabel.text = phone;
    self.emailLabel.text = email;
    
    // ตั้งค่า avatar (initials หรือรูปจริง)
    NSString *initial = name.length > 0 ? [name substringToIndex:1].uppercaseString : @"?";
    // TODO: Set avatar image or show initial letter
}

// ต้องทำ prepareForReuse เพื่อ reset ค่าเมื่อ reuse
- (void)prepareForReuse {
    [super prepareForReuse];
    self.nameLabel.text = nil;
    self.phoneLabel.text = nil;
    self.emailLabel.text = nil;
    self.avatarImageView.image = nil;
}

@end
```

---

## 7. Sections และ Section Headers/Footers

### 7.1 ข้อมูลแบบ Sections

```objc
// Contact.h
@interface Contact : NSObject
@property (nonatomic, copy) NSString *name;
@property (nonatomic, copy) NSString *phone;
@property (nonatomic, copy) NSString *email;
- (instancetype)initWithName:(NSString *)name phone:(NSString *)phone email:(NSString *)email;
@end

// ContactsViewController.m
@interface ContactsViewController ()
@property (nonatomic, strong) NSDictionary<NSString *, NSArray *> *contactsBySection;
@property (nonatomic, strong) NSArray<NSString *> *sectionTitles;
@end

@implementation ContactsViewController

- (void)viewDidLoad {
    [super viewDidLoad];
    [self setupData];
}

- (void)setupData {
    // จัดกลุ่มตามตัวอักษรแรก
    NSMutableDictionary *grouped = [NSMutableDictionary dictionary];
    NSArray *contacts = [self loadContacts];
    
    for (Contact *contact in contacts) {
        NSString *key = [[contact.name substringToIndex:1] uppercaseString];
        NSMutableArray *group = grouped[key];
        if (!group) {
            group = [NSMutableArray array];
            grouped[key] = group;
        }
        [group addObject:contact];
    }
    
    self.contactsBySection = [grouped copy];
    self.sectionTitles = [[grouped.allKeys sortedArrayUsingSelector:@selector(localizedCompare:)] copy];
}

- (NSInteger)numberOfSectionsInTableView:(UITableView *)tableView {
    return self.sectionTitles.count;
}

- (NSInteger)tableView:(UITableView *)tableView numberOfRowsInSection:(NSInteger)section {
    NSString *key = self.sectionTitles[section];
    return [self.contactsBySection[key] count];
}
```

### 7.2 Section Headers และ Footers

```objc
// Title-based headers
- (NSString *)tableView:(UITableView *)tableView titleForHeaderInSection:(NSInteger)section {
    return self.sectionTitles[section];
}

- (NSString *)tableView:(UITableView *)tableView titleForFooterInSection:(NSInteger)section {
    NSString *key = self.sectionTitles[section];
    NSInteger count = [self.contactsBySection[key] count];
    return [NSString stringWithFormat:@"%ld รายการ", (long)count];
}

// Custom view headers
- (UIView *)tableView:(UITableView *)tableView viewForHeaderInSection:(NSInteger)section {
    UIView *headerView = [[UIView alloc] init];
    headerView.backgroundColor = [UIColor systemGray6Color];
    
    UILabel *label = [[UILabel alloc] initWithFrame:CGRectMake(16, 0, tableView.frame.size.width - 32, 30)];
    label.text = self.sectionTitles[section];
    label.font = [UIFont boldSystemFontOfSize:14];
    label.textColor = [UIColor systemGray2Color];
    [headerView addSubview:label];
    
    return headerView;
}

- (CGFloat)tableView:(UITableView *)tableView heightForHeaderInSection:(NSInteger)section {
    return 30.0;
}

// Section index titles (สารบัญตัวอักษรด้านขวา)
- (NSArray<NSString *> *)sectionIndexTitlesForTableView:(UITableView *)tableView {
    return self.sectionTitles;
}
```

---

## 8. Row Heights

### 8.1 Static Height

```objc
- (CGFloat)tableView:(UITableView *)tableView heightForRowAtIndexPath:(NSIndexPath *)indexPath {
    return 70.0; // ความสูงคงที่
}

// หรือกำหนดผ่าน property
self.tableView.rowHeight = 70.0;
```

### 8.2 Dynamic Height (Self-Sizing)

```objc
// กำหนดให้ cell คำนวณความสูงเอง
self.tableView.rowHeight = UITableViewAutomaticDimension;
self.tableView.estimatedRowHeight = 80.0; // ค่า estimated สำหรับ performance

// ใน cell ต้องกำหนด constraints ให้ถูกต้อง
// - มี constraints จาก top ถึง bottom ของ contentView
// - ไม่มี height constraint แบบ fixed
```

### 8.3 Estimated Heights สำหรับ Performance

```objc
- (CGFloat)tableView:(UITableView *)tableView 
    estimatedHeightForRowAtIndexPath:(NSIndexPath *)indexPath {
    // ให้ค่า estimate ที่ใกล้เคียงที่สุด
    return 80.0;
}

- (CGFloat)tableView:(UITableView *)tableView 
    estimatedHeightForHeaderInSection:(NSInteger)section {
    return 30.0;
}

- (CGFloat)tableView:(UITableView *)tableView 
    estimatedHeightForFooterInSection:(NSInteger)section {
    return 30.0;
}
```

---

## 9. UITableView Editing Mode

### 9.1 เปิด/ปิด Editing Mode

```objc
// เปิด editing mode
[self.tableView setEditing:YES animated:YES];

// ปิด editing mode
[self.tableView setEditing:NO animated:YES];

// Toggle
self.navigationItem.rightBarButtonItem = self.editButtonItem; // built-in button

// Override setEditing ใน UITableViewController
- (void)setEditing:(BOOL)editing animated:(BOOL)animated {
    [super setEditing:editing animated:animated];
    // ทำอะไรบางอย่างเมื่อเปลี่ยน editing state
}
```

### 9.2 Delete Rows

```objc
// กำหนดว่า row ไหน edit ได้
- (UITableViewCellEditingStyle)tableView:(UITableView *)tableView 
           editingStyleForRowAtIndexPath:(NSIndexPath *)indexPath {
    return UITableViewCellEditingStyleDelete; // ปุ่มลบ
}

// จัดการเมื่อ commit การ edit
- (void)tableView:(UITableView *)tableView 
commitEditingStyle:(UITableViewCellEditingStyle)editingStyle 
forRowAtIndexPath:(NSIndexPath *)indexPath {
    
    if (editingStyle == UITableViewCellEditingStyleDelete) {
        // ลบจาก data source
        [self.fruits removeObjectAtIndex:indexPath.row];
        
        // ลบ row จาก table view
        [tableView deleteRowsAtIndexPaths:@[indexPath] 
                         withRowAnimation:UITableViewRowAnimationFade];
    }
}
```

### 9.3 Insert Rows

```objc
- (UITableViewCellEditingStyle)tableView:(UITableView *)tableView 
           editingStyleForRowAtIndexPath:(NSIndexPath *)indexPath {
    // row สุดท้ายเป็นปุ่ม insert
    if (indexPath.row == self.fruits.count) {
        return UITableViewCellEditingStyleInsert;
    }
    return UITableViewCellEditingStyleDelete;
}

- (void)tableView:(UITableView *)tableView 
commitEditingStyle:(UITableViewCellEditingStyle)editingStyle 
forRowAtIndexPath:(NSIndexPath *)indexPath {
    
    if (editingStyle == UITableViewCellEditingStyleInsert) {
        // เพิ่มข้อมูลใหม่
        [self.fruits addObject:@"New Fruit"];
        [tableView insertRowsAtIndexPaths:@[indexPath] 
                         withRowAnimation:UITableViewRowAnimationAutomatic];
    }
}
```

### 9.4 Reorder Rows

```objc
// อนุญาตให้เรียงลำดับใหม่
- (BOOL)tableView:(UITableView *)tableView 
canMoveRowAtIndexPath:(NSIndexPath *)indexPath {
    return YES;
}

// จัดการเมื่อย้าย row
- (void)tableView:(UITableView *)tableView 
moveRowAtIndexPath:(NSIndexPath *)sourceIndexPath 
      toIndexPath:(NSIndexPath *)destinationIndexPath {
    
    NSString *item = self.fruits[sourceIndexPath.row];
    [self.fruits removeObjectAtIndex:sourceIndexPath.row];
    [self.fruits insertObject:item atIndex:destinationIndexPath.row];
}

// จำกัดการย้าย (ไม่ให้ข้าม section)
- (NSIndexPath *)tableView:(UITableView *)tableView 
targetIndexPathForMoveFromRowAtIndexPath:(NSIndexPath *)sourceIndexPath 
       toProposedIndexPath:(NSIndexPath *)proposedDestinationIndexPath {
    
    if (sourceIndexPath.section != proposedDestinationIndexPath.section) {
        return sourceIndexPath; // ไม่อนุญาต
    }
    return proposedDestinationIndexPath;
}
```

---

## 10. Swipe Actions

### 10.1 Trailing Swipe Actions (iOS 11+)

```objc
// Swipe จากขวามาซ้าย
- (UISwipeActionsConfiguration *)tableView:(UITableView *)tableView 
trailingSwipeActionsConfigurationForRowAtIndexPath:(NSIndexPath *)indexPath {
    
    // Delete action
    UIContextualAction *deleteAction = [UIContextualAction 
        contextualActionWithStyle:UIContextualActionStyleDestructive
                            title:@"ลบ"
                          handler:^(UIContextualAction *action, UIView *sourceView, void (^completionHandler)(BOOL)) {
        
        [self.fruits removeObjectAtIndex:indexPath.row];
        [tableView deleteRowsAtIndexPaths:@[indexPath] 
                         withRowAnimation:UITableViewRowAnimationAutomatic];
        completionHandler(YES);
    }];
    deleteAction.image = [UIImage systemImageNamed:@"trash"];
    
    // More action
    UIContextualAction *moreAction = [UIContextualAction 
        contextualActionWithStyle:UIContextualActionStyleNormal
                            title:@"เพิ่มเติม"
                          handler:^(UIContextualAction *action, UIView *sourceView, void (^completionHandler)(BOOL)) {
        // แสดง action sheet
        completionHandler(YES);
    }];
    moreAction.backgroundColor = [UIColor systemGray2Color];
    moreAction.image = [UIImage systemImageNamed:@"ellipsis"];
    
    UISwipeActionsConfiguration *config = [UISwipeActionsConfiguration 
                                          configurationWithActions:@[deleteAction, moreAction]];
    config.performsFirstActionWithFullSwipe = YES; // swipe เต็มๆ = delete ทันที
    
    return config;
}
```

### 10.2 Leading Swipe Actions (iOS 11+)

```objc
// Swipe จากซ้ายมาขวา
- (UISwipeActionsConfiguration *)tableView:(UITableView *)tableView 
leadingSwipeActionsConfigurationForRowAtIndexPath:(NSIndexPath *)indexPath {
    
    UIContextualAction *favoriteAction = [UIContextualAction 
        contextualActionWithStyle:UIContextualActionStyleNormal
                            title:@"ชื่นชอบ"
                          handler:^(UIContextualAction *action, UIView *sourceView, void (^completionHandler)(BOOL)) {
        // Toggle favorite
        completionHandler(YES);
    }];
    favoriteAction.backgroundColor = [UIColor systemYellowColor];
    favoriteAction.image = [UIImage systemImageNamed:@"star.fill"];
    
    return [UISwipeActionsConfiguration configurationWithActions:@[favoriteAction]];
}
```

---

## 11. UISearchBar กับ UITableView

### 11.1 เพิ่ม UISearchBar

```objc
@interface SearchViewController () <UISearchBarDelegate, UISearchResultsUpdating>
@property (nonatomic, strong) UISearchController *searchController;
@property (nonatomic, strong) NSArray<NSString *> *allItems;
@property (nonatomic, strong) NSArray<NSString *> *filteredItems;
@end

@implementation SearchViewController

- (void)viewDidLoad {
    [super viewDidLoad];
    
    self.allItems = @[@"Apple", @"Apricot", @"Banana", @"Blueberry", @"Cherry",
                      @"Date", @"Fig", @"Grape", @"Kiwi", @"Lemon", @"Mango"];
    self.filteredItems = self.allItems;
    
    // สร้าง Search Controller
    self.searchController = [[UISearchController alloc] initWithSearchResultsController:nil];
    self.searchController.searchResultsUpdater = self;
    self.searchController.obscuresBackgroundDuringPresentation = NO;
    self.searchController.searchBar.placeholder = @"ค้นหาผลไม้...";
    
    // แสดงใน navigation bar
    self.navigationItem.searchController = self.searchController;
    self.navigationItem.hidesSearchBarWhenScrolling = NO;
    self.definesPresentationContext = YES;
}

// UISearchResultsUpdating
- (void)updateSearchResultsForSearchController:(UISearchController *)searchController {
    NSString *searchText = searchController.searchBar.text;
    
    if (searchText.length == 0) {
        self.filteredItems = self.allItems;
    } else {
        NSPredicate *predicate = [NSPredicate predicateWithFormat:@"SELF CONTAINS[cd] %@", searchText];
        self.filteredItems = [self.allItems filteredArrayUsingPredicate:predicate];
    }
    
    [self.tableView reloadData];
}

- (NSInteger)tableView:(UITableView *)tableView numberOfRowsInSection:(NSInteger)section {
    return self.filteredItems.count;
}

- (UITableViewCell *)tableView:(UITableView *)tableView 
         cellForRowAtIndexPath:(NSIndexPath *)indexPath {
    UITableViewCell *cell = [tableView dequeueReusableCellWithIdentifier:@"Cell" 
                                                            forIndexPath:indexPath];
    cell.textLabel.text = self.filteredItems[indexPath.row];
    return cell;
}

@end
```

---

## 12. Pull-to-Refresh

### 12.1 UIRefreshControl

```objc
@interface RefreshViewController ()
@property (nonatomic, strong) UIRefreshControl *refreshControl;
@end

@implementation RefreshViewController

- (void)viewDidLoad {
    [super viewDidLoad];
    
    // สร้าง refresh control
    self.refreshControl = [[UIRefreshControl alloc] init];
    self.refreshControl.tintColor = [UIColor systemBlueColor];
    self.refreshControl.attributedTitle = [[NSAttributedString alloc] 
                                          initWithString:@"กำลังโหลด..."];
    [self.refreshControl addTarget:self 
                            action:@selector(handleRefresh:) 
                  forControlEvents:UIControlEventValueChanged];
    
    // เพิ่มใน table view
    self.tableView.refreshControl = self.refreshControl; // iOS 10+
    // หรือ: [self.tableView addSubview:self.refreshControl];
}

- (void)handleRefresh:(UIRefreshControl *)sender {
    // โหลดข้อมูลใหม่
    [self loadDataWithCompletion:^{
        dispatch_async(dispatch_get_main_queue(), ^{
            [sender endRefreshing];
            [self.tableView reloadData];
        });
    }];
}

- (void)loadDataWithCompletion:(void(^)(void))completion {
    dispatch_after(dispatch_time(DISPATCH_TIME_NOW, 2 * NSEC_PER_SEC), 
                   dispatch_get_global_queue(QOS_CLASS_UTILITY, 0), ^{
        // Simulate network request
        if (completion) completion();
    });
}

@end
```

---

## 13. Infinite Scroll Pattern

```objc
@interface InfiniteScrollViewController ()
@property (nonatomic, strong) NSMutableArray *items;
@property (nonatomic, assign) NSInteger currentPage;
@property (nonatomic, assign) BOOL isLoading;
@property (nonatomic, assign) BOOL hasMoreData;
@end

@implementation InfiniteScrollViewController

- (void)viewDidLoad {
    [super viewDidLoad];
    self.items = [NSMutableArray array];
    self.currentPage = 1;
    self.hasMoreData = YES;
    self.isLoading = NO;
    
    [self loadNextPage];
}

// ตรวจสอบว่าผู้ใช้ scroll ถึง bottom แล้ว
- (void)scrollViewDidScroll:(UIScrollView *)scrollView {
    CGFloat contentHeight = scrollView.contentSize.height;
    CGFloat scrollHeight = scrollView.frame.size.height;
    CGFloat scrollOffset = scrollView.contentOffset.y;
    
    // ถ้า scroll ถึง 80% ของ content
    if (scrollOffset > contentHeight - scrollHeight * 1.2 && 
        !self.isLoading && 
        self.hasMoreData) {
        [self loadNextPage];
    }
}

- (void)loadNextPage {
    self.isLoading = YES;
    [self showLoadingFooter];
    
    // Simulate API call
    dispatch_after(dispatch_time(DISPATCH_TIME_NOW, 1 * NSEC_PER_SEC), 
                   dispatch_get_global_queue(QOS_CLASS_UTILITY, 0), ^{
        
        // สร้างข้อมูลใหม่ (ปกติมาจาก API)
        NSMutableArray *newItems = [NSMutableArray array];
        for (NSInteger i = 0; i < 20; i++) {
            NSInteger itemNumber = (self.currentPage - 1) * 20 + i + 1;
            [newItems addObject:[NSString stringWithFormat:@"Item %ld", (long)itemNumber]];
        }
        
        dispatch_async(dispatch_get_main_queue(), ^{
            [self.items addObjectsFromArray:newItems];
            self.currentPage++;
            self.isLoading = NO;
            
            // ตรวจสอบว่ามีข้อมูลอีกไหม
            self.hasMoreData = self.currentPage <= 5; // มีแค่ 5 หน้า
            
            [self.tableView reloadData];
            [self hideLoadingFooter];
        });
    });
}

- (void)showLoadingFooter {
    UIActivityIndicatorView *indicator = [[UIActivityIndicatorView alloc] 
                                         initWithActivityIndicatorStyle:UIActivityIndicatorViewStyleMedium];
    indicator.frame = CGRectMake(0, 0, self.tableView.bounds.size.width, 50);
    [indicator startAnimating];
    self.tableView.tableFooterView = indicator;
}

- (void)hideLoadingFooter {
    self.tableView.tableFooterView = nil;
}

@end
```

---

## 14. NSFetchedResultsController

NSFetchedResultsController ใช้ร่วมกับ Core Data เพื่อจัดการข้อมูลและอัปเดต UITableView โดยอัตโนมัติ

### 14.1 การตั้งค่า

```objc
// ContactsWithCoreDataVC.m
#import "ContactsWithCoreDataVC.h"
#import <CoreData/CoreData.h>

@interface ContactsWithCoreDataVC () <NSFetchedResultsControllerDelegate>
@property (nonatomic, strong) NSFetchedResultsController *fetchedResultsController;
@end

@implementation ContactsWithCoreDataVC

- (void)viewDidLoad {
    [super viewDidLoad];
    [self setupFetchedResultsController];
}

- (void)setupFetchedResultsController {
    NSManagedObjectContext *context = [self managedObjectContext];
    
    // สร้าง fetch request
    NSFetchRequest *fetchRequest = [NSFetchRequest fetchRequestWithEntityName:@"Contact"];
    
    // Sort descriptors
    NSSortDescriptor *sortByName = [[NSSortDescriptor alloc] initWithKey:@"name" 
                                                               ascending:YES 
                                                                selector:@selector(localizedCaseInsensitiveCompare:)];
    fetchRequest.sortDescriptors = @[sortByName];
    
    // Predicate (filter)
    // fetchRequest.predicate = [NSPredicate predicateWithFormat:@"isFavorite == YES"];
    
    // Batch size สำหรับ performance
    fetchRequest.fetchBatchSize = 20;
    
    // สร้าง FRC
    self.fetchedResultsController = [[NSFetchedResultsController alloc] 
                                    initWithFetchRequest:fetchRequest
                                    managedObjectContext:context
                                      sectionNameKeyPath:@"nameInitial" // แบ่ง section ตาม initial
                                               cacheName:@"ContactsCache"];
    
    self.fetchedResultsController.delegate = self;
    
    NSError *error;
    if (![self.fetchedResultsController performFetch:&error]) {
        NSLog(@"Fetch error: %@", error);
    }
}

// DataSource methods
- (NSInteger)numberOfSectionsInTableView:(UITableView *)tableView {
    return self.fetchedResultsController.sections.count;
}

- (NSInteger)tableView:(UITableView *)tableView numberOfRowsInSection:(NSInteger)section {
    id<NSFetchedResultsSectionInfo> sectionInfo = self.fetchedResultsController.sections[section];
    return sectionInfo.numberOfObjects;
}

- (UITableViewCell *)tableView:(UITableView *)tableView 
         cellForRowAtIndexPath:(NSIndexPath *)indexPath {
    UITableViewCell *cell = [tableView dequeueReusableCellWithIdentifier:@"ContactCell" 
                                                            forIndexPath:indexPath];
    
    NSManagedObject *contact = [self.fetchedResultsController objectAtIndexPath:indexPath];
    cell.textLabel.text = [contact valueForKey:@"name"];
    cell.detailTextLabel.text = [contact valueForKey:@"phone"];
    
    return cell;
}

- (NSString *)tableView:(UITableView *)tableView titleForHeaderInSection:(NSInteger)section {
    id<NSFetchedResultsSectionInfo> sectionInfo = self.fetchedResultsController.sections[section];
    return sectionInfo.name;
}

// NSFetchedResultsControllerDelegate
- (void)controllerWillChangeContent:(NSFetchedResultsController *)controller {
    [self.tableView beginUpdates];
}

- (void)controller:(NSFetchedResultsController *)controller 
   didChangeObject:(id)anObject 
       atIndexPath:(NSIndexPath *)indexPath 
     forChangeType:(NSFetchedResultsChangeType)type 
      newIndexPath:(NSIndexPath *)newIndexPath {
    
    switch (type) {
        case NSFetchedResultsChangeInsert:
            [self.tableView insertRowsAtIndexPaths:@[newIndexPath] 
                                 withRowAnimation:UITableViewRowAnimationAutomatic];
            break;
        case NSFetchedResultsChangeDelete:
            [self.tableView deleteRowsAtIndexPaths:@[indexPath] 
                                 withRowAnimation:UITableViewRowAnimationAutomatic];
            break;
        case NSFetchedResultsChangeUpdate:
            [self.tableView reloadRowsAtIndexPaths:@[indexPath] 
                                 withRowAnimation:UITableViewRowAnimationAutomatic];
            break;
        case NSFetchedResultsChangeMove:
            [self.tableView moveRowAtIndexPath:indexPath toIndexPath:newIndexPath];
            break;
    }
}

- (void)controllerDidChangeContent:(NSFetchedResultsController *)controller {
    [self.tableView endUpdates];
}

@end
```

---

## 15. ตัวอย่างสมบูรณ์: Contacts App

### 15.1 Model

```objc
// Contact.h
#import <Foundation/Foundation.h>

@interface Contact : NSObject

@property (nonatomic, copy) NSString *name;
@property (nonatomic, copy) NSString *phone;
@property (nonatomic, copy) NSString *email;
@property (nonatomic, strong) UIImage *avatar;
@property (nonatomic, assign) BOOL isFavorite;

- (instancetype)initWithName:(NSString *)name 
                       phone:(NSString *)phone 
                       email:(NSString *)email;
- (NSString *)nameInitial;

@end
```

```objc
// Contact.m
#import "Contact.h"

@implementation Contact

- (instancetype)initWithName:(NSString *)name 
                       phone:(NSString *)phone 
                       email:(NSString *)email {
    self = [super init];
    if (self) {
        _name = [name copy];
        _phone = [phone copy];
        _email = [email copy];
        _isFavorite = NO;
    }
    return self;
}

- (NSString *)nameInitial {
    if (self.name.length > 0) {
        return [[self.name substringToIndex:1] uppercaseString];
    }
    return @"#";
}

@end
```

### 15.2 ContactsViewController (Main)

```objc
// ContactsViewController.h
#import <UIKit/UIKit.h>

@interface ContactsViewController : UITableViewController <UISearchResultsUpdating>
@end
```

```objc
// ContactsViewController.m
#import "ContactsViewController.h"
#import "Contact.h"
#import "ContactCell.h"
#import "ContactDetailViewController.h"

@interface ContactsViewController ()
@property (nonatomic, strong) NSArray<Contact *> *allContacts;
@property (nonatomic, strong) NSArray<Contact *> *filteredContacts;
@property (nonatomic, strong) NSDictionary<NSString *, NSArray<Contact *> *> *groupedContacts;
@property (nonatomic, strong) NSArray<NSString *> *sectionKeys;
@property (nonatomic, strong) UISearchController *searchController;
@property (nonatomic, assign) BOOL isSearching;
@end

@implementation ContactsViewController

- (void)viewDidLoad {
    [super viewDidLoad];
    
    [self setupNavigationBar];
    [self setupSearchController];
    [self setupTableView];
    [self loadContacts];
}

- (void)setupNavigationBar {
    self.title = @"Contacts";
    self.navigationItem.rightBarButtonItem = [[UIBarButtonItem alloc] 
                                             initWithBarButtonSystemItem:UIBarButtonSystemItemAdd 
                                                                  target:self 
                                                                  action:@selector(addContact)];
    self.navigationItem.leftBarButtonItem = self.editButtonItem;
}

- (void)setupSearchController {
    self.searchController = [[UISearchController alloc] initWithSearchResultsController:nil];
    self.searchController.searchResultsUpdater = self;
    self.searchController.obscuresBackgroundDuringPresentation = NO;
    self.searchController.searchBar.placeholder = @"ค้นหา";
    self.navigationItem.searchController = self.searchController;
    self.definesPresentationContext = YES;
}

- (void)setupTableView {
    [self.tableView registerClass:[ContactCell class] forCellReuseIdentifier:@"ContactCell"];
    self.tableView.rowHeight = UITableViewAutomaticDimension;
    self.tableView.estimatedRowHeight = 70;
    
    // Pull to refresh
    UIRefreshControl *refresh = [[UIRefreshControl alloc] init];
    [refresh addTarget:self action:@selector(refreshContacts:) forControlEvents:UIControlEventValueChanged];
    self.tableView.refreshControl = refresh;
}

- (void)loadContacts {
    // ข้อมูลตัวอย่าง
    self.allContacts = @[
        [[Contact alloc] initWithName:@"อาทิตย์ สว่างใจ" phone:@"081-111-1111" email:@"arthit@example.com"],
        [[Contact alloc] initWithName:@"บัว บุญมี" phone:@"082-222-2222" email:@"bua@example.com"],
        [[Contact alloc] initWithName:@"Charlie Brown" phone:@"083-333-3333" email:@"charlie@example.com"],
        [[Contact alloc] initWithName:@"ดาว ดีงาม" phone:@"084-444-4444" email:@"dao@example.com"],
        [[Contact alloc] initWithName:@"Edward Smith" phone:@"085-555-5555" email:@"edward@example.com"],
        [[Contact alloc] initWithName:@"ฟ้า ใสแจ๋ว" phone:@"086-666-6666" email:@"fa@example.com"],
    ];
    
    [self groupContacts];
    [self.tableView reloadData];
}

- (void)groupContacts {
    NSMutableDictionary *grouped = [NSMutableDictionary dictionary];
    NSArray *contacts = self.isSearching ? self.filteredContacts : self.allContacts;
    
    for (Contact *contact in contacts) {
        NSString *key = contact.nameInitial;
        NSMutableArray *group = grouped[key];
        if (!group) {
            group = [NSMutableArray array];
            grouped[key] = group;
        }
        [group addObject:contact];
    }
    
    self.groupedContacts = [grouped copy];
    self.sectionKeys = [[grouped.allKeys sortedArrayUsingSelector:@selector(localizedCompare:)] copy];
}

// UITableViewDataSource
- (NSInteger)numberOfSectionsInTableView:(UITableView *)tableView {
    return self.sectionKeys.count;
}

- (NSInteger)tableView:(UITableView *)tableView numberOfRowsInSection:(NSInteger)section {
    NSString *key = self.sectionKeys[section];
    return [self.groupedContacts[key] count];
}

- (UITableViewCell *)tableView:(UITableView *)tableView 
         cellForRowAtIndexPath:(NSIndexPath *)indexPath {
    ContactCell *cell = [tableView dequeueReusableCellWithIdentifier:@"ContactCell" 
                                                        forIndexPath:indexPath];
    Contact *contact = [self contactAtIndexPath:indexPath];
    [cell configureWithName:contact.name 
                      phone:contact.phone 
                      email:contact.email];
    return cell;
}

- (NSString *)tableView:(UITableView *)tableView titleForHeaderInSection:(NSInteger)section {
    return self.sectionKeys[section];
}

- (NSArray<NSString *> *)sectionIndexTitlesForTableView:(UITableView *)tableView {
    return self.isSearching ? nil : self.sectionKeys;
}

// UITableViewDelegate
- (void)tableView:(UITableView *)tableView didSelectRowAtIndexPath:(NSIndexPath *)indexPath {
    [tableView deselectRowAtIndexPath:indexPath animated:YES];
    Contact *contact = [self contactAtIndexPath:indexPath];
    
    ContactDetailViewController *detailVC = [[ContactDetailViewController alloc] initWithContact:contact];
    [self.navigationController pushViewController:detailVC animated:YES];
}

- (UISwipeActionsConfiguration *)tableView:(UITableView *)tableView 
trailingSwipeActionsConfigurationForRowAtIndexPath:(NSIndexPath *)indexPath {
    
    Contact *contact = [self contactAtIndexPath:indexPath];
    
    UIContextualAction *deleteAction = [UIContextualAction 
        contextualActionWithStyle:UIContextualActionStyleDestructive
                            title:@"ลบ"
                          handler:^(UIContextualAction *action, UIView *view, void (^completion)(BOOL)) {
        [self deleteContact:contact atIndexPath:indexPath];
        completion(YES);
    }];
    
    UIContextualAction *favoriteAction = [UIContextualAction 
        contextualActionWithStyle:UIContextualActionStyleNormal
                            title:contact.isFavorite ? @"เลิกชอบ" : @"ชอบ"
                          handler:^(UIContextualAction *action, UIView *view, void (^completion)(BOOL)) {
        contact.isFavorite = !contact.isFavorite;
        [tableView reloadRowsAtIndexPaths:@[indexPath] withRowAnimation:UITableViewRowAnimationNone];
        completion(YES);
    }];
    favoriteAction.backgroundColor = [UIColor systemYellowColor];
    
    return [UISwipeActionsConfiguration configurationWithActions:@[deleteAction, favoriteAction]];
}

// Helper
- (Contact *)contactAtIndexPath:(NSIndexPath *)indexPath {
    NSString *key = self.sectionKeys[indexPath.section];
    return self.groupedContacts[key][indexPath.row];
}

- (void)deleteContact:(Contact *)contact atIndexPath:(NSIndexPath *)indexPath {
    NSMutableArray *mutable = [self.allContacts mutableCopy];
    [mutable removeObject:contact];
    self.allContacts = [mutable copy];
    [self groupContacts];
    [self.tableView reloadData];
}

// UISearchResultsUpdating
- (void)updateSearchResultsForSearchController:(UISearchController *)searchController {
    NSString *query = searchController.searchBar.text;
    self.isSearching = query.length > 0;
    
    if (self.isSearching) {
        NSPredicate *pred = [NSPredicate predicateWithFormat:
                            @"name CONTAINS[cd] %@ OR phone CONTAINS[cd] %@ OR email CONTAINS[cd] %@", 
                            query, query, query];
        self.filteredContacts = [self.allContacts filteredArrayUsingPredicate:pred];
    }
    
    [self groupContacts];
    [self.tableView reloadData];
}

- (void)refreshContacts:(UIRefreshControl *)sender {
    dispatch_after(dispatch_time(DISPATCH_TIME_NOW, 1 * NSEC_PER_SEC), dispatch_get_main_queue(), ^{
        [sender endRefreshing];
        [self.tableView reloadData];
    });
}

- (void)addContact {
    // แสดง form เพิ่ม contact
    UIAlertController *alert = [UIAlertController alertControllerWithTitle:@"เพิ่มผู้ติดต่อ"
                                                                   message:nil
                                                            preferredStyle:UIAlertControllerStyleAlert];
    [alert addTextFieldWithConfigurationHandler:^(UITextField *textField) {
        textField.placeholder = @"ชื่อ";
    }];
    [alert addTextFieldWithConfigurationHandler:^(UITextField *textField) {
        textField.placeholder = @"เบอร์โทร";
        textField.keyboardType = UIKeyboardTypePhonePad;
    }];
    
    [alert addAction:[UIAlertAction actionWithTitle:@"เพิ่ม" 
                                             style:UIAlertActionStyleDefault 
                                           handler:^(UIAlertAction *action) {
        NSString *name = alert.textFields[0].text;
        NSString *phone = alert.textFields[1].text;
        if (name.length > 0) {
            Contact *newContact = [[Contact alloc] initWithName:name phone:phone email:@""];
            NSMutableArray *updated = [self.allContacts mutableCopy];
            [updated addObject:newContact];
            self.allContacts = [[updated sortedArrayUsingComparator:^NSComparisonResult(Contact *a, Contact *b) {
                return [a.name localizedCompare:b.name];
            }] copy];
            [self groupContacts];
            [self.tableView reloadData];
        }
    }]];
    
    [alert addAction:[UIAlertAction actionWithTitle:@"ยกเลิก" 
                                             style:UIAlertActionStyleCancel 
                                           handler:nil]];
    
    [self presentViewController:alert animated:YES completion:nil];
}

@end
```

---

## 16. Advanced Topics

### 16.1 Batch Updates (iOS 11+)

```objc
// อัปเดตหลาย rows พร้อมกันแบบ animated
[self.tableView performBatchUpdates:^{
    // Insert
    [self.tableView insertRowsAtIndexPaths:@[
        [NSIndexPath indexPathForRow:0 inSection:0]
    ] withRowAnimation:UITableViewRowAnimationAutomatic];
    
    // Delete
    [self.tableView deleteRowsAtIndexPaths:@[
        [NSIndexPath indexPathForRow:3 inSection:0]
    ] withRowAnimation:UITableViewRowAnimationFade];
    
    // Move
    [self.tableView moveRowAtIndexPath:[NSIndexPath indexPathForRow:1 inSection:0]
                           toIndexPath:[NSIndexPath indexPathForRow:4 inSection:0]];
    
} completion:^(BOOL finished) {
    NSLog(@"Batch update complete");
}];
```

### 16.2 Context Menu (iOS 13+)

```objc
- (UIContextMenuConfiguration *)tableView:(UITableView *)tableView 
contextMenuConfigurationForRowAtIndexPath:(NSIndexPath *)indexPath 
                                    point:(CGPoint)point {
    
    return [UIContextMenuConfiguration configurationWithIdentifier:nil
                                                   previewProvider:nil
                                                    actionProvider:^UIMenu *(NSArray<UIMenuElement *> *suggestedActions) {
        
        UIAction *copyAction = [UIAction actionWithTitle:@"คัดลอก" 
                                                   image:[UIImage systemImageNamed:@"doc.on.doc"]
                                              identifier:nil
                                                 handler:^(__kindof UIAction *action) {
            NSString *name = self.fruits[indexPath.row];
            [UIPasteboard generalPasteboard].string = name;
        }];
        
        UIAction *shareAction = [UIAction actionWithTitle:@"แชร์"
                                                    image:[UIImage systemImageNamed:@"square.and.arrow.up"]
                                               identifier:nil
                                                  handler:^(__kindof UIAction *action) {
            NSString *name = self.fruits[indexPath.row];
            UIActivityViewController *vc = [[UIActivityViewController alloc] 
                                           initWithActivityItems:@[name] 
                                           applicationActivities:nil];
            [self presentViewController:vc animated:YES completion:nil];
        }];
        
        UIAction *deleteAction = [UIAction actionWithTitle:@"ลบ"
                                                     image:[UIImage systemImageNamed:@"trash"]
                                                identifier:nil
                                                   handler:^(__kindof UIAction *action) {
            [self.fruits removeObjectAtIndex:indexPath.row];
            [tableView deleteRowsAtIndexPaths:@[indexPath] 
                             withRowAnimation:UITableViewRowAnimationAutomatic];
        }];
        deleteAction.attributes = UIMenuElementAttributesDestructive;
        
        return [UIMenu menuWithTitle:@"" children:@[copyAction, shareAction, deleteAction]];
    }];
}
```

### 16.3 Diffable Data Source (iOS 13+)

```objc
// Modern way to manage table view data
typedef NS_ENUM(NSInteger, Section) {
    SectionMain
};

@interface ModernTableViewController ()
@property (nonatomic, strong) UITableViewDiffableDataSource<NSNumber *, NSString *> *dataSource;
@end

@implementation ModernTableViewController

- (void)setupDataSource {
    self.dataSource = [[UITableViewDiffableDataSource alloc] 
                      initWithTableView:self.tableView
                           cellProvider:^UITableViewCell *(UITableView *tableView, NSIndexPath *indexPath, NSString *item) {
        UITableViewCell *cell = [tableView dequeueReusableCellWithIdentifier:@"Cell" forIndexPath:indexPath];
        cell.textLabel.text = item;
        return cell;
    }];
}

- (void)applySnapshot:(NSArray<NSString *> *)items animated:(BOOL)animated {
    NSDiffableDataSourceSnapshot<NSNumber *, NSString *> *snapshot = 
        [[NSDiffableDataSourceSnapshot alloc] init];
    [snapshot appendSectionsWithIdentifiers:@[@(SectionMain)]];
    [snapshot appendItemsWithIdentifiers:items];
    [self.dataSource applySnapshot:snapshot animatingDifferences:animated];
}

@end
```

---

## 17. Performance Tips

### 17.1 Cell Prefetching

```objc
@interface PerformanceTableVC () <UITableViewDataSourcePrefetching>
@end

@implementation PerformanceTableVC

- (void)viewDidLoad {
    [super viewDidLoad];
    self.tableView.prefetchDataSource = self;
}

// เตรียมข้อมูลล่วงหน้า
- (void)tableView:(UITableView *)tableView 
prefetchRowsAtIndexPaths:(NSArray<NSIndexPath *> *)indexPaths {
    for (NSIndexPath *indexPath in indexPaths) {
        // preload images หรือข้อมูลที่ใช้เวลา
        [self prefetchImageForIndexPath:indexPath];
    }
}

// ยกเลิกถ้า scroll กลับ
- (void)tableView:(UITableView *)tableView 
cancelPrefetchingForRowsAtIndexPaths:(NSArray<NSIndexPath *> *)indexPaths {
    for (NSIndexPath *indexPath in indexPaths) {
        [self cancelPrefetchForIndexPath:indexPath];
    }
}

@end
```

### 17.2 Image Loading

```objc
// โหลดรูปภาพ async ไม่ให้ block main thread
- (UITableViewCell *)tableView:(UITableView *)tableView 
         cellForRowAtIndexPath:(NSIndexPath *)indexPath {
    
    UITableViewCell *cell = [tableView dequeueReusableCellWithIdentifier:@"Cell" 
                                                            forIndexPath:indexPath];
    cell.imageView.image = [UIImage systemImageNamed:@"person.circle"]; // placeholder
    
    NSURL *imageURL = [NSURL URLWithString:self.imageURLs[indexPath.row]];
    NSURLSessionDataTask *task = [[NSURLSession sharedSession] 
        dataTaskWithURL:imageURL 
      completionHandler:^(NSData *data, NSURLResponse *response, NSError *error) {
        if (data && !error) {
            UIImage *image = [UIImage imageWithData:data];
            dispatch_async(dispatch_get_main_queue(), ^{
                // ตรวจสอบว่า cell ยังอยู่ใน screen
                UITableViewCell *currentCell = [tableView cellForRowAtIndexPath:indexPath];
                if (currentCell) {
                    currentCell.imageView.image = image;
                    [currentCell setNeedsLayout];
                }
            });
        }
    }];
    [task resume];
    
    return cell;
}
```

---

## 18. แบบฝึกหัด (Practice Exercises)

### Exercise 1: To-Do List
สร้าง to-do list app ที่มีฟีเจอร์:
- เพิ่ม task ใหม่
- ลบ task ด้วย swipe
- mark as done ด้วย checkmark
- แบ่ง section ตาม pending/completed

```objc
// Template
@interface Task : NSObject
@property (nonatomic, copy) NSString *title;
@property (nonatomic, assign) BOOL isCompleted;
@property (nonatomic, strong) NSDate *createdAt;
@end

@interface TodoViewController : UITableViewController
// TODO: implement to-do list
@end
```

### Exercise 2: Expandable Sections
สร้าง table view ที่ section header สามารถกด expand/collapse ได้

```objc
@interface ExpandableSection : NSObject
@property (nonatomic, copy) NSString *title;
@property (nonatomic, strong) NSArray<NSString *> *items;
@property (nonatomic, assign) BOOL isExpanded;
@end

// คำใบ้: เมื่อ tap header ให้ toggle isExpanded แล้ว reloadSections
```

### Exercise 3: Multi-Selection
สร้าง table view ที่เลือกได้หลาย rows พร้อมปุ่ม delete ที่ navigation bar

```objc
// คำใบ้:
// - tableView.allowsMultipleSelectionDuringEditing = YES
// - tableView.indexPathsForSelectedRows
// - navigationItem.rightBarButtonItem แสดง/ซ่อนตาม selection count
```

### Exercise 4: Grouped Settings Screen
สร้างหน้า Settings ที่มี:
- Grouped style
- Section headers/footers  
- Switch cells
- Value1 cells
- Detail cells ที่ navigate ไปหน้าถัดไป

### Exercise 5: Drag and Drop Reordering
เพิ่ม drag-and-drop reordering ให้ list โดยไม่ต้องเปิด editing mode (iOS 11+)

```objc
// คำใบ้: UITableViewDragDelegate, UITableViewDropDelegate
```

---

## สรุป

ในบทนี้เราได้เรียนรู้:

1. **UITableView Setup** - การสร้างทั้งแบบ programmatic และใช้ UITableViewController
2. **DataSource & Delegate** - สองส่วนสำคัญที่ควบคุมข้อมูลและ interaction
3. **Cell Reuse** - pattern สำคัญสำหรับ performance
4. **Custom Cells** - สร้าง cell เองด้วย Auto Layout
5. **Sections** - จัดกลุ่มข้อมูลด้วย sections
6. **Editing** - insert, delete, reorder
7. **Swipe Actions** - actions เมื่อ swipe row
8. **Search** - UISearchController
9. **Pull-to-Refresh** - UIRefreshControl
10. **Infinite Scroll** - โหลดข้อมูลเพิ่มเมื่อ scroll
11. **NSFetchedResultsController** - ทำงานกับ Core Data
12. **Modern APIs** - Diffable Data Source, Context Menu

UITableView เป็น component พื้นฐานที่ต้องเชี่ยวชาญ ยิ่งฝึกใช้มาก ยิ่งเข้าใจ patterns และสามารถสร้าง UI ที่ดีได้
