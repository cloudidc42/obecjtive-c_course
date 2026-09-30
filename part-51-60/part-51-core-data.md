# Part 51: Core Data ใน Objective-C

## บทนำ (Introduction)

Core Data เป็น framework ของ Apple สำหรับ persistence (การเก็บข้อมูลถาวร) และ object graph management ใน iOS และ macOS มันไม่ใช่แค่ database - มันเป็น object-relational mapping (ORM) framework ที่ช่วยจัดการ lifecycle ของ objects และความสัมพันธ์ระหว่างกัน

### Core Data ทำอะไรได้บ้าง?

1. **Persistence** - บันทึกข้อมูลลง SQLite, XML, Binary, หรือ in-memory store
2. **Object Graph Management** - จัดการความสัมพันธ์ระหว่าง objects
3. **Lazy Loading** - โหลดข้อมูลเมื่อต้องการ ประหยัด memory
4. **Undo/Redo** - รองรับ undo/redo operations
5. **Data Validation** - validate ข้อมูลก่อนบันทึก
6. **Change Tracking** - ติดตามการเปลี่ยนแปลงของ objects

---

## 51.1 Core Data Stack Setup

Core Data Stack ประกอบด้วยส่วนสำคัญ 4 ส่วน:

1. **NSManagedObjectModel** - คำนิบนิยาม entities (schema)
2. **NSPersistentStoreCoordinator** - จัดการ persistent stores
3. **NSManagedObjectContext** - workspace สำหรับ managed objects
4. **NSPersistentContainer** (iOS 10+) - wrapper ที่รวม 3 อย่างข้างต้น

### การ Setup ด้วย NSPersistentContainer (แนะนำ)

```objc
// CoreDataStack.h
@interface CoreDataStack : NSObject

@property (nonatomic, readonly, strong) NSPersistentContainer *persistentContainer;
@property (nonatomic, readonly, strong) NSManagedObjectContext *mainContext;
@property (nonatomic, readonly, strong) NSManagedObjectContext *backgroundContext;

+ (instancetype)shared;
- (void)saveContext;
- (NSManagedObjectContext *)newPrivateContext;

@end

// CoreDataStack.m
@implementation CoreDataStack

+ (instancetype)shared {
    static CoreDataStack *instance = nil;
    static dispatch_once_t onceToken;
    dispatch_once(&onceToken, ^{
        instance = [[CoreDataStack alloc] init];
    });
    return instance;
}

- (instancetype)init {
    self = [super init];
    if (self) {
        [self setupStack];
    }
    return self;
}

- (void)setupStack {
    // สร้าง NSPersistentContainer
    // ชื่อ "MyApp" ต้องตรงกับไฟล์ MyApp.xcdatamodeld
    _persistentContainer = [[NSPersistentContainer alloc] initWithName:@"MyApp"];
    
    [_persistentContainer loadPersistentStoresWithCompletionHandler:^(
        NSPersistentStoreDescription *description, 
        NSError *error) {
        if (error) {
            NSLog(@"❌ Core Data Error: %@, %@", error, error.userInfo);
            // ใน production ควร handle error อย่างเหมาะสม
            abort();
        }
        NSLog(@"✅ Core Data loaded from: %@", description.URL);
    }];
    
    // ตั้งค่า main context
    _persistentContainer.viewContext.automaticallyMergesChangesFromParent = YES;
}

- (NSManagedObjectContext *)mainContext {
    return _persistentContainer.viewContext;
}

- (NSManagedObjectContext *)backgroundContext {
    return _persistentContainer.newBackgroundContext;
}

- (NSManagedObjectContext *)newPrivateContext {
    NSManagedObjectContext *context = [[NSManagedObjectContext alloc] 
        initWithConcurrencyType:NSPrivateQueueConcurrencyType];
    context.parentContext = self.mainContext;
    return context;
}

- (void)saveContext {
    NSManagedObjectContext *context = _persistentContainer.viewContext;
    
    if (!context.hasChanges) return;
    
    NSError *error;
    if (![context save:&error]) {
        NSLog(@"❌ Save error: %@, %@", error, error.userInfo);
    } else {
        NSLog(@"✅ Context saved successfully");
    }
}

@end
```

### การ Setup แบบ Manual (Legacy)

```objc
// AppDelegate.m
@interface AppDelegate ()
@property (nonatomic, strong) NSManagedObjectModel *managedObjectModel;
@property (nonatomic, strong) NSPersistentStoreCoordinator *persistentStoreCoordinator;
@property (nonatomic, strong) NSManagedObjectContext *managedObjectContext;
@end

@implementation AppDelegate

- (NSManagedObjectModel *)managedObjectModel {
    if (_managedObjectModel) return _managedObjectModel;
    
    NSURL *modelURL = [[NSBundle mainBundle] URLForResource:@"MyApp" 
                                             withExtension:@"momd"];
    _managedObjectModel = [[NSManagedObjectModel alloc] initWithContentsOfURL:modelURL];
    return _managedObjectModel;
}

- (NSPersistentStoreCoordinator *)persistentStoreCoordinator {
    if (_persistentStoreCoordinator) return _persistentStoreCoordinator;
    
    _persistentStoreCoordinator = [[NSPersistentStoreCoordinator alloc] 
        initWithManagedObjectModel:self.managedObjectModel];
    
    NSURL *storeURL = [self.applicationDocumentsDirectory 
        URLByAppendingPathComponent:@"MyApp.sqlite"];
    
    NSDictionary *options = @{
        NSMigratePersistentStoresAutomaticallyOption: @YES,
        NSInferMappingModelAutomaticallyOption: @YES
    };
    
    NSError *error;
    if (![_persistentStoreCoordinator addPersistentStoreWithType:NSSQLiteStoreType
                                                   configuration:nil
                                                             URL:storeURL
                                                         options:options
                                                           error:&error]) {
        NSLog(@"Error: %@", error);
        abort();
    }
    
    return _persistentStoreCoordinator;
}

- (NSManagedObjectContext *)managedObjectContext {
    if (_managedObjectContext) return _managedObjectContext;
    
    _managedObjectContext = [[NSManagedObjectContext alloc] 
        initWithConcurrencyType:NSMainQueueConcurrencyType];
    _managedObjectContext.persistentStoreCoordinator = self.persistentStoreCoordinator;
    
    return _managedObjectContext;
}

- (NSURL *)applicationDocumentsDirectory {
    return [[[NSFileManager defaultManager] 
        URLsForDirectory:NSDocumentDirectory 
        inDomains:NSUserDomainMask] lastObject];
}

@end
```

---

## 51.2 NSManagedObjectContext

`NSManagedObjectContext` คือ "scratch pad" หรือ workspace ที่คุณทำงานกับ managed objects ก่อนบันทึก

### Concurrency Types

