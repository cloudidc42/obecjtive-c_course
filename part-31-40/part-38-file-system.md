# Part 38: File System Operations ใน Objective-C

## บทนำ

การจัดการไฟล์และโฟลเดอร์เป็นทักษะพื้นฐานที่จำเป็นในการพัฒนา iOS/macOS application ทุกตัว ไม่ว่าจะเป็นการบันทึกข้อมูลผู้ใช้, การ cache ข้อมูลจากเครือข่าย, หรือการโหลดทรัพยากรต่างๆ

ใน Objective-C เราใช้ `NSFileManager` เป็นตัวกลางหลักในการจัดการ file system โดยในบทนี้จะครอบคลุม:
- NSFileManager และ API หลักๆ ทั้งหมด
- Directories สำคัญในระบบ iOS/macOS
- การอ่านและเขียนไฟล์หลายรูปแบบ
- การจัดการไฟล์และโฟลเดอร์
- File attributes และ metadata
- Sandboxing และข้อจำกัดใน iOS
- Plist files

---

## 38.1 NSFileManager พื้นฐาน

### การได้รับ Instance

```objc
// ใช้ defaultManager สำหรับ single-threaded operations
NSFileManager *fileManager = [NSFileManager defaultManager];

// สร้าง instance ใหม่สำหรับ background threads (thread-safe)
NSFileManager *bgFileManager = [[NSFileManager alloc] init];
```

**หมายเหตุ:** `[NSFileManager defaultManager]` ไม่ thread-safe สำหรับ delegate methods ควรสร้าง instance ใหม่เมื่อใช้บน background thread

### NSFileManager Delegate

```objc
@interface FileOperationManager : NSObject <NSFileManagerDelegate>

@property (nonatomic, strong) NSFileManager *fileManager;

@end

@implementation FileOperationManager

- (instancetype)init {
    self = [super init];
    if (self) {
        _fileManager = [[NSFileManager alloc] init];
        _fileManager.delegate = self;
    }
    return self;
}

// Delegate method - เรียกเมื่อจะ overwrite ไฟล์
- (BOOL)fileManager:(NSFileManager *)fileManager
   shouldCopyItemAtPath:(NSString *)srcPath
                 toPath:(NSString *)dstPath {
    NSLog(@"Will copy from %@ to %@", srcPath, dstPath);
    return YES;  // อนุญาตให้ copy
}

- (BOOL)fileManager:(NSFileManager *)fileManager
   shouldMoveItemAtPath:(NSString *)srcPath
                 toPath:(NSString *)dstPath {
    return YES;
}

- (BOOL)fileManager:(NSFileManager *)fileManager
  shouldRemoveItemAtPath:(NSString *)path {
    NSLog(@"Will delete: %@", path);
    return YES;
}

@end
```

---

## 38.2 Common Directories ใน iOS/macOS

### Directory Structure ใน iOS Sandbox

```
App Sandbox
├── AppName.app          (Bundle - read-only)
│   ├── Info.plist
│   ├── AppName          (executable)
│   ├── Assets.car
│   └── (resources)
│
├── Documents/           (User data - backed up by iCloud)
│   └── (user files)
│
├── Library/             (App support files)
│   ├── Application Support/  (app data - backed up)
│   ├── Caches/              (cache - NOT backed up)
│   └── Preferences/         (user preferences)
│
└── tmp/                 (temporary - NOT backed up, cleaned by OS)
```

### NSSearchPathForDirectoriesInDomains

```objc
// Documents Directory - สำหรับข้อมูลผู้ใช้ที่ต้อง backup
NSArray *paths = NSSearchPathForDirectoriesInDomains(NSDocumentDirectory,
                                                      NSUserDomainMask,
                                                      YES);
NSString *documentsPath = [paths firstObject];
NSLog(@"Documents: %@", documentsPath);
// Output: /var/mobile/Containers/Data/Application/<UUID>/Documents

// Library Directory
paths = NSSearchPathForDirectoriesInDomains(NSLibraryDirectory,
                                             NSUserDomainMask,
                                             YES);
NSString *libraryPath = [paths firstObject];

// Caches Directory - สำหรับ cache ที่ OS อาจลบได้
paths = NSSearchPathForDirectoriesInDomains(NSCachesDirectory,
                                             NSUserDomainMask,
                                             YES);
NSString *cachesPath = [paths firstObject];

// Application Support Directory
paths = NSSearchPathForDirectoriesInDomains(NSApplicationSupportDirectory,
                                             NSUserDomainMask,
                                             YES);
NSString *appSupportPath = [paths firstObject];

// Temporary Directory
NSString *tempPath = NSTemporaryDirectory();
NSLog(@"Temp: %@", tempPath);
```

### ใช้ NSURL API (แนะนำสำหรับ modern code)

```objc
NSFileManager *fm = [NSFileManager defaultManager];

// Documents
NSURL *documentsURL = [[fm URLsForDirectory:NSDocumentDirectory
                                  inDomains:NSUserDomainMask] firstObject];

// Library
NSURL *libraryURL = [[fm URLsForDirectory:NSLibraryDirectory
                                inDomains:NSUserDomainMask] firstObject];

// Caches
NSURL *cachesURL = [[fm URLsForDirectory:NSCachesDirectory
                               inDomains:NSUserDomainMask] firstObject];

// Application Support
NSURL *appSupportURL = [[fm URLsForDirectory:NSApplicationSupportDirectory
                                   inDomains:NSUserDomainMask] firstObject];

// Temp
NSURL *tempURL = [NSURL fileURLWithPath:NSTemporaryDirectory()];

// สร้าง path จาก URL
NSURL *fileURL = [documentsURL URLByAppendingPathComponent:@"data.json"];
NSString *filePath = fileURL.path;
```

### Helper Methods สำหรับ Paths

