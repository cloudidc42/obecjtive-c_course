# ตอนที่ 68: Contacts และ Calendar ใน Objective-C

## บทนำ

ในตอนนี้เราจะเรียนรู้การทำงานกับรายชื่อผู้ติดต่อผ่าน Contacts framework และการจัดการนัดหมายผ่าน EventKit framework ซึ่งเป็น API ที่ Apple จัดเตรียมไว้สำหรับการเข้าถึงข้อมูลส่วนตัวเหล่านี้อย่างปลอดภัย

---

## 68.1 Contacts Framework

Contacts framework (แทนที่ AddressBook framework เก่า) เป็น API สมัยใหม่สำหรับจัดการรายชื่อผู้ติดต่อ

### การเพิ่ม Contacts Framework

```objc
#import <Contacts/Contacts.h>
```

### Info.plist

```xml
<key>NSContactsUsageDescription</key>
<string>แอปต้องการเข้าถึงรายชื่อผู้ติดต่อเพื่อแชร์ข้อมูล</string>
```

### การตรวจสอบและขอสิทธิ์

```objc
- (void)requestContactsPermission {
    CNAuthorizationStatus status = [CNContactStore 
        authorizationStatusForEntityType:CNEntityTypeContacts];
    
    switch (status) {
        case CNAuthorizationStatusNotDetermined: {
            CNContactStore *store = [[CNContactStore alloc] init];
            [store requestAccessForEntityType:CNEntityTypeContacts 
                            completionHandler:^(BOOL granted, NSError *error) {
                dispatch_async(dispatch_get_main_queue(), ^{
                    if (granted) {
                        NSLog(@"ได้รับสิทธิ์ Contacts");
                        [self loadContacts];
                    } else {
                        NSLog(@"ไม่ได้รับสิทธิ์: %@", error);
                    }
                });
            }];
            break;
        }
        
        case CNAuthorizationStatusAuthorized:
            NSLog(@"ได้รับสิทธิ์แล้ว");
            [self loadContacts];
            break;
            
        case CNAuthorizationStatusDenied:
            NSLog(@"ผู้ใช้ปฏิเสธสิทธิ์");
            [self showPermissionAlert];
            break;
            
        case CNAuthorizationStatusRestricted:
            NSLog(@"การเข้าถึงถูกจำกัดโดยนโยบายองค์กร");
            break;
            
        case CNAuthorizationStatusLimited:
            NSLog(@"ได้รับสิทธิ์แบบจำกัด (iOS 18+)");
            [self loadContacts];
            break;
    }
}

- (void)showPermissionAlert {
    UIAlertController *alert = [UIAlertController 
        alertControllerWithTitle:@"ต้องการสิทธิ์รายชื่อ"
        message:@"กรุณาเปิดสิทธิ์ใน Settings > Privacy > Contacts"
        preferredStyle:UIAlertControllerStyleAlert];
    
    [alert addAction:[UIAlertAction actionWithTitle:@"ไปที่ Settings" 
                                             style:UIAlertActionStyleDefault 
                                           handler:^(UIAlertAction *action) {
        [[UIApplication sharedApplication] 
            openURL:[NSURL URLWithString:UIApplicationOpenSettingsURLString] 
            options:@{} completionHandler:nil];
    }]];
    
    [alert addAction:[UIAlertAction actionWithTitle:@"ยกเลิก" 
                                             style:UIAlertActionStyleCancel 
                                           handler:nil]];
    
    [self presentViewController:alert animated:YES completion:nil];
}
```

---

## 68.2 Fetching Contacts

### การดึงรายชื่อทั้งหมด

```objc
- (void)loadContacts {
    CNContactStore *store = [[CNContactStore alloc] init];
    
    // กำหนด keys ที่ต้องการดึง
    NSArray *keysToFetch = @[
        CNContactGivenNameKey,
        CNContactFamilyNameKey,
        CNContactMiddleNameKey,
        CNContactNicknameKey,
        CNContactPhoneNumbersKey,
        CNContactEmailAddressesKey,
        CNContactPostalAddressesKey,
        CNContactBirthdayKey,
        CNContactImageDataKey,
        CNContactImageDataAvailableKey,
        CNContactOrganizationNameKey,
        CNContactJobTitleKey,
        CNContactDepartmentNameKey,
        CNContactNoteKey,
        CNContactURLAddressesKey,
        CNContactSocialProfilesKey,
    ];
    
    // สร้าง fetch request
    CNContactFetchRequest *fetchRequest = [[CNContactFetchRequest alloc] 
        initWithKeysToFetch:keysToFetch];
    
    // เรียงลำดับ
    fetchRequest.sortOrder = CNContactSortOrderGivenName;
    
    // Enumerate contacts
    NSMutableArray<CNContact *> *contacts = [NSMutableArray array];
    NSError *error;
    
    BOOL success = [store enumerateContactsWithFetchRequest:fetchRequest 
                                                      error:&error 
                                                 usingBlock:^(CNContact *contact, 
                                                              BOOL *stop) {
        [contacts addObject:contact];
    }];
    
    if (!success) {
        NSLog(@"Fetch error: %@", error);
        return;
    }
    
    NSLog(@"โหลดรายชื่อสำเร็จ: %lu คน", (unsigned long)contacts.count);
    
    dispatch_async(dispatch_get_main_queue(), ^{
        self.contacts = contacts;
        [self.tableView reloadData];
    });
}
```

### การดึงรายชื่อด้วย Predicate

```objc
- (void)fetchContactsMatchingName:(NSString *)name {
    CNContactStore *store = [[CNContactStore alloc] init];
    
    NSArray *keys = @[
        CNContactGivenNameKey,
        CNContactFamilyNameKey,
        CNContactPhoneNumbersKey,
        CNContactEmailAddressesKey
    ];
    
    // ค้นหาตามชื่อ
    NSPredicate *predicate = [CNContact predicateForContactsMatchingName:name];
    
    NSError *error;
    NSArray<CNContact *> *contacts = [store unifiedContactsMatchingPredicate:predicate 
                                                                keysToFetch:keys 
                                                                      error:&error];
    
    if (error) {
        NSLog(@"Fetch error: %@", error);
        return;
    }
    
    NSLog(@"พบ %lu รายชื่อที่ตรงกับ '%@'", (unsigned long)contacts.count, name);
    
    for (CNContact *contact in contacts) {
        [self logContactInfo:contact];
    }
}

// ค้นหาด้วยเบอร์โทร
- (void)fetchContactWithPhoneNumber:(NSString *)phoneNumber {
    CNContactStore *store = [[CNContactStore alloc] init];
    
    NSArray *keys = @[
        CNContactGivenNameKey,
        CNContactFamilyNameKey,
        CNContactPhoneNumbersKey
    ];
    
    CNPhoneNumber *phoneNum = [CNPhoneNumber phoneNumberWithStringValue:phoneNumber];
    NSPredicate *predicate = [CNContact predicateForContactsMatchingPhoneNumber:phoneNum];
    
    NSError *error;
    NSArray<CNContact *> *contacts = [store unifiedContactsMatchingPredicate:predicate 
                                                                keysToFetch:keys 
                                                                      error:&error];
    
    for (CNContact *contact in contacts) {
        NSLog(@"พบ: %@ %@", contact.givenName, contact.familyName);
    }
}

// ดึง contact ด้วย identifier
- (CNContact *)fetchContactWithIdentifier:(NSString *)identifier {
    CNContactStore *store = [[CNContactStore alloc] init];
    
    NSArray *keys = @[CNContactGivenNameKey, CNContactFamilyNameKey, 
                      CNContactPhoneNumbersKey];
    
    NSError *error;
    CNContact *contact = [store unifiedContactWithIdentifier:identifier 
                                               keysToFetch:keys 
                                                     error:&error];
    
    return contact;
}
```

