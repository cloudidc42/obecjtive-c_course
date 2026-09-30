# Part 61 - SQLite ใน iOS

## บทนำ

SQLite เป็นระบบจัดการฐานข้อมูลเชิงสัมพันธ์ (RDBMS) ที่มีขนาดเล็ก รวดเร็ว และไม่ต้องการเซิร์ฟเวอร์แยกต่างหาก (serverless) SQLite ถูกฝังอยู่ใน iOS ตั้งแต่ต้น ทำให้นักพัฒนาสามารถใช้งานฐานข้อมูลได้โดยไม่ต้องติดตั้งเพิ่มเติม

ในบทนี้เราจะเรียนรู้วิธีใช้ SQLite ใน iOS ด้วย Objective-C ตั้งแต่การเปิด/ปิดฐานข้อมูล ไปจนถึงการทำ CRUD operations, transactions, และ best practices ต่างๆ

---

## 61.1 SQLite ใน iOS คืออะไร?

SQLite เป็นไลบรารีที่เขียนด้วยภาษา C ซึ่งให้บริการฐานข้อมูล SQL แบบ embedded กล่าวคือ ฐานข้อมูลทั้งหมดจะถูกเก็บไว้ในไฟล์เดียวบนเครื่อง (`.sqlite` หรือ `.db`) ข้อดีของ SQLite ได้แก่:

- **ไม่ต้องการเซิร์ฟเวอร์**: ทำงานได้โดยตรงบนอุปกรณ์
- **ขนาดเล็ก**: ไลบรารีมีขนาดเพียงไม่กี่ร้อย KB
- **รวดเร็ว**: เหมาะสำหรับข้อมูลที่ไม่ใหญ่มาก
- **มาตรฐาน SQL**: รองรับ SQL มาตรฐานส่วนใหญ่
- **ฟรี**: ไม่มีค่าลิขสิทธิ์

### การเพิ่ม libsqlite3 ใน Xcode

ก่อนใช้ SQLite ต้องเพิ่ม framework ลงใน project:

1. เปิด Xcode project
2. คลิกที่ project ใน Navigator
3. เลือก target → Build Phases → Link Binary With Libraries
4. กดปุ่ม `+` แล้วค้นหา `libsqlite3.tbd`
5. กด Add

จากนั้น import header ใน code:

```objc
#import <sqlite3.h>
```

---

## 61.2 การเปิดและปิดฐานข้อมูล

### การเปิดฐานข้อมูล

```objc
#import <sqlite3.h>

@interface DatabaseManager : NSObject {
    sqlite3 *_database;
}

@property (nonatomic, strong) NSString *databasePath;

- (BOOL)openDatabase;
- (void)closeDatabase;

@end

@implementation DatabaseManager

- (NSString *)databasePath {
    if (!_databasePath) {
        NSArray *paths = NSSearchPathForDirectoriesInDomains(
            NSDocumentDirectory, 
            NSUserDomainMask, 
            YES
        );
        NSString *documentsDir = [paths firstObject];
        _databasePath = [documentsDir stringByAppendingPathComponent:@"myapp.sqlite"];
    }
    return _databasePath;
}

- (BOOL)openDatabase {
    // ตรวจสอบว่าฐานข้อมูลถูกเปิดแล้วหรือยัง
    if (_database) {
        NSLog(@"Database is already open.");
        return YES;
    }
    
    // เปิดหรือสร้างฐานข้อมูล
    int result = sqlite3_open([self.databasePath UTF8String], &_database);
    
    if (result != SQLITE_OK) {
        NSLog(@"Failed to open database: %s", sqlite3_errmsg(_database));
        sqlite3_close(_database);
        _database = NULL;
        return NO;
    }
    
    NSLog(@"Database opened successfully at: %@", self.databasePath);
    return YES;
}

- (void)closeDatabase {
    if (_database) {
        sqlite3_close(_database);
        _database = NULL;
        NSLog(@"Database closed.");
    }
}

@end
```

### ความแตกต่างระหว่าง sqlite3_open และ sqlite3_open_v2

```objc
// sqlite3_open - เปิดแบบปกติ (อ่าน/เขียน หรือสร้างใหม่)
int rc = sqlite3_open(filename, &db);

// sqlite3_open_v2 - เปิดแบบกำหนด flags ได้
// SQLITE_OPEN_READONLY  - เปิดแบบอ่านอย่างเดียว
// SQLITE_OPEN_READWRITE - เปิดแบบอ่าน/เขียน (ต้องมีอยู่แล้ว)
// SQLITE_OPEN_CREATE    - สร้างใหม่ถ้าไม่มี
// SQLITE_OPEN_FULLMUTEX - thread-safe แบบ serialized

int rc = sqlite3_open_v2(
    filename,
    &db,
    SQLITE_OPEN_READWRITE | SQLITE_OPEN_CREATE | SQLITE_OPEN_FULLMUTEX,
    NULL  // VFS module name (NULL = default)
);
```

### Singleton Pattern สำหรับ Database Manager

```objc
@interface DatabaseManager : NSObject

+ (instancetype)sharedManager;
- (BOOL)openDatabase;
- (void)closeDatabase;
- (sqlite3 *)database;

@end

@implementation DatabaseManager

static DatabaseManager *_sharedManager = nil;

+ (instancetype)sharedManager {
    static dispatch_once_t onceToken;
    dispatch_once(&onceToken, ^{
        _sharedManager = [[DatabaseManager alloc] init];
        [_sharedManager setupDatabase];
    });
    return _sharedManager;
}

- (void)setupDatabase {
    [self copyDatabaseIfNeeded];
    [self openDatabase];
}

// คัดลอก database template จาก Bundle ถ้ายังไม่มีไฟล์
- (void)copyDatabaseIfNeeded {
    NSString *destPath = self.databasePath;
    NSFileManager *fileManager = [NSFileManager defaultManager];
    
    if (![fileManager fileExistsAtPath:destPath]) {
        NSString *sourcePath = [[NSBundle mainBundle] 
            pathForResource:@"myapp" 
            ofType:@"sqlite"];
        
        if (sourcePath) {
            NSError *error;
            [fileManager copyItemAtPath:sourcePath 
                               toPath:destPath 
                                error:&error];
            if (error) {
                NSLog(@"Error copying database: %@", error);
            }
        } else {
            NSLog(@"No template database found, creating empty database.");
        }
    }
}

- (sqlite3 *)database {
    return _database;
}

@end
```