```objc
// Utility class สำหรับจัดการ paths
@interface PathHelper : NSObject

+ (NSString *)documentsDirectory;
+ (NSString *)libraryDirectory;
+ (NSString *)cachesDirectory;
+ (NSString *)tempDirectory;
+ (NSString *)pathInDocuments:(NSString *)filename;
+ (NSString *)pathInCaches:(NSString *)filename;
+ (NSString *)pathInTemp:(NSString *)filename;

@end

@implementation PathHelper

+ (NSString *)documentsDirectory {
    NSArray *paths = NSSearchPathForDirectoriesInDomains(NSDocumentDirectory, NSUserDomainMask, YES);
    return [paths firstObject];
}

+ (NSString *)libraryDirectory {
    NSArray *paths = NSSearchPathForDirectoriesInDomains(NSLibraryDirectory, NSUserDomainMask, YES);
    return [paths firstObject];
}

+ (NSString *)cachesDirectory {
    NSArray *paths = NSSearchPathForDirectoriesInDomains(NSCachesDirectory, NSUserDomainMask, YES);
    return [paths firstObject];
}

+ (NSString *)tempDirectory {
    return NSTemporaryDirectory();
}

+ (NSString *)pathInDocuments:(NSString *)filename {
    return [[self documentsDirectory] stringByAppendingPathComponent:filename];
}

+ (NSString *)pathInCaches:(NSString *)filename {
    return [[self cachesDirectory] stringByAppendingPathComponent:filename];
}

+ (NSString *)pathInTemp:(NSString *)filename {
    return [[self tempDirectory] stringByAppendingPathComponent:filename];
}

@end
```

---

## 38.3 การสร้างไฟล์และโฟลเดอร์

### สร้าง Directory

```objc
NSFileManager *fm = [NSFileManager defaultManager];
NSString *documentsPath = [PathHelper documentsDirectory];

// สร้าง directory เดียว
NSString *newDir = [documentsPath stringByAppendingPathComponent:@"MyFolder"];
NSError *error = nil;

BOOL success = [fm createDirectoryAtPath:newDir
             withIntermediateDirectories:YES  // สร้าง parent directories ด้วย
                              attributes:nil
                                   error:&error];
if (success) {
    NSLog(@"Directory created: %@", newDir);
} else {
    NSLog(@"Failed to create directory: %@", error.localizedDescription);
}

// ใช้ NSURL
NSURL *cachesURL = [[fm URLsForDirectory:NSCachesDirectory inDomains:NSUserDomainMask] firstObject];
NSURL *newDirURL = [cachesURL URLByAppendingPathComponent:@"ImageCache" isDirectory:YES];

success = [fm createDirectoryAtURL:newDirURL
       withIntermediateDirectories:YES
                        attributes:nil
                             error:&error];
```

### สร้างไฟล์

```objc
// สร้างไฟล์เปล่า
NSString *filePath = [PathHelper pathInDocuments:@"notes.txt"];

BOOL created = [fm createFileAtPath:filePath
                           contents:nil    // nil = ไฟล์เปล่า
                         attributes:nil];
NSLog(@"File created: %@", created ? @"YES" : @"NO");

// สร้างไฟล์พร้อม content
NSString *content = @"Hello, World!";
NSData *data = [content dataUsingEncoding:NSUTF8StringEncoding];

created = [fm createFileAtPath:filePath
                      contents:data
                    attributes:nil];
```

### ตรวจสอบการมีอยู่ของไฟล์/โฟลเดอร์

```objc
// ตรวจสอบว่ามีอยู่หรือไม่
BOOL exists = [fm fileExistsAtPath:filePath];

// ตรวจสอบและรู้ว่าเป็น directory หรือเปล่า
BOOL isDirectory = NO;
exists = [fm fileExistsAtPath:filePath isDirectory:&isDirectory];

if (exists) {
    if (isDirectory) {
        NSLog(@"%@ is a directory", filePath);
    } else {
        NSLog(@"%@ is a file", filePath);
    }
} else {
    NSLog(@"%@ does not exist", filePath);
}

// ตรวจสอบ permissions
BOOL readable = [fm isReadableFileAtPath:filePath];
BOOL writable = [fm isWritableFileAtPath:filePath];
BOOL executable = [fm isExecutableFileAtPath:filePath];
BOOL deletable = [fm isDeletableFileAtPath:filePath];
```

---

## 38.4 การอ่านไฟล์

### อ่านเป็น NSString

```objc
NSString *filePath = [PathHelper pathInDocuments:@"story.txt"];

// วิธีที่ 1: อ่านตรงๆ
NSError *error = nil;
NSString *content = [NSString stringWithContentsOfFile:filePath
                                              encoding:NSUTF8StringEncoding
                                                 error:&error];
if (error) {
    NSLog(@"Error reading file: %@", error.localizedDescription);
} else {
    NSLog(@"Content: %@", content);
}

// วิธีที่ 2: ใช้ NSURL
NSURL *fileURL = [NSURL fileURLWithPath:filePath];
NSString *urlContent = [NSString stringWithContentsOfURL:fileURL
                                                encoding:NSUTF8StringEncoding
                                                   error:&error];

// วิธีที่ 3: ผ่าน NSData ก่อน (useful เมื่อต้องการ NSData ด้วย)
NSData *data = [NSData dataWithContentsOfFile:filePath];
if (data) {
    NSString *stringFromData = [[NSString alloc] initWithData:data encoding:NSUTF8StringEncoding];
    NSLog(@"Content from data: %@", stringFromData);
}
```

### อ่านเป็น NSData

```objc
// อ่าน binary data
NSString *imagePath = [PathHelper pathInDocuments:@"image.png"];
NSData *imageData = [NSData dataWithContentsOfFile:imagePath];

if (imageData) {
    NSLog(@"Image size: %lu bytes", (unsigned long)imageData.length);
    UIImage *image = [UIImage imageWithData:imageData];
    // ใช้งาน image
}

// Options สำหรับ memory mapping
NSError *error = nil;
NSData *largeFileData = [NSData dataWithContentsOfFile:filePath
                                               options:NSDataReadingMappedIfSafe
                                                 error:&error];
// NSDataReadingMappedIfSafe: map ไฟล์ใหญ่แทนการโหลดทั้งหมดเข้า memory
```

