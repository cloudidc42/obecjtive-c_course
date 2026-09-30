# ตอนที่ 24 - NSDictionary และ NSMutableDictionary

## บทนำ

`NSDictionary` เป็น key-value pair collection ที่ unordered และ immutable ส่วน `NSMutableDictionary` เป็น subclass ที่ allow modification บทนี้จะครอบคลุมการใช้งาน dictionaries อย่างครบถ้วน ตั้งแต่พื้นฐานไปจนถึง advanced patterns

---

## 24.1 การสร้าง Dictionaries

### 24.1.1 Literal Syntax @{}

```objc
#import <Foundation/Foundation.h>

int main(int argc, const char * argv[]) {
    @autoreleasepool {
        // Empty dictionary
        NSDictionary *empty = @{};
        
        // Simple key-value pairs
        NSDictionary *person = @{
            @"name": @"สมชาย",
            @"age": @30,
            @"city": @"กรุงเทพฯ"
        };
        
        // Mixed value types
        NSDictionary *mixed = @{
            @"string": @"hello",
            @"number": @42,
            @"float": @3.14,
            @"bool": @YES,
            @"array": @[@1, @2, @3],
            @"date": [NSDate date]
        };
        
        // Nested dictionaries
        NSDictionary *nested = @{
            @"user": @{
                @"name": @"Alice",
                @"address": @{
                    @"street": @"123 Main St",
                    @"city": @"Bangkok",
                    @"zip": @"10100"
                }
            }
        };
        
        NSLog(@"Person: %@", person);
        NSLog(@"Nested city: %@", nested[@"user"][@"address"][@"city"]);
        
        // หมายเหตุ: keys ต้อง implement NSCopying
        // values ต้องเป็น non-nil objects
        // @{key: nil} จะ crash!
    }
    return 0;
}
```

### 24.1.2 dictionaryWithObjectsAndKeys:

```objc
// Classic syntax - values ก่อน keys
NSDictionary *d1 = [NSDictionary dictionaryWithObjectsAndKeys:
                    @"สมชาย", @"name",
                    @30, @"age",
                    @"กรุงเทพฯ", @"city",
                    nil]; // ต้องจบด้วย nil!

// dictionaryWithObject:forKey:
NSDictionary *d2 = [NSDictionary dictionaryWithObject:@"value" forKey:@"key"];

// dictionaryWithDictionary: - copy
NSDictionary *d3 = [NSDictionary dictionaryWithDictionary:d1];

// dictionaryWithObjects:forKeys: - parallel arrays
NSArray *keys = @[@"a", @"b", @"c"];
NSArray *values = @[@1, @2, @3];
NSDictionary *d4 = [NSDictionary dictionaryWithObjects:values forKeys:keys];
NSLog(@"d4: %@", d4);

// dictionaryWithObjects:forKeys:count: - C arrays
id objs[] = {@"val1", @"val2"};
id ks[] = {@"key1", @"key2"};
NSDictionary *d5 = [NSDictionary dictionaryWithObjects:objs forKeys:ks count:2];

// อ่านจาก plist
NSDictionary *fromPlist = [NSDictionary dictionaryWithContentsOfFile:
                           @"/path/to/config.plist"];

// alloc-init
NSDictionary *d6 = [[NSDictionary alloc] initWithObjectsAndKeys:
                    @"value1", @"key1",
                    @"value2", @"key2",
                    nil];
```

### 24.1.3 NSMutableDictionary Creation

```objc
// Empty mutable
NSMutableDictionary *m1 = [NSMutableDictionary dictionary];

// With capacity
NSMutableDictionary *m2 = [NSMutableDictionary dictionaryWithCapacity:50];

// From immutable
NSDictionary *immutable = @{@"a": @1, @"b": @2};
NSMutableDictionary *m3 = [immutable mutableCopy];
NSMutableDictionary *m4 = [NSMutableDictionary dictionaryWithDictionary:immutable];

// alloc-init
NSMutableDictionary *m5 = [[NSMutableDictionary alloc] init];
NSMutableDictionary *m6 = [[NSMutableDictionary alloc] initWithCapacity:10];
```

---

## 24.2 การเข้าถึง Values

### objectForKey: vs valueForKey:

```objc
#import <Foundation/Foundation.h>

int main(int argc, const char * argv[]) {
    @autoreleasepool {
        NSDictionary *dict = @{
            @"name": @"สมชาย",
            @"age": @30,
            @"score": @95.5,
            @"active": @YES
        };
        
        // Subscript syntax (modern - แนะนำ)
        NSString *name = dict[@"name"];
        NSLog(@"Name (subscript): %@", name);
        
        // objectForKey: (classic)
        NSNumber *age = [dict objectForKey:@"age"];
        NSLog(@"Age: %@", age);
        
        // objectForKeyedSubscript: (same as subscript)
        id score = [dict objectForKeyedSubscript:@"score"];
        NSLog(@"Score: %@", score);
        
        // ถ้า key ไม่มี จะได้ nil (ไม่ crash)
        id missing = dict[@"email"];
        NSLog(@"Missing: %@", missing); // (null)
        
        // objectForKey: กับ nil key
        // id badKey = nil;
        // dict[badKey]; // crash!
        // [dict objectForKey:nil]; // ไม่ crash แต่ไม่แนะนำ
        
        // valueForKey: - KVC
        // ต่างจาก objectForKey: ตรงที่:
        // 1. เรียกใช้ accessor method ถ้ามี
        // 2. support key path (a.b.c)
        id nameKVC = [dict valueForKey:@"name"];
        NSLog(@"Name (KVC): %@", nameKVC);
        
        // valueForKey: กับ @ operators
        NSArray *arr = @[@{@"val": @10}, @{@"val": @20}, @{@"val": @30}];
        NSNumber *sum = [arr valueForKeyPath:@"@sum.val"];
        NSLog(@"Sum: %@", sum);
        
        // valueForUndefinedKey: จะ throw exception สำหรับ KVC
        // ในขณะที่ objectForKey: จะ return nil
        
        // Safe value access
        NSString *safeStr = [dict[@"name"] description] ?: @"default";
        
        // ตรวจสอบก่อน use
        NSNumber *ageNum = dict[@"age"];
        if (ageNum) {
            NSLog(@"Age: %ld", (long)[ageNum integerValue]);
        }
    }
    return 0;
}
```

---

## 24.3 All Keys และ All Values