```objc
// Main Queue Context - ใช้บน main thread
NSManagedObjectContext *mainContext = [[NSManagedObjectContext alloc] 
    initWithConcurrencyType:NSMainQueueConcurrencyType];

// Private Queue Context - ใช้ใน background
NSManagedObjectContext *bgContext = [[NSManagedObjectContext alloc] 
    initWithConcurrencyType:NSPrivateQueueConcurrencyType];
```

### การทำงานกับ Context

```objc
- (void)contextOperations {
    NSManagedObjectContext *context = [CoreDataStack shared].mainContext;
    
    // ตรวจสอบว่ามีการเปลี่ยนแปลงหรือไม่
    if (context.hasChanges) {
        NSLog(@"Context has unsaved changes");
    }
    
    // undo การเปลี่ยนแปลงล่าสุด
    [context undo];
    
    // redo
    [context redo];
    
    // reset (ล้างทุกอย่างใน context)
    [context reset];
    
    // rollback (ย้อนกลับ changes ทั้งหมดที่ยังไม่ save)
    [context rollback];
}
```

### Thread Safety

```objc
// การทำงานกับ private context ต้องใช้ performBlock:
- (void)performBackgroundWork {
    NSManagedObjectContext *bgContext = [CoreDataStack shared].backgroundContext;
    
    [bgContext performBlock:^{
        // ทำงานใน background thread
        NSError *error;
        NSFetchRequest *request = [NSFetchRequest fetchRequestWithEntityName:@"Person"];
        NSArray *results = [bgContext executeFetchRequest:request error:&error];
        
        // ต้อง update UI ใน main thread
        dispatch_async(dispatch_get_main_queue(), ^{
            NSLog(@"Found %lu people", (unsigned long)results.count);
        });
    }];
}

// performBlockAndWait: - รอให้เสร็จก่อน (synchronous)
[bgContext performBlockAndWait:^{
    // ...
}];
```

---

## 51.3 NSPersistentStoreCoordinator และ NSManagedObjectModel

### NSManagedObjectModel

Model คือ schema ที่กำหนด entities, attributes, และ relationships

```objc
// อ่าน model จาก bundle
NSURL *modelURL = [[NSBundle mainBundle] URLForResource:@"MyApp" 
                                         withExtension:@"momd"];
NSManagedObjectModel *model = [[NSManagedObjectModel alloc] initWithContentsOfURL:modelURL];

// ดู entities ทั้งหมด
NSDictionary *entities = model.entitiesByName;
NSLog(@"Entities: %@", entities.allKeys);

// สร้าง model แบบ programmatic (ไม่แนะนำแต่ทำได้)
- (NSManagedObjectModel *)createModelProgrammatically {
    NSManagedObjectModel *model = [[NSManagedObjectModel alloc] init];
    
    // สร้าง entity
    NSEntityDescription *personEntity = [[NSEntityDescription alloc] init];
    personEntity.name = @"Person";
    personEntity.managedObjectClassName = @"Person";
    
    // สร้าง attributes
    NSAttributeDescription *nameAttr = [[NSAttributeDescription alloc] init];
    nameAttr.name = @"name";
    nameAttr.attributeType = NSStringAttributeType;
    nameAttr.optional = NO;
    
    NSAttributeDescription *ageAttr = [[NSAttributeDescription alloc] init];
    ageAttr.name = @"age";
    ageAttr.attributeType = NSInteger32AttributeType;
    ageAttr.optional = YES;
    
    personEntity.properties = @[nameAttr, ageAttr];
    model.entities = @[personEntity];
    
    return model;
}
```

---

## 51.4 Entity, Attribute, Relationship

### กำหนดใน Xcode Data Model Editor

สร้างไฟล์ `.xcdatamodeld` ใน Xcode และกำหนด:

**Entity: Person**
- name: String (Required)
- age: Integer 16
- email: String (Optional)
- birthDate: Date
- createdAt: Date
- profilePhoto: Binary Data (External storage ON)

**Entity: Post**
- title: String (Required)
- content: String
- publishedAt: Date
- likeCount: Integer 32

**Relationship: Person -> Posts**
- Type: To Many
- Destination: Post
- Inverse: author (Post -> Person)

### NSManagedObject Subclass

```objc
// Person+CoreDataClass.h (auto-generated หรือสร้างเอง)
#import <CoreData/CoreData.h>

NS_ASSUME_NONNULL_BEGIN

@class Post;

@interface Person : NSManagedObject

@end

NS_ASSUME_NONNULL_END

#import "Person+CoreDataProperties.h"

// Person+CoreDataProperties.h
@interface Person (CoreDataProperties)

+ (NSFetchRequest<Person *> *)fetchRequest NS_SWIFT_NAME(fetchRequest());

@property (nullable, nonatomic, copy) NSString *name;
@property (nonatomic) int16_t age;
@property (nullable, nonatomic, copy) NSString *email;
@property (nullable, nonatomic, copy) NSDate *birthDate;
@property (nullable, nonatomic, copy) NSDate *createdAt;
@property (nullable, nonatomic, retain) NSSet<Post *> *posts;

@end

@interface Person (CoreDataGeneratedAccessors)

- (void)addPostsObject:(Post *)value;
- (void)removePostsObject:(Post *)value;
- (void)addPosts:(NSSet<Post *> *)values;
- (void)removePosts:(NSSet<Post *> *)values;

@end
```

---

## 51.5 NSFetchRequest

`NSFetchRequest` ใช้ดึงข้อมูลจาก Core Data store

```objc
// การ fetch อย่างง่าย
- (NSArray<Person *> *)fetchAllPersons {
    NSManagedObjectContext *context = [CoreDataStack shared].mainContext;
    
    NSFetchRequest<Person *> *request = [Person fetchRequest];
    // หรือ
    NSFetchRequest *request2 = [NSFetchRequest fetchRequestWithEntityName:@"Person"];
    
    NSError *error;
    NSArray<Person *> *results = [context executeFetchRequest:request error:&error];
    
    if (error) {
        NSLog(@"Fetch error: %@", error);
        return @[];
    }
    
    return results;
}
```

### Fetch Request Properties

```objc
- (NSArray *)fetchPersonsWithOptions {
    NSManagedObjectContext *context = [CoreDataStack shared].mainContext;
    NSFetchRequest<Person *> *request = [Person fetchRequest];
    
    // จำกัดจำนวน results
    request.fetchLimit = 20;
    
    // ข้าม N records แรก (สำหรับ pagination)
    request.fetchOffset = 40;  // เริ่มจาก record ที่ 41
    
    // Batch size สำหรับ memory efficiency
    request.fetchBatchSize = 20;
    
    // Include/exclude subentities
    request.includesSubentities = YES;
    
    // Return คืน managed objects หรือ IDs เท่านั้น
    request.resultType = NSManagedObjectResultType;  // default
    // NSCountResultType - คืนจำนวน
    // NSManagedObjectIDResultType - คืน object IDs
    // NSDictionaryResultType - คืน dictionaries
    
    NSError *error;
    return [context executeFetchRequest:request error:&error];
}
```

