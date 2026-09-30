# Part 39: Serialization ใน Objective-C

## บทนำ

Serialization คือกระบวนการแปลง object ให้อยู่ในรูปแบบที่สามารถบันทึก, ส่งผ่านเครือข่าย, หรือสร้างขึ้นมาใหม่ได้ในภายหลัง ใน Objective-C มีหลายวิธีในการทำ serialization:

1. **NSCoding / NSArchiver** - สำหรับ Objective-C objects
2. **NSKeyedArchiver / NSKeyedUnarchiver** - มาตรฐานสมัยใหม่
3. **NSJSONSerialization** - สำหรับ JSON format
4. **NSPropertyListSerialization** - สำหรับ Plist format

ในบทนี้เราจะเรียนรู้ทุก approach พร้อมตัวอย่างที่ใช้งานได้จริง

---

## 39.1 NSCoding Protocol

### พื้นฐาน NSCoding

```objc
// NSCoding protocol มี 2 methods:
@protocol NSCoding
- (void)encodeWithCoder:(NSCoder *)coder;      // บันทึก (encode)
- (nullable instancetype)initWithCoder:(NSCoder *)coder;  // โหลด (decode)
@end
```

### การ implement NSCoding

```objc
// Person.h
#import <Foundation/Foundation.h>

@interface Person : NSObject <NSCoding>

@property (nonatomic, copy) NSString *name;
@property (nonatomic, assign) NSInteger age;
@property (nonatomic, copy) NSString *email;
@property (nonatomic, strong) NSDate *birthDate;

- (instancetype)initWithName:(NSString *)name
                         age:(NSInteger)age
                       email:(NSString *)email;

@end

// Person.m
#import "Person.h"

// Keys สำหรับ encoding/decoding
static NSString * const kPersonName = @"name";
static NSString * const kPersonAge = @"age";
static NSString * const kPersonEmail = @"email";
static NSString * const kPersonBirthDate = @"birthDate";

@implementation Person

- (instancetype)initWithName:(NSString *)name
                         age:(NSInteger)age
                       email:(NSString *)email {
    self = [super init];
    if (self) {
        _name = [name copy];
        _age = age;
        _email = [email copy];
    }
    return self;
}

#pragma mark - NSCoding

// เข้ารหัส (บันทึก) object
- (void)encodeWithCoder:(NSCoder *)coder {
    [coder encodeObject:self.name forKey:kPersonName];
    [coder encodeInteger:self.age forKey:kPersonAge];
    [coder encodeObject:self.email forKey:kPersonEmail];
    [coder encodeObject:self.birthDate forKey:kPersonBirthDate];
}

// ถอดรหัส (โหลด) object
- (instancetype)initWithCoder:(NSCoder *)coder {
    self = [super init];
    if (self) {
        _name = [coder decodeObjectForKey:kPersonName];
        _age = [coder decodeIntegerForKey:kPersonAge];
        _email = [coder decodeObjectForKey:kPersonEmail];
        _birthDate = [coder decodeObjectForKey:kPersonBirthDate];
    }
    return self;
}

- (NSString *)description {
    return [NSString stringWithFormat:@"Person(name: %@, age: %ld, email: %@)",
            self.name, (long)self.age, self.email];
}

@end
```

### NSCoder Methods สำหรับ Types ต่างๆ

```objc
- (void)encodeWithCoder:(NSCoder *)coder {
    // Objects
    [coder encodeObject:self.name forKey:@"name"];                   // NSString
    [coder encodeObject:self.address forKey:@"address"];             // NSObject
    
    // Primitives
    [coder encodeBool:self.isActive forKey:@"isActive"];             // BOOL
    [coder encodeInt:self.shortValue forKey:@"shortValue"];          // int
    [coder encodeInt32:self.int32Value forKey:@"int32Value"];        // int32_t
    [coder encodeInt64:self.int64Value forKey:@"int64Value"];        // int64_t
    [coder encodeInteger:self.count forKey:@"count"];                 // NSInteger
    [coder encodeFloat:self.floatValue forKey:@"floatValue"];        // float
    [coder encodeDouble:self.doubleValue forKey:@"doubleValue"];     // double
    
    // CGTypes (ต้องใช้ helper)
    [coder encodeCGPoint:self.position forKey:@"position"];
    [coder encodeCGSize:self.size forKey:@"size"];
    [coder encodeCGRect:self.frame forKey:@"frame"];
    
    // Collections (ทุก element ต้อง NSCoding compliant)
    [coder encodeObject:self.items forKey:@"items"];   // NSArray
    [coder encodeObject:self.info forKey:@"info"];     // NSDictionary
}

- (instancetype)initWithCoder:(NSCoder *)coder {
    self = [super init];
    if (self) {
        // Objects
        _name = [coder decodeObjectOfClass:[NSString class] forKey:@"name"];  // Secure
        _address = [coder decodeObjectForKey:@"address"];                       // Older API
        
        // Primitives
        _isActive = [coder decodeBoolForKey:@"isActive"];
        _shortValue = [coder decodeIntForKey:@"shortValue"];
        _int32Value = [coder decodeInt32ForKey:@"int32Value"];
        _int64Value = [coder decodeInt64ForKey:@"int64Value"];
        _count = [coder decodeIntegerForKey:@"count"];
        _floatValue = [coder decodeFloatForKey:@"floatValue"];
        _doubleValue = [coder decodeDoubleForKey:@"doubleValue"];
        
        // CGTypes
        _position = [coder decodeCGPointForKey:@"position"];
        _size = [coder decodeCGSizeForKey:@"size"];
        _frame = [coder decodeCGRectForKey:@"frame"];
        
        // Collections
        _items = [coder decodeObjectForKey:@"items"];
        _info = [coder decodeObjectForKey:@"info"];
    }
    return self;
}
```