### อ่านทีละบรรทัด (Large Files)

```objc
// สำหรับไฟล์ขนาดใหญ่ควรอ่านทีละ chunk
@interface FileLineReader : NSObject

- (instancetype)initWithPath:(NSString *)path;
- (NSString *)readLine;
- (BOOL)isAtEnd;

@end

@implementation FileLineReader {
    NSFileHandle *_fileHandle;
    NSMutableData *_buffer;
    BOOL _isAtEnd;
}

- (instancetype)initWithPath:(NSString *)path {
    self = [super init];
    if (self) {
        _fileHandle = [NSFileHandle fileHandleForReadingAtPath:path];
        _buffer = [NSMutableData data];
        _isAtEnd = (_fileHandle == nil);
    }
    return self;
}

- (NSString *)readLine {
    if (_isAtEnd) return nil;
    
    NSData *newlineData = [@"\n" dataUsingEncoding:NSUTF8StringEncoding];
    
    while (YES) {
        // ค้นหา newline ใน buffer
        NSRange searchRange = NSMakeRange(0, _buffer.length);
        NSRange newlineRange = [_buffer rangeOfData:newlineData options:0 range:searchRange];
        
        if (newlineRange.location != NSNotFound) {
            // พบ newline - ตัดบรรทัดออกมา
            NSData *lineData = [_buffer subdataWithRange:NSMakeRange(0, newlineRange.location)];
            NSString *line = [[NSString alloc] initWithData:lineData encoding:NSUTF8StringEncoding];
            
            // ลบ line + newline ออกจาก buffer
            [_buffer replaceBytesInRange:NSMakeRange(0, newlineRange.location + newlineRange.length) withBytes:NULL length:0];
            
            return line;
        }
        
        // อ่านข้อมูลเพิ่ม
        NSData *chunk = [_fileHandle readDataOfLength:4096];
        if (chunk.length == 0) {
            _isAtEnd = YES;
            // คืนค่าที่เหลือใน buffer
            if (_buffer.length > 0) {
                NSString *remaining = [[NSString alloc] initWithData:_buffer encoding:NSUTF8StringEncoding];
                [_buffer setLength:0];
                return remaining;
            }
            return nil;
        }
        
        [_buffer appendData:chunk];
    }
}

- (BOOL)isAtEnd {
    return _isAtEnd;
}

@end

// การใช้งาน
FileLineReader *reader = [[FileLineReader alloc] initWithPath:@"/path/to/large/file.txt"];
NSString *line;
NSInteger lineNumber = 0;

while ((line = [reader readLine]) != nil) {
    lineNumber++;
    NSLog(@"Line %ld: %@", (long)lineNumber, line);
}
```

---

## 38.5 การเขียนไฟล์

### เขียน NSString

```objc
NSString *content = @"สวัสดี Objective-C!\nนี่คือข้อความทดสอบ";
NSString *filePath = [PathHelper pathInDocuments:@"output.txt"];

// เขียนแบบง่ายๆ
NSError *error = nil;
BOOL success = [content writeToFile:filePath
                         atomically:YES  // เขียน temp ไฟล์ก่อนแล้ว rename - ปลอดภัยกว่า
                           encoding:NSUTF8StringEncoding
                              error:&error];

if (success) {
    NSLog(@"File written successfully");
} else {
    NSLog(@"Error: %@", error.localizedDescription);
}

// เขียนผ่าน NSURL
NSURL *fileURL = [NSURL fileURLWithPath:filePath];
success = [content writeToURL:fileURL
                   atomically:YES
                     encoding:NSUTF8StringEncoding
                        error:&error];
```

### เขียน NSData

```objc
// เขียน binary data
NSData *data = [content dataUsingEncoding:NSUTF8StringEncoding];

BOOL success = [data writeToFile:filePath atomically:YES];
// หรือ
success = [data writeToFile:filePath
                    options:NSDataWritingAtomic
                      error:&error];

// NSDataWritingAtomic: เขียนปลอดภัย
// NSDataWritingWithoutOverwriting: ไม่ overwrite ถ้ามีอยู่แล้ว
// NSDataWritingFileProtectionComplete: เข้ารหัสไฟล์ (iOS)
```

### เขียนแบบ Append (ต่อท้าย)

```objc
NSString *logPath = [PathHelper pathInDocuments:@"app.log"];

- (void)appendLogMessage:(NSString *)message {
    NSString *logEntry = [NSString stringWithFormat:@"[%@] %@\n",
                          [NSDate date],
                          message];
    NSData *logData = [logEntry dataUsingEncoding:NSUTF8StringEncoding];
    
    NSFileManager *fm = [NSFileManager defaultManager];
    
    if (![fm fileExistsAtPath:logPath]) {
        // สร้างไฟล์ใหม่ถ้ายังไม่มี
        [fm createFileAtPath:logPath contents:logData attributes:nil];
    } else {
        // ต่อท้ายไฟล์ที่มีอยู่
        NSFileHandle *fileHandle = [NSFileHandle fileHandleForWritingAtPath:logPath];
        [fileHandle seekToEndOfFile];
        [fileHandle writeData:logData];
        [fileHandle closeFile];
    }
}
```

### เขียนแบบ Buffered (สำหรับข้อมูลมาก)

