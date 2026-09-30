# ตอนที่ 23 - NSArray และ NSMutableArray

## บทนำ

`NSArray` เป็น ordered collection ที่เก็บ objects แบบ immutable (ไม่สามารถเปลี่ยนแปลงหลัง create แล้ว) ส่วน `NSMutableArray` เป็น subclass ที่ allow modification ในบทนี้เราจะเรียนรู้การใช้งาน arrays อย่างครบถ้วน

---

## 23.1 การสร้าง Arrays

### 23.1.1 Literal Syntax @[]

```objc
#import <Foundation/Foundation.h>

int main(int argc, const char * argv[]) {
    @autoreleasepool {
        // Empty array
        NSArray *empty = @[];
        
        // Array with elements
        NSArray *fruits = @[@"apple", @"banana", @"cherry"];
        
        // Mixed types (ทุกอย่างต้องเป็น objects)
        NSArray *mixed = @[@"text", @42, @3.14, @YES, [NSDate date]];
        
        // Nested array
        NSArray *matrix = @[
            @[@1, @2, @3],
            @[@4, @5, @6],
            @[@7, @8, @9]
        ];
        
        NSLog(@"Fruits: %@", fruits);
        NSLog(@"Mixed: %@", mixed);
        NSLog(@"First row: %@", matrix[0]);
        
        // สร้างจาก single object
        NSArray *single = [NSArray arrayWithObject:@"only"];
        
        // หมายเหตุ: @[] ไม่รับ nil - ถ้าใส่ nil จะ crash
        // NSString *nilStr = nil;
        // NSArray *bad = @[nilStr]; // CRASH!
    }
    return 0;
}
```

### 23.1.2 arrayWithObjects:

```objc
// arrayWithObjects: - รับ nil เป็น terminator
NSArray *arr1 = [NSArray arrayWithObjects:@"a", @"b", @"c", nil];

// arrayWithObjects:count: - ระบุจำนวน
id objects[] = {@"x", @"y", @"z"};
NSArray *arr2 = [NSArray arrayWithObjects:objects count:3];

// arrayWithArray: - copy จาก array อื่น
NSArray *original = @[@1, @2, @3];
NSArray *copy = [NSArray arrayWithArray:original];

// arrayWithContentsOfFile: - อ่านจาก plist file
NSArray *fromFile = [NSArray arrayWithContentsOfFile:@"/path/to/array.plist"];

// arrayWithContentsOfURL: - อ่านจาก URL
NSURL *url = [NSURL fileURLWithPath:@"/path/to/array.plist"];
NSArray *fromURL = [NSArray arrayWithContentsOfURL:url];

NSLog(@"arr1: %@", arr1);
NSLog(@"arr2: %@", arr2);
NSLog(@"copy: %@", copy);
```

### 23.1.3 NSMutableArray Creation

```objc
// Empty mutable array
NSMutableArray *m1 = [NSMutableArray array];

// With capacity (performance hint)
NSMutableArray *m2 = [NSMutableArray arrayWithCapacity:100];

// From immutable
NSMutableArray *m3 = [@[@1, @2, @3] mutableCopy];

// From array
NSMutableArray *m4 = [NSMutableArray arrayWithArray:@[@"a", @"b"]];

// alloc-init
NSMutableArray *m5 = [[NSMutableArray alloc] init];
NSMutableArray *m6 = [[NSMutableArray alloc] initWithCapacity:50];
NSMutableArray *m7 = [[NSMutableArray alloc] initWithArray:@[@1, @2]];

// initWithObjects:
NSMutableArray *m8 = [[NSMutableArray alloc] initWithObjects:@"x", @"y", nil];
```

---

## 23.2 การเข้าถึง Elements

```objc
#import <Foundation/Foundation.h>

int main(int argc, const char * argv[]) {
    @autoreleasepool {
        NSArray *arr = @[@"zero", @"one", @"two", @"three", @"four"];
        
        // Subscript syntax (modern)
        NSString *first = arr[0];
        NSString *third = arr[2];
        NSLog(@"First: %@, Third: %@", first, third);
        
        // objectAtIndex: (classic)
        NSString *last = [arr objectAtIndex:arr.count - 1];
        NSLog(@"Last: %@", last);
        
        // firstObject / lastObject (safe - ไม่ crash ถ้า empty)
        NSLog(@"First: %@", arr.firstObject);
        NSLog(@"Last: %@", arr.lastObject);
        
        NSArray *empty = @[];
        NSLog(@"Empty firstObject: %@", empty.firstObject); // nil (ไม่ crash)
        // empty[0] จะ crash!
        
        // count
        NSLog(@"Count: %lu", (unsigned long)arr.count);
        
        // Safe access
        NSUInteger safeIndex = 10;
        if (safeIndex < arr.count) {
            NSLog(@"Safe: %@", arr[safeIndex]);
        } else {
            NSLog(@"Index out of bounds");
        }
        
        // objectsAtIndexes: - หลาย indexes พร้อมกัน
        NSIndexSet *indexSet = [NSIndexSet indexSetWithIndexesInRange:NSMakeRange(1, 3)];
        NSArray *subset = [arr objectsAtIndexes:indexSet];
        NSLog(@"Subset [1..3]: %@", subset); // [one, two, three]
        
        // subarrayWithRange:
        NSArray *sub = [arr subarrayWithRange:NSMakeRange(1, 3)];
        NSLog(@"Subarray: %@", sub); // [one, two, three]
        
        // valueForKey: - KVC
        NSArray *lengths = [arr valueForKey:@"length"];
        NSLog(@"Lengths: %@", lengths); // [4, 3, 3, 5, 4]
    }
    return 0;
}
```

---

## 23.3 Fast Enumeration (for-in)

```objc
#import <Foundation/Foundation.h>

int main(int argc, const char * argv[]) {
    @autoreleasepool {
        NSArray *numbers = @[@10, @20, @30, @40, @50];
        
        // Basic fast enumeration
        NSLog(@"All numbers:");
        for (NSNumber *num in numbers) {
            NSLog(@"  %@", num);
        }
        
        // กับ typed array (Objective-C generics)
        NSArray<NSString *> *names = @[@"Alice", @"Bob", @"Charlie"];
        for (NSString *name in names) {
            NSLog(@"Hello, %@!", name);
        }
        
        // Enumerate ใน reverse
        for (id obj in [numbers reverseObjectEnumerator]) {
            NSLog(@"Reverse: %@", obj);
        }
        
        // Enumerate with index (ใช้ block แทน)
        // for-in ไม่มี index โดยตรง
        NSUInteger index = 0;
        for (NSNumber *num in numbers) {
            NSLog(@"[%lu] %@", (unsigned long)index, num);
            index++;
        }
        
        // Break ใน for-in
        for (NSNumber *num in numbers) {
            if ([num integerValue] == 30) {
                NSLog(@"Found 30, stopping");
                break;
            }
        }
        
        // Continue ใน for-in
        for (NSNumber *num in numbers) {
            if ([num integerValue] % 20 == 0) continue;
            NSLog(@"Not divisible by 20: %@", num);
        }
        
        // Nested for-in
        NSArray *matrix = @[
            @[@1, @2, @3],
            @[@4, @5, @6],
            @[@7, @8, @9]
        ];
        
        for (NSArray *row in matrix) {
            for (NSNumber *cell in row) {
                NSLog(@"  Cell: %@", cell);
            }
        }
        
        // ข้อสำคัญ: ห้ามแก้ไข array ระหว่าง fast enumeration!
        NSMutableArray *mutable = [@[@1, @2, @3] mutableCopy];
        NSMutableArray *toRemove = [NSMutableArray array];
        
        // วิธีถูก: collect แล้วค่อย remove
        for (NSNumber *num in mutable) {
            if ([num integerValue] % 2 == 0) {
                [toRemove addObject:num];
            }
        }
        [mutable removeObjectsInArray:toRemove];
        NSLog(@"After remove even: %@", mutable);
    }
    return 0;
}
```

