# Part 25: NSSet และ NSOrderedSet

## บทนำ

ในบทที่ผ่านมาเราได้เรียนรู้เกี่ยวกับ NSArray และ NSDictionary ซึ่งเป็น Collection ที่ใช้บ่อยที่สุดใน Objective-C แต่ยังมี Collection อีกประเภทหนึ่งที่มีความสำคัญมากและมีคุณสมบัติพิเศษที่แตกต่างออกไป นั่นก็คือ **NSSet** ซึ่งเป็น Collection ที่เก็บค่าที่ไม่ซ้ำกัน (unique values)

---

## 25.1 Set คืออะไร?

### แนวคิดพื้นฐาน

**Set** หรือ **เซต** ในทางคณิตศาสตร์และโปรแกรมมิ่ง คือกลุ่มของสมาชิกที่มีคุณสมบัติดังนี้:

1. **ไม่มีค่าซ้ำ (Unique Values)** - แต่ละสมาชิกปรากฏได้เพียงครั้งเดียวเท่านั้น
2. **ไม่มีลำดับ (No Order)** - สมาชิกในเซตไม่มีตำแหน่งที่แน่นอน
3. **ตรวจสอบสมาชิกภาพได้เร็ว** - การตรวจสอบว่ามีสมาชิกอยู่ใน Set หรือไม่ทำได้เร็วมาก

### เปรียบเทียบกับ Array

| คุณสมบัติ | NSArray | NSSet |
|---------|---------|-------|
| มีลำดับ | ✅ ใช่ | ❌ ไม่มี |
| ค่าซ้ำได้ | ✅ ได้ | ❌ ไม่ได้ |
| เข้าถึงด้วย index | ✅ ได้ | ❌ ไม่ได้ |
| ตรวจสอบสมาชิก | O(n) ช้า | O(1) เร็ว |
| เหมาะกับ | ข้อมูลเรียงลำดับ | ข้อมูลไม่ซ้ำ |

### เมื่อไหรควรใช้ Set?

- เก็บรายชื่อที่ไม่ซ้ำกัน เช่น รายชื่อผู้ใช้ที่ online
- กรองข้อมูลซ้ำออกจาก Array
- ตรวจสอบว่ามีสมาชิกอยู่หรือไม่อย่างรวดเร็ว
- ทำ Set Operations เช่น หาสมาชิกร่วม (intersection)

---

## 25.2 NSSet การสร้างและใช้งาน

### การสร้าง NSSet

```objc
#import <Foundation/Foundation.h>

int main(int argc, const char * argv[]) {
    @autoreleasepool {
        
        // วิธีที่ 1: สร้าง NSSet ว่าง
        NSSet *emptySet = [NSSet set];
        NSLog(@"Empty set: %@", emptySet);
        
        // วิธีที่ 2: สร้างจาก objects
        NSSet *fruits = [NSSet setWithObjects:@"Apple", @"Banana", @"Orange", nil];
        NSLog(@"Fruits: %@", fruits);
        
        // วิธีที่ 3: สร้างจาก array
        NSArray *colorArray = @[@"Red", @"Green", @"Blue", @"Red", @"Green"];
        NSSet *colorSet = [NSSet setWithArray:colorArray];
        NSLog(@"Colors (unique): %@", colorSet);
        NSLog(@"Array count: %lu, Set count: %lu", 
              (unsigned long)colorArray.count, 
              (unsigned long)colorSet.count);
        
        // วิธีที่ 4: สร้างจาก Set อื่น
        NSSet *copySet = [NSSet setWithSet:fruits];
        NSLog(@"Copy of fruits: %@", copySet);
        
        // วิธีที่ 5: สร้างด้วย object เดียว
        NSSet *singleSet = [NSSet setWithObject:@"OnlyOne"];
        NSLog(@"Single: %@", singleSet);
        
    }
    return 0;
}
```

**ผลลัพธ์ (ลำดับอาจแตกต่างกัน):**
```
Empty set: {(
)}
Fruits: {(
    Orange,
    Apple,
    Banana
)}
Colors (unique): {(
    Blue,
    Red,
    Green
)}
Array count: 5, Set count: 3
```

### การตรวจสอบสมาชิก

```objc
#import <Foundation/Foundation.h>

int main(int argc, const char * argv[]) {
    @autoreleasepool {
        
        NSSet *cities = [NSSet setWithObjects:
                         @"Bangkok", @"Tokyo", @"Paris", @"London", @"New York", nil];
        
        // ตรวจสอบว่ามีสมาชิกอยู่หรือไม่
        if ([cities containsObject:@"Tokyo"]) {
            NSLog(@"Tokyo is in the set");
        }
        
        if (![cities containsObject:@"Seoul"]) {
            NSLog(@"Seoul is NOT in the set");
        }
        
        // นับจำนวนสมาชิก
        NSLog(@"Number of cities: %lu", (unsigned long)cities.count);
        
        // ตรวจสอบว่า set ว่างหรือไม่
        NSLog(@"Is empty: %@", cities.count == 0 ? @"Yes" : @"No");
        
        // หาสมาชิกที่ตรงกับ condition
        NSSet *longNames = [cities objectsPassingTest:^BOOL(id obj, BOOL *stop) {
            NSString *city = (NSString *)obj;
            return city.length > 5;
        }];
        NSLog(@"Cities with long names: %@", longNames);
        
    }
    return 0;
}
```

### การวนซ้ำใน NSSet

```objc
#import <Foundation/Foundation.h>

int main(int argc, const char * argv[]) {
    @autoreleasepool {
        
        NSSet *numbers = [NSSet setWithObjects:
                          @10, @20, @30, @40, @50, nil];
        
        // วิธีที่ 1: Fast Enumeration (for-in)
        NSLog(@"Fast Enumeration:");
        for (NSNumber *num in numbers) {
            NSLog(@"  %@", num);
        }
        
        // วิธีที่ 2: enumerateObjectsUsingBlock
        NSLog(@"\nBlock Enumeration:");
        [numbers enumerateObjectsUsingBlock:^(id obj, BOOL *stop) {
            NSNumber *num = (NSNumber *)obj;
            NSLog(@"  %@", num);
            
            // หยุดเมื่อพบค่า 30
            if ([num intValue] == 30) {
                *stop = YES;
            }
        }];
        
        // วิธีที่ 3: NSEnumerator
        NSLog(@"\nNSEnumerator:");
        NSEnumerator *enumerator = [numbers objectEnumerator];
        id object;
        while ((object = [enumerator nextObject])) {
            NSLog(@"  %@", object);
        }
        
        // แปลงเป็น Array เพื่อเรียงลำดับ
        NSArray *sortedArray = [[numbers allObjects] 
                                sortedArrayUsingSelector:@selector(compare:)];
        NSLog(@"\nSorted: %@", sortedArray);
        
    }
    return 0;
}
```