```objc
#import <Foundation/Foundation.h>

int main(int argc, const char * argv[]) {
    @autoreleasepool {
        NSDictionary *config = @{
            @"host": @"localhost",
            @"port": @5432,
            @"database": @"mydb",
            @"user": @"admin",
            @"timeout": @30
        };
        
        // allKeys
        NSArray *keys = [config allKeys];
        NSLog(@"All keys: %@", keys);
        // ลำดับไม่แน่นอน!
        
        // allValues
        NSArray *values = [config allValues];
        NSLog(@"All values: %@", values);
        
        // allKeysForObject: - หา keys ทั้งหมดที่มี value นี้
        NSDictionary *scores = @{@"Alice": @90, @"Bob": @85, @"Charlie": @90};
        NSArray *keysFor90 = [scores allKeysForObject:@90];
        NSLog(@"Keys with value 90: %@", keysFor90); // [Alice, Charlie]
        
        // count
        NSLog(@"Count: %lu", (unsigned long)config.count);
        
        // Sorted keys
        NSArray *sortedKeys = [[config allKeys]
                               sortedArrayUsingSelector:@selector(compare:)];
        NSLog(@"Sorted keys: %@", sortedKeys);
        
        // keysOfEntriesPassingTest:
        NSIndexSet *portKeys = [config keysOfEntriesPassingTest:
            ^BOOL(id key, id obj, BOOL *stop) {
                return [obj isKindOfClass:[NSNumber class]];
            }];
        
        NSMutableArray *numericKeys = [NSMutableArray array];
        [config enumerateKeysAndObjectsAtIndexes:portKeys
                                         options:0
                                      usingBlock:^(id key, id obj, BOOL *stop) {
            [numericKeys addObject:key];
        }];
        NSLog(@"Numeric keys: %@", numericKeys);
    }
    return 0;
}
```

---

## 24.4 Enumeration

### 24.4.1 for-in Enumeration

```objc
NSDictionary *dict = @{@"a": @1, @"b": @2, @"c": @3};

// for-in บน dictionary วน keys
for (NSString *key in dict) {
    NSLog(@"%@: %@", key, dict[key]);
}

// reverse enumeration ไม่มีความหมายใน dictionary (unordered)
// แต่ทำได้ถ้าต้องการ
for (NSString *key in [[dict allKeys] reverseObjectEnumerator]) {
    NSLog(@"%@: %@", key, dict[key]);
}

// Enumerate ใน sorted order
for (NSString *key in [[dict allKeys] sortedArrayUsingSelector:@selector(compare:)]) {
    NSLog(@"%@: %@", key, dict[key]);
}
```

### 24.4.2 enumerateKeysAndObjectsUsingBlock:

```objc
NSDictionary *employees = @{
    @"E001": @{@"name": @"Alice", @"dept": @"Engineering"},
    @"E002": @{@"name": @"Bob", @"dept": @"Marketing"},
    @"E003": @{@"name": @"Charlie", @"dept": @"Engineering"},
    @"E004": @{@"name": @"Dave", @"dept": @"HR"}
};

// Basic enumeration
[employees enumerateKeysAndObjectsUsingBlock:
    ^(NSString *empId, NSDictionary *empData, BOOL *stop) {
        NSLog(@"%@: %@ (%@)", empId, empData[@"name"], empData[@"dept"]);
    }];

// หยุดกลางทาง
[employees enumerateKeysAndObjectsUsingBlock:
    ^(NSString *empId, NSDictionary *empData, BOOL *stop) {
        if ([empData[@"dept"] isEqualToString:@"HR"]) {
            NSLog(@"Found HR: %@", empData[@"name"]);
            *stop = YES;
        }
    }];

// Concurrent enumeration
[employees enumerateKeysAndObjectsWithOptions:NSEnumerationConcurrent
                                   usingBlock:
    ^(NSString *key, id obj, BOOL *stop) {
        NSLog(@"[concurrent] %@: %@", key, obj);
    }];
```

---

## 24.5 Checking for Keys

```objc
#import <Foundation/Foundation.h>

int main(int argc, const char * argv[]) {
    @autoreleasepool {
        NSDictionary *user = @{
            @"name": @"สมชาย",
            @"email": @"somchai@example.com",
            @"phone": @"081-234-5678"
        };
        
        // ตรวจสอบด้วย objectForKey: (returns nil ถ้าไม่มี)
        if ([user objectForKey:@"email"]) {
            NSLog(@"Has email: %@", user[@"email"]);
        }
        
        // ตรวจสอบด้วย subscript
        if (user[@"phone"] != nil) {
            NSLog(@"Has phone: %@", user[@"phone"]);
        }
        
        // ไม่มี containsKey: โดยตรง
        // ใช้ objectForKey: != nil แทน
        BOOL hasKey = user[@"address"] != nil;
        NSLog(@"Has address: %@", hasKey ? @"YES" : @"NO");
        
        // ตรวจสอบ multiple keys
        NSArray *requiredKeys = @[@"name", @"email"];
        BOOL hasAllRequired = YES;
        
        for (NSString *key in requiredKeys) {
            if (!user[key]) {
                hasAllRequired = NO;
                NSLog(@"Missing required key: %@", key);
                break;
            }
        }
        NSLog(@"Has all required: %@", hasAllRequired ? @"YES" : @"NO");
        
        // ตรวจสอบด้วย NSSet
        NSSet *userKeys = [NSSet setWithArray:[user allKeys]];
        NSSet *requiredKeySet = [NSSet setWithArray:requiredKeys];
        BOOL hasAll = [requiredKeySet isSubsetOfSet:userKeys];
        NSLog(@"Has all (NSSet): %@", hasAll ? @"YES" : @"NO");
        
        // Default value pattern
        NSString *city = user[@"city"] ?: @"Unknown City";
        NSLog(@"City: %@", city); // Unknown City
        
        NSInteger age = [user[@"age"] integerValue]; // 0 ถ้าไม่มี key
        NSLog(@"Age (or 0): %ld", (long)age);
    }
    return 0;
}
```

---

## 24.6 NSMutableDictionary Operations

```objc
#import <Foundation/Foundation.h>

int main(int argc, const char * argv[]) {
    @autoreleasepool {
        NSMutableDictionary *dict = [NSMutableDictionary dictionary];
        
        // setObject:forKey: - เพิ่มหรือแก้ไข
        [dict setObject:@"Alice" forKey:@"name"];
        [dict setObject:@30 forKey:@"age"];
        [dict setObject:@"Engineering" forKey:@"dept"];
        NSLog(@"After set: %@", dict);
        
        // Subscript assignment (modern - แนะนำ)
        dict[@"email"] = @"alice@example.com";
        dict[@"age"] = @31; // แก้ไข value
        NSLog(@"After subscript: %@", dict);
        
        // removeObjectForKey:
        [dict removeObjectForKey:@"dept"];
        NSLog(@"After remove: %@", dict);
        
        // removeObjectsForKeys: - ลบหลาย keys
        [dict removeObjectsForKeys:@[@"age", @"email"]];
        NSLog(@"After multi-remove: %@", dict);
        
        // removeAllObjects
        NSMutableDictionary *dict2 = [@{@"a": @1, @"b": @2} mutableCopy];
        [dict2 removeAllObjects];
        NSLog(@"After removeAll: %@, count: %lu",
              dict2, (unsigned long)dict2.count);
        
        // setValuesForKeysWithDictionary: - merge/update
        NSMutableDictionary *base = [@{@"a": @1, @"b": @2, @"c": @3} mutableCopy];
        [base setValuesForKeysWithDictionary:@{@"b": @20, @"d": @4}];
        NSLog(@"After merge: %@", base); // {a:1, b:20, c:3, d:4}
        
        // addEntriesFromDictionary: - เพิ่ม entries จาก dict อื่น
        NSMutableDictionary *dict3 = [@{@"x": @10} mutableCopy];
        [dict3 addEntriesFromDictionary:@{@"y": @20, @"z": @30}];
        NSLog(@"After addEntries: %@", dict3);
        
        // Conditional set (only if key doesn't exist)
        NSMutableDictionary *settings = [NSMutableDictionary dictionary];
        void (^setDefault)(NSString *, id) = ^(NSString *key, id value) {
            if (!settings[key]) {
                settings[key] = value;
            }
        };
        
        setDefault(@"theme", @"light");
        setDefault(@"language", @"th");
        settings[@"theme"] = @"dark"; // override
        setDefault(@"theme", @"light"); // won't override now
        NSLog(@"Settings: %@", settings); // theme=dark, language=th
    }
    return 0;
}
```