---

## 61.3 CRUD Operations

CRUD ย่อมาจาก Create, Read, Update, Delete ซึ่งเป็นการทำงานพื้นฐานกับฐานข้อมูล

### การสร้างตาราง (Create Table)

```objc
- (BOOL)createUsersTable {
    NSString *sql = @"CREATE TABLE IF NOT EXISTS users ("
                    @"id INTEGER PRIMARY KEY AUTOINCREMENT, "
                    @"name TEXT NOT NULL, "
                    @"email TEXT UNIQUE NOT NULL, "
                    @"age INTEGER, "
                    @"created_at DATETIME DEFAULT CURRENT_TIMESTAMP"
                    @");";
    
    char *errorMessage = NULL;
    int result = sqlite3_exec(
        _database, 
        [sql UTF8String], 
        NULL,       // callback function
        NULL,       // callback argument
        &errorMessage
    );
    
    if (result != SQLITE_OK) {
        NSLog(@"Failed to create table: %s", errorMessage);
        sqlite3_free(errorMessage);
        return NO;
    }
    
    NSLog(@"Table created successfully.");
    return YES;
}
```

### Create - การเพิ่มข้อมูล (INSERT)

```objc
- (BOOL)insertUserWithName:(NSString *)name 
                    email:(NSString *)email 
                      age:(NSInteger)age {
    
    NSString *sql = @"INSERT INTO users (name, email, age) VALUES (?, ?, ?);";
    sqlite3_stmt *statement;
    
    if (sqlite3_prepare_v2(_database, [sql UTF8String], -1, &statement, NULL) != SQLITE_OK) {
        NSLog(@"Failed to prepare INSERT statement: %s", sqlite3_errmsg(_database));
        return NO;
    }
    
    // Bind parameters (index เริ่มจาก 1)
    sqlite3_bind_text(statement, 1, [name UTF8String], -1, SQLITE_TRANSIENT);
    sqlite3_bind_text(statement, 2, [email UTF8String], -1, SQLITE_TRANSIENT);
    sqlite3_bind_int(statement, 3, (int)age);
    
    int result = sqlite3_step(statement);
    sqlite3_finalize(statement);
    
    if (result != SQLITE_DONE) {
        NSLog(@"Failed to insert user: %s", sqlite3_errmsg(_database));
        return NO;
    }
    
    sqlite3_int64 lastRowId = sqlite3_last_insert_rowid(_database);
    NSLog(@"User inserted with ID: %lld", lastRowId);
    return YES;
}
```

### Read - การอ่านข้อมูล (SELECT)

```objc
- (NSArray *)getAllUsers {
    NSMutableArray *users = [NSMutableArray array];
    NSString *sql = @"SELECT id, name, email, age, created_at FROM users ORDER BY name ASC;";
    sqlite3_stmt *statement;
    
    if (sqlite3_prepare_v2(_database, [sql UTF8String], -1, &statement, NULL) != SQLITE_OK) {
        NSLog(@"Failed to prepare SELECT statement: %s", sqlite3_errmsg(_database));
        return users;
    }
    
    while (sqlite3_step(statement) == SQLITE_ROW) {
        NSMutableDictionary *user = [NSMutableDictionary dictionary];
        
        // ดึงข้อมูลจากแต่ละ column (index เริ่มจาก 0)
        int userId = sqlite3_column_int(statement, 0);
        const char *nameCStr = (const char *)sqlite3_column_text(statement, 1);
        const char *emailCStr = (const char *)sqlite3_column_text(statement, 2);
        int age = sqlite3_column_int(statement, 3);
        const char *createdAtCStr = (const char *)sqlite3_column_text(statement, 4);
        
        user[@"id"] = @(userId);
        user[@"name"] = nameCStr ? [NSString stringWithUTF8String:nameCStr] : @"";
        user[@"email"] = emailCStr ? [NSString stringWithUTF8String:emailCStr] : @"";
        user[@"age"] = @(age);
        user[@"created_at"] = createdAtCStr ? [NSString stringWithUTF8String:createdAtCStr] : @"";
        
        [users addObject:user];
    }
    
    sqlite3_finalize(statement);
    return [users copy];
}

// SELECT ด้วยเงื่อนไข WHERE
- (NSDictionary *)getUserById:(NSInteger)userId {
    NSString *sql = @"SELECT id, name, email, age FROM users WHERE id = ?;";
    sqlite3_stmt *statement;
    NSDictionary *user = nil;
    
    if (sqlite3_prepare_v2(_database, [sql UTF8String], -1, &statement, NULL) != SQLITE_OK) {
        NSLog(@"Failed to prepare statement: %s", sqlite3_errmsg(_database));
        return nil;
    }
    
    sqlite3_bind_int(statement, 1, (int)userId);
    
    if (sqlite3_step(statement) == SQLITE_ROW) {
        const char *nameCStr = (const char *)sqlite3_column_text(statement, 1);
        const char *emailCStr = (const char *)sqlite3_column_text(statement, 2);
        
        user = @{
            @"id": @(sqlite3_column_int(statement, 0)),
            @"name": nameCStr ? [NSString stringWithUTF8String:nameCStr] : @"",
            @"email": emailCStr ? [NSString stringWithUTF8String:emailCStr] : @"",
            @"age": @(sqlite3_column_int(statement, 3))
        };
    }
    
    sqlite3_finalize(statement);
    return user;
}
```

### Update - การแก้ไขข้อมูล (UPDATE)