---

## 23.4 Block-based Enumeration

```objc
#import <Foundation/Foundation.h>

int main(int argc, const char * argv[]) {
    @autoreleasepool {
        NSArray *fruits = @[@"apple", @"banana", @"cherry", @"date", @"elderberry"];
        
        // enumerateObjectsUsingBlock:
        [fruits enumerateObjectsUsingBlock:^(id obj, NSUInteger idx, BOOL *stop) {
            NSLog(@"[%lu] %@", (unsigned long)idx, obj);
        }];
        
        // หยุดกลางทาง
        [fruits enumerateObjectsUsingBlock:^(NSString *fruit, NSUInteger idx, BOOL *stop) {
            NSLog(@"Checking: %@", fruit);
            if ([fruit isEqualToString:@"cherry"]) {
                *stop = YES; // หยุด enumeration
            }
        }];
        
        // Concurrent enumeration (ลำดับไม่แน่นอน แต่เร็วกว่า)
        NSLog(@"\nConcurrent (order may vary):");
        [fruits enumerateObjectsWithOptions:NSEnumerationConcurrent
                                 usingBlock:^(NSString *fruit, NSUInteger idx, BOOL *stop) {
            NSLog(@"[%lu] %@", (unsigned long)idx, fruit);
        }];
        
        // Reverse enumeration
        NSLog(@"\nReverse:");
        [fruits enumerateObjectsWithOptions:NSEnumerationReverse
                                 usingBlock:^(NSString *fruit, NSUInteger idx, BOOL *stop) {
            NSLog(@"[%lu] %@", (unsigned long)idx, fruit);
        }];
        
        // makeObjectsPerformSelector: - เรียก method บน ทุก object
        NSArray<NSMutableString *> *mutableStrings = @[
            [@"hello" mutableCopy],
            [@"world" mutableCopy]
        ];
        // [mutableStrings makeObjectsPerformSelector:@selector(appendString:)
        //                                withObject:@"!"];
        
        // Transform ทั้ง array ด้วย valueForKey:
        NSArray *numbers = @[@1, @2, @3, @4, @5];
        
        // ใช้ KVC เพื่อ transform
        NSArray *stringNumbers = [numbers valueForKey:@"stringValue"];
        NSLog(@"String numbers: %@", stringNumbers);
        
        // ใช้ NSSet เพื่อ aggregate
        NSArray *salaries = @[@50000, @70000, @50000, @80000, @70000];
        NSNumber *sum = [salaries valueForKeyPath:@"@sum.self"];
        NSNumber *avg = [salaries valueForKeyPath:@"@avg.self"];
        NSNumber *max = [salaries valueForKeyPath:@"@max.self"];
        NSNumber *min = [salaries valueForKeyPath:@"@min.self"];
        NSNumber *count = [salaries valueForKeyPath:@"@count"];
        
        NSLog(@"\nSalary stats:");
        NSLog(@"Sum: %@", sum);
        NSLog(@"Avg: %@", avg);
        NSLog(@"Max: %@", max);
        NSLog(@"Min: %@", min);
        NSLog(@"Count: %@", count);
    }
    return 0;
}
```

---

## 23.5 Sorting

### 23.5.1 sortedArrayUsingSelector:

```objc
#import <Foundation/Foundation.h>

int main(int argc, const char * argv[]) {
    @autoreleasepool {
        NSArray *numbers = @[@3, @1, @4, @1, @5, @9, @2, @6];
        NSArray *strings = @[@"banana", @"Apple", @"cherry", @"Date"];
        
        // Sort numbers
        NSArray *sortedNums = [numbers sortedArrayUsingSelector:@selector(compare:)];
        NSLog(@"Sorted numbers: %@", sortedNums);
        
        // Sort strings (case-sensitive)
        NSArray *sortedStr1 = [strings sortedArrayUsingSelector:@selector(compare:)];
        NSLog(@"Case-sensitive sort: %@", sortedStr1);
        
        // Sort strings (case-insensitive)
        NSArray *sortedStr2 = [strings sortedArrayUsingSelector:
                               @selector(caseInsensitiveCompare:)];
        NSLog(@"Case-insensitive sort: %@", sortedStr2);
        
        // localizedCompare: - เหมาะสำหรับ user-facing
        NSArray *sortedStr3 = [strings sortedArrayUsingSelector:
                               @selector(localizedCaseInsensitiveCompare:)];
        NSLog(@"Localized sort: %@", sortedStr3);
    }
    return 0;
}
```

### 23.5.2 sortedArrayUsingComparator:

```objc
NSArray *people = @[
    @{@"name": @"Charlie", @"age": @25},
    @{@"name": @"Alice", @"age": @30},
    @{@"name": @"Bob", @"age": @25},
    @{@"name": @"Dave", @"age": @20}
];

// Sort by age ascending
NSArray *byAge = [people sortedArrayUsingComparator:
    ^NSComparisonResult(NSDictionary *a, NSDictionary *b) {
        return [a[@"age"] compare:b[@"age"]];
    }];
NSLog(@"By age: %@", [byAge valueForKey:@"name"]);

// Sort by age desc, then name asc
NSArray *byAgeDescNameAsc = [people sortedArrayUsingComparator:
    ^NSComparisonResult(NSDictionary *a, NSDictionary *b) {
        NSComparisonResult ageResult = [b[@"age"] compare:a[@"age"]]; // reversed
        if (ageResult != NSOrderedSame) return ageResult;
        return [a[@"name"] compare:b[@"name"]];
    }];
NSLog(@"By age desc, name asc: %@",
      [byAgeDescNameAsc valueForKey:@"name"]);
// Charlie, Bob (age 25, alphabetical), Alice (30), Dave (20)
// wait, desc age: Alice(30), Charlie/Bob(25), Dave(20)

// Sort with stability guarantee
// NSArray sortedArrayUsingComparator: ไม่รับประกัน stability
// ใช้ sortedArrayWithOptions:usingComparator: แทน
NSArray *stable = [people sortedArrayWithOptions:NSSortStable
                                 usingComparator:
    ^NSComparisonResult(NSDictionary *a, NSDictionary *b) {
        return [a[@"age"] compare:b[@"age"]];
    }];
NSLog(@"Stable sort: %@", [stable valueForKey:@"name"]);
```

