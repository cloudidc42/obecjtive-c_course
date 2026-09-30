# Part 63 - iCloud

## บทนำ

iCloud เป็นบริการ cloud storage ของ Apple ที่ช่วยให้ข้อมูลของผู้ใช้ sync ระหว่าง devices ต่างๆ โดยอัตโนมัติ iOS นักพัฒนาสามารถใช้ iCloud ใน apps ผ่าน APIs หลายระดับ ตั้งแต่ simple key-value store ไปจนถึง full-featured CloudKit framework

ในบทนี้เราจะเรียนรู้วิธีใช้ iCloud ใน Objective-C ครอบคลุมทั้ง NSUbiquitousKeyValueStore และ CloudKit

---

## 63.1 iCloud Capabilities

### การเปิดใช้ iCloud ใน Xcode

1. เปิด Xcode → เลือก project
2. เลือก Target → Signing & Capabilities
3. กดปุ่ม `+` แล้วค้นหา "iCloud"
4. เปิดใช้ capabilities ที่ต้องการ:
   - **Key-value storage**: สำหรับ NSUbiquitousKeyValueStore
   - **iCloud Documents**: สำหรับ UIDocument/NSDocument
   - **CloudKit**: สำหรับ CKDatabase

### ประเภทของ iCloud Storage

```
1. NSUbiquitousKeyValueStore  - Key-value store (เหมือน UserDefaults)
                                ขีดจำกัด: 1MB รวม, 1024 keys, 1MB/key
   
2. iCloud Documents           - เก็บไฟล์ใน iCloud Drive
                                เหมาะสำหรับ document-based apps
   
3. CloudKit                   - Database บน cloud
                                มี public/private/shared databases
                                เหมาะสำหรับ structured data ซับซ้อน
```

---

## 63.2 NSUbiquitousKeyValueStore

NSUbiquitousKeyValueStore ใช้งานง่ายที่สุด เหมาะสำหรับเก็บ settings หรือข้อมูลขนาดเล็กที่ต้องการ sync

### การใช้งานพื้นฐาน

```objc
#import <Foundation/Foundation.h>

// ตรวจสอบว่า iCloud พร้อมใช้งาน
BOOL iCloudAvailable = [NSFileManager defaultManager].ubiquityIdentityToken != nil;

// เข้าถึง shared store
NSUbiquitousKeyValueStore *store = [NSUbiquitousKeyValueStore defaultStore];

// บันทึกข้อมูล
[store setString:@"John Doe" forKey:@"username"];
[store setLongLong:25 forKey:@"user_age"];
[store setDouble:3.14 forKey:@"pi_value"];
[store setBool:YES forKey:@"notifications_enabled"];
[store setObject:@[@"swift", @"objc"] forKey:@"favorite_languages"];

// อ่านข้อมูล
NSString *username = [store stringForKey:@"username"];
long long age = [store longLongForKey:@"user_age"];
double pi = [store doubleForKey:@"pi_value"];
BOOL notifications = [store boolForKey:@"notifications_enabled"];
NSArray *languages = (NSArray *)[store objectForKey:@"favorite_languages"];

// บังคับ sync ขึ้น iCloud
[store synchronize];
```

### รับแจ้งเตือนเมื่อข้อมูลเปลี่ยน

```objc
@interface SettingsViewController : UIViewController
@end

@implementation SettingsViewController

- (void)viewDidLoad {
    [super viewDidLoad];
    
    // ลงทะเบียน observer สำหรับการเปลี่ยนแปลง
    [[NSNotificationCenter defaultCenter] 
        addObserver:self
           selector:@selector(iCloudDidChange:)
               name:NSUbiquitousKeyValueStoreDidChangeExternallyNotification
             object:[NSUbiquitousKeyValueStore defaultStore]];
    
    // เริ่มรับการเปลี่ยนแปลงจาก iCloud
    [[NSUbiquitousKeyValueStore defaultStore] synchronize];
}

- (void)iCloudDidChange:(NSNotification *)notification {
    // เหตุผลที่เปลี่ยน
    NSInteger reason = [[notification.userInfo 
        objectForKey:NSUbiquitousKeyValueStoreChangeReasonKey] integerValue];
    
    // keys ที่เปลี่ยน
    NSArray *changedKeys = [notification.userInfo 
        objectForKey:NSUbiquitousKeyValueStoreChangedKeysKey];
    
    switch (reason) {
        case NSUbiquitousKeyValueStoreServerChange:
            NSLog(@"Changed by another device");
            break;
        case NSUbiquitousKeyValueStoreInitialSyncChange:
            NSLog(@"Initial sync from iCloud");
            break;
        case NSUbiquitousKeyValueStoreQuotaViolationChange:
            NSLog(@"Quota exceeded! Need to reduce data.");
            break;
        case NSUbiquitousKeyValueStoreAccountChange:
            NSLog(@"iCloud account changed");
            break;
    }
    
    NSLog(@"Changed keys: %@", changedKeys);
    
    // อัปเดต UI บน main thread
    dispatch_async(dispatch_get_main_queue(), ^{
        [self refreshUI];
    });
}

- (void)refreshUI {
    NSUbiquitousKeyValueStore *store = [NSUbiquitousKeyValueStore defaultStore];
    NSString *username = [store stringForKey:@"username"];
    NSLog(@"Updated username: %@", username);
    // อัปเดต UI...
}

- (void)dealloc {
    [[NSNotificationCenter defaultCenter] removeObserver:self];
}

@end
```