### Count Fetch

```objc
- (NSInteger)countPersons {
    NSManagedObjectContext *context = [CoreDataStack shared].mainContext;
    NSFetchRequest *request = [NSFetchRequest fetchRequestWithEntityName:@"Person"];
    
    NSError *error;
    NSInteger count = [context countForFetchRequest:request error:&error];
    
    if (error) {
        NSLog(@"Count error: %@", error);
        return 0;
    }
    
    return count;
}
```

---

## 51.6 NSPredicate กับ Core Data

NSPredicate ใช้กรองข้อมูลใน fetch request

```objc
// Comparison
NSPredicate *namePredicate = [NSPredicate predicateWithFormat:@"name == %@", @"John"];
NSPredicate *agePredicate = [NSPredicate predicateWithFormat:@"age >= %d", 18];
NSPredicate *emailPredicate = [NSPredicate predicateWithFormat:@"email CONTAINS[cd] %@", @"@gmail.com"];

// String matching
// CONTAINS - ประกอบด้วย
// BEGINSWITH - เริ่มต้นด้วย
// ENDSWITH - จบด้วย
// LIKE - wildcard (* = หลายตัว, ? = ตัวเดียว)
// MATCHES - regex
// [c] = case insensitive, [d] = diacritic insensitive, [cd] = ทั้งคู่
```

### ตัวอย่าง Predicates

```objc
- (void)demonstratePredicates {
    NSManagedObjectContext *context = [CoreDataStack shared].mainContext;
    NSFetchRequest<Person *> *request = [Person fetchRequest];
    
    // Example 1: ชื่อที่เฉพาะเจาะจง
    request.predicate = [NSPredicate predicateWithFormat:@"name == %@", @"สมชาย"];
    
    // Example 2: อายุมากกว่าหรือเท่ากับ
    request.predicate = [NSPredicate predicateWithFormat:@"age >= %d", 18];
    
    // Example 3: IN operator
    NSArray *names = @[@"Alice", @"Bob", @"Charlie"];
    request.predicate = [NSPredicate predicateWithFormat:@"name IN %@", names];
    
    // Example 4: BETWEEN
    request.predicate = [NSPredicate predicateWithFormat:@"age BETWEEN {%d, %d}", 20, 30];
    
    // Example 5: Nil check
    request.predicate = [NSPredicate predicateWithFormat:@"email != nil"];
    request.predicate = [NSPredicate predicateWithFormat:@"email == nil"];
    
    // Example 6: Compound predicate (AND)
    NSPredicate *pred1 = [NSPredicate predicateWithFormat:@"age >= %d", 18];
    NSPredicate *pred2 = [NSPredicate predicateWithFormat:@"name BEGINSWITH[c] %@", @"a"];
    NSPredicate *compound = [NSCompoundPredicate andPredicateWithSubpredicates:@[pred1, pred2]];
    request.predicate = compound;
    
    // Example 7: OR
    NSPredicate *orPredicate = [NSCompoundPredicate orPredicateWithSubpredicates:@[pred1, pred2]];
    
    // Example 8: NOT
    NSPredicate *notPredicate = [NSCompoundPredicate notPredicateWithSubpredicate:pred1];
    
    // Example 9: Relationship
    request.predicate = [NSPredicate predicateWithFormat:@"ANY posts.likeCount > %d", 100];
    
    // Example 10: Date comparison
    NSDate *oneMonthAgo = [[NSDate date] dateByAddingTimeInterval:-(30 * 24 * 3600)];
    request.predicate = [NSPredicate predicateWithFormat:@"createdAt >= %@", oneMonthAgo];
    
    NSError *error;
    NSArray *results = [context executeFetchRequest:request error:&error];
    NSLog(@"Results: %lu", (unsigned long)results.count);
}
```

### Predicate Template

```objc
// Predicate template สำหรับ reuse
- (NSPredicate *)predicateForAgeRange:(NSInteger)minAge maxAge:(NSInteger)maxAge {
    return [NSPredicate predicateWithFormat:@"age >= %ld AND age <= %ld", 
            (long)minAge, (long)maxAge];
}

// ใช้ NSExpression
- (NSPredicate *)predicateWithTemplate {
    NSDictionary *variables = @{@"NAME": @"John"};
    NSPredicate *template = [NSPredicate predicateWithFormat:@"name == $NAME"];
    return [template predicateWithSubstitutionVariables:variables];
}
```

---

## 51.7 NSSortDescriptor

```objc
- (NSArray<Person *> *)fetchSortedPersons {
    NSManagedObjectContext *context = [CoreDataStack shared].mainContext;
    NSFetchRequest<Person *> *request = [Person fetchRequest];
    
    // เรียงตามชื่อ A-Z
    NSSortDescriptor *nameSort = [NSSortDescriptor sortDescriptorWithKey:@"name" 
                                                               ascending:YES];
    
    // เรียงตามอายุ มาก -> น้อย
    NSSortDescriptor *ageSort = [NSSortDescriptor sortDescriptorWithKey:@"age" 
                                                              ascending:NO];
    
    // เรียงตามวันที่สร้าง ล่าสุดก่อน
    NSSortDescriptor *dateSort = [NSSortDescriptor sortDescriptorWithKey:@"createdAt" 
                                                               ascending:NO];
    
    // เรียงด้วย selector (สำหรับ string comparison ที่ดีกว่า)
    NSSortDescriptor *localizedSort = [NSSortDescriptor sortDescriptorWithKey:@"name" 
                                                                    ascending:YES 
                                                                     selector:@selector(localizedCaseInsensitiveCompare:)];
    
    // ใช้หลาย sort descriptors (เรียงตามลำดับ)
    request.sortDescriptors = @[dateSort, nameSort];
    
    NSError *error;
    return [context executeFetchRequest:request error:&error];
}
```

---

## 51.8 Creating Managed Objects (Create)

```objc
// วิธีที่ 1: ใช้ insertNewObjectForEntityForName:inManagedObjectContext:
- (Person *)createPersonWithName:(NSString *)name age:(NSInteger)age {
    NSManagedObjectContext *context = [CoreDataStack shared].mainContext;
    
    Person *person = [NSEntityDescription insertNewObjectForEntityForName:@"Person" 
                                                   inManagedObjectContext:context];
    person.name = name;
    person.age = age;
    person.createdAt = [NSDate date];
    
    return person;
}

// วิธีที่ 2: ใช้ NSManagedObject init (ถ้ามี subclass)
- (Person *)createPersonModern:(NSString *)name {
    NSManagedObjectContext *context = [CoreDataStack shared].mainContext;
    
    Person *person = [[Person alloc] initWithContext:context];
    person.name = name;
    person.createdAt = [NSDate date];
    
    return person;
}

// บันทึก
- (void)savePerson:(Person *)person {
    NSManagedObjectContext *context = person.managedObjectContext;
    NSError *error;
    
    if ([context save:&error]) {
        NSLog(@"✅ Saved: %@", person.name);
    } else {
        NSLog(@"❌ Error: %@", error);
    }
}
```