---

## 24.7 Sorting Dictionary by Keys/Values

```objc
#import <Foundation/Foundation.h>

int main(int argc, const char * argv[]) {
    @autoreleasepool {
        NSDictionary *scores = @{
            @"Charlie": @85,
            @"Alice": @95,
            @"Dave": @70,
            @"Bob": @90,
            @"Eve": @88
        };
        
        // Sort by keys
        NSArray *sortedKeys = [[scores allKeys]
                               sortedArrayUsingSelector:@selector(compare:)];
        NSLog(@"Sorted by key:");
        for (NSString *key in sortedKeys) {
            NSLog(@"  %@: %@", key, scores[key]);
        }
        
        // Sort by values (ascending)
        NSArray *sortedByValue = [[scores allKeys]
            sortedArrayUsingComparator:^NSComparisonResult(NSString *a, NSString *b) {
                return [scores[a] compare:scores[b]];
            }];
        
        NSLog(@"\nSorted by value (ascending):");
        for (NSString *key in sortedByValue) {
            NSLog(@"  %@: %@", key, scores[key]);
        }
        
        // Sort by values (descending)
        NSArray *sortedByValueDesc = [[scores allKeys]
            sortedArrayUsingComparator:^NSComparisonResult(NSString *a, NSString *b) {
                return [scores[b] compare:scores[a]]; // reversed
            }];
        
        NSLog(@"\nSorted by value (descending):");
        for (NSString *key in sortedByValueDesc) {
            NSLog(@"  %@: %@", key, scores[key]);
        }
        
        // ใช้ NSSortDescriptor ผ่าน array of entries
        NSMutableArray *entries = [NSMutableArray array];
        [scores enumerateKeysAndObjectsUsingBlock:^(id key, id obj, BOOL *stop) {
            [entries addObject:@{@"key": key, @"value": obj}];
        }];
        
        NSSortDescriptor *byValue = [NSSortDescriptor sortDescriptorWithKey:@"value"
                                                                  ascending:NO];
        NSSortDescriptor *byKey = [NSSortDescriptor sortDescriptorWithKey:@"key"
                                                                ascending:YES];
        
        NSArray *sortedEntries = [entries sortedArrayUsingDescriptors:
                                  @[byValue, byKey]];
        
        NSLog(@"\nWith NSSortDescriptor:");
        for (NSDictionary *entry in sortedEntries) {
            NSLog(@"  %@: %@", entry[@"key"], entry[@"value"]);
        }
        
        // Top N entries
        NSArray *top3 = [sortedByValueDesc subarrayWithRange:
                         NSMakeRange(0, MIN(3, sortedByValueDesc.count))];
        NSLog(@"\nTop 3:");
        for (NSString *name in top3) {
            NSLog(@"  %@: %@", name, scores[name]);
        }
    }
    return 0;
}
```

---

## 24.8 Nested Dictionaries

```objc
#import <Foundation/Foundation.h>

int main(int argc, const char * argv[]) {
    @autoreleasepool {
        // Deep nested structure (JSON-like)
        NSDictionary *company = @{
            @"name": @"Tech Corp",
            @"founded": @2010,
            @"address": @{
                @"street": @"123 Silicon Valley",
                @"city": @"San Francisco",
                @"state": @"CA",
                @"zip": @"94102"
            },
            @"departments": @{
                @"engineering": @{
                    @"head": @"Alice",
                    @"employees": @42,
                    @"budget": @5000000
                },
                @"marketing": @{
                    @"head": @"Bob",
                    @"employees": @15,
                    @"budget": @2000000
                }
            }
        };
        
        // Access nested values
        NSString *city = company[@"address"][@"city"];
        NSLog(@"City: %@", city);
        
        NSString *engHead = company[@"departments"][@"engineering"][@"head"];
        NSLog(@"Engineering head: %@", engHead);
        
        // valueForKeyPath: สำหรับ nested access
        NSString *zipKVC = [company valueForKeyPath:@"address.zip"];
        NSLog(@"ZIP (KVC): %@", zipKVC);
        
        NSNumber *engBudget = [company valueForKeyPath:@"departments.engineering.budget"];
        NSLog(@"Eng budget: %@", engBudget);
        
        // Safe nested access (nil coalescing chain)
        NSString *marketing_head = company[@"departments"][@"marketing"][@"head"] ?: @"Unknown";
        NSLog(@"Marketing head: %@", marketing_head);
        
        // Deep nested nil-safe access
        id safeValue = ((NSDictionary *)((NSDictionary *)company[@"departments"])[@"hr"])[@"head"];
        NSLog(@"HR head: %@", safeValue ?: @"Department not found");
        
        // Modify nested mutable dictionary
        NSMutableDictionary *mutableCompany = [company mutableCopy];
        // mutableCompany[@"address"] ยังเป็น immutable!
        // ต้อง mutableCopy ทุก level
        
        NSMutableDictionary *mutableAddress = [mutableCompany[@"address"] mutableCopy];
        mutableAddress[@"zip"] = @"94103";
        mutableCompany[@"address"] = mutableAddress;
        
        NSLog(@"\nUpdated ZIP: %@", mutableCompany[@"address"][@"zip"]);
        
        // สร้าง deep mutable copy ด้วย NSKeyedArchiver
        NSData *data = [NSKeyedArchiver archivedDataWithRootObject:company
                                             requiringSecureCoding:NO
                                                             error:nil];
        NSDictionary *deepCopy = [NSKeyedUnarchiver unarchivedObjectOfClass:[NSDictionary class]
                                                                   fromData:data
                                                                      error:nil];
        NSLog(@"Deep copy type: %@", NSStringFromClass([deepCopy class]));
    }
    return 0;
}
```

---

## 24.9 Dictionary to JSON