```objc
- (BOOL)updateUserWithId:(NSInteger)userId 
                    name:(NSString *)name 
                   email:(NSString *)email 
                     age:(NSInteger)age {
    
    NSString *sql = @"UPDATE users SET name = ?, email = ?, age = ? WHERE id = ?;";
    sqlite3_stmt *statement;
    
    if (sqlite3_prepare_v2(_database, [sql UTF8String], -1, &statement, NULL) != SQLITE_OK) {
        NSLog(@"Failed to prepare UPDATE statement: %s", sqlite3_errmsg(_database));
        return NO;
    }
    
    sqlite3_bind_text(statement, 1, [name UTF8String], -1, SQLITE_TRANSIENT);
    sqlite3_bind_text(statement, 2, [email UTF8String], -1, SQLITE_TRANSIENT);
    sqlite3_bind_int(statement, 3, (int)age);
    sqlite3_bind_int(statement, 4, (int)userId);
    
    int result = sqlite3_step(statement);
    sqlite3_finalize(statement);
    
    if (result != SQLITE_DONE) {
        NSLog(@"Failed to update user: %s", sqlite3_errmsg(_database));
        return NO;
    }
    
    int changedRows = sqlite3_changes(_database);
    NSLog(@"Updated %d row(s).", changedRows);
    return YES;
}
```

### Delete - การลบข้อมูล (DELETE)

```objc
- (BOOL)deleteUserWithId:(NSInteger)userId {
    NSString *sql = @"DELETE FROM users WHERE id = ?;";
    sqlite3_stmt *statement;
    
    if (sqlite3_prepare_v2(_database, [sql UTF8String], -1, &statement, NULL) != SQLITE_OK) {
        NSLog(@"Failed to prepare DELETE statement: %s", sqlite3_errmsg(_database));
        return NO;
    }
    
    sqlite3_bind_int(statement, 1, (int)userId);
    
    int result = sqlite3_step(statement);
    sqlite3_finalize(statement);
    
    if (result != SQLITE_DONE) {
        NSLog(@"Failed to delete user: %s", sqlite3_errmsg(_database));
        return NO;
    }
    
    NSLog(@"User deleted successfully.");
    return YES;
}

// ลบทุก record ในตาราง
- (BOOL)deleteAllUsers {
    NSString *sql = @"DELETE FROM users;";
    char *errorMessage = NULL;
    
    int result = sqlite3_exec(_database, [sql UTF8String], NULL, NULL, &errorMessage);
    
    if (result != SQLITE_OK) {
        NSLog(@"Failed to delete all users: %s", errorMessage);
        sqlite3_free(errorMessage);
        return NO;
    }
    
    return YES;
}
```

---

## 61.4 Prepared Statements

Prepared statements เป็นวิธีที่แนะนำสำหรับการทำงานกับ SQLite เนื่องจาก:

1. **ความปลอดภัย**: ป้องกัน SQL Injection
2. **ประสิทธิภาพ**: Query ถูก compile ครั้งเดียวและใช้ซ้ำได้
3. **ความสะดวก**: ง่ายต่อการ bind parameters

### วิธีทำงานของ Prepared Statements

```objc
// ขั้นตอน: Prepare → Bind → Step → Reset/Finalize

@interface UserDAO : NSObject {
    sqlite3 *_db;
    sqlite3_stmt *_insertStmt;  // Reusable prepared statement
    sqlite3_stmt *_selectStmt;
}

// Pre-compile statements ตอน init
- (void)prepareStatements;
// ใช้งาน statements
- (BOOL)insertUser:(NSDictionary *)user;
- (NSArray *)searchUsersByName:(NSString *)name;

@end

@implementation UserDAO

- (void)prepareStatements {
    // Prepare INSERT statement
    const char *insertSQL = "INSERT INTO users (name, email, age) VALUES (?, ?, ?);";
    if (sqlite3_prepare_v2(_db, insertSQL, -1, &_insertStmt, NULL) != SQLITE_OK) {
        NSLog(@"Failed to prepare insert statement: %s", sqlite3_errmsg(_db));
    }
    
    // Prepare SELECT statement
    const char *selectSQL = "SELECT * FROM users WHERE name LIKE ?;";
    if (sqlite3_prepare_v2(_db, selectSQL, -1, &_selectStmt, NULL) != SQLITE_OK) {
        NSLog(@"Failed to prepare select statement: %s", sqlite3_errmsg(_db));
    }
}

- (BOOL)insertUser:(NSDictionary *)user {
    if (!_insertStmt) return NO;
    
    // Bind values
    sqlite3_bind_text(_insertStmt, 1, [user[@"name"] UTF8String], -1, SQLITE_TRANSIENT);
    sqlite3_bind_text(_insertStmt, 2, [user[@"email"] UTF8String], -1, SQLITE_TRANSIENT);
    sqlite3_bind_int(_insertStmt, 3, [user[@"age"] intValue]);
    
    // Execute
    int result = sqlite3_step(_insertStmt);
    
    // Reset สำหรับใช้ครั้งต่อไป (ไม่ต้อง finalize)
    sqlite3_reset(_insertStmt);
    sqlite3_clear_bindings(_insertStmt);
    
    return result == SQLITE_DONE;
}

- (NSArray *)searchUsersByName:(NSString *)name {
    NSMutableArray *results = [NSMutableArray array];
    
    if (!_selectStmt) return results;
    
    NSString *pattern = [NSString stringWithFormat:@"%%%@%%", name];
    sqlite3_bind_text(_selectStmt, 1, [pattern UTF8String], -1, SQLITE_TRANSIENT);
    
    while (sqlite3_step(_selectStmt) == SQLITE_ROW) {
        // อ่านข้อมูล...
        const char *nameStr = (const char *)sqlite3_column_text(_selectStmt, 1);
        if (nameStr) {
            [results addObject:[NSString stringWithUTF8String:nameStr]];
        }
    }
    
    // Reset เพื่อใช้ใหม่
    sqlite3_reset(_selectStmt);
    sqlite3_clear_bindings(_selectStmt);
    
    return [results copy];
}

- (void)dealloc {
    // Finalize ทุก statements เมื่อไม่ใช้แล้ว
    if (_insertStmt) sqlite3_finalize(_insertStmt);
    if (_selectStmt) sqlite3_finalize(_selectStmt);
}

@end
```

### ประเภทของการ Bind

```objc
// Text (NSString)
sqlite3_bind_text(stmt, 1, [str UTF8String], -1, SQLITE_TRANSIENT);

// Integer
sqlite3_bind_int(stmt, 2, intValue);
sqlite3_bind_int64(stmt, 2, int64Value);

// Real (double/float)
sqlite3_bind_double(stmt, 3, doubleValue);

// BLOB (NSData)
sqlite3_bind_blob(stmt, 4, [data bytes], (int)[data length], SQLITE_TRANSIENT);

// NULL
sqlite3_bind_null(stmt, 5);

// ความแตกต่าง SQLITE_TRANSIENT vs SQLITE_STATIC
// SQLITE_TRANSIENT: SQLite คัดลอก data ก่อน (ปลอดภัย แต่ช้ากว่า)
// SQLITE_STATIC: SQLite ไม่คัดลอก (เร็วกว่า แต่ต้องแน่ใจว่า data ยังอยู่)
```

