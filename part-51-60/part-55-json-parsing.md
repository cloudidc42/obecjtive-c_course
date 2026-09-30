# ตอนที่ 55: JSON Parsing ใน Objective-C

## บทนำ

JSON (JavaScript Object Notation) เป็นรูปแบบข้อมูลที่ได้รับความนิยมมากที่สุดในการแลกเปลี่ยนข้อมูลระหว่าง Client และ Server ใน Objective-C เราใช้ **NSJSONSerialization** เป็น API หลักในการแปลง JSON เข้าและออกจาก Objective-C Objects

ในบทนี้เราจะเรียนรู้:
- การแปลง JSON เป็น NSDictionary/NSArray
- การแปลง NSDictionary/NSArray เป็น JSON
- การจัดการ JSON ที่ซับซ้อน
- การ Mapping JSON เป็น Model Objects
- Performance สำหรับ JSON ขนาดใหญ่
- ตัวอย่างจริงกับ GitHub API

---

## 55.1 NSJSONSerialization

### พื้นฐาน NSJSONSerialization

NSJSONSerialization เป็น Class ใน Foundation Framework ที่ใช้แปลงข้อมูลระหว่าง JSON และ Objective-C Objects

```objc
// JSON ที่ถูกต้องใน Objective-C Objects:
// - NSString  (string)
// - NSNumber  (number, boolean)
// - NSArray   (array)
// - NSDictionary (object)
// - NSNull    (null)

// ตรวจสอบว่า Object แปลงเป็น JSON ได้หรือไม่
BOOL isValid = [NSJSONSerialization isValidJSONObject:myObject];

// Serialization Options
NSJSONWritingOptions writeOptions = NSJSONWritingPrettyPrinted;  // จัดรูปแบบ
// iOS 13+
NSJSONWritingOptions sortedKeys = NSJSONWritingSortedKeys;       // เรียง Keys

// Deserialization Options  
NSJSONReadingOptions readOptions = NSJSONReadingAllowFragments;  // อนุญาต Fragment
NSJSONReadingOptions mutableContainers = NSJSONReadingMutableContainers; // NSMutableArray/NSMutableDictionary
NSJSONReadingOptions mutableLeaves = NSJSONReadingMutableLeaves; // NSMutableString
```

---

## 55.2 JSON เป็น NSDictionary/NSArray

### แปลง JSON String เป็น Object

```objc
// JSON String ตัวอย่าง
NSString *jsonString = @"{\"name\":\"John\",\"age\":30,\"isActive\":true}";

// แปลง String เป็น Data
NSData *jsonData = [jsonString dataUsingEncoding:NSUTF8StringEncoding];

// แปลง Data เป็น Object
NSError *error;
NSDictionary *dict = [NSJSONSerialization JSONObjectWithData:jsonData
                                                     options:NSJSONReadingAllowFragments
                                                       error:&error];
if (error) {
    NSLog(@"JSON Parse Error: %@", error.localizedDescription);
    NSLog(@"Error Code: %ld", (long)error.code);
} else {
    NSString *name = dict[@"name"];          // "John"
    NSNumber *age = dict[@"age"];            // 30
    BOOL isActive = [dict[@"isActive"] boolValue]; // true
    NSLog(@"Name: %@, Age: %@, Active: %@", name, age, @(isActive));
}
```

### แปลง JSON Array

```objc
// JSON Array String
NSString *jsonArrayString = @"[{\"id\":1,\"name\":\"Alice\"},{\"id\":2,\"name\":\"Bob\"}]";
NSData *arrayData = [jsonArrayString dataUsingEncoding:NSUTF8StringEncoding];

NSError *error;
NSArray *users = [NSJSONSerialization JSONObjectWithData:arrayData
                                                options:0
                                                  error:&error];
if (!error && [users isKindOfClass:[NSArray class]]) {
    for (NSDictionary *user in users) {
        NSLog(@"User ID: %@, Name: %@", user[@"id"], user[@"name"]);
    }
}
```

### แปลง JSON ที่ได้จาก Network

```objc
// จาก NSURLSession Completion Handler
NSURLSessionDataTask *task = [session dataTaskWithURL:url
                                   completionHandler:^(NSData *data, 
                                                       NSURLResponse *response, 
                                                       NSError *networkError) {
    if (networkError) {
        NSLog(@"Network Error: %@", networkError.localizedDescription);
        return;
    }
    
    // ตรวจสอบว่ามีข้อมูล
    if (!data || data.length == 0) {
        NSLog(@"Empty response");
        return;
    }
    
    // Debug: แสดง Raw Response
    NSString *rawResponse = [[NSString alloc] initWithData:data encoding:NSUTF8StringEncoding];
    NSLog(@"Raw Response: %@", rawResponse);
    
    // Parse JSON
    NSError *jsonError;
    id jsonObject = [NSJSONSerialization JSONObjectWithData:data
                                                   options:NSJSONReadingAllowFragments
                                                     error:&jsonError];
    if (jsonError) {
        NSLog(@"JSON Parse Error: %@", jsonError);
        return;
    }
    
    // ตรวจสอบประเภท
    if ([jsonObject isKindOfClass:[NSDictionary class]]) {
        NSDictionary *dict = (NSDictionary *)jsonObject;
        NSLog(@"Dictionary: %@", dict);
        
    } else if ([jsonObject isKindOfClass:[NSArray class]]) {
        NSArray *array = (NSArray *)jsonObject;
        NSLog(@"Array with %lu items", (unsigned long)array.count);
        
    } else {
        NSLog(@"Unexpected JSON type: %@", [jsonObject class]);
    }
}];
[task resume];
```

---

## 55.3 NSDictionary/NSArray เป็น JSON

### แปลง Object เป็น JSON

```objc
// NSDictionary -> JSON
NSDictionary *userData = @{
    @"name": @"สมชาย ใจดี",
    @"email": @"somchai@example.com",
    @"age": @28,
    @"isVerified": @YES,
    @"tags": @[@"admin", @"user"],
    @"address": @{
        @"street": @"123 ถนนสุขุมวิท",
        @"city": @"กรุงเทพมหานคร",
        @"postcode": @"10110"
    }
};

NSError *error;
NSData *jsonData = [NSJSONSerialization dataWithJSONObject:userData
                                                   options:NSJSONWritingPrettyPrinted
                                                     error:&error];
if (!error) {
    NSString *jsonString = [[NSString alloc] initWithData:jsonData 
                                                 encoding:NSUTF8StringEncoding];
    NSLog(@"JSON:\n%@", jsonString);
}

// NSArray -> JSON
NSArray *items = @[
    @{@"id": @1, @"name": @"Item 1"},
    @{@"id": @2, @"name": @"Item 2"},
    @{@"id": @3, @"name": @"Item 3"}
];

NSError *arrayError;
NSData *arrayJSON = [NSJSONSerialization dataWithJSONObject:items
                                                    options:0
                                                      error:&arrayError];

// ส่งไปพร้อม HTTP Request
if (!arrayError) {
    NSMutableURLRequest *request = [NSMutableURLRequest requestWithURL:url];
    [request setHTTPMethod:@"POST"];
    [request setValue:@"application/json" forHTTPHeaderField:@"Content-Type"];
    [request setHTTPBody:arrayJSON];
}
```