### iCloudUserDefaultsManager

```objc
// Wrapper ที่ sync ระหว่าง NSUserDefaults และ iCloud
@interface iCloudUserDefaultsManager : NSObject

+ (instancetype)sharedManager;

- (void)setObject:(id)object forKey:(NSString *)key;
- (id)objectForKey:(NSString *)key;
- (void)removeObjectForKey:(NSString *)key;
- (void)synchronize;

@end

@implementation iCloudUserDefaultsManager {
    NSUbiquitousKeyValueStore *_cloudStore;
    NSUserDefaults *_localDefaults;
}

+ (instancetype)sharedManager {
    static iCloudUserDefaultsManager *instance = nil;
    static dispatch_once_t onceToken;
    dispatch_once(&onceToken, ^{
        instance = [[iCloudUserDefaultsManager alloc] init];
    });
    return instance;
}

- (instancetype)init {
    self = [super init];
    if (self) {
        _cloudStore = [NSUbiquitousKeyValueStore defaultStore];
        _localDefaults = [NSUserDefaults standardUserDefaults];
        
        [[NSNotificationCenter defaultCenter]
            addObserver:self
               selector:@selector(cloudDidChange:)
                   name:NSUbiquitousKeyValueStoreDidChangeExternallyNotification
                 object:_cloudStore];
        
        [_cloudStore synchronize];
        [self mergeCloudToLocal];
    }
    return self;
}

- (void)setObject:(id)object forKey:(NSString *)key {
    [_localDefaults setObject:object forKey:key];
    [_cloudStore setObject:object forKey:key];
}

- (id)objectForKey:(NSString *)key {
    // อ่านจาก local ก่อน (เร็วกว่า)
    return [_localDefaults objectForKey:key];
}

- (void)removeObjectForKey:(NSString *)key {
    [_localDefaults removeObjectForKey:key];
    [_cloudStore removeObjectForKey:key];
}

- (void)synchronize {
    [_localDefaults synchronize];
    [_cloudStore synchronize];
}

// Sync cloud → local เมื่อมีการเปลี่ยนแปลงจาก device อื่น
- (void)mergeCloudToLocal {
    NSDictionary *cloudData = _cloudStore.dictionaryRepresentation;
    for (NSString *key in cloudData) {
        [_localDefaults setObject:cloudData[key] forKey:key];
    }
    [_localDefaults synchronize];
}

- (void)cloudDidChange:(NSNotification *)notification {
    NSArray *changedKeys = notification.userInfo[NSUbiquitousKeyValueStoreChangedKeysKey];
    for (NSString *key in changedKeys) {
        id cloudValue = [_cloudStore objectForKey:key];
        if (cloudValue) {
            [_localDefaults setObject:cloudValue forKey:key];
        }
    }
    [_localDefaults synchronize];
}

@end
```

---

## 63.3 CloudKit Framework Overview

CloudKit เป็น framework ที่ทรงพลังกว่า ช่วยให้ apps เก็บ structured data บน iCloud ได้

### Containers และ Databases

```objc
#import <CloudKit/CloudKit.h>

// Container = กล่องหลักของ app (bundle identifier)
CKContainer *container = [CKContainer defaultContainer];
// หรือระบุ container เอง:
// CKContainer *container = [CKContainer containerWithIdentifier:@"iCloud.com.mycompany.myapp"];

// Databases
CKDatabase *publicDB  = container.publicCloudDatabase;  // ทุก user อ่านได้
CKDatabase *privateDB = container.privateCloudDatabase; // เฉพาะ user นั้น
CKDatabase *sharedDB  = container.sharedCloudDatabase;  // แชร์กับคนที่รับเชิญ
```

### ประเภท Database

| Database | ใครอ่านได้ | ใครเขียนได้ | Quota |
|----------|-----------|------------|-------|
| Public   | ทุกคน | ต้องมี account | 1PB ฟรี |
| Private  | เจ้าของ | เจ้าของ | iCloud storage ของ user |
| Shared   | คนที่รับเชิญ | เจ้าของ | iCloud storage ของ user |

---

## 63.4 CKRecord - การสร้างและจัดการ Records

CKRecord คือ object พื้นฐานใน CloudKit คล้ายกับ row ในฐานข้อมูล

