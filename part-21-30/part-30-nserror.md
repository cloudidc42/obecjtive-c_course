# Part 30: NSError และ Exception Handling ใน Objective-C

## บทนำ

การจัดการข้อผิดพลาด (Error Handling) เป็นส่วนสำคัญของการเขียนโปรแกรมที่ดี ใน Objective-C มีสองแนวทางหลักสำหรับการจัดการข้อผิดพลาด:

1. **NSError** - สำหรับ recoverable errors (ข้อผิดพลาดที่คาดได้และสามารถจัดการได้)
2. **@try/@catch/@throw/@finally** - สำหรับ exceptions (สถานการณ์ที่ไม่คาดคิด รุนแรงกว่า)

ใน Cocoa/Foundation framework มีการแยกแนวคิดชัดเจนว่าควรใช้แบบไหนเมื่อไร และ NSError เป็นแนวทางที่แนะนำสำหรับงานส่วนใหญ่

---

## 30.1 NSError vs Exceptions - ความแตกต่าง

| เรื่อง               | NSError                              | @try/@catch                          |
|---------------------|--------------------------------------|--------------------------------------|
| ใช้เมื่อ            | ข้อผิดพลาดที่คาดได้ (expected)      | สถานการณ์ไม่คาดคิด (unexpected)     |
| ตัวอย่าง            | file not found, network error        | nil dereference, out-of-bounds       |
| Performance         | ดีกว่า                               | แย่กว่า (มี overhead สูง)           |
| ใช้ใน Cocoa API     | บ่อยมาก                              | น้อย                                |
| Propagation         | ต้องส่งต่อด้วยตัวเอง                | unwind stack อัตโนมัติ              |

**หลักการ:** ใน Objective-C ให้ใช้ NSError สำหรับ error handling ปกติ และใช้ exceptions เฉพาะสำหรับ programmer errors (bugs) เท่านั้น

---

## 30.2 NSError พื้นฐาน

### 30.2.1 โครงสร้างของ NSError

```objc
#import <Foundation/Foundation.h>

int main(int argc, const char * argv[]) {
    @autoreleasepool {
        // สร้าง NSError ด้วยข้อมูลพื้นฐาน
        NSError *error = [NSError errorWithDomain:@"com.myapp.ErrorDomain"
                                             code:404
                                         userInfo:@{
            NSLocalizedDescriptionKey: @"ไม่พบข้อมูล",
            NSLocalizedFailureReasonErrorKey: @"ไม่มีข้อมูลที่ตรงกับ ID ที่ระบุ",
            NSLocalizedRecoverySuggestionErrorKey: @"โปรดตรวจสอบ ID และลองใหม่"
        }];
        
        // อ่านข้อมูลจาก NSError
        NSLog(@"Domain:      %@", error.domain);
        NSLog(@"Code:        %ld", (long)error.code);
        NSLog(@"Description: %@", error.localizedDescription);
        NSLog(@"Reason:      %@", error.localizedFailureReason);
        NSLog(@"Suggestion:  %@", error.localizedRecoverySuggestion);
        NSLog(@"UserInfo:    %@", error.userInfo);
    }
    return 0;
}
```

### 30.2.2 Error Domain

Error domain เป็น string ที่ระบุว่าข้อผิดพลาดมาจากที่ไหน ควรใช้ reverse DNS format:

```objc
// ตัวอย่าง domain ที่ใช้ในระบบ
// NSCocoaErrorDomain         - Cocoa framework errors
// NSURLErrorDomain           - URL/network errors
// NSPOSIXErrorDomain         - POSIX errors
// NSOSStatusErrorDomain      - macOS/iOS OS errors
// NSMachErrorDomain          - Mach errors

// สำหรับ app ของเราเอง
NSString *const MyAppErrorDomain = @"com.myapp.errors";
NSString *const NetworkErrorDomain = @"com.myapp.network.errors";
NSString *const DatabaseErrorDomain = @"com.myapp.database.errors";
```

### 30.2.3 Error Codes

```objc
#import <Foundation/Foundation.h>

// กำหนด error codes ด้วย enum
typedef NS_ENUM(NSInteger, MyAppErrorCode) {
    MyAppErrorCodeUnknown           = 0,
    MyAppErrorCodeInvalidInput      = 1001,
    MyAppErrorCodeNotFound          = 1002,
    MyAppErrorCodeNetworkError      = 2001,
    MyAppErrorCodeTimeoutError      = 2002,
    MyAppErrorCodeAuthenticationFailed = 3001,
    MyAppErrorCodePermissionDenied  = 3002,
    MyAppErrorCodeDatabaseError     = 4001,
    MyAppErrorCodeDuplicateEntry    = 4002,
};

NSString *const MyAppErrorDomain = @"com.myapp.errors";

// ฟังก์ชัน helper สร้าง NSError
NSError* makeError(MyAppErrorCode code, NSString *description) {
    return [NSError errorWithDomain:MyAppErrorDomain
                               code:code
                           userInfo:@{NSLocalizedDescriptionKey: description}];
}

int main(int argc, const char * argv[]) {
    @autoreleasepool {
        NSError *e1 = makeError(MyAppErrorCodeNotFound, @"ไม่พบผู้ใช้งาน");
        NSError *e2 = makeError(MyAppErrorCodeNetworkError, @"การเชื่อมต่อล้มเหลว");
        NSError *e3 = makeError(MyAppErrorCodeAuthenticationFailed, @"รหัสผ่านไม่ถูกต้อง");
        
        NSLog(@"Error 1 code: %ld - %@", (long)e1.code, e1.localizedDescription);
        NSLog(@"Error 2 code: %ld - %@", (long)e2.code, e2.localizedDescription);
        NSLog(@"Error 3 code: %ld - %@", (long)e3.code, e3.localizedDescription);
    }
    return 0;
}
```

---

## 30.3 NSLocalizedDescriptionKey และ userInfo Keys

```objc
#import <Foundation/Foundation.h>

int main(int argc, const char * argv[]) {
    @autoreleasepool {
        // Keys ที่ใช้บ่อยใน userInfo
        NSDictionary *userInfo = @{
            // คำอธิบายหลักของ error (สำคัญที่สุด)
            NSLocalizedDescriptionKey: @"ไม่สามารถอ่านไฟล์ได้",
            
            // เหตุผลที่เกิด error
            NSLocalizedFailureReasonErrorKey: @"ไฟล์ถูกล็อคหรือไม่มีสิทธิ์",
            
            // แนะนำวิธีแก้ไข
            NSLocalizedRecoverySuggestionErrorKey: @"ตรวจสอบสิทธิ์การเข้าถึงไฟล์",
            
            // Options สำหรับ recovery (ใช้กับ NSRecoveryAttempterErrorKey)
            NSLocalizedRecoveryOptionsErrorKey: @[@"ลองใหม่", @"ยกเลิก"],
            
            // File path ที่เกิด error
            NSFilePathErrorKey: @"/Users/john/Documents/data.txt",
            
            // URL ที่เกิด error
            NSURLErrorKey: [NSURL fileURLWithPath:@"/Users/john/Documents/data.txt"],
            
            // Underlying error ที่เป็นสาเหตุ
            NSUnderlyingErrorKey: [NSError errorWithDomain:NSPOSIXErrorDomain
                                                      code:EACCES
                                                  userInfo:nil],
        };
        
        NSError *error = [NSError errorWithDomain:@"com.myapp.file"
                                             code:1
                                         userInfo:userInfo];
        
        NSLog(@"Description:  %@", error.localizedDescription);
        NSLog(@"Reason:       %@", error.localizedFailureReason);
        NSLog(@"Suggestion:   %@", error.localizedRecoverySuggestion);
        NSLog(@"File path:    %@", error.userInfo[NSFilePathErrorKey]);
        
        NSError *underlying = error.userInfo[NSUnderlyingErrorKey];
        if (underlying) {
            NSLog(@"Underlying error: %@ (code: %ld)",
                  underlying.domain, (long)underlying.code);
        }
    }
    return 0;
}
```

---

## 30.4 Passing Errors ด้วย NSError**

แนวทางหลักของ Cocoa คือส่ง `NSError **` เป็น parameter เพื่อให้ฟังก์ชันสามารถตั้งค่า error ได้:

### 30.4.1 Pattern พื้นฐาน