### การดึงข้อมูลจาก NSSet

```objc
#import <Foundation/Foundation.h>

int main(int argc, const char * argv[]) {
    @autoreleasepool {
        
        NSSet *animals = [NSSet setWithObjects:
                          @"Cat", @"Dog", @"Bird", @"Fish", @"Rabbit", nil];
        
        // ดึง object ใด ๆ หนึ่งตัว (ไม่แน่นอนว่าได้ตัวไหน)
        id anyObject = [animals anyObject];
        NSLog(@"Any object: %@", anyObject);
        
        // แปลงเป็น Array
        NSArray *array = [animals allObjects];
        NSLog(@"As array: %@", array);
        
        // กรองด้วย predicate
        NSPredicate *pred = [NSPredicate predicateWithFormat:@"length == 3"];
        NSSet *shortNames = [animals filteredSetUsingPredicate:pred];
        NSLog(@"Short names (3 chars): %@", shortNames);
        
        // Map ด้วย valueForKey
        NSSet *uppercased = [NSSet setWithArray:
                             [[animals allObjects] valueForKey:@"uppercaseString"]];
        NSLog(@"Uppercased: %@", uppercased);
        
    }
    return 0;
}
```

---

## 25.3 NSMutableSet - Set ที่แก้ไขได้

NSMutableSet คือ subclass ของ NSSet ที่สามารถเพิ่ม ลบ สมาชิกได้หลังจากสร้างแล้ว

### การสร้างและแก้ไข NSMutableSet

```objc
#import <Foundation/Foundation.h>

int main(int argc, const char * argv[]) {
    @autoreleasepool {
        
        // สร้าง NSMutableSet
        NSMutableSet *mutableSet = [NSMutableSet set];
        
        // เพิ่มสมาชิก
        [mutableSet addObject:@"Apple"];
        [mutableSet addObject:@"Banana"];
        [mutableSet addObject:@"Cherry"];
        NSLog(@"After adding: %@", mutableSet);
        
        // เพิ่มสมาชิกซ้ำ - จะไม่มีผล
        [mutableSet addObject:@"Apple"];
        NSLog(@"After adding Apple again: %@", mutableSet); // ยังคงมี 3 ตัว
        
        // ลบสมาชิก
        [mutableSet removeObject:@"Banana"];
        NSLog(@"After removing Banana: %@", mutableSet);
        
        // ลบทั้งหมด
        // [mutableSet removeAllObjects];
        
        // เพิ่มจาก Array
        NSArray *moreFruits = @[@"Mango", @"Grape", @"Apple"]; // Apple ซ้ำ
        [mutableSet addObjectsFromArray:moreFruits];
        NSLog(@"After adding from array: %@", mutableSet);
        NSLog(@"Count: %lu", (unsigned long)mutableSet.count);
        
    }
    return 0;
}
```

### การใช้ NSMutableSet ในทางปฏิบัติ

```objc
#import <Foundation/Foundation.h>

int main(int argc, const char * argv[]) {
    @autoreleasepool {
        
        // ตัวอย่าง: ติดตามผู้ใช้ที่ online
        NSMutableSet *onlineUsers = [NSMutableSet set];
        
        // ผู้ใช้ login
        [onlineUsers addObject:@"Alice"];
        [onlineUsers addObject:@"Bob"];
        [onlineUsers addObject:@"Charlie"];
        NSLog(@"Online: %@", onlineUsers);
        
        // ผู้ใช้ login ซ้ำ (ไม่มีผล - ยังคงมีคนเดิม)
        [onlineUsers addObject:@"Alice"];
        NSLog(@"After Alice login again: %lu users online", 
              (unsigned long)onlineUsers.count);
        
        // ผู้ใช้ logout
        [onlineUsers removeObject:@"Bob"];
        NSLog(@"After Bob logout: %@", onlineUsers);
        
        // ตรวจสอบ
        if ([onlineUsers containsObject:@"Charlie"]) {
            NSLog(@"Charlie is online");
        }
        
        // ตัวอย่าง: กรอง unique tags
        NSArray *rawTags = @[@"ios", @"swift", @"ios", @"objc", @"swift", @"mobile"];
        NSMutableSet *uniqueTags = [NSMutableSet setWithArray:rawTags];
        NSLog(@"\nRaw tags count: %lu", (unsigned long)rawTags.count);
        NSLog(@"Unique tags count: %lu", (unsigned long)uniqueTags.count);
        NSLog(@"Unique tags: %@", uniqueTags);
        
    }
    return 0;
}
```

---

## 25.4 Set Operations: Union, Intersection, Minus

นี่คือความสามารถที่ทรงพลังที่สุดของ NSMutableSet - การทำ Set Operations ตามหลักทฤษฎีเซต

### Union (สหภาพ) - รวมสมาชิกทั้งหมด

```objc
#import <Foundation/Foundation.h>

int main(int argc, const char * argv[]) {
    @autoreleasepool {
        
        NSSet *setA = [NSSet setWithObjects:@1, @2, @3, @4, @5, nil];
        NSSet *setB = [NSSet setWithObjects:@4, @5, @6, @7, @8, nil];
        
        // Union: A ∪ B
        NSMutableSet *unionSet = [NSMutableSet setWithSet:setA];
        [unionSet unionSet:setB];
        NSLog(@"A = %@", [setA.allObjects sortedArrayUsingSelector:@selector(compare:)]);
        NSLog(@"B = %@", [setB.allObjects sortedArrayUsingSelector:@selector(compare:)]);
        NSLog(@"A ∪ B = %@", [unionSet.allObjects sortedArrayUsingSelector:@selector(compare:)]);
        
    }
    return 0;
}
```

**ผลลัพธ์:**
```
A = (1, 2, 3, 4, 5)
B = (4, 5, 6, 7, 8)
A ∪ B = (1, 2, 3, 4, 5, 6, 7, 8)
```

### Intersection (จุดร่วม) - สมาชิกที่มีทั้งในสองเซต

