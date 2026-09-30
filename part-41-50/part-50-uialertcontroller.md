# Part 50: UIAlertController และ Popovers ใน Objective-C

## บทนำ (Introduction)

`UIAlertController` เป็น class หลักที่ใช้แสดง dialog และ action sheet ใน iOS ตั้งแต่ iOS 8 เป็นต้นมา มันแทนที่ `UIAlertView` และ `UIActionSheet` เดิมที่ถูก deprecated แล้ว

ในบทนี้เราจะเรียนรู้:
- UIAlertController แบบ Alert และ Action Sheet
- UIAlertAction ประเภทต่างๆ
- การเพิ่ม text fields ใน alerts
- UIPopoverPresentationController
- UIActivityViewController
- UIDocumentPickerViewController
- UIImagePickerController

---

## 50.1 UIAlertController พื้นฐาน

### Alert Style

Alert แสดงขึ้นตรงกลางหน้าจอ ใช้สำหรับข้อความสำคัญที่ต้องการการตอบสนองจากผู้ใช้

```objc
- (void)showBasicAlert {
    UIAlertController *alert = [UIAlertController 
        alertControllerWithTitle:@"แจ้งเตือน"
        message:@"คุณต้องการดำเนินการต่อหรือไม่?"
        preferredStyle:UIAlertControllerStyleAlert];
    
    // ปุ่ม OK
    UIAlertAction *okAction = [UIAlertAction 
        actionWithTitle:@"ตกลง"
        style:UIAlertActionStyleDefault
        handler:^(UIAlertAction *action) {
            NSLog(@"ผู้ใช้กด ตกลง");
        }];
    
    // ปุ่ม Cancel
    UIAlertAction *cancelAction = [UIAlertAction 
        actionWithTitle:@"ยกเลิก"
        style:UIAlertActionStyleCancel
        handler:^(UIAlertAction *action) {
            NSLog(@"ผู้ใช้กด ยกเลิก");
        }];
    
    [alert addAction:okAction];
    [alert addAction:cancelAction];
    
    [self presentViewController:alert animated:YES completion:nil];
}
```

### Action Sheet Style

Action Sheet แสดงขึ้นจากด้านล่างหน้าจอ (iPhone) ใช้สำหรับแสดงตัวเลือกหลายๆ อย่าง

```objc
- (void)showActionSheet {
    UIAlertController *actionSheet = [UIAlertController 
        alertControllerWithTitle:@"เลือกการกระทำ"
        message:@"เลือกสิ่งที่ต้องการทำ"
        preferredStyle:UIAlertControllerStyleActionSheet];
    
    UIAlertAction *saveAction = [UIAlertAction 
        actionWithTitle:@"บันทึก"
        style:UIAlertActionStyleDefault
        handler:^(UIAlertAction *action) {
            NSLog(@"บันทึก");
        }];
    
    UIAlertAction *shareAction = [UIAlertAction 
        actionWithTitle:@"แชร์"
        style:UIAlertActionStyleDefault
        handler:^(UIAlertAction *action) {
            NSLog(@"แชร์");
        }];
    
    UIAlertAction *deleteAction = [UIAlertAction 
        actionWithTitle:@"ลบ"
        style:UIAlertActionStyleDestructive  // สีแดง
        handler:^(UIAlertAction *action) {
            NSLog(@"ลบ");
        }];
    
    UIAlertAction *cancelAction = [UIAlertAction 
        actionWithTitle:@"ยกเลิก"
        style:UIAlertActionStyleCancel
        handler:nil];
    
    [actionSheet addAction:saveAction];
    [actionSheet addAction:shareAction];
    [actionSheet addAction:deleteAction];
    [actionSheet addAction:cancelAction];  // Cancel มักใส่ไว้ท้ายสุด
    
    [self presentViewController:actionSheet animated:YES completion:nil];
}
```

---

## 50.2 UIAlertAction ประเภทต่างๆ

```objc
typedef NS_ENUM(NSInteger, UIAlertActionStyle) {
    UIAlertActionStyleDefault = 0,    // สีน้ำเงิน (ค่าเริ่มต้น)
    UIAlertActionStyleCancel,         // หนา (bold)
    UIAlertActionStyleDestructive     // สีแดง (สำหรับการกระทำที่ทำลาย)
};
```

### ตัวอย่างการใช้แต่ละ Style

```objc
- (void)demonstrateActionStyles {
    UIAlertController *alert = [UIAlertController 
        alertControllerWithTitle:@"Action Styles"
        message:@"สาธิต action styles ต่างๆ"
        preferredStyle:UIAlertControllerStyleAlert];
    
    // Default - สำหรับการกระทำทั่วไป
    [alert addAction:[UIAlertAction actionWithTitle:@"Default Action"
                                             style:UIAlertActionStyleDefault
                                           handler:nil]];
    
    // Destructive - สำหรับการกระทำที่ไม่สามารถย้อนกลับได้
    [alert addAction:[UIAlertAction actionWithTitle:@"Delete (Destructive)"
                                             style:UIAlertActionStyleDestructive
                                           handler:^(UIAlertAction *action) {
        NSLog(@"ลบแล้ว!");
    }]];
    
    // Cancel - ปิด alert โดยไม่ทำอะไร
    [alert addAction:[UIAlertAction actionWithTitle:@"Cancel"
                                             style:UIAlertActionStyleCancel
                                           handler:nil]];
    
    [self presentViewController:alert animated:YES completion:nil];
}
```

### Preferred Action

```objc
- (void)showAlertWithPreferredAction {
    UIAlertController *alert = [UIAlertController 
        alertControllerWithTitle:@"บันทึก?"
        message:@"ต้องการบันทึกการเปลี่ยนแปลงหรือไม่?"
        preferredStyle:UIAlertControllerStyleAlert];
    
    UIAlertAction *saveAction = [UIAlertAction actionWithTitle:@"บันทึก"
                                                         style:UIAlertActionStyleDefault
                                                       handler:^(UIAlertAction *action) {
        [self saveChanges];
    }];
    
    UIAlertAction *discardAction = [UIAlertAction actionWithTitle:@"ไม่บันทึก"
                                                            style:UIAlertActionStyleDestructive
                                                          handler:^(UIAlertAction *action) {
        [self discardChanges];
    }];
    
    UIAlertAction *cancelAction = [UIAlertAction actionWithTitle:@"ยกเลิก"
                                                           style:UIAlertActionStyleCancel
                                                         handler:nil];
    
    [alert addAction:saveAction];
    [alert addAction:discardAction];
    [alert addAction:cancelAction];
    
    // กำหนด preferred action (จะแสดงเป็น bold)
    alert.preferredAction = saveAction;
    
    [self presentViewController:alert animated:YES completion:nil];
}
```

### Disable Action

```objc
- (void)showAlertWithDisabledAction {
    UIAlertController *alert = [UIAlertController 
        alertControllerWithTitle:@"ยืนยัน"
        message:@"กรุณายืนยันการดำเนินการ"
        preferredStyle:UIAlertControllerStyleAlert];
    
    UIAlertAction *confirmAction = [UIAlertAction actionWithTitle:@"ยืนยัน"
                                                            style:UIAlertActionStyleDefault
                                                          handler:^(UIAlertAction *action) {
        NSLog(@"ยืนยันแล้ว");
    }];
    
    // Disable action เริ่มต้น
    confirmAction.enabled = NO;
    
    UIAlertAction *cancelAction = [UIAlertAction actionWithTitle:@"ยกเลิก"
                                                           style:UIAlertActionStyleCancel
                                                         handler:nil];
    
    [alert addAction:confirmAction];
    [alert addAction:cancelAction];
    
    [self presentViewController:alert animated:YES completion:nil];
}
```