```objc
#import <Foundation/Foundation.h>

NSString *const MyErrorDomain = @"com.myapp";

typedef NS_ENUM(NSInteger, MyErrorCode) {
    MyErrorCodeEmptyInput = 100,
    MyErrorCodeInvalidFormat = 101,
    MyErrorCodeValueOutOfRange = 102,
};

// ฟังก์ชันที่ return BOOL และรับ NSError** เพื่อแจ้ง error
BOOL validateAge(NSInteger age, NSError **error) {
    if (age < 0) {
        if (error) {
            *error = [NSError errorWithDomain:MyErrorDomain
                                         code:MyErrorCodeValueOutOfRange
                                     userInfo:@{
                NSLocalizedDescriptionKey: @"อายุไม่ถูกต้อง",
                NSLocalizedFailureReasonErrorKey: @"อายุต้องไม่ติดลบ",
            }];
        }
        return NO;
    }
    if (age > 150) {
        if (error) {
            *error = [NSError errorWithDomain:MyErrorDomain
                                         code:MyErrorCodeValueOutOfRange
                                     userInfo:@{
                NSLocalizedDescriptionKey: @"อายุไม่สมเหตุสมผล",
                NSLocalizedFailureReasonErrorKey: @"อายุมากเกินไป (> 150 ปี)",
            }];
        }
        return NO;
    }
    return YES; // valid
}

// ฟังก์ชันที่ return object และ nil เมื่อ error
NSString* parseEmail(NSString *input, NSError **error) {
    if (input.length == 0) {
        if (error) {
            *error = [NSError errorWithDomain:MyErrorDomain
                                         code:MyErrorCodeEmptyInput
                                     userInfo:@{
                NSLocalizedDescriptionKey: @"อีเมลว่างเปล่า"
            }];
        }
        return nil;
    }
    
    // ตรวจสอบมี @ หรือไม่ (simplified validation)
    if (![input containsString:@"@"]) {
        if (error) {
            *error = [NSError errorWithDomain:MyErrorDomain
                                         code:MyErrorCodeInvalidFormat
                                     userInfo:@{
                NSLocalizedDescriptionKey: @"รูปแบบอีเมลไม่ถูกต้อง",
                NSLocalizedFailureReasonErrorKey: @"ต้องมีเครื่องหมาย @"
            }];
        }
        return nil;
    }
    
    return [input lowercaseString]; // คืนค่า normalized email
}

int main(int argc, const char * argv[]) {
    @autoreleasepool {
        // ทดสอบ validateAge
        NSError *error = nil;
        
        if (validateAge(25, &error)) {
            NSLog(@"อายุ 25: valid ✓");
        }
        
        error = nil;
        if (!validateAge(-5, &error)) {
            NSLog(@"อายุ -5: invalid - %@", error.localizedDescription);
        }
        
        error = nil;
        if (!validateAge(200, &error)) {
            NSLog(@"อายุ 200: invalid - %@", error.localizedDescription);
        }
        
        // ทดสอบ parseEmail
        error = nil;
        NSString *email1 = parseEmail(@"user@example.com", &error);
        if (email1) {
            NSLog(@"Email: %@", email1);
        }
        
        error = nil;
        NSString *email2 = parseEmail(@"not-an-email", &error);
        if (!email2) {
            NSLog(@"Error: %@", error.localizedDescription);
        }
        
        // ไม่สนใจ error (pass NULL แทน &error)
        NSString *email3 = parseEmail(@"", NULL);
        NSLog(@"Result (ignoring error): %@", email3 ?: @"nil");
    }
    return 0;
}
```

### 30.4.2 Pattern ที่ถูกต้องในการตรวจสอบ error

```objc
// แบบที่ถูกต้อง - ตรวจสอบ return value ก่อน แล้วค่อยดู error
NSError *error = nil;
BOOL success = someFunction(param, &error);
if (!success) {
    // จัดการ error
    NSLog(@"Error: %@", error.localizedDescription);
}

// แบบที่ผิด - ตรวจสอบ error ก่อน (อาจ error ไม่ถูก nil แม้สำเร็จ)
NSError *error2 = nil;
BOOL success2 = someFunction(param, &error2);
if (error2) { // อย่าทำแบบนี้!
    NSLog(@"Error: %@", error2.localizedDescription);
}
```

---

## 30.5 @try / @catch / @throw / @finally

### 30.5.1 โครงสร้างพื้นฐาน

```objc
#import <Foundation/Foundation.h>

int main(int argc, const char * argv[]) {
    @autoreleasepool {
        @try {
            // โค้ดที่อาจ throw exception
            NSArray *array = @[@1, @2, @3];
            NSLog(@"element: %@", array[10]); // out-of-bounds!
        }
        @catch (NSException *exception) {
            NSLog(@"Exception caught!");
            NSLog(@"Name:   %@", exception.name);
            NSLog(@"Reason: %@", exception.reason);
        }
        @finally {
            // โค้ดที่รันเสมอ ไม่ว่าจะ throw หรือไม่
            NSLog(@"Finally block executed");
        }
        
        NSLog(@"Program continues...");
    }
    return 0;
}
```

**ผลลัพธ์:**
```
Exception caught!
Name:   NSRangeException
Reason: *** -[__NSArrayI objectAtIndexedSubscript:]: index 10 beyond bounds [0 .. 2]
Finally block executed
Program continues...
```

### 30.5.2 Exception Types ที่พบบ่อย

```objc
#import <Foundation/Foundation.h>

int main(int argc, const char * argv[]) {
    @autoreleasepool {
        // 1. NSRangeException - index out of bounds
        @try {
            NSArray *arr = @[@"a", @"b"];
            [arr objectAtIndex:5];
        }
        @catch (NSException *e) {
            NSLog(@"NSRangeException: %@", e.reason);
        }
        
        // 2. NSInvalidArgumentException - argument ไม่ถูกต้อง
        @try {
            NSMutableDictionary *dict = [NSMutableDictionary dictionary];
            [dict setObject:nil forKey:@"key"]; // nil ไม่ได้!
        }
        @catch (NSException *e) {
            NSLog(@"NSInvalidArgumentException: %@", e.reason);
        }
        
        // 3. NSInternalInconsistencyException - ภายในไม่สอดคล้องกัน
        // 4. NSObjectInaccessibleException - ไม่สามารถเข้าถึง object ได้
        
        // catch หลาย exception type
        @try {
            // โค้ดที่อาจ throw หลายชนิด
            NSString *s = nil;
            [s uppercaseString]; // message ส่งไป nil = safe (return nil)
            // ตัวอย่างนี้จะไม่ throw เพราะ Objective-C รับ nil message ได้
        }
        @catch (NSRangeException *e) {
            NSLog(@"Range error: %@", e.reason);
        }
        @catch (NSInvalidArgumentException *e) {
            NSLog(@"Invalid argument: %@", e.reason);
        }
        @catch (NSException *e) {
            NSLog(@"Other exception: %@ - %@", e.name, e.reason);
        }
        @finally {
            NSLog(@"Cleanup done");
        }
    }
    return 0;
}
```

### 30.5.3 @throw

```objc
#import <Foundation/Foundation.h>

// ฟังก์ชันที่ throw exception โดยตั้งใจ
void processData(NSData *data) {
    if (data == nil) {
        NSException *exception = [NSException exceptionWithName:@"InvalidDataException"
                                                         reason:@"Data cannot be nil"
                                                       userInfo:nil];
        @throw exception;
    }
    
    if (data.length == 0) {
        @throw [NSException exceptionWithName:@"EmptyDataException"
                                       reason:@"Data cannot be empty"
                                     userInfo:@{@"expectedMinLength": @1}];
    }
    
    NSLog(@"Processing %lu bytes...", (unsigned long)data.length);
}

// re-throw exception หลังจาก log
void safeProcessData(NSData *data) {
    @try {
        processData(data);
    }
    @catch (NSException *e) {
        NSLog(@"[ERROR] Exception in processData: %@ - %@", e.name, e.reason);
        @throw; // re-throw ให้ caller จัดการต่อ
    }
}

int main(int argc, const char * argv[]) {
    @autoreleasepool {
        // ทดสอบ throw nil
        @try {
            processData(nil);
        }
        @catch (NSException *e) {
            NSLog(@"Caught: %@ - %@", e.name, e.reason);
        }
        
        // ทดสอบ re-throw
        @try {
            safeProcessData(nil);
        }
        @catch (NSException *e) {
            NSLog(@"Re-caught: %@ - %@", e.name, e.reason);
        }
        
        // ทดสอบ data ปกติ
        @try {
            NSData *data = [@"Hello" dataUsingEncoding:NSUTF8StringEncoding];
            processData(data);
        }
        @catch (NSException *e) {
            NSLog(@"Unexpected error: %@", e.reason);
        }
    }
    return 0;
}
```

---

## 30.6 Custom Exception Classes