```objc
#import <Foundation/Foundation.h>

int main(int argc, const char * argv[]) {
    @autoreleasepool {
        
        NSSet *programmersA = [NSSet setWithObjects:
                               @"Alice", @"Bob", @"Charlie", @"David", nil];
        NSSet *programmersB = [NSSet setWithObjects:
                               @"Bob", @"David", @"Eve", @"Frank", nil];
        
        // Intersection: A ∩ B (คนที่อยู่ในทั้งสองทีม)
        NSMutableSet *intersection = [NSMutableSet setWithSet:programmersA];
        [intersection intersectSet:programmersB];
        NSLog(@"Team A: %@", [programmersA.allObjects 
                              sortedArrayUsingSelector:@selector(compare:)]);
        NSLog(@"Team B: %@", [programmersB.allObjects 
                              sortedArrayUsingSelector:@selector(compare:)]);
        NSLog(@"In both teams: %@", [intersection.allObjects 
                                     sortedArrayUsingSelector:@selector(compare:)]);
        
        // ตรวจสอบว่ามีจุดร่วมหรือไม่
        if ([programmersA intersectsSet:programmersB]) {
            NSLog(@"Sets have common members");
        }
        
    }
    return 0;
}
```

**ผลลัพธ์:**
```
Team A: (Alice, Bob, Charlie, David)
Team B: (Bob, David, Eve, Frank)
In both teams: (Bob, David)
Sets have common members
```

### Minus (ผลต่าง) - สมาชิกที่อยู่ใน A แต่ไม่อยู่ใน B

```objc
#import <Foundation/Foundation.h>

int main(int argc, const char * argv[]) {
    @autoreleasepool {
        
        NSSet *allStudents = [NSSet setWithObjects:
                              @"Tom", @"Jerry", @"Spike", @"Tyke", @"Droopy", nil];
        NSSet *passedStudents = [NSSet setWithObjects:
                                 @"Tom", @"Spike", @"Droopy", nil];
        
        // Minus: ผู้ที่ยังไม่ผ่าน (allStudents - passedStudents)
        NSMutableSet *failedStudents = [NSMutableSet setWithSet:allStudents];
        [failedStudents minusSet:passedStudents];
        
        NSLog(@"All students: %@", [allStudents.allObjects 
                                    sortedArrayUsingSelector:@selector(compare:)]);
        NSLog(@"Passed: %@", [passedStudents.allObjects 
                              sortedArrayUsingSelector:@selector(compare:)]);
        NSLog(@"Failed: %@", [failedStudents.allObjects 
                              sortedArrayUsingSelector:@selector(compare:)]);
        
    }
    return 0;
}
```

**ผลลัพธ์:**
```
All students: (Droopy, Jerry, Spike, Tom, Tyke)
Passed: (Droopy, Spike, Tom)
Failed: (Jerry, Tyke)
```

### ตัวอย่างการใช้งาน Set Operations จริง

```objc
#import <Foundation/Foundation.h>

int main(int argc, const char * argv[]) {
    @autoreleasepool {
        
        // ตัวอย่าง: ระบบ permissions
        NSSet *adminPermissions = [NSSet setWithObjects:
            @"read", @"write", @"delete", @"manage_users", @"view_reports", nil];
        NSSet *editorPermissions = [NSSet setWithObjects:
            @"read", @"write", @"view_reports", nil];
        NSSet *userPermissions = [NSSet setWithObjects:
            @"read", nil];
        
        // Admin-only permissions
        NSMutableSet *adminOnly = [NSMutableSet setWithSet:adminPermissions];
        [adminOnly minusSet:editorPermissions];
        NSLog(@"Admin-only perms: %@", adminOnly.allObjects);
        
        // Permissions that both admin and editor share
        NSMutableSet *shared = [NSMutableSet setWithSet:adminPermissions];
        [shared intersectSet:editorPermissions];
        NSLog(@"Admin & Editor shared: %@", shared.allObjects);
        
        // All unique permissions in the system
        NSMutableSet *allPerms = [NSMutableSet setWithSet:adminPermissions];
        [allPerms unionSet:editorPermissions];
        [allPerms unionSet:userPermissions];
        NSLog(@"All permissions: %lu unique", (unsigned long)allPerms.count);
        
        // ตรวจสอบ subset
        if ([editorPermissions isSubsetOfSet:adminPermissions]) {
            NSLog(@"Editor permissions are a subset of Admin permissions");
        }
        
    }
    return 0;
}
```

### การตรวจสอบความสัมพันธ์ระหว่าง Sets

```objc
#import <Foundation/Foundation.h>

int main(int argc, const char * argv[]) {
    @autoreleasepool {
        
        NSSet *setA = [NSSet setWithObjects:@1, @2, @3, nil];
        NSSet *setB = [NSSet setWithObjects:@1, @2, @3, @4, @5, nil];
        NSSet *setC = [NSSet setWithObjects:@6, @7, @8, nil];
        
        // ตรวจสอบ subset (A เป็น subset ของ B หรือไม่)
        BOOL isSubset = [setA isSubsetOfSet:setB];
        NSLog(@"A is subset of B: %@", isSubset ? @"YES" : @"NO");
        
        // ตรวจสอบ intersection
        BOOL hasCommon = [setA intersectsSet:setB];
        NSLog(@"A intersects B: %@", hasCommon ? @"YES" : @"NO");
        
        // A และ C ไม่มีจุดร่วม
        BOOL noCommon = [setA intersectsSet:setC];
        NSLog(@"A intersects C: %@", noCommon ? @"YES" : @"NO");
        
        // ตรวจสอบความเท่ากัน
        NSSet *setD = [NSSet setWithObjects:@1, @2, @3, nil];
        BOOL isEqual = [setA isEqualToSet:setD];
        NSLog(@"A equals D: %@", isEqual ? @"YES" : @"NO");
        
    }
    return 0;
}
```

---

## 25.5 NSSet vs NSArray เปรียบเทียบอย่างละเอียด

### ประสิทธิภาพในการค้นหา

```objc
#import <Foundation/Foundation.h>

int main(int argc, const char * argv[]) {
    @autoreleasepool {
        
        // สร้างข้อมูล 1000 items
        NSMutableArray *mutableArray = [NSMutableArray array];
        NSMutableSet *mutableSet = [NSMutableSet set];
        
        for (int i = 0; i < 1000; i++) {
            NSString *item = [NSString stringWithFormat:@"item%d", i];
            [mutableArray addObject:item];
            [mutableSet addObject:item];
        }
        
        NSArray *largeArray = [mutableArray copy];
        NSSet *largeSet = [mutableSet copy];
        
        // ทดสอบการค้นหา
        NSString *target = @"item999"; // สมาชิกตัวสุดท้าย
        
        // Array - ต้องค้นทีละตัว O(n)
        NSDate *startArray = [NSDate date];
        for (int i = 0; i < 10000; i++) {
            [largeArray containsObject:target];
        }
        NSTimeInterval arrayTime = [[NSDate date] timeIntervalSinceDate:startArray];
        
        // Set - ใช้ hash ค้นหา O(1)
        NSDate *startSet = [NSDate date];
        for (int i = 0; i < 10000; i++) {
            [largeSet containsObject:target];
        }
        NSTimeInterval setTime = [[NSDate date] timeIntervalSinceDate:startSet];
        
        NSLog(@"Array search time: %.4f seconds", arrayTime);
        NSLog(@"Set search time: %.4f seconds", setTime);
        NSLog(@"Set is %.1fx faster", arrayTime / setTime);
        
    }
    return 0;
}
```