```objc
#import <Foundation/Foundation.h>

int main(int argc, const char * argv[]) {
    @autoreleasepool {
        // Dictionary → JSON
        NSDictionary *userData = @{
            @"id": @12345,
            @"name": @"สมชาย ใจดี",
            @"email": @"somchai@example.com",
            @"scores": @[@90, @85, @92],
            @"address": @{
                @"city": @"กรุงเทพฯ",
                @"country": @"Thailand"
            },
            @"active": @YES,
            @"balance": @1234.56
        };
        
        // Serialize to JSON data
        NSError *error;
        NSData *jsonData = [NSJSONSerialization dataWithJSONObject:userData
                                                          options:NSJSONWritingPrettyPrinted
                                                            error:&error];
        if (error) {
            NSLog(@"Serialization error: %@", error.localizedDescription);
        } else {
            NSString *jsonString = [[NSString alloc] initWithData:jsonData
                                                         encoding:NSUTF8StringEncoding];
            NSLog(@"JSON:\n%@", jsonString);
        }
        
        // JSON data → Dictionary
        NSString *jsonInput = @"{"
            "\"name\": \"Alice\","
            "\"age\": 30,"
            "\"scores\": [95, 87, 92],"
            "\"address\": {\"city\": \"Bangkok\"}"
        "}";
        
        NSData *inputData = [jsonInput dataUsingEncoding:NSUTF8StringEncoding];
        NSError *parseError;
        NSDictionary *parsed = [NSJSONSerialization JSONObjectWithData:inputData
                                                               options:0
                                                                 error:&parseError];
        if (parseError) {
            NSLog(@"Parse error: %@", parseError.localizedDescription);
        } else {
            NSLog(@"\nParsed name: %@", parsed[@"name"]);
            NSLog(@"Parsed scores: %@", parsed[@"scores"]);
            NSLog(@"Parsed city: %@", parsed[@"address"][@"city"]);
        }
        
        // JSON Array
        NSArray *jsonArray = @[
            @{@"id": @1, @"value": @"first"},
            @{@"id": @2, @"value": @"second"},
            @{@"id": @3, @"value": @"third"}
        ];
        
        NSData *arrayJSON = [NSJSONSerialization dataWithJSONObject:jsonArray
                                                           options:NSJSONWritingPrettyPrinted
                                                             error:nil];
        NSLog(@"\nArray JSON:\n%@",
              [[NSString alloc] initWithData:arrayJSON encoding:NSUTF8StringEncoding]);
        
        // ตรวจสอบว่า valid JSON object หรือไม่
        BOOL isValid = [NSJSONSerialization isValidJSONObject:userData];
        NSLog(@"\nIs valid JSON object: %@", isValid ? @"YES" : @"NO");
        
        // Keys ที่ไม่ valid สำหรับ JSON (ต้องเป็น NSString)
        NSDictionary *invalidJSON = @{@42: @"number key"}; // key ต้องเป็น string
        BOOL isValidJSON = [NSJSONSerialization isValidJSONObject:invalidJSON];
        NSLog(@"Dict with number key is valid JSON: %@",
              isValidJSON ? @"YES" : @"NO"); // NO
        
        // ถ้าต้องการ number keys ใน JSON ต้อง convert
        NSMutableDictionary *jsonFriendly = [NSMutableDictionary dictionary];
        [invalidJSON enumerateKeysAndObjectsUsingBlock:^(id key, id obj, BOOL *stop) {
            jsonFriendly[[key description]] = obj; // convert key to string
        }];
        NSLog(@"Fixed: %@", jsonFriendly);
    }
    return 0;
}
```

---

## 24.10 Merging Dictionaries

```objc
#import <Foundation/Foundation.h>

int main(int argc, const char * argv[]) {
    @autoreleasepool {
        NSDictionary *defaults = @{
            @"theme": @"light",
            @"language": @"en",
            @"fontSize": @14,
            @"notifications": @YES,
            @"timeout": @30
        };
        
        NSDictionary *userPrefs = @{
            @"theme": @"dark",          // override
            @"language": @"th",         // override
            @"notifications": @NO,      // override
            @"customKey": @"value"      // new key
        };
        
        // Strategy 1: addEntriesFromDictionary: (user prefs win)
        NSMutableDictionary *merged1 = [defaults mutableCopy];
        [merged1 addEntriesFromDictionary:userPrefs]; // userPrefs override defaults
        NSLog(@"Merged (user wins): %@", merged1);
        
        // Strategy 2: defaults win (existing keys not overwritten)
        NSMutableDictionary *merged2 = [userPrefs mutableCopy];
        [defaults enumerateKeysAndObjectsUsingBlock:^(id key, id obj, BOOL *stop) {
            if (!merged2[key]) { // only add if not already present
                merged2[key] = obj;
            }
        }];
        NSLog(@"\nMerged (defaults win): %@", merged2);
        
        // Strategy 3: Deep merge
        NSDictionary *config1 = @{
            @"db": @{@"host": @"localhost", @"port": @5432},
            @"app": @{@"debug": @YES, @"version": @"1.0"}
        };
        
        NSDictionary *config2 = @{
            @"db": @{@"host": @"production.server.com", @"ssl": @YES},
            @"app": @{@"version": @"2.0", @"name": @"MyApp"}
        };
        
        // Deep merge function
        NSDictionary *deepMerge(NSDictionary *base, NSDictionary *override);
        
        NSMutableDictionary *deepMerged = [config1 mutableCopy];
        [config2 enumerateKeysAndObjectsUsingBlock:^(id key, id obj, BOOL *stop) {
            if ([obj isKindOfClass:[NSDictionary class]] &&
                [deepMerged[key] isKindOfClass:[NSDictionary class]]) {
                // Recursively merge nested dictionaries
                NSMutableDictionary *nestedMerge = [deepMerged[key] mutableCopy];
                [nestedMerge addEntriesFromDictionary:obj];
                deepMerged[key] = nestedMerge;
            } else {
                deepMerged[key] = obj;
            }
        }];
        
        NSLog(@"\nDeep merged db: %@", deepMerged[@"db"]);
        // {host: production.server.com, port: 5432, ssl: YES}
        NSLog(@"Deep merged app: %@", deepMerged[@"app"]);
        // {debug: YES, version: 2.0, name: MyApp}
        
        // Merge multiple dictionaries
        NSArray<NSDictionary *> *allConfigs = @[defaults, userPrefs,
                                                @{@"extra": @"data"}];
        NSMutableDictionary *finalMerge = [NSMutableDictionary dictionary];
        for (NSDictionary *config in allConfigs) {
            [finalMerge addEntriesFromDictionary:config];
        }
        NSLog(@"\nFinal merged: %@", finalMerge);
    }
    return 0;
}
```

---

## 24.11 Dictionary กับ Custom Key Types (NSCopying)