### 23.5.3 NSSortDescriptor

```objc
// Sort ด้วย NSSortDescriptor
NSArray *employees = @[
    @{@"name": @"Charlie", @"dept": @"Engineering", @"salary": @70000},
    @{@"name": @"Alice", @"dept": @"HR", @"salary": @50000},
    @{@"name": @"Bob", @"dept": @"Engineering", @"salary": @80000},
    @{@"name": @"Dave", @"dept": @"HR", @"salary": @60000},
    @{@"name": @"Eve", @"dept": @"Engineering", @"salary": @65000}
];

// Sort by dept ascending
NSSortDescriptor *byDept = [NSSortDescriptor sortDescriptorWithKey:@"dept"
                                                         ascending:YES];

// Sort by salary descending
NSSortDescriptor *bySalaryDesc = [NSSortDescriptor sortDescriptorWithKey:@"salary"
                                                              ascending:NO];

// Sort by name ascending, case-insensitive
NSSortDescriptor *byName = [NSSortDescriptor sortDescriptorWithKey:@"name"
                                                         ascending:YES
                                                          selector:@selector(caseInsensitiveCompare:)];

// รวม descriptors (dept first, then salary)
NSArray *descriptors = @[byDept, bySalaryDesc, byName];
NSArray *sorted = [employees sortedArrayUsingDescriptors:descriptors];

for (NSDictionary *emp in sorted) {
    NSLog(@"%@ (%@) - $%@",
          emp[@"name"], emp[@"dept"], emp[@"salary"]);
}
// Engineering: Bob(80k), Eve(65k), Charlie(70k)
// HR: Dave(60k), Alice(50k)

// In-place sort สำหรับ NSMutableArray
NSMutableArray *mutableEmp = [employees mutableCopy];
[mutableEmp sortUsingDescriptors:descriptors];

// Sort strings with custom locale
NSSortDescriptor *localizedSort = [[NSSortDescriptor alloc]
    initWithKey:@"name"
      ascending:YES
     comparator:^NSComparisonResult(NSString *a, NSString *b) {
         return [a localizedCompare:b];
     }];
```

---

## 23.6 Filtering

### 23.6.1 filteredArrayUsingPredicate:

```objc
#import <Foundation/Foundation.h>

int main(int argc, const char * argv[]) {
    @autoreleasepool {
        NSArray *numbers = @[@1, @5, @3, @8, @2, @9, @4, @7, @6];
        
        // กรองเลขคู่
        NSPredicate *evenPredicate = [NSPredicate predicateWithBlock:
            ^BOOL(NSNumber *n, NSDictionary *bindings) {
                return [n integerValue] % 2 == 0;
            }];
        NSArray *even = [numbers filteredArrayUsingPredicate:evenPredicate];
        NSLog(@"Even: %@", even);
        
        // กรองด้วย format string
        NSPredicate *greaterThan5 = [NSPredicate predicateWithFormat:@"self > 5"];
        NSArray *big = [numbers filteredArrayUsingPredicate:greaterThan5];
        NSLog(@"Greater than 5: %@", big);
        
        // กรอง objects ที่ซับซ้อน
        NSArray *products = @[
            @{@"name": @"MacBook", @"price": @59900, @"category": @"computer"},
            @{@"name": @"iPhone", @"price": @29900, @"category": @"phone"},
            @{@"name": @"iPad", @"price": @19900, @"category": @"tablet"},
            @{@"name": @"iMac", @"price": @49900, @"category": @"computer"},
            @{@"name": @"AirPods", @"price": @5900, @"category": @"audio"},
        ];
        
        // กรองตาม category
        NSPredicate *computers = [NSPredicate predicateWithFormat:
                                   @"category == %@", @"computer"];
        NSArray *computerProducts = [products filteredArrayUsingPredicate:computers];
        NSLog(@"\nComputers: %@", [computerProducts valueForKey:@"name"]);
        
        // กรองตาม price range
        NSPredicate *midRange = [NSPredicate predicateWithFormat:
                                  @"price >= %@ AND price <= %@", @10000, @40000];
        NSArray *midRangeProducts = [products filteredArrayUsingPredicate:midRange];
        NSLog(@"Mid-range: %@", [midRangeProducts valueForKey:@"name"]);
        
        // กรองด้วย string operations
        NSPredicate *startsWithi = [NSPredicate predicateWithFormat:
                                     @"name BEGINSWITH[c] %@", @"i"];
        NSArray *appleDevices = [products filteredArrayUsingPredicate:startsWithi];
        NSLog(@"Starts with 'i': %@", [appleDevices valueForKey:@"name"]);
        
        // กรองด้วย CONTAINS
        NSPredicate *containsA = [NSPredicate predicateWithFormat:
                                   @"name CONTAINS[c] %@", @"a"];
        NSArray *withA = [products filteredArrayUsingPredicate:containsA];
        NSLog(@"Contains 'a': %@", [withA valueForKey:@"name"]);
        
        // กรองด้วย IN
        NSArray *validCategories = @[@"computer", @"phone"];
        NSPredicate *inCategories = [NSPredicate predicateWithFormat:
                                      @"category IN %@", validCategories];
        NSArray *filtered = [products filteredArrayUsingPredicate:inCategories];
        NSLog(@"In categories: %@", [filtered valueForKey:@"name"]);
        
        // กรอง NSMutableArray in-place
        NSMutableArray *mutableNumbers = [numbers mutableCopy];
        [mutableNumbers filterUsingPredicate:
         [NSPredicate predicateWithFormat:@"self > 5"]];
        NSLog(@"\nFiltered mutable: %@", mutableNumbers);
    }
    return 0;
}
```

---

## 23.7 Searching

