# ตอนที่ 100: Capstone Project - TaskMaster Pro

## บทนำ

ยินดีต้อนรับสู่ตอนสุดท้ายของ Objective-C Course! บทนี้เป็น capstone project ที่จะรวมทุกสิ่งที่เรียนมาตลอด 100 ตอน เราจะสร้าง **TaskMaster Pro** - แอปพลิเคชัน Task Management System ที่มีฟีเจอร์ครบครัน

---

## ภาพรวมโปรเจ็กต์

### TaskMaster Pro - Features
- Task management with categories and priorities
- Core Data for persistent storage
- REST API integration (JSONPlaceholder)
- Push notifications for due tasks
- Biometric authentication (Face ID/Touch ID)
- Custom UI components
- MVVM architecture
- Comprehensive test suite
- CI/CD with GitHub Actions

### Tech Stack
```
Language:    Objective-C (primary)
Architecture: MVVM
Persistence: Core Data
Networking:  NSURLSession
Auth:        LocalAuthentication framework
UI:         UIKit + Custom Components
Testing:    XCTest, OCMock
CI/CD:      GitHub Actions + Fastlane
```

---

## ส่วนที่ 1: Project Setup

### 1.1 Xcode Project Structure

```
TaskMasterPro/
├── App/
│   ├── AppDelegate.h
│   ├── AppDelegate.m
│   ├── SceneDelegate.h
│   └── SceneDelegate.m
├── Core/
│   ├── Data/
│   │   ├── CoreData/
│   │   │   ├── TaskMasterPro.xcdatamodeld
│   │   │   ├── TaskEntity+CoreDataClass.h
│   │   │   ├── TaskEntity+CoreDataClass.m
│   │   │   ├── TaskEntity+CoreDataProperties.h
│   │   │   └── TaskEntity+CoreDataProperties.m
│   │   ├── Repositories/
│   │   │   ├── TaskRepository.h
│   │   │   ├── TaskRepository.m
│   │   │   ├── CategoryRepository.h
│   │   │   └── CategoryRepository.m
│   │   └── CoreDataManager.h
│   │   └── CoreDataManager.m
│   ├── Network/
│   │   ├── APIClient.h
│   │   ├── APIClient.m
│   │   ├── Endpoints.h
│   │   └── Models/
│   │       ├── APITask.h
│   │       └── APITask.m
│   └── Auth/
│       ├── BiometricAuthManager.h
│       └── BiometricAuthManager.m
├── Features/
│   ├── TaskList/
│   │   ├── TaskListViewController.h
│   │   ├── TaskListViewController.m
│   │   ├── TaskListViewModel.h
│   │   └── TaskListViewModel.m
│   ├── TaskDetail/
│   │   ├── TaskDetailViewController.h
│   │   ├── TaskDetailViewController.m
│   │   ├── TaskDetailViewModel.h
│   │   └── TaskDetailViewModel.m
│   ├── CreateTask/
│   │   ├── CreateTaskViewController.h
│   │   ├── CreateTaskViewController.m
│   │   └── CreateTaskViewModel.h
│   │   └── CreateTaskViewModel.m
│   └── Settings/
│       ├── SettingsViewController.h
│       └── SettingsViewController.m
├── Shared/
│   ├── Models/
│   │   ├── Task.h
│   │   ├── Task.m
│   │   ├── Category.h
│   │   └── Category.m
│   ├── UI/
│   │   ├── Components/
│   │   │   ├── TaskCell.h
│   │   │   ├── TaskCell.m
│   │   │   ├── PriorityBadge.h
│   │   │   ├── PriorityBadge.m
│   │   │   ├── CategoryPicker.h
│   │   │   └── CategoryPicker.m
│   │   └── Theme/
│   │       ├── AppTheme.h
│   │       └── AppTheme.m
│   └── Utilities/
│       ├── DateUtils.h
│       ├── DateUtils.m
│       ├── ValidationUtils.h
│       └── ValidationUtils.m
├── Resources/
│   ├── Assets.xcassets
│   ├── LaunchScreen.storyboard
│   └── Localizable.strings
├── Tests/
│   ├── Unit/
│   │   ├── TaskTests.m
│   │   ├── TaskRepositoryTests.m
│   │   └── TaskListViewModelTests.m
│   └── UI/
│       └── TaskListUITests.m
└── Supporting Files/
    ├── Info.plist
    └── TaskMasterPro-Prefix.pch
```

---

### 1.2 AppDelegate

```objc
// AppDelegate.h
#import <UIKit/UIKit.h>

@interface AppDelegate : UIResponder <UIApplicationDelegate>

@property (strong, nonatomic) UIWindow *window;

@end

// AppDelegate.m
#import "AppDelegate.h"
#import "CoreDataManager.h"
#import "BiometricAuthManager.h"
#import "AppTheme.h"

@implementation AppDelegate

- (BOOL)application:(UIApplication *)application 
    didFinishLaunchingWithOptions:(NSDictionary *)launchOptions {
    
    // Setup theme
    [AppTheme setupAppearance];
    
    // Setup Core Data
    [[CoreDataManager sharedManager] setup];
    
    // Setup push notifications
    [self requestPushNotificationPermission:application];
    
    return YES;
}

- (void)requestPushNotificationPermission:(UIApplication *)application {
    UNUserNotificationCenter *center = [UNUserNotificationCenter currentNotificationCenter];
    center.delegate = self;
    
    [center requestAuthorizationWithOptions:
        UNAuthorizationOptionAlert | UNAuthorizationOptionBadge | UNAuthorizationOptionSound
     completionHandler:^(BOOL granted, NSError *error) {
        if (granted) {
            dispatch_async(dispatch_get_main_queue(), ^{
                [application registerForRemoteNotifications];
            });
        }
        NSLog(@"Push notification permission: %@", granted ? @"granted" : @"denied");
    }];
}

- (void)application:(UIApplication *)application 
    didRegisterForRemoteNotificationsWithDeviceToken:(NSData *)deviceToken {
    NSString *token = [self tokenStringFromData:deviceToken];
    NSLog(@"Device token: %@", token);
    // Send token to server
    [[APIClient sharedClient] registerDeviceToken:token];
}

- (NSString *)tokenStringFromData:(NSData *)data {
    const unsigned char *bytes = (const unsigned char *)data.bytes;
    NSMutableString *token = [NSMutableString string];
    for (NSUInteger i = 0; i < data.length; i++) {
        [token appendFormat:@"%02x", bytes[i]];
    }
    return [token copy];
}

// Core Data save on background
- (void)applicationDidEnterBackground:(UIApplication *)application {
    [[CoreDataManager sharedManager] saveContext];
}

- (void)applicationWillTerminate:(UIApplication *)application {
    [[CoreDataManager sharedManager] saveContext];
}

@end
```

---

## ส่วนที่ 2: Data Models

### 2.1 Task Model

```objc
// Task.h
#import <Foundation/Foundation.h>

typedef NS_ENUM(NSInteger, TaskPriority) {
    TaskPriorityLow = 0,
    TaskPriorityMedium = 1,
    TaskPriorityHigh = 2,
    TaskPriorityCritical = 3
};

typedef NS_ENUM(NSInteger, TaskStatus) {
    TaskStatusTodo = 0,
    TaskStatusInProgress = 1,
    TaskStatusDone = 2,
    TaskStatusCancelled = 3
};

@interface Task : NSObject <NSCopying>

@property (nonatomic, strong, readonly) NSString *taskId;
@property (nonatomic, strong) NSString *title;
@property (nonatomic, strong) NSString *taskDescription;
@property (nonatomic, assign) TaskPriority priority;
@property (nonatomic, assign) TaskStatus status;
@property (nonatomic, strong) NSDate *dueDate;
@property (nonatomic, strong) NSDate *createdAt;
@property (nonatomic, strong) NSDate *updatedAt;
@property (nonatomic, strong) NSString *categoryId;
@property (nonatomic, strong) NSArray<NSString *> *tags;
@property (nonatomic, assign) BOOL hasReminder;
@property (nonatomic, strong) NSDate *reminderDate;

// Computed properties
@property (nonatomic, readonly) BOOL isOverdue;
@property (nonatomic, readonly) BOOL isCompleted;
@property (nonatomic, readonly) NSString *priorityDisplayName;
@property (nonatomic, readonly) NSString *statusDisplayName;
@property (nonatomic, readonly) NSString *dueDateDisplayString;

+ (instancetype)taskWithTitle:(NSString *)title;
- (instancetype)initWithTitle:(NSString *)title 
                     priority:(TaskPriority)priority;
- (NSDictionary *)toDictionary;
+ (instancetype)fromDictionary:(NSDictionary *)dict;
- (BOOL)isValidForSaving:(NSError **)error;

@end

// Task.m
#import "Task.h"
#import "DateUtils.h"

@implementation Task

+ (instancetype)taskWithTitle:(NSString *)title {
    return [[self alloc] initWithTitle:title priority:TaskPriorityMedium];
}

- (instancetype)initWithTitle:(NSString *)title priority:(TaskPriority)priority {
    self = [super init];
    if (self) {
        _taskId = [[NSUUID UUID] UUIDString];
        _title = title;
        _priority = priority;
        _status = TaskStatusTodo;
        _createdAt = [NSDate date];
        _updatedAt = [NSDate date];
        _tags = @[];
    }
    return self;
}

- (BOOL)isOverdue {
    if (!self.dueDate) return NO;
    if (self.status == TaskStatusDone || self.status == TaskStatusCancelled) return NO;
    return [self.dueDate compare:[NSDate date]] == NSOrderedAscending;
}

- (BOOL)isCompleted {
    return self.status == TaskStatusDone;
}

- (NSString *)priorityDisplayName {
    switch (self.priority) {
        case TaskPriorityLow: return @"ต่ำ";
        case TaskPriorityMedium: return @"ปานกลาง";
        case TaskPriorityHigh: return @"สูง";
        case TaskPriorityCritical: return @"วิกฤต";
    }
}

- (NSString *)statusDisplayName {
    switch (self.status) {
        case TaskStatusTodo: return @"รอดำเนินการ";
        case TaskStatusInProgress: return @"กำลังดำเนินการ";
        case TaskStatusDone: return @"เสร็จแล้ว";
        case TaskStatusCancelled: return @"ยกเลิก";
    }
}

- (NSString *)dueDateDisplayString {
    if (!self.dueDate) return @"ไม่มีกำหนด";
    return [DateUtils relativeDateString:self.dueDate];
}

- (NSDictionary *)toDictionary {
    NSMutableDictionary *dict = [NSMutableDictionary dictionary];
    dict[@"id"] = self.taskId;
    dict[@"title"] = self.title ?: [NSNull null];
    dict[@"description"] = self.taskDescription ?: [NSNull null];
    dict[@"priority"] = @(self.priority);
    dict[@"status"] = @(self.status);
    if (self.dueDate) dict[@"dueDate"] = @(self.dueDate.timeIntervalSince1970);
    dict[@"createdAt"] = @(self.createdAt.timeIntervalSince1970);
    dict[@"tags"] = self.tags ?: @[];
    return [dict copy];
}

+ (instancetype)fromDictionary:(NSDictionary *)dict {
    Task *task = [[Task alloc] init];
    task->_taskId = dict[@"id"] ?: [[NSUUID UUID] UUIDString];
    task->_title = dict[@"title"];
    task->_taskDescription = dict[@"description"];
    task->_priority = [dict[@"priority"] integerValue];
    task->_status = [dict[@"status"] integerValue];
    
    if (dict[@"dueDate"] && ![dict[@"dueDate"] isKindOfClass:[NSNull class]]) {
        task->_dueDate = [NSDate dateWithTimeIntervalSince1970:[dict[@"dueDate"] doubleValue]];
    }
    if (dict[@"createdAt"]) {
        task->_createdAt = [NSDate dateWithTimeIntervalSince1970:[dict[@"createdAt"] doubleValue]];
    }
    task->_tags = dict[@"tags"] ?: @[];
    return task;
}

- (BOOL)isValidForSaving:(NSError **)error {
    if (self.title.length == 0) {
        if (error) {
            *error = [NSError errorWithDomain:@"TaskValidation" code:1
                                     userInfo:@{NSLocalizedDescriptionKey: @"ชื่อ Task ไม่ควรว่างเปล่า"}];
        }
        return NO;
    }
    if (self.title.length > 200) {
        if (error) {
            *error = [NSError errorWithDomain:@"TaskValidation" code:2
                                     userInfo:@{NSLocalizedDescriptionKey: @"ชื่อ Task ยาวเกินไป (max 200 ตัวอักษร)"}];
        }
        return NO;
    }
    return YES;
}

- (id)copyWithZone:(NSZone *)zone {
    Task *copy = [[Task allocWithZone:zone] init];
    copy->_taskId = _taskId;
    copy->_title = [_title copy];
    copy->_taskDescription = [_taskDescription copy];
    copy->_priority = _priority;
    copy->_status = _status;
    copy->_dueDate = [_dueDate copy];
    copy->_createdAt = [_createdAt copy];
    copy->_updatedAt = [_updatedAt copy];
    copy->_categoryId = [_categoryId copy];
    copy->_tags = [_tags copy];
    copy->_hasReminder = _hasReminder;
    copy->_reminderDate = [_reminderDate copy];
    return copy;
}

- (NSString *)description {
    return [NSString stringWithFormat:@"Task{id=%@, title=%@, priority=%@, status=%@}",
            self.taskId, self.title, self.priorityDisplayName, self.statusDisplayName];
}

@end
```