---

## 50.3 Text Fields ใน Alerts

Alert controller รองรับการเพิ่ม text fields เพื่อรับ input จากผู้ใช้

### Alert พร้อม Text Field

```objc
- (void)showAlertWithTextField {
    UIAlertController *alert = [UIAlertController 
        alertControllerWithTitle:@"เพิ่มรายการ"
        message:@"กรุณากรอกชื่อรายการ"
        preferredStyle:UIAlertControllerStyleAlert];
    
    // เพิ่ม text field
    [alert addTextFieldWithConfigurationHandler:^(UITextField *textField) {
        textField.placeholder = @"ชื่อรายการ";
        textField.clearButtonMode = UITextFieldViewModeWhileEditing;
        textField.autocapitalizationType = UITextAutocapitalizationTypeSentences;
    }];
    
    UIAlertAction *addAction = [UIAlertAction actionWithTitle:@"เพิ่ม"
                                                        style:UIAlertActionStyleDefault
                                                      handler:^(UIAlertAction *action) {
        // ดึงค่าจาก text field
        UITextField *textField = alert.textFields.firstObject;
        NSString *itemName = textField.text;
        
        if (itemName.length > 0) {
            NSLog(@"เพิ่มรายการ: %@", itemName);
            [self addItemWithName:itemName];
        }
    }];
    
    UIAlertAction *cancelAction = [UIAlertAction actionWithTitle:@"ยกเลิก"
                                                           style:UIAlertActionStyleCancel
                                                         handler:nil];
    
    [alert addAction:addAction];
    [alert addAction:cancelAction];
    
    [self presentViewController:alert animated:YES completion:nil];
}
```

### Login Alert (Username + Password)

```objc
- (void)showLoginAlert {
    UIAlertController *loginAlert = [UIAlertController 
        alertControllerWithTitle:@"เข้าสู่ระบบ"
        message:@"กรุณากรอกชื่อผู้ใช้และรหัสผ่าน"
        preferredStyle:UIAlertControllerStyleAlert];
    
    // Username field
    [loginAlert addTextFieldWithConfigurationHandler:^(UITextField *textField) {
        textField.placeholder = @"ชื่อผู้ใช้หรืออีเมล";
        textField.keyboardType = UIKeyboardTypeEmailAddress;
        textField.autocorrectionType = UITextAutocorrectionTypeNo;
        textField.autocapitalizationType = UITextAutocapitalizationTypeNone;
    }];
    
    // Password field
    [loginAlert addTextFieldWithConfigurationHandler:^(UITextField *textField) {
        textField.placeholder = @"รหัสผ่าน";
        textField.secureTextEntry = YES;
    }];
    
    UIAlertAction *loginAction = [UIAlertAction actionWithTitle:@"เข้าสู่ระบบ"
                                                          style:UIAlertActionStyleDefault
                                                        handler:^(UIAlertAction *action) {
        UITextField *usernameField = loginAlert.textFields[0];
        UITextField *passwordField = loginAlert.textFields[1];
        
        NSString *username = usernameField.text;
        NSString *password = passwordField.text;
        
        [self loginWithUsername:username password:password];
    }];
    
    UIAlertAction *cancelAction = [UIAlertAction actionWithTitle:@"ยกเลิก"
                                                           style:UIAlertActionStyleCancel
                                                         handler:nil];
    
    [loginAlert addAction:loginAction];
    [loginAlert addAction:cancelAction];
    
    [self presentViewController:loginAlert animated:YES completion:nil];
}
```

### Dynamic Enable/Disable ตาม Text Field Input

```objc
- (void)showAlertWithDynamicAction {
    UIAlertController *alert = [UIAlertController 
        alertControllerWithTitle:@"กรอกชื่อ"
        message:nil
        preferredStyle:UIAlertControllerStyleAlert];
    
    [alert addTextFieldWithConfigurationHandler:^(UITextField *textField) {
        textField.placeholder = @"ชื่อ (ต้องมีอย่างน้อย 3 ตัวอักษร)";
        
        // Observer สำหรับ text changes
        [[NSNotificationCenter defaultCenter] 
            addObserver:self
            selector:@selector(textFieldDidChange:)
            name:UITextFieldTextDidChangeNotification
            object:textField];
    }];
    
    UIAlertAction *confirmAction = [UIAlertAction actionWithTitle:@"ยืนยัน"
                                                            style:UIAlertActionStyleDefault
                                                          handler:^(UIAlertAction *action) {
        NSString *name = alert.textFields.firstObject.text;
        NSLog(@"ชื่อที่กรอก: %@", name);
    }];
    confirmAction.enabled = NO;  // ปิดไว้ก่อน
    
    // เก็บ reference ของ action เพื่อ enable/disable ทีหลัง
    objc_setAssociatedObject(alert, @selector(textFieldDidChange:), 
                              confirmAction, OBJC_ASSOCIATION_RETAIN_NONATOMIC);
    
    [alert addAction:confirmAction];
    [alert addAction:[UIAlertAction actionWithTitle:@"ยกเลิก"
                                              style:UIAlertActionStyleCancel
                                            handler:nil]];
    
    [self presentViewController:alert animated:YES completion:nil];
}

- (void)textFieldDidChange:(NSNotification *)notification {
    UITextField *textField = notification.object;
    
    // หา alert ที่แสดงอยู่
    UIAlertController *alert = (UIAlertController *)self.presentedViewController;
    if ([alert isKindOfClass:[UIAlertController class]]) {
        UIAlertAction *confirmAction = objc_getAssociatedObject(alert, 
            @selector(textFieldDidChange:));
        confirmAction.enabled = textField.text.length >= 3;
    }
}
```

---

## 50.4 Custom Alert Presentations

### สร้าง Custom Alert View Controller