```objc
@interface BufferedFileWriter : NSObject

- (instancetype)initWithPath:(NSString *)path;
- (void)writeLine:(NSString *)line;
- (void)flush;
- (void)close;

@end

@implementation BufferedFileWriter {
    NSFileHandle *_fileHandle;
    NSMutableData *_buffer;
    NSUInteger _bufferThreshold;
}

- (instancetype)initWithPath:(NSString *)path {
    self = [super init];
    if (self) {
        NSFileManager *fm = [NSFileManager defaultManager];
        if (![fm fileExistsAtPath:path]) {
            [fm createFileAtPath:path contents:nil attributes:nil];
        }
        _fileHandle = [NSFileHandle fileHandleForWritingAtPath:path];
        [_fileHandle truncateFileAtOffset:0];
        _buffer = [NSMutableData data];
        _bufferThreshold = 65536;  // 64KB buffer
    }
    return self;
}

- (void)writeLine:(NSString *)line {
    NSString *lineWithNewline = [line stringByAppendingString:@"\n"];
    NSData *data = [lineWithNewline dataUsingEncoding:NSUTF8StringEncoding];
    [_buffer appendData:data];
    
    if (_buffer.length >= _bufferThreshold) {
        [self flush];
    }
}

- (void)flush {
    if (_buffer.length > 0) {
        [_fileHandle writeData:_buffer];
        [_buffer setLength:0];
    }
}

- (void)close {
    [self flush];
    [_fileHandle closeFile];
}

@end
```

---

## 38.6 การย้าย, คัดลอก, และลบไฟล์

### Copy Files

```objc
NSFileManager *fm = [NSFileManager defaultManager];
NSError *error = nil;

NSString *sourcePath = [PathHelper pathInDocuments:@"original.txt"];
NSString *destPath = [PathHelper pathInCaches:@"copy.txt"];

// คัดลอกไฟล์
BOOL success = [fm copyItemAtPath:sourcePath
                           toPath:destPath
                            error:&error];

if (!success) {
    NSLog(@"Copy failed: %@", error.localizedDescription);
    
    // error codes
    // NSFileNoSuchFileError: ไม่พบไฟล์ต้นทาง
    // NSFileWriteFileExistsError: ปลายทางมีอยู่แล้ว
}

// Copy ด้วย URL
NSURL *sourceURL = [NSURL fileURLWithPath:sourcePath];
NSURL *destURL = [NSURL fileURLWithPath:destPath];

success = [fm copyItemAtURL:sourceURL toURL:destURL error:&error];
```

### Move Files

```objc
// ย้ายไฟล์ (rename ก็ใช้ moveItem ด้วย)
NSString *oldPath = [PathHelper pathInDocuments:@"old_name.txt"];
NSString *newPath = [PathHelper pathInDocuments:@"new_name.txt"];

BOOL success = [fm moveItemAtPath:oldPath
                           toPath:newPath
                            error:&error];

// ถ้า destination มีอยู่แล้ว ต้องลบก่อน
if ([fm fileExistsAtPath:newPath]) {
    [fm removeItemAtPath:newPath error:nil];
}
success = [fm moveItemAtPath:oldPath toPath:newPath error:&error];
```

### Delete Files

```objc
// ลบไฟล์
NSString *filePath = [PathHelper pathInDocuments:@"temp.txt"];

if ([fm fileExistsAtPath:filePath]) {
    BOOL success = [fm removeItemAtPath:filePath error:&error];
    
    if (!success) {
        NSLog(@"Delete failed: %@", error.localizedDescription);
    }
}

// ลบ directory และเนื้อหาทั้งหมด
NSString *dirPath = [PathHelper pathInDocuments:@"OldData"];

BOOL success = [fm removeItemAtPath:dirPath error:&error];
// จะลบ directory พร้อมทุกไฟล์ข้างใน

// Safe delete function
- (BOOL)safeDeleteItemAtPath:(NSString *)path error:(NSError **)error {
    NSFileManager *fm = [NSFileManager defaultManager];
    
    if (![fm fileExistsAtPath:path]) {
        return YES;  // ไม่มีอยู่แล้ว ถือว่า success
    }
    
    return [fm removeItemAtPath:path error:error];
}
```

---

## 38.7 Directory Listing

### รายการไฟล์ใน Directory

```objc
NSFileManager *fm = [NSFileManager defaultManager];
NSString *documentsPath = [PathHelper documentsDirectory];

// วิธีที่ 1: รายการตรงๆ (ไม่รวม subdirectory)
NSError *error = nil;
NSArray *contents = [fm contentsOfDirectoryAtPath:documentsPath error:&error];

if (error) {
    NSLog(@"Error listing directory: %@", error.localizedDescription);
} else {
    for (NSString *item in contents) {
        NSLog(@"- %@", item);
    }
}

// วิธีที่ 2: Subpaths (รายการทั้งหมดรวม subdirectory)
NSArray *allPaths = [fm subpathsOfDirectoryAtPath:documentsPath error:&error];
for (NSString *path in allPaths) {
    NSLog(@"  %@", path);
}

// วิธีที่ 3: ใช้ Enumerator (สำหรับ custom traversal)
NSDirectoryEnumerator *enumerator = [fm enumeratorAtPath:documentsPath];
NSString *file;

while ((file = [enumerator nextObject]) != nil) {
    NSString *fullPath = [documentsPath stringByAppendingPathComponent:file];
    
    // ข้ามถ้าเป็น hidden file
    if ([file hasPrefix:@"."]) {
        [enumerator skipDescendants];
        continue;
    }
    
    BOOL isDir = NO;
    [fm fileExistsAtPath:fullPath isDirectory:&isDir];
    
    if (isDir) {
        NSLog(@"DIR: %@", file);
    } else {
        NSLog(@"FILE: %@", file);
    }
}
```

### Directory Enumerator พร้อม Options