---

## ส่วนที่ 3: Core Data Stack

### 3.1 CoreDataManager

```objc
// CoreDataManager.h
#import <CoreData/CoreData.h>

@interface CoreDataManager : NSObject

+ (instancetype)sharedManager;

@property (readonly, strong) NSPersistentContainer *persistentContainer;
@property (readonly, strong) NSManagedObjectContext *mainContext;

- (void)setup;
- (void)saveContext;
- (void)performInBackground:(void (^)(NSManagedObjectContext *context))block;

@end

// CoreDataManager.m
#import "CoreDataManager.h"

@implementation CoreDataManager

+ (instancetype)sharedManager {
    static CoreDataManager *manager = nil;
    static dispatch_once_t onceToken;
    dispatch_once(&onceToken, ^{
        manager = [[self alloc] init];
    });
    return manager;
}

- (void)setup {
    // Pre-warm the container
    [self persistentContainer];
}

- (NSPersistentContainer *)persistentContainer {
    static NSPersistentContainer *container = nil;
    static dispatch_once_t onceToken;
    dispatch_once(&onceToken, ^{
        container = [[NSPersistentContainer alloc] initWithName:@"TaskMasterPro"];
        
        NSPersistentStoreDescription *description = container.persistentStoreDescriptions.firstObject;
        description.shouldMigrateStoreAutomatically = YES;
        description.shouldInferMappingModelAutomatically = YES;
        
        [container loadPersistentStoresWithCompletionHandler:
            ^(NSPersistentStoreDescription *desc, NSError *error) {
            if (error) {
                NSLog(@"Core Data error: %@", error);
                // In production: report to crash analytics
                // NOT fatalError - handle gracefully
            } else {
                NSLog(@"Core Data loaded successfully");
            }
        }];
        
        container.viewContext.automaticallyMergesChangesFromParent = YES;
        container.viewContext.mergePolicy = NSMergeByPropertyObjectTrumpMergePolicy;
    });
    return container;
}

- (NSManagedObjectContext *)mainContext {
    return self.persistentContainer.viewContext;
}

- (void)saveContext {
    NSManagedObjectContext *context = self.mainContext;
    [context performBlock:^{
        if (context.hasChanges) {
            NSError *error = nil;
            if (![context save:&error]) {
                NSLog(@"Save error: %@", error);
            }
        }
    }];
}

- (void)performInBackground:(void (^)(NSManagedObjectContext *))block {
    [self.persistentContainer performBackgroundTask:block];
}

@end
```

---

### 3.2 Core Data Entity (TaskEntity)

```objc
// TaskEntity+CoreDataProperties.h (generated by Xcode)
#import "TaskEntity+CoreDataClass.h"

NS_ASSUME_NONNULL_BEGIN

@interface TaskEntity (CoreDataProperties)

+ (NSFetchRequest<TaskEntity *> *)fetchRequest;

@property (nullable, nonatomic, copy) NSString *taskId;
@property (nullable, nonatomic, copy) NSString *title;
@property (nullable, nonatomic, copy) NSString *taskDescription;
@property (nonatomic) int16_t priority;
@property (nonatomic) int16_t status;
@property (nullable, nonatomic, copy) NSDate *dueDate;
@property (nullable, nonatomic, copy) NSDate *createdAt;
@property (nullable, nonatomic, copy) NSDate *updatedAt;
@property (nullable, nonatomic, copy) NSString *categoryId;
@property (nonatomic) BOOL hasReminder;
@property (nullable, nonatomic, copy) NSDate *reminderDate;

// Transformable
@property (nullable, nonatomic, retain) NSArray *tags;

@end

NS_ASSUME_NONNULL_END
```

---

### 3.3 TaskRepository

```objc
// TaskRepository.h
#import <Foundation/Foundation.h>
#import "Task.h"

typedef void (^TaskCompletion)(Task * _Nullable task, NSError * _Nullable error);
typedef void (^TasksCompletion)(NSArray<Task *> * _Nonnull tasks, NSError * _Nullable error);
typedef void (^BoolCompletion)(BOOL success, NSError * _Nullable error);

@interface TaskRepository : NSObject

+ (instancetype)sharedRepository;

- (void)createTask:(Task *)task completion:(TaskCompletion)completion;
- (void)updateTask:(Task *)task completion:(BoolCompletion)completion;
- (void)deleteTask:(Task *)task completion:(BoolCompletion)completion;
- (void)fetchTaskById:(NSString *)taskId completion:(TaskCompletion)completion;
- (void)fetchAllTasksWithCompletion:(TasksCompletion)completion;
- (void)fetchTasksWithStatus:(TaskStatus)status completion:(TasksCompletion)completion;
- (void)fetchTasksForCategory:(NSString *)categoryId completion:(TasksCompletion)completion;
- (void)searchTasks:(NSString *)query completion:(TasksCompletion)completion;
- (void)fetchOverdueTasksWithCompletion:(TasksCompletion)completion;

@end

// TaskRepository.m
#import "TaskRepository.h"
#import "CoreDataManager.h"
#import "TaskEntity+CoreDataClass.h"
#import "TaskEntity+CoreDataProperties.h"
#import <CoreData/CoreData.h>

@implementation TaskRepository

+ (instancetype)sharedRepository {
    static TaskRepository *repo = nil;
    static dispatch_once_t onceToken;
    dispatch_once(&onceToken, ^{
        repo = [[self alloc] init];
    });
    return repo;
}

- (void)createTask:(Task *)task completion:(TaskCompletion)completion {
    NSError *validationError = nil;
    if (![task isValidForSaving:&validationError]) {
        if (completion) dispatch_async(dispatch_get_main_queue(), ^{
            completion(nil, validationError);
        });
        return;
    }
    
    [[CoreDataManager sharedManager] performInBackground:^(NSManagedObjectContext *context) {
        TaskEntity *entity = [NSEntityDescription insertNewObjectForEntityForName:@"TaskEntity"
                                                          inManagedObjectContext:context];
        [self populateEntity:entity fromTask:task];
        
        NSError *saveError = nil;
        if ([context save:&saveError]) {
            dispatch_async(dispatch_get_main_queue(), ^{
                if (completion) completion(task, nil);
            });
        } else {
            dispatch_async(dispatch_get_main_queue(), ^{
                if (completion) completion(nil, saveError);
            });
        }
    }];
}

- (void)updateTask:(Task *)task completion:(BoolCompletion)completion {
    [[CoreDataManager sharedManager] performInBackground:^(NSManagedObjectContext *context) {
        NSFetchRequest *request = [NSFetchRequest fetchRequestWithEntityName:@"TaskEntity"];
        request.predicate = [NSPredicate predicateWithFormat:@"taskId == %@", task.taskId];
        
        NSError *fetchError = nil;
        NSArray *results = [context executeFetchRequest:request error:&fetchError];
        
        if (fetchError || results.count == 0) {
            dispatch_async(dispatch_get_main_queue(), ^{
                if (completion) completion(NO, fetchError);
            });
            return;
        }
        
        TaskEntity *entity = results.firstObject;
        [self populateEntity:entity fromTask:task];
        entity.updatedAt = [NSDate date];
        
        NSError *saveError = nil;
        BOOL saved = [context save:&saveError];
        
        dispatch_async(dispatch_get_main_queue(), ^{
            if (completion) completion(saved, saveError);
        });
    }];
}

- (void)deleteTask:(Task *)task completion:(BoolCompletion)completion {
    [[CoreDataManager sharedManager] performInBackground:^(NSManagedObjectContext *context) {
        NSFetchRequest *request = [NSFetchRequest fetchRequestWithEntityName:@"TaskEntity"];
        request.predicate = [NSPredicate predicateWithFormat:@"taskId == %@", task.taskId];
        
        NSError *error = nil;
        NSArray *results = [context executeFetchRequest:request error:&error];
        
        if (results.count > 0) {
            [context deleteObject:results.firstObject];
            BOOL saved = [context save:&error];
            dispatch_async(dispatch_get_main_queue(), ^{
                if (completion) completion(saved, error);
            });
        } else {
            dispatch_async(dispatch_get_main_queue(), ^{
                if (completion) completion(NO, error);
            });
        }
    }];
}

- (void)fetchAllTasksWithCompletion:(TasksCompletion)completion {
    NSManagedObjectContext *context = [CoreDataManager sharedManager].mainContext;
    
    [context performBlock:^{
        NSFetchRequest *request = [NSFetchRequest fetchRequestWithEntityName:@"TaskEntity"];
        request.sortDescriptors = @[
            [NSSortDescriptor sortDescriptorWithKey:@"priority" ascending:NO],
            [NSSortDescriptor sortDescriptorWithKey:@"createdAt" ascending:NO]
        ];
        
        NSError *error = nil;
        NSArray *entities = [context executeFetchRequest:request error:&error];
        
        NSArray<Task *> *tasks = [self tasksFromEntities:entities];
        
        dispatch_async(dispatch_get_main_queue(), ^{
            if (completion) completion(tasks, error);
        });
    }];
}

- (void)fetchTasksWithStatus:(TaskStatus)status completion:(TasksCompletion)completion {
    NSManagedObjectContext *context = [CoreDataManager sharedManager].mainContext;
    
    [context performBlock:^{
        NSFetchRequest *request = [NSFetchRequest fetchRequestWithEntityName:@"TaskEntity"];
        request.predicate = [NSPredicate predicateWithFormat:@"status == %d", (int)status];
        request.sortDescriptors = @[
            [NSSortDescriptor sortDescriptorWithKey:@"dueDate" ascending:YES],
            [NSSortDescriptor sortDescriptorWithKey:@"priority" ascending:NO]
        ];
        
        NSError *error = nil;
        NSArray *entities = [context executeFetchRequest:request error:&error];
        NSArray<Task *> *tasks = [self tasksFromEntities:entities];
        
        dispatch_async(dispatch_get_main_queue(), ^{
            if (completion) completion(tasks, error);
        });
    }];
}

- (void)searchTasks:(NSString *)query completion:(TasksCompletion)completion {
    NSManagedObjectContext *context = [CoreDataManager sharedManager].mainContext;
    
    [context performBlock:^{
        NSFetchRequest *request = [NSFetchRequest fetchRequestWithEntityName:@"TaskEntity"];
        request.predicate = [NSPredicate predicateWithFormat:
                             @"title CONTAINS[cd] %@ OR taskDescription CONTAINS[cd] %@",
                             query, query];
        request.sortDescriptors = @[
            [NSSortDescriptor sortDescriptorWithKey:@"createdAt" ascending:NO]
        ];
        
        NSError *error = nil;
        NSArray *entities = [context executeFetchRequest:request error:&error];
        NSArray<Task *> *tasks = [self tasksFromEntities:entities];
        
        dispatch_async(dispatch_get_main_queue(), ^{
            if (completion) completion(tasks, error);
        });
    }];
}

- (void)fetchOverdueTasksWithCompletion:(TasksCompletion)completion {
    NSManagedObjectContext *context = [CoreDataManager sharedManager].mainContext;
    
    [context performBlock:^{
        NSFetchRequest *request = [NSFetchRequest fetchRequestWithEntityName:@"TaskEntity"];
        request.predicate = [NSPredicate predicateWithFormat:
                             @"dueDate < %@ AND status != %d AND status != %d",
                             [NSDate date], (int)TaskStatusDone, (int)TaskStatusCancelled];
        
        NSError *error = nil;
        NSArray *entities = [context executeFetchRequest:request error:&error];
        NSArray<Task *> *tasks = [self tasksFromEntities:entities];
        
        dispatch_async(dispatch_get_main_queue(), ^{
            if (completion) completion(tasks, error);
        });
    }];
}

#pragma mark - Private helpers

- (void)populateEntity:(TaskEntity *)entity fromTask:(Task *)task {
    entity.taskId = task.taskId;
    entity.title = task.title;
    entity.taskDescription = task.taskDescription;
    entity.priority = (int16_t)task.priority;
    entity.status = (int16_t)task.status;
    entity.dueDate = task.dueDate;
    entity.createdAt = task.createdAt ?: [NSDate date];
    entity.updatedAt = [NSDate date];
    entity.categoryId = task.categoryId;
    entity.tags = task.tags;
    entity.hasReminder = task.hasReminder;
    entity.reminderDate = task.reminderDate;
}

- (Task *)taskFromEntity:(TaskEntity *)entity {
    Task *task = [[Task alloc] init];
    task->_taskId = entity.taskId;
    task->_title = entity.title;
    task->_taskDescription = entity.taskDescription;
    task->_priority = (TaskPriority)entity.priority;
    task->_status = (TaskStatus)entity.status;
    task->_dueDate = entity.dueDate;
    task->_createdAt = entity.createdAt;
    task->_updatedAt = entity.updatedAt;
    task->_categoryId = entity.categoryId;
    task->_tags = entity.tags ?: @[];
    task->_hasReminder = entity.hasReminder;
    task->_reminderDate = entity.reminderDate;
    return task;
}

- (NSArray<Task *> *)tasksFromEntities:(NSArray *)entities {
    NSMutableArray<Task *> *tasks = [NSMutableArray arrayWithCapacity:entities.count];
    for (TaskEntity *entity in entities) {
        [tasks addObject:[self taskFromEntity:entity]];
    }
    return [tasks copy];
}

@end
```