### JSON Encoding ด้วย Thai Characters

```objc
// ตรวจสอบการ Encode ภาษาไทย
NSString *thaiText = @"สวัสดีครับ";
NSDictionary *thaiData = @{@"message": thaiText};

NSError *error;
NSData *jsonData = [NSJSONSerialization dataWithJSONObject:thaiData
                                                   options:NSJSONWritingPrettyPrinted
                                                     error:&error];

NSString *jsonString = [[NSString alloc] initWithData:jsonData 
                                             encoding:NSUTF8StringEncoding];
NSLog(@"JSON with Thai: %@", jsonString);
// Output: {"message": "สวัสดีครับ"}

// หรือใช้ Unicode Encoding Options (iOS 17+)
// options: NSJSONWritingWithoutEscapingSlashes | NSJSONWritingPrettyPrinted
```

---

## 55.4 Handling Nested JSON

### JSON ที่ซับซ้อน

```objc
// JSON ตัวอย่างที่ซับซ้อน
NSString *complexJSON = @"{"
    "\"user\": {"
        "\"id\": 123,"
        "\"profile\": {"
            "\"firstName\": \"John\","
            "\"lastName\": \"Doe\","
            "\"avatar\": null"
        "},"
        "\"orders\": ["
            "{"
                "\"orderId\": \"ORD001\","
                "\"items\": ["
                    "{\"productId\": 1, \"quantity\": 2, \"price\": 299.99},"
                    "{\"productId\": 2, \"quantity\": 1, \"price\": 149.50}"
                "],"
                "\"status\": \"delivered\","
                "\"total\": 749.48"
            "}"
        "]"
    "}"
"}";

NSData *jsonData = [complexJSON dataUsingEncoding:NSUTF8StringEncoding];
NSError *error;
NSDictionary *root = [NSJSONSerialization JSONObjectWithData:jsonData
                                                     options:0
                                                       error:&error];

// ดึงข้อมูลแบบ Safe
NSDictionary *user = root[@"user"];
NSDictionary *profile = user[@"profile"];
NSArray *orders = user[@"orders"];

NSString *firstName = profile[@"firstName"];  // "John"
NSString *avatar = profile[@"avatar"];        // NSNull!

// Handle Nested Safely
NSString *lastName = [self safeStringFromDict:profile key:@"lastName"];
NSNumber *userID = [self safeNumberFromDict:user key:@"id"];

// Safe Accessor Methods
- (NSString *)safeStringFromDict:(NSDictionary *)dict key:(NSString *)key {
    id value = dict[key];
    if ([value isKindOfClass:[NSString class]]) {
        return (NSString *)value;
    }
    return nil;
}

- (NSNumber *)safeNumberFromDict:(NSDictionary *)dict key:(NSString *)key {
    id value = dict[key];
    if ([value isKindOfClass:[NSNumber class]]) {
        return (NSNumber *)value;
    }
    return nil;
}

// ดึงข้อมูลจาก Array ใน JSON
for (NSDictionary *order in orders) {
    NSString *orderId = order[@"orderId"];
    NSArray *items = order[@"items"];
    NSNumber *total = order[@"total"];
    
    NSLog(@"Order: %@, Total: %@", orderId, total);
    
    for (NSDictionary *item in items) {
        NSNumber *productId = item[@"productId"];
        NSNumber *quantity = item[@"quantity"];
        NSNumber *price = item[@"price"];
        NSLog(@"  Product %@: %@ x $%@", productId, quantity, price);
    }
}
```

### Value Path Accessor (เข้าถึง Nested ด้วย Dot Notation)

```objc
// Utility สำหรับเข้าถึง Nested JSON ด้วย Path
@interface JSONAccessor : NSObject

+ (id)valueFromDictionary:(NSDictionary *)dict path:(NSString *)path;
+ (NSString *)stringFromDictionary:(NSDictionary *)dict path:(NSString *)path;
+ (NSInteger)integerFromDictionary:(NSDictionary *)dict path:(NSString *)path;
+ (NSArray *)arrayFromDictionary:(NSDictionary *)dict path:(NSString *)path;

@end

@implementation JSONAccessor

+ (id)valueFromDictionary:(NSDictionary *)dict path:(NSString *)path {
    NSArray *components = [path componentsSeparatedByString:@"."];
    id current = dict;
    
    for (NSString *key in components) {
        if ([current isKindOfClass:[NSDictionary class]]) {
            current = [(NSDictionary *)current objectForKey:key];
        } else if ([current isKindOfClass:[NSArray class]]) {
            NSInteger index = [key integerValue];
            NSArray *array = (NSArray *)current;
            if (index >= 0 && index < (NSInteger)array.count) {
                current = array[index];
            } else {
                return nil;
            }
        } else {
            return nil;
        }
        
        if (!current || [current isKindOfClass:[NSNull class]]) {
            return nil;
        }
    }
    
    return current;
}

+ (NSString *)stringFromDictionary:(NSDictionary *)dict path:(NSString *)path {
    id value = [self valueFromDictionary:dict path:path];
    if ([value isKindOfClass:[NSString class]]) return value;
    if ([value isKindOfClass:[NSNumber class]]) return [value stringValue];
    return nil;
}

+ (NSInteger)integerFromDictionary:(NSDictionary *)dict path:(NSString *)path {
    id value = [self valueFromDictionary:dict path:path];
    if ([value isKindOfClass:[NSNumber class]]) return [value integerValue];
    if ([value isKindOfClass:[NSString class]]) return [value integerValue];
    return 0;
}

+ (NSArray *)arrayFromDictionary:(NSDictionary *)dict path:(NSString *)path {
    id value = [self valueFromDictionary:dict path:path];
    if ([value isKindOfClass:[NSArray class]]) return value;
    return @[];
}

@end

// การใช้งาน
NSDictionary *response = /* JSON ที่ได้จาก API */;
NSString *firstName = [JSONAccessor stringFromDictionary:response path:@"user.profile.firstName"];
NSInteger userID = [JSONAccessor integerFromDictionary:response path:@"user.id"];
NSArray *orders = [JSONAccessor arrayFromDictionary:response path:@"user.orders"];
NSString *firstOrderId = [JSONAccessor stringFromDictionary:response path:@"user.orders.0.orderId"];
```

---

## 55.5 Error Handling During Parsing

### ประเภทของ JSON Errors