```objc
// CustomAlertViewController.h
@interface CustomAlertViewController : UIViewController
@property (nonatomic, copy) NSString *alertTitle;
@property (nonatomic, copy) NSString *alertMessage;
@property (nonatomic, copy) void (^confirmHandler)(void);
@property (nonatomic, copy) void (^cancelHandler)(void);
+ (instancetype)alertWithTitle:(NSString *)title 
                        message:(NSString *)message;
@end

// CustomAlertViewController.m
@interface CustomAlertViewController ()
@property (nonatomic, strong) UIView *containerView;
@property (nonatomic, strong) UILabel *titleLabel;
@property (nonatomic, strong) UILabel *messageLabel;
@property (nonatomic, strong) UIButton *confirmButton;
@property (nonatomic, strong) UIButton *cancelButton;
@end

@implementation CustomAlertViewController

+ (instancetype)alertWithTitle:(NSString *)title message:(NSString *)message {
    CustomAlertViewController *vc = [[CustomAlertViewController alloc] init];
    vc.alertTitle = title;
    vc.alertMessage = message;
    vc.modalPresentationStyle = UIModalPresentationOverCurrentContext;
    vc.modalTransitionStyle = UIModalTransitionStyleCrossDissolve;
    return vc;
}

- (void)viewDidLoad {
    [super viewDidLoad];
    
    // Background overlay
    self.view.backgroundColor = [[UIColor blackColor] colorWithAlphaComponent:0.5];
    
    // Container
    self.containerView = [[UIView alloc] init];
    self.containerView.translatesAutoresizingMaskIntoConstraints = NO;
    self.containerView.backgroundColor = [UIColor systemBackgroundColor];
    self.containerView.layer.cornerRadius = 16;
    self.containerView.clipsToBounds = YES;
    [self.view addSubview:self.containerView];
    
    // Title
    self.titleLabel = [[UILabel alloc] init];
    self.titleLabel.translatesAutoresizingMaskIntoConstraints = NO;
    self.titleLabel.text = self.alertTitle;
    self.titleLabel.font = [UIFont boldSystemFontOfSize:18];
    self.titleLabel.textAlignment = NSTextAlignmentCenter;
    [self.containerView addSubview:self.titleLabel];
    
    // Message
    self.messageLabel = [[UILabel alloc] init];
    self.messageLabel.translatesAutoresizingMaskIntoConstraints = NO;
    self.messageLabel.text = self.alertMessage;
    self.messageLabel.font = [UIFont systemFontOfSize:14];
    self.messageLabel.textAlignment = NSTextAlignmentCenter;
    self.messageLabel.numberOfLines = 0;
    self.messageLabel.textColor = [UIColor systemGrayColor];
    [self.containerView addSubview:self.messageLabel];
    
    // Divider
    UIView *divider = [[UIView alloc] init];
    divider.translatesAutoresizingMaskIntoConstraints = NO;
    divider.backgroundColor = [UIColor separatorColor];
    [self.containerView addSubview:divider];
    
    // Confirm button
    self.confirmButton = [UIButton buttonWithType:UIButtonTypeSystem];
    self.confirmButton.translatesAutoresizingMaskIntoConstraints = NO;
    [self.confirmButton setTitle:@"ยืนยัน" forState:UIControlStateNormal];
    self.confirmButton.titleLabel.font = [UIFont boldSystemFontOfSize:17];
    [self.confirmButton addTarget:self action:@selector(confirmTapped) 
                 forControlEvents:UIControlEventTouchUpInside];
    [self.containerView addSubview:self.confirmButton];
    
    // Cancel button
    self.cancelButton = [UIButton buttonWithType:UIButtonTypeSystem];
    self.cancelButton.translatesAutoresizingMaskIntoConstraints = NO;
    [self.cancelButton setTitle:@"ยกเลิก" forState:UIControlStateNormal];
    [self.cancelButton setTitleColor:[UIColor systemGrayColor] forState:UIControlStateNormal];
    [self.cancelButton addTarget:self action:@selector(cancelTapped) 
                forControlEvents:UIControlEventTouchUpInside];
    [self.containerView addSubview:self.cancelButton];
    
    // Vertical divider between buttons
    UIView *vDivider = [[UIView alloc] init];
    vDivider.translatesAutoresizingMaskIntoConstraints = NO;
    vDivider.backgroundColor = [UIColor separatorColor];
    [self.containerView addSubview:vDivider];
    
    [NSLayoutConstraint activateConstraints:@[
        // Container
        [self.containerView.centerXAnchor constraintEqualToAnchor:self.view.centerXAnchor],
        [self.containerView.centerYAnchor constraintEqualToAnchor:self.view.centerYAnchor],
        [self.containerView.widthAnchor constraintEqualToConstant:270],
        
        // Title
        [self.titleLabel.topAnchor constraintEqualToAnchor:self.containerView.topAnchor constant:20],
        [self.titleLabel.leadingAnchor constraintEqualToAnchor:self.containerView.leadingAnchor constant:16],
        [self.titleLabel.trailingAnchor constraintEqualToAnchor:self.containerView.trailingAnchor constant:-16],
        
        // Message
        [self.messageLabel.topAnchor constraintEqualToAnchor:self.titleLabel.bottomAnchor constant:10],
        [self.messageLabel.leadingAnchor constraintEqualToAnchor:self.containerView.leadingAnchor constant:16],
        [self.messageLabel.trailingAnchor constraintEqualToAnchor:self.containerView.trailingAnchor constant:-16],
        
        // Horizontal divider
        [divider.topAnchor constraintEqualToAnchor:self.messageLabel.bottomAnchor constant:20],
        [divider.leadingAnchor constraintEqualToAnchor:self.containerView.leadingAnchor],
        [divider.trailingAnchor constraintEqualToAnchor:self.containerView.trailingAnchor],
        [divider.heightAnchor constraintEqualToConstant:0.5],
        
        // Buttons
        [self.cancelButton.topAnchor constraintEqualToAnchor:divider.bottomAnchor],
        [self.cancelButton.leadingAnchor constraintEqualToAnchor:self.containerView.leadingAnchor],
        [self.cancelButton.heightAnchor constraintEqualToConstant:44],
        [self.cancelButton.bottomAnchor constraintEqualToAnchor:self.containerView.bottomAnchor],
        
        [vDivider.topAnchor constraintEqualToAnchor:divider.bottomAnchor],
        [vDivider.widthAnchor constraintEqualToConstant:0.5],
        [vDivider.leadingAnchor constraintEqualToAnchor:self.cancelButton.trailingAnchor],
        [vDivider.bottomAnchor constraintEqualToAnchor:self.containerView.bottomAnchor],
        
        [self.confirmButton.topAnchor constraintEqualToAnchor:divider.bottomAnchor],
        [self.confirmButton.leadingAnchor constraintEqualToAnchor:vDivider.trailingAnchor],
        [self.confirmButton.trailingAnchor constraintEqualToAnchor:self.containerView.trailingAnchor],
        [self.confirmButton.widthAnchor constraintEqualToAnchor:self.cancelButton.widthAnchor],
        [self.confirmButton.heightAnchor constraintEqualToConstant:44],
        [self.confirmButton.bottomAnchor constraintEqualToAnchor:self.containerView.bottomAnchor],
    ]];
}

- (void)confirmTapped {
    [self dismissViewControllerAnimated:YES completion:^{
        if (self.confirmHandler) {
            self.confirmHandler();
        }
    }];
}

- (void)cancelTapped {
    [self dismissViewControllerAnimated:YES completion:^{
        if (self.cancelHandler) {
            self.cancelHandler();
        }
    }];
}

// Animate in
- (void)viewWillAppear:(BOOL)animated {
    [super viewWillAppear:animated];
    self.containerView.transform = CGAffineTransformMakeScale(0.8, 0.8);
    self.containerView.alpha = 0;
    
    [UIView animateWithDuration:0.3
                          delay:0
         usingSpringWithDamping:0.8
          initialSpringVelocity:0.5
                        options:0
                     animations:^{
        self.containerView.transform = CGAffineTransformIdentity;
        self.containerView.alpha = 1;
    } completion:nil];
}

@end
```

### การใช้งาน Custom Alert

```objc
- (void)showCustomAlert {
    CustomAlertViewController *alert = [CustomAlertViewController 
        alertWithTitle:@"ยืนยันการลบ"
                message:@"คุณแน่ใจหรือไม่ที่จะลบรายการนี้? การดำเนินการนี้ไม่สามารถย้อนกลับได้"];
    
    alert.confirmHandler = ^{
        [self deleteItem];
    };
    
    alert.cancelHandler = ^{
        NSLog(@"ยกเลิกการลบ");
    };
    
    [self presentViewController:alert animated:YES completion:nil];
}
```

---

## 50.5 UIPopoverPresentationController

Popover แสดงเนื้อหาเหมือน popup bubble ที่ชี้ไปยัง anchor point (iPad สนับสนุน popover เต็มรูปแบบ บน iPhone จะแสดงเป็น action sheet หรือ fullscreen)