```objc
#import <Foundation/Foundation.h>

int main(int argc, const char * argv[]) {
    @autoreleasepool {
        NSArray *fruits = @[@"apple", @"banana", @"cherry", @"apple", @"date"];
        
        // containsObject: - ตรวจสอบว่ามีหรือไม่
        NSLog(@"Contains 'apple': %@",
              [fruits containsObject:@"apple"] ? @"YES" : @"NO");
        NSLog(@"Contains 'grape': %@",
              [fruits containsObject:@"grape"] ? @"YES" : @"NO");
        
        // indexOfObject: - หา index แรก
        NSUInteger idx = [fruits indexOfObject:@"apple"];
        NSLog(@"First 'apple' at index: %lu", (unsigned long)idx);
        
        // indexOfObject:inRange: - หาใน range
        NSUInteger idx2 = [fruits indexOfObject:@"apple"
                                        inRange:NSMakeRange(1, fruits.count - 1)];
        NSLog(@"Second 'apple' at index: %lu", (unsigned long)idx2);
        
        // NSNotFound
        NSUInteger notFoundIdx = [fruits indexOfObject:@"mango"];
        if (notFoundIdx == NSNotFound) {
            NSLog(@"'mango' not found");
        }
        
        // indexOfObjectPassingTest: - ค้นหาด้วย block
        NSArray *numbers = @[@15, @42, @8, @37, @100, @3];
        
        NSUInteger firstBig = [numbers indexOfObjectPassingTest:
            ^BOOL(NSNumber *n, NSUInteger idx, BOOL *stop) {
                return [n integerValue] > 30;
            }];
        NSLog(@"First > 30 at index: %lu, value: %@",
              (unsigned long)firstBig, numbers[firstBig]);
        
        // indexesOfObjectsPassingTest: - หาทุก indexes ที่ผ่าน
        NSIndexSet *bigIndexes = [numbers indexesOfObjectsPassingTest:
            ^BOOL(NSNumber *n, NSUInteger idx, BOOL *stop) {
                return [n integerValue] > 30;
            }];
        
        NSLog(@"Indexes of > 30:");
        [bigIndexes enumerateIndexesUsingBlock:^(NSUInteger idx2, BOOL *stop) {
            NSLog(@"  [%lu] = %@", (unsigned long)idx2, numbers[idx2]);
        }];
        
        // indexOfObjectWithOptions:passingTest: - concurrent search
        NSUInteger anyBig = [numbers indexOfObjectWithOptions:NSEnumerationConcurrent
                                                  passingTest:
            ^BOOL(NSNumber *n, NSUInteger idx, BOOL *stop) {
                return [n integerValue] > 50;
            }];
        NSLog(@"Any > 50 at: %lu", (unsigned long)anyBig);
        
        // Binary search (สำหรับ sorted array)
        NSArray *sorted = @[@1, @5, @10, @15, @20, @25, @30];
        NSNumber *target = @15;
        
        NSUInteger binaryIdx = [sorted indexOfObject:target
                                       inSortedRange:NSMakeRange(0, sorted.count)
                                             options:NSBinarySearchingFirstEqual
                                     usingComparator:^NSComparisonResult(NSNumber *a, NSNumber *b) {
            return [a compare:b];
        }];
        NSLog(@"Binary search for %@: index %lu", target, (unsigned long)binaryIdx);
        
        // หาตำแหน่งที่จะ insert (maintain sort order)
        NSNumber *insertValue = @12;
        NSUInteger insertIdx = [sorted indexOfObject:insertValue
                                      inSortedRange:NSMakeRange(0, sorted.count)
                                            options:NSBinarySearchingInsertionIndex
                                    usingComparator:^NSComparisonResult(NSNumber *a, NSNumber *b) {
            return [a compare:b];
        }];
        NSLog(@"Insert %@ at index: %lu", insertValue, (unsigned long)insertIdx);
    }
    return 0;
}
```

---

## 23.8 Array Manipulation

```objc
#import <Foundation/Foundation.h>

int main(int argc, const char * argv[]) {
    @autoreleasepool {
        NSArray *arr = @[@1, @2, @3, @4, @5];
        
        // arrayByAddingObject: - immutable add
        NSArray *arr2 = [arr arrayByAddingObject:@6];
        NSLog(@"Added: %@", arr2); // arr ยังคงเดิม
        
        // arrayByAddingObjectsFromArray:
        NSArray *arr3 = [arr arrayByAddingObjectsFromArray:@[@6, @7, @8]];
        NSLog(@"Added array: %@", arr3);
        
        // subarrayWithRange:
        NSArray *sub = [arr subarrayWithRange:NSMakeRange(1, 3)];
        NSLog(@"Subarray [1..3]: %@", sub); // [2, 3, 4]
        
        // Reverse
        NSArray *reversed = [[arr reverseObjectEnumerator] allObjects];
        NSLog(@"Reversed: %@", reversed);
        
        // Unique (remove duplicates)
        NSArray *withDups = @[@1, @2, @2, @3, @1, @4, @3];
        NSOrderedSet *set = [NSOrderedSet orderedSetWithArray:withDups];
        NSArray *unique = [set array];
        NSLog(@"Unique: %@", unique);
        
        // Intersection (ตัดกัน)
        NSArray *setA = @[@1, @2, @3, @4, @5];
        NSArray *setB = @[@3, @4, @5, @6, @7];
        NSPredicate *inB = [NSPredicate predicateWithFormat:@"SELF IN %@", setB];
        NSArray *intersection = [setA filteredArrayUsingPredicate:inB];
        NSLog(@"Intersection: %@", intersection); // [3, 4, 5]
        
        // Union
        NSMutableSet *unionSet = [NSMutableSet setWithArray:setA];
        [unionSet addObjectsFromArray:setB];
        NSArray *union_ = [unionSet allObjects];
        NSLog(@"Union count: %lu", (unsigned long)union_.count);
        
        // Difference (ส่วนที่ไม่ตัดกัน)
        NSPredicate *notInB = [NSPredicate predicateWithFormat:@"NOT (SELF IN %@)", setB];
        NSArray *diff = [setA filteredArrayUsingPredicate:notInB];
        NSLog(@"A - B: %@", diff); // [1, 2]
        
        // Zip two arrays (combine corresponding elements)
        NSArray *names = @[@"Alice", @"Bob", @"Charlie"];
        NSArray *scores = @[@95, @87, @92];
        NSMutableArray *zipped = [NSMutableArray array];
        for (NSUInteger i = 0; i < MIN(names.count, scores.count); i++) {
            [zipped addObject:@{@"name": names[i], @"score": scores[i]}];
        }
        NSLog(@"Zipped: %@", zipped);
        
        // Flatten nested array
        NSArray *nested = @[@[@1, @2], @[@3, @4], @[@5, @6]];
        NSMutableArray *flat = [NSMutableArray array];
        for (NSArray *subArr in nested) {
            [flat addObjectsFromArray:subArr];
        }
        NSLog(@"Flat: %@", flat);
    }
    return 0;
}
```

---

## 23.9 NSMutableArray Operations