```objc
// NSJSONSerialization Error Codes
// NSPropertyListReadCorruptError (3840) - JSON ผิดรูปแบบ
// NSPropertyListReadUnknownVersionError (3841) - ไม่รู้จัก version
// NSPropertyListReadStreamError (3842) - Stream error
// NSPropertyListWriteStreamError (3851) - Write stream error

// Error Handling ที่ครอบคลุม
@interface SafeJSONParser : NSObject

+ (id)parseData:(NSData *)data error:(NSError **)outError;
+ (NSData *)serializeObject:(id)object error:(NSError **)outError;

@end

@implementation SafeJSONParser

+ (id)parseData:(NSData *)data error:(NSError **)outError {
    // ตรวจสอบ Input
    if (!data) {
        if (outError) {
            *outError = [NSError errorWithDomain:@"JSONParserError"
                                           code:1001
                                       userInfo:@{NSLocalizedDescriptionKey: @"Data is nil"}];
        }
        return nil;
    }
    
    if (data.length == 0) {
        if (outError) {
            *outError = [NSError errorWithDomain:@"JSONParserError"
                                           code:1002
                                       userInfo:@{NSLocalizedDescriptionKey: @"Data is empty"}];
        }
        return nil;
    }
    
    // ตรวจสอบ Content Type (BOM)
    const uint8_t *bytes = data.bytes;
    if (data.length >= 3 && bytes[0] == 0xEF && bytes[1] == 0xBB && bytes[2] == 0xBF) {
        // มี UTF-8 BOM - ตัดออก
        data = [data subdataWithRange:NSMakeRange(3, data.length - 3)];
    }
    
    NSError *jsonError;
    id result = [NSJSONSerialization JSONObjectWithData:data
                                               options:NSJSONReadingAllowFragments
                                                 error:&jsonError];
    if (jsonError) {
        // แสดง Raw Data เพื่อ Debug
        NSString *rawString = [[NSString alloc] initWithData:data encoding:NSUTF8StringEncoding];
        NSLog(@"Failed to parse JSON. Raw data: %@", rawString);
        
        if (outError) {
            NSMutableDictionary *userInfo = [NSMutableDictionary dictionary];
            userInfo[NSLocalizedDescriptionKey] = @"Failed to parse JSON response";
            userInfo[NSUnderlyingErrorKey] = jsonError;
            userInfo[@"rawResponse"] = rawString ?: @"(cannot decode)";
            
            *outError = [NSError errorWithDomain:@"JSONParserError"
                                           code:jsonError.code
                                       userInfo:userInfo];
        }
        return nil;
    }
    
    return result;
}

+ (NSData *)serializeObject:(id)object error:(NSError **)outError {
    if (!object) {
        if (outError) {
            *outError = [NSError errorWithDomain:@"JSONParserError"
                                           code:1003
                                       userInfo:@{NSLocalizedDescriptionKey: @"Object is nil"}];
        }
        return nil;
    }
    
    if (![NSJSONSerialization isValidJSONObject:object]) {
        if (outError) {
            *outError = [NSError errorWithDomain:@"JSONParserError"
                                           code:1004
                                       userInfo:@{NSLocalizedDescriptionKey: 
                                           [NSString stringWithFormat:@"Object is not valid JSON: %@", 
                                            NSStringFromClass([object class])]}];
        }
        return nil;
    }
    
    return [NSJSONSerialization dataWithJSONObject:object options:0 error:outError];
}

@end

// การใช้งาน
NSError *parseError;
id result = [SafeJSONParser parseData:responseData error:&parseError];

if (parseError) {
    NSLog(@"Parse Error: %@", parseError.localizedDescription);
    NSLog(@"Raw Response: %@", parseError.userInfo[@"rawResponse"]);
    
    // Handle ตามประเภท Error
    if ([parseError.domain isEqualToString:@"JSONParserError"]) {
        switch (parseError.code) {
            case 1001:
                NSLog(@"No data received");
                break;
            case 1002:
                NSLog(@"Empty response");
                break;
            default:
                NSLog(@"JSON format error");
                break;
        }
    }
} else {
    // ใช้งาน result
}
```

---

## 55.6 Manual Model Mapping

### การ Map JSON เป็น Model Objects

```objc
// User Model
@interface User : NSObject

@property (nonatomic, assign) NSInteger userID;
@property (nonatomic, copy) NSString *name;
@property (nonatomic, copy) NSString *email;
@property (nonatomic, copy) NSString *avatarURL;
@property (nonatomic, strong) NSDate *createdAt;
@property (nonatomic, strong) NSArray<Address *> *addresses;

+ (instancetype)userFromDictionary:(NSDictionary *)dict;
- (NSDictionary *)toDictionary;

@end

@implementation User

+ (instancetype)userFromDictionary:(NSDictionary *)dict {
    if (!dict || ![dict isKindOfClass:[NSDictionary class]]) return nil;
    
    User *user = [[User alloc] init];
    user.userID = [dict[@"id"] integerValue];
    user.name = [dict[@"name"] isKindOfClass:[NSString class]] ? dict[@"name"] : @"";
    user.email = [dict[@"email"] isKindOfClass:[NSString class]] ? dict[@"email"] : @"";
    user.avatarURL = [dict[@"avatar_url"] isKindOfClass:[NSString class]] ? dict[@"avatar_url"] : nil;
    
    // Parse Date
    NSString *dateString = dict[@"created_at"];
    if ([dateString isKindOfClass:[NSString class]]) {
        NSDateFormatter *formatter = [[NSDateFormatter alloc] init];
        formatter.dateFormat = @"yyyy-MM-dd'T'HH:mm:ssZ";
        formatter.locale = [NSLocale localeWithLocaleIdentifier:@"en_US_POSIX"];
        user.createdAt = [formatter dateFromString:dateString];
    }
    
    // Parse Nested Array
    NSArray *addressDicts = dict[@"addresses"];
    if ([addressDicts isKindOfClass:[NSArray class]]) {
        NSMutableArray *addresses = [NSMutableArray array];
        for (NSDictionary *addrDict in addressDicts) {
            Address *addr = [Address addressFromDictionary:addrDict];
            if (addr) [addresses addObject:addr];
        }
        user.addresses = [addresses copy];
    }
    
    return user;
}

- (NSDictionary *)toDictionary {
    NSMutableDictionary *dict = [NSMutableDictionary dictionary];
    dict[@"id"] = @(self.userID);
    if (self.name) dict[@"name"] = self.name;
    if (self.email) dict[@"email"] = self.email;
    if (self.avatarURL) dict[@"avatar_url"] = self.avatarURL;
    
    if (self.createdAt) {
        NSDateFormatter *formatter = [[NSDateFormatter alloc] init];
        formatter.dateFormat = @"yyyy-MM-dd'T'HH:mm:ssZ";
        formatter.locale = [NSLocale localeWithLocaleIdentifier:@"en_US_POSIX"];
        dict[@"created_at"] = [formatter stringFromDate:self.createdAt];
    }
    
    if (self.addresses) {
        NSMutableArray *addressDicts = [NSMutableArray array];
        for (Address *addr in self.addresses) {
            [addressDicts addObject:[addr toDictionary]];
        }
        dict[@"addresses"] = [addressDicts copy];
    }
    
    return [dict copy];
}

- (NSString *)description {
    return [NSString stringWithFormat:@"<User: id=%ld, name=%@, email=%@>",
            (long)self.userID, self.name, self.email];
}

@end

// Address Model
@interface Address : NSObject

@property (nonatomic, copy) NSString *street;
@property (nonatomic, copy) NSString *city;
@property (nonatomic, copy) NSString *country;
@property (nonatomic, copy) NSString *postalCode;
@property (nonatomic, assign) BOOL isPrimary;

+ (instancetype)addressFromDictionary:(NSDictionary *)dict;
- (NSDictionary *)toDictionary;

@end

@implementation Address

+ (instancetype)addressFromDictionary:(NSDictionary *)dict {
    if (!dict || ![dict isKindOfClass:[NSDictionary class]]) return nil;
    
    Address *address = [[Address alloc] init];
    address.street = dict[@"street"];
    address.city = dict[@"city"];
    address.country = dict[@"country"];
    address.postalCode = dict[@"postal_code"];
    address.isPrimary = [dict[@"is_primary"] boolValue];
    return address;
}

- (NSDictionary *)toDictionary {
    return @{
        @"street": self.street ?: @"",
        @"city": self.city ?: @"",
        @"country": self.country ?: @"",
        @"postal_code": self.postalCode ?: @"",
        @"is_primary": @(self.isPrimary)
    };
}

@end
```