---

## 61.5 Transactions

Transaction คือกลุ่มของ SQL operations ที่ต้องทำสำเร็จทั้งหมดหรือไม่ทำเลย (atomicity)

### ทำไมต้องใช้ Transaction?

```
ตัวอย่าง: โอนเงิน 1000 บาทจาก A ไป B
1. ลด balance ของ A: -1000
2. เพิ่ม balance ของ B: +1000

ถ้าขั้นตอนที่ 1 สำเร็จ แต่ขั้นตอนที่ 2 ล้มเหลว เงิน 1000 บาทจะหายไป!
Transaction แก้ปัญหานี้โดย rollback ทุกอย่างถ้ามีข้อผิดพลาด
```

### การใช้ Transaction ใน SQLite

```objc
- (BOOL)transferAmount:(double)amount 
            fromAccount:(NSInteger)fromId 
             toAccount:(NSInteger)toId {
    
    // เริ่ม transaction
    if (sqlite3_exec(_database, "BEGIN TRANSACTION;", NULL, NULL, NULL) != SQLITE_OK) {
        NSLog(@"Failed to begin transaction: %s", sqlite3_errmsg(_database));
        return NO;
    }
    
    BOOL success = YES;
    
    // ลด balance ของ sender
    NSString *deductSQL = @"UPDATE accounts SET balance = balance - ? WHERE id = ?;";
    sqlite3_stmt *deductStmt;
    
    if (sqlite3_prepare_v2(_database, [deductSQL UTF8String], -1, &deductStmt, NULL) == SQLITE_OK) {
        sqlite3_bind_double(deductStmt, 1, amount);
        sqlite3_bind_int(deductStmt, 2, (int)fromId);
        
        if (sqlite3_step(deductStmt) != SQLITE_DONE) {
            success = NO;
            NSLog(@"Failed to deduct amount.");
        }
        sqlite3_finalize(deductStmt);
    } else {
        success = NO;
    }
    
    // เพิ่ม balance ของ receiver
    if (success) {
        NSString *addSQL = @"UPDATE accounts SET balance = balance + ? WHERE id = ?;";
        sqlite3_stmt *addStmt;
        
        if (sqlite3_prepare_v2(_database, [addSQL UTF8String], -1, &addStmt, NULL) == SQLITE_OK) {
            sqlite3_bind_double(addStmt, 1, amount);
            sqlite3_bind_int(addStmt, 2, (int)toId);
            
            if (sqlite3_step(addStmt) != SQLITE_DONE) {
                success = NO;
                NSLog(@"Failed to add amount.");
            }
            sqlite3_finalize(addStmt);
        } else {
            success = NO;
        }
    }
    
    // Commit หรือ Rollback
    if (success) {
        if (sqlite3_exec(_database, "COMMIT;", NULL, NULL, NULL) != SQLITE_OK) {
            NSLog(@"Failed to commit transaction.");
            sqlite3_exec(_database, "ROLLBACK;", NULL, NULL, NULL);
            return NO;
        }
        NSLog(@"Transfer completed successfully.");
    } else {
        sqlite3_exec(_database, "ROLLBACK;", NULL, NULL, NULL);
        NSLog(@"Transfer failed. Rolled back.");
    }
    
    return success;
}
```

### Batch Insert ด้วย Transaction (เพิ่มประสิทธิภาพ)

```objc
// การ insert ทีละ record โดยไม่ใช้ transaction จะช้ามาก
// เพราะ SQLite เปิด/ปิด transaction ทุกครั้ง

- (BOOL)batchInsertUsers:(NSArray *)users {
    // เริ่ม transaction ครั้งเดียว
    sqlite3_exec(_database, "BEGIN TRANSACTION;", NULL, NULL, NULL);
    
    NSString *sql = @"INSERT INTO users (name, email, age) VALUES (?, ?, ?);";
    sqlite3_stmt *statement;
    
    if (sqlite3_prepare_v2(_database, [sql UTF8String], -1, &statement, NULL) != SQLITE_OK) {
        sqlite3_exec(_database, "ROLLBACK;", NULL, NULL, NULL);
        return NO;
    }
    
    BOOL success = YES;
    
    for (NSDictionary *user in users) {
        sqlite3_bind_text(statement, 1, [user[@"name"] UTF8String], -1, SQLITE_TRANSIENT);
        sqlite3_bind_text(statement, 2, [user[@"email"] UTF8String], -1, SQLITE_TRANSIENT);
        sqlite3_bind_int(statement, 3, [user[@"age"] intValue]);
        
        if (sqlite3_step(statement) != SQLITE_DONE) {
            success = NO;
            break;
        }
        
        sqlite3_reset(statement);
        sqlite3_clear_bindings(statement);
    }
    
    sqlite3_finalize(statement);
    
    if (success) {
        sqlite3_exec(_database, "COMMIT;", NULL, NULL, NULL);
        NSLog(@"Batch insert completed: %lu records", (unsigned long)users.count);
    } else {
        sqlite3_exec(_database, "ROLLBACK;", NULL, NULL, NULL);
        NSLog(@"Batch insert failed, rolled back.");
    }
    
    return success;
}
```

---

## 61.6 SQLite Wrapper Class

การเขียน SQLite wrapper ทำให้ code สะอาดและง่ายต่อการใช้งาน

### Database.h