### การแสดงข้อมูล Contact

```objc
- (void)logContactInfo:(CNContact *)contact {
    NSLog(@"=== รายชื่อ ===");
    
    // ชื่อ
    NSString *fullName = [CNContactFormatter stringFromContact:contact 
                                                         style:CNContactFormatterStyleFullName];
    NSLog(@"ชื่อเต็ม: %@", fullName ?: @"ไม่มีชื่อ");
    NSLog(@"ชื่อต้น: %@", contact.givenName);
    NSLog(@"นามสกุล: %@", contact.familyName);
    NSLog(@"ชื่อกลาง: %@", contact.middleName);
    NSLog(@"ชื่อเล่น: %@", contact.nickname);
    
    // องค์กร
    NSLog(@"บริษัท: %@", contact.organizationName);
    NSLog(@"ตำแหน่ง: %@", contact.jobTitle);
    NSLog(@"แผนก: %@", contact.departmentName);
    
    // เบอร์โทร
    NSLog(@"เบอร์โทร (%lu):", (unsigned long)contact.phoneNumbers.count);
    for (CNLabeledValue<CNPhoneNumber *> *phone in contact.phoneNumbers) {
        NSString *label = [CNLabeledValue localizedStringForLabel:phone.label];
        NSLog(@"  [%@]: %@", label, phone.value.stringValue);
    }
    
    // Email
    NSLog(@"Email (%lu):", (unsigned long)contact.emailAddresses.count);
    for (CNLabeledValue<NSString *> *email in contact.emailAddresses) {
        NSString *label = [CNLabeledValue localizedStringForLabel:email.label];
        NSLog(@"  [%@]: %@", label, email.value);
    }
    
    // ที่อยู่
    for (CNLabeledValue<CNPostalAddress *> *addr in contact.postalAddresses) {
        NSString *label = [CNLabeledValue localizedStringForLabel:addr.label];
        NSString *formattedAddress = [CNPostalAddressFormatter 
            stringFromPostalAddress:addr.value 
            style:CNPostalAddressFormatterStyleMailingAddress];
        NSLog(@"  [%@]: %@", label, formattedAddress);
    }
    
    // วันเกิด
    if (contact.birthday) {
        NSDateComponents *bday = contact.birthday;
        NSLog(@"วันเกิด: %ld/%ld/%ld", 
              (long)bday.day, (long)bday.month, (long)bday.year);
    }
    
    // หมายเหตุ
    if (contact.note.length > 0) {
        NSLog(@"หมายเหตุ: %@", contact.note);
    }
}
```

---

## 68.3 Creating/Modifying Contacts

### การสร้าง Contact ใหม่

```objc
- (void)createNewContact {
    CNMutableContact *contact = [[CNMutableContact alloc] init];
    
    // ชื่อ
    contact.givenName = @"สมชาย";
    contact.familyName = @"ใจดี";
    contact.nickname = @"เบิ้ม";
    
    // องค์กร
    contact.organizationName = @"บริษัท ไทยเทค จำกัด";
    contact.jobTitle = @"Software Engineer";
    
    // เบอร์โทร
    contact.phoneNumbers = @[
        [CNLabeledValue labeledValueWithLabel:CNLabelPhoneNumberMobile 
                                        value:[CNPhoneNumber phoneNumberWithStringValue:@"0812345678"]],
        [CNLabeledValue labeledValueWithLabel:CNLabelPhoneNumberMain 
                                        value:[CNPhoneNumber phoneNumberWithStringValue:@"021234567"]]
    ];
    
    // Email
    contact.emailAddresses = @[
        [CNLabeledValue labeledValueWithLabel:CNLabelWork 
                                        value:@"somchai@thaitech.com"],
        [CNLabeledValue labeledValueWithLabel:CNLabelHome 
                                        value:@"somchai.personal@gmail.com"]
    ];
    
    // ที่อยู่
    CNMutablePostalAddress *address = [[CNMutablePostalAddress alloc] init];
    address.street = @"123 ถนนสุขุมวิท";
    address.subLocality = @"คลองเตย";
    address.city = @"กรุงเทพมหานคร";
    address.postalCode = @"10110";
    address.country = @"Thailand";
    address.ISOCountryCode = @"TH";
    
    contact.postalAddresses = @[
        [CNLabeledValue labeledValueWithLabel:CNLabelWork value:address]
    ];
    
    // วันเกิด
    NSDateComponents *birthday = [[NSDateComponents alloc] init];
    birthday.year = 1990;
    birthday.month = 5;
    birthday.day = 15;
    contact.birthday = birthday;
    
    // URL
    contact.urlAddresses = @[
        [CNLabeledValue labeledValueWithLabel:CNLabelURLAddressHomePage 
                                        value:@"https://somchai.dev"]
    ];
    
    // รูปภาพ
    UIImage *photo = [UIImage imageNamed:@"profile_photo"];
    if (photo) {
        contact.imageData = UIImageJPEGRepresentation(photo, 0.8);
    }
    
    // หมายเหตุ
    contact.note = @"เพื่อนร่วมงาน พบกันที่งาน iOS Dev Meetup";
    
    // บันทึก
    [self saveContact:contact];
}

- (void)saveContact:(CNMutableContact *)contact {
    CNContactStore *store = [[CNContactStore alloc] init];
    CNSaveRequest *saveRequest = [[CNSaveRequest alloc] init];
    
    [saveRequest addContact:contact toContainerWithIdentifier:nil];
    
    NSError *error;
    BOOL success = [store executeSaveRequest:saveRequest error:&error];
    
    if (success) {
        NSLog(@"บันทึก contact สำเร็จ: %@", contact.identifier);
    } else {
        NSLog(@"บันทึก contact ล้มเหลว: %@", error.localizedDescription);
    }
}
```

### การแก้ไข Contact