```objc
#import <Foundation/Foundation.h>

// Custom key type ต้อง implement NSCopying และ isEqual:/hash
@interface Coordinate : NSObject <NSCopying>

@property (nonatomic, assign) NSInteger x;
@property (nonatomic, assign) NSInteger y;

+ (instancetype)coordinateWithX:(NSInteger)x y:(NSInteger)y;

@end

@implementation Coordinate

+ (instancetype)coordinateWithX:(NSInteger)x y:(NSInteger)y {
    Coordinate *c = [[Coordinate alloc] init];
    c.x = x;
    c.y = y;
    return c;
}

// isEqual: และ hash จำเป็นสำหรับ correctness
- (BOOL)isEqual:(id)other {
    if (self == other) return YES;
    if (![other isKindOfClass:[Coordinate class]]) return NO;
    Coordinate *o = (Coordinate *)other;
    return _x == o.x && _y == o.y;
}

- (NSUInteger)hash {
    return (_x * 31) ^ _y;
}

// NSCopying จำเป็นสำหรับ dictionary key
- (id)copyWithZone:(NSZone *)zone {
    Coordinate *copy = [[[self class] allocWithZone:zone] init];
    copy.x = _x;
    copy.y = _y;
    return copy;
}

- (NSString *)description {
    return [NSString stringWithFormat:@"(%ld,%ld)", (long)_x, (long)_y];
}

@end

int main(int argc, const char * argv[]) {
    @autoreleasepool {
        // ใช้ custom object เป็น key
        NSMutableDictionary *grid = [NSMutableDictionary dictionary];
        
        Coordinate *c1 = [Coordinate coordinateWithX:0 y:0];
        Coordinate *c2 = [Coordinate coordinateWithX:1 y:0];
        Coordinate *c3 = [Coordinate coordinateWithX:0 y:1];
        Coordinate *c4 = [Coordinate coordinateWithX:1 y:1];
        
        grid[c1] = @"origin";
        grid[c2] = @"right";
        grid[c3] = @"top";
        grid[c4] = @"top-right";
        
        // ค้นหาด้วย new coordinate object (isEqual: เปรียบเทียบ content)
        Coordinate *lookup = [Coordinate coordinateWithX:1 y:0];
        NSLog(@"At (1,0): %@", grid[lookup]); // "right"
        
        // ต้อง implement isEqual: ให้ถูกต้อง
        // ถ้าไม่ implement hash/isEqual: แต่ละ object จะ unique ตาม pointer
        
        NSLog(@"\nGrid contents:");
        [grid enumerateKeysAndObjectsUsingBlock:
            ^(Coordinate *coord, NSString *value, BOOL *stop) {
                NSLog(@"  %@ → %@", coord, value);
            }];
        
        // NSNumber เป็น key (มี NSCopying โดย default)
        NSDictionary *indexedValues = @{
            @1: @"one",
            @2: @"two",
            @3: @"three"
        };
        NSLog(@"\nNumeric key: %@", indexedValues[@2]); // "two"
        
        // NSDate เป็น key
        NSDate *today = [NSDate date];
        NSMutableDictionary *calendar = [NSMutableDictionary dictionary];
        calendar[today] = @"Meeting at 10am";
        NSLog(@"Today's event: %@", calendar[today]);
    }
    return 0;
}
```

---

## 24.12 Advanced Patterns

### Pattern 1: Registry Pattern

```objc
#import <Foundation/Foundation.h>

// ใช้ NSDictionary เป็น registry สำหรับ factory
@interface HandlerRegistry : NSObject

+ (instancetype)sharedRegistry;
- (void)registerHandler:(id)handler forEventType:(NSString *)eventType;
- (id)handlerForEventType:(NSString *)eventType;

@end

@implementation HandlerRegistry {
    NSMutableDictionary *_handlers;
}

+ (instancetype)sharedRegistry {
    static HandlerRegistry *instance;
    static dispatch_once_t token;
    dispatch_once(&token, ^{ instance = [[self alloc] init]; });
    return instance;
}

- (instancetype)init {
    self = [super init];
    if (self) {
        _handlers = [NSMutableDictionary dictionary];
    }
    return self;
}

- (void)registerHandler:(id)handler forEventType:(NSString *)eventType {
    _handlers[eventType] = handler;
}

- (id)handlerForEventType:(NSString *)eventType {
    return _handlers[eventType];
}

@end
```

### Pattern 2: Lookup Table / Dispatch Table

```objc
// ใช้ Dictionary แทน if-else chain หรือ switch
typedef void(^ActionBlock)(void);

NSMutableDictionary<NSString *, ActionBlock> *dispatchTable =
    [NSMutableDictionary dictionary];

dispatchTable[@"save"] = ^{
    NSLog(@"Saving...");
};

dispatchTable[@"load"] = ^{
    NSLog(@"Loading...");
};

dispatchTable[@"delete"] = ^{
    NSLog(@"Deleting...");
};

// ใช้งาน
NSString *action = @"save";
ActionBlock handler = dispatchTable[action];
if (handler) {
    handler();
} else {
    NSLog(@"Unknown action: %@", action);
}
```

### Pattern 3: Memoization

```objc
// Cache คำนวณที่แพงใน dictionary
@interface Fibonacci : NSObject

@property (nonatomic, strong) NSMutableDictionary<NSNumber *, NSNumber *> *cache;

- (NSInteger)compute:(NSInteger)n;

@end

@implementation Fibonacci

- (instancetype)init {
    self = [super init];
    if (self) {
        _cache = [NSMutableDictionary dictionary];
    }
    return self;
}

- (NSInteger)compute:(NSInteger)n {
    if (n <= 1) return n;
    
    NSNumber *key = @(n);
    if (_cache[key]) {
        return [_cache[key] integerValue];
    }
    
    NSInteger result = [self compute:n-1] + [self compute:n-2];
    _cache[key] = @(result);
    return result;
}

@end

// ใช้งาน
Fibonacci *fib = [[Fibonacci alloc] init];
for (int i = 0; i <= 20; i++) {
    NSLog(@"F(%d) = %ld", i, (long)[fib compute:i]);
}
NSLog(@"Cache size: %lu", (unsigned long)fib.cache.count);
```

### Pattern 4: NSUserDefaults เป็น Dictionary-like Storage

```objc
// NSUserDefaults ทำงานคล้าย NSDictionary persistent
NSUserDefaults *defaults = [NSUserDefaults standardUserDefaults];

// บันทึก
[defaults setObject:@"dark" forKey:@"theme"];
[defaults setInteger:14 forKey:@"fontSize"];
[defaults setBool:YES forKey:@"notifications"];
[defaults synchronize];

// อ่าน
NSString *theme = [defaults stringForKey:@"theme"];
NSInteger fontSize = [defaults integerForKey:@"fontSize"];
BOOL notifications = [defaults boolForKey:@"notifications"];

NSLog(@"Theme: %@, Font: %ld, Notifications: %d",
      theme, (long)fontSize, notifications);

// Dictionary ของ defaults ทั้งหมด
NSDictionary *allDefaults = [defaults dictionaryRepresentation];
NSLog(@"All defaults count: %lu", (unsigned long)allDefaults.count);
```

---

## 24.13 แบบฝึกหัด 10+ ข้อ

### แบบฝึกหัดที่ 1: Phone Book