```objc
// Enumerate ด้วย URL (modern API)
NSURL *documentsURL = [[fm URLsForDirectory:NSDocumentDirectory
                                  inDomains:NSUserDomainMask] firstObject];

NSArray *keys = @[NSURLNameKey,
                  NSURLIsDirectoryKey,
                  NSURLFileSizeKey,
                  NSURLCreationDateKey];

NSDirectoryEnumerator *urlEnumerator = [fm enumeratorAtURL:documentsURL
                                includingPropertiesForKeys:keys
                                                   options:NSDirectoryEnumerationSkipsHiddenFiles |
                                                           NSDirectoryEnumerationSkipsPackageDescendants
                                              errorHandler:^BOOL(NSURL *url, NSError *error) {
    NSLog(@"Error at %@: %@", url, error);
    return YES;  // ทำต่อ
}];

for (NSURL *fileURL in urlEnumerator) {
    NSString *name;
    NSNumber *isDirectory;
    NSNumber *fileSize;
    NSDate *creationDate;
    
    [fileURL getResourceValue:&name forKey:NSURLNameKey error:nil];
    [fileURL getResourceValue:&isDirectory forKey:NSURLIsDirectoryKey error:nil];
    [fileURL getResourceValue:&fileSize forKey:NSURLFileSizeKey error:nil];
    [fileURL getResourceValue:&creationDate forKey:NSURLCreationDateKey error:nil];
    
    if (![isDirectory boolValue]) {
        NSLog(@"%@ - %.2f KB - Created: %@",
              name,
              fileSize.floatValue / 1024.0,
              creationDate);
    }
}
```

### กรอง Files ตาม Extension

```objc
- (NSArray<NSString *> *)filesWithExtension:(NSString *)extension
                                inDirectory:(NSString *)dirPath {
    NSFileManager *fm = [NSFileManager defaultManager];
    NSError *error = nil;
    NSArray *allFiles = [fm contentsOfDirectoryAtPath:dirPath error:&error];
    
    if (error) return @[];
    
    NSPredicate *predicate = [NSPredicate predicateWithFormat:@"pathExtension == %@", extension];
    NSArray *filtered = [allFiles filteredArrayUsingPredicate:predicate];
    
    return [filtered valueForKeyPath:@"self.stringByDeletingPathExtension"];
}

// ตัวอย่างการใช้
NSArray *plistFiles = [self filesWithExtension:@"plist" inDirectory:documentsPath];
NSArray *imageFiles = [self filesWithExtension:@"png" inDirectory:documentsPath];
```

---

## 38.8 File Attributes

### อ่าน Attributes

```objc
NSFileManager *fm = [NSFileManager defaultManager];
NSString *filePath = [PathHelper pathInDocuments:@"data.txt"];

NSError *error = nil;
NSDictionary *attributes = [fm attributesOfItemAtPath:filePath error:&error];

if (attributes) {
    // ขนาดไฟล์
    NSNumber *fileSize = attributes[NSFileSize];
    NSLog(@"File size: %@ bytes", fileSize);
    
    // วันที่สร้าง
    NSDate *creationDate = attributes[NSFileCreationDate];
    NSLog(@"Created: %@", creationDate);
    
    // วันที่แก้ไขล่าสุด
    NSDate *modDate = attributes[NSFileModificationDate];
    NSLog(@"Modified: %@", modDate);
    
    // ประเภทไฟล์
    NSString *fileType = attributes[NSFileType];
    if ([fileType isEqualToString:NSFileTypeRegular]) {
        NSLog(@"Regular file");
    } else if ([fileType isEqualToString:NSFileTypeDirectory]) {
        NSLog(@"Directory");
    } else if ([fileType isEqualToString:NSFileTypeSymbolicLink]) {
        NSLog(@"Symbolic link");
    }
    
    // Permissions
    NSNumber *permissions = attributes[NSFilePosixPermissions];
    NSLog(@"Permissions: %o", [permissions shortValue]);
    
    // Owner
    NSString *owner = attributes[NSFileOwnerAccountName];
    NSLog(@"Owner: %@", owner);
}
```

### แก้ไข Attributes

```objc
// เปลี่ยนวันที่แก้ไข
NSDate *newDate = [NSDate dateWithTimeIntervalSinceNow:-3600];  // 1 ชั่วโมงที่แล้ว
NSDictionary *newAttribs = @{NSFileModificationDate: newDate};

BOOL success = [fm setAttributes:newAttribs ofItemAtPath:filePath error:&error];

// เปลี่ยน permissions (macOS/Unix)
// 0644 = owner read+write, group read, others read
NSDictionary *permAttribs = @{NSFilePosixPermissions: @(0644)};
[fm setAttributes:permAttribs ofItemAtPath:filePath error:nil];
```

### File Size Calculation

```objc
// คำนวณขนาด directory ทั้งหมด
- (unsigned long long)sizeOfDirectory:(NSString *)dirPath {
    NSFileManager *fm = [NSFileManager defaultManager];
    NSDirectoryEnumerator *enumerator = [fm enumeratorAtPath:dirPath];
    
    unsigned long long totalSize = 0;
    NSString *file;
    
    while ((file = [enumerator nextObject]) != nil) {
        NSString *fullPath = [dirPath stringByAppendingPathComponent:file];
        NSDictionary *attributes = [fm attributesOfItemAtPath:fullPath error:nil];
        
        if ([attributes[NSFileType] isEqualToString:NSFileTypeRegular]) {
            totalSize += [attributes[NSFileSize] unsignedLongLongValue];
        }
    }
    
    return totalSize;
}

// แสดงขนาดแบบ human-readable
- (NSString *)humanReadableSize:(unsigned long long)bytes {
    double value = bytes;
    NSArray *units = @[@"B", @"KB", @"MB", @"GB", @"TB"];
    NSInteger unitIndex = 0;
    
    while (value > 1024 && unitIndex < units.count - 1) {
        value /= 1024;
        unitIndex++;
    }
    
    return [NSString stringWithFormat:@"%.2f %@", value, units[unitIndex]];
}
```

---

## 38.9 File Monitoring

### ใช้ kqueue หรือ DispatchSource