```objc
// สร้าง record ใหม่
CKRecord *noteRecord = [[CKRecord alloc] initWithRecordType:@"Note"];
noteRecord[@"title"] = @"My First Note";
noteRecord[@"body"] = @"This is the content of my note.";
noteRecord[@"isPinned"] = @YES;
noteRecord[@"viewCount"] = @(42);
noteRecord[@"tags"] = @[@"work", @"important"];

// ตรวจสอบ Record ID
NSLog(@"Record ID: %@", noteRecord.recordID.recordName);
NSLog(@"Record Type: %@", noteRecord.recordType);

// สร้าง record ด้วย Record ID ที่กำหนดเอง
CKRecordID *customID = [[CKRecordID alloc] initWithRecordName:@"note-001"];
CKRecord *customRecord = [[CKRecord alloc] initWithRecordType:@"Note" 
                                                     recordID:customID];

// ประเภทข้อมูลที่รองรับใน CKRecord
// NSString, NSNumber, NSDate, NSData, CKReference, CKAsset
// NSArray ของประเภทข้างต้น (ยกเว้น NSData และ CKAsset)

// สร้าง CKAsset สำหรับไฟล์/รูปภาพ
NSURL *fileURL = [NSURL fileURLWithPath:@"/path/to/image.jpg"];
CKAsset *imageAsset = [[CKAsset alloc] initWithFileURL:fileURL];
noteRecord[@"coverImage"] = imageAsset;

// สร้าง Reference ไปยัง record อื่น
CKRecord *authorRecord = [[CKRecord alloc] initWithRecordType:@"Author"];
authorRecord[@"name"] = @"John Doe";
CKReference *authorRef = [[CKReference alloc] initWithRecord:authorRecord 
                                                       action:CKReferenceActionDeleteSelf];
noteRecord[@"author"] = authorRef;
```

---

## 63.5 บันทึกและดึงข้อมูล Records

### การบันทึก (Save)

```objc
- (void)saveNoteRecord:(CKRecord *)record {
    CKDatabase *db = [CKContainer defaultContainer].privateCloudDatabase;
    
    CKModifyRecordsOperation *operation = [[CKModifyRecordsOperation alloc]
        initWithRecordsToSave:@[record]
            recordIDsToDelete:nil];
    
    operation.savePolicy = CKRecordSaveIfServerRecordUnchanged; // ป้องกัน conflict
    
    operation.perRecordProgressBlock = ^(CKRecord *record, double progress) {
        NSLog(@"Saving record %@: %.0f%%", record.recordID.recordName, progress * 100);
    };
    
    operation.perRecordCompletionBlock = ^(CKRecord *record, NSError *error) {
        if (error) {
            NSLog(@"Failed to save record %@: %@", record.recordID.recordName, error);
        } else {
            NSLog(@"Record saved: %@", record.recordID.recordName);
        }
    };
    
    operation.modifyRecordsCompletionBlock = ^(
        NSArray<CKRecord *> *savedRecords,
        NSArray<CKRecordID *> *deletedRecordIDs,
        NSError *operationError
    ) {
        if (operationError) {
            NSLog(@"Modify operation failed: %@", operationError);
            // Handle specific errors
            if ([operationError.domain isEqualToString:CKErrorDomain]) {
                switch (operationError.code) {
                    case CKErrorNetworkUnavailable:
                        NSLog(@"No network connection");
                        break;
                    case CKErrorQuotaExceeded:
                        NSLog(@"iCloud quota exceeded");
                        break;
                    case CKErrorNotAuthenticated:
                        NSLog(@"User not logged in to iCloud");
                        break;
                    default:
                        break;
                }
            }
        } else {
            NSLog(@"Saved %lu records", (unsigned long)savedRecords.count);
        }
    };
    
    [db addOperation:operation];
}
```

### การดึงข้อมูล Record โดยตรง (Fetch by ID)

```objc
- (void)fetchNoteWithID:(CKRecordID *)recordID 
             completion:(void (^)(CKRecord *record, NSError *error))completion {
    
    CKDatabase *db = [CKContainer defaultContainer].privateCloudDatabase;
    
    [db fetchRecordWithID:recordID completionHandler:^(CKRecord *record, NSError *error) {
        dispatch_async(dispatch_get_main_queue(), ^{
            completion(record, error);
        });
    }];
}

// Fetch หลาย records พร้อมกัน
- (void)fetchMultipleRecords:(NSArray<CKRecordID *> *)recordIDs 
                  completion:(void (^)(NSDictionary *recordsByID, NSError *error))completion {
    
    CKDatabase *db = [CKContainer defaultContainer].privateCloudDatabase;
    
    CKFetchRecordsOperation *operation = [[CKFetchRecordsOperation alloc]
        initWithRecordIDs:recordIDs];
    
    operation.desiredKeys = @[@"title", @"body", @"isPinned"]; // ดึงเฉพาะ fields ที่ต้องการ
    
    operation.perRecordCompletionBlock = ^(CKRecord *record, CKRecordID *recordID, NSError *error) {
        if (error) {
            NSLog(@"Failed to fetch record %@: %@", recordID.recordName, error);
        }
    };
    
    operation.fetchRecordsCompletionBlock = ^(NSDictionary *recordsByRecordID, NSError *error) {
        dispatch_async(dispatch_get_main_queue(), ^{
            completion(recordsByRecordID, error);
        });
    };
    
    [db addOperation:operation];
}
```

---

## 63.6 Queries - การค้นหาข้อมูล