```objc
// สร้างระบบ phone book ด้วย NSDictionary

@interface PhoneBook : NSObject

- (void)addContact:(NSString *)name phone:(NSString *)phone;
- (void)removeContact:(NSString *)name;
- (NSString *)phoneForContact:(NSString *)name;
- (NSArray<NSString *> *)contactsStartingWith:(NSString *)prefix;
- (NSDictionary *)allContacts;

@end

@implementation PhoneBook {
    NSMutableDictionary<NSString *, NSString *> *_contacts;
}

- (instancetype)init {
    self = [super init];
    if (self) { _contacts = [NSMutableDictionary dictionary]; }
    return self;
}

- (void)addContact:(NSString *)name phone:(NSString *)phone {
    _contacts[name] = phone;
}

- (void)removeContact:(NSString *)name {
    [_contacts removeObjectForKey:name];
}

- (NSString *)phoneForContact:(NSString *)name {
    return _contacts[name];
}

- (NSArray<NSString *> *)contactsStartingWith:(NSString *)prefix {
    NSPredicate *pred = [NSPredicate predicateWithFormat:
                         @"SELF BEGINSWITH[c] %@", prefix];
    return [[_contacts allKeys] filteredArrayUsingPredicate:pred];
}

- (NSDictionary *)allContacts {
    return [_contacts copy];
}

@end

// ทดสอบ
PhoneBook *book = [[PhoneBook alloc] init];
[book addContact:@"Alice" phone:@"081-111-1111"];
[book addContact:@"Bob" phone:@"082-222-2222"];
[book addContact:@"Alice2" phone:@"083-333-3333"];

NSLog(@"Alice: %@", [book phoneForContact:@"Alice"]);
NSLog(@"Starts with 'A': %@", [book contactsStartingWith:@"A"]);
```

### แบบฝึกหัดที่ 2: Word Frequency Counter

```objc
// นับความถี่ของคำ
NSString *text = @"the quick brown fox jumps over the lazy dog the fox";
NSArray *words = [text componentsSeparatedByString:@" "];

NSMutableDictionary<NSString *, NSNumber *> *freq = [NSMutableDictionary dictionary];
for (NSString *word in words) {
    freq[word] = @([freq[word] integerValue] + 1);
}

// เรียงตามความถี่
NSArray *byFreq = [[freq allKeys]
    sortedArrayUsingComparator:^NSComparisonResult(NSString *a, NSString *b) {
        return [freq[b] compare:freq[a]]; // descending
    }];

NSLog(@"Word frequencies:");
for (NSString *word in byFreq) {
    NSLog(@"  '%@': %@", word, freq[word]);
}
```

### แบบฝึกหัดที่ 3: Config File Parser

```objc
// Parse .env-like config file
NSString *configContent = @"DB_HOST=localhost\n"
                           "DB_PORT=5432\n"
                           "APP_SECRET=mysecretkey\n"
                           "DEBUG=true\n"
                           "# This is a comment\n"
                           "MAX_CONNECTIONS=100";

NSMutableDictionary *config = [NSMutableDictionary dictionary];
NSArray *lines = [configContent componentsSeparatedByString:@"\n"];

for (NSString *line in lines) {
    NSString *trimmed = [line stringByTrimmingCharactersInSet:
                         [NSCharacterSet whitespaceCharacterSet]];
    
    // ข้าม comments และ empty lines
    if ([trimmed hasPrefix:@"#"] || trimmed.length == 0) continue;
    
    NSRange equalsRange = [trimmed rangeOfString:@"="];
    if (equalsRange.location != NSNotFound) {
        NSString *key = [trimmed substringToIndex:equalsRange.location];
        NSString *value = [trimmed substringFromIndex:equalsRange.location + 1];
        config[key] = value;
    }
}

NSLog(@"DB_HOST: %@", config[@"DB_HOST"]);
NSLog(@"DEBUG: %@", config[@"DEBUG"]);
NSLog(@"All config keys: %@", [[config allKeys] sortedArrayUsingSelector:@selector(compare:)]);
```

### แบบฝึกหัดที่ 4: Grade Calculator

```objc
// คำนวณ grade distribution
NSArray<NSDictionary *> *students = @[
    @{@"name": @"Alice", @"score": @92},
    @{@"name": @"Bob", @"score": @78},
    @{@"name": @"Charlie", @"score": @85},
    @{@"name": @"Dave", @"score": @65},
    @{@"name": @"Eve", @"score": @95},
    @{@"name": @"Frank", @"score": @72},
];

// กำหนด grade ranges
NSDictionary *gradeRanges = @{
    @"A": @[@90, @100],
    @"B": @[@80, @89],
    @"C": @[@70, @79],
    @"D": @[@60, @69],
    @"F": @[@0, @59]
};

// คำนวณ grade distribution
NSMutableDictionary *distribution = [NSMutableDictionary dictionary];
for (NSDictionary *student in students) {
    NSInteger score = [student[@"score"] integerValue];
    NSString *grade = @"F";
    
    for (NSString *g in gradeRanges) {
        NSArray *range = gradeRanges[g];
        if (score >= [range[0] integerValue] &&
            score <= [range[1] integerValue]) {
            grade = g;
            break;
        }
    }
    
    distribution[grade] = @([distribution[grade] integerValue] + 1);
    NSLog(@"%@: %@", student[@"name"], grade);
}

NSLog(@"\nDistribution: %@", distribution);
```

### แบบฝึกหัดที่ 5: Cache Manager

```objc
@interface CacheManager : NSObject

- (void)setObject:(id)object forKey:(NSString *)key expiresIn:(NSTimeInterval)seconds;
- (id)objectForKey:(NSString *)key;
- (void)removeExpired;
- (void)removeAll;

@end

@implementation CacheManager {
    NSMutableDictionary *_cache;
    NSMutableDictionary *_expiry;
}

- (instancetype)init {
    self = [super init];
    if (self) {
        _cache = [NSMutableDictionary dictionary];
        _expiry = [NSMutableDictionary dictionary];
    }
    return self;
}

- (void)setObject:(id)object forKey:(NSString *)key expiresIn:(NSTimeInterval)seconds {
    _cache[key] = object;
    _expiry[key] = [NSDate dateWithTimeIntervalSinceNow:seconds];
}

- (id)objectForKey:(NSString *)key {
    NSDate *expiry = _expiry[key];
    if (expiry && [expiry timeIntervalSinceNow] < 0) {
        // Expired
        [_cache removeObjectForKey:key];
        [_expiry removeObjectForKey:key];
        return nil;
    }
    return _cache[key];
}

- (void)removeExpired {
    NSDate *now = [NSDate date];
    NSMutableArray *expiredKeys = [NSMutableArray array];
    
    [_expiry enumerateKeysAndObjectsUsingBlock:^(id key, NSDate *expiry, BOOL *stop) {
        if ([expiry timeIntervalSinceDate:now] < 0) {
            [expiredKeys addObject:key];
        }
    }];
    
    [_cache removeObjectsForKeys:expiredKeys];
    [_expiry removeObjectsForKeys:expiredKeys];
    
    NSLog(@"Removed %lu expired items", (unsigned long)expiredKeys.count);
}

- (void)removeAll {
    [_cache removeAllObjects];
    [_expiry removeAllObjects];
}

@end
```

### แบบฝึกหัดที่ 6: Grouping/Categorizing