---

## 39.2 NSKeyedArchiver / NSKeyedUnarchiver

NSKeyedArchiver เป็น implementation ของ NSCoder ที่ใช้ key-based encoding ซึ่งดีกว่า NSArchiver แบบเก่าเพราะ:
- ทนต่อการเปลี่ยน property order
- สามารถ ignore ค่าที่ไม่รู้จักได้
- Forward compatible ดีกว่า

### Archive ไปยัง File

```objc
// Archive (บันทึก)
Person *person = [[Person alloc] initWithName:@"สมชาย"
                                           age:30
                                         email:@"somchai@example.com"];

NSString *filePath = [NSTemporaryDirectory() stringByAppendingPathComponent:@"person.archive"];

// iOS 11+
NSError *error = nil;
NSData *archivedData = [NSKeyedArchiver archivedDataWithRootObject:person
                                          requiringSecureCoding:NO
                                                          error:&error];
if (error) {
    NSLog(@"Archive error: %@", error);
} else {
    [archivedData writeToFile:filePath atomically:YES];
    NSLog(@"Archived successfully");
}

// Unarchive (โหลด)
NSData *loadedData = [NSData dataWithContentsOfFile:filePath];
if (loadedData) {
    Person *loadedPerson = [NSKeyedUnarchiver unarchivedObjectOfClass:[Person class]
                                                             fromData:loadedData
                                                               error:&error];
    if (error) {
        NSLog(@"Unarchive error: %@", error);
    } else {
        NSLog(@"Loaded: %@", loadedPerson);
    }
}
```

### Archive หลาย Objects

```objc
// Archive collection
NSArray *people = @[
    [[Person alloc] initWithName:@"สมชาย" age:30 email:@"a@test.com"],
    [[Person alloc] initWithName:@"สมหญิง" age:25 email:@"b@test.com"],
    [[Person alloc] initWithName:@"สมศักดิ์" age:35 email:@"c@test.com"]
];

NSData *archivedPeople = [NSKeyedArchiver archivedDataWithRootObject:people
                                            requiringSecureCoding:NO
                                                            error:nil];

// Unarchive
NSArray *loadedPeople = [NSKeyedUnarchiver unarchivedObjectOfClasses:[NSSet setWithObjects:[NSArray class], [Person class], nil]
                                                            fromData:archivedPeople
                                                              error:nil];
```

### NSKeyedArchiver ใช้ multiple keys

```objc
// บันทึกหลาย objects ด้วย custom keys
NSMutableData *data = [NSMutableData data];
NSKeyedArchiver *archiver = [[NSKeyedArchiver alloc] initForWritingWithMutableData:data];
archiver.outputFormat = NSPropertyListXMLFormat_v1_0;  // readable XML

[archiver encodeObject:person forKey:@"person"];
[archiver encodeInteger:42 forKey:@"version"];
[archiver encodeBool:YES forKey:@"isValid"];
[archiver encodeObject:[NSDate date] forKey:@"timestamp"];
[archiver finishEncoding];

// บันทึกลงไฟล์
[data writeToFile:filePath atomically:YES];

// โหลดกลับ
NSData *loadData = [NSData dataWithContentsOfFile:filePath];
NSKeyedUnarchiver *unarchiver = [[NSKeyedUnarchiver alloc] initForReadingFromData:loadData error:nil];
unarchiver.requiresSecureCoding = NO;

Person *restoredPerson = [unarchiver decodeObjectForKey:@"person"];
NSInteger version = [unarchiver decodeIntegerForKey:@"version"];
BOOL isValid = [unarchiver decodeBoolForKey:@"isValid"];
NSDate *timestamp = [unarchiver decodeObjectForKey:@"timestamp"];
[unarchiver finishDecoding];

NSLog(@"Version: %ld, Valid: %@, Person: %@", (long)version, isValid ? @"YES" : @"NO", restoredPerson);
```

---

## 39.3 NSSecureCoding

NSSecureCoding เป็น protocol ที่ปลอดภัยกว่า NSCoding โดยป้องกัน substitution attacks:

```objc
// SecurePerson.h
@interface SecurePerson : NSObject <NSSecureCoding>

@property (nonatomic, copy) NSString *name;
@property (nonatomic, assign) NSInteger age;

@end

// SecurePerson.m
@implementation SecurePerson

// ต้อง implement method นี้
+ (BOOL)supportsSecureCoding {
    return YES;
}

- (void)encodeWithCoder:(NSCoder *)coder {
    [coder encodeObject:self.name forKey:@"name"];
    [coder encodeInteger:self.age forKey:@"age"];
}

- (instancetype)initWithCoder:(NSCoder *)coder {
    self = [super init];
    if (self) {
        // ใช้ decodeObjectOfClass: แทน decodeObjectForKey:
        _name = [coder decodeObjectOfClass:[NSString class] forKey:@"name"];
        _age = [coder decodeIntegerForKey:@"age"];
    }
    return self;
}

@end

// การใช้งาน
SecurePerson *person = [[SecurePerson alloc] init];
person.name = @"Test";
person.age = 25;

NSError *error = nil;

// ต้องใช้ requiringSecureCoding:YES
NSData *data = [NSKeyedArchiver archivedDataWithRootObject:person
                                    requiringSecureCoding:YES
                                                    error:&error];

SecurePerson *restored = [NSKeyedUnarchiver unarchivedObjectOfClass:[SecurePerson class]
                                                           fromData:data
                                                             error:&error];
```