```objc
- (void)updateContact:(CNContact *)contact 
         withNewPhone:(NSString *)newPhone {
    
    // ต้องดึง contact ที่ mutable ได้ก่อน
    CNContactStore *store = [[CNContactStore alloc] init];
    
    // ดึง contact พร้อม keys ที่จะแก้ไข
    NSArray *keys = @[CNContactPhoneNumbersKey, 
                      CNContactGivenNameKey,
                      CNContactFamilyNameKey];
    
    NSError *fetchError;
    CNContact *fetchedContact = [store unifiedContactWithIdentifier:contact.identifier 
                                                      keysToFetch:keys 
                                                            error:&fetchError];
    
    if (fetchError) {
        NSLog(@"Fetch error: %@", fetchError);
        return;
    }
    
    // สร้าง mutable copy
    CNMutableContact *mutableContact = [fetchedContact mutableCopy];
    
    // เพิ่มเบอร์โทรใหม่
    NSMutableArray *phoneNumbers = [mutableContact.phoneNumbers mutableCopy];
    [phoneNumbers addObject:[CNLabeledValue 
        labeledValueWithLabel:CNLabelPhoneNumberMobile 
                        value:[CNPhoneNumber phoneNumberWithStringValue:newPhone]]];
    
    mutableContact.phoneNumbers = phoneNumbers;
    
    // บันทึกการแก้ไข
    CNSaveRequest *saveRequest = [[CNSaveRequest alloc] init];
    [saveRequest updateContact:mutableContact];
    
    NSError *saveError;
    if ([store executeSaveRequest:saveRequest error:&saveError]) {
        NSLog(@"อัปเดต contact สำเร็จ");
    } else {
        NSLog(@"อัปเดต contact ล้มเหลว: %@", saveError);
    }
}

// ลบ Contact
- (void)deleteContact:(CNContact *)contact {
    CNContactStore *store = [[CNContactStore alloc] init];
    
    // ดึง mutable version
    NSArray *keys = @[[CNContactIdentifierKey]];
    NSError *fetchError;
    CNContact *fetchedContact = [store unifiedContactWithIdentifier:contact.identifier 
                                                      keysToFetch:keys 
                                                            error:&fetchError];
    
    CNMutableContact *mutableContact = [fetchedContact mutableCopy];
    
    CNSaveRequest *saveRequest = [[CNSaveRequest alloc] init];
    [saveRequest deleteContact:mutableContact];
    
    NSError *deleteError;
    if ([store executeSaveRequest:saveRequest error:&deleteError]) {
        NSLog(@"ลบ contact สำเร็จ");
    } else {
        NSLog(@"ลบ contact ล้มเหลว: %@", deleteError);
    }
}
```

---

## 68.4 Contact Picker (CNContactPickerViewController)

CNContactPickerViewController ให้ผู้ใช้เลือก contact โดยไม่ต้องขอสิทธิ์

```objc
// ContactsViewController.h
#import <UIKit/UIKit.h>
#import <Contacts/Contacts.h>
#import <ContactsUI/ContactsUI.h>

@interface ContactsViewController : UIViewController 
    <CNContactPickerDelegate, CNContactViewControllerDelegate>

@end
```

```objc
// ContactsViewController.m
#import "ContactsViewController.h"

@implementation ContactsViewController

// แสดง Contact Picker
- (IBAction)showContactPickerTapped:(UIButton *)sender {
    CNContactPickerViewController *picker = [[CNContactPickerViewController alloc] init];
    picker.delegate = self;
    
    // ระบุ properties ที่ต้องการแสดง (optional)
    picker.displayedPropertyKeys = @[
        CNContactGivenNameKey,
        CNContactFamilyNameKey,
        CNContactPhoneNumbersKey,
        CNContactEmailAddressesKey
    ];
    
    // ถ้าต้องการให้ผู้ใช้เลือก property เฉพาะ (เช่น เฉพาะเบอร์โทร)
    picker.predicateForSelectionOfProperty = [NSPredicate predicateWithBlock:
        ^BOOL(CNContactProperty *property, NSDictionary *bindings) {
        return [property.key isEqualToString:CNContactPhoneNumbersKey];
    }];
    
    [self presentViewController:picker animated:YES completion:nil];
}

#pragma mark - CNContactPickerDelegate

// เมื่อผู้ใช้เลือก contact
- (void)contactPicker:(CNContactPickerViewController *)picker 
      didSelectContact:(CNContact *)contact {
    NSLog(@"เลือก contact: %@ %@", contact.givenName, contact.familyName);
    
    // แสดงชื่อเต็ม
    NSString *name = [CNContactFormatter stringFromContact:contact 
                                                     style:CNContactFormatterStyleFullName];
    NSLog(@"ชื่อ: %@", name);
    
    // แสดงเบอร์แรก
    if (contact.phoneNumbers.count > 0) {
        CNPhoneNumber *phone = contact.phoneNumbers.firstObject.value;
        NSLog(@"เบอร์: %@", phone.stringValue);
    }
}

// เมื่อผู้ใช้เลือก property เฉพาะ
- (void)contactPicker:(CNContactPickerViewController *)picker 
    didSelectContactProperty:(CNContactProperty *)contactProperty {
    
    CNContact *contact = contactProperty.contact;
    NSLog(@"เลือก property '%@' ของ %@", 
          contactProperty.key, contact.givenName);
    
    if ([contactProperty.key isEqualToString:CNContactPhoneNumbersKey]) {
        CNPhoneNumber *phone = (CNPhoneNumber *)contactProperty.value;
        NSLog(@"เบอร์ที่เลือก: %@", phone.stringValue);
        [self dialPhoneNumber:phone.stringValue];
    }
    
    if ([contactProperty.key isEqualToString:CNContactEmailAddressesKey]) {
        NSString *email = (NSString *)contactProperty.value;
        NSLog(@"Email ที่เลือก: %@", email);
    }
}

// เมื่อผู้ใช้เลือกหลาย contacts
- (void)contactPicker:(CNContactPickerViewController *)picker 
    didSelectContacts:(NSArray<CNContact *> *)contacts {
    NSLog(@"เลือก %lu contacts", (unsigned long)contacts.count);
    
    for (CNContact *contact in contacts) {
        NSLog(@"  - %@ %@", contact.givenName, contact.familyName);
    }
}

// เมื่อผู้ใช้ปิด picker
- (void)contactPickerDidCancel:(CNContactPickerViewController *)picker {
    NSLog(@"ผู้ใช้ยกเลิก contact picker");
}

- (void)dialPhoneNumber:(NSString *)phoneNumber {
    NSString *cleanNumber = [[phoneNumber componentsSeparatedByCharactersInSet:
        [[NSCharacterSet decimalDigitCharacterSet] invertedSet]] 
        componentsJoinedByString:@""];
    
    NSURL *url = [NSURL URLWithString:[NSString stringWithFormat:@"tel://%@", cleanNumber]];
    
    if ([[UIApplication sharedApplication] canOpenURL:url]) {
        [[UIApplication sharedApplication] openURL:url options:@{} completionHandler:nil];
    }
}
```