```objc
#import <Foundation/Foundation.h>

int main(int argc, const char * argv[]) {
    @autoreleasepool {
        NSMutableArray *arr = [NSMutableArray array];
        
        // เพิ่ม elements
        [arr addObject:@"first"];
        [arr addObject:@"second"];
        [arr addObject:@"third"];
        NSLog(@"After add: %@", arr);
        
        // แทรก element
        [arr insertObject:@"zero" atIndex:0];
        [arr insertObject:@"between" atIndex:2];
        NSLog(@"After insert: %@", arr);
        
        // ลบ element
        [arr removeObject:@"second"]; // ลบทุก occurrences
        [arr removeObjectAtIndex:0];   // ลบตาม index
        [arr removeLastObject];        // ลบตัวสุดท้าย
        NSLog(@"After remove: %@", arr);
        
        // ลบ range
        NSMutableArray *arr2 = [@[@1, @2, @3, @4, @5] mutableCopy];
        [arr2 removeObjectsInRange:NSMakeRange(1, 3)]; // ลบ index 1,2,3
        NSLog(@"After range remove: %@", arr2); // [1, 5]
        
        // ลบทั้งหมด
        NSMutableArray *arr3 = [@[@1, @2, @3] mutableCopy];
        [arr3 removeAllObjects];
        NSLog(@"After removeAll: %@, count: %lu", arr3, (unsigned long)arr3.count);
        
        // แทนที่ element
        NSMutableArray *arr4 = [@[@"a", @"b", @"c"] mutableCopy];
        [arr4 replaceObjectAtIndex:1 withObject:@"B"];
        NSLog(@"After replace: %@", arr4);
        
        // Exchange objects
        [arr4 exchangeObjectAtIndex:0 withObjectAtIndex:2];
        NSLog(@"After exchange: %@", arr4);
        
        // removeObjectsInArray: - ลบ elements ที่มีใน array อื่น
        NSMutableArray *arr5 = [@[@1, @2, @3, @4, @5] mutableCopy];
        [arr5 removeObjectsInArray:@[@2, @4]];
        NSLog(@"After removeInArray: %@", arr5); // [1, 3, 5]
        
        // addObjectsFromArray:
        NSMutableArray *arr6 = [@[@1, @2, @3] mutableCopy];
        [arr6 addObjectsFromArray:@[@4, @5, @6]];
        NSLog(@"After addObjects: %@", arr6);
        
        // setArray: - replace all contents
        NSMutableArray *arr7 = [@[@1, @2, @3] mutableCopy];
        [arr7 setArray:@[@10, @20, @30, @40]];
        NSLog(@"After setArray: %@", arr7);
        
        // Stack-like operations
        NSMutableArray *stack = [NSMutableArray array];
        [stack addObject:@"push1"]; // push
        [stack addObject:@"push2"];
        [stack addObject:@"push3"];
        
        NSString *top = stack.lastObject;
        [stack removeLastObject]; // pop
        NSLog(@"Popped: %@, Stack: %@", top, stack);
        
        // Queue-like operations
        NSMutableArray *queue = [NSMutableArray array];
        [queue addObject:@"enqueue1"]; // enqueue
        [queue addObject:@"enqueue2"];
        [queue addObject:@"enqueue3"];
        
        NSString *front = queue.firstObject;
        [queue removeObjectAtIndex:0]; // dequeue
        NSLog(@"Dequeued: %@, Queue: %@", front, queue);
    }
    return 0;
}
```

---

## 23.10 Arrays of Custom Objects

```objc
#import <Foundation/Foundation.h>

// Custom class
@interface Student : NSObject

@property (nonatomic, strong) NSString *name;
@property (nonatomic, assign) NSInteger grade;
@property (nonatomic, assign) double gpa;
@property (nonatomic, strong) NSString *major;

- (instancetype)initWithName:(NSString *)name
                       grade:(NSInteger)grade
                         gpa:(double)gpa
                       major:(NSString *)major;

@end

@implementation Student

- (instancetype)initWithName:(NSString *)name
                       grade:(NSInteger)grade
                         gpa:(double)gpa
                       major:(NSString *)major {
    self = [super init];
    if (self) {
        _name = name;
        _grade = grade;
        _gpa = gpa;
        _major = major;
    }
    return self;
}

- (NSString *)description {
    return [NSString stringWithFormat:@"%@ (Grade %ld, GPA: %.2f, %@)",
            _name, (long)_grade, _gpa, _major];
}

@end

int main(int argc, const char * argv[]) {
    @autoreleasepool {
        NSArray<Student *> *students = @[
            [[Student alloc] initWithName:@"Alice" grade:10 gpa:3.8 major:@"CS"],
            [[Student alloc] initWithName:@"Bob" grade:11 gpa:3.2 major:@"Math"],
            [[Student alloc] initWithName:@"Charlie" grade:10 gpa:3.9 major:@"CS"],
            [[Student alloc] initWithName:@"Dave" grade:12 gpa:3.5 major:@"Physics"],
            [[Student alloc] initWithName:@"Eve" grade:11 gpa:3.7 major:@"CS"],
        ];
        
        // Sort by GPA descending
        NSArray *byGPA = [students sortedArrayUsingComparator:
            ^NSComparisonResult(Student *a, Student *b) {
                if (a.gpa > b.gpa) return NSOrderedAscending;
                if (a.gpa < b.gpa) return NSOrderedDescending;
                return NSOrderedSame;
            }];
        
        NSLog(@"By GPA (desc):");
        for (Student *s in byGPA) {
            NSLog(@"  %@", s);
        }
        
        // Filter CS students
        NSPredicate *csPred = [NSPredicate predicateWithFormat:@"major == %@", @"CS"];
        NSArray *csStudents = [students filteredArrayUsingPredicate:csPred];
        NSLog(@"\nCS Students:");
        for (Student *s in csStudents) {
            NSLog(@"  %@", s);
        }
        
        // คำนวณ average GPA
        double totalGPA = 0;
        for (Student *s in students) {
            totalGPA += s.gpa;
        }
        double avgGPA = totalGPA / students.count;
        NSLog(@"\nAverage GPA: %.2f", avgGPA);
        
        // ใช้ KVC สำหรับ calculations
        NSNumber *gpaSum = [students valueForKeyPath:@"@sum.gpa"];
        NSNumber *gpaAvg = [students valueForKeyPath:@"@avg.gpa"];
        NSNumber *gpaMax = [students valueForKeyPath:@"@max.gpa"];
        NSLog(@"KVC - Sum: %@, Avg: %@, Max: %@", gpaSum, gpaAvg, gpaMax);
        
        // Group by major
        NSMutableDictionary *grouped = [NSMutableDictionary dictionary];
        for (Student *s in students) {
            if (!grouped[s.major]) {
                grouped[s.major] = [NSMutableArray array];
            }
            [grouped[s.major] addObject:s];
        }
        
        NSLog(@"\nGrouped by major:");
        for (NSString *major in grouped) {
            NSArray *group = grouped[major];
            NSLog(@"  %@ (%lu students):", major, (unsigned long)group.count);
            for (Student *s in group) {
                NSLog(@"    - %@", s.name);
            }
        }
        
        // NSSortDescriptor บน custom objects
        NSSortDescriptor *byGrade = [NSSortDescriptor sortDescriptorWithKey:@"grade"
                                                                  ascending:YES];
        NSSortDescriptor *byName = [NSSortDescriptor sortDescriptorWithKey:@"name"
                                                                 ascending:YES
                                                                  selector:@selector(localizedCompare:)];
        NSArray *sorted = [students sortedArrayUsingDescriptors:@[byGrade, byName]];
        NSLog(@"\nBy grade then name:");
        for (Student *s in sorted) {
            NSLog(@"  %@", s);
        }
    }
    return 0;
}
```

---

## 23.11 Deep Copy vs Shallow Copy