---

## 39.4 NSJSONSerialization

### พื้นฐาน JSON

JSON (JavaScript Object Notation) เป็นรูปแบบข้อมูลที่นิยมใช้กับ REST APIs:

```json
{
    "name": "สมชาย",
    "age": 30,
    "email": "somchai@example.com",
    "active": true,
    "scores": [95, 87, 92],
    "address": {
        "city": "กรุงเทพ",
        "country": "Thailand"
    }
}
```

### JSON to NSDictionary/NSArray

```objc
// JSON String -> NSDictionary
NSString *jsonString = @"{\"name\":\"สมชาย\",\"age\":30,\"email\":\"somchai@test.com\"}";
NSData *jsonData = [jsonString dataUsingEncoding:NSUTF8StringEncoding];

NSError *error = nil;
NSDictionary *dict = [NSJSONSerialization JSONObjectWithData:jsonData
                                                     options:0
                                                       error:&error];

if (error) {
    NSLog(@"JSON parse error: %@", error);
} else {
    NSLog(@"Name: %@", dict[@"name"]);
    NSLog(@"Age: %@", dict[@"age"]);
}

// JSON Array String -> NSArray
NSString *jsonArrayString = @"[{\"id\":1,\"name\":\"Alice\"},{\"id\":2,\"name\":\"Bob\"}]";
NSData *arrayData = [jsonArrayString dataUsingEncoding:NSUTF8StringEncoding];

NSArray *array = [NSJSONSerialization JSONObjectWithData:arrayData
                                                 options:0
                                                   error:&error];
for (NSDictionary *item in array) {
    NSLog(@"ID: %@, Name: %@", item[@"id"], item[@"name"]);
}
```

### NSJSONReadingOptions

```objc
NSData *jsonData = ...;

// NSJSONReadingMutableContainers: ส่งคืน NSMutableDictionary/NSMutableArray
NSDictionary *mutableDict = [NSJSONSerialization JSONObjectWithData:jsonData
                                                            options:NSJSONReadingMutableContainers
                                                              error:&error];
// ตอนนี้ mutableDict เป็น NSMutableDictionary

// NSJSONReadingMutableLeaves: leaf values เป็น mutable
NSDictionary *mutableLeaves = [NSJSONSerialization JSONObjectWithData:jsonData
                                                              options:NSJSONReadingMutableLeaves
                                                                error:&error];

// NSJSONReadingFragmentsAllowed: อนุญาต JSON fragment (ไม่ต้องมี root object)
// เช่น JSON เป็นแค่ number หรือ string
NSData *fragmentData = [@"42" dataUsingEncoding:NSUTF8StringEncoding];
NSNumber *number = [NSJSONSerialization JSONObjectWithData:fragmentData
                                                   options:NSJSONReadingFragmentsAllowed
                                                     error:nil];
NSLog(@"Number: %@", number);
```

### NSDictionary/NSArray to JSON

```objc
// NSDictionary -> JSON Data
NSDictionary *userData = @{
    @"name": @"สมชาย ใจดี",
    @"age": @30,
    @"email": @"somchai@example.com",
    @"active": @YES,
    @"scores": @[@95, @87, @92],
    @"tags": @[@"admin", @"user"],
    @"address": @{
        @"city": @"กรุงเทพมหานคร",
        @"country": @"Thailand",
        @"zipCode": @"10110"
    }
};

NSError *error = nil;

// ตรวจสอบว่าสามารถ serialize ได้
if ([NSJSONSerialization isValidJSONObject:userData]) {
    // Compact output
    NSData *jsonData = [NSJSONSerialization dataWithJSONObject:userData
                                                       options:0
                                                         error:&error];
    
    // Pretty-printed output
    NSData *prettyData = [NSJSONSerialization dataWithJSONObject:userData
                                                         options:NSJSONWritingPrettyPrinted
                                                           error:&error];
    
    if (!error) {
        NSString *jsonString = [[NSString alloc] initWithData:prettyData
                                                     encoding:NSUTF8StringEncoding];
        NSLog(@"JSON:\n%@", jsonString);
    }
} else {
    NSLog(@"Object is not JSON-serializable");
}
```

### JSON ที่ Invalid