---

## 55.7 Building a Model Layer

### Generic JSON-to-Model Protocol

```objc
// Protocol สำหรับ JSON Mappable
@protocol JSONMappable <NSObject>
@required
+ (instancetype)modelFromDictionary:(NSDictionary *)dictionary;
- (NSDictionary *)dictionaryRepresentation;
@optional
+ (NSArray *)modelsFromArray:(NSArray *)array;
@end

// Base Implementation
@interface JSONModel : NSObject <JSONMappable>
@end

@implementation JSONModel

+ (NSArray *)modelsFromArray:(NSArray *)array {
    if (![array isKindOfClass:[NSArray class]]) return @[];
    
    NSMutableArray *models = [NSMutableArray arrayWithCapacity:array.count];
    for (NSDictionary *dict in array) {
        id model = [self modelFromDictionary:dict];
        if (model) {
            [models addObject:model];
        }
    }
    return [models copy];
}

+ (instancetype)modelFromDictionary:(NSDictionary *)dictionary {
    // Override ใน Subclass
    return nil;
}

- (NSDictionary *)dictionaryRepresentation {
    // Override ใน Subclass
    return @{};
}

@end

// Product Model
@interface Product : JSONModel

@property (nonatomic, assign) NSInteger productID;
@property (nonatomic, copy) NSString *name;
@property (nonatomic, copy) NSString *description;
@property (nonatomic, assign) double price;
@property (nonatomic, copy) NSString *currency;
@property (nonatomic, assign) NSInteger stockQuantity;
@property (nonatomic, copy) NSArray<NSString *> *imageURLs;
@property (nonatomic, strong) ProductCategory *category;
@property (nonatomic, assign) BOOL isAvailable;

@end

@implementation Product

+ (instancetype)modelFromDictionary:(NSDictionary *)dict {
    if (![dict isKindOfClass:[NSDictionary class]]) return nil;
    
    Product *product = [[Product alloc] init];
    product.productID = [dict[@"id"] integerValue];
    product.name = [dict[@"name"] isKindOfClass:[NSString class]] ? dict[@"name"] : @"";
    product.description = dict[@"description"];
    product.price = [dict[@"price"] doubleValue];
    product.currency = dict[@"currency"] ?: @"USD";
    product.stockQuantity = [dict[@"stock"] integerValue];
    product.isAvailable = [dict[@"is_available"] boolValue];
    
    // Parse Images Array
    id images = dict[@"images"];
    if ([images isKindOfClass:[NSArray class]]) {
        product.imageURLs = [(NSArray *)images filteredArrayUsingPredicate:
                            [NSPredicate predicateWithBlock:^BOOL(id obj, NSDictionary *bindings) {
            return [obj isKindOfClass:[NSString class]];
        }]];
    }
    
    // Parse Nested Object
    NSDictionary *categoryDict = dict[@"category"];
    if ([categoryDict isKindOfClass:[NSDictionary class]]) {
        product.category = [ProductCategory modelFromDictionary:categoryDict];
    }
    
    return product;
}

- (NSDictionary *)dictionaryRepresentation {
    NSMutableDictionary *dict = [@{
        @"id": @(self.productID),
        @"name": self.name ?: @"",
        @"price": @(self.price),
        @"currency": self.currency ?: @"USD",
        @"stock": @(self.stockQuantity),
        @"is_available": @(self.isAvailable)
    } mutableCopy];
    
    if (self.description) dict[@"description"] = self.description;
    if (self.imageURLs) dict[@"images"] = self.imageURLs;
    if (self.category) dict[@"category"] = [self.category dictionaryRepresentation];
    
    return [dict copy];
}

@end
```

### Response Wrapper

```objc
// API Response Wrapper
@interface APIResponse<T> : NSObject

@property (nonatomic, assign) BOOL success;
@property (nonatomic, copy) NSString *message;
@property (nonatomic, strong) T data;
@property (nonatomic, assign) NSInteger statusCode;
@property (nonatomic, strong) NSDictionary *meta;

+ (instancetype)responseFromDictionary:(NSDictionary *)dict 
                           dataParser:(id(^)(id rawData))parser;

@end

@implementation APIResponse

+ (instancetype)responseFromDictionary:(NSDictionary *)dict 
                            dataParser:(id(^)(id rawData))parser {
    
    APIResponse *response = [[APIResponse alloc] init];
    response.success = [dict[@"success"] boolValue];
    response.message = dict[@"message"];
    response.statusCode = [dict[@"status_code"] integerValue];
    response.meta = dict[@"meta"];
    
    id rawData = dict[@"data"];
    if (rawData && parser) {
        response.data = parser(rawData);
    }
    
    return response;
}

@end

// การใช้งาน
NSData *responseData = /* ข้อมูลจาก API */;
NSError *error;
NSDictionary *json = [NSJSONSerialization JSONObjectWithData:responseData options:0 error:&error];

APIResponse *apiResponse = [APIResponse responseFromDictionary:json 
                                                   dataParser:^id(id rawData) {
    if ([rawData isKindOfClass:[NSArray class]]) {
        return [Product modelsFromArray:rawData];
    }
    return nil;
}];

if (apiResponse.success) {
    NSArray<Product *> *products = apiResponse.data;
    NSLog(@"Got %lu products", (unsigned long)products.count);
} else {
    NSLog(@"API Error: %@", apiResponse.message);
}
```