```objc
@interface FileMonitor : NSObject

@property (nonatomic, copy) void(^fileChangedBlock)(NSString *path);

- (void)startMonitoringPath:(NSString *)path;
- (void)stopMonitoring;

@end

@implementation FileMonitor {
    dispatch_source_t _source;
    int _fileDescriptor;
    NSString *_monitoredPath;
}

- (void)startMonitoringPath:(NSString *)path {
    _monitoredPath = [path copy];
    
    // เปิดไฟล์
    _fileDescriptor = open([path fileSystemRepresentation], O_EVTONLY);
    
    if (_fileDescriptor < 0) {
        NSLog(@"Cannot open file for monitoring: %@", path);
        return;
    }
    
    // สร้าง dispatch source
    _source = dispatch_source_create(DISPATCH_SOURCE_TYPE_VNODE,
                                      _fileDescriptor,
                                      DISPATCH_VNODE_DELETE |
                                      DISPATCH_VNODE_WRITE |
                                      DISPATCH_VNODE_EXTEND |
                                      DISPATCH_VNODE_RENAME,
                                      dispatch_get_main_queue());
    
    dispatch_source_set_event_handler(_source, ^{
        unsigned long events = dispatch_source_get_data(self->_source);
        
        if (events & DISPATCH_VNODE_DELETE) {
            NSLog(@"File deleted: %@", self->_monitoredPath);
        }
        if (events & DISPATCH_VNODE_WRITE) {
            NSLog(@"File written: %@", self->_monitoredPath);
            if (self.fileChangedBlock) {
                self.fileChangedBlock(self->_monitoredPath);
            }
        }
        if (events & DISPATCH_VNODE_EXTEND) {
            NSLog(@"File extended: %@", self->_monitoredPath);
        }
        if (events & DISPATCH_VNODE_RENAME) {
            NSLog(@"File renamed: %@", self->_monitoredPath);
        }
    });
    
    dispatch_source_set_cancel_handler(_source, ^{
        close(self->_fileDescriptor);
    });
    
    dispatch_resume(_source);
    NSLog(@"Monitoring started for: %@", path);
}

- (void)stopMonitoring {
    if (_source) {
        dispatch_source_cancel(_source);
        _source = nil;
    }
}

- (void)dealloc {
    [self stopMonitoring];
}

@end

// การใช้งาน
FileMonitor *monitor = [[FileMonitor alloc] init];
monitor.fileChangedBlock = ^(NSString *path) {
    NSLog(@"File changed! Re-loading data from: %@", path);
    // โหลดข้อมูลใหม่
};

NSString *configPath = [PathHelper pathInDocuments:@"config.json"];
[monitor startMonitoringPath:configPath];
```

---

## 38.10 Sandboxing ใน iOS

### Sandbox Boundaries

```objc
// iOS Sandbox อนุญาตให้เข้าถึงเฉพาะ:
// 1. App bundle (read-only)
// 2. App's Documents, Library, tmp directories
// ไม่สามารถเข้าถึง:
// - Documents ของ app อื่น
// - System directories ส่วนใหญ่
// - Storage ภายนอก (ยกเว้นผ่าน UIDocumentPickerViewController)

// ตรวจสอบว่าเราอยู่ใน app sandbox
NSString *homePath = NSHomeDirectory();
NSLog(@"App Home: %@", homePath);
// Output: /var/mobile/Containers/Data/Application/<UUID>

// Bundle path (app resources - read only)
NSString *bundlePath = [[NSBundle mainBundle] bundlePath];
NSLog(@"Bundle: %@", bundlePath);

// ไม่สามารถเขียนใน bundle
NSString *bundleFile = [bundlePath stringByAppendingPathComponent:@"test.txt"];
NSError *error = nil;
[@"test" writeToFile:bundleFile atomically:YES encoding:NSUTF8StringEncoding error:&error];
NSLog(@"Write to bundle error: %@", error);  // จะ error!
```

### File Protection ใน iOS

```objc
// ตั้งค่า File Protection
NSString *sensitivePath = [PathHelper pathInDocuments:@"sensitive_data.dat"];

// สร้างไฟล์พร้อม file protection
NSDictionary *attributes = @{
    NSFileProtectionKey: NSFileProtectionComplete
    // NSFileProtectionComplete: encrypt เต็มที่ (ไม่สามารถเข้าถึงเมื่อล็อค)
    // NSFileProtectionCompleteUnlessOpen: ยอมให้ open ก่อนล็อค แล้วยังเข้าถึงได้
    // NSFileProtectionCompleteUntilFirstUserAuthentication: เข้าถึงได้หลัง boot จนกว่าจะล็อค
    // NSFileProtectionNone: ไม่มี protection
};

NSFileManager *fm = [NSFileManager defaultManager];
[fm createFileAtPath:sensitivePath contents:nil attributes:attributes];

// เปลี่ยน protection ของไฟล์ที่มีอยู่
[fm setAttributes:@{NSFileProtectionKey: NSFileProtectionComplete}
     ofItemAtPath:sensitivePath
            error:nil];
```

---

## 38.11 Plist Files

### อ่าน Plist จาก Bundle

```objc
// อ่าน plist จาก app bundle
NSString *plistPath = [[NSBundle mainBundle] pathForResource:@"Config" ofType:@"plist"];
NSDictionary *config = [NSDictionary dictionaryWithContentsOfFile:plistPath];

if (config) {
    NSString *apiURL = config[@"APIBaseURL"];
    NSInteger timeout = [config[@"TimeoutInterval"] integerValue];
    NSArray *features = config[@"EnabledFeatures"];
    
    NSLog(@"API URL: %@", apiURL);
    NSLog(@"Timeout: %ld", (long)timeout);
    NSLog(@"Features: %@", features);
}

// อ่าน Array plist
NSString *arrayPlistPath = [[NSBundle mainBundle] pathForResource:@"Countries" ofType:@"plist"];
NSArray *countries = [NSArray arrayWithContentsOfFile:arrayPlistPath];
```

### เขียน Plist ลง Documents