```objc
// Group objects ตาม property
NSArray *products = @[
    @{@"name": @"MacBook", @"category": @"computer", @"price": @59900},
    @{@"name": @"iPhone", @"category": @"phone", @"price": @29900},
    @{@"name": @"iPad", @"category": @"tablet", @"price": @19900},
    @{@"name": @"iMac", @"category": @"computer", @"price": @49900},
    @{@"name": @"Apple Watch", @"category": @"wearable", @"price": @12900},
    @{@"name": @"MacBook Air", @"category": @"computer", @"price": @39900},
];

// Group by category
NSMutableDictionary<NSString *, NSMutableArray *> *grouped =
    [NSMutableDictionary dictionary];

for (NSDictionary *product in products) {
    NSString *category = product[@"category"];
    if (!grouped[category]) {
        grouped[category] = [NSMutableArray array];
    }
    [grouped[category] addObject:product];
}

NSLog(@"Categories:");
[[grouped allKeys] sortedArrayUsingSelector:@selector(compare:)];
for (NSString *cat in [[grouped allKeys] sortedArrayUsingSelector:@selector(compare:)]) {
    NSArray *items = grouped[cat];
    double total = [[items valueForKeyPath:@"@sum.price"] doubleValue];
    NSLog(@"  %@ (%lu items, total: %.0f):", cat,
          (unsigned long)items.count, total);
    for (NSDictionary *p in items) {
        NSLog(@"    - %@ (%.0f)", p[@"name"], [p[@"price"] doubleValue]);
    }
}
```

### แบบฝึกหัดที่ 7: Two-way Lookup

```objc
// สร้าง bidirectional dictionary
@interface BiDictionary : NSObject

- (void)setKey:(id)key value:(id)value;
- (id)valueForKey:(id)key;
- (id)keyForValue:(id)value;
- (void)removeKey:(id)key;

@end

@implementation BiDictionary {
    NSMutableDictionary *_forward;  // key → value
    NSMutableDictionary *_reverse;  // value → key
}

- (instancetype)init {
    self = [super init];
    if (self) {
        _forward = [NSMutableDictionary dictionary];
        _reverse = [NSMutableDictionary dictionary];
    }
    return self;
}

- (void)setKey:(id<NSCopying>)key value:(id<NSCopying>)value {
    // ลบ old mappings ถ้ามี
    id oldValue = _forward[key];
    if (oldValue) [_reverse removeObjectForKey:oldValue];
    
    id oldKey = _reverse[value];
    if (oldKey) [_forward removeObjectForKey:oldKey];
    
    _forward[key] = value;
    _reverse[value] = key;
}

- (id)valueForKey:(id)key {
    return _forward[key];
}

- (id)keyForValue:(id)value {
    return _reverse[value];
}

- (void)removeKey:(id)key {
    id value = _forward[key];
    if (value) [_reverse removeObjectForKey:value];
    [_forward removeObjectForKey:key];
}

@end

// ใช้งาน
BiDictionary *biDict = [[BiDictionary alloc] init];
[biDict setKey:@"TH" value:@"Thailand"];
[biDict setKey:@"US" value:@"United States"];
[biDict setKey:@"JP" value:@"Japan"];

NSLog(@"TH → %@", [biDict valueForKey:@"TH"]); // Thailand
NSLog(@"Thailand → %@", [biDict keyForValue:@"Thailand"]); // TH
```

### แบบฝึกหัดที่ 8: Histogram

```objc
// สร้าง histogram จาก data
NSArray *scores = @[@85, @92, @78, @65, @90, @88, @72, @95, @83, @76];

NSMutableDictionary *histogram = [NSMutableDictionary dictionary];
NSArray *buckets = @[@"50-59", @"60-69", @"70-79", @"80-89", @"90-100"];

// Initialize buckets
for (NSString *bucket in buckets) {
    histogram[bucket] = @0;
}

// Fill histogram
for (NSNumber *score in scores) {
    NSInteger s = [score integerValue];
    NSString *bucket;
    if (s >= 90) bucket = @"90-100";
    else if (s >= 80) bucket = @"80-89";
    else if (s >= 70) bucket = @"70-79";
    else if (s >= 60) bucket = @"60-69";
    else bucket = @"50-59";
    
    histogram[bucket] = @([histogram[bucket] integerValue] + 1);
}

// แสดง histogram
NSLog(@"Score Distribution:");
for (NSString *bucket in buckets) {
    NSInteger count = [histogram[bucket] integerValue];
    NSMutableString *bar = [NSMutableString string];
    for (NSInteger i = 0; i < count; i++) [bar appendString:@"█"];
    NSLog(@"  %@: %@ (%ld)", bucket, bar, (long)count);
}
```

### แบบฝึกหัดที่ 9: Configuration Manager

```objc
// Configuration manager with type-safe access
@interface ConfigManager : NSObject

+ (instancetype)sharedManager;
- (void)loadDefaults:(NSDictionary *)defaults;
- (void)setString:(NSString *)value forKey:(NSString *)key;
- (void)setInteger:(NSInteger)value forKey:(NSString *)key;
- (void)setBool:(BOOL)value forKey:(NSString *)key;
- (NSString *)stringForKey:(NSString *)key defaultValue:(NSString *)def;
- (NSInteger)integerForKey:(NSString *)key defaultValue:(NSInteger)def;
- (BOOL)boolForKey:(NSString *)key defaultValue:(BOOL)def;

@end

@implementation ConfigManager {
    NSMutableDictionary *_config;
}

+ (instancetype)sharedManager {
    static ConfigManager *instance;
    static dispatch_once_t token;
    dispatch_once(&token, ^{ instance = [[self alloc] init]; });
    return instance;
}

- (instancetype)init {
    self = [super init];
    if (self) { _config = [NSMutableDictionary dictionary]; }
    return self;
}

- (void)loadDefaults:(NSDictionary *)defaults {
    [_config addEntriesFromDictionary:defaults];
}

- (void)setString:(NSString *)value forKey:(NSString *)key {
    _config[key] = value;
}

- (void)setInteger:(NSInteger)value forKey:(NSString *)key {
    _config[key] = @(value);
}

- (void)setBool:(BOOL)value forKey:(NSString *)key {
    _config[key] = @(value);
}

- (NSString *)stringForKey:(NSString *)key defaultValue:(NSString *)def {
    return _config[key] ?: def;
}

- (NSInteger)integerForKey:(NSString *)key defaultValue:(NSInteger)def {
    return _config[key] ? [_config[key] integerValue] : def;
}

- (BOOL)boolForKey:(NSString *)key defaultValue:(BOOL)def {
    return _config[key] ? [_config[key] boolValue] : def;
}

@end

// ใช้งาน
ConfigManager *config = [ConfigManager sharedManager];
[config loadDefaults:@{
    @"theme": @"light",
    @"fontSize": @14,
    @"debug": @NO
}];

[config setString:@"dark" forKey:@"theme"];
[config setBool:YES forKey:@"debug"];

NSLog(@"Theme: %@", [config stringForKey:@"theme" defaultValue:@"light"]);
NSLog(@"Font: %ld", (long)[config integerForKey:@"fontSize" defaultValue:12]);
NSLog(@"Debug: %d", [config boolForKey:@"debug" defaultValue:NO]);
NSLog(@"Unknown: %@", [config stringForKey:@"unknown" defaultValue:@"N/A"]);
```

### แบบฝึกหัดที่ 10: JSON API Response Parser