---

## 55.8 JSON กับ Custom Date Formats

```objc
@interface DateParserFactory : NSObject

+ (NSDateFormatter *)iso8601Formatter;
+ (NSDateFormatter *)shortDateFormatter;
+ (NSDateFormatter *)unixTimestampFormatter;
+ (NSDate *)dateFromString:(NSString *)string;
+ (NSDate *)dateFromTimestamp:(NSNumber *)timestamp;

@end

@implementation DateParserFactory

+ (NSDateFormatter *)iso8601Formatter {
    static NSDateFormatter *formatter = nil;
    static dispatch_once_t onceToken;
    dispatch_once(&onceToken, ^{
        formatter = [[NSDateFormatter alloc] init];
        formatter.locale = [NSLocale localeWithLocaleIdentifier:@"en_US_POSIX"];
        formatter.timeZone = [NSTimeZone timeZoneWithAbbreviation:@"UTC"];
        formatter.dateFormat = @"yyyy-MM-dd'T'HH:mm:ss.SSSZ";
    });
    return formatter;
}

+ (NSDateFormatter *)shortDateFormatter {
    static NSDateFormatter *formatter = nil;
    static dispatch_once_t onceToken;
    dispatch_once(&onceToken, ^{
        formatter = [[NSDateFormatter alloc] init];
        formatter.locale = [NSLocale localeWithLocaleIdentifier:@"en_US_POSIX"];
        formatter.dateFormat = @"yyyy-MM-dd";
    });
    return formatter;
}

+ (NSDate *)dateFromString:(NSString *)string {
    if (!string || [string isKindOfClass:[NSNull class]]) return nil;
    
    // ลอง Formats หลายแบบ
    NSArray *formatters = @[
        [self iso8601Formatter],
        [self shortDateFormatter]
    ];
    
    // ลอง ISO8601DateFormatter (iOS 10+) ก่อน
    if (@available(iOS 10.0, *)) {
        NSISO8601DateFormatter *isoFormatter = [[NSISO8601DateFormatter alloc] init];
        isoFormatter.formatOptions = NSISO8601DateFormatWithInternetDateTime | 
                                      NSISO8601DateFormatWithFractionalSeconds;
        NSDate *date = [isoFormatter dateFromString:string];
        if (date) return date;
    }
    
    for (NSDateFormatter *formatter in formatters) {
        NSDate *date = [formatter dateFromString:string];
        if (date) return date;
    }
    
    return nil;
}

+ (NSDate *)dateFromTimestamp:(NSNumber *)timestamp {
    if (!timestamp || [timestamp isKindOfClass:[NSNull class]]) return nil;
    return [NSDate dateWithTimeIntervalSince1970:[timestamp doubleValue]];
}

@end

// ใช้ใน Model Parsing
@implementation Event

+ (instancetype)modelFromDictionary:(NSDictionary *)dict {
    Event *event = [[Event alloc] init];
    
    // หลายรูปแบบ Date
    event.startDate = [DateParserFactory dateFromString:dict[@"start_date"]];
    event.endDate = [DateParserFactory dateFromString:dict[@"end_date"]];
    event.createdAt = [DateParserFactory dateFromTimestamp:dict[@"created_at_timestamp"]];
    
    return event;
}

@end
```

---

## 55.9 Handling Null Values (NSNull)

```objc
// NSNull คือ Singleton ที่แทน JSON null
// json: {"name": "John", "bio": null}
// Objective-C: @{@"name": @"John", @"bio": [NSNull null]}

// วิธีตรวจสอบ NSNull
id bioValue = dict[@"bio"];

// วิธีที่ 1: ตรวจสอบ isKindOfClass
if ([bioValue isKindOfClass:[NSNull class]]) {
    NSLog(@"bio is null");
}

// วิธีที่ 2: เปรียบเทียบกับ NSNull.null
if (bioValue == [NSNull null]) {
    NSLog(@"bio is null");
}

// วิธีที่ 3: ใช้ Category (แนะนำ)
@interface NSNull (JSONHelper)
- (BOOL)isJSON_null;
@end

@implementation NSNull (JSONHelper)
- (BOOL)isJSON_null { return YES; }
@end

@interface NSObject (JSONHelper)
- (BOOL)isJSON_null;
@end

@implementation NSObject (JSONHelper)
- (BOOL)isJSON_null { return NO; }
@end

// Safe Value Accessors
@interface NSDictionary (SafeJSON)
- (NSString *)safeStringForKey:(NSString *)key;
- (NSNumber *)safeNumberForKey:(NSString *)key;
- (NSArray *)safeArrayForKey:(NSString *)key;
- (NSDictionary *)safeDictionaryForKey:(NSString *)key;
- (BOOL)safeBoolForKey:(NSString *)key;
- (NSInteger)safeIntegerForKey:(NSString *)key;
- (double)safeDoubleForKey:(NSString *)key;
@end

@implementation NSDictionary (SafeJSON)

- (NSString *)safeStringForKey:(NSString *)key {
    id value = self[key];
    if (!value || [value isKindOfClass:[NSNull class]]) return nil;
    if ([value isKindOfClass:[NSString class]]) return value;
    if ([value isKindOfClass:[NSNumber class]]) return [value stringValue];
    return nil;
}

- (NSNumber *)safeNumberForKey:(NSString *)key {
    id value = self[key];
    if (!value || [value isKindOfClass:[NSNull class]]) return nil;
    if ([value isKindOfClass:[NSNumber class]]) return value;
    if ([value isKindOfClass:[NSString class]]) {
        NSNumberFormatter *formatter = [[NSNumberFormatter alloc] init];
        return [formatter numberFromString:value];
    }
    return nil;
}

- (NSArray *)safeArrayForKey:(NSString *)key {
    id value = self[key];
    if ([value isKindOfClass:[NSArray class]]) return value;
    return @[];
}

- (NSDictionary *)safeDictionaryForKey:(NSString *)key {
    id value = self[key];
    if ([value isKindOfClass:[NSDictionary class]]) return value;
    return @{};
}

- (BOOL)safeBoolForKey:(NSString *)key {
    id value = self[key];
    if (!value || [value isKindOfClass:[NSNull class]]) return NO;
    if ([value isKindOfClass:[NSNumber class]]) return [value boolValue];
    if ([value isKindOfClass:[NSString class]]) {
        NSString *str = [(NSString *)value lowercaseString];
        return [str isEqualToString:@"true"] || [str isEqualToString:@"yes"] || 
               [str isEqualToString:@"1"];
    }
    return NO;
}

- (NSInteger)safeIntegerForKey:(NSString *)key {
    return [[self safeNumberForKey:key] integerValue];
}

- (double)safeDoubleForKey:(NSString *)key {
    return [[self safeNumberForKey:key] doubleValue];
}

@end

// การใช้งาน
NSDictionary *userData = response[@"user"];
NSString *name = [userData safeStringForKey:@"name"];
NSInteger age = [userData safeIntegerForKey:@"age"];
BOOL isActive = [userData safeBoolForKey:@"is_active"];
NSArray *friends = [userData safeArrayForKey:@"friends"];
NSLog(@"User: %@, Age: %ld, Active: %@", name, (long)age, isActive ? @"YES" : @"NO");
```