```objc
#import <Foundation/Foundation.h>

// Custom exception class
@interface AppException : NSException

@property (nonatomic, strong) NSError *underlyingError;
@property (nonatomic, assign) NSInteger errorCode;

+ (instancetype)exceptionWithCode:(NSInteger)code
                           reason:(NSString *)reason;

+ (instancetype)exceptionWithCode:(NSInteger)code
                           reason:(NSString *)reason
                  underlyingError:(NSError *)error;

@end

@implementation AppException

+ (instancetype)exceptionWithCode:(NSInteger)code
                           reason:(NSString *)reason {
    return [self exceptionWithCode:code reason:reason underlyingError:nil];
}

+ (instancetype)exceptionWithCode:(NSInteger)code
                           reason:(NSString *)reason
                  underlyingError:(NSError *)error {
    AppException *exception = [self exceptionWithName:@"AppException"
                                               reason:reason
                                             userInfo:error ? @{NSUnderlyingErrorKey: error} : nil];
    exception->_errorCode = code;
    exception->_underlyingError = error;
    return exception;
}

@end

// NetworkException
@interface NetworkException : AppException
@property (nonatomic, assign) NSInteger httpStatusCode;
@end

@implementation NetworkException
+ (instancetype)exceptionWithStatusCode:(NSInteger)statusCode
                                message:(NSString *)message {
    NetworkException *ex = [self exceptionWithCode:statusCode reason:message];
    ex->_httpStatusCode = statusCode;
    return ex;
}
@end

// ฟังก์ชันที่ใช้ custom exceptions
void fetchData(NSString *url) {
    if (url.length == 0) {
        @throw [AppException exceptionWithCode:1001 reason:@"URL ว่างเปล่า"];
    }
    
    // จำลอง network error
    if ([url hasPrefix:@"http://"]) {
        @throw [NetworkException exceptionWithStatusCode:403
                                                 message:@"HTTP ไม่ได้รับอนุญาต ใช้ HTTPS"];
    }
    
    NSLog(@"Fetching: %@", url);
}

int main(int argc, const char * argv[]) {
    @autoreleasepool {
        @try {
            fetchData(@"http://example.com/api"); // ใช้ HTTP
        }
        @catch (NetworkException *e) {
            NSLog(@"Network error (HTTP %ld): %@",
                  (long)e.httpStatusCode, e.reason);
        }
        @catch (AppException *e) {
            NSLog(@"App error (code %ld): %@",
                  (long)e.errorCode, e.reason);
        }
        @catch (NSException *e) {
            NSLog(@"Unknown error: %@ - %@", e.name, e.reason);
        }
        
        @try {
            fetchData(@""); // URL ว่าง
        }
        @catch (AppException *e) {
            NSLog(@"App error: %@", e.reason);
        }
    }
    return 0;
}
```

---

## 30.7 Error Propagation Patterns

### 30.7.1 Pattern 1: Pass NSError** ขึ้นไปใน call stack

```objc
#import <Foundation/Foundation.h>

NSString *const FileErrorDomain = @"com.myapp.file";

typedef NS_ENUM(NSInteger, FileErrorCode) {
    FileErrorCodeNotFound = 1,
    FileErrorCodePermission = 2,
    FileErrorCodeCorrupted = 3,
};

// ระดับล่างสุด: อ่านข้อมูล raw
NSData* readRawFile(NSString *path, NSError **error) {
    if (![[NSFileManager defaultManager] fileExistsAtPath:path]) {
        if (error) {
            *error = [NSError errorWithDomain:FileErrorDomain
                                         code:FileErrorCodeNotFound
                                     userInfo:@{
                NSLocalizedDescriptionKey: @"ไม่พบไฟล์",
                NSFilePathErrorKey: path
            }];
        }
        return nil;
    }
    
    NSData *data = [NSData dataWithContentsOfFile:path options:0 error:error];
    return data;
}

// ระดับกลาง: parse ข้อมูล
NSDictionary* parseJSONFile(NSString *path, NSError **error) {
    NSError *readError = nil;
    NSData *data = readRawFile(path, &readError);
    
    if (!data) {
        // Wrap the lower-level error
        if (error) {
            *error = [NSError errorWithDomain:FileErrorDomain
                                         code:FileErrorCodeNotFound
                                     userInfo:@{
                NSLocalizedDescriptionKey: @"ไม่สามารถอ่าน JSON file ได้",
                NSUnderlyingErrorKey: readError
            }];
        }
        return nil;
    }
    
    NSError *jsonError = nil;
    NSDictionary *json = [NSJSONSerialization JSONObjectWithData:data
                                                        options:0
                                                          error:&jsonError];
    if (!json) {
        if (error) {
            *error = [NSError errorWithDomain:FileErrorDomain
                                         code:FileErrorCodeCorrupted
                                     userInfo:@{
                NSLocalizedDescriptionKey: @"JSON ไม่ถูกต้อง",
                NSUnderlyingErrorKey: jsonError
            }];
        }
        return nil;
    }
    
    return json;
}

// ระดับบน: โหลด config
BOOL loadConfig(NSString *configPath, NSError **error) {
    NSDictionary *config = parseJSONFile(configPath, error);
    if (!config) {
        return NO;
    }
    
    NSLog(@"Config loaded: %@", config);
    return YES;
}

int main(int argc, const char * argv[]) {
    @autoreleasepool {
        NSError *error = nil;
        
        if (!loadConfig(@"/nonexistent/config.json", &error)) {
            NSLog(@"Load config failed: %@", error.localizedDescription);
            
            // ตรวจสอบ underlying error chain
            NSError *underlying = error.userInfo[NSUnderlyingErrorKey];
            while (underlying) {
                NSLog(@"  Caused by: %@ (code: %ld)",
                      underlying.localizedDescription,
                      (long)underlying.code);
                underlying = underlying.userInfo[NSUnderlyingErrorKey];
            }
        }
    }
    return 0;
}
```

### 30.7.2 Pattern 2: Result Pattern (ใช้ NSDictionary หรือ Custom Object)

```objc
#import <Foundation/Foundation.h>

// Result wrapper class
@interface Result : NSObject

@property (nonatomic, assign, readonly) BOOL isSuccess;
@property (nonatomic, strong, readonly) id value;
@property (nonatomic, strong, readonly) NSError *error;

+ (instancetype)successWithValue:(id)value;
+ (instancetype)failureWithError:(NSError *)error;

@end

@implementation Result

+ (instancetype)successWithValue:(id)value {
    Result *r = [[self alloc] init];
    r->_isSuccess = YES;
    r->_value = value;
    return r;
}

+ (instancetype)failureWithError:(NSError *)error {
    Result *r = [[self alloc] init];
    r->_isSuccess = NO;
    r->_error = error;
    return r;
}

@end

// ฟังก์ชันที่ return Result
Result* divideNumbers(double a, double b) {
    if (b == 0) {
        NSError *error = [NSError errorWithDomain:@"com.math"
                                             code:1
                                         userInfo:@{
            NSLocalizedDescriptionKey: @"หารด้วยศูนย์ไม่ได้"
        }];
        return [Result failureWithError:error];
    }
    return [Result successWithValue:@(a / b)];
}

Result* parsePositiveInt(NSString *str) {
    NSInteger value = [str integerValue];
    if (value <= 0 && ![str isEqualToString:@"0"]) {
        NSError *error = [NSError errorWithDomain:@"com.parse"
                                             code:1
                                         userInfo:@{
            NSLocalizedDescriptionKey: [NSString stringWithFormat:
                                        @"'%@' ไม่ใช่จำนวนเต็มบวก", str]
        }];
        return [Result failureWithError:error];
    }
    if (value < 0) {
        NSError *error = [NSError errorWithDomain:@"com.parse"
                                             code:2
                                         userInfo:@{
            NSLocalizedDescriptionKey: @"ค่าต้องเป็นบวก"
        }];
        return [Result failureWithError:error];
    }
    return [Result successWithValue:@(value)];
}

int main(int argc, const char * argv[]) {
    @autoreleasepool {
        // ทดสอบ Result pattern
        Result *r1 = divideNumbers(10, 3);
        if (r1.isSuccess) {
            NSLog(@"10 / 3 = %.4f", [r1.value doubleValue]);
        }
        
        Result *r2 = divideNumbers(5, 0);
        if (!r2.isSuccess) {
            NSLog(@"Error: %@", r2.error.localizedDescription);
        }
        
        NSArray *inputs = @[@"42", @"-5", @"abc", @"0"];
        for (NSString *input in inputs) {
            Result *r = parsePositiveInt(input);
            if (r.isSuccess) {
                NSLog(@"'%@' => %@", input, r.value);
            } else {
                NSLog(@"'%@' => Error: %@", input, r.error.localizedDescription);
            }
        }
    }
    return 0;
}
```

---

## 30.8 Error Handling ใน Real Scenarios

### 30.8.1 File Operations