### CRUD - Create ครบถ้วน

```objc
// PersonRepository.h
@interface PersonRepository : NSObject

- (Person *)createPersonWithName:(NSString *)name 
                             age:(NSInteger)age 
                           email:(nullable NSString *)email;
- (NSArray<Person *> *)fetchAllPersons;
- (NSArray<Person *> *)fetchPersonsWithPredicate:(NSPredicate *)predicate;
- (Person *)fetchPersonWithObjectID:(NSManagedObjectID *)objectID;
- (void)updatePerson:(Person *)person name:(NSString *)name;
- (BOOL)deletePerson:(Person *)person;
- (BOOL)save;

@end

// PersonRepository.m
@implementation PersonRepository

- (Person *)createPersonWithName:(NSString *)name 
                             age:(NSInteger)age 
                           email:(nullable NSString *)email {
    NSManagedObjectContext *context = [CoreDataStack shared].mainContext;
    
    Person *person = [[Person alloc] initWithContext:context];
    person.name = name;
    person.age = age;
    person.email = email;
    person.createdAt = [NSDate date];
    
    return person;
}

- (NSArray<Person *> *)fetchAllPersons {
    NSManagedObjectContext *context = [CoreDataStack shared].mainContext;
    NSFetchRequest<Person *> *request = [Person fetchRequest];
    request.sortDescriptors = @[[NSSortDescriptor sortDescriptorWithKey:@"name" ascending:YES]];
    
    NSError *error;
    NSArray *results = [context executeFetchRequest:request error:&error];
    return results ?: @[];
}

- (NSArray<Person *> *)fetchPersonsWithPredicate:(NSPredicate *)predicate {
    NSManagedObjectContext *context = [CoreDataStack shared].mainContext;
    NSFetchRequest<Person *> *request = [Person fetchRequest];
    request.predicate = predicate;
    
    NSError *error;
    return [context executeFetchRequest:request error:&error] ?: @[];
}

- (Person *)fetchPersonWithObjectID:(NSManagedObjectID *)objectID {
    NSManagedObjectContext *context = [CoreDataStack shared].mainContext;
    NSError *error;
    return [context existingObjectWithID:objectID error:&error];
}

- (void)updatePerson:(Person *)person name:(NSString *)name {
    person.name = name;
    // การเปลี่ยนแปลงจะถูก track โดย context อัตโนมัติ
}

- (BOOL)deletePerson:(Person *)person {
    NSManagedObjectContext *context = person.managedObjectContext;
    [context deleteObject:person];
    return [self save];
}

- (BOOL)save {
    NSManagedObjectContext *context = [CoreDataStack shared].mainContext;
    if (!context.hasChanges) return YES;
    
    NSError *error;
    BOOL success = [context save:&error];
    if (!success) {
        NSLog(@"Save error: %@", error);
    }
    return success;
}

@end
```

---

## 51.9 Updating และ Deleting Objects

### Update

```objc
- (void)updatePersonEmail:(Person *)person newEmail:(NSString *)email {
    // เพียงแค่เปลี่ยน property
    person.email = email;
    
    // Context จะ track การเปลี่ยนแปลงอัตโนมัติ
    // บันทึกเมื่อพร้อม
    NSError *error;
    [person.managedObjectContext save:&error];
}

// Batch Update (iOS 8+) - ประสิทธิภาพดีกว่าสำหรับ mass update
- (void)batchUpdateEmail {
    NSManagedObjectContext *context = [CoreDataStack shared].mainContext;
    
    NSBatchUpdateRequest *batchUpdate = [[NSBatchUpdateRequest alloc] 
        initWithEntityName:@"Person"];
    batchUpdate.predicate = [NSPredicate predicateWithFormat:@"email == nil"];
    batchUpdate.propertiesToUpdate = @{@"email": @"unknown@example.com"};
    batchUpdate.resultType = NSUpdatedObjectsCountResultType;
    
    NSError *error;
    NSBatchUpdateResult *result = [context executeRequest:batchUpdate error:&error];
    NSLog(@"Updated %@ objects", result.result);
    
    // Merge changes กับ context
    [NSManagedObjectContext mergeChangesFromRemoteContextSave:@{
        NSUpdatedObjectsKey: result.result
    } intoContexts:@[context]];
}
```

### Delete

```objc
// Delete เดี่ยว
- (void)deletePerson:(Person *)person {
    NSManagedObjectContext *context = person.managedObjectContext;
    [context deleteObject:person];
    
    NSError *error;
    [context save:&error];
}

// Delete ทุก person ที่ตรงตาม predicate
- (void)deletePersonsWithPredicate:(NSPredicate *)predicate {
    NSManagedObjectContext *context = [CoreDataStack shared].mainContext;
    NSFetchRequest<Person *> *request = [Person fetchRequest];
    request.predicate = predicate;
    
    NSError *error;
    NSArray<Person *> *persons = [context executeFetchRequest:request error:&error];
    
    for (Person *person in persons) {
        [context deleteObject:person];
    }
    
    [context save:&error];
}

// Batch Delete (iOS 9+) - เร็วมาก ไม่โหลด objects เข้า memory
- (void)batchDeleteOldPersons {
    NSManagedObjectContext *context = [CoreDataStack shared].mainContext;
    
    NSFetchRequest *request = [NSFetchRequest fetchRequestWithEntityName:@"Person"];
    NSDate *oneYearAgo = [[NSDate date] dateByAddingTimeInterval:-(365 * 24 * 3600)];
    request.predicate = [NSPredicate predicateWithFormat:@"createdAt < %@", oneYearAgo];
    
    NSBatchDeleteRequest *batchDelete = [[NSBatchDeleteRequest alloc] 
        initWithFetchRequest:request];
    batchDelete.resultType = NSBatchDeleteResultTypeObjectIDs;
    
    NSError *error;
    NSBatchDeleteResult *result = [context executeRequest:batchDelete error:&error];
    
    // Merge changes
    NSArray *deletedObjectIDs = result.result;
    [NSManagedObjectContext mergeChangesFromRemoteContextSave:@{
        NSDeletedObjectsKey: deletedObjectIDs
    } intoContexts:@[context]];
}
```

---

## 51.10 Saving Context