```objc
- (void)showPopover:(UIButton *)sender {
    UIViewController *popoverContent = [[UIViewController alloc] init];
    popoverContent.preferredContentSize = CGSizeMake(300, 200);
    popoverContent.view.backgroundColor = [UIColor systemBackgroundColor];
    
    // เพิ่ม label
    UILabel *label = [[UILabel alloc] init];
    label.text = @"Popover Content";
    label.translatesAutoresizingMaskIntoConstraints = NO;
    [popoverContent.view addSubview:label];
    [NSLayoutConstraint activateConstraints:@[
        [label.centerXAnchor constraintEqualToAnchor:popoverContent.view.centerXAnchor],
        [label.centerYAnchor constraintEqualToAnchor:popoverContent.view.centerYAnchor],
    ]];
    
    // ตั้งค่า presentation style
    popoverContent.modalPresentationStyle = UIModalPresentationPopover;
    
    // Configure popover
    UIPopoverPresentationController *popoverPC = popoverContent.popoverPresentationController;
    popoverPC.sourceView = sender;  // view ที่ popover ชี้ไป
    popoverPC.sourceRect = sender.bounds;  // rect ภายใน sourceView
    popoverPC.permittedArrowDirections = UIPopoverArrowDirectionAny;  // ทิศทางลูกศร
    popoverPC.delegate = self;
    
    [self presentViewController:popoverContent animated:YES completion:nil];
}

// UIPopoverPresentationControllerDelegate
- (UIModalPresentationStyle)adaptivePresentationStyleForPresentationController:(UIPresentationController *)controller {
    // บน iPhone บังคับให้แสดงเป็น popover (ไม่ใช่ fullscreen)
    return UIModalPresentationNone;
}
```

### Popover จาก BarButtonItem

```objc
- (void)showPopoverFromBarButton:(UIBarButtonItem *)barButton {
    UIViewController *content = [[UIViewController alloc] init];
    content.preferredContentSize = CGSizeMake(250, 150);
    content.modalPresentationStyle = UIModalPresentationPopover;
    
    UIPopoverPresentationController *popover = content.popoverPresentationController;
    popover.barButtonItem = barButton;  // ชี้ไปที่ bar button
    popover.permittedArrowDirections = UIPopoverArrowDirectionUp;
    
    [self presentViewController:content animated:YES completion:nil];
}
```

### Popover พร้อม TableView

```objc
// PopoverTableViewController.h
@interface PopoverTableViewController : UITableViewController
@property (nonatomic, strong) NSArray<NSString *> *options;
@property (nonatomic, copy) void (^selectionHandler)(NSString *option, NSInteger index);
@end

// PopoverTableViewController.m
@implementation PopoverTableViewController

- (void)viewDidLoad {
    [super viewDidLoad];
    [self.tableView registerClass:[UITableViewCell class] 
             forCellReuseIdentifier:@"Cell"];
}

- (NSInteger)tableView:(UITableView *)tableView numberOfRowsInSection:(NSInteger)section {
    return self.options.count;
}

- (UITableViewCell *)tableView:(UITableView *)tableView 
         cellForRowAtIndexPath:(NSIndexPath *)indexPath {
    UITableViewCell *cell = [tableView dequeueReusableCellWithIdentifier:@"Cell" 
                                                            forIndexPath:indexPath];
    cell.textLabel.text = self.options[indexPath.row];
    return cell;
}

- (void)tableView:(UITableView *)tableView didSelectRowAtIndexPath:(NSIndexPath *)indexPath {
    [tableView deselectRowAtIndexPath:indexPath animated:YES];
    
    if (self.selectionHandler) {
        self.selectionHandler(self.options[indexPath.row], indexPath.row);
    }
    
    [self dismissViewControllerAnimated:YES completion:nil];
}

@end

// การใช้งาน
- (void)showOptionsPopover:(UIButton *)sender {
    PopoverTableViewController *popoverVC = [[PopoverTableViewController alloc] init];
    popoverVC.options = @[@"ตัวเลือกที่ 1", @"ตัวเลือกที่ 2", @"ตัวเลือกที่ 3"];
    popoverVC.preferredContentSize = CGSizeMake(200, 132); // 44 × 3 rows
    popoverVC.modalPresentationStyle = UIModalPresentationPopover;
    
    popoverVC.selectionHandler = ^(NSString *option, NSInteger index) {
        NSLog(@"เลือก: %@ (index: %ld)", option, (long)index);
    };
    
    UIPopoverPresentationController *pc = popoverVC.popoverPresentationController;
    pc.sourceView = sender;
    pc.sourceRect = sender.bounds;
    pc.delegate = self;
    
    [self presentViewController:popoverVC animated:YES completion:nil];
}
```

---

## 50.6 UIActivityViewController

`UIActivityViewController` แสดง share sheet ให้ผู้ใช้แชร์เนื้อหาผ่านแอปหรือบริการต่างๆ

```objc
- (void)shareContent {
    NSString *textToShare = @"ข้อความที่ต้องการแชร์";
    NSURL *urlToShare = [NSURL URLWithString:@"https://www.example.com"];
    UIImage *imageToShare = [UIImage systemImageNamed:@"photo"];
    
    NSArray *activityItems = @[textToShare, urlToShare, imageToShare];
    
    UIActivityViewController *activityVC = [[UIActivityViewController alloc] 
        initWithActivityItems:activityItems 
        applicationActivities:nil];
    
    // Exclude บางกิจกรรม
    activityVC.excludedActivityTypes = @[
        UIActivityTypeAddToReadingList,
        UIActivityTypeAssignToContact,
        UIActivityTypeOpenInIBooks,
    ];
    
    // Completion handler
    activityVC.completionWithItemsHandler = ^(UIActivityType activityType, 
                                               BOOL completed, 
                                               NSArray *returnedItems, 
                                               NSError *error) {
        if (completed) {
            NSLog(@"แชร์สำเร็จด้วย: %@", activityType);
        } else {
            NSLog(@"ยกเลิกการแชร์");
        }
    };
    
    // บน iPad ต้องกำหนด sourceView
    if (UIDevice.currentDevice.userInterfaceIdiom == UIUserInterfaceIdiomPad) {
        activityVC.popoverPresentationController.sourceView = self.view;
        activityVC.popoverPresentationController.sourceRect = 
            CGRectMake(self.view.bounds.size.width / 2, 
                       self.view.bounds.size.height / 2, 
                       0, 0);
    }
    
    [self presentViewController:activityVC animated:YES completion:nil];
}
```

### Share จาก UIButton

```objc
- (void)shareButtonTapped:(UIButton *)sender {
    NSArray *items = @[@"ดูสินค้าใหม่ที่น่าสนใจ!", 
                       [NSURL URLWithString:@"https://shop.example.com"]];
    
    UIActivityViewController *vc = [[UIActivityViewController alloc] 
        initWithActivityItems:items 
        applicationActivities:nil];
    
    // สำหรับ iPhone
    if ([UIDevice currentDevice].userInterfaceIdiom == UIUserInterfaceIdiomPhone) {
        [self presentViewController:vc animated:YES completion:nil];
    }
    // สำหรับ iPad
    else {
        vc.popoverPresentationController.sourceView = sender;
        vc.popoverPresentationController.sourceRect = sender.bounds;
        [self presentViewController:vc animated:YES completion:nil];
    }
}
```

### Custom Activity

```objc
// CustomActivity.h
@interface CustomActivity : UIActivity
@end

// CustomActivity.m
@implementation CustomActivity

- (NSString *)activityType {
    return @"com.myapp.custom-activity";
}

- (NSString *)activityTitle {
    return @"บันทึกในแอป";
}

- (UIImage *)activityImage {
    return [UIImage systemImageNamed:@"square.and.arrow.down"];
}

- (BOOL)canPerformWithActivityItems:(NSArray *)activityItems {
    return YES;
}

- (void)prepareWithActivityItems:(NSArray *)activityItems {
    // เตรียม items ที่จะใช้งาน
}

- (void)performActivity {
    // ทำงานที่ต้องการ
    NSLog(@"Custom activity performed!");
    [self activityDidFinish:YES];
}

+ (UIActivityCategory)activityCategory {
    return UIActivityCategoryAction;  // หรือ UIActivityCategoryShare
}

@end

// การใช้งาน
- (void)shareWithCustomActivity {
    CustomActivity *customActivity = [[CustomActivity alloc] init];
    
    UIActivityViewController *vc = [[UIActivityViewController alloc] 
        initWithActivityItems:@[@"เนื้อหา"]
        applicationActivities:@[customActivity]];
    
    [self presentViewController:vc animated:YES completion:nil];
}
```