---

## 55.10 Large JSON Performance

### Performance Tips สำหรับ JSON ขนาดใหญ่

```objc
@interface LargeJSONProcessor : NSObject

// Process ใน Background Thread
- (void)processLargeJSON:(NSData *)jsonData 
              completion:(void(^)(NSArray *results, NSError *error))completion;

// Stream Processing สำหรับ JSON ใหญ่มาก
- (void)streamProcessJSONArray:(NSData *)jsonData 
                   batchSize:(NSInteger)batchSize
                 batchHandler:(void(^)(NSArray *batch))batchHandler
                   completion:(void(^)(NSError *error))completion;

@end

@implementation LargeJSONProcessor

- (void)processLargeJSON:(NSData *)jsonData 
             completion:(void(^)(NSArray *results, NSError *error))completion {
    
    // ทำใน Background
    dispatch_async(dispatch_get_global_queue(DISPATCH_QUEUE_PRIORITY_DEFAULT, 0), ^{
        NSDate *startTime = [NSDate date];
        
        NSError *error;
        id result = [NSJSONSerialization JSONObjectWithData:jsonData
                                                   options:NSJSONReadingAllowFragments
                                                     error:&error];
        
        NSTimeInterval parseTime = [[NSDate date] timeIntervalSinceDate:startTime];
        NSLog(@"JSON parsing took: %.3f seconds for %lu bytes",
              parseTime, (unsigned long)jsonData.length);
        
        if (error || ![result isKindOfClass:[NSArray class]]) {
            dispatch_async(dispatch_get_main_queue(), ^{
                completion(nil, error);
            });
            return;
        }
        
        NSArray *rawArray = (NSArray *)result;
        NSLog(@"Processing %lu items...", (unsigned long)rawArray.count);
        
        // Process ทีละ Batch เพื่อไม่ block Thread นานเกินไป
        NSMutableArray *processedItems = [NSMutableArray arrayWithCapacity:rawArray.count];
        NSInteger batchSize = 100;
        
        for (NSInteger i = 0; i < (NSInteger)rawArray.count; i += batchSize) {
            NSInteger end = MIN(i + batchSize, (NSInteger)rawArray.count);
            NSArray *batch = [rawArray subarrayWithRange:NSMakeRange(i, end - i)];
            
            for (NSDictionary *dict in batch) {
                // Map to Model
                id model = /* ทำ mapping */dict;
                [processedItems addObject:model];
            }
        }
        
        NSTimeInterval totalTime = [[NSDate date] timeIntervalSinceDate:startTime];
        NSLog(@"Total processing time: %.3f seconds", totalTime);
        
        dispatch_async(dispatch_get_main_queue(), ^{
            completion([processedItems copy], nil);
        });
    });
}

// Lazy Loading สำหรับ JSON ขนาดใหญ่
- (void)streamProcessJSONArray:(NSData *)jsonData 
                    batchSize:(NSInteger)batchSize
                  batchHandler:(void(^)(NSArray *batch))batchHandler
                    completion:(void(^)(NSError *error))completion {
    
    dispatch_async(dispatch_get_global_queue(DISPATCH_QUEUE_PRIORITY_DEFAULT, 0), ^{
        NSError *error;
        NSArray *fullArray = [NSJSONSerialization JSONObjectWithData:jsonData
                                                            options:NSJSONReadingMutableContainers
                                                              error:&error];
        if (error) {
            dispatch_async(dispatch_get_main_queue(), ^{ completion(error); });
            return;
        }
        
        // ส่ง Batch ทีละชุด
        for (NSInteger i = 0; i < (NSInteger)fullArray.count; i += batchSize) {
            NSInteger end = MIN(i + batchSize, (NSInteger)fullArray.count);
            NSArray *batch = [fullArray subarrayWithRange:NSMakeRange(i, end - i)];
            
            dispatch_sync(dispatch_get_main_queue(), ^{
                batchHandler(batch);
            });
        }
        
        dispatch_async(dispatch_get_main_queue(), ^{ completion(nil); });
    });
}

@end

// การใช้งาน
LargeJSONProcessor *processor = [[LargeJSONProcessor alloc] init];

[processor streamProcessJSONArray:hugeJSONData 
                        batchSize:50
                     batchHandler:^(NSArray *batch) {
    // อัปเดต UI ทีละ 50 รายการ
    NSLog(@"Processing batch of %lu items", (unsigned long)batch.count);
}
                       completion:^(NSError *error) {
    if (error) {
        NSLog(@"Error: %@", error.localizedDescription);
    } else {
        NSLog(@"All done!");
    }
}];
```

### Memory-Efficient JSON Processing

```objc
// วัด Memory ใช้ก่อน/หลัง
- (void)measureMemoryForJSONParsing:(NSData *)data {
    // ก่อน Parse
    mach_task_basic_info_data_t info;
    mach_msg_type_number_t size = MACH_TASK_BASIC_INFO_COUNT;
    kern_return_t kerr = task_info(mach_task_self(), MACH_TASK_BASIC_INFO, 
                                    (task_info_t)&info, &size);
    uint64_t beforeBytes = (kerr == KERN_SUCCESS) ? info.resident_size : 0;
    
    // Parse
    NSError *error;
    id result = [NSJSONSerialization JSONObjectWithData:data options:0 error:&error];
    
    // หลัง Parse
    kerr = task_info(mach_task_self(), MACH_TASK_BASIC_INFO, (task_info_t)&info, &size);
    uint64_t afterBytes = (kerr == KERN_SUCCESS) ? info.resident_size : 0;
    
    NSLog(@"Memory used for parsing: %llu bytes (%.2f MB)",
          afterBytes - beforeBytes,
          (double)(afterBytes - beforeBytes) / 1024.0 / 1024.0);
}
```

---

## 55.11 ตัวอย่างสมบูรณ์: GitHub API

### GitHub API Client