```objc
#import <Foundation/Foundation.h>
#import <sqlite3.h>

typedef void(^DatabaseResultBlock)(NSArray *results, NSError *error);
typedef void(^DatabaseSuccessBlock)(BOOL success, NSError *error);

@interface Database : NSObject

@property (nonatomic, readonly) NSString *databasePath;

+ (instancetype)sharedDatabase;

// Setup
- (BOOL)openWithPath:(NSString *)path;
- (void)close;

// Execute (INSERT, UPDATE, DELETE)
- (BOOL)executeSQL:(NSString *)sql;
- (BOOL)executeSQL:(NSString *)sql withParameters:(NSArray *)params;
- (BOOL)executeSQL:(NSString *)sql 
    withParameters:(NSArray *)params 
             error:(NSError **)error;

// Query (SELECT)
- (NSArray *)querySQL:(NSString *)sql;
- (NSArray *)querySQL:(NSString *)sql withParameters:(NSArray *)params;

// Transaction
- (BOOL)beginTransaction;
- (BOOL)commitTransaction;
- (BOOL)rollbackTransaction;
- (BOOL)inTransaction:(BOOL (^)(void))block;

// Utility
- (NSInteger)lastInsertRowId;
- (NSInteger)affectedRows;

@end
```

### Database.m

```objc
#import "Database.h"

@interface Database () {
    sqlite3 *_db;
    BOOL _isOpen;
}
@end

@implementation Database

+ (instancetype)sharedDatabase {
    static Database *instance = nil;
    static dispatch_once_t onceToken;
    dispatch_once(&onceToken, ^{
        instance = [[Database alloc] init];
    });
    return instance;
}

- (BOOL)openWithPath:(NSString *)path {
    if (_isOpen) {
        NSLog(@"Database already open");
        return YES;
    }
    
    _databasePath = path;
    
    int flags = SQLITE_OPEN_READWRITE | SQLITE_OPEN_CREATE | SQLITE_OPEN_FULLMUTEX;
    int rc = sqlite3_open_v2([path UTF8String], &_db, flags, NULL);
    
    if (rc != SQLITE_OK) {
        NSLog(@"Cannot open database: %s", sqlite3_errmsg(_db));
        return NO;
    }
    
    // เปิด WAL mode เพื่อประสิทธิภาพที่ดีขึ้น
    sqlite3_exec(_db, "PRAGMA journal_mode=WAL;", NULL, NULL, NULL);
    // ตั้งค่า foreign keys
    sqlite3_exec(_db, "PRAGMA foreign_keys=ON;", NULL, NULL, NULL);
    
    _isOpen = YES;
    return YES;
}

- (void)close {
    if (_db) {
        sqlite3_close_v2(_db);
        _db = NULL;
        _isOpen = NO;
    }
}

// Helper: แปลง NSArray เป็น SQLite bindings
- (void)bindParameters:(NSArray *)params toStatement:(sqlite3_stmt *)stmt {
    for (int i = 0; i < params.count; i++) {
        id param = params[i];
        int idx = i + 1;
        
        if ([param isKindOfClass:[NSNull class]]) {
            sqlite3_bind_null(stmt, idx);
        } else if ([param isKindOfClass:[NSString class]]) {
            sqlite3_bind_text(stmt, idx, [param UTF8String], -1, SQLITE_TRANSIENT);
        } else if ([param isKindOfClass:[NSNumber class]]) {
            // ตรวจสอบว่าเป็น integer หรือ double
            const char *type = [param objCType];
            if (strcmp(type, @encode(double)) == 0 || strcmp(type, @encode(float)) == 0) {
                sqlite3_bind_double(stmt, idx, [param doubleValue]);
            } else {
                sqlite3_bind_int64(stmt, idx, [param longLongValue]);
            }
        } else if ([param isKindOfClass:[NSData class]]) {
            sqlite3_bind_blob(stmt, idx, [param bytes], (int)[param length], SQLITE_TRANSIENT);
        }
    }
}

- (BOOL)executeSQL:(NSString *)sql withParameters:(NSArray *)params error:(NSError **)error {
    if (!_isOpen) {
        if (error) {
            *error = [NSError errorWithDomain:@"DatabaseError" 
                                         code:-1 
                                     userInfo:@{NSLocalizedDescriptionKey: @"Database is not open"}];
        }
        return NO;
    }
    
    sqlite3_stmt *stmt;
    int rc = sqlite3_prepare_v2(_db, [sql UTF8String], -1, &stmt, NULL);
    
    if (rc != SQLITE_OK) {
        if (error) {
            NSString *msg = [NSString stringWithUTF8String:sqlite3_errmsg(_db)];
            *error = [NSError errorWithDomain:@"DatabaseError" 
                                         code:rc 
                                     userInfo:@{NSLocalizedDescriptionKey: msg}];
        }
        return NO;
    }
    
    if (params) {
        [self bindParameters:params toStatement:stmt];
    }
    
    rc = sqlite3_step(stmt);
    sqlite3_finalize(stmt);
    
    if (rc != SQLITE_DONE) {
        if (error) {
            NSString *msg = [NSString stringWithUTF8String:sqlite3_errmsg(_db)];
            *error = [NSError errorWithDomain:@"DatabaseError" 
                                         code:rc 
                                     userInfo:@{NSLocalizedDescriptionKey: msg}];
        }
        return NO;
    }
    
    return YES;
}

- (NSArray *)querySQL:(NSString *)sql withParameters:(NSArray *)params {
    NSMutableArray *results = [NSMutableArray array];
    
    if (!_isOpen) return results;
    
    sqlite3_stmt *stmt;
    if (sqlite3_prepare_v2(_db, [sql UTF8String], -1, &stmt, NULL) != SQLITE_OK) {
        NSLog(@"Query failed: %s", sqlite3_errmsg(_db));
        return results;
    }
    
    if (params) {
        [self bindParameters:params toStatement:stmt];
    }
    
    int columnCount = sqlite3_column_count(stmt);
    
    while (sqlite3_step(stmt) == SQLITE_ROW) {
        NSMutableDictionary *row = [NSMutableDictionary dictionaryWithCapacity:columnCount];
        
        for (int i = 0; i < columnCount; i++) {
            NSString *columnName = [NSString stringWithUTF8String:sqlite3_column_name(stmt, i)];
            id value = [NSNull null];
            
            int columnType = sqlite3_column_type(stmt, i);
            
            switch (columnType) {
                case SQLITE_INTEGER:
                    value = @(sqlite3_column_int64(stmt, i));
                    break;
                case SQLITE_FLOAT:
                    value = @(sqlite3_column_double(stmt, i));
                    break;
                case SQLITE_TEXT: {
                    const char *text = (const char *)sqlite3_column_text(stmt, i);
                    value = text ? [NSString stringWithUTF8String:text] : [NSNull null];
                    break;
                }
                case SQLITE_BLOB: {
                    const void *blob = sqlite3_column_blob(stmt, i);
                    int bytes = sqlite3_column_bytes(stmt, i);
                    value = blob ? [NSData dataWithBytes:blob length:bytes] : [NSNull null];
                    break;
                }
                case SQLITE_NULL:
                default:
                    value = [NSNull null];
                    break;
            }
            
            row[columnName] = value;
        }
        
        [results addObject:[row copy]];
    }
    
    sqlite3_finalize(stmt);
    return [results copy];
}

- (BOOL)inTransaction:(BOOL (^)(void))block {
    if (![self beginTransaction]) return NO;
    
    BOOL success = NO;
    @try {
        success = block();
    } @catch (NSException *exception) {
        NSLog(@"Exception in transaction: %@", exception);
        success = NO;
    }
    
    if (success) {
        return [self commitTransaction];
    } else {
        [self rollbackTransaction];
        return NO;
    }
}

- (BOOL)beginTransaction {
    return sqlite3_exec(_db, "BEGIN TRANSACTION;", NULL, NULL, NULL) == SQLITE_OK;
}

- (BOOL)commitTransaction {
    return sqlite3_exec(_db, "COMMIT;", NULL, NULL, NULL) == SQLITE_OK;
}

- (BOOL)rollbackTransaction {
    return sqlite3_exec(_db, "ROLLBACK;", NULL, NULL, NULL) == SQLITE_OK;
}

- (NSInteger)lastInsertRowId {
    return (NSInteger)sqlite3_last_insert_rowid(_db);
}

- (NSInteger)affectedRows {
    return (NSInteger)sqlite3_changes(_db);
}

@end
```