---

## ส่วนที่ 4: Network Layer

### 4.1 APIClient

```objc
// APIClient.h
#import <Foundation/Foundation.h>

typedef void (^APICompletion)(id _Nullable responseObject, NSError * _Nullable error);
typedef void (^APITasksCompletion)(NSArray * _Nullable tasks, NSError * _Nullable error);

@interface APIClient : NSObject

+ (instancetype)sharedClient;

// Task API
- (void)fetchTasksWithPage:(NSInteger)page 
               completion:(APITasksCompletion)completion;
- (void)createTask:(NSDictionary *)taskData 
        completion:(APICompletion)completion;
- (void)updateTask:(NSString *)taskId 
              data:(NSDictionary *)data 
        completion:(APICompletion)completion;
- (void)deleteTask:(NSString *)taskId 
        completion:(APICompletion)completion;
- (void)syncTasks:(NSArray<NSDictionary *> *)tasks 
       completion:(APICompletion)completion;

// Device
- (void)registerDeviceToken:(NSString *)token;

@end

// APIClient.m
#import "APIClient.h"

static NSString * const kBaseURL = @"https://jsonplaceholder.typicode.com";
static NSString * const kAPIVersion = @"v1";
static NSTimeInterval const kRequestTimeout = 30.0;

@implementation APIClient {
    NSURLSession *_session;
    NSOperationQueue *_operationQueue;
    NSMutableDictionary *_requestHeaders;
}

+ (instancetype)sharedClient {
    static APIClient *client = nil;
    static dispatch_once_t onceToken;
    dispatch_once(&onceToken, ^{
        client = [[self alloc] init];
    });
    return client;
}

- (instancetype)init {
    self = [super init];
    if (self) {
        NSURLSessionConfiguration *config = [NSURLSessionConfiguration defaultSessionConfiguration];
        config.timeoutIntervalForRequest = kRequestTimeout;
        config.timeoutIntervalForResource = 120.0;
        config.waitsForConnectivity = YES;
        
        _operationQueue = [[NSOperationQueue alloc] init];
        _operationQueue.maxConcurrentOperationCount = 4;
        
        _session = [NSURLSession sessionWithConfiguration:config
                                                 delegate:self
                                            delegateQueue:_operationQueue];
        
        _requestHeaders = [@{
            @"Content-Type": @"application/json",
            @"Accept": @"application/json",
            @"X-API-Version": kAPIVersion
        } mutableCopy];
    }
    return self;
}

- (void)fetchTasksWithPage:(NSInteger)page completion:(APITasksCompletion)completion {
    NSString *path = [NSString stringWithFormat:@"/todos?_page=%ld&_limit=20", (long)page];
    
    [self GET:path completion:^(id response, NSError *error) {
        if (error) {
            if (completion) completion(nil, error);
            return;
        }
        
        NSArray *rawTasks = [response isKindOfClass:[NSArray class]] ? response : @[];
        NSMutableArray *mappedTasks = [NSMutableArray array];
        
        for (NSDictionary *raw in rawTasks) {
            NSMutableDictionary *mapped = [NSMutableDictionary dictionary];
            mapped[@"id"] = [raw[@"id"] stringValue];
            mapped[@"title"] = raw[@"title"] ?: @"Untitled";
            mapped[@"status"] = [raw[@"completed"] boolValue] ? @(TaskStatusDone) : @(TaskStatusTodo);
            mapped[@"priority"] = @(arc4random_uniform(4)); // random for demo
            mapped[@"createdAt"] = @([[NSDate date] timeIntervalSince1970]);
            [mappedTasks addObject:[mapped copy]];
        }
        
        if (completion) completion([mappedTasks copy], nil);
    }];
}

- (void)createTask:(NSDictionary *)taskData completion:(APICompletion)completion {
    [self POST:@"/todos" body:taskData completion:completion];
}

- (void)updateTask:(NSString *)taskId data:(NSDictionary *)data completion:(APICompletion)completion {
    NSString *path = [NSString stringWithFormat:@"/todos/%@", taskId];
    [self PUT:path body:data completion:completion];
}

- (void)deleteTask:(NSString *)taskId completion:(APICompletion)completion {
    NSString *path = [NSString stringWithFormat:@"/todos/%@", taskId];
    [self DELETE:path completion:completion];
}

- (void)registerDeviceToken:(NSString *)token {
    NSDictionary *body = @{@"token": token, @"platform": @"ios"};
    [self POST:@"/devices" body:body completion:^(id response, NSError *error) {
        if (error) NSLog(@"Device registration failed: %@", error);
        else NSLog(@"Device registered successfully");
    }];
}

#pragma mark - HTTP Methods

- (void)GET:(NSString *)path completion:(APICompletion)completion {
    NSURL *url = [NSURL URLWithString:[kBaseURL stringByAppendingString:path]];
    NSMutableURLRequest *request = [NSMutableURLRequest requestWithURL:url];
    request.HTTPMethod = @"GET";
    [self applyHeaders:request];
    [self executeRequest:request completion:completion];
}

- (void)POST:(NSString *)path body:(NSDictionary *)body completion:(APICompletion)completion {
    NSURL *url = [NSURL URLWithString:[kBaseURL stringByAppendingString:path]];
    NSMutableURLRequest *request = [NSMutableURLRequest requestWithURL:url];
    request.HTTPMethod = @"POST";
    [self applyHeaders:request];
    
    NSError *jsonError = nil;
    request.HTTPBody = [NSJSONSerialization dataWithJSONObject:body options:0 error:&jsonError];
    
    if (jsonError) {
        if (completion) completion(nil, jsonError);
        return;
    }
    
    [self executeRequest:request completion:completion];
}

- (void)PUT:(NSString *)path body:(NSDictionary *)body completion:(APICompletion)completion {
    NSURL *url = [NSURL URLWithString:[kBaseURL stringByAppendingString:path]];
    NSMutableURLRequest *request = [NSMutableURLRequest requestWithURL:url];
    request.HTTPMethod = @"PUT";
    [self applyHeaders:request];
    request.HTTPBody = [NSJSONSerialization dataWithJSONObject:body options:0 error:nil];
    [self executeRequest:request completion:completion];
}

- (void)DELETE:(NSString *)path completion:(APICompletion)completion {
    NSURL *url = [NSURL URLWithString:[kBaseURL stringByAppendingString:path]];
    NSMutableURLRequest *request = [NSMutableURLRequest requestWithURL:url];
    request.HTTPMethod = @"DELETE";
    [self applyHeaders:request];
    [self executeRequest:request completion:completion];
}

- (void)applyHeaders:(NSMutableURLRequest *)request {
    [_requestHeaders enumerateKeysAndObjectsUsingBlock:^(NSString *key, NSString *value, BOOL *stop) {
        [request setValue:value forHTTPHeaderField:key];
    }];
    
    // Add auth token if available
    NSString *token = [[NSUserDefaults standardUserDefaults] stringForKey:@"authToken"];
    if (token) {
        [request setValue:[NSString stringWithFormat:@"Bearer %@", token]
       forHTTPHeaderField:@"Authorization"];
    }
}

- (void)executeRequest:(NSURLRequest *)request completion:(APICompletion)completion {
    NSURLSessionDataTask *task = [_session dataTaskWithRequest:request
                                            completionHandler:^(NSData *data, 
                                                               NSURLResponse *response, 
                                                               NSError *error) {
        if (error) {
            dispatch_async(dispatch_get_main_queue(), ^{
                if (completion) completion(nil, error);
            });
            return;
        }
        
        NSHTTPURLResponse *httpResponse = (NSHTTPURLResponse *)response;
        
        if (httpResponse.statusCode < 200 || httpResponse.statusCode >= 300) {
            NSError *statusError = [NSError errorWithDomain:@"APIClient"
                                                       code:httpResponse.statusCode
                                                   userInfo:@{
                NSLocalizedDescriptionKey: [NSHTTPURLResponse localizedStringForStatusCode:httpResponse.statusCode]
            }];
            dispatch_async(dispatch_get_main_queue(), ^{
                if (completion) completion(nil, statusError);
            });
            return;
        }
        
        if (!data || data.length == 0) {
            dispatch_async(dispatch_get_main_queue(), ^{
                if (completion) completion(@{}, nil);
            });
            return;
        }
        
        NSError *jsonError = nil;
        id parsed = [NSJSONSerialization JSONObjectWithData:data options:0 error:&jsonError];
        
        dispatch_async(dispatch_get_main_queue(), ^{
            if (completion) completion(parsed, jsonError);
        });
    }];
    
    [task resume];
}

// NSURLSessionDelegate - Certificate Pinning
- (void)URLSession:(NSURLSession *)session 
    didReceiveChallenge:(NSURLAuthenticationChallenge *)challenge
     completionHandler:(void (^)(NSURLSessionAuthChallengeDisposition, NSURLCredential *))completionHandler {
    
    // For demo - accept any certificate
    // In production: implement certificate pinning
    completionHandler(NSURLSessionAuthChallengePerformDefaultHandling, nil);
}

@end
```