```objc
NSDictionary *userData = @{
    @"username": @"john_doe",
    @"score": @(1500),
    @"level": @(10),
    @"achievements": @[@"first_win", @"speed_demon", @"100_games"],
    @"lastPlayed": [NSDate date],
    @"settings": @{
        @"soundEnabled": @YES,
        @"musicVolume": @(0.7f),
        @"difficulty": @"medium"
    }
};

NSString *savePath = [PathHelper pathInDocuments:@"user_data.plist"];

// เขียน plist
BOOL success = [userData writeToFile:savePath atomically:YES];
NSLog(@"Save success: %@", success ? @"YES" : @"NO");

// อ่าน plist กลับมา
NSDictionary *loadedData = [NSDictionary dictionaryWithContentsOfFile:savePath];
NSLog(@"Username: %@", loadedData[@"username"]);
NSLog(@"Score: %@", loadedData[@"score"]);
```

### Plist ที่ซับซ้อนขึ้น

```objc
// UserPreferences Manager
@interface UserPreferences : NSObject

+ (instancetype)sharedPreferences;

- (void)setObject:(id)object forKey:(NSString *)key;
- (id)objectForKey:(NSString *)key;
- (void)setBool:(BOOL)value forKey:(NSString *)key;
- (BOOL)boolForKey:(NSString *)key;
- (void)setInteger:(NSInteger)value forKey:(NSString *)key;
- (NSInteger)integerForKey:(NSString *)key;
- (void)synchronize;

@end

@implementation UserPreferences {
    NSMutableDictionary *_prefs;
    NSString *_filePath;
    BOOL _isDirty;
}

+ (instancetype)sharedPreferences {
    static UserPreferences *instance = nil;
    static dispatch_once_t onceToken;
    dispatch_once(&onceToken, ^{
        instance = [[UserPreferences alloc] init];
    });
    return instance;
}

- (instancetype)init {
    self = [super init];
    if (self) {
        _filePath = [PathHelper pathInDocuments:@"user_preferences.plist"];
        [self load];
    }
    return self;
}

- (void)load {
    if ([[NSFileManager defaultManager] fileExistsAtPath:_filePath]) {
        _prefs = [NSMutableDictionary dictionaryWithContentsOfFile:_filePath];
    }
    if (!_prefs) {
        _prefs = [NSMutableDictionary dictionary];
    }
}

- (void)setObject:(id)object forKey:(NSString *)key {
    if (object) {
        _prefs[key] = object;
    } else {
        [_prefs removeObjectForKey:key];
    }
    _isDirty = YES;
}

- (id)objectForKey:(NSString *)key {
    return _prefs[key];
}

- (void)setBool:(BOOL)value forKey:(NSString *)key {
    [self setObject:@(value) forKey:key];
}

- (BOOL)boolForKey:(NSString *)key {
    return [[self objectForKey:key] boolValue];
}

- (void)setInteger:(NSInteger)value forKey:(NSString *)key {
    [self setObject:@(value) forKey:key];
}

- (NSInteger)integerForKey:(NSString *)key {
    return [[self objectForKey:key] integerValue];
}

- (void)synchronize {
    if (_isDirty) {
        [_prefs writeToFile:_filePath atomically:YES];
        _isDirty = NO;
    }
}

@end

// การใช้งาน
UserPreferences *prefs = [UserPreferences sharedPreferences];
[prefs setBool:YES forKey:@"notifications_enabled"];
[prefs setInteger:42 forKey:@"high_score"];
[prefs setObject:@"dark" forKey:@"theme"];
[prefs synchronize];

// อ่านค่า
BOOL notifs = [prefs boolForKey:@"notifications_enabled"];
NSInteger score = [prefs integerForKey:@"high_score"];
NSString *theme = [prefs objectForKey:@"theme"];
```

---

## 38.12 ตัวอย่าง Complete: FileStorageManager

```objc
// FileStorageManager.h
@interface FileStorageManager : NSObject

+ (instancetype)sharedManager;

// Text files
- (BOOL)saveText:(NSString *)text toFile:(NSString *)filename;
- (NSString *)loadTextFromFile:(NSString *)filename;

// Data files
- (BOOL)saveData:(NSData *)data toFile:(NSString *)filename;
- (NSData *)loadDataFromFile:(NSString *)filename;

// JSON files
- (BOOL)saveJSON:(id)object toFile:(NSString *)filename;
- (id)loadJSONFromFile:(NSString *)filename;

// File management
- (BOOL)deleteFile:(NSString *)filename;
- (BOOL)fileExists:(NSString *)filename;
- (NSArray *)allFiles;
- (unsigned long long)fileSizeForFile:(NSString *)filename;
- (unsigned long long)totalStorageUsed;

@end

@implementation FileStorageManager

+ (instancetype)sharedManager {
    static FileStorageManager *instance = nil;
    static dispatch_once_t onceToken;
    dispatch_once(&onceToken, ^{
        instance = [[FileStorageManager alloc] init];
    });
    return instance;
}

- (NSString *)pathForFile:(NSString *)filename {
    return [PathHelper pathInDocuments:filename];
}

- (BOOL)saveText:(NSString *)text toFile:(NSString *)filename {
    NSError *error = nil;
    BOOL success = [text writeToFile:[self pathForFile:filename]
                          atomically:YES
                            encoding:NSUTF8StringEncoding
                               error:&error];
    if (!success) {
        NSLog(@"[FileStorage] Error saving text: %@", error);
    }
    return success;
}

- (NSString *)loadTextFromFile:(NSString *)filename {
    NSString *path = [self pathForFile:filename];
    if (![[NSFileManager defaultManager] fileExistsAtPath:path]) {
        return nil;
    }
    
    NSError *error = nil;
    NSString *content = [NSString stringWithContentsOfFile:path
                                                  encoding:NSUTF8StringEncoding
                                                     error:&error];
    if (error) {
        NSLog(@"[FileStorage] Error loading text: %@", error);
    }
    return content;
}

- (BOOL)saveData:(NSData *)data toFile:(NSString *)filename {
    NSError *error = nil;
    BOOL success = [data writeToFile:[self pathForFile:filename]
                             options:NSDataWritingAtomic
                               error:&error];
    if (!success) {
        NSLog(@"[FileStorage] Error saving data: %@", error);
    }
    return success;
}

- (NSData *)loadDataFromFile:(NSString *)filename {
    return [NSData dataWithContentsOfFile:[self pathForFile:filename]];
}

- (BOOL)saveJSON:(id)object toFile:(NSString *)filename {
    NSError *error = nil;
    NSData *jsonData = [NSJSONSerialization dataWithJSONObject:object
                                                       options:NSJSONWritingPrettyPrinted
                                                         error:&error];
    if (error) {
        NSLog(@"[FileStorage] JSON serialization error: %@", error);
        return NO;
    }
    return [self saveData:jsonData toFile:filename];
}

- (id)loadJSONFromFile:(NSString *)filename {
    NSData *data = [self loadDataFromFile:filename];
    if (!data) return nil;
    
    NSError *error = nil;
    id object = [NSJSONSerialization JSONObjectWithData:data options:0 error:&error];
    if (error) {
        NSLog(@"[FileStorage] JSON parse error: %@", error);
    }
    return object;
}

- (BOOL)deleteFile:(NSString *)filename {
    NSError *error = nil;
    BOOL success = [[NSFileManager defaultManager] removeItemAtPath:[self pathForFile:filename]
                                                              error:&error];
    if (!success) {
        NSLog(@"[FileStorage] Error deleting file: %@", error);
    }
    return success;
}

- (BOOL)fileExists:(NSString *)filename {
    return [[NSFileManager defaultManager] fileExistsAtPath:[self pathForFile:filename]];
}

- (NSArray *)allFiles {
    NSError *error = nil;
    return [[NSFileManager defaultManager] contentsOfDirectoryAtPath:[PathHelper documentsDirectory]
                                                               error:&error];
}

- (unsigned long long)fileSizeForFile:(NSString *)filename {
    NSString *path = [self pathForFile:filename];
    NSDictionary *attrs = [[NSFileManager defaultManager] attributesOfItemAtPath:path error:nil];
    return [attrs[NSFileSize] unsignedLongLongValue];
}

- (unsigned long long)totalStorageUsed {
    unsigned long long total = 0;
    for (NSString *file in [self allFiles]) {
        total += [self fileSizeForFile:file];
    }
    return total;
}

@end
```