### เมื่อไหรควรใช้อะไร

```objc
#import <Foundation/Foundation.h>

int main(int argc, const char * argv[]) {
    @autoreleasepool {
        
        // ใช้ NSArray เมื่อ: ต้องการลำดับ, เข้าถึงด้วย index, ค่าซ้ำได้
        NSArray *playlist = @[@"Song1", @"Song2", @"Song3", @"Song1"]; // ซ้ำได้
        NSLog(@"Playlist item 2: %@", playlist[1]);
        NSLog(@"Playlist has duplicates: %lu items", (unsigned long)playlist.count);
        
        // ใช้ NSSet เมื่อ: ต้องการ unique values, ตรวจสอบสมาชิกเร็ว
        NSMutableSet *visitedURLs = [NSMutableSet set];
        NSArray *urls = @[@"google.com", @"apple.com", @"google.com", @"github.com"];
        for (NSString *url in urls) {
            if (![visitedURLs containsObject:url]) {
                NSLog(@"Visiting: %@", url);
                [visitedURLs addObject:url];
            } else {
                NSLog(@"Already visited: %@", url);
            }
        }
        
        // แปลงระหว่างกัน
        NSArray *uniqueArray = [visitedURLs allObjects];
        NSSet *urlSet = [NSSet setWithArray:playlist]; // กรอง duplicates
        NSLog(@"\nPlaylist unique songs: %lu", (unsigned long)urlSet.count);
        
    }
    return 0;
}
```

---

## 25.6 NSCountedSet - Set ที่นับจำนวน

NSCountedSet เป็น subclass ของ NSMutableSet ที่นับจำนวนครั้งที่ object ถูกเพิ่มเข้ามา (แม้จะเก็บ object ไว้แค่หนึ่งตัว)

```objc
#import <Foundation/Foundation.h>

int main(int argc, const char * argv[]) {
    @autoreleasepool {
        
        // สร้าง NSCountedSet
        NSCountedSet *countedSet = [NSCountedSet set];
        
        // เพิ่มข้อมูล
        [countedSet addObject:@"Apple"];
        [countedSet addObject:@"Apple"];
        [countedSet addObject:@"Apple"];
        [countedSet addObject:@"Banana"];
        [countedSet addObject:@"Banana"];
        [countedSet addObject:@"Cherry"];
        
        // นับจำนวน
        NSLog(@"Apple count: %lu", (unsigned long)[countedSet countForObject:@"Apple"]);
        NSLog(@"Banana count: %lu", (unsigned long)[countedSet countForObject:@"Banana"]);
        NSLog(@"Cherry count: %lu", (unsigned long)[countedSet countForObject:@"Cherry"]);
        NSLog(@"Grape count: %lu", (unsigned long)[countedSet countForObject:@"Grape"]);
        
        // จำนวน unique objects
        NSLog(@"Unique objects: %lu", (unsigned long)countedSet.count);
        
        // ลบ object (ลดจำนวนครั้งลง 1)
        [countedSet removeObject:@"Apple"];
        NSLog(@"Apple count after remove: %lu", 
              (unsigned long)[countedSet countForObject:@"Apple"]);
        
    }
    return 0;
}
```

### ตัวอย่างใช้งานจริง: นับคำในข้อความ

```objc
#import <Foundation/Foundation.h>

int main(int argc, const char * argv[]) {
    @autoreleasepool {
        
        NSString *text = @"the quick brown fox jumps over the lazy dog the fox";
        NSArray *words = [text componentsSeparatedByString:@" "];
        
        NSCountedSet *wordCount = [NSCountedSet setWithArray:words];
        
        NSLog(@"Word frequencies:");
        // เรียงตามตัวอักษร
        NSArray *sortedWords = [[wordCount allObjects] 
                                sortedArrayUsingSelector:@selector(compare:)];
        for (NSString *word in sortedWords) {
            NSLog(@"  '%@': %lu times", word, 
                  (unsigned long)[wordCount countForObject:word]);
        }
        
        // หาคำที่ใช้บ่อยที่สุด
        NSString *mostFrequent = nil;
        NSUInteger maxCount = 0;
        for (NSString *word in wordCount) {
            NSUInteger count = [wordCount countForObject:word];
            if (count > maxCount) {
                maxCount = count;
                mostFrequent = word;
            }
        }
        NSLog(@"\nMost frequent word: '%@' (%lu times)", mostFrequent, 
              (unsigned long)maxCount);
        
    }
    return 0;
}
```

**ผลลัพธ์:**
```
Word frequencies:
  'brown': 1 times
  'dog': 1 times
  'fox': 2 times
  'jumps': 1 times
  'lazy': 1 times
  'over': 1 times
  'quick': 1 times
  'the': 3 times

Most frequent word: 'the' (3 times)
```

---

## 25.7 NSOrderedSet - Set ที่มีลำดับ

NSOrderedSet รวมคุณสมบัติของทั้ง NSArray (มีลำดับ, เข้าถึงด้วย index) และ NSSet (ไม่มีซ้ำ, ค้นหาเร็ว) เข้าด้วยกัน

### การสร้างและใช้งาน NSOrderedSet

```objc
#import <Foundation/Foundation.h>

int main(int argc, const char * argv[]) {
    @autoreleasepool {
        
        // สร้าง NSOrderedSet
        NSOrderedSet *orderedSet = [NSOrderedSet orderedSetWithObjects:
                                    @"First", @"Second", @"Third", @"First", // First ซ้ำ
                                    nil];
        
        NSLog(@"OrderedSet: %@", orderedSet);
        NSLog(@"Count: %lu", (unsigned long)orderedSet.count);
        
        // เข้าถึงด้วย index (เหมือน Array)
        NSLog(@"Index 0: %@", [orderedSet objectAtIndex:0]);
        NSLog(@"Index 1: %@", [orderedSet objectAtIndex:1]);
        
        // หา index ของ object
        NSUInteger index = [orderedSet indexOfObject:@"Second"];
        NSLog(@"Index of 'Second': %lu", (unsigned long)index);
        
        // ตรวจสอบสมาชิก (เหมือน Set)
        BOOL contains = [orderedSet containsObject:@"Third"];
        NSLog(@"Contains 'Third': %@", contains ? @"YES" : @"NO");
        
        // first และ last object
        NSLog(@"First: %@", orderedSet.firstObject);
        NSLog(@"Last: %@", orderedSet.lastObject);
        
        // สร้างจาก Array
        NSArray *array = @[@"B", @"A", @"C", @"A", @"B"];
        NSOrderedSet *fromArray = [NSOrderedSet orderedSetWithArray:array];
        NSLog(@"\nFrom array (preserving order): %@", fromArray);
        
    }
    return 0;
}
```