```objc
// GitHubModels.h
@interface GitHubUser : JSONModel

@property (nonatomic, assign) NSInteger userID;
@property (nonatomic, copy) NSString *login;
@property (nonatomic, copy) NSString *name;
@property (nonatomic, copy) NSString *email;
@property (nonatomic, copy) NSString *bio;
@property (nonatomic, copy) NSString *avatarURL;
@property (nonatomic, copy) NSString *location;
@property (nonatomic, copy) NSString *company;
@property (nonatomic, assign) NSInteger publicRepos;
@property (nonatomic, assign) NSInteger followers;
@property (nonatomic, assign) NSInteger following;
@property (nonatomic, strong) NSDate *createdAt;

@end

@interface GitHubRepo : JSONModel

@property (nonatomic, assign) NSInteger repoID;
@property (nonatomic, copy) NSString *name;
@property (nonatomic, copy) NSString *fullName;
@property (nonatomic, copy) NSString *repoDescription;
@property (nonatomic, copy) NSString *htmlURL;
@property (nonatomic, copy) NSString *cloneURL;
@property (nonatomic, assign) NSInteger stargazersCount;
@property (nonatomic, assign) NSInteger forksCount;
@property (nonatomic, assign) NSInteger openIssuesCount;
@property (nonatomic, copy) NSString *language;
@property (nonatomic, strong) NSDate *updatedAt;
@property (nonatomic, assign) BOOL isPrivate;
@property (nonatomic, assign) BOOL isFork;
@property (nonatomic, strong) GitHubUser *owner;

@end

// GitHubModels.m
@implementation GitHubUser

+ (instancetype)modelFromDictionary:(NSDictionary *)dict {
    if (![dict isKindOfClass:[NSDictionary class]]) return nil;
    
    GitHubUser *user = [[GitHubUser alloc] init];
    user.userID = [dict safeIntegerForKey:@"id"];
    user.login = [dict safeStringForKey:@"login"];
    user.name = [dict safeStringForKey:@"name"];
    user.email = [dict safeStringForKey:@"email"];
    user.bio = [dict safeStringForKey:@"bio"];
    user.avatarURL = [dict safeStringForKey:@"avatar_url"];
    user.location = [dict safeStringForKey:@"location"];
    user.company = [dict safeStringForKey:@"company"];
    user.publicRepos = [dict safeIntegerForKey:@"public_repos"];
    user.followers = [dict safeIntegerForKey:@"followers"];
    user.following = [dict safeIntegerForKey:@"following"];
    user.createdAt = [DateParserFactory dateFromString:[dict safeStringForKey:@"created_at"]];
    return user;
}

- (NSDictionary *)dictionaryRepresentation {
    NSMutableDictionary *dict = [NSMutableDictionary dictionary];
    dict[@"id"] = @(self.userID);
    if (self.login) dict[@"login"] = self.login;
    if (self.name) dict[@"name"] = self.name;
    if (self.email) dict[@"email"] = self.email;
    if (self.bio) dict[@"bio"] = self.bio;
    if (self.avatarURL) dict[@"avatar_url"] = self.avatarURL;
    return [dict copy];
}

@end

@implementation GitHubRepo

+ (instancetype)modelFromDictionary:(NSDictionary *)dict {
    if (![dict isKindOfClass:[NSDictionary class]]) return nil;
    
    GitHubRepo *repo = [[GitHubRepo alloc] init];
    repo.repoID = [dict safeIntegerForKey:@"id"];
    repo.name = [dict safeStringForKey:@"name"];
    repo.fullName = [dict safeStringForKey:@"full_name"];
    repo.repoDescription = [dict safeStringForKey:@"description"];
    repo.htmlURL = [dict safeStringForKey:@"html_url"];
    repo.cloneURL = [dict safeStringForKey:@"clone_url"];
    repo.stargazersCount = [dict safeIntegerForKey:@"stargazers_count"];
    repo.forksCount = [dict safeIntegerForKey:@"forks_count"];
    repo.openIssuesCount = [dict safeIntegerForKey:@"open_issues_count"];
    repo.language = [dict safeStringForKey:@"language"];
    repo.updatedAt = [DateParserFactory dateFromString:[dict safeStringForKey:@"updated_at"]];
    repo.isPrivate = [dict safeBoolForKey:@"private"];
    repo.isFork = [dict safeBoolForKey:@"fork"];
    
    NSDictionary *ownerDict = [dict safeDictionaryForKey:@"owner"];
    if (ownerDict.count > 0) {
        repo.owner = [GitHubUser modelFromDictionary:ownerDict];
    }
    
    return repo;
}

@end

// GitHubAPIClient.h
@interface GitHubAPIClient : NSObject

+ (instancetype)shared;

- (void)fetchUserWithLogin:(NSString *)login
                completion:(void(^)(GitHubUser *user, NSError *error))completion;

- (void)fetchRepositoriesForUser:(NSString *)login
                            page:(NSInteger)page
                            perPage:(NSInteger)perPage
                       completion:(void(^)(NSArray<GitHubRepo *> *repos, 
                                          NSError *error))completion;

- (void)searchRepositories:(NSString *)query
                  language:(NSString *)language
               completion:(void(^)(NSArray<GitHubRepo *> *repos, 
                                   NSInteger totalCount,
                                   NSError *error))completion;

@end

// GitHubAPIClient.m
@interface GitHubAPIClient ()
@property (nonatomic, strong) NSURLSession *session;
@end

static NSString *const GitHubBaseURL = @"https://api.github.com";

@implementation GitHubAPIClient

+ (instancetype)shared {
    static GitHubAPIClient *instance = nil;
    static dispatch_once_t onceToken;
    dispatch_once(&onceToken, ^{ instance = [[GitHubAPIClient alloc] init]; });
    return instance;
}

- (instancetype)init {
    self = [super init];
    if (self) {
        NSURLSessionConfiguration *config = [NSURLSessionConfiguration defaultSessionConfiguration];
        config.HTTPAdditionalHeaders = @{
            @"Accept": @"application/vnd.github.v3+json",
            @"User-Agent": @"MyiOSApp/1.0"
        };
        config.timeoutIntervalForRequest = 30.0;
        self.session = [NSURLSession sessionWithConfiguration:config];
    }
    return self;
}

- (void)fetchUserWithLogin:(NSString *)login
                completion:(void(^)(GitHubUser *user, NSError *error))completion {
    
    NSString *urlString = [NSString stringWithFormat:@"%@/users/%@", GitHubBaseURL, login];
    NSURL *url = [NSURL URLWithString:urlString];
    
    NSURLSessionDataTask *task = [self.session dataTaskWithURL:url
                                             completionHandler:^(NSData *data, 
                                                                 NSURLResponse *response, 
                                                                 NSError *error) {
        if (error) {
            dispatch_async(dispatch_get_main_queue(), ^{ completion(nil, error); });
            return;
        }
        
        NSHTTPURLResponse *httpResponse = (NSHTTPURLResponse *)response;
        if (httpResponse.statusCode == 404) {
            NSError *notFoundError = [NSError errorWithDomain:@"GitHubError"
                                                        code:404
                                                    userInfo:@{NSLocalizedDescriptionKey: 
                                                        [NSString stringWithFormat:@"User '%@' not found", login]}];
            dispatch_async(dispatch_get_main_queue(), ^{ completion(nil, notFoundError); });
            return;
        }
        
        NSError *jsonError;
        NSDictionary *dict = [NSJSONSerialization JSONObjectWithData:data options:0 error:&jsonError];
        
        dispatch_async(dispatch_get_main_queue(), ^{
            if (jsonError) {
                completion(nil, jsonError);
            } else {
                GitHubUser *user = [GitHubUser modelFromDictionary:dict];
                completion(user, nil);
            }
        });
    }];
    
    [task resume];
}

- (void)fetchRepositoriesForUser:(NSString *)login
                            page:(NSInteger)page
                         perPage:(NSInteger)perPage
                      completion:(void(^)(NSArray<GitHubRepo *> *repos, NSError *error))completion {
    
    NSString *urlString = [NSString stringWithFormat:
                           @"%@/users/%@/repos?page=%ld&per_page=%ld&sort=updated",
                           GitHubBaseURL, login, (long)page, (long)perPage];
    NSURL *url = [NSURL URLWithString:urlString];
    
    NSURLSessionDataTask *task = [self.session dataTaskWithURL:url
                                             completionHandler:^(NSData *data,
                                                                 NSURLResponse *response,
                                                                 NSError *error) {
        if (error) {
            dispatch_async(dispatch_get_main_queue(), ^{ completion(nil, error); });
            return;
        }
        
        NSError *jsonError;
        NSArray *rawArray = [NSJSONSerialization JSONObjectWithData:data options:0 error:&jsonError];
        
        if (jsonError || ![rawArray isKindOfClass:[NSArray class]]) {
            dispatch_async(dispatch_get_main_queue(), ^{ completion(@[], jsonError); });
            return;
        }
        
        // Parse บน Background Thread
        NSArray<GitHubRepo *> *repos = [GitHubRepo modelsFromArray:rawArray];
        
        dispatch_async(dispatch_get_main_queue(), ^{
            completion(repos, nil);
        });
    }];
    
    [task resume];
}

- (void)searchRepositories:(NSString *)query
                  language:(NSString *)language
               completion:(void(^)(NSArray<GitHubRepo *> *repos, 
                                   NSInteger totalCount,
                                   NSError *error))completion {
    
    // URL Encode
    NSString *encodedQuery = [query stringByAddingPercentEncodingWithAllowedCharacters:
                              [NSCharacterSet URLQueryAllowedCharacterSet]];
    NSString *fullQuery = language ? 
        [NSString stringWithFormat:@"%@+language:%@", encodedQuery, language] : 
        encodedQuery;
    
    NSString *urlString = [NSString stringWithFormat:
                           @"%@/search/repositories?q=%@&sort=stars&order=desc",
                           GitHubBaseURL, fullQuery];
    NSURL *url = [NSURL URLWithString:urlString];
    
    NSURLSessionDataTask *task = [self.session dataTaskWithURL:url
                                             completionHandler:^(NSData *data,
                                                                 NSURLResponse *response,
                                                                 NSError *error) {
        if (error) {
            dispatch_async(dispatch_get_main_queue(), ^{ completion(nil, 0, error); });
            return;
        }
        
        NSError *jsonError;
        NSDictionary *result = [NSJSONSerialization JSONObjectWithData:data options:0 error:&jsonError];
        
        if (jsonError || ![result isKindOfClass:[NSDictionary class]]) {
            dispatch_async(dispatch_get_main_queue(), ^{ completion(nil, 0, jsonError); });
            return;
        }
        
        NSInteger totalCount = [result safeIntegerForKey:@"total_count"];
        NSArray *items = [result safeArrayForKey:@"items"];
        NSArray<GitHubRepo *> *repos = [GitHubRepo modelsFromArray:items];
        
        dispatch_async(dispatch_get_main_queue(), ^{
            completion(repos, totalCount, nil);
        });
    }];
    
    [task resume];
}

@end

// การใช้งาน
[[GitHubAPIClient shared] fetchUserWithLogin:@"apple" 
                                  completion:^(GitHubUser *user, NSError *error) {
    if (user) {
        NSLog(@"GitHub User: %@", user.login);
        NSLog(@"Name: %@", user.name);
        NSLog(@"Followers: %ld", (long)user.followers);
        NSLog(@"Public Repos: %ld", (long)user.publicRepos);
    }
}];

[[GitHubAPIClient shared] searchRepositories:@"iOS" 
                                    language:@"Objective-C"
                                  completion:^(NSArray<GitHubRepo *> *repos, 
                                               NSInteger totalCount, 
                                               NSError *error) {
    NSLog(@"Found %ld repos", (long)totalCount);
    for (GitHubRepo *repo in repos) {
        NSLog(@"⭐ %ld - %@", (long)repo.stargazersCount, repo.fullName);
    }
}];
```