```objc
#import <Foundation/Foundation.h>

int main(int argc, const char * argv[]) {
    @autoreleasepool {
        // Shallow copy - สร้าง array ใหม่ แต่ elements ชี้ object เดิม
        NSMutableArray *inner = [NSMutableArray arrayWithObjects:@1, @2, @3, nil];
        NSArray *original = @[inner, @"hello"];
        
        NSArray *shallowCopy = [original copy]; // shallow
        NSMutableArray *shallowMutable = [original mutableCopy]; // shallow mutable
        
        // เพิ่ม element ใน inner ของ original
        [inner addObject:@4];
        
        // shallow copies จะเห็นการเปลี่ยนแปลงด้วย
        NSLog(@"Original inner: %@", original[0]);     // [1,2,3,4]
        NSLog(@"Shallow copy inner: %@", shallowCopy[0]); // [1,2,3,4] !!!
        
        // Deep copy - copy ทุก levels
        NSArray *original2 = @[
            @[@1, @2, @3],
            @[@4, @5, @6]
        ];
        
        // initWithArray:copyItems: - deep copy ระดับแรก
        NSArray *deepCopy = [[NSArray alloc] initWithArray:original2 copyItems:YES];
        
        // สร้าง deep copy แบบ manual ด้วย NSKeyedArchiver (full deep copy)
        NSData *archiveData = [NSKeyedArchiver archivedDataWithRootObject:original2
                                                    requiringSecureCoding:NO
                                                                    error:nil];
        NSArray *fullDeepCopy = [NSKeyedUnarchiver unarchivedObjectOfClass:[NSArray class]
                                                                  fromData:archiveData
                                                                     error:nil];
        
        NSLog(@"Original: %@", original2);
        NSLog(@"Deep copy: %@", deepCopy);
        NSLog(@"Full deep copy: %@", fullDeepCopy);
        
        // ตัวอย่างที่ชัดเจน
        NSMutableArray *mutableInner = [NSMutableArray arrayWithObjects:@"a", @"b", nil];
        NSMutableArray *container = [NSMutableArray arrayWithObject:mutableInner];
        
        // Shallow copy
        NSMutableArray *shallow = [container mutableCopy];
        [[shallow objectAtIndex:0] addObject:@"c"]; // แก้ไข inner
        
        NSLog(@"\nAfter modifying shallow copy's inner:");
        NSLog(@"Original: %@", container[0]); // a, b, c - ได้รับผลกระทบ!
        NSLog(@"Shallow: %@", shallow[0]);    // a, b, c
    }
    return 0;
}
```

---

## 23.12 Performance Considerations

```objc
#import <Foundation/Foundation.h>

int main(int argc, const char * argv[]) {
    @autoreleasepool {
        // 1. ใช้ capacity เมื่อรู้ขนาดล่วงหน้า
        NSMutableArray *optimized = [NSMutableArray arrayWithCapacity:1000];
        for (int i = 0; i < 1000; i++) {
            [optimized addObject:@(i)];
        }
        
        // 2. Fast enumeration เร็วกว่า indexing
        // ช้า:
        NSArray *arr = optimized;
        NSDate *start1 = [NSDate date];
        NSInteger sum1 = 0;
        for (NSUInteger i = 0; i < arr.count; i++) {
            sum1 += [arr[i] integerValue];
        }
        NSTimeInterval time1 = -[start1 timeIntervalSinceNow];
        
        // เร็วกว่า:
        NSDate *start2 = [NSDate date];
        NSInteger sum2 = 0;
        for (NSNumber *n in arr) {
            sum2 += [n integerValue];
        }
        NSTimeInterval time2 = -[start2 timeIntervalSinceNow];
        
        NSLog(@"Index: %.6fs, Fast enum: %.6fs", time1, time2);
        
        // 3. Binary search สำหรับ sorted arrays
        NSMutableArray *sorted = [NSMutableArray array];
        for (int i = 0; i < 10000; i++) {
            [sorted addObject:@(i * 2)]; // 0, 2, 4, 6, ...
        }
        
        NSNumber *searchValue = @9998;
        
        // Linear search
        NSDate *linearStart = [NSDate date];
        NSUInteger linearIdx = [sorted indexOfObject:searchValue];
        NSTimeInterval linearTime = -[linearStart timeIntervalSinceNow];
        
        // Binary search
        NSDate *binaryStart = [NSDate date];
        NSUInteger binaryIdx = [sorted indexOfObject:searchValue
                                       inSortedRange:NSMakeRange(0, sorted.count)
                                             options:NSBinarySearchingFirstEqual
                                     usingComparator:^NSComparisonResult(NSNumber *a, NSNumber *b) {
            return [a compare:b];
        }];
        NSTimeInterval binaryTime = -[binaryStart timeIntervalSinceNow];
        
        NSLog(@"Linear: %.6fs (idx=%lu), Binary: %.6fs (idx=%lu)",
              linearTime, (unsigned long)linearIdx,
              binaryTime, (unsigned long)binaryIdx);
        
        // 4. Use NSSet for membership testing (O(1) vs O(n))
        NSSet *setForLookup = [NSSet setWithArray:sorted];
        NSDate *setStart = [NSDate date];
        BOOL found = [setForLookup containsObject:@9998];
        NSTimeInterval setTime = -[setStart timeIntervalSinceNow];
        NSLog(@"NSSet contains: %@ in %.6fs", found ? @"YES" : @"NO", setTime);
        
        // 5. Copy semantics - ใช้ copy เมื่อจำเป็นเท่านั้น
        // Avoid unnecessary copies
        NSArray *immutable = @[@1, @2, @3];
        // ถ้าไม่ต้องการ modify ไม่ต้อง mutableCopy
        NSInteger total = 0;
        for (NSNumber *n in immutable) { // อ่านได้โดยตรง
            total += [n integerValue];
        }
    }
    return 0;
}
```

---

## 23.13 แบบฝึกหัด 10+ ข้อ

### แบบฝึกหัดที่ 1: Student Grade System

```objc
// สร้าง array ของ students และคำนวณสถิติต่างๆ

@interface StudentGrade : NSObject
@property NSString *name;
@property NSArray<NSNumber *> *scores;
- (double)average;
- (NSString *)letterGrade;
@end

@implementation StudentGrade
- (double)average {
    if (_scores.count == 0) return 0;
    double sum = [[_scores valueForKeyPath:@"@sum.self"] doubleValue];
    return sum / _scores.count;
}

- (NSString *)letterGrade {
    double avg = self.average;
    if (avg >= 90) return @"A";
    if (avg >= 80) return @"B";
    if (avg >= 70) return @"C";
    if (avg >= 60) return @"D";
    return @"F";
}
@end

// แบบฝึกหัด: สร้างระบบที่
// 1. สร้าง array ของ 10 students พร้อม scores หลายวิชา
// 2. หาค่าเฉลี่ยสูงสุดและต่ำสุด
// 3. กรองเฉพาะนักเรียนที่ได้ A
// 4. เรียงลำดับตาม average desc
// 5. แสดง top 3
```