```objc
#import <Foundation/Foundation.h>

// การอ่าน/เขียนไฟล์พร้อม error handling
BOOL writeStringToFile(NSString *content, NSString *filePath, NSError **error) {
    NSError *writeError = nil;
    BOOL success = [content writeToFile:filePath
                             atomically:YES
                               encoding:NSUTF8StringEncoding
                                  error:&writeError];
    if (!success && error) {
        *error = [NSError errorWithDomain:@"com.myapp.file"
                                     code:1
                                 userInfo:@{
            NSLocalizedDescriptionKey: @"ไม่สามารถเขียนไฟล์ได้",
            NSUnderlyingErrorKey: writeError,
            NSFilePathErrorKey: filePath
        }];
    }
    return success;
}

NSString* readStringFromFile(NSString *filePath, NSError **error) {
    NSError *readError = nil;
    NSString *content = [NSString stringWithContentsOfFile:filePath
                                                  encoding:NSUTF8StringEncoding
                                                     error:&readError];
    if (!content && error) {
        *error = [NSError errorWithDomain:@"com.myapp.file"
                                     code:2
                                 userInfo:@{
            NSLocalizedDescriptionKey: @"ไม่สามารถอ่านไฟล์ได้",
            NSUnderlyingErrorKey: readError,
            NSFilePathErrorKey: filePath
        }];
    }
    return content;
}

int main(int argc, const char * argv[]) {
    @autoreleasepool {
        NSString *tempDir = NSTemporaryDirectory();
        NSString *filePath = [tempDir stringByAppendingPathComponent:@"test_output.txt"];
        
        // เขียนไฟล์
        NSError *error = nil;
        NSString *content = @"สวัสดีจาก Objective-C!\nThis is a test file.";
        
        if (writeStringToFile(content, filePath, &error)) {
            NSLog(@"เขียนไฟล์สำเร็จที่: %@", filePath);
        } else {
            NSLog(@"เขียนไฟล์ล้มเหลว: %@", error.localizedDescription);
        }
        
        // อ่านไฟล์
        error = nil;
        NSString *readContent = readStringFromFile(filePath, &error);
        if (readContent) {
            NSLog(@"อ่านไฟล์สำเร็จ:\n%@", readContent);
        } else {
            NSLog(@"อ่านไฟล์ล้มเหลว: %@", error.localizedDescription);
        }
        
        // ลองอ่านไฟล์ที่ไม่มีอยู่
        error = nil;
        NSString *missing = readStringFromFile(@"/nonexistent/file.txt", &error);
        if (!missing) {
            NSLog(@"คาดหวังว่าจะล้มเหลว: %@", error.localizedDescription);
        }
        
        // cleanup
        [[NSFileManager defaultManager] removeItemAtPath:filePath error:nil];
    }
    return 0;
}
```

### 30.8.2 JSON Parsing

```objc
#import <Foundation/Foundation.h>

NSString *const JSONErrorDomain = @"com.myapp.json";

typedef NS_ENUM(NSInteger, JSONErrorCode) {
    JSONErrorCodeInvalidData     = 1,
    JSONErrorCodeMissingField    = 2,
    JSONErrorCodeWrongType       = 3,
    JSONErrorCodeConstraintFailed = 4,
};

// Parse และ validate JSON user data
NSDictionary* parseUserJSON(NSData *jsonData, NSError **error) {
    if (!jsonData || jsonData.length == 0) {
        if (error) {
            *error = [NSError errorWithDomain:JSONErrorDomain
                                         code:JSONErrorCodeInvalidData
                                     userInfo:@{
                NSLocalizedDescriptionKey: @"JSON data ว่างเปล่า"
            }];
        }
        return nil;
    }
    
    NSError *parseError = nil;
    id parsed = [NSJSONSerialization JSONObjectWithData:jsonData
                                               options:0
                                                 error:&parseError];
    if (!parsed) {
        if (error) {
            *error = [NSError errorWithDomain:JSONErrorDomain
                                         code:JSONErrorCodeInvalidData
                                     userInfo:@{
                NSLocalizedDescriptionKey: @"JSON ไม่ถูกต้อง",
                NSUnderlyingErrorKey: parseError
            }];
        }
        return nil;
    }
    
    if (![parsed isKindOfClass:[NSDictionary class]]) {
        if (error) {
            *error = [NSError errorWithDomain:JSONErrorDomain
                                         code:JSONErrorCodeWrongType
                                     userInfo:@{
                NSLocalizedDescriptionKey: @"คาดหวัง JSON object ไม่ใช่ array"
            }];
        }
        return nil;
    }
    
    NSDictionary *dict = (NSDictionary *)parsed;
    
    // ตรวจสอบ required fields
    NSArray *requiredFields = @[@"id", @"name", @"email"];
    for (NSString *field in requiredFields) {
        if (!dict[field]) {
            if (error) {
                *error = [NSError errorWithDomain:JSONErrorDomain
                                             code:JSONErrorCodeMissingField
                                         userInfo:@{
                    NSLocalizedDescriptionKey: [NSString stringWithFormat:
                                                @"ขาด field '%@'", field]
                }];
            }
            return nil;
        }
    }
    
    // ตรวจสอบ constraint
    NSString *email = dict[@"email"];
    if (![email containsString:@"@"]) {
        if (error) {
            *error = [NSError errorWithDomain:JSONErrorDomain
                                         code:JSONErrorCodeConstraintFailed
                                     userInfo:@{
                NSLocalizedDescriptionKey: @"รูปแบบ email ไม่ถูกต้อง"
            }];
        }
        return nil;
    }
    
    return dict;
}

int main(int argc, const char * argv[]) {
    @autoreleasepool {
        NSError *error = nil;
        
        // Test 1: valid JSON
        NSString *validJSON = @"{\"id\":1,\"name\":\"สมชาย\",\"email\":\"somchai@example.com\"}";
        NSDictionary *user = parseUserJSON([validJSON dataUsingEncoding:NSUTF8StringEncoding],
                                           &error);
        if (user) {
            NSLog(@"Valid user: id=%@, name=%@, email=%@",
                  user[@"id"], user[@"name"], user[@"email"]);
        } else {
            NSLog(@"Error: %@", error.localizedDescription);
        }
        
        // Test 2: missing field
        error = nil;
        NSString *missingField = @"{\"id\":2,\"name\":\"สมหญิง\"}"; // ขาด email
        user = parseUserJSON([missingField dataUsingEncoding:NSUTF8StringEncoding], &error);
        if (!user) {
            NSLog(@"Missing field error: %@", error.localizedDescription);
        }
        
        // Test 3: invalid JSON
        error = nil;
        user = parseUserJSON([@"not json" dataUsingEncoding:NSUTF8StringEncoding], &error);
        if (!user) {
            NSLog(@"Parse error: %@", error.localizedDescription);
        }
        
        // Test 4: invalid email format
        error = nil;
        NSString *badEmail = @"{\"id\":3,\"name\":\"ทดสอบ\",\"email\":\"notvalid\"}";
        user = parseUserJSON([badEmail dataUsingEncoding:NSUTF8StringEncoding], &error);
        if (!user) {
            NSLog(@"Validation error: %@", error.localizedDescription);
        }
    }
    return 0;
}
```

### 30.8.3 Database Operations (จำลอง)