---

## 55.12 แบบฝึกหัด

### แบบฝึกหัดที่ 1: JSON Parser สมบูรณ์

สร้าง Class `JSONMapper` ที่รองรับ:
```objc
@interface JSONMapper : NSObject
// Map JSON Dict to Model ด้วย Block
+ (id)mapDictionary:(NSDictionary *)dict toClass:(Class)modelClass;
// Map JSON Array to Models
+ (NSArray *)mapArray:(NSArray *)array toClass:(Class)modelClass;
// Validate JSON Structure ตาม Schema
+ (BOOL)validateJSON:(id)json againstSchema:(NSDictionary *)schema error:(NSError **)error;
@end
```

### แบบฝึกหัดที่ 2: JSON Cache

สร้าง `JSONCache` ที่:
- บันทึก JSON Response ลง Disk
- Load จาก Cache เมื่อ Offline
- มี TTL (Time-to-Live)
- Handle Cache Expiry

### แบบฝึกหัดที่ 3: JSON Diff

สร้าง `JSONDiff` ที่เปรียบเทียบ JSON สอง ชุดและแสดงความแตกต่าง:
- Added keys
- Removed keys
- Changed values

### แบบฝึกหัดที่ 4: GitHub Explorer

สร้าง App ง่าย ๆ ที่:
1. ค้นหา GitHub User
2. แสดงข้อมูล User และ Repositories
3. Cache ข้อมูล
4. รองรับ Offline Mode

---

## สรุปบทที่ 55

ในบทนี้เราได้เรียนรู้:

1. **NSJSONSerialization** - API หลักสำหรับ JSON Parsing ใน Objective-C
2. **JSON to NSDictionary/NSArray** - การแปลงข้อมูลจาก JSON
3. **NSDictionary/NSArray to JSON** - การแปลงข้อมูลเป็น JSON
4. **Nested JSON** - การจัดการ JSON ที่ซับซ้อน
5. **Error Handling** - การจัดการ Error ระหว่าง Parse
6. **Manual Mapping** - การสร้าง Model จาก JSON
7. **NSNull Handling** - การจัดการค่า Null
8. **Large JSON Performance** - การเพิ่มประสิทธิภาพ
9. **GitHub API Example** - ตัวอย่างการใช้งานจริง

---

*ต่อไป: ตอนที่ 56 - REST API Integration*