### แบบฝึกหัดที่ 2: Inventory Management

```objc
// สร้างระบบ inventory
// - เพิ่ม/ลบ products
// - ค้นหาตาม category
// - เรียงตาม price
// - หาของที่ stock น้อยกว่า threshold
// - คำนวณมูลค่ารวม

@interface Product : NSObject
@property NSString *name;
@property NSString *category;
@property double price;
@property NSInteger stock;
@end

// ตัวอย่าง solution
NSMutableArray<Product *> *inventory = [NSMutableArray array];

// เพิ่ม products
// ...

// หา low stock
NSPredicate *lowStock = [NSPredicate predicateWithFormat:@"stock < %d", 10];
NSArray *lowStockItems = [inventory filteredArrayUsingPredicate:lowStock];

// คำนวณ total value
double totalValue = [[inventory valueForKeyPath:@"@sum.price"] doubleValue];
```

### แบบฝึกหัดที่ 3: Shopping Cart

```objc
// Implement shopping cart ด้วย NSMutableArray:
// - addItem:quantity:
// - removeItem:
// - updateQuantity:forItem:
// - calculateTotal
// - applyDiscount:
// - getItemsSortedByPrice

@interface CartItem : NSObject
@property NSString *name;
@property double price;
@property NSInteger quantity;
- (double)subtotal;
@end

@interface ShoppingCart : NSObject
- (void)addItem:(CartItem *)item;
- (void)removeItemNamed:(NSString *)name;
- (double)total;
- (NSArray<CartItem *> *)itemsSortedByPrice;
@end
```

### แบบฝึกหัดที่ 4: เฉลยตัวอย่างที่ 1

```objc
#import <Foundation/Foundation.h>

@interface StudentGrade : NSObject
@property (nonatomic, strong) NSString *name;
@property (nonatomic, strong) NSArray<NSNumber *> *scores;
- (instancetype)initWithName:(NSString *)name scores:(NSArray<NSNumber *> *)scores;
- (double)average;
- (NSString *)letterGrade;
@end

@implementation StudentGrade

- (instancetype)initWithName:(NSString *)name scores:(NSArray<NSNumber *> *)scores {
    self = [super init];
    if (self) { _name = name; _scores = scores; }
    return self;
}

- (double)average {
    if (_scores.count == 0) return 0;
    double sum = [[_scores valueForKeyPath:@"@sum.self"] doubleValue];
    return sum / _scores.count;
}

- (NSString *)letterGrade {
    double avg = [self average];
    if (avg >= 90) return @"A";
    if (avg >= 80) return @"B";
    if (avg >= 70) return @"C";
    if (avg >= 60) return @"D";
    return @"F";
}

- (NSString *)description {
    return [NSString stringWithFormat:@"%@ - Avg: %.1f (%@)",
            _name, [self average], [self letterGrade]];
}

@end

int main(int argc, const char * argv[]) {
    @autoreleasepool {
        NSArray<StudentGrade *> *students = @[
            [[StudentGrade alloc] initWithName:@"Alice" scores:@[@95, @88, @92]],
            [[StudentGrade alloc] initWithName:@"Bob" scores:@[@72, @68, @75]],
            [[StudentGrade alloc] initWithName:@"Charlie" scores:@[@85, @90, @88]],
            [[StudentGrade alloc] initWithName:@"Dave" scores:@[@55, @60, @58]],
            [[StudentGrade alloc] initWithName:@"Eve" scores:@[@98, @95, @97]],
        ];
        
        // หา max/min average
        StudentGrade *topStudent = [students objectAtIndex:0];
        StudentGrade *bottomStudent = [students objectAtIndex:0];
        
        for (StudentGrade *s in students) {
            if ([s average] > [topStudent average]) topStudent = s;
            if ([s average] < [bottomStudent average]) bottomStudent = s;
        }
        
        NSLog(@"Top: %@", topStudent);
        NSLog(@"Bottom: %@", bottomStudent);
        
        // กรอง grade A
        NSPredicate *gradeA = [NSPredicate predicateWithBlock:
            ^BOOL(StudentGrade *s, NSDictionary *b) {
                return [[s letterGrade] isEqualToString:@"A"];
            }];
        NSArray *aStudents = [students filteredArrayUsingPredicate:gradeA];
        NSLog(@"\nGrade A students:");
        for (StudentGrade *s in aStudents) {
            NSLog(@"  %@", s);
        }
        
        // เรียงลำดับ
        NSArray *sorted = [students sortedArrayUsingComparator:
            ^NSComparisonResult(StudentGrade *a, StudentGrade *b) {
                if ([a average] > [b average]) return NSOrderedAscending;
                if ([a average] < [b average]) return NSOrderedDescending;
                return NSOrderedSame;
            }];
        
        NSLog(@"\nRankings:");
        for (NSUInteger i = 0; i < MIN(3, sorted.count); i++) {
            NSLog(@"  #%lu: %@", (unsigned long)(i+1), sorted[i]);
        }
    }
    return 0;
}
```

### แบบฝึกหัดที่ 5: Pagination

```objc
// ทำ pagination สำหรับ array ขนาดใหญ่
NSArray *allItems = /* array ขนาดใหญ่ */;
NSInteger pageSize = 10;
NSInteger currentPage = 2; // 0-indexed

NSInteger startIndex = currentPage * pageSize;
NSInteger endIndex = MIN(startIndex + pageSize, (NSInteger)allItems.count);
NSInteger length = endIndex - startIndex;

if (startIndex < allItems.count) {
    NSArray *pageItems = [allItems subarrayWithRange:
                          NSMakeRange(startIndex, length)];
    NSLog(@"Page %ld: %@", (long)currentPage, pageItems);
}

NSInteger totalPages = (allItems.count + pageSize - 1) / pageSize;
NSLog(@"Total pages: %ld", (long)totalPages);
```

### แบบฝึกหัดที่ 6: Array Chunk

```objc
// แบ่ง array เป็น chunks ขนาดเท่ากัน
NSArray *arr = @[@1,@2,@3,@4,@5,@6,@7,@8,@9,@10];
NSInteger chunkSize = 3;
NSMutableArray *chunks = [NSMutableArray array];

for (NSUInteger i = 0; i < arr.count; i += chunkSize) {
    NSUInteger length = MIN(chunkSize, arr.count - i);
    NSArray *chunk = [arr subarrayWithRange:NSMakeRange(i, length)];
    [chunks addObject:chunk];
}

NSLog(@"Chunks:");
for (NSArray *chunk in chunks) {
    NSLog(@"  %@", chunk);
}
// [1,2,3], [4,5,6], [7,8,9], [10]
```

### แบบฝึกหัดที่ 7: Frequency Counter