---

## ส่วนที่ 5: MVVM - ViewModels

### 5.1 TaskListViewModel

```objc
// TaskListViewModel.h
#import <Foundation/Foundation.h>
#import "Task.h"

typedef NS_ENUM(NSInteger, TaskListFilter) {
    TaskListFilterAll = 0,
    TaskListFilterTodo,
    TaskListFilterInProgress,
    TaskListFilterDone,
    TaskListFilterOverdue
};

@protocol TaskListViewModelDelegate <NSObject>
- (void)viewModelDidUpdateTasks;
- (void)viewModelDidFailWithError:(NSError *)error;
- (void)viewModelDidStartLoading;
- (void)viewModelDidFinishLoading;
@end

@interface TaskListViewModel : NSObject

@property (nonatomic, weak) id<TaskListViewModelDelegate> delegate;
@property (nonatomic, readonly) NSArray<Task *> *tasks;
@property (nonatomic, readonly) NSInteger totalCount;
@property (nonatomic, readonly) NSInteger completedCount;
@property (nonatomic, readonly) NSInteger overdueCount;
@property (nonatomic, readonly) BOOL isLoading;
@property (nonatomic, assign) TaskListFilter currentFilter;
@property (nonatomic, strong) NSString *searchQuery;

- (void)loadTasks;
- (void)refreshFromAPI;
- (void)deleteTask:(Task *)task;
- (void)toggleTaskCompletion:(Task *)task;
- (Task *)taskAtIndex:(NSUInteger)index;
- (NSString *)summaryText;

@end

// TaskListViewModel.m
#import "TaskListViewModel.h"
#import "TaskRepository.h"
#import "APIClient.h"

@implementation TaskListViewModel {
    NSArray<Task *> *_allTasks;
    NSArray<Task *> *_filteredTasks;
}

- (instancetype)init {
    self = [super init];
    if (self) {
        _currentFilter = TaskListFilterAll;
        _searchQuery = @"";
    }
    return self;
}

- (NSArray<Task *> *)tasks {
    return _filteredTasks ?: @[];
}

- (NSInteger)totalCount {
    return _allTasks.count;
}

- (NSInteger)completedCount {
    NSInteger count = 0;
    for (Task *task in _allTasks) {
        if (task.isCompleted) count++;
    }
    return count;
}

- (NSInteger)overdueCount {
    NSInteger count = 0;
    for (Task *task in _allTasks) {
        if (task.isOverdue) count++;
    }
    return count;
}

- (void)loadTasks {
    _isLoading = YES;
    [self.delegate viewModelDidStartLoading];
    
    [[TaskRepository sharedRepository] fetchAllTasksWithCompletion:^(NSArray<Task *> *tasks, NSError *error) {
        self->_isLoading = NO;
        [self.delegate viewModelDidFinishLoading];
        
        if (error) {
            [self.delegate viewModelDidFailWithError:error];
            return;
        }
        
        self->_allTasks = tasks;
        [self applyFiltersAndSearch];
        [self.delegate viewModelDidUpdateTasks];
    }];
}

- (void)refreshFromAPI {
    _isLoading = YES;
    [self.delegate viewModelDidStartLoading];
    
    [[APIClient sharedClient] fetchTasksWithPage:1 completion:^(NSArray *apiTasks, NSError *error) {
        self->_isLoading = NO;
        [self.delegate viewModelDidFinishLoading];
        
        if (error) {
            [self.delegate viewModelDidFailWithError:error];
            return;
        }
        
        // Save to Core Data
        dispatch_group_t group = dispatch_group_create();
        
        for (NSDictionary *taskData in apiTasks) {
            dispatch_group_enter(group);
            
            Task *task = [Task fromDictionary:taskData];
            [[TaskRepository sharedRepository] createTask:task completion:^(Task *saved, NSError *saveError) {
                dispatch_group_leave(group);
            }];
        }
        
        dispatch_group_notify(group, dispatch_get_main_queue(), ^{
            [self loadTasks]; // reload after sync
        });
    }];
}

- (void)deleteTask:(Task *)task {
    [[TaskRepository sharedRepository] deleteTask:task completion:^(BOOL success, NSError *error) {
        if (success) {
            NSMutableArray *mutable = [self->_allTasks mutableCopy];
            [mutable removeObject:task];
            self->_allTasks = [mutable copy];
            [self applyFiltersAndSearch];
            [self.delegate viewModelDidUpdateTasks];
        } else {
            [self.delegate viewModelDidFailWithError:error];
        }
    }];
}

- (void)toggleTaskCompletion:(Task *)task {
    Task *updatedTask = [task copy];
    updatedTask.status = task.isCompleted ? TaskStatusTodo : TaskStatusDone;
    updatedTask.updatedAt = [NSDate date];
    
    [[TaskRepository sharedRepository] updateTask:updatedTask completion:^(BOOL success, NSError *error) {
        if (success) {
            NSMutableArray *mutable = [self->_allTasks mutableCopy];
            NSInteger idx = [mutable indexOfObject:task];
            if (idx != NSNotFound) {
                mutable[idx] = updatedTask;
            }
            self->_allTasks = [mutable copy];
            [self applyFiltersAndSearch];
            [self.delegate viewModelDidUpdateTasks];
        } else {
            [self.delegate viewModelDidFailWithError:error];
        }
    }];
}

- (Task *)taskAtIndex:(NSUInteger)index {
    if (index >= _filteredTasks.count) return nil;
    return _filteredTasks[index];
}

- (NSString *)summaryText {
    return [NSString stringWithFormat:@"ทั้งหมด %ld งาน | เสร็จแล้ว %ld | เกินกำหนด %ld",
            (long)self.totalCount, (long)self.completedCount, (long)self.overdueCount];
}

- (void)setCurrentFilter:(TaskListFilter)currentFilter {
    _currentFilter = currentFilter;
    [self applyFiltersAndSearch];
    [self.delegate viewModelDidUpdateTasks];
}

- (void)setSearchQuery:(NSString *)searchQuery {
    _searchQuery = searchQuery;
    [self applyFiltersAndSearch];
    [self.delegate viewModelDidUpdateTasks];
}

- (void)applyFiltersAndSearch {
    NSArray<Task *> *filtered = _allTasks ?: @[];
    
    // Apply status filter
    switch (self.currentFilter) {
        case TaskListFilterTodo:
            filtered = [filtered filteredArrayUsingPredicate:
                       [NSPredicate predicateWithBlock:^BOOL(Task *task, NSDictionary *bindings) {
                return task.status == TaskStatusTodo;
            }]];
            break;
        case TaskListFilterInProgress:
            filtered = [filtered filteredArrayUsingPredicate:
                       [NSPredicate predicateWithBlock:^BOOL(Task *task, NSDictionary *bindings) {
                return task.status == TaskStatusInProgress;
            }]];
            break;
        case TaskListFilterDone:
            filtered = [filtered filteredArrayUsingPredicate:
                       [NSPredicate predicateWithBlock:^BOOL(Task *task, NSDictionary *bindings) {
                return task.isCompleted;
            }]];
            break;
        case TaskListFilterOverdue:
            filtered = [filtered filteredArrayUsingPredicate:
                       [NSPredicate predicateWithBlock:^BOOL(Task *task, NSDictionary *bindings) {
                return task.isOverdue;
            }]];
            break;
        default:
            break;
    }
    
    // Apply search
    if (self.searchQuery.length > 0) {
        filtered = [filtered filteredArrayUsingPredicate:
                   [NSPredicate predicateWithBlock:^BOOL(Task *task, NSDictionary *bindings) {
            NSRange titleRange = [task.title rangeOfString:self.searchQuery 
                                                   options:NSCaseInsensitiveSearch];
            NSRange descRange = [task.taskDescription rangeOfString:self.searchQuery 
                                                            options:NSCaseInsensitiveSearch];
            return titleRange.location != NSNotFound || descRange.location != NSNotFound;
        }]];
    }
    
    // Sort by priority then due date
    NSSortDescriptor *prioritySort = [NSSortDescriptor sortDescriptorWithKey:@"priority" 
                                                                   ascending:NO];
    NSSortDescriptor *dueDateSort = [NSSortDescriptor sortDescriptorWithKey:@"dueDate" 
                                                                  ascending:YES];
    _filteredTasks = [filtered sortedArrayUsingDescriptors:@[prioritySort, dueDateSort]];
}

@end
```

---

## ส่วนที่ 6: UI Components

### 6.1 TaskCell