### CNContactViewController (แสดง/แก้ไข Contact)

```objc
// แสดง Contact Detail
- (void)showContactDetail:(CNContact *)contact {
    // ต้องดึง contact พร้อม all keys
    CNContactStore *store = [[CNContactStore alloc] init];
    NSArray *keys = [CNContactViewController descriptorForRequiredKeys];
    
    NSError *error;
    CNContact *fullContact = [store unifiedContactWithIdentifier:contact.identifier 
                                                    keysToFetch:@[keys] 
                                                          error:&error];
    
    if (error || !fullContact) return;
    
    CNContactViewController *vc = [CNContactViewController 
        viewControllerForContact:fullContact];
    vc.delegate = self;
    vc.allowsEditing = YES;
    vc.allowsActions = YES;
    
    [self.navigationController pushViewController:vc animated:YES];
}

// สร้าง Contact ใหม่ผ่าน UI
- (void)showNewContactUI {
    CNMutableContact *contact = [[CNMutableContact alloc] init];
    
    CNContactViewController *vc = [CNContactViewController 
        viewControllerForNewContact:contact];
    vc.delegate = self;
    
    UINavigationController *nav = [[UINavigationController alloc] 
        initWithRootViewController:vc];
    [self presentViewController:nav animated:YES completion:nil];
}

// CNContactViewControllerDelegate
- (void)contactViewController:(CNContactViewController *)viewController 
    didCompleteWithContact:(CNContact *)contact {
    [viewController dismissViewControllerAnimated:YES completion:nil];
    
    if (contact) {
        NSLog(@"บันทึก contact: %@ %@", contact.givenName, contact.familyName);
    } else {
        NSLog(@"ยกเลิกการสร้าง contact");
    }
}

@end
```

---

## 68.5 Contact Groups

```objc
- (void)manageContactGroups {
    CNContactStore *store = [[CNContactStore alloc] init];
    
    // ดึง groups ทั้งหมด
    NSError *error;
    NSArray<CNGroup *> *groups = [store groupsMatchingPredicate:nil error:&error];
    
    NSLog(@"พบ %lu groups:", (unsigned long)groups.count);
    for (CNGroup *group in groups) {
        NSLog(@"  - %@ (id: %@)", group.name, group.identifier);
    }
    
    // สร้าง group ใหม่
    CNMutableGroup *newGroup = [[CNMutableGroup alloc] init];
    newGroup.name = @"เพื่อนร่วมงาน";
    
    CNSaveRequest *saveRequest = [[CNSaveRequest alloc] init];
    [saveRequest addGroup:newGroup toContainerWithIdentifier:nil];
    
    if ([store executeSaveRequest:saveRequest error:&error]) {
        NSLog(@"สร้าง group สำเร็จ: %@", newGroup.identifier);
        
        // เพิ่ม contact เข้า group
        // [self addContact:someContact toGroup:newGroup store:store];
    }
}

- (void)addContact:(CNContact *)contact 
           toGroup:(CNGroup *)group 
             store:(CNContactStore *)store {
    
    CNSaveRequest *saveRequest = [[CNSaveRequest alloc] init];
    [saveRequest addMember:contact toGroup:group];
    
    NSError *error;
    if ([store executeSaveRequest:saveRequest error:&error]) {
        NSLog(@"เพิ่ม %@ เข้า group %@ สำเร็จ", 
              contact.givenName, group.name);
    }
}
```

---

## 68.6 EventKit Framework

EventKit ใช้สำหรับจัดการ Events และ Reminders

### การเพิ่ม EventKit

```objc
#import <EventKit/EventKit.h>
```

### Info.plist

```xml
<key>NSCalendarsUsageDescription</key>
<string>แอปต้องการเข้าถึงปฏิทินเพื่อเพิ่มนัดหมาย</string>

<key>NSCalendarsWriteOnlyAccessUsageDescription</key>  
<string>แอปต้องการบันทึกนัดหมายลงปฏิทิน</string>

<key>NSRemindersUsageDescription</key>
<string>แอปต้องการเข้าถึงการแจ้งเตือนของคุณ</string>
```

### การขอสิทธิ์

```objc
// EventKitManager.h
@interface EventKitManager : NSObject

@property (nonatomic, strong) EKEventStore *eventStore;

+ (instancetype)sharedManager;
- (void)requestCalendarAccess:(void(^)(BOOL granted))completion;
- (void)requestRemindersAccess:(void(^)(BOOL granted))completion;

@end
```

```objc
// EventKitManager.m
@implementation EventKitManager

+ (instancetype)sharedManager {
    static EventKitManager *instance;
    static dispatch_once_t onceToken;
    dispatch_once(&onceToken, ^{
        instance = [[EventKitManager alloc] init];
    });
    return instance;
}

- (instancetype)init {
    self = [super init];
    if (self) {
        _eventStore = [[EKEventStore alloc] init];
    }
    return self;
}

- (void)requestCalendarAccess:(void(^)(BOOL granted))completion {
    EKAuthorizationStatus status = [EKEventStore 
        authorizationStatusForEntityType:EKEntityTypeEvent];
    
    switch (status) {
        case EKAuthorizationStatusNotDetermined:
            // iOS 17+ ต้องระบุ entity type
            if (@available(iOS 17.0, *)) {
                [self.eventStore requestFullAccessToEventsWithCompletion:
                    ^(BOOL granted, NSError *error) {
                    dispatch_async(dispatch_get_main_queue(), ^{
                        if (completion) completion(granted);
                    });
                }];
            } else {
                [self.eventStore requestAccessToEntityType:EKEntityTypeEvent 
                                        completion:^(BOOL granted, NSError *error) {
                    dispatch_async(dispatch_get_main_queue(), ^{
                        if (completion) completion(granted);
                    });
                }];
            }
            break;
            
        case EKAuthorizationStatusAuthorized:
        case EKAuthorizationStatusWriteOnly: // iOS 17+
            if (completion) completion(YES);
            break;
            
        case EKAuthorizationStatusDenied:
        case EKAuthorizationStatusRestricted:
            if (completion) completion(NO);
            break;
    }
}

- (void)requestRemindersAccess:(void(^)(BOOL granted))completion {
    if (@available(iOS 17.0, *)) {
        [self.eventStore requestFullAccessToRemindersWithCompletion:
            ^(BOOL granted, NSError *error) {
            dispatch_async(dispatch_get_main_queue(), ^{
                if (completion) completion(granted);
            });
        }];
    } else {
        [self.eventStore requestAccessToEntityType:EKEntityTypeReminder 
                                completion:^(BOOL granted, NSError *error) {
            dispatch_async(dispatch_get_main_queue(), ^{
                if (completion) completion(granted);
            });
        }];
    }
}

@end
```