---

## 50.7 UIDocumentPickerViewController

ใช้สำหรับเปิด/บันทึกไฟล์จาก Files app และ cloud storage providers

```objc
#import <UniformTypeIdentifiers/UniformTypeIdentifiers.h>

- (void)openDocumentPicker {
    // กำหนดประเภทไฟล์ที่รองรับ
    NSArray<UTType *> *contentTypes = @[
        UTTypeImage,
        UTTypePDF,
        UTTypePlainText,
    ];
    
    UIDocumentPickerViewController *picker = [[UIDocumentPickerViewController alloc] 
        initForOpeningContentTypes:contentTypes
        asCopy:YES];  // YES = copy ไฟล์มาใน app sandbox, NO = access โดยตรง
    
    picker.delegate = self;
    picker.allowsMultipleSelection = NO;  // อนุญาตเลือกหลายไฟล์หรือไม่
    picker.shouldShowFileExtensions = YES;
    
    [self presentViewController:picker animated:YES completion:nil];
}

// UIDocumentPickerDelegate
- (void)documentPicker:(UIDocumentPickerViewController *)controller 
didPickDocumentsAtURLs:(NSArray<NSURL *> *)urls {
    NSURL *selectedURL = urls.firstObject;
    
    if ([selectedURL startAccessingSecurityScopedResource]) {
        // อ่านไฟล์
        NSData *data = [NSData dataWithContentsOfURL:selectedURL];
        NSLog(@"ไฟล์ขนาด: %lu bytes", (unsigned long)data.length);
        
        [selectedURL stopAccessingSecurityScopedResource];
    }
}

- (void)documentPickerWasCancelled:(UIDocumentPickerViewController *)controller {
    NSLog(@"ผู้ใช้ยกเลิก");
}
```

### Export/Save Document

```objc
- (void)saveDocumentToFiles {
    // สร้างไฟล์ใน temp directory ก่อน
    NSString *tempPath = [NSTemporaryDirectory() 
        stringByAppendingPathComponent:@"document.txt"];
    
    NSString *content = @"เนื้อหาของเอกสาร";
    NSError *error;
    [content writeToFile:tempPath 
              atomically:YES 
                encoding:NSUTF8StringEncoding 
                   error:&error];
    
    if (!error) {
        NSURL *fileURL = [NSURL fileURLWithPath:tempPath];
        
        UIDocumentPickerViewController *picker = [[UIDocumentPickerViewController alloc] 
            initForExportingURLs:@[fileURL]
            asCopy:YES];
        
        picker.delegate = self;
        [self presentViewController:picker animated:YES completion:nil];
    }
}
```

---

## 50.8 UIImagePickerController

ใช้สำหรับถ่ายรูปหรือเลือกรูปจาก photo library

```objc
#import <Photos/Photos.h>

- (void)showImagePickerOptions {
    UIAlertController *alert = [UIAlertController 
        alertControllerWithTitle:@"เลือกรูปภาพ"
        message:nil
        preferredStyle:UIAlertControllerStyleActionSheet];
    
    // ถ่ายรูปด้วยกล้อง
    if ([UIImagePickerController isSourceTypeAvailable:UIImagePickerControllerSourceTypeCamera]) {
        [alert addAction:[UIAlertAction actionWithTitle:@"ถ่ายรูป"
                                                  style:UIAlertActionStyleDefault
                                                handler:^(UIAlertAction *action) {
            [self presentImagePickerWithSource:UIImagePickerControllerSourceTypeCamera];
        }]];
    }
    
    // เลือกจาก Photo Library
    [alert addAction:[UIAlertAction actionWithTitle:@"เลือกจากคลัง"
                                              style:UIAlertActionStyleDefault
                                            handler:^(UIAlertAction *action) {
        [self presentImagePickerWithSource:UIImagePickerControllerSourceTypePhotoLibrary];
    }]];
    
    [alert addAction:[UIAlertAction actionWithTitle:@"ยกเลิก"
                                              style:UIAlertActionStyleCancel
                                            handler:nil]];
    
    // iPad support
    alert.popoverPresentationController.sourceView = self.view;
    alert.popoverPresentationController.sourceRect = CGRectMake(
        self.view.bounds.size.width / 2, self.view.bounds.size.height / 2, 0, 0);
    
    [self presentViewController:alert animated:YES completion:nil];
}

- (void)presentImagePickerWithSource:(UIImagePickerControllerSourceType)sourceType {
    // ตรวจสอบ permission ก่อน (สำหรับ Photo Library)
    if (sourceType == UIImagePickerControllerSourceTypePhotoLibrary) {
        PHAuthorizationStatus status = [PHPhotoLibrary authorizationStatus];
        if (status == PHAuthorizationStatusNotDetermined) {
            [PHPhotoLibrary requestAuthorizationWithCompletionHandler:^(PHAuthorizationStatus status) {
                if (status == PHAuthorizationStatusAuthorized) {
                    dispatch_async(dispatch_get_main_queue(), ^{
                        [self presentImagePickerWithSource:sourceType];
                    });
                }
            }];
            return;
        }
        
        if (status != PHAuthorizationStatusAuthorized) {
            [self showPermissionAlert];
            return;
        }
    }
    
    UIImagePickerController *picker = [[UIImagePickerController alloc] init];
    picker.sourceType = sourceType;
    picker.delegate = self;
    picker.allowsEditing = YES;  // อนุญาตการ crop รูป
    
    if (sourceType == UIImagePickerControllerSourceTypeCamera) {
        picker.cameraCaptureMode = UIImagePickerControllerCameraCaptureModePhoto;
        picker.cameraDevice = UIImagePickerControllerCameraDeviceRear;
    }
    
    [self presentViewController:picker animated:YES completion:nil];
}

// UIImagePickerControllerDelegate
- (void)imagePickerController:(UIImagePickerController *)picker 
didFinishPickingMediaWithInfo:(NSDictionary<UIImagePickerControllerInfoKey, id> *)info {
    
    // รูปที่ถูก edit (ถ้า allowsEditing = YES)
    UIImage *editedImage = info[UIImagePickerControllerEditedImage];
    // รูปต้นฉบับ
    UIImage *originalImage = info[UIImagePickerControllerOriginalImage];
    
    UIImage *selectedImage = editedImage ?: originalImage;
    
    if (selectedImage) {
        self.imageView.image = selectedImage;
        
        // บันทึก metadata
        if (@available(iOS 11.0, *)) {
            NSURL *imageURL = info[UIImagePickerControllerImageURL];
            NSLog(@"Image URL: %@", imageURL);
        }
    }
    
    [picker dismissViewControllerAnimated:YES completion:nil];
}

- (void)imagePickerControllerDidCancel:(UIImagePickerController *)picker {
    [picker dismissViewControllerAnimated:YES completion:nil];
}
```

### PHPickerViewController (iOS 14+)

Modern approach ที่ไม่ต้องขอ permission สำหรับ basic selection:

```objc
#import <PhotosUI/PhotosUI.h>

- (void)presentPHPicker {
    PHPickerConfiguration *config = [[PHPickerConfiguration alloc] init];
    config.filter = [PHPickerFilter imagesFilter];  // รูปภาพเท่านั้น
    config.selectionLimit = 1;  // เลือกได้ 1 รูป (0 = ไม่จำกัด)
    
    PHPickerViewController *picker = [[PHPickerViewController alloc] initWithConfiguration:config];
    picker.delegate = self;
    
    [self presentViewController:picker animated:YES completion:nil];
}

// PHPickerViewControllerDelegate
- (void)picker:(PHPickerViewController *)picker 
didFinishPicking:(NSArray<PHPickerResult *> *)results {
    [picker dismissViewControllerAnimated:YES completion:nil];
    
    if (results.count == 0) return;
    
    PHPickerResult *result = results.firstObject;
    
    if ([result.itemProvider canLoadObjectOfClass:[UIImage class]]) {
        [result.itemProvider loadObjectOfClass:[UIImage class] 
                             completionHandler:^(__kindof id<NSItemProviderReading> object, 
                                                NSError *error) {
            if ([object isKindOfClass:[UIImage class]]) {
                UIImage *image = (UIImage *)object;
                dispatch_async(dispatch_get_main_queue(), ^{
                    self.imageView.image = image;
                });
            }
        }];
    }
}
```

---

## 50.9 Action Sheet บน iPad

บน iPad Action Sheet จะแสดงเป็น Popover อัตโนมัติ ต้องกำหนด sourceView/sourceRect:

```objc
- (void)showActionSheetOnIPad:(UIButton *)sender {
    UIAlertController *actionSheet = [UIAlertController 
        alertControllerWithTitle:@"ตัวเลือก"
        message:nil
        preferredStyle:UIAlertControllerStyleActionSheet];
    
    [actionSheet addAction:[UIAlertAction actionWithTitle:@"ดาวน์โหลด"
                                                    style:UIAlertActionStyleDefault
                                                  handler:nil]];
    
    [actionSheet addAction:[UIAlertAction actionWithTitle:@"ลบ"
                                                    style:UIAlertActionStyleDestructive
                                                  handler:nil]];
    
    [actionSheet addAction:[UIAlertAction actionWithTitle:@"ยกเลิก"
                                                    style:UIAlertActionStyleCancel
                                                  handler:nil]];
    
    // iPad REQUIRED: กำหนด source
    UIPopoverPresentationController *popoverPC = actionSheet.popoverPresentationController;
    if (popoverPC) {
        popoverPC.sourceView = sender;
        popoverPC.sourceRect = sender.bounds;
    }
    
    [self presentViewController:actionSheet animated:YES completion:nil];
}
```

### Universal Method (iPhone + iPad)

```objc
- (void)showActionSheetFromView:(UIView *)sourceView {
    UIAlertController *sheet = [UIAlertController 
        alertControllerWithTitle:nil
        message:nil
        preferredStyle:UIAlertControllerStyleActionSheet];
    
    // เพิ่ม actions...
    [sheet addAction:[UIAlertAction actionWithTitle:@"ตัวเลือก 1"
                                              style:UIAlertActionStyleDefault
                                            handler:nil]];
    [sheet addAction:[UIAlertAction actionWithTitle:@"ยกเลิก"
                                              style:UIAlertActionStyleCancel
                                            handler:nil]];
    
    // ตรวจสอบ popover controller (จะไม่เป็น nil บน iPad)
    if (sheet.popoverPresentationController) {
        sheet.popoverPresentationController.sourceView = sourceView;
        sheet.popoverPresentationController.sourceRect = sourceView.bounds;
    }
    
    [self presentViewController:sheet animated:YES completion:nil];
}
```

---

## 50.10 ตัวอย่างครบถ้วน - Context Menu / Action Handling