```objc
// ข้อมูลที่ NSJSONSerialization ไม่รองรับ:
// - NSDate (ต้องแปลงเป็น string ก่อน)
// - NSURL (ต้องใช้ .absoluteString)
// - Primitive types ที่ไม่ wrap ใน NSNumber
// - Custom objects โดยตรง

// ❌ Invalid
NSDictionary *invalid = @{
    @"date": [NSDate date],    // NSDate ไม่รองรับ!
    @"url": [NSURL URLWithString:@"https://example.com"]  // NSURL ไม่รองรับ!
};
// [NSJSONSerialization isValidJSONObject:invalid] = NO

// ✅ แก้ไข
NSDateFormatter *formatter = [[NSDateFormatter alloc] init];
formatter.dateFormat = @"yyyy-MM-dd'T'HH:mm:ssZ";

NSDictionary *valid = @{
    @"date": [formatter stringFromDate:[NSDate date]],  // แปลงเป็น String
    @"url": [NSURL URLWithString:@"https://example.com"].absoluteString  // ใช้ .absoluteString
};
```

---

## 39.5 Custom Object JSON Mapping

### แปลง JSON เป็น Custom Object

```objc
// Product.h
@interface Product : NSObject

@property (nonatomic, assign) NSInteger productId;
@property (nonatomic, copy) NSString *name;
@property (nonatomic, copy) NSString *description;
@property (nonatomic, assign) double price;
@property (nonatomic, assign) NSInteger stock;
@property (nonatomic, strong) NSArray<NSString *> *categories;
@property (nonatomic, strong) NSDictionary *specifications;

+ (instancetype)productFromDictionary:(NSDictionary *)dict;
- (NSDictionary *)toDictionary;

@end

// Product.m
@implementation Product

+ (instancetype)productFromDictionary:(NSDictionary *)dict {
    if (![dict isKindOfClass:[NSDictionary class]]) return nil;
    
    Product *product = [[Product alloc] init];
    
    // ใช้ safe value extraction
    product.productId = [dict[@"id"] integerValue];
    product.name = [dict[@"name"] isKindOfClass:[NSString class]] ? dict[@"name"] : @"";
    product.description = dict[@"description"];
    product.price = [dict[@"price"] doubleValue];
    product.stock = [dict[@"stock"] integerValue];
    
    // Handle arrays
    if ([dict[@"categories"] isKindOfClass:[NSArray class]]) {
        product.categories = dict[@"categories"];
    }
    
    // Handle nested objects
    if ([dict[@"specifications"] isKindOfClass:[NSDictionary class]]) {
        product.specifications = dict[@"specifications"];
    }
    
    return product;
}

- (NSDictionary *)toDictionary {
    NSMutableDictionary *dict = [NSMutableDictionary dictionary];
    
    dict[@"id"] = @(self.productId);
    dict[@"name"] = self.name ?: [NSNull null];
    dict[@"description"] = self.description ?: [NSNull null];
    dict[@"price"] = @(self.price);
    dict[@"stock"] = @(self.stock);
    
    if (self.categories) {
        dict[@"categories"] = self.categories;
    }
    
    if (self.specifications) {
        dict[@"specifications"] = self.specifications;
    }
    
    return [dict copy];
}

- (NSString *)toJSONString {
    NSDictionary *dict = [self toDictionary];
    NSError *error = nil;
    NSData *jsonData = [NSJSONSerialization dataWithJSONObject:dict
                                                       options:NSJSONWritingPrettyPrinted
                                                         error:&error];
    if (error) return nil;
    return [[NSString alloc] initWithData:jsonData encoding:NSUTF8StringEncoding];
}

@end
```

### JSON Model Mapper

```objc
// JSONMapper.h - Generic JSON to Object mapper
@interface JSONMapper : NSObject

+ (id)mapObject:(id)jsonObject toClass:(Class)targetClass;
+ (NSArray *)mapArray:(NSArray *)jsonArray toClass:(Class)targetClass;

@end

@implementation JSONMapper

+ (id)mapObject:(id)jsonObject toClass:(Class)targetClass {
    if (![jsonObject isKindOfClass:[NSDictionary class]]) return nil;
    
    // ตรวจสอบว่า class มี method fromDictionary: หรือไม่
    if ([targetClass respondsToSelector:@selector(productFromDictionary:)] ||
        [targetClass respondsToSelector:NSSelectorFromString(@"objectFromDictionary:")]) {
        
        SEL selector = NSSelectorFromString(@"objectFromDictionary:");
        if ([targetClass respondsToSelector:selector]) {
            return [targetClass performSelector:selector withObject:jsonObject];
        }
    }
    
    return nil;
}

+ (NSArray *)mapArray:(NSArray *)jsonArray toClass:(Class)targetClass {
    if (![jsonArray isKindOfClass:[NSArray class]]) return @[];
    
    NSMutableArray *result = [NSMutableArray arrayWithCapacity:jsonArray.count];
    
    for (id item in jsonArray) {
        id mapped = [self mapObject:item toClass:targetClass];
        if (mapped) {
            [result addObject:mapped];
        }
    }
    
    return [result copy];
}

@end
```

### ตัวอย่าง API Response Parsing