### ตัวอย่างการใช้งาน Wrapper

```objc
// การใช้งาน
Database *db = [Database sharedDatabase];
NSString *dbPath = [[NSSearchPathForDirectoriesInDomains(
    NSDocumentDirectory, NSUserDomainMask, YES) firstObject]
    stringByAppendingPathComponent:@"app.sqlite"];
    
[db openWithPath:dbPath];

// Execute SQL
[db executeSQL:@"CREATE TABLE IF NOT EXISTS notes (id INTEGER PRIMARY KEY, title TEXT, body TEXT);"];

// Insert
NSError *error;
BOOL ok = [db executeSQL:@"INSERT INTO notes (title, body) VALUES (?, ?);"
         withParameters:@[@"My Note", @"Hello World"]
                  error:&error];
if (!ok) {
    NSLog(@"Insert failed: %@", error.localizedDescription);
}

// Query
NSArray *notes = [db querySQL:@"SELECT * FROM notes WHERE title LIKE ?;"
              withParameters:@[@"%Note%"]];

for (NSDictionary *note in notes) {
    NSLog(@"Note: %@ - %@", note[@"title"], note[@"body"]);
}

// Transaction
BOOL success = [db inTransaction:^BOOL{
    BOOL r1 = [db executeSQL:@"UPDATE accounts SET balance = balance - 500 WHERE id = 1;"];
    BOOL r2 = [db executeSQL:@"UPDATE accounts SET balance = balance + 500 WHERE id = 2;"];
    return r1 && r2;
}];
```

---

## 61.7 Error Handling

```objc
// ประเภทของ SQLite Error Codes
typedef NS_ENUM(NSInteger, SQLiteErrorCode) {
    SQLiteOK          = SQLITE_OK,       // 0: ไม่มี error
    SQLiteError       = SQLITE_ERROR,    // 1: SQL error หรือ missing database
    SQLiteInternal    = SQLITE_INTERNAL, // 2: Internal logic error in SQLite
    SQLitePerm        = SQLITE_PERM,     // 3: Access permission denied
    SQLiteAbort       = SQLITE_ABORT,    // 4: Callback routine requested an abort
    SQLiteBusy        = SQLITE_BUSY,     // 5: Database file locked
    SQLiteLocked      = SQLITE_LOCKED,   // 6: Table locked
    SQLiteNoMem       = SQLITE_NOMEM,    // 7: Out of memory
    SQLiteReadOnly    = SQLITE_READONLY, // 8: Attempt to write to read-only database
    SQLiteConstraint  = SQLITE_CONSTRAINT, // 19: Constraint violation
    SQLiteNotFound    = SQLITE_NOTFOUND, // 12: Unknown opcode
    SQLiteCorrupt     = SQLITE_CORRUPT,  // 11: Database disk image is malformed
    SQLiteFull        = SQLITE_FULL,     // 13: Disk is full
};

// Error Handler
@interface SQLiteErrorHandler : NSObject
+ (NSError *)errorFromDatabase:(sqlite3 *)db code:(int)code;
+ (NSString *)messageForCode:(int)code;
@end

@implementation SQLiteErrorHandler

+ (NSError *)errorFromDatabase:(sqlite3 *)db code:(int)code {
    NSString *message;
    if (db) {
        message = [NSString stringWithUTF8String:sqlite3_errmsg(db)];
    } else {
        message = [self messageForCode:code];
    }
    
    return [NSError errorWithDomain:@"SQLiteErrorDomain"
                               code:code
                           userInfo:@{
                               NSLocalizedDescriptionKey: message,
                               @"SQLiteErrorCode": @(code)
                           }];
}

+ (NSString *)messageForCode:(int)code {
    switch (code) {
        case SQLITE_OK:       return @"No error";
        case SQLITE_ERROR:    return @"SQL error or missing database";
        case SQLITE_BUSY:     return @"Database is locked";
        case SQLITE_LOCKED:   return @"Table is locked";
        case SQLITE_NOMEM:    return @"Out of memory";
        case SQLITE_READONLY: return @"Attempt to write to read-only database";
        case SQLITE_CONSTRAINT: return @"Constraint violation";
        case SQLITE_CORRUPT:  return @"Database disk image is malformed";
        case SQLITE_FULL:     return @"Disk is full";
        default:              return [NSString stringWithFormat:@"SQLite error code: %d", code];
    }
}

@end
```

---

## 61.8 Database Migration

Migration คือกระบวนการอัปเดต schema ของฐานข้อมูลเมื่อ app มีการเปลี่ยนแปลง