```objc
// Parse JSON API response
NSString *apiResponse = @"{"
    "\"status\": \"success\","
    "\"data\": {"
        "\"users\": ["
            "{\"id\": 1, \"name\": \"Alice\", \"role\": \"admin\"},"
            "{\"id\": 2, \"name\": \"Bob\", \"role\": \"user\"},"
            "{\"id\": 3, \"name\": \"Charlie\", \"role\": \"user\"}"
        "],"
        "\"total\": 3,"
        "\"page\": 1"
    "}"
"}";

NSData *responseData = [apiResponse dataUsingEncoding:NSUTF8StringEncoding];
NSDictionary *parsed = [NSJSONSerialization JSONObjectWithData:responseData
                                                       options:0
                                                         error:nil];

NSString *status = parsed[@"status"];
NSDictionary *data = parsed[@"data"];
NSArray *users = data[@"users"];
NSInteger total = [data[@"total"] integerValue];

NSLog(@"Status: %@", status);
NSLog(@"Total users: %ld", (long)total);

// หา admins
NSPredicate *adminPred = [NSPredicate predicateWithFormat:@"role == %@", @"admin"];
NSArray *admins = [users filteredArrayUsingPredicate:adminPred];
NSLog(@"Admins:");
for (NSDictionary *admin in admins) {
    NSLog(@"  %@: %@", admin[@"id"], admin[@"name"]);
}

// Build lookup dictionary
NSMutableDictionary *userById = [NSMutableDictionary dictionary];
for (NSDictionary *user in users) {
    userById[user[@"id"]] = user;
}

NSLog(@"User #2: %@", userById[@2][@"name"]); // Bob
```

---

## 24.14 Performance Considerations

```objc
#import <Foundation/Foundation.h>

int main(int argc, const char * argv[]) {
    @autoreleasepool {
        // 1. ใช้ capacity เมื่อรู้ขนาดล่วงหน้า
        NSMutableDictionary *dict = [NSMutableDictionary dictionaryWithCapacity:1000];
        
        // 2. Dictionary lookup เร็วมาก O(1) average
        NSMutableDictionary *lookup = [NSMutableDictionary dictionary];
        for (int i = 0; i < 100000; i++) {
            lookup[@(i)] = @(i * i);
        }
        
        NSDate *start = [NSDate date];
        for (int i = 0; i < 1000; i++) {
            NSNumber *val = lookup[@(arc4random_uniform(100000))];
            (void)val;
        }
        NSLog(@"1000 lookups: %.4fs", -[start timeIntervalSinceNow]);
        
        // 3. Immutable ดีกว่า mutable สำหรับ thread safety
        NSDictionary *immutable = [lookup copy]; // thread-safe reads
        
        // 4. ระวัง mutable dictionary ใน multi-threaded code
        // ใช้ NSLock หรือ dispatch_sync
        NSMutableDictionary *sharedDict = [NSMutableDictionary dictionary];
        NSLock *lock = [[NSLock alloc] init];
        
        // Thread-safe access
        void (^safeSet)(NSString *, id) = ^(NSString *key, id value) {
            [lock lock];
            sharedDict[key] = value;
            [lock unlock];
        };
        
        void (^safeGet)(NSString *) = ^(NSString *key) {
            [lock lock];
            id value = sharedDict[key];
            [lock unlock];
            NSLog(@"Got: %@", value);
        };
        
        // 5. NSDictionary กับ NSCache สำหรับ memory-sensitive caching
        NSCache *cache = [[NSCache alloc] init];
        cache.countLimit = 100; // ไม่เกิน 100 items
        cache.totalCostLimit = 1024 * 1024; // ไม่เกิน 1MB
        
        [cache setObject:@"cached value" forKey:@"key1" cost:100];
        NSLog(@"Cache: %@", [cache objectForKey:@"key1"]);
    }
    return 0;
}
```

---

## 24.15 Best Practices

### 1. ใช้ typed generics

```objc
// ดี - ชัดเจนเรื่อง types
NSDictionary<NSString *, NSNumber *> *scores = @{
    @"Alice": @95,
    @"Bob": @87
};

// ไม่ดี
NSDictionary *scores2 = @{@"Alice": @95};
```

### 2. Return immutable จาก methods

```objc
- (NSDictionary<NSString *, id> *)configuration {
    return [_mutableConfig copy]; // return immutable copy
}
```

### 3. Nil-safe value access

```objc
// Pattern ที่ดี
NSString *value = dict[@"key"] ?: @"default";

// หรือ
NSString *value2;
if ((value2 = dict[@"key"]) == nil) {
    value2 = @"default";
}
```

### 4. ระวัง valueForKey: ใน NSDictionary

```objc
// objectForKey: ปลอดภัย
id val1 = [dict objectForKey:@"key"]; // nil ถ้าไม่มี

// valueForKey: อาจ throw สำหรับ keys พิเศษ
id val2 = [dict valueForKey:@"@count"]; // ได้ count ไม่ใช่ nil!
id val3 = [dict valueForKey:@"key"];    // ปกติ
```

### 5. JSON serialization requirements

```objc
// Keys ต้องเป็น NSString
// Values ต้องเป็น: NSString, NSNumber, NSArray, NSDictionary, NSNull

// ไม่สามารถ serialize:
NSDictionary *invalid = @{
    @"date": [NSDate date],  // NSDate ไม่ valid สำหรับ JSON
    @"data": [NSData data]   // NSData ไม่ valid สำหรับ JSON
};

// ต้อง convert ก่อน
NSDictionary *valid = @{
    @"date": [[NSDateFormatter new] stringFromDate:[NSDate date]],
    @"data": [[NSData data] base64EncodedStringWithOptions:0]
};
```

---

## สรุป

NSDictionary และ NSMutableDictionary เป็น collection ที่ทรงพลังสำหรับ key-value storage:

1. **สร้าง** ด้วย literal `@{}`, `dictionaryWithObjectsAndKeys:` (ระวัง key-value order)
2. **เข้าถึง** ด้วย subscript `dict[key]` หรือ `objectForKey:` (คืน nil ถ้าไม่มี)
3. **Enumerate** ด้วย for-in (keys) หรือ `enumerateKeysAndObjectsUsingBlock:`
4. **Sort** ผ่านการ sort ใน array of keys
5. **Filter** ด้วย `keysOfEntriesPassingTest:` หรือ filter บน keys/values arrays
6. **JSON** ด้วย `NSJSONSerialization` สองทิศทาง
7. **Custom keys** ต้อง implement `NSCopying`, `isEqual:`, `hash`
8. **Merge** ด้วย `addEntriesFromDictionary:` หรือ `setValuesForKeysWithDictionary:`

### Pattern สรุป

| Use Case | Approach |
|----------|---------|
| Store key-value | `NSDictionary` literal `@{}` |
| Modify after create | `NSMutableDictionary` |
| Lookup by key | `dict[key]` หรือ `objectForKey:` |
| Iterate all | `enumerateKeysAndObjectsUsingBlock:` |
| Type-safe | `NSDictionary<KeyType, ValueType *>` |
| JSON storage | `NSJSONSerialization` |
| Persistent | `NSUserDefaults` หรือ plist file |
| Memory-sensitive | `NSCache` |
| Thread-safe | `NSLock` + `NSMutableDictionary` |

ในบทต่อไปเราจะเรียนรู้เกี่ยวกับ **NSSet, NSOrderedSet และ NSCountedSet** ซึ่งเป็น collection สำหรับ unique elements