```objc
// TaskCell.h
#import <UIKit/UIKit.h>
#import "Task.h"

@interface TaskCell : UITableViewCell

@property (nonatomic, strong) UILabel *titleLabel;
@property (nonatomic, strong) UILabel *descriptionLabel;
@property (nonatomic, strong) UILabel *dueDateLabel;
@property (nonatomic, strong) UIView *priorityIndicator;
@property (nonatomic, strong) UIButton *checkButton;

- (void)configureWithTask:(Task *)task;

@end

// TaskCell.m
#import "TaskCell.h"
#import "AppTheme.h"

@implementation TaskCell

- (instancetype)initWithStyle:(UITableViewCellStyle)style 
              reuseIdentifier:(NSString *)reuseIdentifier {
    self = [super initWithStyle:style reuseIdentifier:reuseIdentifier];
    if (self) {
        [self setupUI];
    }
    return self;
}

- (void)setupUI {
    // Priority indicator
    self.priorityIndicator = [[UIView alloc] init];
    self.priorityIndicator.translatesAutoresizingMaskIntoConstraints = NO;
    self.priorityIndicator.layer.cornerRadius = 3;
    
    // Checkbox button
    self.checkButton = [UIButton buttonWithType:UIButtonTypeSystem];
    self.checkButton.translatesAutoresizingMaskIntoConstraints = NO;
    
    // Title label
    self.titleLabel = [[UILabel alloc] init];
    self.titleLabel.translatesAutoresizingMaskIntoConstraints = NO;
    self.titleLabel.font = [AppTheme bodyFont];
    self.titleLabel.numberOfLines = 2;
    
    // Description label
    self.descriptionLabel = [[UILabel alloc] init];
    self.descriptionLabel.translatesAutoresizingMaskIntoConstraints = NO;
    self.descriptionLabel.font = [AppTheme captionFont];
    self.descriptionLabel.textColor = [AppTheme secondaryTextColor];
    self.descriptionLabel.numberOfLines = 1;
    
    // Due date label
    self.dueDateLabel = [[UILabel alloc] init];
    self.dueDateLabel.translatesAutoresizingMaskIntoConstraints = NO;
    self.dueDateLabel.font = [AppTheme captionFont];
    
    // Add subviews
    [self.contentView addSubview:self.priorityIndicator];
    [self.contentView addSubview:self.checkButton];
    [self.contentView addSubview:self.titleLabel];
    [self.contentView addSubview:self.descriptionLabel];
    [self.contentView addSubview:self.dueDateLabel];
    
    // Layout
    [NSLayoutConstraint activateConstraints:@[
        // Priority indicator
        [self.priorityIndicator.leadingAnchor constraintEqualToAnchor:self.contentView.leadingAnchor constant:12],
        [self.priorityIndicator.centerYAnchor constraintEqualToAnchor:self.contentView.centerYAnchor],
        [self.priorityIndicator.widthAnchor constraintEqualToConstant:6],
        [self.priorityIndicator.heightAnchor constraintEqualToConstant:40],
        
        // Check button
        [self.checkButton.leadingAnchor constraintEqualToAnchor:self.priorityIndicator.trailingAnchor constant:12],
        [self.checkButton.centerYAnchor constraintEqualToAnchor:self.contentView.centerYAnchor],
        [self.checkButton.widthAnchor constraintEqualToConstant:24],
        [self.checkButton.heightAnchor constraintEqualToConstant:24],
        
        // Title
        [self.titleLabel.leadingAnchor constraintEqualToAnchor:self.checkButton.trailingAnchor constant:12],
        [self.titleLabel.trailingAnchor constraintEqualToAnchor:self.dueDateLabel.leadingAnchor constant:-8],
        [self.titleLabel.topAnchor constraintEqualToAnchor:self.contentView.topAnchor constant:12],
        
        // Description
        [self.descriptionLabel.leadingAnchor constraintEqualToAnchor:self.titleLabel.leadingAnchor],
        [self.descriptionLabel.trailingAnchor constraintEqualToAnchor:self.titleLabel.trailingAnchor],
        [self.descriptionLabel.topAnchor constraintEqualToAnchor:self.titleLabel.bottomAnchor constant:4],
        [self.descriptionLabel.bottomAnchor constraintEqualToAnchor:self.contentView.bottomAnchor constant:-12],
        
        // Due date
        [self.dueDateLabel.trailingAnchor constraintEqualToAnchor:self.contentView.trailingAnchor constant:-16],
        [self.dueDateLabel.topAnchor constraintEqualToAnchor:self.titleLabel.topAnchor],
        [self.dueDateLabel.widthAnchor constraintLessThanOrEqualToConstant:80]
    ]];
}

- (void)configureWithTask:(Task *)task {
    self.titleLabel.text = task.title;
    self.descriptionLabel.text = task.taskDescription.length > 0 ? task.taskDescription : @"ไม่มีคำอธิบาย";
    
    // Due date
    if (task.dueDate) {
        self.dueDateLabel.text = task.dueDateDisplayString;
        self.dueDateLabel.textColor = task.isOverdue ? [AppTheme errorColor] : [AppTheme secondaryTextColor];
    } else {
        self.dueDateLabel.text = @"";
    }
    
    // Priority color
    self.priorityIndicator.backgroundColor = [self colorForPriority:task.priority];
    
    // Completed state
    if (task.isCompleted) {
        // Strikethrough
        NSDictionary *attrs = @{NSStrikethroughStyleAttributeName: @(NSUnderlineStyleSingle),
                                NSForegroundColorAttributeName: [AppTheme secondaryTextColor]};
        self.titleLabel.attributedText = [[NSAttributedString alloc] initWithString:task.title ?: @""
                                                                         attributes:attrs];
        
        UIImage *checkImage = [UIImage systemImageNamed:@"checkmark.circle.fill"];
        [self.checkButton setImage:checkImage forState:UIControlStateNormal];
        self.checkButton.tintColor = [AppTheme successColor];
    } else {
        self.titleLabel.attributedText = nil;
        self.titleLabel.text = task.title;
        self.titleLabel.textColor = [AppTheme primaryTextColor];
        
        UIImage *emptyImage = [UIImage systemImageNamed:@"circle"];
        [self.checkButton setImage:emptyImage forState:UIControlStateNormal];
        self.checkButton.tintColor = [AppTheme secondaryTextColor];
    }
}

- (UIColor *)colorForPriority:(TaskPriority)priority {
    switch (priority) {
        case TaskPriorityLow: return [UIColor systemGreenColor];
        case TaskPriorityMedium: return [UIColor systemOrangeColor];
        case TaskPriorityHigh: return [UIColor systemRedColor];
        case TaskPriorityCritical: return [UIColor systemPurpleColor];
    }
}

@end
```

---

## ส่วนที่ 7: Authentication

### 7.1 BiometricAuthManager

```objc
// BiometricAuthManager.h
#import <Foundation/Foundation.h>
#import <LocalAuthentication/LocalAuthentication.h>

typedef void (^AuthCompletion)(BOOL success, NSError *error);

typedef NS_ENUM(NSInteger, BiometricType) {
    BiometricTypeNone = 0,
    BiometricTypeTouchID,
    BiometricTypeFaceID,
    BiometricTypeOpticID
};

@interface BiometricAuthManager : NSObject

+ (instancetype)sharedManager;

@property (nonatomic, readonly) BOOL isBiometricAvailable;
@property (nonatomic, readonly) BiometricType biometricType;
@property (nonatomic, readonly) NSString *biometricName;

- (void)authenticateWithReason:(NSString *)reason 
                    completion:(AuthCompletion)completion;

- (void)authenticateForFeature:(NSString *)feature 
                    completion:(AuthCompletion)completion;

@end

// BiometricAuthManager.m
#import "BiometricAuthManager.h"

@implementation BiometricAuthManager {
    LAContext *_context;
}

+ (instancetype)sharedManager {
    static BiometricAuthManager *manager = nil;
    static dispatch_once_t onceToken;
    dispatch_once(&onceToken, ^{
        manager = [[self alloc] init];
    });
    return manager;
}

- (instancetype)init {
    self = [super init];
    if (self) {
        _context = [[LAContext alloc] init];
    }
    return self;
}

- (BOOL)isBiometricAvailable {
    NSError *error = nil;
    return [_context canEvaluatePolicy:LAPolicyDeviceOwnerAuthenticationWithBiometrics error:&error];
}

- (BiometricType)biometricType {
    if (!self.isBiometricAvailable) return BiometricTypeNone;
    
    switch (_context.biometryType) {
        case LABiometryTypeTouchID: return BiometricTypeTouchID;
        case LABiometryTypeFaceID: return BiometricTypeFaceID;
        default: return BiometricTypeNone;
    }
}

- (NSString *)biometricName {
    switch (self.biometricType) {
        case BiometricTypeTouchID: return @"Touch ID";
        case BiometricTypeFaceID: return @"Face ID";
        case BiometricTypeOpticID: return @"Optic ID";
        default: return @"Biometric";
    }
}

- (void)authenticateWithReason:(NSString *)reason completion:(AuthCompletion)completion {
    // Create fresh context for each auth
    LAContext *context = [[LAContext alloc] init];
    context.localizedFallbackTitle = @"ใช้รหัสผ่าน";
    
    NSError *evaluateError = nil;
    if (![context canEvaluatePolicy:LAPolicyDeviceOwnerAuthenticationWithBiometrics 
                               error:&evaluateError]) {
        // Fallback to passcode
        [context evaluatePolicy:LAPolicyDeviceOwnerAuthentication
                localizedReason:reason
                          reply:^(BOOL success, NSError *authError) {
            dispatch_async(dispatch_get_main_queue(), ^{
                if (completion) completion(success, authError);
            });
        }];
        return;
    }
    
    [context evaluatePolicy:LAPolicyDeviceOwnerAuthenticationWithBiometrics
            localizedReason:reason
                      reply:^(BOOL success, NSError *authError) {
        dispatch_async(dispatch_get_main_queue(), ^{
            if (completion) completion(success, authError);
        });
    }];
}

- (void)authenticateForFeature:(NSString *)feature completion:(AuthCompletion)completion {
    NSString *reason = [NSString stringWithFormat:@"ยืนยันตัวตนเพื่อ%@", feature];
    [self authenticateWithReason:reason completion:completion];
}

@end
```

---

## ส่วนที่ 8: Push Notifications

### 8.1 NotificationManager