### ระบบ Version-based Migration

```objc
@interface MigrationManager : NSObject

@property (nonatomic, assign) NSInteger currentVersion;
@property (nonatomic, assign) NSInteger targetVersion;

- (void)migrateDatabase:(sqlite3 *)db;

@end

@implementation MigrationManager

- (NSInteger)getDatabaseVersion:(sqlite3 *)db {
    NSString *sql = @"PRAGMA user_version;";
    sqlite3_stmt *stmt;
    NSInteger version = 0;
    
    if (sqlite3_prepare_v2(db, [sql UTF8String], -1, &stmt, NULL) == SQLITE_OK) {
        if (sqlite3_step(stmt) == SQLITE_ROW) {
            version = sqlite3_column_int(stmt, 0);
        }
        sqlite3_finalize(stmt);
    }
    return version;
}

- (void)setDatabaseVersion:(NSInteger)version forDatabase:(sqlite3 *)db {
    NSString *sql = [NSString stringWithFormat:@"PRAGMA user_version = %ld;", (long)version];
    sqlite3_exec(db, [sql UTF8String], NULL, NULL, NULL);
}

- (void)migrateDatabase:(sqlite3 *)db {
    NSInteger currentVersion = [self getDatabaseVersion:db];
    NSLog(@"Current DB version: %ld, Target: %ld", (long)currentVersion, (long)_targetVersion);
    
    // รัน migration จาก version ปัจจุบันไปยัง version เป้าหมาย
    for (NSInteger v = currentVersion + 1; v <= _targetVersion; v++) {
        BOOL success = [self runMigration:v forDatabase:db];
        if (!success) {
            NSLog(@"Migration to version %ld failed!", (long)v);
            break;
        }
        NSLog(@"Successfully migrated to version %ld", (long)v);
    }
}

- (BOOL)runMigration:(NSInteger)version forDatabase:(sqlite3 *)db {
    sqlite3_exec(db, "BEGIN TRANSACTION;", NULL, NULL, NULL);
    
    BOOL success = YES;
    char *errorMsg = NULL;
    int rc;
    
    switch (version) {
        case 1:
            // Version 1: สร้างตาราง users
            rc = sqlite3_exec(db,
                "CREATE TABLE IF NOT EXISTS users ("
                "id INTEGER PRIMARY KEY AUTOINCREMENT,"
                "name TEXT NOT NULL,"
                "email TEXT UNIQUE);",
                NULL, NULL, &errorMsg);
            success = (rc == SQLITE_OK);
            break;
            
        case 2:
            // Version 2: เพิ่ม column age และ phone
            rc = sqlite3_exec(db, "ALTER TABLE users ADD COLUMN age INTEGER;", NULL, NULL, &errorMsg);
            if (rc == SQLITE_OK) {
                rc = sqlite3_exec(db, "ALTER TABLE users ADD COLUMN phone TEXT;", NULL, NULL, &errorMsg);
            }
            success = (rc == SQLITE_OK);
            break;
            
        case 3:
            // Version 3: สร้างตาราง posts
            rc = sqlite3_exec(db,
                "CREATE TABLE IF NOT EXISTS posts ("
                "id INTEGER PRIMARY KEY AUTOINCREMENT,"
                "user_id INTEGER,"
                "title TEXT,"
                "body TEXT,"
                "created_at DATETIME DEFAULT CURRENT_TIMESTAMP,"
                "FOREIGN KEY (user_id) REFERENCES users(id));",
                NULL, NULL, &errorMsg);
            success = (rc == SQLITE_OK);
            break;
            
        case 4:
            // Version 4: เพิ่ม index
            rc = sqlite3_exec(db,
                "CREATE INDEX IF NOT EXISTS idx_posts_user_id ON posts(user_id);",
                NULL, NULL, &errorMsg);
            success = (rc == SQLITE_OK);
            break;
    }
    
    if (!success) {
        if (errorMsg) {
            NSLog(@"Migration v%ld failed: %s", (long)version, errorMsg);
            sqlite3_free(errorMsg);
        }
        sqlite3_exec(db, "ROLLBACK;", NULL, NULL, NULL);
        return NO;
    }
    
    [self setDatabaseVersion:version forDatabase:db];
    sqlite3_exec(db, "COMMIT;", NULL, NULL, NULL);
    return YES;
}

@end

// การใช้งาน
MigrationManager *migrationMgr = [[MigrationManager alloc] init];
migrationMgr.targetVersion = 4; // เวอร์ชันล่าสุด
[migrationMgr migrateDatabase:_database];
```

---

## 61.9 Performance Tips

### 1. ใช้ WAL Mode (Write-Ahead Logging)

```objc
// WAL mode ทำให้ read และ write ทำงานพร้อมกันได้
sqlite3_exec(db, "PRAGMA journal_mode=WAL;", NULL, NULL, NULL);
```

### 2. ใช้ Indexes อย่างเหมาะสม

```objc
// สร้าง index สำหรับ column ที่ค้นหาบ่อย
sqlite3_exec(db, 
    "CREATE INDEX IF NOT EXISTS idx_users_email ON users(email);", 
    NULL, NULL, NULL);

// สร้าง composite index
sqlite3_exec(db,
    "CREATE INDEX IF NOT EXISTS idx_posts_user_date ON posts(user_id, created_at);",
    NULL, NULL, NULL);
```

### 3. ใช้ Transaction สำหรับ Batch Operations

```objc
// ช้า: insert ทีละ 1000 rows ไม่มี transaction
for (int i = 0; i < 1000; i++) {
    [db executeSQL:@"INSERT INTO data VALUES (?);" withParameters:@[@(i)]];
}
// ประมาณ 1000 transaction = ช้ามาก

// เร็ว: ใช้ transaction ครั้งเดียว
[db beginTransaction];
for (int i = 0; i < 1000; i++) {
    [db executeSQL:@"INSERT INTO data VALUES (?);" withParameters:@[@(i)]];
}
[db commitTransaction];
// ประมาณ 10-100x เร็วกว่า
```

### 4. ตั้งค่า Cache Size

```objc
// เพิ่ม page cache (default = 2000 pages = ~8MB)
sqlite3_exec(db, "PRAGMA cache_size = 10000;", NULL, NULL, NULL);
// หรือกำหนดเป็น bytes (ใส่ค่าลบ)
sqlite3_exec(db, "PRAGMA cache_size = -20000;", NULL, NULL, NULL); // 20MB
```