```objc
// Simple Query
- (void)fetchAllNotes:(void (^)(NSArray *records, NSError *error))completion {
    CKDatabase *db = [CKContainer defaultContainer].privateCloudDatabase;
    
    // Predicate สำหรับ filter
    NSPredicate *predicate = [NSPredicate predicateWithValue:YES]; // ดึงทั้งหมด
    
    CKQuery *query = [[CKQuery alloc] initWithRecordType:@"Note" predicate:predicate];
    query.sortDescriptors = @[[NSSortDescriptor sortDescriptorWithKey:@"creationDate" 
                                                          ascending:NO]];
    
    [db performQuery:query 
           inZoneWithID:nil 
     completionHandler:^(NSArray *results, NSError *error) {
        dispatch_async(dispatch_get_main_queue(), ^{
            completion(results, error);
        });
    }];
}

// Query ด้วย Predicate
- (void)fetchPinnedNotes:(void (^)(NSArray *records, NSError *error))completion {
    CKDatabase *db = [CKContainer defaultContainer].privateCloudDatabase;
    
    NSPredicate *predicate = [NSPredicate predicateWithFormat:@"isPinned == %@", @YES];
    
    CKQuery *query = [[CKQuery alloc] initWithRecordType:@"Note" predicate:predicate];
    
    [db performQuery:query 
           inZoneWithID:nil 
     completionHandler:^(NSArray *results, NSError *error) {
        dispatch_async(dispatch_get_main_queue(), ^{
            completion(results, error);
        });
    }];
}

// Query ด้วย Full Text Search
- (void)searchNotesByTitle:(NSString *)searchText 
                completion:(void (^)(NSArray *records, NSError *error))completion {
    
    CKDatabase *db = [CKContainer defaultContainer].privateCloudDatabase;
    
    // ต้องสร้าง Queryable index สำหรับ field นี้ใน CloudKit Dashboard
    NSPredicate *predicate = [NSPredicate predicateWithFormat:@"title BEGINSWITH %@", searchText];
    
    CKQuery *query = [[CKQuery alloc] initWithRecordType:@"Note" predicate:predicate];
    
    [db performQuery:query
           inZoneWithID:nil
     completionHandler:^(NSArray *results, NSError *error) {
        dispatch_async(dispatch_get_main_queue(), ^{
            completion(results, error);
        });
    }];
}

// Paginated Query ด้วย CKQueryOperation
- (void)fetchNotesPaginated:(NSString *)cursor pageSize:(NSInteger)pageSize
                 completion:(void (^)(NSArray *records, NSString *nextCursor, NSError *error))completion {
    
    CKDatabase *db = [CKContainer defaultContainer].privateCloudDatabase;
    
    CKQueryOperation *operation;
    
    if (cursor) {
        CKQueryCursor *ckCursor = nil; // ควรแปลงจาก string จริงๆ
        operation = [[CKQueryOperation alloc] initWithCursor:ckCursor];
    } else {
        NSPredicate *pred = [NSPredicate predicateWithValue:YES];
        CKQuery *query = [[CKQuery alloc] initWithRecordType:@"Note" predicate:pred];
        query.sortDescriptors = @[[NSSortDescriptor sortDescriptorWithKey:@"creationDate" ascending:NO]];
        operation = [[CKQueryOperation alloc] initWithQuery:query];
    }
    
    operation.resultsLimit = pageSize;
    
    NSMutableArray *fetchedRecords = [NSMutableArray array];
    
    operation.recordFetchedBlock = ^(CKRecord *record) {
        [fetchedRecords addObject:record];
    };
    
    operation.queryCompletionBlock = ^(CKQueryCursor *nextCursor, NSError *error) {
        dispatch_async(dispatch_get_main_queue(), ^{
            // nextCursor != nil หมายความว่ายังมีข้อมูลอีก
            completion([fetchedRecords copy], nil, error);
        });
    };
    
    [db addOperation:operation];
}
```

---

## 63.7 Subscriptions - การรับ Real-time Updates

Subscription ทำให้ app ได้รับ push notification เมื่อข้อมูลใน CloudKit เปลี่ยนแปลง

### สร้าง Subscription

```objc
- (void)subscribeToNoteChanges {
    CKDatabase *db = [CKContainer defaultContainer].privateCloudDatabase;
    
    // Predicate สำหรับ filter events ที่สนใจ
    NSPredicate *predicate = [NSPredicate predicateWithValue:YES]; // ทุกการเปลี่ยนแปลง
    
    CKQuerySubscription *subscription = [[CKQuerySubscription alloc]
        initWithRecordType:@"Note"
                predicate:predicate
           subscriptionID:@"note-changes"
                  options:CKQuerySubscriptionOptionsFiresOnRecordCreation |
                          CKQuerySubscriptionOptionsFiresOnRecordUpdate |
                          CKQuerySubscriptionOptionsFiresOnRecordDeletion];
    
    // กำหนด notification ที่จะส่ง
    CKNotificationInfo *notificationInfo = [[CKNotificationInfo alloc] init];
    notificationInfo.alertBody = @"Notes have been updated";
    notificationInfo.shouldSendContentAvailable = YES; // Silent notification สำหรับ background update
    notificationInfo.desiredKeys = @[@"title"]; // ส่งข้อมูล key นี้มาใน notification
    
    subscription.notificationInfo = notificationInfo;
    
    [db saveSubscription:subscription completionHandler:^(CKSubscription *sub, NSError *error) {
        if (error) {
            // ถ้ามี subscription อยู่แล้ว (duplicate) ถือว่า ok
            if (error.code == CKErrorServerRejectedRequest) {
                NSLog(@"Subscription already exists");
            } else {
                NSLog(@"Failed to subscribe: %@", error);
            }
        } else {
            NSLog(@"Subscription created: %@", sub.subscriptionID);
        }
    }];
}

// ยกเลิก Subscription
- (void)unsubscribeFromNoteChanges {
    CKDatabase *db = [CKContainer defaultContainer].privateCloudDatabase;
    
    [db deleteSubscriptionWithID:@"note-changes" completionHandler:^(NSString *subscriptionID, NSError *error) {
        if (error) {
            NSLog(@"Failed to delete subscription: %@", error);
        } else {
            NSLog(@"Subscription deleted: %@", subscriptionID);
        }
    }];
}
```