```objc
// API Response Parser
@interface APIResponseParser : NSObject

+ (NSArray<Product *> *)parseProductsFromData:(NSData *)data error:(NSError **)error;
+ (Product *)parseProductFromData:(NSData *)data error:(NSError **)error;

@end

@implementation APIResponseParser

+ (NSArray<Product *> *)parseProductsFromData:(NSData *)data error:(NSError **)error {
    if (!data) {
        if (error) *error = [NSError errorWithDomain:@"ParseError" code:0
                                            userInfo:@{NSLocalizedDescriptionKey: @"No data"}];
        return nil;
    }
    
    NSError *jsonError = nil;
    id jsonObject = [NSJSONSerialization JSONObjectWithData:data options:0 error:&jsonError];
    
    if (jsonError) {
        if (error) *error = jsonError;
        return nil;
    }
    
    // Handle response formats:
    // 1. Array: [{...}, {...}]
    // 2. Object with data key: {"data": [{...}, {...}], "total": 100}
    // 3. Object with items key: {"items": [{...}]}
    
    NSArray *productsData = nil;
    
    if ([jsonObject isKindOfClass:[NSArray class]]) {
        productsData = jsonObject;
    } else if ([jsonObject isKindOfClass:[NSDictionary class]]) {
        productsData = jsonObject[@"data"] ?: jsonObject[@"items"] ?: jsonObject[@"products"];
    }
    
    if (![productsData isKindOfClass:[NSArray class]]) {
        if (error) *error = [NSError errorWithDomain:@"ParseError" code:1
                                            userInfo:@{NSLocalizedDescriptionKey: @"Unexpected format"}];
        return nil;
    }
    
    NSMutableArray *products = [NSMutableArray array];
    for (NSDictionary *dict in productsData) {
        Product *product = [Product productFromDictionary:dict];
        if (product) [products addObject:product];
    }
    
    return [products copy];
}

+ (Product *)parseProductFromData:(NSData *)data error:(NSError **)error {
    NSError *jsonError = nil;
    NSDictionary *dict = [NSJSONSerialization JSONObjectWithData:data options:0 error:&jsonError];
    
    if (jsonError) {
        if (error) *error = jsonError;
        return nil;
    }
    
    return [Product productFromDictionary:dict];
}

@end
```

---

## 39.6 Property List (Plist) Serialization

### NSPropertyListSerialization

```objc
// Plist รองรับ types เหล่านี้:
// - NSString
// - NSData
// - NSNumber (int, float, bool)
// - NSDate
// - NSArray (elements ต้องเป็น plist types)
// - NSDictionary (keys ต้องเป็น NSString)

NSDictionary *plistObject = @{
    @"name": @"MyApp Config",
    @"version": @(2),
    @"enableLogging": @YES,
    @"maxRetries": @(3),
    @"timeout": @(30.5),
    @"serverDate": [NSDate date],
    @"endpoints": @[@"api.example.com", @"backup.example.com"],
    @"features": @{
        @"darkMode": @YES,
        @"notifications": @YES
    }
};

NSError *error = nil;

// Serialize เป็น XML format
NSData *xmlData = [NSPropertyListSerialization dataWithPropertyList:plistObject
                                                             format:NSPropertyListXMLFormat_v1_0
                                                            options:0
                                                              error:&error];
if (!error) {
    NSString *xmlString = [[NSString alloc] initWithData:xmlData encoding:NSUTF8StringEncoding];
    NSLog(@"Plist XML:\n%@", xmlString);
}

// Serialize เป็น Binary format (compact, iOS preferred)
NSData *binaryData = [NSPropertyListSerialization dataWithPropertyList:plistObject
                                                                format:NSPropertyListBinaryFormat_v1_0
                                                               options:0
                                                                 error:&error];

// Deserialize กลับมา
NSPropertyListFormat format;
id restoredObject = [NSPropertyListSerialization propertyListWithData:binaryData
                                                              options:NSPropertyListImmutable
                                                               format:&format
                                                                error:&error];

NSDictionary *config = (NSDictionary *)restoredObject;
NSLog(@"App: %@, Version: %@", config[@"name"], config[@"version"]);
```

### Plist File Operations

```objc
@interface PlistManager : NSObject

+ (BOOL)writePlist:(id)plist toFile:(NSString *)filename;
+ (id)readPlistFromFile:(NSString *)filename;
+ (id)readBundlePlist:(NSString *)filename;

@end

@implementation PlistManager

+ (NSString *)pathForFile:(NSString *)filename {
    NSArray *paths = NSSearchPathForDirectoriesInDomains(NSDocumentDirectory, NSUserDomainMask, YES);
    NSString *docPath = [paths firstObject];
    return [docPath stringByAppendingPathComponent:filename];
}

+ (BOOL)writePlist:(id)plist toFile:(NSString *)filename {
    NSString *filePath = [self pathForFile:filename];
    NSError *error = nil;
    
    NSData *data = [NSPropertyListSerialization dataWithPropertyList:plist
                                                              format:NSPropertyListBinaryFormat_v1_0
                                                             options:0
                                                               error:&error];
    if (error) {
        NSLog(@"Plist serialization error: %@", error);
        return NO;
    }
    
    return [data writeToFile:filePath atomically:YES];
}

+ (id)readPlistFromFile:(NSString *)filename {
    NSString *filePath = [self pathForFile:filename];
    NSData *data = [NSData dataWithContentsOfFile:filePath];
    
    if (!data) return nil;
    
    NSError *error = nil;
    id plist = [NSPropertyListSerialization propertyListWithData:data
                                                         options:NSPropertyListImmutable
                                                          format:nil
                                                           error:&error];
    if (error) {
        NSLog(@"Plist parse error: %@", error);
    }
    
    return plist;
}

+ (id)readBundlePlist:(NSString *)filename {
    NSString *path = [[NSBundle mainBundle] pathForResource:filename ofType:@"plist"];
    if (!path) return nil;
    
    NSData *data = [NSData dataWithContentsOfFile:path];
    if (!data) return nil;
    
    return [NSPropertyListSerialization propertyListWithData:data
                                                     options:NSPropertyListImmutable
                                                      format:nil
                                                       error:nil];
}

@end
```