---

## 68.7 Fetching Events

### ดึง Events จากปฏิทิน

```objc
- (void)fetchEventsForMonth:(NSDate *)date {
    EventKitManager *manager = [EventKitManager sharedManager];
    
    [manager requestCalendarAccess:^(BOOL granted) {
        if (!granted) {
            NSLog(@"ไม่ได้รับสิทธิ์ปฏิทิน");
            return;
        }
        
        EKEventStore *store = manager.eventStore;
        
        // กำหนดช่วงเวลา
        NSCalendar *calendar = [NSCalendar currentCalendar];
        NSDateComponents *startComponents = [calendar components:
            NSCalendarUnitYear | NSCalendarUnitMonth 
            fromDate:date];
        startComponents.day = 1;
        NSDate *startDate = [calendar dateFromComponents:startComponents];
        
        NSDateComponents *monthRange = [[NSDateComponents alloc] init];
        monthRange.month = 1;
        NSDate *endDate = [calendar dateByAddingComponents:monthRange 
                                                    toDate:startDate 
                                                   options:0];
        
        // ดึง calendars ทั้งหมด
        NSArray<EKCalendar *> *calendars = [store calendarsForEntityType:EKEntityTypeEvent];
        
        // สร้าง predicate
        NSPredicate *predicate = [store predicateForEventsWithStartDate:startDate 
                                                                endDate:endDate 
                                                              calendars:calendars];
        
        // ดึง events
        NSArray<EKEvent *> *events = [store eventsMatchingPredicate:predicate];
        
        // เรียงลำดับตามวันที่
        NSArray<EKEvent *> *sortedEvents = [events sortedArrayUsingComparator:
            ^NSComparisonResult(EKEvent *e1, EKEvent *e2) {
            return [e1.startDate compare:e2.startDate];
        }];
        
        NSLog(@"พบ %lu events ในเดือนนี้:", (unsigned long)sortedEvents.count);
        
        for (EKEvent *event in sortedEvents) {
            [self logEventInfo:event];
        }
    }];
}

- (void)logEventInfo:(EKEvent *)event {
    NSDateFormatter *formatter = [[NSDateFormatter alloc] init];
    formatter.dateFormat = @"dd/MM/yyyy HH:mm";
    
    NSLog(@"=== %@ ===", event.title);
    NSLog(@"เริ่ม: %@", [formatter stringFromDate:event.startDate]);
    
    if (!event.isAllDay) {
        NSLog(@"สิ้นสุด: %@", [formatter stringFromDate:event.endDate]);
    } else {
        NSLog(@"(ทั้งวัน)");
    }
    
    NSLog(@"ปฏิทิน: %@ (%@)", event.calendar.title, event.calendar.calendarIdentifier);
    
    if (event.location) {
        NSLog(@"สถานที่: %@", event.location);
    }
    
    if (event.notes) {
        NSLog(@"หมายเหตุ: %@", event.notes);
    }
    
    if (event.hasAlarms) {
        for (EKAlarm *alarm in event.alarms) {
            NSLog(@"แจ้งเตือน: %.0f นาทีก่อน", -alarm.relativeOffset / 60.0);
        }
    }
    
    if (event.hasRecurrenceRules) {
        NSLog(@"ซ้ำทุก: %@", [self recurrenceDescription:event.recurrenceRules.firstObject]);
    }
    
    NSLog(@"URL: %@", event.URL ?: @"ไม่มี");
}

- (NSString *)recurrenceDescription:(EKRecurrenceRule *)rule {
    switch (rule.frequency) {
        case EKRecurrenceFrequencyDaily:   return @"วัน";
        case EKRecurrenceFrequencyWeekly:  return @"สัปดาห์";
        case EKRecurrenceFrequencyMonthly: return @"เดือน";
        case EKRecurrenceFrequencyYearly:  return @"ปี";
        default: return @"ไม่ทราบ";
    }
}
```

---

## 68.8 Creating Events

### สร้าง Event ใหม่

```objc
- (void)createEvent {
    EventKitManager *manager = [EventKitManager sharedManager];
    
    [manager requestCalendarAccess:^(BOOL granted) {
        if (!granted) return;
        
        EKEventStore *store = manager.eventStore;
        EKEvent *event = [EKEvent eventWithEventStore:store];
        
        // ข้อมูลพื้นฐาน
        event.title = @"ประชุมทีม iOS";
        event.notes = @"ประชุมรายสัปดาห์เพื่อ review งาน";
        event.location = @"ห้องประชุม A3 ชั้น 5";
        event.URL = [NSURL URLWithString:@"https://meet.example.com/ios-team"];
        
        // วันเวลา
        NSCalendar *cal = [NSCalendar currentCalendar];
        NSDateComponents *startComp = [[NSDateComponents alloc] init];
        startComp.year = 2026;
        startComp.month = 10;
        startComp.day = 5;
        startComp.hour = 10;
        startComp.minute = 0;
        event.startDate = [cal dateFromComponents:startComp];
        
        NSDateComponents *endComp = [[NSDateComponents alloc] init];
        endComp.year = 2026;
        endComp.month = 10;
        endComp.day = 5;
        endComp.hour = 11;
        endComp.minute = 30;
        event.endDate = [cal dateFromComponents:endComp];
        
        // กำหนดปฏิทิน (ใช้ default calendar)
        event.calendar = store.defaultCalendarForNewEvents;
        
        // การแจ้งเตือน
        EKAlarm *alarm1 = [EKAlarm alarmWithRelativeOffset:-900]; // 15 นาทีก่อน
        EKAlarm *alarm2 = [EKAlarm alarmWithRelativeOffset:-3600]; // 1 ชั่วโมงก่อน
        event.alarms = @[alarm1, alarm2];
        
        // Availability
        event.availability = EKEventAvailabilityBusy;
        
        // บันทึก
        NSError *error;
        BOOL success = [store saveEvent:event span:EKSpanThisEvent error:&error];
        
        if (success) {
            NSLog(@"สร้าง event สำเร็จ: %@", event.eventIdentifier);
        } else {
            NSLog(@"สร้าง event ล้มเหลว: %@", error);
        }
    }];
}

// สร้าง recurring event
- (void)createRecurringEvent {
    EventKitManager *manager = [EventKitManager sharedManager];
    EKEventStore *store = manager.eventStore;
    
    EKEvent *event = [EKEvent eventWithEventStore:store];
    event.title = @"Exercise Daily";
    
    NSDate *startDate = [NSDate date];
    event.startDate = startDate;
    event.endDate = [startDate dateByAddingTimeInterval:3600]; // 1 ชั่วโมง
    event.calendar = store.defaultCalendarForNewEvents;
    
    // Recurrence rule - ทุกวัน ไม่มีวันสิ้นสุด
    EKRecurrenceRule *rule = [[EKRecurrenceRule alloc] 
        initRecurrenceWithFrequency:EKRecurrenceFrequencyDaily 
                           interval:1 
                              end:nil];
    
    event.recurrenceRules = @[rule];
    
    // หรือ ทุกสัปดาห์ วันจันทร์ พุธ ศุกร์
    EKRecurrenceDayOfWeek *mon = [EKRecurrenceDayOfWeek dayOfWeek:2]; // จันทร์
    EKRecurrenceDayOfWeek *wed = [EKRecurrenceDayOfWeek dayOfWeek:4]; // พุธ
    EKRecurrenceDayOfWeek *fri = [EKRecurrenceDayOfWeek dayOfWeek:6]; // ศุกร์
    
    EKRecurrenceRule *weeklyRule = [[EKRecurrenceRule alloc] 
        initRecurrenceWithFrequency:EKRecurrenceFrequencyWeekly 
                           interval:1 
                       daysOfTheWeek:@[mon, wed, fri] 
                      daysOfTheMonth:nil 
                     monthsOfTheYear:nil 
                      weeksOfTheYear:nil 
                       daysOfTheYear:nil 
                        setPositions:nil 
                                 end:[EKRecurrenceEnd recurrenceEndWithOccurrenceCount:52]];
    
    event.recurrenceRules = @[weeklyRule];
    
    NSError *error;
    [store saveEvent:event span:EKSpanThisEvent error:&error];
}

// สร้าง All-Day Event
- (void)createAllDayEvent:(NSDate *)date title:(NSString *)title {
    EventKitManager *manager = [EventKitManager sharedManager];
    EKEventStore *store = manager.eventStore;
    
    EKEvent *event = [EKEvent eventWithEventStore:store];
    event.title = title;
    event.allDay = YES;
    
    NSCalendar *cal = [NSCalendar currentCalendar];
    event.startDate = [cal startOfDayForDate:date];
    event.endDate = [cal startOfDayForDate:date]; // same day for all-day
    
    event.calendar = store.defaultCalendarForNewEvents;
    
    NSError *error;
    [store saveEvent:event span:EKSpanThisEvent error:&error];
    NSLog(@"สร้าง all-day event: %@", error ? error.localizedDescription : @"สำเร็จ");
}
```