```objc
// Save method ที่ robust
- (BOOL)saveContext:(NSManagedObjectContext *)context error:(NSError **)error {
    if (!context.hasChanges) {
        NSLog(@"No changes to save");
        return YES;
    }
    
    NSError *saveError;
    BOOL success = [context save:&saveError];
    
    if (!success) {
        NSLog(@"Failed to save: %@\n%@", saveError, saveError.userInfo);
        if (error) {
            *error = saveError;
        }
        return NO;
    }
    
    // ถ้าเป็น child context ต้อง save parent ด้วย
    if (context.parentContext) {
        return [self saveContext:context.parentContext error:error];
    }
    
    return YES;
}

// Save ทั้ง background และ main context
- (void)saveBackgroundContext:(NSManagedObjectContext *)bgContext 
                   completion:(void(^)(BOOL success, NSError *error))completion {
    [bgContext performBlock:^{
        NSError *error;
        BOOL success = [bgContext save:&error];
        
        if (success && bgContext.parentContext) {
            // Save parent context ใน main thread
            [bgContext.parentContext performBlock:^{
                NSError *parentError;
                BOOL parentSuccess = [bgContext.parentContext save:&parentError];
                
                dispatch_async(dispatch_get_main_queue(), ^{
                    if (completion) completion(parentSuccess, parentError);
                });
            }];
        } else {
            dispatch_async(dispatch_get_main_queue(), ^{
                if (completion) completion(success, error);
            });
        }
    }];
}
```

---

## 51.11 NSFetchedResultsController

`NSFetchedResultsController` ใช้สำหรับ efficiently แสดงข้อมูลจาก Core Data ใน table view หรือ collection view

```objc
// ViewController.h
@interface PersonListViewController : UITableViewController <NSFetchedResultsControllerDelegate>
@property (nonatomic, strong) NSFetchedResultsController<Person *> *fetchedResultsController;
@end

// ViewController.m
@implementation PersonListViewController

- (void)viewDidLoad {
    [super viewDidLoad];
    [self.tableView registerClass:[UITableViewCell class] forCellReuseIdentifier:@"Cell"];
    self.title = @"รายการบุคคล";
    
    // Setup FRC
    [self setupFetchedResultsController];
    
    // Add button
    self.navigationItem.rightBarButtonItem = [[UIBarButtonItem alloc] 
        initWithBarButtonSystemItem:UIBarButtonSystemItemAdd
        target:self action:@selector(addPerson)];
}

- (void)setupFetchedResultsController {
    NSManagedObjectContext *context = [CoreDataStack shared].mainContext;
    
    NSFetchRequest<Person *> *request = [Person fetchRequest];
    
    // ต้องมี sort descriptor อย่างน้อย 1 ตัว
    NSSortDescriptor *nameSort = [NSSortDescriptor sortDescriptorWithKey:@"name" ascending:YES];
    request.sortDescriptors = @[nameSort];
    
    // Batch size สำหรับ performance
    request.fetchBatchSize = 20;
    
    // สร้าง FRC
    self.fetchedResultsController = [[NSFetchedResultsController alloc] 
        initWithFetchRequest:request
        managedObjectContext:context
        sectionNameKeyPath:nil  // ถ้าต้องการ sections ใส่ key path เช่น @"name.firstLetter"
        cacheName:nil];  // ใส่ชื่อ cache เพื่อ performance
    
    self.fetchedResultsController.delegate = self;
    
    NSError *error;
    if (![self.fetchedResultsController performFetch:&error]) {
        NSLog(@"Fetch error: %@", error);
    }
}

#pragma mark - UITableViewDataSource

- (NSInteger)numberOfSectionsInTableView:(UITableView *)tableView {
    return self.fetchedResultsController.sections.count;
}

- (NSInteger)tableView:(UITableView *)tableView numberOfRowsInSection:(NSInteger)section {
    id<NSFetchedResultsSectionInfo> sectionInfo = self.fetchedResultsController.sections[section];
    return sectionInfo.numberOfObjects;
}

- (UITableViewCell *)tableView:(UITableView *)tableView 
         cellForRowAtIndexPath:(NSIndexPath *)indexPath {
    UITableViewCell *cell = [tableView dequeueReusableCellWithIdentifier:@"Cell" 
                                                            forIndexPath:indexPath];
    
    Person *person = [self.fetchedResultsController objectAtIndexPath:indexPath];
    cell.textLabel.text = person.name;
    cell.detailTextLabel.text = [NSString stringWithFormat:@"อายุ: %d", person.age];
    
    return cell;
}

- (NSString *)tableView:(UITableView *)tableView titleForHeaderInSection:(NSInteger)section {
    id<NSFetchedResultsSectionInfo> sectionInfo = self.fetchedResultsController.sections[section];
    return sectionInfo.name;
}

- (void)tableView:(UITableView *)tableView 
commitEditingStyle:(UITableViewCellEditingStyle)editingStyle 
forRowAtIndexPath:(NSIndexPath *)indexPath {
    if (editingStyle == UITableViewCellEditingStyleDelete) {
        Person *person = [self.fetchedResultsController objectAtIndexPath:indexPath];
        NSManagedObjectContext *context = [CoreDataStack shared].mainContext;
        [context deleteObject:person];
        
        NSError *error;
        [context save:&error];
    }
}

#pragma mark - NSFetchedResultsControllerDelegate

- (void)controllerWillChangeContent:(NSFetchedResultsController *)controller {
    [self.tableView beginUpdates];
}

- (void)controller:(NSFetchedResultsController *)controller 
  didChangeSection:(id<NSFetchedResultsSectionInfo>)sectionInfo
           atIndex:(NSUInteger)sectionIndex 
     forChangeType:(NSFetchedResultsChangeType)type {
    switch (type) {
        case NSFetchedResultsChangeInsert:
            [self.tableView insertSections:[NSIndexSet indexSetWithIndex:sectionIndex]
                          withRowAnimation:UITableViewRowAnimationFade];
            break;
        case NSFetchedResultsChangeDelete:
            [self.tableView deleteSections:[NSIndexSet indexSetWithIndex:sectionIndex]
                          withRowAnimation:UITableViewRowAnimationFade];
            break;
        default:
            break;
    }
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
                                 withRowAnimation:UITableViewRowAnimationFade];
            break;
        case NSFetchedResultsChangeUpdate:
            [self.tableView reloadRowsAtIndexPaths:@[indexPath]
                                 withRowAnimation:UITableViewRowAnimationNone];
            break;
        case NSFetchedResultsChangeMove:
            [self.tableView moveRowAtIndexPath:indexPath 
                                  toIndexPath:newIndexPath];
            break;
    }
}

- (void)controllerDidChangeContent:(NSFetchedResultsController *)controller {
    [self.tableView endUpdates];
}

#pragma mark - Actions

- (void)addPerson {
    NSManagedObjectContext *context = [CoreDataStack shared].mainContext;
    Person *person = [[Person alloc] initWithContext:context];
    person.name = [NSString stringWithFormat:@"Person %d", arc4random_uniform(100)];
    person.age = 20 + arc4random_uniform(40);
    person.createdAt = [NSDate date];
    
    NSError *error;
    [context save:&error];
}

@end
```