### 5. ใช้ EXPLAIN QUERY PLAN วิเคราะห์ Query

```objc
// ดูว่า SQLite ใช้ index หรือเปล่า
NSString *sql = @"EXPLAIN QUERY PLAN SELECT * FROM users WHERE email = ?;";
NSArray *plan = [db querySQL:sql withParameters:@[@"test@example.com"]];
for (NSDictionary *row in plan) {
    NSLog(@"Query Plan: %@", row[@"detail"]);
}
// ถ้าเห็น "SCAN TABLE" หมายถึงไม่มี index (ช้า)
// ถ้าเห็น "SEARCH TABLE USING INDEX" หมายถึงมี index (เร็ว)
```

### 6. Synchronous Mode

```objc
// FULL: ปลอดภัยที่สุด แต่ช้า (default)
sqlite3_exec(db, "PRAGMA synchronous=FULL;", NULL, NULL, NULL);

// NORMAL: สมดุลระหว่างความเร็วและความปลอดภัย
sqlite3_exec(db, "PRAGMA synchronous=NORMAL;", NULL, NULL, NULL);

// OFF: เร็วที่สุด แต่เสี่ยงข้อมูลเสียหายถ้าไฟดับ
sqlite3_exec(db, "PRAGMA synchronous=OFF;", NULL, NULL, NULL);
```

---

## 61.10 SQLite vs Core Data

| คุณสมบัติ | SQLite | Core Data |
|----------|--------|-----------|
| ระดับ | Low-level (C API) | High-level (ORM) |
| ความยืดหยุ่น | สูงมาก | ปานกลาง |
| ความง่าย | ต้องเขียน code เยอะ | ง่ายกว่า |
| Performance | ดีมากถ้า tune ถูก | ดีสำหรับทั่วไป |
| iCloud Sync | ทำเองได้ยาก | รองรับในตัว |
| Relationships | ต้องจัดการเอง | จัดการให้อัตโนมัติ |
| Thread Safety | ต้องระวังด้วยตัวเอง | มี Concurrency API |

### เมื่อไรควรใช้ SQLite โดยตรง

```
1. ต้องการ query ที่ซับซ้อนแบบ custom SQL
2. Migration จาก SQLite database เดิม
3. ต้องการ full text search (FTS5)
4. ต้องการควบคุม performance อย่างละเอียด
5. ข้อมูลมาจาก SQLite ที่แชร์กับ platform อื่น
```

### เมื่อไรควรใช้ Core Data

```
1. App ใหม่บน iOS/macOS
2. ต้องการ iCloud sync
3. ต้องการ relationship management
4. ทีมไม่คุ้นเคยกับ SQL
5. ต้องการ Undo/Redo support
```

---

## 61.11 แบบฝึกหัด

### แบบฝึกหัดที่ 1: Todo App Database

สร้าง SQLite database สำหรับ Todo App พร้อม:
- ตาราง `todos` (id, title, is_completed, priority, due_date, created_at)
- ตาราง `categories` (id, name, color)
- ตาราง `todo_categories` (todo_id, category_id)
- CRUD operations ครบถ้วน
- Query: ดู todos ที่ยังไม่เสร็จ เรียงตาม priority
- Query: ดู todos ในแต่ละ category

### แบบฝึกหัดที่ 2: Caching System

สร้าง cache system ด้วย SQLite:
- เก็บ API response ลง SQLite
- กำหนด expiry time
- Auto-cleanup records ที่หมดอายุ
- Thread-safe implementation

```objc
// โครงสร้างที่คาดหวัง
@interface CacheManager : NSObject
- (void)cacheData:(NSData *)data forKey:(NSString *)key expiresIn:(NSTimeInterval)seconds;
- (NSData *)cachedDataForKey:(NSString *)key;
- (void)invalidateCacheForKey:(NSString *)key;
- (void)cleanExpiredCache;
@end
```

### แบบฝึกหัดที่ 3: Full Text Search

เพิ่ม FTS5 (Full Text Search) ให้กับตาราง articles:

```sql
-- สร้าง FTS5 virtual table
CREATE VIRTUAL TABLE articles_fts USING fts5(
    title, 
    body,
    content='articles',
    content_rowid='id'
);

-- Triggers สำหรับ sync
CREATE TRIGGER articles_ai AFTER INSERT ON articles BEGIN
    INSERT INTO articles_fts(rowid, title, body) 
    VALUES (new.id, new.title, new.body);
END;
```

```objc
// Search
NSArray *results = [db querySQL:@"SELECT articles.* FROM articles "
                                @"JOIN articles_fts ON articles.id = articles_fts.rowid "
                                @"WHERE articles_fts MATCH ? "
                                @"ORDER BY rank;"
             withParameters:@[searchText]];
```

### แบบฝึกหัดที่ 4: Migration System

สร้าง migration system ที่:
- อ่าน migration files จาก bundle (เช่น `001_create_users.sql`, `002_add_index.sql`)
- รัน migrations ที่ยังไม่เคยรัน
- เก็บประวัติ migration ลงตาราง `schema_migrations`
- Support rollback

---

## สรุป

ในบทนี้เราได้เรียนรู้:

1. **SQLite Basics**: วิธีเปิด/ปิดฐานข้อมูล และ API พื้นฐาน
2. **CRUD Operations**: INSERT, SELECT, UPDATE, DELETE
3. **Prepared Statements**: วิธีที่ปลอดภัยและมีประสิทธิภาพ
4. **Transactions**: การทำงานแบบ atomic
5. **Wrapper Class**: การสร้าง abstraction layer ที่ใช้งานง่าย
6. **Error Handling**: การจัดการ errors อย่างเหมาะสม
7. **Migration**: การอัปเดต database schema
8. **Performance**: เทคนิคเพิ่มประสิทธิภาพ
9. **SQLite vs Core Data**: เปรียบเทียบและเลือกใช้อย่างเหมาะสม

บทต่อไปเราจะเรียนรู้เกี่ยวกับ **Keychain** ซึ่งเป็นที่เก็บข้อมูลที่ปลอดภัยสำหรับ passwords และ sensitive data