```objc
// นับความถี่ของแต่ละ element
NSArray *words = @[@"apple", @"banana", @"apple", @"cherry", @"banana", @"apple"];

NSCountedSet *counted = [NSCountedSet setWithArray:words];
NSLog(@"Unique: %lu", (unsigned long)counted.count);
for (NSString *word in counted) {
    NSLog(@"'%@': %lu times", word, (unsigned long)[counted countForObject:word]);
}

// หาที่พบบ่อยสุด
NSString *mostFrequent = nil;
NSUInteger maxCount = 0;
for (NSString *word in counted) {
    NSUInteger cnt = [counted countForObject:word];
    if (cnt > maxCount) {
        maxCount = cnt;
        mostFrequent = word;
    }
}
NSLog(@"Most frequent: '%@' (%lu times)", mostFrequent, (unsigned long)maxCount);
```

### แบบฝึกหัดที่ 8: Merge Sort Implementation

```objc
NSArray *mergeSort(NSArray *arr) {
    if (arr.count <= 1) return arr;
    
    NSUInteger mid = arr.count / 2;
    NSArray *left = mergeSort([arr subarrayWithRange:NSMakeRange(0, mid)]);
    NSArray *right = mergeSort([arr subarrayWithRange:NSMakeRange(mid, arr.count - mid)]);
    
    // Merge
    NSMutableArray *merged = [NSMutableArray array];
    NSUInteger i = 0, j = 0;
    
    while (i < left.count && j < right.count) {
        if ([left[i] compare:right[j]] == NSOrderedAscending) {
            [merged addObject:left[i++]];
        } else {
            [merged addObject:right[j++]];
        }
    }
    
    while (i < left.count) [merged addObject:left[i++]];
    while (j < right.count) [merged addObject:right[j++]];
    
    return [merged copy];
}

// ใช้งาน
NSArray *unsorted = @[@3, @1, @4, @1, @5, @9, @2, @6, @5, @3];
NSArray *sorted2 = mergeSort(unsorted);
NSLog(@"Merge sorted: %@", sorted2);
```

### แบบฝึกหัดที่ 9: Pipeline Processing

```objc
// Chain operations บน array (functional style)
NSArray *data = @[@10, @-5, @3, @-8, @15, @2, @-1, @20];

// Pipeline: filter positives → double → sort → take top 3
NSArray *result = data;

// Step 1: filter positive
result = [result filteredArrayUsingPredicate:
          [NSPredicate predicateWithFormat:@"self > 0"]];

// Step 2: double values
result = [result valueForKeyPath:@"@unionOfObjects.self"]; // identity
NSMutableArray *doubled = [NSMutableArray array];
for (NSNumber *n in result) {
    [doubled addObject:@([n integerValue] * 2)];
}
result = [doubled copy];

// Step 3: sort descending
result = [result sortedArrayUsingComparator:
          ^NSComparisonResult(NSNumber *a, NSNumber *b) {
              return [b compare:a]; // reversed
          }];

// Step 4: take top 3
result = [result subarrayWithRange:NSMakeRange(0, MIN(3, result.count))];

NSLog(@"Pipeline result: %@", result); // [40, 30, 20]
```

### แบบฝึกหัดที่ 10: Matrix Operations

```objc
// Matrix operations ด้วย nested arrays
typedef NSArray<NSArray<NSNumber *> *> Matrix;

Matrix *createMatrix(NSInteger rows, NSInteger cols, NSInteger defaultValue) {
    NSMutableArray *matrix = [NSMutableArray array];
    for (NSInteger r = 0; r < rows; r++) {
        NSMutableArray *row = [NSMutableArray array];
        for (NSInteger c = 0; c < cols; c++) {
            [row addObject:@(defaultValue)];
        }
        [matrix addObject:[row copy]];
    }
    return [matrix copy];
}

Matrix *multiplyMatrix(Matrix *a, Matrix *b) {
    NSInteger rowsA = a.count;
    NSInteger colsA = a[0].count;
    NSInteger colsB = b[0].count;
    
    NSMutableArray *result = [NSMutableArray array];
    for (NSInteger i = 0; i < rowsA; i++) {
        NSMutableArray *row = [NSMutableArray array];
        for (NSInteger j = 0; j < colsB; j++) {
            NSInteger sum = 0;
            for (NSInteger k = 0; k < colsA; k++) {
                sum += [a[i][k] integerValue] * [b[k][j] integerValue];
            }
            [row addObject:@(sum)];
        }
        [result addObject:[row copy]];
    }
    return [result copy];
}

// ใช้งาน
Matrix *m1 = @[@[@1, @2], @[@3, @4]];
Matrix *m2 = @[@[@5, @6], @[@7, @8]];
Matrix *product = multiplyMatrix(m1, m2);
NSLog(@"Matrix product: %@", product);
// [[19, 22], [43, 50]]
```

---

## 23.14 Best Practices

### 1. ใช้ typed generics

```objc
// ดี - compiler ช่วย type checking
NSArray<NSString *> *names = @[@"Alice", @"Bob"];
for (NSString *name in names) {
    // compiler รู้ว่า name เป็น NSString
}

// ไม่ดี - ไม่มี type safety
NSArray *names2 = @[@"Alice", @"Bob"];
```

### 2. ตรวจสอบ bounds ก่อน access

```objc
// ดี
if (index < array.count) {
    id obj = array[index];
}

// หรือใช้ firstObject/lastObject
id first = array.firstObject; // ปลอดภัย
id last = array.lastObject;   // ปลอดภัย
```

### 3. Prefer immutable when possible

```objc
// ดี - ป้องกัน accidental modification
NSArray *list = [mutableList copy];

// ส่ง array ออกไปจาก method ควร return immutable
- (NSArray<NSString *> *)getAllNames {
    return [_mutableNames copy]; // return immutable copy
}
```

### 4. ใช้ NSSet สำหรับ unique elements

```objc
// แทนที่จะใช้ array เพื่อ uniqueness
NSMutableSet *uniqueItems = [NSMutableSet set];
for (id item in sourceArray) {
    [uniqueItems addObject:item]; // ไม่มี duplicates อัตโนมัติ
}
```

---

## สรุป

NSArray และ NSMutableArray เป็น collection ที่ทรงพลังใน Objective-C:

1. **สร้าง** ด้วย literal `@[]`, `arrayWithObjects:`, หรือ builder methods
2. **เข้าถึง** ด้วย subscript `arr[i]`, `objectAtIndex:`, `firstObject`/`lastObject`
3. **Enumerate** ด้วย for-in (เร็ว) หรือ `enumerateObjectsUsingBlock:` (มี index)
4. **Sort** ด้วย `NSSortDescriptor`, comparator block, หรือ selector
5. **Filter** ด้วย `NSPredicate`
6. **Search** ด้วย `containsObject:`, `indexOfObject:`, binary search
7. **Manipulation** ด้วย `subarrayWithRange:`, `arrayByAddingObject:`
8. **NSMutableArray** สำหรับ add, remove, replace, reorder

ในบทต่อไปเราจะเรียนรู้เกี่ยวกับ **NSDictionary และ NSMutableDictionary** ซึ่งเป็น key-value collection ที่สำคัญมาก