### FRC กับ Sections

```objc
- (void)setupFRCWithSections {
    NSManagedObjectContext *context = [CoreDataStack shared].mainContext;
    NSFetchRequest<Person *> *request = [Person fetchRequest];
    
    // ต้อง sort ตาม section key ก่อน
    NSSortDescriptor *sectionSort = [NSSortDescriptor sortDescriptorWithKey:@"department" ascending:YES];
    NSSortDescriptor *nameSort = [NSSortDescriptor sortDescriptorWithKey:@"name" ascending:YES];
    request.sortDescriptors = @[sectionSort, nameSort];
    
    // sectionNameKeyPath - key path ที่ใช้ group เป็น sections
    self.fetchedResultsController = [[NSFetchedResultsController alloc] 
        initWithFetchRequest:request
        managedObjectContext:context
        sectionNameKeyPath:@"department"  // แบ่ง section ตาม department
        cacheName:@"PersonCache"];
    
    self.fetchedResultsController.delegate = self;
    [self.fetchedResultsController performFetch:nil];
}
```

---

## 51.12 Core Data Migrations (Lightweight)

เมื่อ data model เปลี่ยน ต้องทำ migration

### Lightweight Migration (อัตโนมัติ)

รองรับการเปลี่ยนแปลงที่ไม่ซับซ้อน:
- เพิ่ม entity ใหม่
- เพิ่ม attribute ใหม่ (ต้องมี default value หรือ optional)
- เปลี่ยนชื่อ entity/attribute โดยใช้ renaming identifier
- ลบ entity/attribute (ข้อมูลจะหาย)

```objc
// ตั้งค่า migration options
NSDictionary *options = @{
    NSMigratePersistentStoresAutomaticallyOption: @YES,
    NSInferMappingModelAutomaticallyOption: @YES
};

[coordinator addPersistentStoreWithType:NSSQLiteStoreType
                          configuration:nil
                                    URL:storeURL
                                options:options
                                  error:&error];
```

### การทำ Migration Version

1. ใน Xcode เปิด `.xcdatamodeld`
2. Editor > Add Model Version
3. ตั้งชื่อ version ใหม่ (เช่น MyApp 2)
4. ทำการเปลี่ยนแปลงใน version ใหม่
5. ตั้ง current version เป็น version ใหม่

```objc
// ตรวจสอบ version ปัจจุบัน
- (void)checkModelVersion {
    NSManagedObjectModel *model = [CoreDataStack shared].persistentContainer.managedObjectModel;
    NSDictionary *metadata = [NSPersistentStoreCoordinator 
        metadataForPersistentStoreOfType:NSSQLiteStoreType 
        URL:storeURL 
        options:nil 
        error:nil];
    
    BOOL isCompatible = [model isConfiguration:nil 
        compatibleWithStoreMetadata:metadata];
    NSLog(@"Model compatible: %@", isCompatible ? @"YES" : @"NO");
}
```

### Custom Migration (Heavy Migration)

```objc
// สำหรับการเปลี่ยนแปลงที่ซับซ้อน ต้องสร้าง NSMigrationManager
- (void)performHeavyMigration {
    NSURL *sourceURL = [self storeURL];
    NSURL *destinationURL = [self newStoreURL];
    
    // โหลด source model
    NSManagedObjectModel *sourceModel = [self modelForStoreAtURL:sourceURL];
    // โหลด destination model (ล่าสุด)
    NSManagedObjectModel *destModel = [NSManagedObjectModel mergedModelFromBundles:nil];
    
    // สร้าง mapping model
    NSMappingModel *mappingModel = [NSMappingModel inferredMappingModelForSourceModel:sourceModel
                                                                     destinationModel:destModel
                                                                               error:nil];
    
    // Migrate
    NSMigrationManager *manager = [[NSMigrationManager alloc] 
        initWithSourceModel:sourceModel 
        destinationModel:destModel];
    
    NSError *error;
    BOOL success = [manager migrateStoreFromURL:sourceURL
                                           type:NSSQLiteStoreType
                                        options:nil
                               withMappingModel:mappingModel
                               toDestinationURL:destinationURL
                                destinationType:NSSQLiteStoreType
                             destinationOptions:nil
                                          error:&error];
    
    if (success) {
        NSLog(@"Migration successful");
        // replace old store with new
    }
}
```

---

## 51.13 Core Data Concurrency (Parent-Child Contexts)

```objc
// Parent-Child Context Pattern
- (void)performBackgroundSave {
    // Main context (parent)
    NSManagedObjectContext *mainContext = [CoreDataStack shared].mainContext;
    
    // Child context (background)
    NSManagedObjectContext *childContext = [[NSManagedObjectContext alloc] 
        initWithConcurrencyType:NSPrivateQueueConcurrencyType];
    childContext.parentContext = mainContext;
    
    [childContext performBlock:^{
        // ทำงาน heavy ใน background
        for (NSInteger i = 0; i < 1000; i++) {
            Person *person = [[Person alloc] initWithContext:childContext];
            person.name = [NSString stringWithFormat:@"Person %ld", (long)i];
            person.age = 20 + (i % 50);
            person.createdAt = [NSDate date];
        }
        
        // Save child context (push changes to parent)
        NSError *childError;
        if ([childContext save:&childError]) {
            // Save parent context ใน main thread
            [mainContext performBlock:^{
                NSError *mainError;
                [mainContext save:&mainError];
                
                dispatch_async(dispatch_get_main_queue(), ^{
                    NSLog(@"✅ 1000 persons saved");
                });
            }];
        }
    }];
}

// ใช้ NSPersistentContainer background context
- (void)performTaskWithPersistentContainerBG {
    NSPersistentContainer *container = [CoreDataStack shared].persistentContainer;
    
    [container performBackgroundTask:^(NSManagedObjectContext *context) {
        // context นี้เป็น private queue context
        // ทำงานที่นี่...
        
        NSFetchRequest *request = [NSFetchRequest fetchRequestWithEntityName:@"Person"];
        NSError *error;
        NSArray *persons = [context executeFetchRequest:request error:&error];
        NSLog(@"Background fetch: %lu persons", (unsigned long)persons.count);
        
        // Save
        [context save:&error];
        
        // Update UI ใน main thread
        dispatch_async(dispatch_get_main_queue(), ^{
            [self.tableView reloadData];
        });
    }];
}
```

### Thread Safety Best Practices