**ผลลัพธ์:**
```
OrderedSet: {(
    First,
    Second,
    Third
)}
Count: 3
Index 0: First
Index of 'Second': 1
Contains 'Third': YES
First: First
Last: Third
From array (preserving order): {(B, A, C)}
```

### การแปลงระหว่าง NSOrderedSet, NSSet, NSArray

```objc
#import <Foundation/Foundation.h>

int main(int argc, const char * argv[]) {
    @autoreleasepool {
        
        NSOrderedSet *orderedSet = [NSOrderedSet orderedSetWithObjects:
                                    @"C", @"A", @"B", nil];
        
        // แปลงเป็น NSArray
        NSArray *array = [orderedSet array];
        NSLog(@"As Array: %@", array);
        
        // แปลงเป็น NSSet (สูญเสียลำดับ)
        NSSet *set = [orderedSet set];
        NSLog(@"As Set: %@", set);
        
        // สร้าง NSOrderedSet จาก NSSet (ลำดับไม่แน่นอน)
        NSOrderedSet *fromSet = [NSOrderedSet orderedSetWithSet:set];
        NSLog(@"From Set: %@", fromSet);
        
        // สร้าง NSOrderedSet จาก NSArray (กรอง duplicates, รักษาลำดับ)
        NSArray *withDups = @[@"Z", @"A", @"Z", @"B", @"A"];
        NSOrderedSet *uniqueOrdered = [NSOrderedSet orderedSetWithArray:withDups];
        NSLog(@"Unique ordered from array: %@", uniqueOrdered);
        
    }
    return 0;
}
```

---

## 25.8 NSMutableOrderedSet - OrderedSet ที่แก้ไขได้

```objc
#import <Foundation/Foundation.h>

int main(int argc, const char * argv[]) {
    @autoreleasepool {
        
        // สร้าง NSMutableOrderedSet
        NSMutableOrderedSet *tasks = [NSMutableOrderedSet orderedSet];
        
        // เพิ่ม tasks (รักษาลำดับ)
        [tasks addObject:@"Design UI"];
        [tasks addObject:@"Write Code"];
        [tasks addObject:@"Test"];
        [tasks addObject:@"Deploy"];
        [tasks addObject:@"Design UI"]; // ซ้ำ - จะไม่เพิ่ม
        
        NSLog(@"Tasks: %@", tasks);
        NSLog(@"Count: %lu", (unsigned long)tasks.count);
        
        // แทรกที่ตำแหน่งเฉพาะ
        [tasks insertObject:@"Review" atIndex:2];
        NSLog(@"After insert at index 2: %@", tasks);
        
        // ลบที่ตำแหน่งเฉพาะ
        [tasks removeObjectAtIndex:0];
        NSLog(@"After remove index 0: %@", tasks);
        
        // แทนที่ object
        [tasks replaceObjectAtIndex:0 withObject:@"Plan"];
        NSLog(@"After replace: %@", tasks);
        
        // ย้าย object
        [tasks moveObjectsAtIndexes:[NSIndexSet indexSetWithIndex:0] toIndex:2];
        NSLog(@"After move: %@", tasks);
        
        // เรียงลำดับ
        [tasks sortUsingComparator:^NSComparisonResult(id obj1, id obj2) {
            return [obj1 compare:obj2];
        }];
        NSLog(@"Sorted: %@", tasks);
        
    }
    return 0;
}
```

### Set Operations บน NSMutableOrderedSet

```objc
#import <Foundation/Foundation.h>

int main(int argc, const char * argv[]) {
    @autoreleasepool {
        
        NSMutableOrderedSet *setA = [NSMutableOrderedSet 
                                     orderedSetWithObjects:@1, @2, @3, @4, @5, nil];
        NSOrderedSet *setB = [NSOrderedSet 
                              orderedSetWithObjects:@4, @5, @6, @7, nil];
        
        // Union - เพิ่มทั้งหมดจาก B ที่ไม่มีใน A
        NSMutableOrderedSet *unionSet = [setA mutableCopy];
        [unionSet unionOrderedSet:setB];
        NSLog(@"Union: %@", unionSet);
        
        // Intersection - คงเฉพาะที่มีทั้งใน A และ B
        NSMutableOrderedSet *intersect = [setA mutableCopy];
        [intersect intersectOrderedSet:setB];
        NSLog(@"Intersection: %@", intersect);
        
        // Minus - ลบสมาชิกที่อยู่ใน B ออกจาก A
        NSMutableOrderedSet *minus = [setA mutableCopy];
        [minus minusOrderedSet:setB];
        NSLog(@"Minus: %@", minus);
        
    }
    return 0;
}
```

---

## 25.9 กรณีการใช้งานจริง (Practical Use Cases)

### Use Case 1: ระบบ Tag/Category

```objc
#import <Foundation/Foundation.h>

@interface Article : NSObject
@property (nonatomic, strong) NSString *title;
@property (nonatomic, strong) NSSet *tags;
- (instancetype)initWithTitle:(NSString *)title tags:(NSArray *)tags;
@end

@implementation Article
- (instancetype)initWithTitle:(NSString *)title tags:(NSArray *)tags {
    self = [super init];
    if (self) {
        _title = title;
        _tags = [NSSet setWithArray:tags];
    }
    return self;
}
@end

int main(int argc, const char * argv[]) {
    @autoreleasepool {
        
        Article *article1 = [[Article alloc] initWithTitle:@"Swift Tutorial"
                             tags:@[@"ios", @"swift", @"mobile"]];
        Article *article2 = [[Article alloc] initWithTitle:@"Objective-C Guide"
                             tags:@[@"ios", @"objc", @"mobile"]];
        Article *article3 = [[Article alloc] initWithTitle:@"Android Basics"
                             tags:@[@"android", @"mobile", @"java"]];
        
        NSArray *articles = @[article1, article2, article3];
        
        // หา articles ที่มี tag "ios"
        NSPredicate *iosPred = [NSPredicate predicateWithBlock:
            ^BOOL(Article *article, NSDictionary *bindings) {
                return [article.tags containsObject:@"ios"];
            }];
        NSArray *iosArticles = [articles filteredArrayUsingPredicate:iosPred];
        NSLog(@"iOS articles:");
        for (Article *a in iosArticles) {
            NSLog(@"  - %@", a.title);
        }
        
        // รวม tags ทั้งหมด
        NSMutableSet *allTags = [NSMutableSet set];
        for (Article *a in articles) {
            [allTags unionSet:a.tags];
        }
        NSLog(@"\nAll unique tags: %lu", (unsigned long)allTags.count);
        NSLog(@"%@", [allTags.allObjects sortedArrayUsingSelector:@selector(compare:)]);
        
    }
    return 0;
}
```