### จัดการ Notification ที่มาจาก CloudKit

```objc
// ใน AppDelegate
- (void)application:(UIApplication *)app 
    didReceiveRemoteNotification:(NSDictionary *)userInfo 
          fetchCompletionHandler:(void (^)(UIBackgroundFetchResult))completionHandler {
    
    CKNotification *notification = [CKNotification notificationFromRemoteNotificationDictionary:userInfo];
    
    if (notification.notificationType == CKNotificationTypeQuery) {
        CKQueryNotification *queryNotification = (CKQueryNotification *)notification;
        
        NSLog(@"Record changed: %@", queryNotification.recordID.recordName);
        NSLog(@"Query reason: %ld", (long)queryNotification.queryNotificationReason);
        
        switch (queryNotification.queryNotificationReason) {
            case CKQueryNotificationReasonRecordCreated:
                [self handleRecordCreated:queryNotification.recordID];
                break;
            case CKQueryNotificationReasonRecordUpdated:
                [self handleRecordUpdated:queryNotification.recordID];
                break;
            case CKQueryNotificationReasonRecordDeleted:
                [self handleRecordDeleted:queryNotification.recordID];
                break;
        }
    }
    
    completionHandler(UIBackgroundFetchResultNewData);
}

- (void)handleRecordUpdated:(CKRecordID *)recordID {
    // ดึงข้อมูลล่าสุดจาก CloudKit
    CKDatabase *db = [CKContainer defaultContainer].privateCloudDatabase;
    [db fetchRecordWithID:recordID completionHandler:^(CKRecord *record, NSError *error) {
        if (record) {
            dispatch_async(dispatch_get_main_queue(), ^{
                // อัปเดต local data
                NSLog(@"Updated record: %@ - %@", record.recordID.recordName, record[@"title"]);
                // แจ้ง notification ให้ ViewControllers รีเฟรช
                [[NSNotificationCenter defaultCenter] postNotificationName:@"CloudKitRecordUpdated"
                                                                    object:record];
            });
        }
    }];
}
```

---

## 63.8 iCloud Drive Documents

สำหรับ document-based apps ที่ต้องการให้ user เข้าถึงไฟล์จาก Files app

### การเข้าถึง iCloud Container

```objc
- (NSURL *)iCloudDocumentsURL {
    // ต้องรันบน background thread
    NSURL *containerURL = [[NSFileManager defaultManager] 
        URLForUbiquityContainerIdentifier:nil]; // nil = container หลักของ app
    
    if (!containerURL) {
        NSLog(@"iCloud not available");
        return nil;
    }
    
    NSURL *documentsURL = [containerURL URLByAppendingPathComponent:@"Documents"];
    
    // สร้าง folder ถ้ายังไม่มี
    NSError *error;
    [[NSFileManager defaultManager] createDirectoryAtURL:documentsURL
                             withIntermediateDirectories:YES
                                             attributes:nil
                                                  error:&error];
    return documentsURL;
}

// ควรเรียกบน background thread
- (void)checkiCloudAvailability:(void (^)(BOOL available, NSURL *containerURL))completion {
    dispatch_async(dispatch_get_global_queue(DISPATCH_QUEUE_PRIORITY_DEFAULT, 0), ^{
        NSURL *containerURL = [[NSFileManager defaultManager] 
            URLForUbiquityContainerIdentifier:nil];
        
        dispatch_async(dispatch_get_main_queue(), ^{
            completion(containerURL != nil, containerURL);
        });
    });
}
```

### UIDocument สำหรับ iCloud Documents

```objc
// สร้าง UIDocument subclass
@interface NoteDocument : UIDocument

@property (nonatomic, strong) NSString *title;
@property (nonatomic, strong) NSString *body;
@property (nonatomic, strong) NSDate *lastModified;

@end

@implementation NoteDocument

// serialize ข้อมูลเป็น NSData สำหรับบันทึก
- (id)contentsForType:(NSString *)typeName error:(NSError **)outError {
    NSDictionary *data = @{
        @"title": self.title ?: @"",
        @"body": self.body ?: @"",
        @"lastModified": self.lastModified ?: [NSDate date]
    };
    
    NSError *error;
    NSData *jsonData = [NSJSONSerialization dataWithJSONObject:data options:0 error:&error];
    
    if (error) {
        if (outError) *outError = error;
        return nil;
    }
    
    return jsonData;
}

// deserialize ข้อมูลจาก NSData ที่โหลดมา
- (BOOL)loadFromContents:(id)contents 
                 ofType:(NSString *)typeName 
                  error:(NSError **)outError {
    if (![contents isKindOfClass:[NSData class]]) return NO;
    
    NSError *error;
    NSDictionary *data = [NSJSONSerialization JSONObjectWithData:contents 
                                                        options:0 
                                                          error:&error];
    if (error) {
        if (outError) *outError = error;
        return NO;
    }
    
    self.title = data[@"title"];
    self.body = data[@"body"];
    self.lastModified = data[@"lastModified"];
    
    return YES;
}

@end

// การใช้งาน UIDocument
- (void)openNoteDocument:(NSURL *)fileURL {
    NoteDocument *doc = [[NoteDocument alloc] initWithFileURL:fileURL];
    
    [doc openWithCompletionHandler:^(BOOL success) {
        if (success) {
            NSLog(@"Document opened: %@", doc.title);
            // แสดงเนื้อหาใน UI
        } else {
            NSLog(@"Failed to open document");
        }
    }];
}

- (void)saveNoteDocument:(NoteDocument *)doc {
    [doc saveToURL:doc.fileURL 
      forSaveOperation:UIDocumentSaveForCreating 
     completionHandler:^(BOOL success) {
        if (success) {
            NSLog(@"Document saved to iCloud");
        }
    }];
}
```