```objc
#import <Foundation/Foundation.h>

NSString *const DBErrorDomain = @"com.myapp.database";

typedef NS_ENUM(NSInteger, DBErrorCode) {
    DBErrorCodeConnectionFailed  = 100,
    DBErrorCodeQueryFailed       = 101,
    DBErrorCodeNotFound          = 102,
    DBErrorCodeDuplicateKey      = 103,
    DBErrorCodeConstraintFailed  = 104,
};

// Mock database
@interface MockDatabase : NSObject

@property (nonatomic, strong) NSMutableDictionary *store;
@property (nonatomic, assign) BOOL isConnected;

- (BOOL)connectWithError:(NSError **)error;
- (BOOL)insertUser:(NSDictionary *)user error:(NSError **)error;
- (NSDictionary *)getUserWithID:(NSInteger)userID error:(NSError **)error;
- (BOOL)updateUser:(NSDictionary *)user error:(NSError **)error;
- (BOOL)deleteUserWithID:(NSInteger)userID error:(NSError **)error;

@end

@implementation MockDatabase

- (instancetype)init {
    self = [super init];
    if (self) {
        _store = [NSMutableDictionary dictionary];
        _isConnected = NO;
    }
    return self;
}

- (BOOL)connectWithError:(NSError **)error {
    // จำลองการเชื่อมต่อ
    _isConnected = YES;
    NSLog(@"Database connected");
    return YES;
}

- (BOOL)insertUser:(NSDictionary *)user error:(NSError **)error {
    if (!_isConnected) {
        if (error) {
            *error = [NSError errorWithDomain:DBErrorDomain
                                         code:DBErrorCodeConnectionFailed
                                     userInfo:@{NSLocalizedDescriptionKey: @"ยังไม่ได้เชื่อมต่อ database"}];
        }
        return NO;
    }
    
    NSNumber *userID = user[@"id"];
    if (!userID) {
        if (error) {
            *error = [NSError errorWithDomain:DBErrorDomain
                                         code:DBErrorCodeConstraintFailed
                                     userInfo:@{NSLocalizedDescriptionKey: @"ต้องมี id"}];
        }
        return NO;
    }
    
    if (_store[userID]) {
        if (error) {
            *error = [NSError errorWithDomain:DBErrorDomain
                                         code:DBErrorCodeDuplicateKey
                                     userInfo:@{
                NSLocalizedDescriptionKey: [NSString stringWithFormat:
                                            @"User ID %@ มีอยู่แล้ว", userID]
            }];
        }
        return NO;
    }
    
    _store[userID] = [user copy];
    return YES;
}

- (NSDictionary *)getUserWithID:(NSInteger)userID error:(NSError **)error {
    if (!_isConnected) {
        if (error) {
            *error = [NSError errorWithDomain:DBErrorDomain
                                         code:DBErrorCodeConnectionFailed
                                     userInfo:@{NSLocalizedDescriptionKey: @"ยังไม่ได้เชื่อมต่อ database"}];
        }
        return nil;
    }
    
    NSDictionary *user = _store[@(userID)];
    if (!user) {
        if (error) {
            *error = [NSError errorWithDomain:DBErrorDomain
                                         code:DBErrorCodeNotFound
                                     userInfo:@{
                NSLocalizedDescriptionKey: [NSString stringWithFormat:
                                            @"ไม่พบ user ID %ld", (long)userID]
            }];
        }
        return nil;
    }
    
    return user;
}

- (BOOL)updateUser:(NSDictionary *)user error:(NSError **)error {
    NSNumber *userID = user[@"id"];
    if (!_store[userID]) {
        if (error) {
            *error = [NSError errorWithDomain:DBErrorDomain
                                         code:DBErrorCodeNotFound
                                     userInfo:@{
                NSLocalizedDescriptionKey: [NSString stringWithFormat:
                                            @"ไม่พบ user ID %@ สำหรับ update", userID]
            }];
        }
        return NO;
    }
    
    NSMutableDictionary *existing = [_store[userID] mutableCopy];
    [existing addEntriesFromDictionary:user];
    _store[userID] = [existing copy];
    return YES;
}

- (BOOL)deleteUserWithID:(NSInteger)userID error:(NSError **)error {
    if (!_store[@(userID)]) {
        if (error) {
            *error = [NSError errorWithDomain:DBErrorDomain
                                         code:DBErrorCodeNotFound
                                     userInfo:@{
                NSLocalizedDescriptionKey: [NSString stringWithFormat:
                                            @"ไม่พบ user ID %ld สำหรับลบ", (long)userID]
            }];
        }
        return NO;
    }
    
    [_store removeObjectForKey:@(userID)];
    return YES;
}

@end

int main(int argc, const char * argv[]) {
    @autoreleasepool {
        MockDatabase *db = [[MockDatabase alloc] init];
        NSError *error = nil;
        
        // เชื่อมต่อ
        [db connectWithError:&error];
        
        // Insert users
        NSArray *users = @[
            @{@"id": @1, @"name": @"สมชาย", @"email": @"somchai@example.com"},
            @{@"id": @2, @"name": @"สมหญิง", @"email": @"somying@example.com"},
        ];
        
        for (NSDictionary *user in users) {
            error = nil;
            if ([db insertUser:user error:&error]) {
                NSLog(@"Insert user %@ สำเร็จ", user[@"name"]);
            } else {
                NSLog(@"Insert error: %@", error.localizedDescription);
            }
        }
        
        // Insert ซ้ำ
        error = nil;
        if (![db insertUser:@{@"id": @1, @"name": @"ซ้ำ"} error:&error]) {
            NSLog(@"Expected duplicate error: %@", error.localizedDescription);
        }
        
        // Get user
        error = nil;
        NSDictionary *user = [db getUserWithID:1 error:&error];
        if (user) {
            NSLog(@"Get user: %@", user[@"name"]);
        }
        
        // Get user ที่ไม่มี
        error = nil;
        user = [db getUserWithID:999 error:&error];
        if (!user) {
            NSLog(@"Expected not found: %@", error.localizedDescription);
        }
        
        // Update
        error = nil;
        if ([db updateUser:@{@"id": @1, @"email": @"new@example.com"} error:&error]) {
            NSDictionary *updated = [db getUserWithID:1 error:nil];
            NSLog(@"Updated email: %@", updated[@"email"]);
        }
        
        // Delete
        error = nil;
        if ([db deleteUserWithID:2 error:&error]) {
            NSLog(@"User 2 ถูกลบแล้ว");
        }
    }
    return 0;
}
```

---

## 30.9 Custom Error Classes

```objc
#import <Foundation/Foundation.h>

// Protocol สำหรับ custom errors
@protocol AppErrorProtocol <NSObject>
@property (nonatomic, readonly) NSString *errorTitle;
@property (nonatomic, readonly) NSString *errorMessage;
@property (nonatomic, readonly) BOOL isRecoverable;
@end

// Base custom error
@interface AppError : NSError <AppErrorProtocol>

- (instancetype)initWithCode:(NSInteger)code
                       title:(NSString *)title
                     message:(NSString *)message
                 recoverable:(BOOL)recoverable;

@end

@implementation AppError

NSString *const AppErrorDomainKey = @"com.myapp";

- (instancetype)initWithCode:(NSInteger)code
                       title:(NSString *)title
                     message:(NSString *)message
                 recoverable:(BOOL)recoverable {
    NSDictionary *userInfo = @{
        NSLocalizedDescriptionKey: message,
        @"title": title,
        @"recoverable": @(recoverable),
    };
    self = [super initWithDomain:AppErrorDomainKey code:code userInfo:userInfo];
    return self;
}

- (NSString *)errorTitle {
    return self.userInfo[@"title"];
}

- (NSString *)errorMessage {
    return self.localizedDescription;
}

- (BOOL)isRecoverable {
    return [self.userInfo[@"recoverable"] boolValue];
}

@end

// Specific error types
@interface NetworkError : AppError
@property (nonatomic, assign, readonly) NSInteger httpStatus;
- (instancetype)initWithHttpStatus:(NSInteger)status message:(NSString *)message;
@end

@implementation NetworkError
- (instancetype)initWithHttpStatus:(NSInteger)status message:(NSString *)message {
    self = [super initWithCode:status
                         title:@"Network Error"
                       message:message
                   recoverable:YES];
    _httpStatus = status;
    return self;
}
@end

@interface ValidationError : AppError
@property (nonatomic, strong, readonly) NSString *fieldName;
- (instancetype)initWithField:(NSString *)field message:(NSString *)message;
@end

@implementation ValidationError
- (instancetype)initWithField:(NSString *)field message:(NSString *)message {
    self = [super initWithCode:422
                         title:@"Validation Error"
                       message:message
                   recoverable:YES];
    _fieldName = field;
    return self;
}
@end

// ฟังก์ชันสาธิต
BOOL submitForm(NSDictionary *formData, NSError **error) {
    // Validate fields
    if ([formData[@"username"] length] < 3) {
        if (error) {
            *error = [[ValidationError alloc] initWithField:@"username"
                                                    message:@"ชื่อผู้ใช้ต้องมีอย่างน้อย 3 ตัวอักษร"];
        }
        return NO;
    }
    
    if (![formData[@"email"] containsString:@"@"]) {
        if (error) {
            *error = [[ValidationError alloc] initWithField:@"email"
                                                    message:@"รูปแบบ email ไม่ถูกต้อง"];
        }
        return NO;
    }
    
    // จำลอง network error
    if ([formData[@"network_error"] boolValue]) {
        if (error) {
            *error = [[NetworkError alloc] initWithHttpStatus:503
                                                      message:@"Server ไม่พร้อมใช้งาน"];
        }
        return NO;
    }
    
    return YES;
}

int main(int argc, const char * argv[]) {
    @autoreleasepool {
        NSError *error = nil;
        
        // Test 1: validation error
        NSDictionary *badForm = @{@"username": @"ab", @"email": @"test@test.com"};
        if (!submitForm(badForm, &error)) {
            if ([error isKindOfClass:[ValidationError class]]) {
                ValidationError *ve = (ValidationError *)error;
                NSLog(@"Validation error on field '%@': %@",
                      ve.fieldName, ve.errorMessage);
                NSLog(@"Is recoverable: %@", ve.isRecoverable ? @"yes" : @"no");
            }
        }
        
        // Test 2: network error
        error = nil;
        NSDictionary *networkFailForm = @{@"username": @"johndoe",
                                           @"email": @"john@test.com",
                                           @"network_error": @YES};
        if (!submitForm(networkFailForm, &error)) {
            if ([error isKindOfClass:[NetworkError class]]) {
                NetworkError *ne = (NetworkError *)error;
                NSLog(@"Network error HTTP %ld: %@",
                      (long)ne.httpStatus, ne.errorTitle);
            }
        }
        
        // Test 3: success
        error = nil;
        NSDictionary *validForm = @{@"username": @"johndoe",
                                     @"email": @"john@test.com"};
        if (submitForm(validForm, &error)) {
            NSLog(@"Form submitted successfully!");
        }
    }
    return 0;
}
```