### Use Case 2: ระบบ Friend/Following

```objc
#import <Foundation/Foundation.h>

@interface User : NSObject
@property (nonatomic, strong) NSString *name;
@property (nonatomic, strong) NSMutableSet *following;

- (instancetype)initWithName:(NSString *)name;
- (void)follow:(User *)user;
- (void)unfollow:(User *)user;
- (NSSet *)mutualFriendsWith:(User *)user;
@end

@implementation User

- (instancetype)initWithName:(NSString *)name {
    self = [super init];
    if (self) {
        _name = name;
        _following = [NSMutableSet set];
    }
    return self;
}

- (void)follow:(User *)user {
    [self.following addObject:user.name];
}

- (void)unfollow:(User *)user {
    [self.following removeObject:user.name];
}

- (NSSet *)mutualFriendsWith:(User *)user {
    NSMutableSet *mutual = [NSMutableSet setWithSet:self.following];
    [mutual intersectSet:user.following];
    return [mutual copy];
}

@end

int main(int argc, const char * argv[]) {
    @autoreleasepool {
        
        User *alice = [[User alloc] initWithName:@"Alice"];
        User *bob = [[User alloc] initWithName:@"Bob"];
        User *charlie = [[User alloc] initWithName:@"Charlie"];
        User *dave = [[User alloc] initWithName:@"Dave"];
        
        // Alice follows
        [alice follow:bob];
        [alice follow:charlie];
        [alice follow:dave];
        
        // Bob follows
        [bob follow:alice];
        [bob follow:charlie];
        [bob follow:dave];
        
        NSLog(@"Alice follows: %@", 
              [alice.following.allObjects sortedArrayUsingSelector:@selector(compare:)]);
        NSLog(@"Bob follows: %@", 
              [bob.following.allObjects sortedArrayUsingSelector:@selector(compare:)]);
        
        // หา mutual friends
        NSSet *mutual = [alice mutualFriendsWith:bob];
        NSLog(@"Alice & Bob both follow: %@", 
              [mutual.allObjects sortedArrayUsingSelector:@selector(compare:)]);
        
    }
    return 0;
}
```

### Use Case 3: Cache ด้วย NSSet

```objc
#import <Foundation/Foundation.h>

@interface SimpleCache : NSObject
@property (nonatomic, strong) NSMutableSet *loadedItems;
- (BOOL)isLoaded:(NSString *)key;
- (void)markLoaded:(NSString *)key;
- (void)invalidate:(NSString *)key;
- (void)clearAll;
@end

@implementation SimpleCache

- (instancetype)init {
    self = [super init];
    if (self) {
        _loadedItems = [NSMutableSet set];
    }
    return self;
}

- (BOOL)isLoaded:(NSString *)key {
    return [self.loadedItems containsObject:key];
}

- (void)markLoaded:(NSString *)key {
    [self.loadedItems addObject:key];
}

- (void)invalidate:(NSString *)key {
    [self.loadedItems removeObject:key];
}

- (void)clearAll {
    [self.loadedItems removeAllObjects];
}

@end

int main(int argc, const char * argv[]) {
    @autoreleasepool {
        
        SimpleCache *cache = [[SimpleCache alloc] init];
        
        NSArray *pages = @[@"home", @"about", @"products", @"contact"];
        
        for (NSString *page in pages) {
            if (![cache isLoaded:page]) {
                NSLog(@"Loading page: %@", page);
                // simulate loading
                [cache markLoaded:page];
            } else {
                NSLog(@"Page cached: %@", page);
            }
        }
        
        // ขอโหลดซ้ำ - จะใช้ cache
        NSLog(@"\nRequesting pages again:");
        for (NSString *page in pages) {
            if (![cache isLoaded:page]) {
                NSLog(@"Loading page: %@", page);
            } else {
                NSLog(@"Using cached: %@", page);
            }
        }
        
        // invalidate
        [cache invalidate:@"products"];
        NSLog(@"\nAfter invalidating 'products':");
        if (![cache isLoaded:@"products"]) {
            NSLog(@"Products needs reload");
        }
        
    }
    return 0;
}
```

---

## 25.10 Encoding และ Decoding NSSet

### NSCoding กับ NSSet

```objc
#import <Foundation/Foundation.h>

int main(int argc, const char * argv[]) {
    @autoreleasepool {
        
        NSSet *originalSet = [NSSet setWithObjects:
                              @"item1", @"item2", @"item3", nil];
        
        // Encode เป็น NSData
        NSError *encodeError = nil;
        NSData *data = [NSKeyedArchiver archivedDataWithRootObject:originalSet 
                                             requiringSecureCoding:NO 
                                                            error:&encodeError];
        if (encodeError) {
            NSLog(@"Encode error: %@", encodeError);
        } else {
            NSLog(@"Encoded to %lu bytes", (unsigned long)data.length);
        }
        
        // Decode กลับมา
        NSError *decodeError = nil;
        NSSet *decodedSet = [NSKeyedUnarchiver unarchivedObjectOfClass:[NSSet class]
                                                              fromData:data
                                                                error:&decodeError];
        if (decodeError) {
            NSLog(@"Decode error: %@", decodeError);
        } else {
            NSLog(@"Decoded set: %@", decodedSet);
            NSLog(@"Sets equal: %@", [originalSet isEqualToSet:decodedSet] ? @"YES" : @"NO");
        }
        
    }
    return 0;
}
```

### บันทึกและอ่าน NSSet จากไฟล์

```objc
#import <Foundation/Foundation.h>

int main(int argc, const char * argv[]) {
    @autoreleasepool {
        
        NSSet *preferences = [NSSet setWithObjects:
                              @"darkMode", @"notifications", @"autoSave", nil];
        
        // บันทึกลงไฟล์
        NSString *filePath = @"/tmp/preferences.plist";
        NSArray *array = [preferences allObjects]; // Set -> Array สำหรับ plist
        BOOL success = [array writeToFile:filePath atomically:YES];
        NSLog(@"Save success: %@", success ? @"YES" : @"NO");
        
        // อ่านจากไฟล์
        NSArray *loadedArray = [NSArray arrayWithContentsOfFile:filePath];
        NSSet *loadedSet = [NSSet setWithArray:loadedArray];
        NSLog(@"Loaded: %@", [loadedSet.allObjects 
                              sortedArrayUsingSelector:@selector(compare:)]);
        
        // ตรวจสอบความเท่ากัน
        NSLog(@"Equal: %@", [preferences isEqualToSet:loadedSet] ? @"YES" : @"NO");
        
    }
    return 0;
}
```