---

## 39.7 Complex Object Graph Archiving

### Object ที่มี Relationships

```objc
// Department.h
@interface Department : NSObject <NSCoding>
@property (nonatomic, copy) NSString *name;
@property (nonatomic, strong) NSArray<Person *> *employees;
@property (nonatomic, strong) Person *manager;
@end

// Department.m
@implementation Department

- (void)encodeWithCoder:(NSCoder *)coder {
    [coder encodeObject:self.name forKey:@"name"];
    [coder encodeObject:self.employees forKey:@"employees"];
    [coder encodeObject:self.manager forKey:@"manager"];
}

- (instancetype)initWithCoder:(NSCoder *)coder {
    self = [super init];
    if (self) {
        _name = [coder decodeObjectForKey:@"name"];
        _employees = [coder decodeObjectForKey:@"employees"];
        _manager = [coder decodeObjectForKey:@"manager"];
    }
    return self;
}

@end

// การใช้งาน
Person *manager = [[Person alloc] initWithName:@"ผู้จัดการ" age:45 email:@"mgr@test.com"];
NSArray *employees = @[
    [[Person alloc] initWithName:@"พนักงาน A" age:28 email:@"a@test.com"],
    [[Person alloc] initWithName:@"พนักงาน B" age:32 email:@"b@test.com"]
];

Department *dept = [[Department alloc] init];
dept.name = @"Engineering";
dept.manager = manager;
dept.employees = employees;

// Archive ทั้ง graph
NSData *deptData = [NSKeyedArchiver archivedDataWithRootObject:dept
                                        requiringSecureCoding:NO
                                                        error:nil];
NSLog(@"Archived department size: %lu bytes", (unsigned long)deptData.length);

// Restore
Department *restoredDept = [NSKeyedUnarchiver unarchivedObjectOfClass:[Department class]
                                                             fromData:deptData
                                                               error:nil];
NSLog(@"Department: %@, Manager: %@, Employees: %lu",
      restoredDept.name,
      restoredDept.manager.name,
      (unsigned long)restoredDept.employees.count);
```

---

## 39.8 Game State Example

```objc
// GameState.h - ตัวอย่าง complete serialization
@interface GameState : NSObject <NSCoding>

@property (nonatomic, assign) NSInteger level;
@property (nonatomic, assign) NSInteger score;
@property (nonatomic, assign) NSInteger lives;
@property (nonatomic, strong) NSDate *lastPlayedDate;
@property (nonatomic, copy) NSString *playerName;
@property (nonatomic, strong) NSArray<NSString *> *achievements;
@property (nonatomic, strong) NSDictionary<NSString *, NSNumber *> *inventory;

+ (instancetype)loadFromFile;
- (BOOL)saveToFile;
+ (void)deleteFile;

@end

@implementation GameState

static NSString * const kSavePath = @"game_save.dat";

+ (NSString *)savePath {
    NSArray *paths = NSSearchPathForDirectoriesInDomains(NSDocumentDirectory, NSUserDomainMask, YES);
    return [[paths firstObject] stringByAppendingPathComponent:kSavePath];
}

- (instancetype)init {
    self = [super init];
    if (self) {
        _level = 1;
        _score = 0;
        _lives = 3;
        _achievements = @[];
        _inventory = @{};
        _lastPlayedDate = [NSDate date];
    }
    return self;
}

- (void)encodeWithCoder:(NSCoder *)coder {
    [coder encodeInteger:self.level forKey:@"level"];
    [coder encodeInteger:self.score forKey:@"score"];
    [coder encodeInteger:self.lives forKey:@"lives"];
    [coder encodeObject:self.lastPlayedDate forKey:@"lastPlayedDate"];
    [coder encodeObject:self.playerName forKey:@"playerName"];
    [coder encodeObject:self.achievements forKey:@"achievements"];
    [coder encodeObject:self.inventory forKey:@"inventory"];
}

- (instancetype)initWithCoder:(NSCoder *)coder {
    self = [super init];
    if (self) {
        _level = [coder decodeIntegerForKey:@"level"];
        _score = [coder decodeIntegerForKey:@"score"];
        _lives = [coder decodeIntegerForKey:@"lives"];
        _lastPlayedDate = [coder decodeObjectForKey:@"lastPlayedDate"];
        _playerName = [coder decodeObjectForKey:@"playerName"];
        _achievements = [coder decodeObjectForKey:@"achievements"];
        _inventory = [coder decodeObjectForKey:@"inventory"];
    }
    return self;
}

+ (instancetype)loadFromFile {
    NSString *path = [self savePath];
    NSData *data = [NSData dataWithContentsOfFile:path];
    
    if (!data) {
        NSLog(@"No saved game found, creating new game state");
        return [[GameState alloc] init];
    }
    
    NSError *error = nil;
    GameState *state = [NSKeyedUnarchiver unarchivedObjectOfClass:[GameState class]
                                                        fromData:data
                                                          error:&error];
    if (error) {
        NSLog(@"Failed to load game: %@", error);
        return [[GameState alloc] init];
    }
    
    NSLog(@"Game loaded! Level: %ld, Score: %ld", (long)state.level, (long)state.score);
    return state;
}

- (BOOL)saveToFile {
    NSError *error = nil;
    NSData *data = [NSKeyedArchiver archivedDataWithRootObject:self
                                        requiringSecureCoding:NO
                                                        error:&error];
    if (error) {
        NSLog(@"Failed to archive game: %@", error);
        return NO;
    }
    
    BOOL success = [data writeToFile:[[self class] savePath] atomically:YES];
    if (success) {
        NSLog(@"Game saved successfully!");
    }
    return success;
}

+ (void)deleteFile {
    NSError *error = nil;
    [[NSFileManager defaultManager] removeItemAtPath:[self savePath] error:&error];
    NSLog(@"Save file deleted");
}

@end

// การใช้งาน
GameState *state = [GameState loadFromFile];
state.playerName = @"Player1";
state.level = 5;
state.score = 12500;

// เพิ่ม achievement
NSMutableArray *achiev = [state.achievements mutableCopy];
[achiev addObject:@"speed_runner"];
state.achievements = [achiev copy];

// บันทึก
[state saveToFile];
```