```objc
// ❌ ผิด - อย่า pass managed objects ระหว่าง threads
- (void)wrongWay {
    NSManagedObjectContext *bgContext = [CoreDataStack shared].backgroundContext;
    __block Person *person;
    
    [bgContext performBlockAndWait:^{
        person = [[Person alloc] initWithContext:bgContext];
        person.name = @"Test";
        [bgContext save:nil];
    }];
    
    // ❌ ห้ามใช้ person ที่สร้างจาก bgContext ใน main thread!
    dispatch_async(dispatch_get_main_queue(), ^{
        NSLog(@"Person: %@", person.name);  // ❌ UNSAFE!
    });
}

// ✅ ถูก - ส่ง objectID แทน
- (void)rightWay {
    NSManagedObjectContext *bgContext = [CoreDataStack shared].backgroundContext;
    NSManagedObjectContext *mainContext = [CoreDataStack shared].mainContext;
    
    __block NSManagedObjectID *personID;
    
    [bgContext performBlock:^{
        Person *person = [[Person alloc] initWithContext:bgContext];
        person.name = @"Test";
        [bgContext save:nil];
        
        personID = person.objectID;
        
        // ส่ง objectID ไป main thread
        dispatch_async(dispatch_get_main_queue(), ^{
            // ดึง object ใน main context ด้วย objectID
            Person *mainPerson = [mainContext objectWithID:personID];
            NSLog(@"Person: %@", mainPerson.name);  // ✅ SAFE!
        });
    }];
}
```

---

## 51.14 ตัวอย่าง CRUD ครบถ้วน - Task Manager

```objc
// Task.h (NSManagedObject subclass)
@interface Task : NSManagedObject
@property (nullable, nonatomic, copy) NSString *title;
@property (nullable, nonatomic, copy) NSString *taskDescription;
@property (nonatomic) BOOL isCompleted;
@property (nullable, nonatomic, copy) NSDate *dueDate;
@property (nullable, nonatomic, copy) NSDate *createdAt;
@property (nonatomic) int16_t priority;  // 1=Low, 2=Medium, 3=High
@end

// TaskManager.h
@interface TaskManager : NSObject
+ (instancetype)shared;
- (Task *)createTaskWithTitle:(NSString *)title 
                  description:(nullable NSString *)description 
                      dueDate:(nullable NSDate *)dueDate 
                     priority:(NSInteger)priority;
- (NSArray<Task *> *)fetchAllTasks;
- (NSArray<Task *> *)fetchPendingTasks;
- (NSArray<Task *> *)fetchCompletedTasks;
- (NSArray<Task *> *)fetchHighPriorityTasks;
- (NSArray<Task *> *)searchTasksWithText:(NSString *)text;
- (void)completeTask:(Task *)task;
- (void)updateTask:(Task *)task title:(NSString *)title;
- (void)deleteTask:(Task *)task;
- (void)deleteCompletedTasks;
- (NSInteger)pendingTaskCount;
@end

// TaskManager.m
@implementation TaskManager

+ (instancetype)shared {
    static TaskManager *instance = nil;
    static dispatch_once_t onceToken;
    dispatch_once(&onceToken, ^{
        instance = [[TaskManager alloc] init];
    });
    return instance;
}

- (Task *)createTaskWithTitle:(NSString *)title 
                  description:(nullable NSString *)description 
                      dueDate:(nullable NSDate *)dueDate 
                     priority:(NSInteger)priority {
    NSManagedObjectContext *ctx = [CoreDataStack shared].mainContext;
    
    Task *task = [[Task alloc] initWithContext:ctx];
    task.title = title;
    task.taskDescription = description;
    task.dueDate = dueDate;
    task.priority = priority;
    task.isCompleted = NO;
    task.createdAt = [NSDate date];
    
    NSError *error;
    if (![ctx save:&error]) {
        NSLog(@"Error creating task: %@", error);
        [ctx rollback];
        return nil;
    }
    
    return task;
}

- (NSArray<Task *> *)fetchAllTasks {
    NSManagedObjectContext *ctx = [CoreDataStack shared].mainContext;
    NSFetchRequest<Task *> *req = [Task fetchRequest];
    req.sortDescriptors = @[
        [NSSortDescriptor sortDescriptorWithKey:@"priority" ascending:NO],
        [NSSortDescriptor sortDescriptorWithKey:@"createdAt" ascending:NO],
    ];
    
    NSError *error;
    return [ctx executeFetchRequest:req error:&error] ?: @[];
}

- (NSArray<Task *> *)fetchPendingTasks {
    NSManagedObjectContext *ctx = [CoreDataStack shared].mainContext;
    NSFetchRequest<Task *> *req = [Task fetchRequest];
    req.predicate = [NSPredicate predicateWithFormat:@"isCompleted == NO"];
    req.sortDescriptors = @[
        [NSSortDescriptor sortDescriptorWithKey:@"dueDate" ascending:YES],
        [NSSortDescriptor sortDescriptorWithKey:@"priority" ascending:NO],
    ];
    
    NSError *error;
    return [ctx executeFetchRequest:req error:&error] ?: @[];
}

- (NSArray<Task *> *)fetchCompletedTasks {
    NSManagedObjectContext *ctx = [CoreDataStack shared].mainContext;
    NSFetchRequest<Task *> *req = [Task fetchRequest];
    req.predicate = [NSPredicate predicateWithFormat:@"isCompleted == YES"];
    req.sortDescriptors = @[[NSSortDescriptor sortDescriptorWithKey:@"createdAt" ascending:NO]];
    
    NSError *error;
    return [ctx executeFetchRequest:req error:&error] ?: @[];
}

- (NSArray<Task *> *)fetchHighPriorityTasks {
    NSManagedObjectContext *ctx = [CoreDataStack shared].mainContext;
    NSFetchRequest<Task *> *req = [Task fetchRequest];
    req.predicate = [NSCompoundPredicate andPredicateWithSubpredicates:@[
        [NSPredicate predicateWithFormat:@"priority == %d", 3],
        [NSPredicate predicateWithFormat:@"isCompleted == NO"],
    ]];
    
    NSError *error;
    return [ctx executeFetchRequest:req error:&error] ?: @[];
}

- (NSArray<Task *> *)searchTasksWithText:(NSString *)text {
    if (text.length == 0) return [self fetchAllTasks];
    
    NSManagedObjectContext *ctx = [CoreDataStack shared].mainContext;
    NSFetchRequest<Task *> *req = [Task fetchRequest];
    
    NSPredicate *titlePred = [NSPredicate predicateWithFormat:@"title CONTAINS[cd] %@", text];
    NSPredicate *descPred = [NSPredicate predicateWithFormat:@"taskDescription CONTAINS[cd] %@", text];
    req.predicate = [NSCompoundPredicate orPredicateWithSubpredicates:@[titlePred, descPred]];
    req.sortDescriptors = @[[NSSortDescriptor sortDescriptorWithKey:@"priority" ascending:NO]];
    
    NSError *error;
    return [ctx executeFetchRequest:req error:&error] ?: @[];
}

- (void)completeTask:(Task *)task {
    task.isCompleted = YES;
    [[CoreDataStack shared] saveContext];
}

- (void)updateTask:(Task *)task title:(NSString *)title {
    task.title = title;
    [[CoreDataStack shared] saveContext];
}

- (void)deleteTask:(Task *)task {
    NSManagedObjectContext *ctx = task.managedObjectContext;
    [ctx deleteObject:task];
    [[CoreDataStack shared] saveContext];
}

- (void)deleteCompletedTasks {
    NSManagedObjectContext *ctx = [CoreDataStack shared].mainContext;
    NSFetchRequest *req = [NSFetchRequest fetchRequestWithEntityName:@"Task"];
    req.predicate = [NSPredicate predicateWithFormat:@"isCompleted == YES"];
    
    NSBatchDeleteRequest *batchDelete = [[NSBatchDeleteRequest alloc] initWithFetchRequest:req];
    batchDelete.resultType = NSBatchDeleteResultTypeObjectIDs;
    
    NSError *error;
    NSBatchDeleteResult *result = [ctx executeRequest:batchDelete error:&error];
    
    [NSManagedObjectContext mergeChangesFromRemoteContextSave:@{
        NSDeletedObjectsKey: result.result ?: @[]
    } intoContexts:@[ctx]];
}

- (NSInteger)pendingTaskCount {
    NSManagedObjectContext *ctx = [CoreDataStack shared].mainContext;
    NSFetchRequest *req = [NSFetchRequest fetchRequestWithEntityName:@"Task"];
    req.predicate = [NSPredicate predicateWithFormat:@"isCompleted == NO"];
    
    NSError *error;
    return [ctx countForFetchRequest:req error:&error];
}

@end
```