---

## 25.11 NSSet กับ KVC (Key-Value Coding)

```objc
#import <Foundation/Foundation.h>

@interface Person : NSObject
@property (nonatomic, strong) NSString *name;
@property (nonatomic, assign) NSInteger age;
- (instancetype)initWithName:(NSString *)name age:(NSInteger)age;
@end

@implementation Person
- (instancetype)initWithName:(NSString *)name age:(NSInteger)age {
    self = [super init];
    if (self) {
        _name = name;
        _age = age;
    }
    return self;
}
- (NSString *)description {
    return [NSString stringWithFormat:@"%@(%ld)", self.name, (long)self.age];
}
@end

int main(int argc, const char * argv[]) {
    @autoreleasepool {
        
        NSSet *people = [NSSet setWithObjects:
                         [[Person alloc] initWithName:@"Alice" age:30],
                         [[Person alloc] initWithName:@"Bob" age:25],
                         [[Person alloc] initWithName:@"Charlie" age:35],
                         nil];
        
        // ดึง names ทั้งหมดด้วย KVC
        NSSet *names = [people valueForKey:@"name"];
        NSLog(@"Names: %@", [names.allObjects 
                             sortedArrayUsingSelector:@selector(compare:)]);
        
        // ดึง ages ทั้งหมด
        NSSet *ages = [people valueForKey:@"age"];
        NSLog(@"Ages: %@", ages.allObjects);
        
        // กรองด้วย Predicate
        NSPredicate *youngPred = [NSPredicate predicateWithFormat:@"age < 30"];
        NSSet *youngPeople = [people filteredSetUsingPredicate:youngPred];
        NSLog(@"Young people: %@", youngPeople);
        
    }
    return 0;
}
```

---

## 25.12 ข้อควรระวังในการใช้ NSSet

### 1. Objects ต้องมี hash และ isEqual ที่ถูกต้อง

```objc
#import <Foundation/Foundation.h>

// Custom class ที่ต้อง override hash และ isEqual
@interface Point : NSObject
@property (nonatomic) CGFloat x;
@property (nonatomic) CGFloat y;
- (instancetype)initWithX:(CGFloat)x y:(CGFloat)y;
@end

@implementation Point

- (instancetype)initWithX:(CGFloat)x y:(CGFloat)y {
    self = [super init];
    if (self) {
        _x = x;
        _y = y;
    }
    return self;
}

// MUST override isEqual: for NSSet to work correctly
- (BOOL)isEqual:(id)object {
    if (self == object) return YES;
    if (![object isKindOfClass:[Point class]]) return NO;
    Point *other = (Point *)object;
    return self.x == other.x && self.y == other.y;
}

// MUST override hash: if two objects are equal, they must have the same hash
- (NSUInteger)hash {
    return [@(self.x) hash] ^ [@(self.y) hash];
}

- (NSString *)description {
    return [NSString stringWithFormat:@"(%.0f, %.0f)", self.x, self.y];
}

@end

int main(int argc, const char * argv[]) {
    @autoreleasepool {
        
        Point *p1 = [[Point alloc] initWithX:1 y:2];
        Point *p2 = [[Point alloc] initWithX:1 y:2]; // เหมือนกับ p1
        Point *p3 = [[Point alloc] initWithX:3 y:4];
        
        NSSet *points = [NSSet setWithObjects:p1, p2, p3, nil];
        NSLog(@"Points (unique): %@", points); // ควรมีแค่ 2 ตัว
        NSLog(@"Count: %lu", (unsigned long)points.count);
        
    }
    return 0;
}
```

### 2. NSSet ไม่ Thread-Safe

```objc
#import <Foundation/Foundation.h>

int main(int argc, const char * argv[]) {
    @autoreleasepool {
        
        // ใช้ dispatch_queue เพื่อทำให้ thread-safe
        NSMutableSet *sharedSet = [NSMutableSet set];
        dispatch_queue_t queue = dispatch_queue_create("com.example.setQueue", 
                                                        DISPATCH_QUEUE_SERIAL);
        
        // เพิ่มข้อมูลแบบ thread-safe
        dispatch_async(queue, ^{
            [sharedSet addObject:@"item1"];
        });
        
        dispatch_async(queue, ^{
            [sharedSet addObject:@"item2"];
        });
        
        // อ่านข้อมูลแบบ thread-safe
        dispatch_sync(queue, ^{
            NSLog(@"Set contents: %@", sharedSet);
        });
        
    }
    return 0;
}
```

---

## 25.13 สรุปเปรียบเทียบ Collection Types

| Feature | NSArray | NSSet | NSOrderedSet | NSDictionary |
|---------|---------|-------|--------------|--------------|
| Ordered | ✅ | ❌ | ✅ | ❌ |
| Unique values | ❌ | ✅ | ✅ | Keys: ✅ |
| Access by index | ✅ | ❌ | ✅ | By key |
| Search speed | O(n) | O(1) | O(1) | O(1) |
| Mutable version | NSMutableArray | NSMutableSet | NSMutableOrderedSet | NSMutableDictionary |
| Count tracking | ❌ | NSCountedSet | ❌ | ❌ |

---

## 25.14 แบบฝึกหัด (Practice Exercises)

### แบบฝึกหัดที่ 1: กรองค่าซ้ำ

เขียนโปรแกรมที่รับ NSArray ที่มีตัวเลขซ้ำกัน แล้วส่งคืน NSArray ที่ไม่มีค่าซ้ำ โดยรักษาลำดับการปรากฏตัวแรกของแต่ละค่า

```objc
#import <Foundation/Foundation.h>

NSArray *removeDuplicates(NSArray *input) {
    NSMutableOrderedSet *seen = [NSMutableOrderedSet orderedSet];
    [seen addObjectsFromArray:input];
    return [seen array];
}

int main(int argc, const char * argv[]) {
    @autoreleasepool {
        NSArray *input = @[@3, @1, @4, @1, @5, @9, @2, @6, @5, @3, @5];
        NSArray *result = removeDuplicates(input);
        NSLog(@"Input:  %@", input);
        NSLog(@"Output: %@", result);
        // ควรได้: (3, 1, 4, 5, 9, 2, 6)
    }
    return 0;
}
```