---

## 30.10 Error Handling Best Practices

```objc
#import <Foundation/Foundation.h>

// ✅ 1. ตรวจสอบ return value เสมอ ไม่ใช่แค่ error
void goodPattern(void) {
    NSError *error = nil;
    NSString *content = [NSString stringWithContentsOfFile:@"/some/file"
                                                  encoding:NSUTF8StringEncoding
                                                     error:&error];
    if (content) { // ตรวจสอบ return value
        NSLog(@"Got content");
    } else {
        NSLog(@"Error: %@", error);
    }
}

// ❌ 2. อย่า check error ก่อน return value
void badPattern(void) {
    NSError *error = nil;
    NSString *content = [NSString stringWithContentsOfFile:@"/some/file"
                                                  encoding:NSUTF8StringEncoding
                                                     error:&error];
    if (error) { // บางครั้ง error ถูก set แม้จะสำเร็จ
        NSLog(@"Error: %@", error);
    }
}

// ✅ 3. เสมอตรวจสอบ error != nil ก่อน dereference
BOOL safeSetError(NSError **error, NSString *domain, NSInteger code, NSString *desc) {
    if (error) { // ตรวจก่อน!
        *error = [NSError errorWithDomain:domain
                                     code:code
                                 userInfo:@{NSLocalizedDescriptionKey: desc}];
    }
    return NO;
}

// ✅ 4. Log errors ที่ layer ที่เหมาะสม
BOOL processWithLogging(NSError **outError) {
    NSError *internalError = nil;
    BOOL ok = [@"test" writeToFile:@"/readonly/file"
                        atomically:YES
                          encoding:NSUTF8StringEncoding
                             error:&internalError];
    if (!ok) {
        // Log ที่นี่เพื่อ debugging
        NSLog(@"[DEBUG] File write failed: %@", internalError.localizedDescription);
        
        // ส่งต่อ (ไม่ต้อง log ซ้ำที่ caller)
        if (outError) {
            *outError = internalError;
        }
    }
    return ok;
}

// ✅ 5. Error recovery
void handleWithRecovery(NSError *error) {
    if ([error.domain isEqualToString:NSURLErrorDomain]) {
        switch (error.code) {
            case NSURLErrorNotConnectedToInternet:
                NSLog(@"ไม่มีอินเทอร์เน็ต - แนะนำ: เชื่อมต่อ WiFi หรือ Mobile data");
                break;
            case NSURLErrorTimedOut:
                NSLog(@"Timeout - แนะนำ: ลองใหม่อีกครั้ง");
                break;
            default:
                NSLog(@"Network error %ld: %@", (long)error.code,
                      error.localizedDescription);
        }
    } else {
        NSLog(@"Unknown error: %@", error.localizedDescription);
    }
}

int main(int argc, const char * argv[]) {
    @autoreleasepool {
        goodPattern();
        
        NSError *e = nil;
        safeSetError(&e, @"com.test", 1, @"Test error");
        NSLog(@"Error set: %@", e.localizedDescription);
        
        // ทดสอบ NULL pointer (ไม่ crash)
        safeSetError(NULL, @"com.test", 1, @"Test error");
        NSLog(@"NULL pointer test passed");
    }
    return 0;
}
```

---

## 30.11 สรุป: เมื่อไรใช้อะไร

```objc
#import <Foundation/Foundation.h>

/*
 เมื่อไรใช้ NSError:
 ✅ File operations (อ่าน/เขียน ไฟล์)
 ✅ Network calls
 ✅ JSON parsing
 ✅ Database queries
 ✅ User input validation
 ✅ Permission/Authorization failures
 ✅ ทุกอย่างที่เป็น "expected error" (คาดว่าจะเกิดได้)
 
 เมื่อไรใช้ @try/@catch:
 ✅ Programmer errors (bugs ที่ไม่ควรเกิด)
 ✅ Integration กับ C++ exceptions
 ✅ KVO/KVC ที่อาจ throw
 ✅ ใช้ใน main() เพื่อ catch unhandled exceptions
 ✅ ทำ cleanup ที่จำเป็นด้วย @finally
 
 ❌ ห้ามใช้ exceptions สำหรับ flow control ปกติ
 ❌ ห้าม catch NSException แล้วเงียบ (silent swallow)
*/

// Pattern ที่ดีสำหรับ main()
int main(int argc, const char * argv[]) {
    @autoreleasepool {
        @try {
            // โปรแกรมหลัก
            NSLog(@"Program running...");
            
            // ... code ปกติที่ใช้ NSError สำหรับ error handling
            
        }
        @catch (NSException *exception) {
            // Catch ที่ main() เพื่อป้องกัน crash และ log
            NSLog(@"FATAL: Unhandled exception: %@\n%@",
                  exception.name, exception.reason);
            NSLog(@"Stack trace:\n%@", exception.callStackSymbols);
            return 1;
        }
        @finally {
            NSLog(@"Program cleanup");
        }
    }
    return 0;
}
```

---

## แบบฝึกหัด (Practice Exercises)

### แบบฝึกหัดที่ 1: สร้าง NSError ด้วย userInfo ครบถ้วน
สร้าง NSError สำหรับ "Password Too Short" ที่มีทั้ง description, reason, และ recovery suggestion

```objc
// เฉลย
NSError *passwordError = [NSError errorWithDomain:@"com.myapp.auth"
                                              code:1001
                                          userInfo:@{
    NSLocalizedDescriptionKey: @"รหัสผ่านสั้นเกินไป",
    NSLocalizedFailureReasonErrorKey: @"รหัสผ่านต้องมีอย่างน้อย 8 ตัวอักษร",
    NSLocalizedRecoverySuggestionErrorKey: @"โปรดตั้งรหัสผ่านที่มีอย่างน้อย 8 ตัวอักษร รวมตัวเลขและตัวพิมพ์ใหญ่"
}];

NSLog(@"Error: %@", passwordError.localizedDescription);
NSLog(@"Reason: %@", passwordError.localizedFailureReason);
NSLog(@"Suggestion: %@", passwordError.localizedRecoverySuggestion);
```

### แบบฝึกหัดที่ 2: ฟังก์ชัน validate password
เขียนฟังก์ชัน `validatePassword(NSString *password, NSError **error)` ที่ตรวจสอบว่ารหัสผ่านมีความยาวอย่างน้อย 8 ตัว มีตัวเลขอย่างน้อย 1 ตัว

```objc
typedef NS_ENUM(NSInteger, PasswordErrorCode) {
    PasswordErrorTooShort = 1,
    PasswordErrorNoDigit = 2,
    PasswordErrorNoUppercase = 3,
};

BOOL validatePassword(NSString *password, NSError **error) {
    if (password.length < 8) {
        if (error) {
            *error = [NSError errorWithDomain:@"com.auth"
                                         code:PasswordErrorTooShort
                                     userInfo:@{
                NSLocalizedDescriptionKey: @"รหัสผ่านสั้นเกินไป (ต้องการ 8+ ตัว)"
            }];
        }
        return NO;
    }
    
    BOOL hasDigit = NO;
    for (NSUInteger i = 0; i < password.length; i++) {
        unichar c = [password characterAtIndex:i];
        if (c >= '0' && c <= '9') { hasDigit = YES; break; }
    }
    
    if (!hasDigit) {
        if (error) {
            *error = [NSError errorWithDomain:@"com.auth"
                                         code:PasswordErrorNoDigit
                                     userInfo:@{
                NSLocalizedDescriptionKey: @"รหัสผ่านต้องมีตัวเลขอย่างน้อย 1 ตัว"
            }];
        }
        return NO;
    }
    
    return YES;
}

// ทดสอบ
NSError *err = nil;
NSArray *passwords = @[@"abc", @"abcdefgh", @"abcdefg8", @"Passw0rd!"];
for (NSString *pw in passwords) {
    err = nil;
    BOOL valid = validatePassword(pw, &err);
    NSLog(@"'%@': %@", pw, valid ? @"valid ✓" : err.localizedDescription);
}
```

### แบบฝึกหัดที่ 3: Error Chain
สร้าง function chain ที่ส่ง error ขึ้นไปพร้อม underlying error