---

## 51.15 แบบฝึกหัด (Practice Exercises)

### แบบฝึกหัดที่ 1: Notes App

สร้าง Note entity ที่มี:
- title (String, Required)
- content (String, Optional)
- color (String, Default: "yellow")
- isPinned (Boolean, Default: NO)
- createdAt (Date)
- modifiedAt (Date)

Implement:
1. `createNote(title:content:)` - สร้าง note
2. `fetchPinnedNotes()` - ดึง pinned notes (เรียงล่าสุด)
3. `searchNotes(text:)` - ค้นหาใน title และ content
4. `pinNote:` / `unpinNote:` - toggle pin
5. `deleteNote:` - ลบ note

```objc
// ตัวอย่าง implementation
@implementation NoteManager

- (Note *)createNoteWithTitle:(NSString *)title content:(NSString *)content {
    NSManagedObjectContext *ctx = [CoreDataStack shared].mainContext;
    Note *note = [[Note alloc] initWithContext:ctx];
    note.title = title;
    note.content = content;
    note.color = @"yellow";
    note.isPinned = NO;
    note.createdAt = [NSDate date];
    note.modifiedAt = [NSDate date];
    
    NSError *error;
    [ctx save:&error];
    return note;
}

- (NSArray<Note *> *)fetchPinnedNotes {
    NSManagedObjectContext *ctx = [CoreDataStack shared].mainContext;
    NSFetchRequest<Note *> *req = [Note fetchRequest];
    req.predicate = [NSPredicate predicateWithFormat:@"isPinned == YES"];
    req.sortDescriptors = @[[NSSortDescriptor sortDescriptorWithKey:@"modifiedAt" ascending:NO]];
    
    return [ctx executeFetchRequest:req error:nil] ?: @[];
}

@end
```

### แบบฝึกหัดที่ 2: Expense Tracker

สร้าง Entity:
- Category (name, icon, color)
- Expense (amount, note, date, category)

Implement:
1. สร้าง expense พร้อม category
2. ดึงรายจ่ายในช่วงวันที่
3. คำนวณรายจ่ายรวมต่อ category
4. ลบ expense ที่เก่ากว่า 1 ปี

```objc
// Fetch expenses ใน date range
- (NSArray *)fetchExpensesFrom:(NSDate *)startDate to:(NSDate *)endDate {
    NSFetchRequest *req = [NSFetchRequest fetchRequestWithEntityName:@"Expense"];
    req.predicate = [NSPredicate predicateWithFormat:@"date >= %@ AND date <= %@", 
                     startDate, endDate];
    req.sortDescriptors = @[[NSSortDescriptor sortDescriptorWithKey:@"date" ascending:NO]];
    
    NSManagedObjectContext *ctx = [CoreDataStack shared].mainContext;
    return [ctx executeFetchRequest:req error:nil] ?: @[];
}

// คำนวณยอดรวม
- (double)totalExpenseForCategory:(NSString *)categoryName {
    NSManagedObjectContext *ctx = [CoreDataStack shared].mainContext;
    NSFetchRequest *req = [NSFetchRequest fetchRequestWithEntityName:@"Expense"];
    req.predicate = [NSPredicate predicateWithFormat:@"category.name == %@", categoryName];
    req.resultType = NSDictionaryResultType;
    
    NSExpressionDescription *sumDesc = [[NSExpressionDescription alloc] init];
    sumDesc.name = @"totalAmount";
    sumDesc.expression = [NSExpression expressionForFunction:@"sum:" 
        arguments:@[[NSExpression expressionForKeyPath:@"amount"]]];
    sumDesc.expressionResultType = NSDoubleAttributeType;
    
    req.propertiesToFetch = @[sumDesc];
    
    NSError *error;
    NSArray *results = [ctx executeFetchRequest:req error:&error];
    NSDictionary *result = results.firstObject;
    return [result[@"totalAmount"] doubleValue];
}
```

---

## สรุป (Summary)

- **Core Data Stack** ประกอบด้วย NSPersistentContainer, NSManagedObjectContext, NSPersistentStoreCoordinator, NSManagedObjectModel
- **NSPersistentContainer** (iOS 10+) คือวิธีที่แนะนำในการ setup Core Data
- **NSManagedObjectContext** คือ workspace ที่ทำงานกับ objects ก่อน save
- **NSFetchRequest** ใช้ดึงข้อมูล พร้อม predicate และ sort descriptors
- **NSPredicate** ใช้กรองข้อมูล รองรับ operators ต่างๆ
- **NSFetchedResultsController** ใช้ connect Core Data กับ table/collection view อย่างมีประสิทธิภาพ
- **Thread Safety** - ต้องใช้ context ใน thread ที่มันสร้าง, ส่ง objectID แทน object ระหว่าง threads
- **Batch Operations** (iOS 9+) ใช้ mass update/delete โดยไม่โหลด objects ทั้งหมดเข้า memory
- **Migration** - lightweight migration รองรับการเปลี่ยนแปลงง่ายๆ โดยอัตโนมัติ
- **Parent-Child Contexts** ใช้สำหรับ background operations