```objc
#import "ContextMenuViewController.h"

@interface ContextMenuViewController () <UITableViewDataSource, UITableViewDelegate,
                                          UIPopoverPresentationControllerDelegate>
@property (nonatomic, strong) UITableView *tableView;
@property (nonatomic, strong) NSMutableArray<NSString *> *items;
@end

@implementation ContextMenuViewController

- (void)viewDidLoad {
    [super viewDidLoad];
    self.title = @"รายการ";
    self.view.backgroundColor = [UIColor systemBackgroundColor];
    
    self.items = [NSMutableArray arrayWithArray:@[
        @"รายการที่ 1", @"รายการที่ 2", @"รายการที่ 3",
        @"รายการที่ 4", @"รายการที่ 5"
    ]];
    
    [self setupTableView];
    [self setupNavigationBar];
}

- (void)setupTableView {
    self.tableView = [[UITableView alloc] initWithFrame:CGRectZero style:UITableViewStyleInsetGrouped];
    self.tableView.translatesAutoresizingMaskIntoConstraints = NO;
    self.tableView.dataSource = self;
    self.tableView.delegate = self;
    [self.tableView registerClass:[UITableViewCell class] forCellReuseIdentifier:@"Cell"];
    [self.view addSubview:self.tableView];
    
    [NSLayoutConstraint activateConstraints:@[
        [self.tableView.topAnchor constraintEqualToAnchor:self.view.safeAreaLayoutGuide.topAnchor],
        [self.tableView.leadingAnchor constraintEqualToAnchor:self.view.leadingAnchor],
        [self.tableView.trailingAnchor constraintEqualToAnchor:self.view.trailingAnchor],
        [self.tableView.bottomAnchor constraintEqualToAnchor:self.view.bottomAnchor],
    ]];
}

- (void)setupNavigationBar {
    UIBarButtonItem *addButton = [[UIBarButtonItem alloc] 
        initWithBarButtonSystemItem:UIBarButtonSystemItemAdd
        target:self
        action:@selector(addItemTapped:)];
    self.navigationItem.rightBarButtonItem = addButton;
}

- (void)addItemTapped:(UIBarButtonItem *)sender {
    UIAlertController *alert = [UIAlertController 
        alertControllerWithTitle:@"เพิ่มรายการ"
        message:nil
        preferredStyle:UIAlertControllerStyleAlert];
    
    [alert addTextFieldWithConfigurationHandler:^(UITextField *textField) {
        textField.placeholder = @"ชื่อรายการ";
    }];
    
    UIAlertAction *addAction = [UIAlertAction actionWithTitle:@"เพิ่ม"
                                                        style:UIAlertActionStyleDefault
                                                      handler:^(UIAlertAction *action) {
        NSString *name = alert.textFields.firstObject.text;
        if (name.length > 0) {
            [self.items addObject:name];
            [self.tableView insertRowsAtIndexPaths:@[
                [NSIndexPath indexPathForRow:self.items.count - 1 inSection:0]
            ] withRowAnimation:UITableViewRowAnimationAutomatic];
        }
    }];
    
    UIAlertAction *cancelAction = [UIAlertAction actionWithTitle:@"ยกเลิก"
                                                           style:UIAlertActionStyleCancel
                                                         handler:nil];
    
    [alert addAction:addAction];
    [alert addAction:cancelAction];
    
    [self presentViewController:alert animated:YES completion:nil];
}

#pragma mark - UITableViewDataSource

- (NSInteger)tableView:(UITableView *)tableView numberOfRowsInSection:(NSInteger)section {
    return self.items.count;
}

- (UITableViewCell *)tableView:(UITableView *)tableView 
         cellForRowAtIndexPath:(NSIndexPath *)indexPath {
    UITableViewCell *cell = [tableView dequeueReusableCellWithIdentifier:@"Cell" 
                                                            forIndexPath:indexPath];
    cell.textLabel.text = self.items[indexPath.row];
    cell.accessoryType = UITableViewCellAccessoryDisclosureIndicator;
    return cell;
}

- (void)tableView:(UITableView *)tableView 
commitEditingStyle:(UITableViewCellEditingStyle)editingStyle 
forRowAtIndexPath:(NSIndexPath *)indexPath {
    if (editingStyle == UITableViewCellEditingStyleDelete) {
        [self confirmDeleteItemAtIndexPath:indexPath];
    }
}

- (void)confirmDeleteItemAtIndexPath:(NSIndexPath *)indexPath {
    NSString *itemName = self.items[indexPath.row];
    
    UIAlertController *alert = [UIAlertController 
        alertControllerWithTitle:@"ยืนยันการลบ"
        message:[NSString stringWithFormat:@"ต้องการลบ '%@' หรือไม่?", itemName]
        preferredStyle:UIAlertControllerStyleAlert];
    
    UIAlertAction *deleteAction = [UIAlertAction actionWithTitle:@"ลบ"
                                                           style:UIAlertActionStyleDestructive
                                                         handler:^(UIAlertAction *action) {
        [self.items removeObjectAtIndex:indexPath.row];
        [self.tableView deleteRowsAtIndexPaths:@[indexPath] 
                             withRowAnimation:UITableViewRowAnimationFade];
    }];
    
    UIAlertAction *cancelAction = [UIAlertAction actionWithTitle:@"ยกเลิก"
                                                           style:UIAlertActionStyleCancel
                                                         handler:^(UIAlertAction *action) {
        // ยกเลิก -> เอา cell กลับ
        [self.tableView reloadRowsAtIndexPaths:@[indexPath] 
                              withRowAnimation:UITableViewRowAnimationAutomatic];
    }];
    
    [alert addAction:deleteAction];
    [alert addAction:cancelAction];
    alert.preferredAction = cancelAction;
    
    [self presentViewController:alert animated:YES completion:nil];
}

#pragma mark - UITableViewDelegate

- (void)tableView:(UITableView *)tableView didSelectRowAtIndexPath:(NSIndexPath *)indexPath {
    [tableView deselectRowAtIndexPath:indexPath animated:YES];
    
    NSString *item = self.items[indexPath.row];
    UITableViewCell *cell = [tableView cellForRowAtIndexPath:indexPath];
    
    [self showOptionsForItem:item fromCell:cell];
}

- (void)showOptionsForItem:(NSString *)item fromCell:(UITableViewCell *)cell {
    UIAlertController *sheet = [UIAlertController 
        alertControllerWithTitle:item
        message:@"เลือกการดำเนินการ"
        preferredStyle:UIAlertControllerStyleActionSheet];
    
    [sheet addAction:[UIAlertAction actionWithTitle:@"ดูรายละเอียด"
                                              style:UIAlertActionStyleDefault
                                            handler:^(UIAlertAction *action) {
        NSLog(@"ดูรายละเอียด: %@", item);
    }]];
    
    [sheet addAction:[UIAlertAction actionWithTitle:@"แก้ไข"
                                              style:UIAlertActionStyleDefault
                                            handler:^(UIAlertAction *action) {
        [self editItem:item];
    }]];
    
    [sheet addAction:[UIAlertAction actionWithTitle:@"แชร์"
                                              style:UIAlertActionStyleDefault
                                            handler:^(UIAlertAction *action) {
        [self shareItem:item fromView:cell];
    }]];
    
    [sheet addAction:[UIAlertAction actionWithTitle:@"ยกเลิก"
                                              style:UIAlertActionStyleCancel
                                            handler:nil]];
    
    // iPad support
    if (sheet.popoverPresentationController) {
        sheet.popoverPresentationController.sourceView = cell;
        sheet.popoverPresentationController.sourceRect = cell.bounds;
    }
    
    [self presentViewController:sheet animated:YES completion:nil];
}

- (void)editItem:(NSString *)item {
    UIAlertController *alert = [UIAlertController 
        alertControllerWithTitle:@"แก้ไขรายการ"
        message:nil
        preferredStyle:UIAlertControllerStyleAlert];
    
    [alert addTextFieldWithConfigurationHandler:^(UITextField *textField) {
        textField.text = item;
        textField.clearButtonMode = UITextFieldViewModeWhileEditing;
    }];
    
    UIAlertAction *saveAction = [UIAlertAction actionWithTitle:@"บันทึก"
                                                         style:UIAlertActionStyleDefault
                                                       handler:^(UIAlertAction *action) {
        NSString *newName = alert.textFields.firstObject.text;
        NSInteger index = [self.items indexOfObject:item];
        if (index != NSNotFound && newName.length > 0) {
            self.items[index] = newName;
            [self.tableView reloadData];
        }
    }];
    
    [alert addAction:saveAction];
    [alert addAction:[UIAlertAction actionWithTitle:@"ยกเลิก"
                                              style:UIAlertActionStyleCancel
                                            handler:nil]];
    
    [self presentViewController:alert animated:YES completion:nil];
}

- (void)shareItem:(NSString *)item fromView:(UIView *)sourceView {
    UIActivityViewController *activityVC = [[UIActivityViewController alloc] 
        initWithActivityItems:@[item] 
        applicationActivities:nil];
    
    if (activityVC.popoverPresentationController) {
        activityVC.popoverPresentationController.sourceView = sourceView;
        activityVC.popoverPresentationController.sourceRect = sourceView.bounds;
    }
    
    [self presentViewController:activityVC animated:YES completion:nil];
}

@end
```

---

## 50.11 Loading และ Progress Alerts

```objc
// Loading Alert (ไม่มีปุ่ม)
- (UIAlertController *)showLoadingAlert {
    UIAlertController *loadingAlert = [UIAlertController 
        alertControllerWithTitle:nil
        message:@"กำลังโหลด..."
        preferredStyle:UIAlertControllerStyleAlert];
    
    // เพิ่ม activity indicator
    UIActivityIndicatorView *indicator = [[UIActivityIndicatorView alloc] 
        initWithActivityIndicatorStyle:UIActivityIndicatorViewStyleMedium];
    indicator.translatesAutoresizingMaskIntoConstraints = NO;
    [indicator startAnimating];
    [loadingAlert.view addSubview:indicator];
    
    [NSLayoutConstraint activateConstraints:@[
        [indicator.centerXAnchor constraintEqualToAnchor:loadingAlert.view.centerXAnchor],
        [indicator.bottomAnchor constraintEqualToAnchor:loadingAlert.view.bottomAnchor constant:-20],
    ]];
    
    [self presentViewController:loadingAlert animated:YES completion:nil];
    return loadingAlert;
}

// การใช้งาน
- (void)performLongTask {
    UIAlertController *loading = [self showLoadingAlert];
    
    dispatch_async(dispatch_get_global_queue(DISPATCH_QUEUE_PRIORITY_DEFAULT, 0), ^{
        // ทำงานใน background...
        [NSThread sleepForTimeInterval:2.0];
        
        dispatch_async(dispatch_get_main_queue(), ^{
            [loading dismissViewControllerAnimated:YES completion:^{
                [self showSuccessAlert];
            }];
        });
    });
}

- (void)showSuccessAlert {
    UIAlertController *success = [UIAlertController 
        alertControllerWithTitle:@"สำเร็จ!"
        message:@"ดำเนินการเสร็จสมบูรณ์"
        preferredStyle:UIAlertControllerStyleAlert];
    
    [success addAction:[UIAlertAction actionWithTitle:@"ตกลง"
                                                style:UIAlertActionStyleDefault
                                              handler:nil]];
    
    [self presentViewController:success animated:YES completion:nil];
}
```

---

## 50.12 แบบฝึกหัด (Practice Exercises)

### แบบฝึกหัดที่ 1: Settings Menu

สร้าง Settings menu ที่มี:
- Profile section (ชื่อ, อีเมล สามารถแก้ไขด้วย alert)
- Notifications section (toggle)
- Account section (logout ด้วย destructive alert)