### แบบฝึกหัดที่ 2: ตรวจสอบ Anagram

เขียนฟังก์ชันตรวจสอบว่าสองคำเป็น Anagram กันหรือไม่ (ใช้ตัวอักษรเดียวกัน ลำดับต่างกัน) ด้วย NSCountedSet

```objc
#import <Foundation/Foundation.h>

BOOL isAnagram(NSString *word1, NSString *word2) {
    if (word1.length != word2.length) return NO;
    
    NSCountedSet *chars1 = [NSCountedSet set];
    NSCountedSet *chars2 = [NSCountedSet set];
    
    for (int i = 0; i < word1.length; i++) {
        NSString *ch = [word1 substringWithRange:NSMakeRange(i, 1)];
        [chars1 addObject:ch.lowercaseString];
    }
    
    for (int i = 0; i < word2.length; i++) {
        NSString *ch = [word2 substringWithRange:NSMakeRange(i, 1)];
        [chars2 addObject:ch.lowercaseString];
    }
    
    return [chars1 isEqualToSet:chars2];
}

int main(int argc, const char * argv[]) {
    @autoreleasepool {
        NSLog(@"'listen' and 'silent': %@", 
              isAnagram(@"listen", @"silent") ? @"Anagram" : @"Not Anagram");
        NSLog(@"'hello' and 'world': %@", 
              isAnagram(@"hello", @"world") ? @"Anagram" : @"Not Anagram");
        NSLog(@"'Astronomer' and 'Moon starer': %@", 
              isAnagram(@"Astronomer", @"Moon starer") ? @"Anagram" : @"Not Anagram");
    }
    return 0;
}
```

### แบบฝึกหัดที่ 3: ระบบสิทธิ์

```objc
#import <Foundation/Foundation.h>

@interface RoleSystem : NSObject
@property (nonatomic, strong) NSMutableDictionary *roles; // role -> NSSet of permissions

- (void)addRole:(NSString *)role permissions:(NSArray *)permissions;
- (BOOL)role:(NSString *)role hasPermission:(NSString *)permission;
- (NSSet *)permissionsForRole:(NSString *)role;
- (NSSet *)allPermissionsForRoles:(NSArray *)roles;
@end

@implementation RoleSystem

- (instancetype)init {
    self = [super init];
    if (self) {
        _roles = [NSMutableDictionary dictionary];
    }
    return self;
}

- (void)addRole:(NSString *)role permissions:(NSArray *)permissions {
    self.roles[role] = [NSSet setWithArray:permissions];
}

- (BOOL)role:(NSString *)role hasPermission:(NSString *)permission {
    NSSet *perms = self.roles[role];
    return [perms containsObject:permission];
}

- (NSSet *)permissionsForRole:(NSString *)role {
    return self.roles[role] ?: [NSSet set];
}

- (NSSet *)allPermissionsForRoles:(NSArray *)roles {
    NSMutableSet *all = [NSMutableSet set];
    for (NSString *role in roles) {
        [all unionSet:[self permissionsForRole:role]];
    }
    return [all copy];
}

@end

int main(int argc, const char * argv[]) {
    @autoreleasepool {
        
        RoleSystem *system = [[RoleSystem alloc] init];
        
        [system addRole:@"viewer" permissions:@[@"read"]];
        [system addRole:@"editor" permissions:@[@"read", @"write", @"comment"]];
        [system addRole:@"admin" permissions:@[@"read", @"write", @"delete", 
                                               @"manage_users", @"view_logs"]];
        
        NSLog(@"Can editor write? %@", 
              [system role:@"editor" hasPermission:@"write"] ? @"YES" : @"NO");
        NSLog(@"Can viewer delete? %@", 
              [system role:@"viewer" hasPermission:@"delete"] ? @"YES" : @"NO");
        
        // User ที่มีทั้ง viewer และ editor role
        NSSet *userPerms = [system allPermissionsForRoles:@[@"viewer", @"editor"]];
        NSLog(@"\nViewer+Editor permissions: %@", 
              [userPerms.allObjects sortedArrayUsingSelector:@selector(compare:)]);
        
    }
    return 0;
}
```

### แบบฝึกหัดที่ 4: Two Sum ด้วย NSSet

```objc
#import <Foundation/Foundation.h>

// หา pair ของตัวเลขที่รวมกันได้ target
NSArray *twoSum(NSArray *numbers, NSInteger target) {
    NSMutableSet *seen = [NSMutableSet set];
    
    for (NSNumber *num in numbers) {
        NSInteger complement = target - num.integerValue;
        NSNumber *compNum = @(complement);
        
        if ([seen containsObject:compNum]) {
            return @[compNum, num];
        }
        [seen addObject:num];
    }
    
    return nil;
}

int main(int argc, const char * argv[]) {
    @autoreleasepool {
        NSArray *numbers = @[@2, @7, @11, @15];
        NSArray *result = twoSum(numbers, 9);
        if (result) {
            NSLog(@"Pair that sums to 9: %@ + %@ = 9", result[0], result[1]);
        }
        
        result = twoSum(numbers, 18);
        if (result) {
            NSLog(@"Pair that sums to 18: %@ + %@ = 18", result[0], result[1]);
        }
    }
    return 0;
}
```

---

## สรุปบทที่ 25

ในบทนี้เราได้เรียนรู้:

1. **NSSet** - Collection ที่ไม่มีค่าซ้ำและไม่มีลำดับ ค้นหาข้อมูลเร็ว O(1)
2. **NSMutableSet** - NSSet ที่แก้ไขได้ รองรับการเพิ่ม/ลบสมาชิก
3. **Set Operations** - union (รวม), intersection (จุดร่วม), minus (ผลต่าง)
4. **NSCountedSet** - Set ที่นับจำนวนการเพิ่มซ้ำ
5. **NSOrderedSet** - Set ที่มีลำดับ รวมข้อดีของ Array และ Set
6. **NSMutableOrderedSet** - OrderedSet ที่แก้ไขได้
7. **การเลือกใช้** - Array เมื่อต้องการลำดับ, Set เมื่อต้องการ unique values

### คำถามทบทวน

1. ทำไม NSSet ถึงค้นหาข้อมูลเร็วกว่า NSArray?
2. ความแตกต่างระหว่าง NSSet และ NSOrderedSet คืออะไร?
3. เมื่อไหรควรใช้ NSCountedSet?
4. ทำไม custom class ที่ใช้ใน NSSet ต้อง override `hash` และ `isEqual:`?

---

**ต่อไป:** Part 26 - NSNumber และ NSValue