```objc
// NotificationManager.h
#import <Foundation/Foundation.h>
#import <UserNotifications/UserNotifications.h>
#import "Task.h"

@interface NotificationManager : NSObject <UNUserNotificationCenterDelegate>

+ (instancetype)sharedManager;

- (void)scheduleReminderForTask:(Task *)task;
- (void)cancelReminderForTask:(Task *)task;
- (void)cancelAllReminders;
- (void)requestPermission:(void (^)(BOOL granted))completion;

@end

// NotificationManager.m
#import "NotificationManager.h"

@implementation NotificationManager

+ (instancetype)sharedManager {
    static NotificationManager *manager = nil;
    static dispatch_once_t onceToken;
    dispatch_once(&onceToken, ^{
        manager = [[self alloc] init];
        [UNUserNotificationCenter currentNotificationCenter].delegate = manager;
    });
    return manager;
}

- (void)requestPermission:(void (^)(BOOL))completion {
    UNUserNotificationCenter *center = [UNUserNotificationCenter currentNotificationCenter];
    [center requestAuthorizationWithOptions:
        UNAuthorizationOptionAlert | UNAuthorizationOptionBadge | UNAuthorizationOptionSound
     completionHandler:^(BOOL granted, NSError *error) {
        dispatch_async(dispatch_get_main_queue(), ^{
            if (completion) completion(granted);
        });
    }];
}

- (void)scheduleReminderForTask:(Task *)task {
    if (!task.hasReminder || !task.reminderDate) return;
    
    // ยกเลิก notification เดิมก่อน
    [self cancelReminderForTask:task];
    
    UNMutableNotificationContent *content = [[UNMutableNotificationContent alloc] init];
    content.title = @"TaskMaster Pro";
    content.body = [NSString stringWithFormat:@"เตือน: %@", task.title];
    content.sound = [UNNotificationSound defaultSound];
    content.badge = @1;
    content.userInfo = @{
        @"taskId": task.taskId,
        @"type": @"reminder"
    };
    
    // Category สำหรับ action buttons
    content.categoryIdentifier = @"TASK_REMINDER";
    
    NSDateComponents *components = [[NSCalendar currentCalendar]
                                     components:NSCalendarUnitYear | NSCalendarUnitMonth | 
                                                NSCalendarUnitDay | NSCalendarUnitHour | 
                                                NSCalendarUnitMinute
                                       fromDate:task.reminderDate];
    
    UNCalendarNotificationTrigger *trigger = [UNCalendarNotificationTrigger 
                                               triggerWithDateMatchingComponents:components
                                               repeats:NO];
    
    NSString *identifier = [NSString stringWithFormat:@"task_%@", task.taskId];
    UNNotificationRequest *request = [UNNotificationRequest requestWithIdentifier:identifier
                                                                          content:content
                                                                          trigger:trigger];
    
    [[UNUserNotificationCenter currentNotificationCenter] 
        addNotificationRequest:request
     withCompletionHandler:^(NSError *error) {
        if (error) {
            NSLog(@"Notification schedule error: %@", error);
        } else {
            NSLog(@"Scheduled reminder for: %@", task.title);
        }
    }];
    
    // Register categories
    [self registerNotificationCategories];
}

- (void)cancelReminderForTask:(Task *)task {
    NSString *identifier = [NSString stringWithFormat:@"task_%@", task.taskId];
    [[UNUserNotificationCenter currentNotificationCenter] 
        removePendingNotificationRequestsWithIdentifiers:@[identifier]];
}

- (void)cancelAllReminders {
    [[UNUserNotificationCenter currentNotificationCenter] removeAllPendingNotificationRequests];
}

- (void)registerNotificationCategories {
    UNNotificationAction *doneAction = [UNNotificationAction 
        actionWithIdentifier:@"MARK_DONE"
                       title:@"ทำเสร็จแล้ว"
                     options:UNNotificationActionOptionNone];
    
    UNNotificationAction *snoozeAction = [UNNotificationAction 
        actionWithIdentifier:@"SNOOZE"
                       title:@"เลื่อนออกไป 1 ชั่วโมง"
                     options:UNNotificationActionOptionNone];
    
    UNNotificationCategory *category = [UNNotificationCategory 
        categoryWithIdentifier:@"TASK_REMINDER"
                       actions:@[doneAction, snoozeAction]
             intentIdentifiers:@[]
                       options:UNNotificationCategoryOptionNone];
    
    [[UNUserNotificationCenter currentNotificationCenter] 
        setNotificationCategories:[NSSet setWithObject:category]];
}

#pragma mark - UNUserNotificationCenterDelegate

// Notification arrived while app is in foreground
- (void)userNotificationCenter:(UNUserNotificationCenter *)center 
       willPresentNotification:(UNNotification *)notification 
         withCompletionHandler:(void (^)(UNNotificationPresentationOptions))completionHandler {
    completionHandler(UNNotificationPresentationOptionBanner | 
                     UNNotificationPresentationOptionSound |
                     UNNotificationPresentationOptionBadge);
}

// User tapped notification or action
- (void)userNotificationCenter:(UNUserNotificationCenter *)center 
    didReceiveNotificationResponse:(UNNotificationResponse *)response 
             withCompletionHandler:(void (^)(void))completionHandler {
    
    NSDictionary *userInfo = response.notification.request.content.userInfo;
    NSString *taskId = userInfo[@"taskId"];
    
    if ([response.actionIdentifier isEqualToString:@"MARK_DONE"]) {
        // Mark task as done
        [[NSNotificationCenter defaultCenter] postNotificationName:@"TaskMarkedDoneFromNotification"
                                                            object:nil
                                                          userInfo:@{@"taskId": taskId}];
    } else if ([response.actionIdentifier isEqualToString:@"SNOOZE"]) {
        // Snooze for 1 hour
        [[NSNotificationCenter defaultCenter] postNotificationName:@"TaskSnoozedFromNotification"
                                                            object:nil
                                                          userInfo:@{@"taskId": taskId}];
    } else {
        // Open task detail
        [[NSNotificationCenter defaultCenter] postNotificationName:@"OpenTaskFromNotification"
                                                            object:nil
                                                          userInfo:@{@"taskId": taskId}];
    }
    
    completionHandler();
}

@end
```

---

## ส่วนที่ 9: AppTheme

```objc
// AppTheme.h
#import <UIKit/UIKit.h>

@interface AppTheme : NSObject

// Setup
+ (void)setupAppearance;

// Colors
+ (UIColor *)primaryColor;
+ (UIColor *)secondaryColor;
+ (UIColor *)accentColor;
+ (UIColor *)primaryTextColor;
+ (UIColor *)secondaryTextColor;
+ (UIColor *)backgroundColor;
+ (UIColor *)cardBackgroundColor;
+ (UIColor *)errorColor;
+ (UIColor *)successColor;
+ (UIColor *)warningColor;

// Fonts
+ (UIFont *)largeTitleFont;
+ (UIFont *)titleFont;
+ (UIFont *)headlineFont;
+ (UIFont *)bodyFont;
+ (UIFont *)captionFont;

// Dimensions
+ (CGFloat)cornerRadius;
+ (CGFloat)standardPadding;
+ (UIEdgeInsets)cardInsets;

@end

// AppTheme.m
#import "AppTheme.h"

@implementation AppTheme

+ (void)setupAppearance {
    // Navigation bar
    UINavigationBarAppearance *navAppearance = [[UINavigationBarAppearance alloc] init];
    [navAppearance configureWithOpaqueBackground];
    navAppearance.backgroundColor = [self primaryColor];
    navAppearance.titleTextAttributes = @{
        NSForegroundColorAttributeName: [UIColor whiteColor],
        NSFontAttributeName: [self titleFont]
    };
    navAppearance.largeTitleTextAttributes = @{
        NSForegroundColorAttributeName: [UIColor whiteColor]
    };
    
    [UINavigationBar appearance].standardAppearance = navAppearance;
    [UINavigationBar appearance].scrollEdgeAppearance = navAppearance;
    [UINavigationBar appearance].tintColor = [UIColor whiteColor];
    
    // Tab bar
    UITabBarAppearance *tabAppearance = [[UITabBarAppearance alloc] init];
    [tabAppearance configureWithOpaqueBackground];
    tabAppearance.backgroundColor = [self cardBackgroundColor];
    
    [UITabBar appearance].standardAppearance = tabAppearance;
    [UITabBar appearance].tintColor = [self primaryColor];
    
    // Table view
    [UITableView appearance].backgroundColor = [self backgroundColor];
    [UITableViewCell appearance].backgroundColor = [self cardBackgroundColor];
}

+ (UIColor *)primaryColor {
    return [UIColor colorWithRed:0.2 green:0.55 blue:1.0 alpha:1.0];
}

+ (UIColor *)secondaryColor {
    return [UIColor colorWithRed:0.4 green:0.7 blue:1.0 alpha:1.0];
}

+ (UIColor *)accentColor {
    return [UIColor colorWithRed:1.0 green:0.6 blue:0.0 alpha:1.0];
}

+ (UIColor *)primaryTextColor {
    return [UIColor labelColor]; // Adaptive dark/light mode
}

+ (UIColor *)secondaryTextColor {
    return [UIColor secondaryLabelColor];
}

+ (UIColor *)backgroundColor {
    return [UIColor systemGroupedBackgroundColor];
}

+ (UIColor *)cardBackgroundColor {
    return [UIColor secondarySystemGroupedBackgroundColor];
}

+ (UIColor *)errorColor {
    return [UIColor systemRedColor];
}

+ (UIColor *)successColor {
    return [UIColor systemGreenColor];
}

+ (UIColor *)warningColor {
    return [UIColor systemOrangeColor];
}

+ (UIFont *)largeTitleFont {
    return [UIFont systemFontOfSize:34 weight:UIFontWeightBold];
}

+ (UIFont *)titleFont {
    return [UIFont systemFontOfSize:17 weight:UIFontWeightSemibold];
}

+ (UIFont *)headlineFont {
    return [UIFont systemFontOfSize:15 weight:UIFontWeightMedium];
}

+ (UIFont *)bodyFont {
    return [UIFont systemFontOfSize:15 weight:UIFontWeightRegular];
}

+ (UIFont *)captionFont {
    return [UIFont systemFontOfSize:12 weight:UIFontWeightRegular];
}

+ (CGFloat)cornerRadius {
    return 12.0;
}

+ (CGFloat)standardPadding {
    return 16.0;
}

+ (UIEdgeInsets)cardInsets {
    return UIEdgeInsetsMake(12, 16, 12, 16);
}

@end
```

---

## ส่วนที่ 10: Unit Tests

### 10.1 TaskTests