### แก้ไขและลบ Event

```objc
- (void)updateEvent:(EKEvent *)event newTitle:(NSString *)newTitle {
    EKEventStore *store = [[EventKitManager sharedManager] eventStore];
    
    event.title = newTitle;
    
    NSError *error;
    // EKSpanThisEvent - แก้เฉพาะ occurrence นี้
    // EKSpanFutureEvents - แก้ occurrence นี้และที่จะมาทีหลัง
    BOOL success = [store saveEvent:event span:EKSpanThisEvent error:&error];
    
    NSLog(@"อัปเดต event: %@", success ? @"สำเร็จ" : error.localizedDescription);
}

- (void)deleteEvent:(EKEvent *)event {
    EKEventStore *store = [[EventKitManager sharedManager] eventStore];
    
    NSError *error;
    BOOL success = [store removeEvent:event span:EKSpanThisEvent error:&error];
    
    NSLog(@"ลบ event: %@", success ? @"สำเร็จ" : error.localizedDescription);
}
```

---

## 68.9 Reminders

### Fetching Reminders

```objc
- (void)fetchReminders {
    EventKitManager *manager = [EventKitManager sharedManager];
    
    [manager requestRemindersAccess:^(BOOL granted) {
        if (!granted) {
            NSLog(@"ไม่ได้รับสิทธิ์ Reminders");
            return;
        }
        
        EKEventStore *store = manager.eventStore;
        
        // ดึง reminder lists
        NSArray<EKCalendar *> *reminderLists = [store calendarsForEntityType:EKEntityTypeReminder];
        NSLog(@"Reminder lists: %lu", (unsigned long)reminderLists.count);
        
        for (EKCalendar *list in reminderLists) {
            NSLog(@"  - %@", list.title);
        }
        
        // ดึง incomplete reminders
        NSPredicate *predicate = [store predicateForIncompleteRemindersWithDueDateStarting:nil 
                                                                                   ending:nil 
                                                                               calendars:nil];
        
        [store fetchRemindersMatchingPredicate:predicate 
                                    completion:^(NSArray<EKReminder *> *reminders) {
            NSLog(@"Incomplete reminders: %lu", (unsigned long)reminders.count);
            
            for (EKReminder *reminder in reminders) {
                [self logReminderInfo:reminder];
            }
        }];
    }];
}

- (void)logReminderInfo:(EKReminder *)reminder {
    NSLog(@"=== Reminder: %@ ===", reminder.title);
    NSLog(@"เสร็จแล้ว: %@", reminder.isCompleted ? @"ใช่" : @"ไม่");
    NSLog(@"ลำดับ: %ld", (long)reminder.priority);
    NSLog(@"หมายเหตุ: %@", reminder.notes ?: @"ไม่มี");
    
    if (reminder.dueDateComponents) {
        NSCalendar *cal = [NSCalendar currentCalendar];
        NSDate *dueDate = [cal dateFromComponents:reminder.dueDateComponents];
        NSDateFormatter *f = [[NSDateFormatter alloc] init];
        f.dateStyle = NSDateFormatterMediumStyle;
        f.timeStyle = NSDateFormatterShortStyle;
        NSLog(@"กำหนดส่ง: %@", [f stringFromDate:dueDate]);
    }
}
```

### สร้าง Reminder