---

## แบบฝึกหัด (Practice Exercises)

### Exercise 1: Document Browser
สร้าง `DocumentBrowser` class ที่:
- List ไฟล์ทั้งหมดใน Documents directory
- แสดง metadata (ชื่อ, ขนาด, วันที่แก้ไข)
- กรองตาม file type
- เรียงลำดับตามขนาดหรือวันที่

### Exercise 2: Log File Manager
สร้าง `LogManager` ที่:
- เขียน log entries ต่อท้ายไฟล์
- Rotate ไฟล์เมื่อใหญ่เกิน limit
- Archive ไฟล์เก่า (rename เป็น `log_YYYYMMDD.txt`)
- อ่าน log ย้อนหลัง N entries

### Exercise 3: Image Cache
สร้าง `ImageCache` ที่:
- บันทึก UIImage ลง Caches directory
- อ่าน cached image กลับมา
- ลบ cache เก่ากว่า 7 วัน
- จำกัดขนาด cache ไม่เกิน 50MB

### Exercise 4: Config File Manager
สร้าง app ที่:
- อ่าน default config จาก bundle
- Copy ไปยัง Documents เมื่อ launch ครั้งแรก
- อนุญาตให้ user แก้ไข config
- Reset กลับเป็น default ได้

### Exercise 5: File Search
สร้าง function ที่:
- ค้นหาไฟล์ตาม filename pattern
- ค้นหา text ภายในไฟล์
- Return results พร้อม path

### Exercise 6: Secure File Storage
สร้าง `SecureStorage` ที่:
- บันทึกข้อมูลใน Documents พร้อม NSFileProtectionComplete
- Verify ว่าไฟล์มี protection ถูกต้อง
- Handle error เมื่อพยายามเข้าถึงขณะล็อค device

### Exercise 7: Directory Watcher
สร้าง class ที่ monitor directory และ:
- แจ้งเมื่อมีไฟล์ใหม่
- แจ้งเมื่อไฟล์ถูกลบ
- แจ้งเมื่อไฟล์เปลี่ยนแปลง

### Exercise 8: File Import/Export
สร้างระบบที่:
- Export ข้อมูลเป็น CSV
- Import CSV กลับมา
- Handle error cases (format ผิด, ไฟล์เสีย)

### Exercise 9: Temp File Manager
สร้าง `TempFileManager` ที่:
- สร้าง unique temp file names
- ติดตามไฟล์ temp ที่สร้าง
- ลบไฟล์ temp อัตโนมัติเมื่อ app terminate

### Exercise 10: Storage Usage Report
สร้าง function ที่รายงาน:
- ขนาดของแต่ละ directory (Documents, Library, Caches, tmp)
- ไฟล์ 10 อันที่ใหญ่ที่สุด
- จำนวนไฟล์ตาม extension
- พื้นที่ว่างที่เหลือบน device

---

## สรุป

NSFileManager เป็น API หลักในการจัดการ file system:

1. **ใช้ NSURL แทน NSString** สำหรับ paths ใน modern code
2. **atomically:YES** เสมอเมื่อเขียนไฟล์ เพื่อความปลอดภัย
3. **ตรวจสอบ error** ทุกครั้งหลัง file operations
4. **เลือก directory ให้ถูก**: Documents สำหรับข้อมูลผู้ใช้, Caches สำหรับ cache
5. **File Protection** สำหรับข้อมูลสำคัญ
6. **Plist** เหมาะสำหรับ configuration และ simple data

ในบทถัดไปเราจะเรียนรู้ Serialization ที่จะช่วยให้เราแปลง Objective-C objects เป็นรูปแบบที่บันทึกลงไฟล์ได้