```objc
// TaskTests.m
#import <XCTest/XCTest.h>
#import "Task.h"

@interface TaskTests : XCTestCase
@end

@implementation TaskTests

- (void)testTaskCreation {
    Task *task = [Task taskWithTitle:@"Test Task"];
    
    XCTAssertNotNil(task, "Task should not be nil");
    XCTAssertEqualObjects(task.title, @"Test Task");
    XCTAssertEqual(task.status, TaskStatusTodo);
    XCTAssertEqual(task.priority, TaskPriorityMedium);
    XCTAssertNotNil(task.taskId, "Task ID should be auto-generated");
    XCTAssertNotNil(task.createdAt, "Created date should be set");
}

- (void)testTaskIsOverdue {
    Task *task = [Task taskWithTitle:@"Overdue Task"];
    task.dueDate = [NSDate dateWithTimeIntervalSinceNow:-3600]; // 1 hour ago
    
    XCTAssertTrue(task.isOverdue, "Task should be overdue");
}

- (void)testCompletedTaskIsNotOverdue {
    Task *task = [Task taskWithTitle:@"Completed Task"];
    task.dueDate = [NSDate dateWithTimeIntervalSinceNow:-3600];
    task.status = TaskStatusDone;
    
    XCTAssertFalse(task.isOverdue, "Completed task should not be overdue");
}

- (void)testTaskWithFutureDueDate {
    Task *task = [Task taskWithTitle:@"Future Task"];
    task.dueDate = [NSDate dateWithTimeIntervalSinceNow:3600]; // 1 hour future
    
    XCTAssertFalse(task.isOverdue, "Future task should not be overdue");
}

- (void)testTaskValidation {
    // Empty title
    Task *emptyTask = [Task taskWithTitle:@""];
    NSError *error = nil;
    BOOL valid = [emptyTask isValidForSaving:&error];
    
    XCTAssertFalse(valid, "Empty title should be invalid");
    XCTAssertNotNil(error, "Error should be set for invalid task");
    
    // Valid task
    Task *validTask = [Task taskWithTitle:@"Valid Task"];
    NSError *validError = nil;
    BOOL validResult = [validTask isValidForSaving:&validError];
    
    XCTAssertTrue(validResult, "Valid task should pass validation");
    XCTAssertNil(validError, "No error for valid task");
}

- (void)testTaskCopy {
    Task *original = [Task taskWithTitle:@"Original"];
    original.taskDescription = @"Description";
    original.priority = TaskPriorityHigh;
    original.status = TaskStatusInProgress;
    
    Task *copy = [original copy];
    
    XCTAssertEqualObjects(copy.taskId, original.taskId, "Copy should have same ID");
    XCTAssertEqualObjects(copy.title, original.title);
    XCTAssertEqualObjects(copy.taskDescription, original.taskDescription);
    XCTAssertEqual(copy.priority, original.priority);
    XCTAssertEqual(copy.status, original.status);
    XCTAssertFalse(copy == original, "Copy should be different object");
}

- (void)testTaskSerialization {
    Task *task = [[Task alloc] initWithTitle:@"Test" priority:TaskPriorityHigh];
    task.taskDescription = @"Description";
    task.status = TaskStatusInProgress;
    
    NSDictionary *dict = [task toDictionary];
    
    XCTAssertNotNil(dict[@"id"]);
    XCTAssertEqualObjects(dict[@"title"], @"Test");
    XCTAssertEqualObjects(dict[@"priority"], @(TaskPriorityHigh));
    
    Task *restored = [Task fromDictionary:dict];
    XCTAssertEqualObjects(restored.title, task.title);
    XCTAssertEqual(restored.priority, task.priority);
    XCTAssertEqual(restored.status, task.status);
}

- (void)testTaskDisplayStrings {
    Task *task = [Task taskWithTitle:@"Test"];
    
    task.priority = TaskPriorityLow;
    XCTAssertEqualObjects(task.priorityDisplayName, @"ต่ำ");
    
    task.priority = TaskPriorityHigh;
    XCTAssertEqualObjects(task.priorityDisplayName, @"สูง");
    
    task.status = TaskStatusTodo;
    XCTAssertEqualObjects(task.statusDisplayName, @"รอดำเนินการ");
    
    task.status = TaskStatusDone;
    XCTAssertEqualObjects(task.statusDisplayName, @"เสร็จแล้ว");
    XCTAssertTrue(task.isCompleted);
}

@end
```

---

### 10.2 TaskListViewModelTests

```objc
// TaskListViewModelTests.m
#import <XCTest/XCTest.h>
#import "TaskListViewModel.h"
#import "Task.h"

// Mock Repository
@interface MockTaskRepository : NSObject

@property (nonatomic, strong) NSArray<Task *> *mockedTasks;
@property (nonatomic, strong) NSError *mockedError;

@end

@implementation MockTaskRepository

- (void)fetchAllTasksWithCompletion:(void (^)(NSArray<Task *> *, NSError *))completion {
    if (completion) {
        completion(self.mockedTasks ?: @[], self.mockedError);
    }
}

@end

// Mock Delegate
@interface MockViewModelDelegate : NSObject <TaskListViewModelDelegate>

@property (nonatomic, assign) NSInteger updateCount;
@property (nonatomic, assign) NSInteger errorCount;
@property (nonatomic, strong) NSError *lastError;

@end

@implementation MockViewModelDelegate

- (void)viewModelDidUpdateTasks {
    self.updateCount++;
}

- (void)viewModelDidFailWithError:(NSError *)error {
    self.errorCount++;
    self.lastError = error;
}

- (void)viewModelDidStartLoading {}
- (void)viewModelDidFinishLoading {}

@end

@interface TaskListViewModelTests : XCTestCase

@property (nonatomic, strong) TaskListViewModel *viewModel;
@property (nonatomic, strong) MockViewModelDelegate *mockDelegate;

@end

@implementation TaskListViewModelTests

- (void)setUp {
    [super setUp];
    self.viewModel = [[TaskListViewModel alloc] init];
    self.mockDelegate = [[MockViewModelDelegate alloc] init];
    self.viewModel.delegate = self.mockDelegate;
}

- (void)tearDown {
    self.viewModel = nil;
    self.mockDelegate = nil;
    [super tearDown];
}

- (void)testInitialState {
    XCTAssertEqual(self.viewModel.tasks.count, 0);
    XCTAssertEqual(self.viewModel.totalCount, 0);
    XCTAssertFalse(self.viewModel.isLoading);
}

- (void)testFilterByStatus {
    // Create test tasks
    Task *todoTask = [Task taskWithTitle:@"Todo Task"];
    todoTask.status = TaskStatusTodo;
    
    Task *doneTask = [Task taskWithTitle:@"Done Task"];
    doneTask.status = TaskStatusDone;
    
    Task *inProgressTask = [Task taskWithTitle:@"In Progress"];
    inProgressTask.status = TaskStatusInProgress;
    
    // Manually set allTasks for testing (using KVC)
    [self.viewModel setValue:@[todoTask, doneTask, inProgressTask] forKey:@"_allTasks"];
    
    // Filter by Done
    self.viewModel.currentFilter = TaskListFilterDone;
    [self.viewModel setValue:@[doneTask] forKey:@"_filteredTasks"]; // simulate filter
    
    // Test completed count
    XCTAssertEqual(self.viewModel.completedCount, 1);
}

- (void)testSummaryText {
    Task *task1 = [Task taskWithTitle:@"Task 1"];
    Task *task2 = [Task taskWithTitle:@"Task 2"];
    task2.status = TaskStatusDone;
    
    [self.viewModel setValue:@[task1, task2] forKey:@"_allTasks"];
    [self.viewModel setValue:@[task1, task2] forKey:@"_filteredTasks"];
    
    NSString *summary = [self.viewModel summaryText];
    XCTAssertNotNil(summary);
    XCTAssertTrue([summary containsString:@"2"]);
}

- (void)testAsyncTaskLoading {
    XCTestExpectation *expectation = [self expectationWithDescription:@"Tasks loaded"];
    
    MockViewModelDelegate *delegate = self.mockDelegate;
    
    // Override delegate callback
    __weak typeof(self) weakSelf = self;
    dispatch_after(dispatch_time(DISPATCH_TIME_NOW, (int64_t)(2.0 * NSEC_PER_SEC)),
                  dispatch_get_main_queue(), ^{
        // Simulate tasks loaded
        XCTAssertNotNil(weakSelf.viewModel);
        [expectation fulfill];
    });
    
    [self waitForExpectationsWithTimeout:5.0 handler:^(NSError *error) {
        if (error) XCTFail(@"Expectation failed: %@", error);
    }];
}

@end
```

---

## ส่วนที่ 11: CI/CD Setup

### 11.1 GitHub Actions

```yaml
# .github/workflows/ios.yml
name: iOS CI

on:
  push:
    branches: [ main, develop ]
  pull_request:
    branches: [ main ]

jobs:
  test:
    name: Run Tests
    runs-on: macos-14
    
    steps:
    - name: Checkout code
      uses: actions/checkout@v4
    
    - name: Select Xcode version
      run: sudo xcode-select -switch /Applications/Xcode_15.4.app
    
    - name: Show Xcode version
      run: xcodebuild -version
    
    - name: Cache CocoaPods
      uses: actions/cache@v3
      with:
        path: Pods
        key: ${{ runner.os }}-pods-${{ hashFiles('**/Podfile.lock') }}
        restore-keys: |
          ${{ runner.os }}-pods-
    
    - name: Install CocoaPods dependencies
      run: pod install --repo-update
    
    - name: Build and test
      run: |
        xcodebuild test \
          -workspace TaskMasterPro.xcworkspace \
          -scheme TaskMasterPro \
          -sdk iphonesimulator \
          -destination 'platform=iOS Simulator,name=iPhone 15,OS=17.4' \
          -enableCodeCoverage YES \
          CODE_SIGN_IDENTITY="" \
          CODE_SIGNING_REQUIRED=NO \
          | xcpretty
    
    - name: Upload test results
      uses: actions/upload-artifact@v3
      if: failure()
      with:
        name: test-results
        path: TestResults/
    
    - name: Code coverage report
      run: |
        xcrun llvm-cov export \
          -format="lcov" \
          -instr-profile=$(find . -name "Coverage.profdata" | head -1) \
          $(find . -name "TaskMasterPro" -type f | head -1) \
          > coverage.lcov
    
    - name: Upload coverage to Codecov
      uses: codecov/codecov-action@v3
      with:
        file: coverage.lcov

  lint:
    name: SwiftLint / OCLint
    runs-on: macos-14
    
    steps:
    - uses: actions/checkout@v4
    
    - name: Run OCLint
      run: |
        brew install oclint
        xcodebuild clean build \
          -workspace TaskMasterPro.xcworkspace \
          -scheme TaskMasterPro \
          -sdk iphonesimulator \
          -destination 'platform=iOS Simulator,name=iPhone 15' \
          | xcpretty -r json-compilation-database
        oclint-json-compilation-database \
          -e Pods \
          -- -max-priority-1=0 -max-priority-2=10

  build:
    name: Build Archive
    runs-on: macos-14
    needs: test
    if: github.ref == 'refs/heads/main'
    
    steps:
    - uses: actions/checkout@v4
    
    - name: Import certificates
      uses: apple-actions/import-codesign-certs@v2
      with:
        p12-file-base64: ${{ secrets.P12_BASE64 }}
        p12-password: ${{ secrets.P12_PASSWORD }}
    
    - name: Download provisioning profiles
      uses: apple-actions/download-provisioning-profiles@v2
      with:
        bundle-id: com.example.TaskMasterPro
        issuer-id: ${{ secrets.APPSTORE_ISSUER_ID }}
        api-key-id: ${{ secrets.APPSTORE_KEY_ID }}
        api-private-key: ${{ secrets.APPSTORE_PRIVATE_KEY }}
    
    - name: Build archive
      run: |
        xcodebuild archive \
          -workspace TaskMasterPro.xcworkspace \
          -scheme TaskMasterPro \
          -archivePath TaskMasterPro.xcarchive \
          -configuration Release
    
    - name: Export IPA
      run: |
        xcodebuild -exportArchive \
          -archivePath TaskMasterPro.xcarchive \
          -exportPath . \
          -exportOptionsPlist ExportOptions.plist
    
    - name: Upload to TestFlight
      run: |
        xcrun altool --upload-app \
          -f TaskMasterPro.ipa \
          -t ios \
          -u ${{ secrets.APPLE_ID }} \
          -p ${{ secrets.APP_PASSWORD }}
```

---

### 11.2 Fastlane