```objc
- (void)createReminder {
    EventKitManager *manager = [EventKitManager sharedManager];
    EKEventStore *store = manager.eventStore;
    
    EKReminder *reminder = [EKReminder reminderWithEventStore:store];
    
    reminder.title = @"ส่งรายงานประจำเดือน";
    reminder.notes = @"รายงานยอดขายเดือนกันยายน";
    
    // กำหนด calendar (default reminder calendar)
    reminder.calendar = [store defaultCalendarForNewReminders];
    
    // กำหนดวัน
    NSCalendar *cal = [NSCalendar currentCalendar];
    NSDateComponents *dueComponents = [[NSDateComponents alloc] init];
    dueComponents.year = 2026;
    dueComponents.month = 10;
    dueComponents.day = 1;
    dueComponents.hour = 9;
    dueComponents.minute = 0;
    reminder.dueDateComponents = dueComponents;
    
    // ลำดับความสำคัญ (1=สูงสุด, 5=ปกติ, 9=ต่ำสุด, 0=ไม่กำหนด)
    reminder.priority = 1;
    
    // การแจ้งเตือน
    EKAlarm *alarm = [EKAlarm alarmWithRelativeOffset:-3600]; // 1 ชั่วโมงก่อน
    reminder.alarms = @[alarm];
    
    // Location-based alarm
    EKAlarm *locationAlarm = [[EKAlarm alloc] init];
    CLLocation *officeLocation = [[CLLocation alloc] 
        initWithLatitude:13.7563 longitude:100.5018];
    EKStructuredLocation *structuredLocation = [EKStructuredLocation 
        locationWithTitle:@"สำนักงาน"];
    structuredLocation.geoLocation = officeLocation;
    structuredLocation.radius = 100.0; // 100 เมตร
    locationAlarm.structuredLocation = structuredLocation;
    locationAlarm.proximity = EKAlarmProximityEnter; // เมื่อถึง
    reminder.alarms = @[alarm, locationAlarm];
    
    // บันทึก
    NSError *error;
    BOOL success = [store saveReminder:reminder commit:YES error:&error];
    
    NSLog(@"สร้าง reminder: %@", success ? @"สำเร็จ" : error.localizedDescription);
}

// Mark reminder as complete
- (void)completeReminder:(EKReminder *)reminder {
    EKEventStore *store = [[EventKitManager sharedManager] eventStore];
    
    reminder.completed = YES;
    reminder.completionDate = [NSDate date];
    
    NSError *error;
    [store saveReminder:reminder commit:YES error:&error];
    NSLog(@"Mark reminder เสร็จสิ้น: %@", error ? error.localizedDescription : @"สำเร็จ");
}

// ลบ reminder
- (void)deleteReminder:(EKReminder *)reminder {
    EKEventStore *store = [[EventKitManager sharedManager] eventStore];
    
    NSError *error;
    [store removeReminder:reminder commit:YES error:&error];
    NSLog(@"ลบ reminder: %@", error ? error.localizedDescription : @"สำเร็จ");
}
```

---

## 68.10 EKEventEditViewController

UI controller สำหรับสร้างและแก้ไข events

```objc
// CalendarViewController.h
#import <UIKit/UIKit.h>
#import <EventKit/EventKit.h>
#import <EventKitUI/EventKitUI.h>

@interface CalendarViewController : UIViewController <EKEventEditViewDelegate>

@end
```

```objc
// CalendarViewController.m
#import "CalendarViewController.h"

@implementation CalendarViewController

// แสดง Event Editor
- (void)showEventEditor {
    EKEventStore *store = [[EventKitManager sharedManager] eventStore];
    
    EKEventEditViewController *editVC = [[EKEventEditViewController alloc] init];
    editVC.eventStore = store;
    editVC.editViewDelegate = self;
    
    // สร้าง event template
    EKEvent *event = [EKEvent eventWithEventStore:store];
    event.title = @"นัดหมายใหม่";
    event.startDate = [NSDate date];
    event.endDate = [[NSDate date] dateByAddingTimeInterval:3600];
    editVC.event = event;
    
    [self presentViewController:editVC animated:YES completion:nil];
}

// แก้ไข event ที่มีอยู่
- (void)editExistingEvent:(EKEvent *)event {
    EKEventStore *store = [[EventKitManager sharedManager] eventStore];
    
    EKEventEditViewController *editVC = [[EKEventEditViewController alloc] init];
    editVC.eventStore = store;
    editVC.event = event;
    editVC.editViewDelegate = self;
    
    [self presentViewController:editVC animated:YES completion:nil];
}

// EKEventEditViewDelegate
- (void)eventEditViewController:(EKEventEditViewController *)controller 
         didCompleteWithAction:(EKEventEditViewAction)action {
    
    switch (action) {
        case EKEventEditViewActionCanceled:
            NSLog(@"ผู้ใช้ยกเลิก");
            break;
        case EKEventEditViewActionSaved:
            NSLog(@"บันทึก event: %@", controller.event.title);
            break;
        case EKEventEditViewActionDeleted:
            NSLog(@"ลบ event");
            break;
    }
    
    [controller dismissViewControllerAnimated:YES completion:nil];
}

@end
```

---

## 68.11 Calendar Permissions (iOS 17+)

iOS 17 แนะนำ Write-Only access สำหรับปฏิทิน

```objc
- (void)checkCalendarPermission {
    if (@available(iOS 17.0, *)) {
        EKAuthorizationStatus status = [EKEventStore 
            authorizationStatusForEntityType:EKEntityTypeEvent];
        
        switch (status) {
            case EKAuthorizationStatusNotDetermined:
                NSLog(@"ยังไม่ตัดสินใจ");
                break;
            case EKAuthorizationStatusAuthorized:
                NSLog(@"Full Access - อ่านและเขียนได้");
                break;
            case EKAuthorizationStatusWriteOnly:
                NSLog(@"Write Only - เขียนเท่านั้น (iOS 17+)");
                // ไม่สามารถอ่าน events ได้
                break;
            case EKAuthorizationStatusDenied:
                NSLog(@"ปฏิเสธ");
                break;
            case EKAuthorizationStatusRestricted:
                NSLog(@"ถูกจำกัด");
                break;
        }
    }
}

// Request Write-Only Access (iOS 17+)
- (void)requestWriteOnlyAccess {
    if (@available(iOS 17.0, *)) {
        EKEventStore *store = [[EventKitManager sharedManager] eventStore];
        
        [store requestWriteOnlyAccessToEventsWithCompletion:^(BOOL granted, NSError *error) {
            if (granted) {
                NSLog(@"ได้รับ Write-Only access");
                // สามารถเพิ่ม events ได้แต่อ่านไม่ได้
            }
        }];
    }
}
```

---

## 68.12 Contacts และ Calendar: ตัวอย่างใช้งานจริง

### TableViewController สำหรับ Contacts

```objc
// ContactsTableViewController.h
#import <UIKit/UIKit.h>
#import <Contacts/Contacts.h>
#import <ContactsUI/ContactsUI.h>

@interface ContactsTableViewController : UITableViewController 
    <UISearchResultsUpdating, CNContactPickerDelegate>

@property (nonatomic, strong) NSArray<CNContact *> *allContacts;
@property (nonatomic, strong) NSArray<CNContact *> *filteredContacts;
@property (nonatomic, strong) UISearchController *searchController;

@end
```