```objc
// Layer 1: Low-level
NSData* fetchFromNetwork(NSString *url, NSError **error) {
    // จำลอง network failure
    if ([url containsString:@"fail"]) {
        if (error) {
            *error = [NSError errorWithDomain:NSURLErrorDomain
                                         code:NSURLErrorNotConnectedToInternet
                                     userInfo:@{NSLocalizedDescriptionKey: @"ไม่มีอินเทอร์เน็ต"}];
        }
        return nil;
    }
    return [@"mock data" dataUsingEncoding:NSUTF8StringEncoding];
}

// Layer 2: Mid-level
NSDictionary* fetchUserData(NSInteger userID, NSError **error) {
    NSString *url = [NSString stringWithFormat:@"https://api.example.com/users/%ld", (long)userID];
    NSError *networkError = nil;
    NSData *data = fetchFromNetwork(url, &networkError);
    
    if (!data) {
        if (error) {
            *error = [NSError errorWithDomain:@"com.api"
                                         code:500
                                     userInfo:@{
                NSLocalizedDescriptionKey: @"ไม่สามารถดึงข้อมูล user ได้",
                NSUnderlyingErrorKey: networkError
            }];
        }
        return nil;
    }
    return @{@"id": @(userID), @"name": @"Test User"};
}

// Layer 3: High-level
BOOL loadAndDisplayUser(NSInteger userID, NSError **error) {
    NSError *fetchError = nil;
    NSDictionary *user = fetchUserData(userID, &fetchError);
    
    if (!user) {
        if (error) {
            *error = [NSError errorWithDomain:@"com.app"
                                         code:1
                                     userInfo:@{
                NSLocalizedDescriptionKey: @"ไม่สามารถแสดงข้อมูล user ได้",
                NSUnderlyingErrorKey: fetchError
            }];
        }
        return NO;
    }
    
    NSLog(@"User: %@", user[@"name"]);
    return YES;
}

// ทดสอบ
NSError *err = nil;
// จำลอง URL ที่ fail
if (!loadAndDisplayUser(999, &err)) {
    NSLog(@"Top-level error: %@", err.localizedDescription);
    
    // traverse error chain
    NSError *current = err;
    int level = 0;
    while (current.userInfo[NSUnderlyingErrorKey]) {
        current = current.userInfo[NSUnderlyingErrorKey];
        level++;
        NSLog(@"Level %d: %@ (code: %ld)", level,
              current.localizedDescription, (long)current.code);
    }
}
```

### แบบฝึกหัดที่ 4: @try/@catch สำหรับ KVC
ใช้ @try/@catch กับ Key-Value Coding ที่อาจ throw exception

```objc
@interface Person : NSObject
@property (nonatomic, strong) NSString *name;
@property (nonatomic, assign) NSInteger age;
@end

@implementation Person
@end

// ฟังก์ชัน safe KVC get
id safeValueForKey(id object, NSString *key) {
    @try {
        return [object valueForKey:key];
    }
    @catch (NSException *e) {
        NSLog(@"KVC exception for key '%@': %@", key, e.reason);
        return nil;
    }
}

// ทดสอบ
Person *person = [[Person alloc] init];
person.name = @"สมชาย";
person.age = 30;

NSLog(@"name: %@", safeValueForKey(person, @"name"));
NSLog(@"age: %@", safeValueForKey(person, @"age"));
NSLog(@"invalid: %@", safeValueForKey(person, @"nonexistentKey")); // จะ log exception แล้วคืน nil
```

### แบบฝึกหัดที่ 5: Retry Logic
เขียน function ที่ retry หากเกิด recoverable error

```objc
typedef BOOL (^RetryableOperation)(NSError **error);

BOOL performWithRetry(RetryableOperation operation,
                       NSInteger maxRetries,
                       NSError **finalError) {
    NSError *error = nil;
    for (NSInteger attempt = 1; attempt <= maxRetries; attempt++) {
        error = nil;
        BOOL success = operation(&error);
        if (success) {
            if (attempt > 1) {
                NSLog(@"สำเร็จในครั้งที่ %ld", (long)attempt);
            }
            return YES;
        }
        NSLog(@"ครั้งที่ %ld ล้มเหลว: %@", (long)attempt, error.localizedDescription);
        if (attempt < maxRetries) {
            NSLog(@"กำลังลองใหม่...");
        }
    }
    if (finalError) *finalError = error;
    return NO;
}

// ทดสอบ
__block NSInteger callCount = 0;

NSError *finalErr = nil;
BOOL result = performWithRetry(^BOOL(NSError **err) {
    callCount++;
    if (callCount < 3) { // ล้มเหลว 2 ครั้งแรก
        if (err) {
            *err = [NSError errorWithDomain:@"com.net"
                                       code:503
                                   userInfo:@{NSLocalizedDescriptionKey: @"Server ยุ่ง"}];
        }
        return NO;
    }
    return YES; // สำเร็จในครั้งที่ 3
}, 3, &finalErr);

NSLog(@"Result: %@ (tried %ld times)", result ? @"success" : @"failed", (long)callCount);
if (!result) {
    NSLog(@"Final error: %@", finalErr.localizedDescription);
}
```

### แบบฝึกหัดที่ 6: Error Mapping
เขียน function แปลง HTTP status code เป็น NSError

```objc
NSError* errorFromHTTPStatus(NSInteger statusCode, NSString *responseBody) {
    NSString *description;
    NSInteger errorCode = statusCode;
    
    switch (statusCode) {
        case 400: description = @"Bad Request - คำขอไม่ถูกต้อง"; break;
        case 401: description = @"Unauthorized - ต้องเข้าสู่ระบบก่อน"; break;
        case 403: description = @"Forbidden - ไม่มีสิทธิ์"; break;
        case 404: description = @"Not Found - ไม่พบสิ่งที่ต้องการ"; break;
        case 409: description = @"Conflict - ข้อมูลขัดแย้ง"; break;
        case 422: description = @"Unprocessable - ข้อมูลไม่ถูกต้อง"; break;
        case 429: description = @"Too Many Requests - ส่งคำขอมากเกินไป"; break;
        case 500: description = @"Internal Server Error - Server มีปัญหา"; break;
        case 503: description = @"Service Unavailable - Server ไม่พร้อมใช้งาน"; break;
        default:  description = [NSString stringWithFormat:@"HTTP Error %ld", (long)statusCode];
    }
    
    NSMutableDictionary *userInfo = [NSMutableDictionary dictionary];
    userInfo[NSLocalizedDescriptionKey] = description;
    if (responseBody.length > 0) {
        userInfo[@"responseBody"] = responseBody;
    }
    
    return [NSError errorWithDomain:NSURLErrorDomain
                               code:errorCode
                           userInfo:[userInfo copy]];
}

// ทดสอบ
NSArray *statuses = @[@400, @401, @404, @500];
for (NSNumber *status in statuses) {
    NSError *error = errorFromHTTPStatus([status integerValue], @"");
    NSLog(@"HTTP %ld: %@", (long)[status integerValue], error.localizedDescription);
}
```

### แบบฝึกหัดที่ 7: Error Accumulator
เก็บ errors หลาย ๆ ตัวรวมกัน (multi-error validation)

```objc
@interface ErrorAccumulator : NSObject
@property (nonatomic, strong, readonly) NSArray<NSError *> *errors;
@property (nonatomic, assign, readonly) BOOL hasErrors;
- (void)addError:(NSError *)error;
- (NSError *)combinedError;
@end

@implementation ErrorAccumulator

- (instancetype)init {
    self = [super init];
    if (self) {
        _errors = [NSArray array];
    }
    return self;
}

- (void)addError:(NSError *)error {
    _errors = [_errors arrayByAddingObject:error];
}

- (BOOL)hasErrors {
    return _errors.count > 0;
}

- (NSError *)combinedError {
    if (!self.hasErrors) return nil;
    
    NSMutableArray *descriptions = [NSMutableArray array];
    for (NSError *e in _errors) {
        [descriptions addObject:e.localizedDescription];
    }
    
    NSString *combinedDesc = [descriptions componentsJoinedByString:@"\n• "];
    return [NSError errorWithDomain:@"com.validation"
                               code:422
                           userInfo:@{
        NSLocalizedDescriptionKey: [@"• " stringByAppendingString:combinedDesc],
        @"errors": _errors
    }];
}

@end

// ทดสอบ validation form
void validateRegistrationForm(NSDictionary *form, ErrorAccumulator *acc) {
    if ([form[@"username"] length] < 3) {
        [acc addError:[NSError errorWithDomain:@"com.v" code:1
                                     userInfo:@{NSLocalizedDescriptionKey: @"ชื่อผู้ใช้ต้องมีอย่างน้อย 3 ตัว"}]];
    }
    if (![form[@"email"] containsString:@"@"]) {
        [acc addError:[NSError errorWithDomain:@"com.v" code:2
                                     userInfo:@{NSLocalizedDescriptionKey: @"รูปแบบ email ไม่ถูกต้อง"}]];
    }
    if ([form[@"password"] length] < 8) {
        [acc addError:[NSError errorWithDomain:@"com.v" code:3
                                     userInfo:@{NSLocalizedDescriptionKey: @"รหัสผ่านต้องมีอย่างน้อย 8 ตัว"}]];
    }
    if (![form[@"password"] isEqualToString:form[@"confirmPassword"]]) {
        [acc addError:[NSError errorWithDomain:@"com.v" code:4
                                     userInfo:@{NSLocalizedDescriptionKey: @"รหัสผ่านไม่ตรงกัน"}]];
    }
}

ErrorAccumulator *acc = [[ErrorAccumulator alloc] init];
NSDictionary *badForm = @{
    @"username": @"ab",
    @"email": @"notvalid",
    @"password": @"123",
    @"confirmPassword": @"456"
};

validateRegistrationForm(badForm, acc);

if (acc.hasErrors) {
    NSLog(@"มีข้อผิดพลาด %lu รายการ:", (unsigned long)acc.errors.count);
    NSLog(@"%@", acc.combinedError.localizedDescription);
}
```