```ruby
# fastlane/Fastfile

default_platform(:ios)

platform :ios do

  before_all do
    setup_circle_ci if ENV['CI']
  end

  desc "Run all tests"
  lane :test do
    run_tests(
      workspace: "TaskMasterPro.xcworkspace",
      scheme: "TaskMasterPro",
      devices: ["iPhone 15", "iPhone SE (3rd generation)"],
      clean: true,
      code_coverage: true,
      output_directory: "./test_output"
    )
    
    # Generate coverage report
    xcov(
      workspace: "TaskMasterPro.xcworkspace",
      scheme: "TaskMasterPro",
      output_directory: "./coverage_output",
      minimum_coverage_percentage: 70.0
    )
  end

  desc "Build for TestFlight"
  lane :beta do
    # Increment build number
    increment_build_number(
      build_number: latest_testflight_build_number + 1
    )
    
    # Build
    build_app(
      workspace: "TaskMasterPro.xcworkspace",
      scheme: "TaskMasterPro",
      configuration: "Release",
      export_method: "app-store",
      export_options: {
        provisioningProfiles: {
          "com.example.TaskMasterPro" => "TaskMasterPro AppStore"
        }
      }
    )
    
    # Upload to TestFlight
    upload_to_testflight(
      skip_waiting_for_build_processing: true,
      notify_external_testers: false
    )
    
    # Notify team
    slack(
      message: "New TestFlight build uploaded! v#{get_version_number} (#{get_build_number})",
      channel: "#ios-releases"
    )
  end

  desc "Release to App Store"
  lane :release do
    # Take screenshots
    capture_screenshots(
      workspace: "TaskMasterPro.xcworkspace",
      scheme: "TaskMasterPro UI Tests"
    )
    
    # Frame screenshots
    frame_screenshots(white: true)
    
    # Build
    build_app(
      workspace: "TaskMasterPro.xcworkspace",
      scheme: "TaskMasterPro",
      configuration: "Release"
    )
    
    # Upload to App Store
    deliver(
      submit_for_review: true,
      automatic_release: false,
      force: true,
      metadata_path: "./metadata",
      screenshots_path: "./screenshots"
    )
  end

  error do |lane, exception|
    slack(
      message: "Error in lane #{lane}: #{exception.message}",
      success: false
    )
  end

end
```

---

## ส่วนที่ 12: App Store Preparation

### 12.1 Info.plist Privacy Descriptions

```xml
<!-- Info.plist -->
<dict>
    <!-- Face ID -->
    <key>NSFaceIDUsageDescription</key>
    <string>ใช้ Face ID เพื่อปลดล็อคแอปและยืนยันตัวตนอย่างรวดเร็ว</string>
    
    <!-- Notifications -->
    <key>NSUserNotificationsUsageDescription</key>
    <string>รับการแจ้งเตือนสำหรับงานที่ใกล้ถึงกำหนดส่ง</string>
    
    <!-- Background fetch -->
    <key>UIBackgroundModes</key>
    <array>
        <string>fetch</string>
        <string>remote-notification</string>
    </array>
    
    <!-- App Transport Security -->
    <key>NSAppTransportSecurity</key>
    <dict>
        <key>NSAllowsArbitraryLoads</key>
        <false/>
        <key>NSExceptionDomains</key>
        <dict>
            <key>jsonplaceholder.typicode.com</key>
            <dict>
                <key>NSIncludesSubdomains</key>
                <true/>
                <key>NSExceptionAllowsInsecureHTTPLoads</key>
                <false/>
                <key>NSExceptionRequiresForwardSecrecy</key>
                <true/>
            </dict>
        </dict>
    </dict>
</dict>
```

---

## ส่วนที่ 13: Utilities

### 13.1 DateUtils

```objc
// DateUtils.h
#import <Foundation/Foundation.h>

@interface DateUtils : NSObject

+ (NSString *)relativeDateString:(NSDate *)date;
+ (NSString *)formattedDateString:(NSDate *)date format:(NSString *)format;
+ (BOOL)isToday:(NSDate *)date;
+ (BOOL)isTomorrow:(NSDate *)date;
+ (BOOL)isPast:(NSDate *)date;
+ (NSDate *)startOfDay:(NSDate *)date;
+ (NSDate *)endOfDay:(NSDate *)date;
+ (NSDate *)addDays:(NSInteger)days toDate:(NSDate *)date;

@end

// DateUtils.m
#import "DateUtils.h"

@implementation DateUtils

+ (NSString *)relativeDateString:(NSDate *)date {
    if (!date) return @"ไม่มีกำหนด";
    
    NSCalendar *calendar = [NSCalendar currentCalendar];
    NSDate *now = [NSDate date];
    
    if ([self isToday:date]) {
        NSDateFormatter *timeFormatter = [[NSDateFormatter alloc] init];
        timeFormatter.dateFormat = @"HH:mm";
        return [NSString stringWithFormat:@"วันนี้ %@", [timeFormatter stringFromDate:date]];
    }
    
    if ([self isTomorrow:date]) {
        return @"พรุ่งนี้";
    }
    
    NSDateComponents *components = [calendar components:NSCalendarUnitDay 
                                              fromDate:now 
                                                toDate:date 
                                               options:0];
    
    if (components.day < 0) {
        return [NSString stringWithFormat:@"เกินกำหนด %ld วัน", (long)abs((int)components.day)];
    }
    
    if (components.day < 7) {
        return [NSString stringWithFormat:@"ใน %ld วัน", (long)components.day];
    }
    
    NSDateFormatter *formatter = [[NSDateFormatter alloc] init];
    formatter.dateStyle = NSDateFormatterMediumStyle;
    formatter.locale = [NSLocale localeWithLocaleIdentifier:@"th_TH"];
    return [formatter stringFromDate:date];
}

+ (NSString *)formattedDateString:(NSDate *)date format:(NSString *)format {
    NSDateFormatter *formatter = [[NSDateFormatter alloc] init];
    formatter.dateFormat = format;
    return [formatter stringFromDate:date];
}

+ (BOOL)isToday:(NSDate *)date {
    return [[NSCalendar currentCalendar] isDateInToday:date];
}

+ (BOOL)isTomorrow:(NSDate *)date {
    return [[NSCalendar currentCalendar] isDateInTomorrow:date];
}

+ (BOOL)isPast:(NSDate *)date {
    return [date compare:[NSDate date]] == NSOrderedAscending;
}

+ (NSDate *)startOfDay:(NSDate *)date {
    return [[NSCalendar currentCalendar] startOfDayForDate:date];
}

+ (NSDate *)endOfDay:(NSDate *)date {
    NSDateComponents *components = [[NSDateComponents alloc] init];
    components.hour = 23;
    components.minute = 59;
    components.second = 59;
    
    NSDate *startOfDay = [self startOfDay:date];
    return [[NSCalendar currentCalendar] dateByAddingComponents:components
                                                         toDate:startOfDay
                                                        options:0];
}

+ (NSDate *)addDays:(NSInteger)days toDate:(NSDate *)date {
    NSDateComponents *components = [[NSDateComponents alloc] init];
    components.day = days;
    return [[NSCalendar currentCalendar] dateByAddingComponents:components
                                                         toDate:date
                                                        options:0];
}

@end
```

---

## ส่วนที่ 14: ทบทวน 100 ตอน

ตลอด 100 ตอนของ Objective-C Course นี้ เราได้เรียนรู้:

### ตอน 1-10: พื้นฐาน
- Syntax และ structure ของ Objective-C
- Variables, types, operators
- Control flow: if, switch, loops
- Functions และ methods
- First class

### ตอน 11-20: OOP
- Classes และ objects
- Inheritance และ polymorphism
- Categories และ extensions
- Protocols
- Memory management basics

### ตอน 21-30: Foundation Framework
- NSString, NSArray, NSDictionary, NSSet
- NSNumber, NSValue
- NSDate, NSCalendar
- File system: NSFileManager
- NSUserDefaults

### ตอน 31-40: Advanced ObjC
- Blocks và closures
- KVC/KVO
- Notifications
- Multithreading (GCD, NSOperation)
- Error handling

### ตอน 41-50: UIKit Basics
- UIViewController lifecycle
- UIView, UILabel, UIButton
- UITableView, UICollectionView
- Auto Layout
- Storyboard vs programmatic UI

### ตอน 51-60: Data Persistence
- NSUserDefaults
- File system
- Core Data fundamentals
- SQLite
- Keychain

### ตอน 61-70: Networking
- NSURLSession
- REST APIs
- JSON parsing
- Authentication
- Background fetch

### ตอน 71-80: Advanced UIKit
- Custom views
- Animations
- Gesture recognizers
- Navigation patterns
- Tab bar, drawer

### ตอน 81-90: Architecture
- MVC deep dive
- MVVM
- Dependency injection
- Design patterns
- Refactoring

### ตอน 91-100: Expert Level
- Runtime manipulation
- Performance optimization
- Testing strategies
- Interview preparation
- Advanced patterns
- Future of Objective-C
- Capstone project

---

## ใบประกาศนียบัตร

```
╔═══════════════════════════════════════════════════════════════╗
║                                                               ║
║            CERTIFICATE OF COMPLETION                         ║
║                                                               ║
║         Objective-C Programming Course                       ║
║                    100 Parts                                  ║
║                                                               ║
║  นี่เป็นการรับรองว่าผู้เรียนได้ผ่านการเรียนรู้               ║
║  Objective-C Programming ครบทั้ง 100 ตอน                     ║
║                                                               ║
║  ครอบคลุมหัวข้อ:                                              ║
║  • พื้นฐาน Objective-C และ OOP                               ║
║  • Foundation Framework และ UIKit                             ║
║  • Core Data, Networking, Security                           ║
║  • Concurrency, Performance, Testing                         ║
║  • Architecture Patterns (MVC, MVVM)                        ║
║  • Objective-C Runtime และ Advanced Patterns                 ║
║  • Interview Preparation และ Career Guidance                 ║
║  • Capstone Project: TaskMaster Pro                          ║
║                                                               ║
║                    [วันที่สำเร็จ]                             ║
║                                                               ║
║              ยินดีด้วย! คุณพร้อมสำหรับ                       ║
║           การพัฒนา iOS Apps อย่างมืออาชีพ!                  ║
║                                                               ║
╚═══════════════════════════════════════════════════════════════╝
```

---

## คำส่งท้าย

การเดินทางตลอด 100 ตอนนี้ไม่ใช่เพียงแค่การเรียนภาษาโปรแกรมหนึ่ง แต่คือการสร้างรากฐานสำหรับการเป็น iOS Developer มืออาชีพ

Objective-C สอนให้เราเข้าใจ:
- **ความชัดเจน**: code ยาวขึ้นแต่ intention ชัดกว่า
- **Runtime**: รู้ว่า "magic" เกิดขึ้นได้อย่างไร
- **Memory**: ดูแล resources อย่างมีความรับผิดชอบ
- **API Design**: ตั้งชื่อ method ให้ readable

เส้นทางต่อไป:
1. **ฝึกฝน**: สร้าง projects จริงๆ
2. **เรียน Swift**: รากฐาน ObjC ทำให้ Swift เข้าใจง่ายขึ้น
3. **Community**: เข้าร่วม iOS developer community
4. **Open Source**: contribute กลับสู่ community
5. **Share**: สอนคนอื่น = เข้าใจลึกขึ้น

**"The best way to learn is to teach"**

ขอให้โชคดีในการพัฒนา iOS Applications!

---

*จบตอนที่ 100 - Capstone Project และการสิ้นสุดของ Objective-C Course (100 ตอน)*

*"Code is not just instructions for computers - it's communication between developers"*