```objc
// ContactsTableViewController.m
@implementation ContactsTableViewController

- (void)viewDidLoad {
    [super viewDidLoad];
    self.title = @"รายชื่อผู้ติดต่อ";
    
    // Setup search
    self.searchController = [[UISearchController alloc] initWithSearchResultsController:nil];
    self.searchController.searchResultsUpdater = self;
    self.searchController.obscuresBackgroundDuringPresentation = NO;
    self.navigationItem.searchController = self.searchController;
    
    // ปุ่มเพิ่ม
    self.navigationItem.rightBarButtonItem = [[UIBarButtonItem alloc] 
        initWithBarButtonSystemItem:UIBarButtonSystemItemAdd 
                             target:self 
                             action:@selector(addContactTapped)];
    
    [self loadContacts];
}

- (void)loadContacts {
    CNContactStore *store = [[CNContactStore alloc] init];
    
    CNAuthorizationStatus status = [CNContactStore 
        authorizationStatusForEntityType:CNEntityTypeContacts];
    
    if (status == CNAuthorizationStatusAuthorized) {
        [self fetchContacts:store];
    } else if (status == CNAuthorizationStatusNotDetermined) {
        [store requestAccessForEntityType:CNEntityTypeContacts 
                        completionHandler:^(BOOL granted, NSError *error) {
            if (granted) {
                dispatch_async(dispatch_get_main_queue(), ^{
                    [self fetchContacts:store];
                });
            }
        }];
    }
}

- (void)fetchContacts:(CNContactStore *)store {
    NSArray *keys = @[
        CNContactGivenNameKey,
        CNContactFamilyNameKey,
        CNContactPhoneNumbersKey,
        CNContactEmailAddressesKey,
        CNContactImageDataAvailableKey,
        CNContactThumbnailImageDataKey,
    ];
    
    CNContactFetchRequest *request = [[CNContactFetchRequest alloc] initWithKeysToFetch:keys];
    request.sortOrder = CNContactSortOrderGivenName;
    
    NSMutableArray *contacts = [NSMutableArray array];
    NSError *error;
    
    [store enumerateContactsWithFetchRequest:request error:&error usingBlock:^(CNContact *contact, BOOL *stop) {
        [contacts addObject:contact];
    }];
    
    self.allContacts = contacts;
    self.filteredContacts = contacts;
    
    dispatch_async(dispatch_get_main_queue(), ^{
        [self.tableView reloadData];
    });
}

#pragma mark - UISearchResultsUpdating

- (void)updateSearchResultsForSearchController:(UISearchController *)searchController {
    NSString *query = searchController.searchBar.text.lowercaseString;
    
    if (query.length == 0) {
        self.filteredContacts = self.allContacts;
    } else {
        self.filteredContacts = [self.allContacts filteredArrayUsingPredicate:
            [NSPredicate predicateWithBlock:^BOOL(CNContact *contact, NSDictionary *bindings) {
                NSString *name = [NSString stringWithFormat:@"%@ %@", 
                                  contact.givenName, contact.familyName].lowercaseString;
                return [name containsString:query];
            }]];
    }
    
    [self.tableView reloadData];
}

#pragma mark - TableView DataSource

- (NSInteger)tableView:(UITableView *)tableView numberOfRowsInSection:(NSInteger)section {
    return self.filteredContacts.count;
}

- (UITableViewCell *)tableView:(UITableView *)tableView 
         cellForRowAtIndexPath:(NSIndexPath *)indexPath {
    
    UITableViewCell *cell = [tableView dequeueReusableCellWithIdentifier:@"ContactCell" 
                                                            forIndexPath:indexPath];
    
    CNContact *contact = self.filteredContacts[indexPath.row];
    
    NSString *name = [CNContactFormatter stringFromContact:contact 
                                                     style:CNContactFormatterStyleFullName];
    cell.textLabel.text = name ?: @"(ไม่มีชื่อ)";
    
    if (contact.phoneNumbers.count > 0) {
        cell.detailTextLabel.text = contact.phoneNumbers.firstObject.value.stringValue;
    }
    
    if (contact.imageDataAvailable && contact.thumbnailImageData) {
        cell.imageView.image = [UIImage imageWithData:contact.thumbnailImageData];
        cell.imageView.layer.cornerRadius = 20;
        cell.imageView.clipsToBounds = YES;
    } else {
        cell.imageView.image = [UIImage systemImageNamed:@"person.circle.fill"];
    }
    
    return cell;
}

- (void)tableView:(UITableView *)tableView 
    didSelectRowAtIndexPath:(NSIndexPath *)indexPath {
    
    CNContact *contact = self.filteredContacts[indexPath.row];
    
    // ดึง full contact data
    CNContactStore *store = [[CNContactStore alloc] init];
    NSError *error;
    CNContact *fullContact = [store unifiedContactWithIdentifier:contact.identifier 
                                                    keysToFetch:[CNContactViewController descriptorForRequiredKeys]
                                                          error:&error];
    
    if (fullContact) {
        CNContactViewController *vc = [CNContactViewController viewControllerForContact:fullContact];
        vc.allowsEditing = YES;
        [self.navigationController pushViewController:vc animated:YES];
    }
    
    [tableView deselectRowAtIndexPath:indexPath animated:YES];
}

- (void)addContactTapped {
    CNContactPickerViewController *picker = [[CNContactPickerViewController alloc] init];
    picker.delegate = self;
    [self presentViewController:picker animated:YES completion:nil];
}

- (void)contactPickerDidCancel:(CNContactPickerViewController *)picker {
    [picker dismissViewControllerAnimated:YES completion:nil];
}

@end
```

---

## 68.13 แบบฝึกหัด

### แบบฝึกหัดที่ 1: Contacts Manager App
สร้าง Contacts app ที่:
- แสดงรายชื่อทั้งหมดพร้อม search
- ดู/แก้ไข/ลบ contact ได้
- เพิ่ม contact ใหม่ผ่าน UI
- Export contacts เป็น vCard (.vcf)

### แบบฝึกหัดที่ 2: Birthday Reminder App
สร้างแอปวันเกิดที่:
- ดึงวันเกิดจาก Contacts
- แสดงวันเกิดที่จะมาถึงใน 30 วัน
- สร้าง reminder อัตโนมัติ 1 วันก่อน
- แสดงอายุที่จะครบในปีนี้

### แบบฝึกหัดที่ 3: Calendar App
สร้าง calendar app ที่:
- แสดง events แบบ monthly view
- เพิ่ม/แก้ไข/ลบ events
- ตั้ง recurring events
- แสดง reminders

### แบบฝึกหัดที่ 4: Meeting Scheduler
สร้างระบบจัดประชุมที่:
- เลือกผู้เข้าร่วมจาก Contacts
- สร้าง calendar event
- ส่ง invite (ผ่าน email)
- จัดการ RSVP status

---

## สรุป

ในตอนนี้เราได้เรียนรู้:
- **Contacts Framework** การดึง, สร้าง, แก้ไขรายชื่อ
- **CNContactPickerViewController** UI สำหรับเลือก contact
- **CNContactViewController** UI สำหรับดู/แก้ไข contact
- **CNContactStore** การ fetch ด้วย predicate ต่างๆ
- **EventKit Framework** การจัดการปฏิทินและ reminders
- **EKEvent** สร้างและจัดการ events
- **EKReminder** สร้างและจัดการ reminders
- **EKEventEditViewController** UI สำหรับ event editor
- **Calendar Permissions** การขอสิทธิ์ iOS 17+

---

*หมายเหตุ: Contacts และ Calendar ต้องได้รับสิทธิ์จากผู้ใช้ก่อนเสมอ และข้อมูลส่วนตัวเหล่านี้ต้องได้รับการปกป้องตามนโยบายของ Apple App Store*