---

## 39.9 JSON สำหรับ Web API

### URLSession + JSON Parsing

```objc
@interface APIClient : NSObject

typedef void(^APICompletion)(id result, NSError *error);

- (void)fetchUsersWithCompletion:(void(^)(NSArray *users, NSError *error))completion;
- (void)createUser:(NSDictionary *)userData completion:(APICompletion)completion;

@end

@implementation APIClient

- (void)fetchUsersWithCompletion:(void(^)(NSArray *users, NSError *error))completion {
    NSURL *url = [NSURL URLWithString:@"https://jsonplaceholder.typicode.com/users"];
    
    NSURLSessionDataTask *task = [[NSURLSession sharedSession]
                                  dataTaskWithURL:url
                                  completionHandler:^(NSData *data, NSURLResponse *response, NSError *error) {
        if (error) {
            dispatch_async(dispatch_get_main_queue(), ^{
                completion(nil, error);
            });
            return;
        }
        
        NSError *jsonError = nil;
        NSArray *users = [NSJSONSerialization JSONObjectWithData:data options:0 error:&jsonError];
        
        dispatch_async(dispatch_get_main_queue(), ^{
            if (jsonError) {
                completion(nil, jsonError);
            } else {
                completion(users, nil);
            }
        });
    }];
    
    [task resume];
}

- (void)createUser:(NSDictionary *)userData completion:(APICompletion)completion {
    NSURL *url = [NSURL URLWithString:@"https://jsonplaceholder.typicode.com/users"];
    NSMutableURLRequest *request = [NSMutableURLRequest requestWithURL:url];
    request.HTTPMethod = @"POST";
    [request setValue:@"application/json" forHTTPHeaderField:@"Content-Type"];
    
    NSError *jsonError = nil;
    NSData *body = [NSJSONSerialization dataWithJSONObject:userData options:0 error:&jsonError];
    
    if (jsonError) {
        completion(nil, jsonError);
        return;
    }
    
    request.HTTPBody = body;
    
    NSURLSessionDataTask *task = [[NSURLSession sharedSession]
                                  dataTaskWithRequest:request
                                  completionHandler:^(NSData *data, NSURLResponse *response, NSError *error) {
        if (error) {
            dispatch_async(dispatch_get_main_queue(), ^{
                completion(nil, error);
            });
            return;
        }
        
        NSError *parseError = nil;
        id result = [NSJSONSerialization JSONObjectWithData:data options:0 error:&parseError];
        
        dispatch_async(dispatch_get_main_queue(), ^{
            completion(parseError ? nil : result, parseError);
        });
    }];
    
    [task resume];
}

@end

// การใช้งาน
APIClient *client = [[APIClient alloc] init];
[client fetchUsersWithCompletion:^(NSArray *users, NSError *error) {
    if (error) {
        NSLog(@"Error: %@", error);
        return;
    }
    
    for (NSDictionary *user in users) {
        NSLog(@"User: %@ - %@", user[@"name"], user[@"email"]);
    }
}];
```

---

## 39.10 Codable Pattern Preparation

เตรียมความพร้อมสำหรับ Swift Codable pattern ที่ใช้ใน modern iOS development:

```objc
// Objective-C approach ที่คล้าย Swift Codable
@protocol JSONCodable <NSObject>

+ (instancetype)fromJSON:(NSDictionary *)json;
- (NSDictionary *)toJSON;

@end

// Base class
@interface JSONModel : NSObject <NSCoding, JSONCodable>

+ (NSArray *)fromJSONArray:(NSArray *)jsonArray;
+ (instancetype)fromJSONData:(NSData *)data error:(NSError **)error;
- (NSData *)toJSONData:(NSError **)error;

@end

@implementation JSONModel

+ (instancetype)fromJSON:(NSDictionary *)json {
    return [[self alloc] init];  // Override ใน subclasses
}

- (NSDictionary *)toJSON {
    return @{};  // Override ใน subclasses
}

+ (NSArray *)fromJSONArray:(NSArray *)jsonArray {
    NSMutableArray *result = [NSMutableArray array];
    for (NSDictionary *json in jsonArray) {
        id obj = [self fromJSON:json];
        if (obj) [result addObject:obj];
    }
    return [result copy];
}

+ (instancetype)fromJSONData:(NSData *)data error:(NSError **)error {
    NSDictionary *json = [NSJSONSerialization JSONObjectWithData:data options:0 error:error];
    if (*error) return nil;
    return [self fromJSON:json];
}

- (NSData *)toJSONData:(NSError **)error {
    return [NSJSONSerialization dataWithJSONObject:[self toJSON] options:0 error:error];
}

// NSCoding ใช้ JSON เป็น intermediate
- (void)encodeWithCoder:(NSCoder *)coder {
    NSError *error = nil;
    NSData *jsonData = [self toJSONData:&error];
    if (!error) {
        [coder encodeObject:jsonData forKey:@"jsonData"];
    }
}

- (instancetype)initWithCoder:(NSCoder *)coder {
    NSData *jsonData = [coder decodeObjectForKey:@"jsonData"];
    if (!jsonData) return nil;
    
    NSError *error = nil;
    NSDictionary *json = [NSJSONSerialization JSONObjectWithData:jsonData options:0 error:&error];
    if (error) return nil;
    
    return [[self class] fromJSON:json];
}

@end

// ตัวอย่าง Subclass
@implementation Product (JSONCodable)

+ (instancetype)fromJSON:(NSDictionary *)json {
    Product *product = [[Product alloc] init];
    product.productId = [json[@"id"] integerValue];
    product.name = json[@"name"];
    product.price = [json[@"price"] doubleValue];
    product.stock = [json[@"stock"] integerValue];
    return product;
}

- (NSDictionary *)toJSON {
    return @{
        @"id": @(self.productId),
        @"name": self.name ?: @"",
        @"price": @(self.price),
        @"stock": @(self.stock)
    };
}

@end
```

---

## แบบฝึกหัด (Practice Exercises)

### Exercise 1: Student Grade Book
สร้าง `Student` class ที่ implement NSCoding:
- Properties: name, studentId, grades (Dictionary), courses (Array)
- Archive/unarchive student ลงไฟล์
- Archive array ของ students ทั้งหมด

### Exercise 2: Shopping Cart Serialization
สร้าง cart ที่:
- บันทึก cart items เป็น NSKeyedArchiver
- โหลดกลับมาได้เมื่อ app restart
- Export เป็น JSON สำหรับส่งไป server

### Exercise 3: JSON API Client
สร้าง API client ที่:
- Fetch posts จาก jsonplaceholder.typicode.com/posts
- Parse JSON เป็น `Post` objects
- Cache ผล JSON response ลง Caches directory
- ใช้ cached data ถ้าไม่มีเน็ต

### Exercise 4: Custom NSCoder
สร้าง `EncryptedCoder` ที่:
- Wrap NSKeyedArchiver
- เข้ารหัสข้อมูลก่อนบันทึก
- ถอดรหัสเมื่ออ่าน

### Exercise 5: Version Migration
สร้าง `UserProfile` ที่:
- Version 1: name, email
- Version 2: เพิ่ม profileImage
- Version 3: เพิ่ม preferences (dict)
- Handle backward compatibility ใน initWithCoder:

### Exercise 6: JSON Schema Validator
สร้าง `JSONValidator` ที่:
- ตรวจสอบว่า JSON มี required keys ครบ
- ตรวจสอบ types ของแต่ละ key
- คืน validation errors

### Exercise 7: Plist Configuration
สร้าง app configuration ที่:
- Default config อยู่ใน bundle plist
- Override config อยู่ใน Documents plist
- Merge ทั้งสองเข้าด้วยกัน

### Exercise 8: Object Graph Serialization
สร้าง class ที่มี circular references:
- `Company` มี list ของ `Employee`
- แต่ละ `Employee` มี reference กลับไป `Company`
- Archive/unarchive โดยไม่ทำให้ infinite loop

### Exercise 9: JSON Diff
สร้าง function ที่:
- เปรียบเทียบ JSON สอง objects
- แสดงว่า keys อะไรเปลี่ยน, เพิ่ม, หรือลบ
- คืนผลเป็น structured diff

### Exercise 10: Multi-format Serializer
สร้าง `Serializer` ที่:
- รองรับ JSON, Plist, NSKeyedArchiver
- เลือก format ได้
- Compare ขนาดของแต่ละ format

---

## สรุป

Serialization ใน Objective-C มีหลายวิธี:

1. **NSCoding + NSKeyedArchiver**: เหมาะสำหรับ Objective-C objects ซับซ้อน
2. **NSJSONSerialization**: สำหรับ web APIs และข้อมูล JSON
3. **NSPropertyListSerialization**: สำหรับ configuration files
4. **ใช้ requiringSecureCoding:YES** เมื่อ unarchive data จากแหล่งที่ไม่น่าเชื่อถือ
5. **Validate JSON** ก่อน serialize เสมอ
6. **Handle errors** ทุกขั้นตอน

ในบทถัดไปเราจะเรียนรู้ Objective-C Runtime ซึ่งเป็น mechanism เบื้องหลังที่ทำให้ Objective-C มีความยืดหยุ่นสูง