### แบบฝึกหัดที่ 8: @finally สำหรับ Resource Cleanup

```objc
// จำลอง resource ที่ต้องปล่อยเสมอ
@interface MockFileHandle : NSObject
@property (nonatomic, strong) NSString *path;
@property (nonatomic, assign) BOOL isOpen;
- (instancetype)openFile:(NSString *)path;
- (void)close;
- (NSString *)readLine;
@end

@implementation MockFileHandle
- (instancetype)openFile:(NSString *)path {
    _path = path;
    _isOpen = YES;
    NSLog(@"Opened: %@", path);
    return self;
}

- (void)close {
    if (_isOpen) {
        _isOpen = NO;
        NSLog(@"Closed: %@", _path);
    }
}

- (NSString *)readLine {
    if (!_isOpen) {
        @throw [NSException exceptionWithName:@"FileException"
                                       reason:@"File is not open"
                                     userInfo:nil];
    }
    return @"mock line content";
}
@end

// ใช้ @finally เพื่อ ensure ปิดไฟล์เสมอ
void processFileSafely(NSString *path) {
    MockFileHandle *handle = [[MockFileHandle alloc] openFile:path];
    
    @try {
        NSString *line = [handle readLine];
        NSLog(@"Read: %@", line);
        
        // จำลอง error ระหว่างประมวลผล
        if ([path containsString:@"error"]) {
            @throw [NSException exceptionWithName:@"ProcessingException"
                                           reason:@"Processing failed"
                                         userInfo:nil];
        }
        
        NSLog(@"Processing complete");
    }
    @catch (NSException *e) {
        NSLog(@"Error during processing: %@", e.reason);
    }
    @finally {
        [handle close]; // ปิดไฟล์เสมอ ไม่ว่าจะเกิด exception หรือไม่
    }
}

processFileSafely(@"/normal/file.txt");
NSLog(@"---");
processFileSafely(@"/error/file.txt");
```

### แบบฝึกหัดที่ 9: Custom Error Domain Constants
สร้าง error domain และ codes ที่ถูกต้องสำหรับ app

```objc
// ไฟล์ AppErrors.h (จำลอง)
NSString *const AuthErrorDomain        = @"com.myapp.auth";
NSString *const PaymentErrorDomain     = @"com.myapp.payment";
NSString *const StorageErrorDomain     = @"com.myapp.storage";

typedef NS_ENUM(NSInteger, AuthErrorCode) {
    AuthErrorInvalidCredentials = 1001,
    AuthErrorAccountLocked      = 1002,
    AuthErrorSessionExpired     = 1003,
    AuthErrorTwoFactorRequired  = 1004,
};

typedef NS_ENUM(NSInteger, PaymentErrorCode) {
    PaymentErrorInsufficientFunds   = 2001,
    PaymentErrorCardDeclined        = 2002,
    PaymentErrorExpiredCard         = 2003,
    PaymentErrorInvalidCardNumber   = 2004,
};

// สร้าง helper functions
NSError* authError(AuthErrorCode code, NSString *message) {
    return [NSError errorWithDomain:AuthErrorDomain code:code
                           userInfo:@{NSLocalizedDescriptionKey: message}];
}

NSError* paymentError(PaymentErrorCode code, NSString *message) {
    return [NSError errorWithDomain:PaymentErrorDomain code:code
                           userInfo:@{NSLocalizedDescriptionKey: message}];
}

// ทดสอบ
NSError *e1 = authError(AuthErrorInvalidCredentials, @"อีเมลหรือรหัสผ่านไม่ถูกต้อง");
NSError *e2 = paymentError(PaymentErrorCardDeclined, @"บัตรถูกปฏิเสธ โปรดติดต่อธนาคาร");

NSLog(@"Auth error: [%@] code=%ld - %@",
      e1.domain, (long)e1.code, e1.localizedDescription);
NSLog(@"Payment error: [%@] code=%ld - %@",
      e2.domain, (long)e2.code, e2.localizedDescription);

// ตรวจสอบ error type
if ([e1.domain isEqualToString:AuthErrorDomain] &&
    e1.code == AuthErrorInvalidCredentials) {
    NSLog(@"แสดง UI: กรอกข้อมูล login ใหม่");
}
```

### แบบฝึกหัดที่ 10: Exception Safety ด้วย @finally

```objc
// จำลอง transaction ที่ต้อง rollback หาก error
@interface MockTransaction : NSObject
@property (nonatomic, assign) BOOL isActive;
- (void)begin;
- (void)commit;
- (void)rollback;
@end

@implementation MockTransaction
- (void)begin    { _isActive = YES;  NSLog(@"Transaction begun"); }
- (void)commit   { _isActive = NO;   NSLog(@"Transaction committed"); }
- (void)rollback { _isActive = NO;   NSLog(@"Transaction rolled back"); }
@end

BOOL executeTransaction(MockTransaction *tx,
                         NSArray *operations,
                         NSError **error) {
    [tx begin];
    BOOL success = NO;
    
    @try {
        for (NSString *op in operations) {
            NSLog(@"Executing: %@", op);
            if ([op isEqualToString:@"FAIL"]) {
                @throw [NSException exceptionWithName:@"OperationFailed"
                                               reason:@"Operation failed"
                                             userInfo:nil];
            }
        }
        [tx commit];
        success = YES;
    }
    @catch (NSException *e) {
        if (error) {
            *error = [NSError errorWithDomain:@"com.db" code:1
                                     userInfo:@{NSLocalizedDescriptionKey: e.reason}];
        }
    }
    @finally {
        if (tx.isActive) {
            [tx rollback]; // rollback ถ้ายังไม่ committed
        }
    }
    
    return success;
}

// ทดสอบ
MockTransaction *tx1 = [[MockTransaction alloc] init];
NSError *err = nil;
if (executeTransaction(tx1, @[@"INSERT", @"UPDATE", @"DELETE"], &err)) {
    NSLog(@"Transaction 1: สำเร็จ");
} else {
    NSLog(@"Transaction 1: ล้มเหลว - %@", err.localizedDescription);
}

NSLog(@"---");

MockTransaction *tx2 = [[MockTransaction alloc] init];
err = nil;
if (executeTransaction(tx2, @[@"INSERT", @"FAIL", @"DELETE"], &err)) {
    NSLog(@"Transaction 2: สำเร็จ");
} else {
    NSLog(@"Transaction 2: ล้มเหลว - %@", err.localizedDescription);
}
```

---

## สรุป

ในบทนี้เราได้เรียนรู้:

1. **NSError vs Exceptions** - ความแตกต่างและเมื่อไรควรใช้แบบใด
2. **NSError** - การสร้าง, domain, code, userInfo
3. **NSLocalizedDescriptionKey** - keys สำคัญใน userInfo
4. **NSError** pattern** - การส่ง error ผ่าน `NSError **` parameter
5. **@try/@catch/@throw/@finally** - syntax และการใช้งาน
6. **Exception Types** - NSRangeException, NSInvalidArgumentException ฯลฯ
7. **Error Propagation** - การส่ง error ขึ้น call stack
8. **Custom Error Classes** - สร้าง error class เองที่ extends NSError
9. **Result Pattern** - Alternative สำหรับ return value + error
10. **Best Practices** - แนวทางที่ถูกต้องในการจัดการ error

ในบทถัดไปเราจะเรียนรู้เรื่อง **Blocks** ใน Objective-C ซึ่งเป็น feature ที่ทรงพลังมาก