### NSMetadataQuery สำหรับค้นหาไฟล์ใน iCloud

```objc
@interface iCloudFileManager : NSObject {
    NSMetadataQuery *_metadataQuery;
}

- (void)startQueryingDocuments;
- (void)stopQueryingDocuments;

@end

@implementation iCloudFileManager

- (void)startQueryingDocuments {
    _metadataQuery = [[NSMetadataQuery alloc] init];
    
    // ค้นหาใน iCloud Documents
    _metadataQuery.searchScopes = @[NSMetadataQueryUbiquitousDocumentsScope];
    
    // Filter เฉพาะไฟล์ .note
    _metadataQuery.predicate = [NSPredicate predicateWithFormat:@"%K ENDSWITH '.note'",
                                 NSMetadataItemFSNameKey];
    
    // Sort by name
    _metadataQuery.sortDescriptors = @[
        [NSSortDescriptor sortDescriptorWithKey:NSMetadataItemFSNameKey ascending:YES]
    ];
    
    [[NSNotificationCenter defaultCenter] 
        addObserver:self
           selector:@selector(queryDidFinish:)
               name:NSMetadataQueryDidFinishGatheringNotification
             object:_metadataQuery];
    
    [[NSNotificationCenter defaultCenter] 
        addObserver:self
           selector:@selector(queryDidUpdate:)
               name:NSMetadataQueryDidUpdateNotification
             object:_metadataQuery];
    
    [_metadataQuery startQuery];
}

- (void)queryDidFinish:(NSNotification *)notification {
    [_metadataQuery disableUpdates];
    
    NSMutableArray *files = [NSMutableArray array];
    
    for (NSMetadataItem *item in _metadataQuery.results) {
        NSURL *fileURL = [item valueForAttribute:NSMetadataItemURLKey];
        NSString *fileName = [item valueForAttribute:NSMetadataItemFSNameKey];
        NSDate *modDate = [item valueForAttribute:NSMetadataItemFSContentChangeDateKey];
        NSNumber *fileSize = [item valueForAttribute:NSMetadataItemFSSizeKey];
        
        // ตรวจสอบสถานะ download
        NSString *downloadStatus = [item valueForAttribute:NSMetadataUbiquitousItemDownloadingStatusKey];
        BOOL isDownloaded = [downloadStatus isEqualToString:NSMetadataUbiquitousItemDownloadingStatusCurrent];
        
        [files addObject:@{
            @"url": fileURL,
            @"name": fileName ?: @"Unknown",
            @"modDate": modDate ?: [NSDate date],
            @"size": fileSize ?: @0,
            @"downloaded": @(isDownloaded)
        }];
    }
    
    NSLog(@"Found %lu iCloud documents", (unsigned long)files.count);
    
    [_metadataQuery enableUpdates];
    
    dispatch_async(dispatch_get_main_queue(), ^{
        [[NSNotificationCenter defaultCenter] postNotificationName:@"iCloudFilesUpdated"
                                                            object:files];
    });
}

- (void)queryDidUpdate:(NSNotification *)notification {
    // ไฟล์มีการเปลี่ยนแปลง (เพิ่ม/ลบ/แก้ไข)
    [self queryDidFinish:notification];
}

// ดาวน์โหลดไฟล์จาก iCloud
- (void)downloadFileAtURL:(NSURL *)url completion:(void (^)(NSError *))completion {
    NSError *error;
    BOOL started = [[NSFileManager defaultManager] 
        startDownloadingUbiquitousItemAtURL:url 
                                      error:&error];
    if (!started) {
        completion(error);
    }
    // ต้อง monitor ด้วย NSMetadataQuery เพื่อรู้ว่าดาวน์โหลดเสร็จเมื่อไร
}

- (void)stopQueryingDocuments {
    [_metadataQuery stopQuery];
    [[NSNotificationCenter defaultCenter] removeObserver:self];
}

@end
```

---

## 63.9 Conflict Resolution

เมื่อข้อมูลถูกแก้ไขจากหลาย devices พร้อมกัน CloudKit จะตรวจพบ conflict

### การจัดการ Conflict ใน CloudKit