```objc
@implementation SettingsViewController

- (void)logoutTapped {
    UIAlertController *alert = [UIAlertController 
        alertControllerWithTitle:@"ออกจากระบบ"
        message:@"คุณแน่ใจหรือไม่ที่จะออกจากระบบ? ข้อมูลที่ไม่ได้บันทึกจะสูญหาย"
        preferredStyle:UIAlertControllerStyleAlert];
    
    UIAlertAction *logoutAction = [UIAlertAction actionWithTitle:@"ออกจากระบบ"
                                                           style:UIAlertActionStyleDestructive
                                                         handler:^(UIAlertAction *action) {
        [self performLogout];
    }];
    
    [alert addAction:logoutAction];
    [alert addAction:[UIAlertAction actionWithTitle:@"ยกเลิก"
                                              style:UIAlertActionStyleCancel
                                            handler:nil]];
    
    [self presentViewController:alert animated:YES completion:nil];
}

- (void)editProfileTapped {
    UIAlertController *alert = [UIAlertController 
        alertControllerWithTitle:@"แก้ไขโปรไฟล์"
        message:nil
        preferredStyle:UIAlertControllerStyleAlert];
    
    [alert addTextFieldWithConfigurationHandler:^(UITextField *textField) {
        textField.placeholder = @"ชื่อ";
        textField.text = @"John Doe";
    }];
    
    [alert addTextFieldWithConfigurationHandler:^(UITextField *textField) {
        textField.placeholder = @"อีเมล";
        textField.keyboardType = UIKeyboardTypeEmailAddress;
        textField.text = @"john@example.com";
    }];
    
    UIAlertAction *saveAction = [UIAlertAction actionWithTitle:@"บันทึก"
                                                         style:UIAlertActionStyleDefault
                                                       handler:^(UIAlertAction *action) {
        NSString *name = alert.textFields[0].text;
        NSString *email = alert.textFields[1].text;
        NSLog(@"บันทึก: %@ <%@>", name, email);
    }];
    
    [alert addAction:saveAction];
    [alert addAction:[UIAlertAction actionWithTitle:@"ยกเลิก"
                                              style:UIAlertActionStyleCancel
                                            handler:nil]];
    
    [self presentViewController:alert animated:YES completion:nil];
}

@end
```

### แบบฝึกหัดที่ 2: Photo Upload Flow

สร้าง flow สำหรับอัปโหลดรูปที่:
1. แสดง action sheet ให้เลือก "ถ่ายรูป" หรือ "เลือกจากคลัง"
2. หลังเลือกรูปแล้ว แสดง loading alert
3. แสดง success/error alert หลังอัปโหลด

```objc
- (void)startPhotoUpload {
    UIAlertController *sheet = [UIAlertController 
        alertControllerWithTitle:@"อัปโหลดรูปโปรไฟล์"
        message:@"เลือกแหล่งที่มาของรูปภาพ"
        preferredStyle:UIAlertControllerStyleActionSheet];
    
    if ([UIImagePickerController isSourceTypeAvailable:UIImagePickerControllerSourceTypeCamera]) {
        [sheet addAction:[UIAlertAction actionWithTitle:@"ถ่ายรูป"
                                                  style:UIAlertActionStyleDefault
                                                handler:^(UIAlertAction *action) {
            [self presentImagePickerWithSource:UIImagePickerControllerSourceTypeCamera];
        }]];
    }
    
    [sheet addAction:[UIAlertAction actionWithTitle:@"เลือกจากคลัง"
                                              style:UIAlertActionStyleDefault
                                            handler:^(UIAlertAction *action) {
        [self presentImagePickerWithSource:UIImagePickerControllerSourceTypePhotoLibrary];
    }]];
    
    [sheet addAction:[UIAlertAction actionWithTitle:@"ลบรูปปัจจุบัน"
                                              style:UIAlertActionStyleDestructive
                                            handler:^(UIAlertAction *action) {
        [self removeProfilePhoto];
    }]];
    
    [sheet addAction:[UIAlertAction actionWithTitle:@"ยกเลิก"
                                              style:UIAlertActionStyleCancel
                                            handler:nil]];
    
    if (sheet.popoverPresentationController) {
        sheet.popoverPresentationController.sourceView = self.profileImageView;
        sheet.popoverPresentationController.sourceRect = self.profileImageView.bounds;
    }
    
    [self presentViewController:sheet animated:YES completion:nil];
}
```

### แบบฝึกหัดที่ 3: Rate App Alert

สร้าง rate app alert ที่ปรากฏหลังใช้งานครบ 5 ครั้ง

```objc
- (void)checkAndShowRatingPrompt {
    NSInteger useCount = [[NSUserDefaults standardUserDefaults] integerForKey:@"appUseCount"];
    useCount++;
    [[NSUserDefaults standardUserDefaults] setInteger:useCount forKey:@"appUseCount"];
    
    BOOL alreadyRated = [[NSUserDefaults standardUserDefaults] boolForKey:@"hasRated"];
    
    if (useCount == 5 && !alreadyRated) {
        [self showRatingAlert];
    }
}

- (void)showRatingAlert {
    UIAlertController *alert = [UIAlertController 
        alertControllerWithTitle:@"ชอบแอปนี้ไหม?"
        message:@"คุณใช้งานแอปนี้มาสักพักแล้ว ช่วยให้คะแนนเราหน่อยได้ไหม?"
        preferredStyle:UIAlertControllerStyleAlert];
    
    [alert addAction:[UIAlertAction actionWithTitle:@"ให้คะแนน ⭐"
                                              style:UIAlertActionStyleDefault
                                            handler:^(UIAlertAction *action) {
        [[NSUserDefaults standardUserDefaults] setBool:YES forKey:@"hasRated"];
        // ไปที่ App Store
        NSURL *url = [NSURL URLWithString:@"https://apps.apple.com/app/idXXXXXXXXX?action=write-review"];
        [[UIApplication sharedApplication] openURL:url options:@{} completionHandler:nil];
    }]];
    
    [alert addAction:[UIAlertAction actionWithTitle:@"ถามทีหลัง"
                                              style:UIAlertActionStyleDefault
                                            handler:^(UIAlertAction *action) {
        // Reset count เพื่อถามอีก 5 ครั้งต่อมา
        [[NSUserDefaults standardUserDefaults] setInteger:0 forKey:@"appUseCount"];
    }]];
    
    [alert addAction:[UIAlertAction actionWithTitle:@"ไม่ขอบคุณ"
                                              style:UIAlertActionStyleCancel
                                            handler:^(UIAlertAction *action) {
        [[NSUserDefaults standardUserDefaults] setBool:YES forKey:@"hasRated"];
    }]];
    
    [self presentViewController:alert animated:YES completion:nil];
}
```

---

## สรุป (Summary)

- **UIAlertController** มีสอง style: Alert (กลางหน้าจอ) และ Action Sheet (ด้านล่าง)
- **UIAlertActionStyleDestructive** ใช้กับ action ที่ไม่สามารถย้อนกลับได้ (สีแดง)
- **UIAlertActionStyleCancel** แสดงเป็น bold และมักอยู่ล่างสุด
- เพิ่ม **text fields** ใน alert ด้วย `addTextFieldWithConfigurationHandler:`
- **UIPopoverPresentationController** แสดง content แบบ popup bubble ชี้ไปที่ view
- บน iPad ต้องกำหนด `sourceView`/`sourceRect` หรือ `barButtonItem` สำหรับ action sheet
- **UIActivityViewController** ใช้แชร์เนื้อหาผ่าน share sheet
- **UIDocumentPickerViewController** ใช้เปิดหรือบันทึกไฟล์
- **UIImagePickerController** ใช้ถ่ายรูปหรือเลือกรูป (ต้องขอ permission)
- **PHPickerViewController** (iOS 14+) เป็นทางเลือกที่ดีกว่าสำหรับ photo selection