```objc
// เมื่อ save record แล้วเจอ conflict
- (void)saveRecord:(CKRecord *)record {
    CKDatabase *db = [CKContainer defaultContainer].privateCloudDatabase;
    
    [db saveRecord:record completionHandler:^(CKRecord *savedRecord, NSError *error) {
        if (error) {
            if ([error.domain isEqualToString:CKErrorDomain] && 
                error.code == CKErrorServerRecordChanged) {
                
                // ดึง records จาก error info
                CKRecord *clientRecord = error.userInfo[CKRecordChangedErrorClientRecordKey];
                CKRecord *serverRecord = error.userInfo[CKRecordChangedErrorServerRecordKey];
                CKRecord *ancestorRecord = error.userInfo[CKRecordChangedErrorAncestorRecordKey];
                
                NSLog(@"Conflict detected!");
                NSLog(@"Client: %@", clientRecord[@"title"]);
                NSLog(@"Server: %@", serverRecord[@"title"]);
                
                // Resolve conflict
                CKRecord *resolvedRecord = [self resolveConflictBetween:clientRecord
                                                           serverRecord:serverRecord
                                                         ancestorRecord:ancestorRecord];
                
                // ลองบันทึกอีกครั้ง
                [self saveRecord:resolvedRecord];
            } else {
                NSLog(@"Save failed: %@", error);
            }
        }
    }];
}

// กลยุทธ์การ resolve conflict
- (CKRecord *)resolveConflictBetween:(CKRecord *)clientRecord
                        serverRecord:(CKRecord *)serverRecord
                      ancestorRecord:(CKRecord *)ancestorRecord {
    
    // Strategy 1: Server wins (ง่ายที่สุด)
    // return serverRecord;
    
    // Strategy 2: Client wins
    // CKRecord *mergedRecord = [serverRecord copy];
    // [clientRecord.allKeys enumerateObjectsUsingBlock:^(NSString *key, ...) {
    //     mergedRecord[key] = clientRecord[key];
    // }];
    // return mergedRecord;
    
    // Strategy 3: Last write wins (เปรียบเทียบ modificationDate)
    if ([clientRecord.modificationDate compare:serverRecord.modificationDate] == NSOrderedDescending) {
        // Client record ใหม่กว่า → ใช้ client version แต่ base บน server record
        CKRecord *mergedRecord = serverRecord; // ต้องใช้ server record เป็น base เพื่อ changeTag
        
        // Copy fields จาก client
        for (NSString *key in clientRecord.allKeys) {
            mergedRecord[key] = clientRecord[key];
        }
        
        return mergedRecord;
    } else {
        return serverRecord;
    }
}
```

### Optimistic Locking

```objc
// ใช้ savePolicy เพื่อควบคุม conflict resolution
CKModifyRecordsOperation *op = [[CKModifyRecordsOperation alloc]
    initWithRecordsToSave:@[record]
        recordIDsToDelete:nil];

// CKRecordSaveIfServerRecordUnchanged: save เฉพาะถ้า record ไม่ถูกแก้ไขจาก server
// ถ้า server มีการเปลี่ยนแปลง จะ error CKErrorServerRecordChanged
op.savePolicy = CKRecordSaveIfServerRecordUnchanged;

// CKRecordSaveChangedKeys: save เฉพาะ fields ที่เปลี่ยน (ลด conflict)
op.savePolicy = CKRecordSaveChangedKeys;

// CKRecordSaveAllKeys: save ทุก fields (overwrite server)
op.savePolicy = CKRecordSaveAllKeys;
```

---

## 63.10 Best Practices

### 1. ตรวจสอบ iCloud Availability ก่อนใช้

```objc
- (void)checkiCloudAndProceed {
    // ตรวจสอบว่า user login iCloud หรือยัง
    if ([[NSFileManager defaultManager] ubiquityIdentityToken] == nil) {
        // User ไม่ได้ login iCloud
        UIAlertController *alert = [UIAlertController 
            alertControllerWithTitle:@"iCloud ไม่พร้อมใช้งาน"
                             message:@"กรุณา login iCloud ใน Settings เพื่อ sync ข้อมูล"
                      preferredStyle:UIAlertControllerStyleAlert];
        
        [alert addAction:[UIAlertAction actionWithTitle:@"ตกลง" 
                                                 style:UIAlertActionStyleDefault 
                                               handler:nil]];
        
        [self presentViewController:alert animated:YES completion:nil];
        return;
    }
    
    // iCloud พร้อมใช้งาน
    [self syncWithiCloud];
}
```

### 2. Handle iCloud Account Changes

```objc
// ใน AppDelegate
- (void)setupiCloudNotifications {
    [[NSNotificationCenter defaultCenter]
        addObserver:self
           selector:@selector(iCloudAccountChanged:)
               name:NSUbiquityIdentityDidChangeNotification
             object:nil];
}

- (void)iCloudAccountChanged:(NSNotification *)notification {
    // User เปลี่ยน iCloud account หรือ sign out
    NSLog(@"iCloud account changed!");
    
    // ล้าง local cache และ reload
    [self clearLocalCache];
    [self checkiCloudAndProceed];
}
```

### 3. Background Sync

```objc
// sync ใน background เพื่อไม่ block UI
- (void)syncInBackground {
    dispatch_async(dispatch_get_global_queue(DISPATCH_QUEUE_PRIORITY_BACKGROUND, 0), ^{
        // ทำ CloudKit operations ที่นี่
        // Operations จะ complete บน background thread
        // ต้องอัปเดต UI บน main thread เสมอ
    });
}
```

### 4. Batch Operations

```objc
// ทำหลาย operations พร้อมกันเพื่อประสิทธิภาพ
- (void)batchSaveRecords:(NSArray<CKRecord *> *)records {
    CKDatabase *db = [CKContainer defaultContainer].privateCloudDatabase;
    
    // CloudKit รองรับ max 400 records ต่อ operation
    NSInteger batchSize = 400;
    
    for (NSInteger i = 0; i < records.count; i += batchSize) {
        NSRange range = NSMakeRange(i, MIN(batchSize, records.count - i));
        NSArray *batch = [records subarrayWithRange:range];
        
        CKModifyRecordsOperation *op = [[CKModifyRecordsOperation alloc]
            initWithRecordsToSave:batch
                recordIDsToDelete:nil];
        
        op.isAtomic = NO; // ถ้า YES = ทั้ง batch ต้องสำเร็จทั้งหมด
        
        op.modifyRecordsCompletionBlock = ^(NSArray *saved, NSArray *deleted, NSError *error) {
            NSLog(@"Batch %ld: saved %lu records", (long)(i/batchSize), (unsigned long)saved.count);
        };
        
        [db addOperation:op];
    }
}
```

### 5. Retry Logic สำหรับ Network Errors

```objc
- (void)saveRecordWithRetry:(CKRecord *)record attempt:(NSInteger)attempt {
    if (attempt > 3) {
        NSLog(@"Max retry attempts reached");
        return;
    }
    
    CKDatabase *db = [CKContainer defaultContainer].privateCloudDatabase;
    
    [db saveRecord:record completionHandler:^(CKRecord *saved, NSError *error) {
        if (error) {
            if (error.code == CKErrorNetworkUnavailable || 
                error.code == CKErrorNetworkFailure ||
                error.code == CKErrorServiceUnavailable) {
                
                // Retry หลัง delay แบบ exponential backoff
                NSTimeInterval delay = pow(2, attempt);
                NSLog(@"Retrying in %.0f seconds (attempt %ld)...", delay, (long)(attempt + 1));
                
                dispatch_after(dispatch_time(DISPATCH_TIME_NOW, (int64_t)(delay * NSEC_PER_SEC)),
                               dispatch_get_global_queue(DISPATCH_QUEUE_PRIORITY_DEFAULT, 0), ^{
                    [self saveRecordWithRetry:record attempt:attempt + 1];
                });
            } else {
                NSLog(@"Save failed with non-retryable error: %@", error);
            }
        }
    }];
}
```

---

## 63.11 แบบฝึกหัด

### แบบฝึกหัดที่ 1: Settings Sync

สร้าง SettingsSyncManager ที่:
- บันทึก app settings ลง NSUbiquitousKeyValueStore
- รับการเปลี่ยนแปลงจาก devices อื่น
- Merge กับ local settings อย่างถูกต้อง
- Handle quota exceeded error

### แบบฝึกหัดที่ 2: Notes App ด้วย CloudKit

สร้าง Notes app ที่:
- บันทึก notes ลง CloudKit private database
- รองรับ offline (เก็บ local cache)
- Sync เมื่อ network กลับมา
- Handle conflicts

```objc
@interface Note : NSObject
@property (nonatomic, strong) NSString *noteID;
@property (nonatomic, strong) NSString *title;
@property (nonatomic, strong) NSString *body;
@property (nonatomic, strong) NSDate *createdAt;
@property (nonatomic, strong) NSDate *modifiedAt;
@property (nonatomic, assign) BOOL isPendingSync;
@end

@interface NoteCloudKitManager : NSObject
- (void)saveNote:(Note *)note completion:(void (^)(NSError *))completion;
- (void)fetchAllNotes:(void (^)(NSArray *notes, NSError *))completion;
- (void)deleteNote:(Note *)note completion:(void (^)(NSError *))completion;
- (void)syncPendingNotes;
@end
```

### แบบฝึกหัดที่ 3: Photo Sharing

สร้างระบบแชร์รูปภาพด้วย CloudKit:
- อัปโหลดรูปเป็น CKAsset
- เก็บ metadata เป็น CKRecord
- Subscribe to changes
- Public database สำหรับ community photos

### แบบฝึกหัดที่ 4: Document App

สร้าง document-based app ที่:
- ใช้ UIDocument + iCloud Drive
- แสดงรายการไฟล์ด้วย NSMetadataQuery
- รองรับ conflict resolution
- รองรับ offline editing

---

## สรุป

ในบทนี้เราได้เรียนรู้:

1. **iCloud Capabilities**: การเปิดใช้งานและประเภทต่างๆ
2. **NSUbiquitousKeyValueStore**: sync ข้อมูลขนาดเล็กง่ายๆ
3. **CloudKit**: framework ทรงพลังสำหรับ structured data
4. **CKRecord**: การสร้างและจัดการข้อมูล
5. **Queries**: การค้นหาข้อมูลด้วยเงื่อนไข
6. **Subscriptions**: รับ real-time updates
7. **iCloud Documents**: จัดการไฟล์ใน iCloud Drive
8. **Conflict Resolution**: จัดการ conflicts อย่างถูกต้อง
9. **Best Practices**: การใช้งาน iCloud อย่างมีประสิทธิภาพ

บทต่อไปเราจะเรียนรู้เกี่ยวกับ **Push Notifications** ซึ่งทำให้ app ส่งข้อความถึง users แม้ app จะปิดอยู่
