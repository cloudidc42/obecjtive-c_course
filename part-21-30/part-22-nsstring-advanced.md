# ตอนที่ 22 - NSString Advanced

## บทนำ

`NSString` เป็นหนึ่งในคลาสที่ใช้บ่อยที่สุดใน Objective-C ในบทนี้เราจะเจาะลึก NSString อย่างครบถ้วน ตั้งแต่การสร้าง string ไปจนถึง advanced features เช่น regular expressions, attributed strings, และ scanning

---

## 22.1 วิธีสร้าง NSString

### 22.1.1 String Literals

```objc
#import <Foundation/Foundation.h>

int main(int argc, const char * argv[]) {
    @autoreleasepool {
        // Literal syntax (แนะนำ)
        NSString *s1 = @"สวัสดี Objective-C";
        
        // สร้างจาก C string
        NSString *s2 = [NSString stringWithUTF8String:"Hello World"];
        const char *cString = "Objective-C";
        NSString *s3 = [NSString stringWithCString:cString
                                          encoding:NSUTF8StringEncoding];
        
        // สร้างจาก format
        NSString *s4 = [NSString stringWithFormat:@"ชื่อ: %@, อายุ: %d",
                        @"สมชาย", 30];
        
        // สร้างจาก file
        NSString *s5 = [NSString stringWithContentsOfFile:@"/tmp/test.txt"
                                                 encoding:NSUTF8StringEncoding
                                                    error:nil];
        
        // สร้างจาก URL
        NSURL *url = [NSURL URLWithString:@"https://example.com/data.txt"];
        NSString *s6 = [NSString stringWithContentsOfURL:url
                                                encoding:NSUTF8StringEncoding
                                                   error:nil];
        
        // สร้าง empty string
        NSString *empty1 = @"";
        NSString *empty2 = [NSString string];
        
        // สร้างจาก NSData
        NSData *data = [@"Hello" dataUsingEncoding:NSUTF8StringEncoding];
        NSString *s7 = [[NSString alloc] initWithData:data
                                             encoding:NSUTF8StringEncoding];
        
        NSLog(@"s1: %@", s1);
        NSLog(@"s2: %@", s2);
        NSLog(@"s4: %@", s4);
        NSLog(@"s7: %@", s7);
    }
    return 0;
}
```

### 22.1.2 NSMutableString

```objc
// สร้าง mutable string
NSMutableString *ms1 = [NSMutableString string];
NSMutableString *ms2 = [NSMutableString stringWithCapacity:100]; // pre-allocate
NSMutableString *ms3 = [NSMutableString stringWithString:@"Hello"];
NSMutableString *ms4 = [@"World" mutableCopy]; // copy immutable to mutable

// การแก้ไข
[ms1 appendString:@"Hello"];
[ms1 appendString:@" World"];
[ms1 appendFormat:@" %d times", 3];

[ms3 insertString:@"Say: " atIndex:0];
[ms3 deleteCharactersInRange:NSMakeRange(0, 5)]; // ลบ 5 ตัวแรก
[ms3 replaceCharactersInRange:NSMakeRange(0, 5) withString:@"Hi"];

NSLog(@"ms1: %@", ms1);
NSLog(@"ms3: %@", ms3);
```

---

## 22.2 Format Strings Deep Dive

### Format Specifiers ทั้งหมด

```objc
#import <Foundation/Foundation.h>

int main(int argc, const char * argv[]) {
    @autoreleasepool {
        // Integer formats
        int i = 42;
        long l = 1234567890L;
        NSInteger ni = 100;
        
        NSLog(@"int %%d: %d", i);
        NSLog(@"int %%i: %i", i);         // เหมือน %d
        NSLog(@"int %%o: %o", i);         // octal: 52
        NSLog(@"int %%x: %x", i);         // hex lowercase: 2a
        NSLog(@"int %%X: %X", i);         // hex uppercase: 2A
        NSLog(@"long %%ld: %ld", l);
        NSLog(@"NSInteger %%ld: %ld", (long)ni);
        
        // Float formats
        double d = 3.14159265358979;
        float f = 3.14f;
        CGFloat cgf = 2.71828;
        
        NSLog(@"double %%f: %f", d);           // 3.141593
        NSLog(@"double %%.2f: %.2f", d);        // 3.14
        NSLog(@"double %%e: %e", d);            // scientific: 3.141593e+00
        NSLog(@"double %%g: %g", d);            // shorter of %e or %f
        NSLog(@"CGFloat: %f", (double)cgf);
        
        // String formats
        NSString *str = @"Objective-C";
        const char *cstr = "C String";
        
        NSLog(@"NSString %%@: %@", str);
        NSLog(@"C String %%s: %s", cstr);
        NSLog(@"C String %%S (unicode): %S", (unichar *)L"Unicode");
        
        // Width and alignment
        NSLog(@"Width 10, right: '%10d'", i);      // '        42'
        NSLog(@"Width 10, left:  '%-10d'", i);     // '42        '
        NSLog(@"Zero padded: '%010d'", i);          // '0000000042'
        NSLog(@"Width string: '%20@'", str);        // right aligned
        NSLog(@"Left string:  '%-20@'", str);       // left aligned
        
        // Multiple args
        NSLog(@"%@ is %d years old and %.1fcm tall",
              @"สมชาย", 30, 175.5);
        
        // Object
        NSDate *date = [NSDate date];
        NSLog(@"Date: %@", date); // calls [date description]
        
        // Boolean
        BOOL flag = YES;
        NSLog(@"BOOL: %@", flag ? @"YES" : @"NO");
        NSLog(@"BOOL as int: %d", flag); // 1 or 0
    }
    return 0;
}
```

### String Formatting แบบขั้นสูง

```objc
// NSString format arguments
NSString *template = @"สวัสดี %1$@! คุณอายุ %2$d ปี และสูง %3$.1f ซม.";
NSString *result = [NSString stringWithFormat:template,
                    @"สมหญิง", 25, 162.5];
// Positional arguments (1$, 2$, 3$) ช่วยให้ reorder ได้

// stringByAppendingFormat
NSMutableString *builder = [NSMutableString string];
for (int i = 1; i <= 5; i++) {
    [builder appendFormat:@"Item %d: %@\n", i, @"Value"];
}
NSLog(@"%@", builder);
```

---

## 22.3 String Comparison Options

### NSStringCompareOptions

```objc
#import <Foundation/Foundation.h>

int main(int argc, const char * argv[]) {
    @autoreleasepool {
        NSString *str1 = @"Hello World";
        NSString *str2 = @"hello world";
        NSString *str3 = @"Héllo";
        NSString *str4 = @"Hello";
        
        // Default comparison (case-sensitive)
        NSComparisonResult result = [str1 compare:str2];
        NSLog(@"Default: %ld", (long)result); // NSOrderedDescending (1)
        
        // Case-insensitive
        result = [str1 compare:str2 options:NSCaseInsensitiveSearch];
        NSLog(@"Case-insensitive: %ld", (long)result); // NSOrderedSame (0)
        
        // Diacritic-insensitive (ไม่สนเครื่องหมายกำกับเสียง)
        result = [str3 compare:str4 options:NSDiacriticInsensitiveSearch
                                          | NSCaseInsensitiveSearch];
        NSLog(@"Diacritic-insensitive: %ld", (long)result); // NSOrderedSame
        
        // Numeric comparison
        NSString *v1 = @"file10.txt";
        NSString *v2 = @"file9.txt";
        result = [v1 compare:v2 options:NSNumericSearch];
        NSLog(@"Numeric: '%@' vs '%@': %ld",
              v1, v2, (long)result); // NSOrderedDescending (10 > 9)
        
        // Backward search
        NSString *path = @"/home/user/documents/file.txt";
        NSRange lastSlash = [path rangeOfString:@"/" options:NSBackwardsSearch];
        NSLog(@"Last slash at: %lu", (unsigned long)lastSlash.location);
        
        // Literal comparison (exact bytes)
        result = [str1 compare:str2 options:NSLiteralSearch];
        NSLog(@"Literal: %ld", (long)result);
        
        // Anchored (match only at beginning or end)
        BOOL startsWithH = [str1 rangeOfString:@"He"
                                       options:NSAnchoredSearch].location != NSNotFound;
        NSLog(@"Starts with 'He': %@", startsWithH ? @"YES" : @"NO");
        
        BOOL endsWithWorld = [str1 rangeOfString:@"World"
                                         options:NSAnchoredSearch | NSBackwardsSearch].location != NSNotFound;
        NSLog(@"Ends with 'World': %@", endsWithWorld ? @"YES" : @"NO");
        
        // Summary of options:
        // NSCaseInsensitiveSearch      - ไม่สนตัวพิมพ์เล็ก/ใหญ่
        // NSDiacriticInsensitiveSearch - ไม่สนเครื่องหมายกำกับเสียง
        // NSNumericSearch              - เปรียบเทียบตัวเลขแบบ numeric
        // NSBackwardsSearch            - หาจากท้าย
        // NSAnchoredSearch             - match เฉพาะต้นหรือท้าย
        // NSLiteralSearch              - exact byte comparison
        // NSForcedOrderingSearch       - รับประกันว่า != จะได้ผลที่ไม่เปลี่ยน
        // NSRegularExpressionSearch    - ใช้ regex
        // NSWidthInsensitiveSearch     - ไม่สนความกว้างของ char (full/half-width)
    }
    return 0;
}
```

### เปรียบเทียบสำหรับ Sorting

```objc
NSArray *words = @[@"banana", @"Apple", @"cherry", @"Date", @"elderberry"];

// Sort case-insensitive
NSArray *sorted1 = [words sortedArrayUsingComparator:^NSComparisonResult(NSString *a, NSString *b) {
    return [a compare:b options:NSCaseInsensitiveSearch];
}];
NSLog(@"Case-insensitive sort: %@", sorted1);

// Sort localized (เหมาะสำหรับ user-facing sorting)
NSArray *sorted2 = [words sortedArrayUsingSelector:@selector(localizedCaseInsensitiveCompare:)];
NSLog(@"Localized sort: %@", sorted2);

// Sort natural/human order
NSArray *files = @[@"file1.txt", @"file10.txt", @"file2.txt", @"file20.txt"];
NSArray *sortedFiles = [files sortedArrayUsingComparator:^NSComparisonResult(NSString *a, NSString *b) {
    return [a compare:b options:NSNumericSearch | NSCaseInsensitiveSearch];
}];
NSLog(@"Natural sort: %@", sortedFiles); // file1, file2, file10, file20
```

---

## 22.4 NSRange และ String Operations

### ทำความเข้าใจ NSRange

```objc
#import <Foundation/Foundation.h>

int main(int argc, const char * argv[]) {
    @autoreleasepool {
        // NSRange คือ struct: { NSUInteger location; NSUInteger length; }
        NSRange range1 = NSMakeRange(5, 10); // เริ่มที่ index 5, ยาว 10 ตัว
        NSLog(@"Location: %lu, Length: %lu",
              (unsigned long)range1.location,
              (unsigned long)range1.length);
        
        NSString *text = @"Hello, World! How are you?";
        
        // rangeOfString: - หาตำแหน่ง substring
        NSRange range = [text rangeOfString:@"World"];
        if (range.location != NSNotFound) {
            NSLog(@"'World' found at %lu, length %lu",
                  (unsigned long)range.location,
                  (unsigned long)range.length);
        }
        
        // สกัด substring จาก range
        NSString *extracted = [text substringWithRange:range];
        NSLog(@"Extracted: '%@'", extracted); // World
        
        // ตรวจสอบ range ที่ invalid
        NSRange notFound = [text rangeOfString:@"xyz"];
        if (notFound.location == NSNotFound) {
            NSLog(@"'xyz' not found");
        }
        
        // NSUnionRange - รวม 2 ranges
        NSRange r1 = NSMakeRange(0, 5);
        NSRange r2 = NSMakeRange(10, 5);
        NSRange union1 = NSUnionRange(r1, r2);
        NSLog(@"Union: {%lu, %lu}",
              (unsigned long)union1.location,
              (unsigned long)union1.length); // {0, 15}
        
        // NSIntersectionRange - ตัดกันของ 2 ranges
        NSRange r3 = NSMakeRange(3, 10);
        NSRange r4 = NSMakeRange(8, 10);
        NSRange intersection = NSIntersectionRange(r3, r4);
        NSLog(@"Intersection: {%lu, %lu}",
              (unsigned long)intersection.location,
              (unsigned long)intersection.length); // {8, 5}
        
        // NSRangeFromString / NSStringFromRange
        NSString *rangeStr = NSStringFromRange(NSMakeRange(5, 10));
        NSLog(@"Range as string: %@", rangeStr); // {5, 10}
        NSRange parsedRange = NSRangeFromString(rangeStr);
        NSLog(@"Parsed: {%lu, %lu}",
              (unsigned long)parsedRange.location,
              (unsigned long)parsedRange.length);
    }
    return 0;
}
```

---

## 22.5 Substrings, Prefix, Suffix

```objc
#import <Foundation/Foundation.h>

int main(int argc, const char * argv[]) {
    @autoreleasepool {
        NSString *text = @"The quick brown fox jumps over the lazy dog";
        
        // substringFromIndex: - จาก index จนสุด
        NSString *sub1 = [text substringFromIndex:4];
        NSLog(@"From index 4: '%@'", sub1); // "quick brown fox..."
        
        // substringToIndex: - ตั้งแต่ต้นถึง index
        NSString *sub2 = [text substringToIndex:9];
        NSLog(@"To index 9: '%@'", sub2); // "The quick"
        
        // substringWithRange: - ตามช่วง
        NSString *sub3 = [text substringWithRange:NSMakeRange(4, 5)];
        NSLog(@"Range [4,5]: '%@'", sub3); // "quick"
        
        // hasPrefix: และ hasSuffix:
        NSLog(@"Has prefix 'The': %@",
              [text hasPrefix:@"The"] ? @"YES" : @"NO");
        NSLog(@"Has suffix 'dog': %@",
              [text hasSuffix:@"dog"] ? @"YES" : @"NO");
        NSLog(@"Has prefix 'Quick': %@", // case-sensitive
              [text hasPrefix:@"Quick"] ? @"YES" : @"NO");
        
        // Case-insensitive prefix/suffix check
        BOOL caseInsensitivePrefix = [[text lowercaseString]
                                      hasPrefix:[@"quick" lowercaseString]];
        NSLog(@"Has prefix 'quick' (insensitive): %@",
              caseInsensitivePrefix ? @"YES" : @"NO");
        
        // ดึง extension จาก filename
        NSString *filename = @"document.pdf";
        NSString *extension = [filename pathExtension]; // pdf
        NSString *basename = [filename stringByDeletingPathExtension]; // document
        NSLog(@"Extension: %@, Basename: %@", extension, basename);
        
        // ดึง path components
        NSString *path = @"/home/user/documents/file.txt";
        NSLog(@"Last component: %@", [path lastPathComponent]); // file.txt
        NSLog(@"Dir path: %@", [path stringByDeletingLastPathComponent]);
        NSLog(@"Components: %@", [path pathComponents]);
    }
    return 0;
}
```

---

## 22.6 String Splitting

```objc
#import <Foundation/Foundation.h>

int main(int argc, const char * argv[]) {
    @autoreleasepool {
        // componentsSeparatedByString:
        NSString *csv = @"apple,banana,cherry,date";
        NSArray *fruits = [csv componentsSeparatedByString:@","];
        NSLog(@"Fruits: %@", fruits);
        // ["apple", "banana", "cherry", "date"]
        
        // แยกด้วยหลาย separators
        NSString *text = @"one,two;three|four";
        NSArray *parts = [text componentsSeparatedByCharactersInSet:
                          [NSCharacterSet characterSetWithCharactersInString:@",;|"]];
        NSLog(@"Parts: %@", parts);
        // ["one", "two", "three", "four"]
        
        // แยกบรรทัด
        NSString *multiline = @"Line 1\nLine 2\nLine 3\nLine 4";
        NSArray *lines = [multiline componentsSeparatedByString:@"\n"];
        NSLog(@"Lines count: %lu", (unsigned long)lines.count);
        for (NSString *line in lines) {
            NSLog(@"  '%@'", line);
        }
        
        // แยกด้วย whitespace
        NSString *sentence = @"Hello   World  Foo   Bar";
        NSArray *words = [sentence componentsSeparatedByCharactersInSet:
                          [NSCharacterSet whitespaceCharacterSet]];
        // กรอง empty strings ออก
        NSArray *filteredWords = [words filteredArrayUsingPredicate:
                                  [NSPredicate predicateWithFormat:@"SELF != ''"]];
        NSLog(@"Words: %@", filteredWords);
        
        // แยก string ที่ซับซ้อน (path)
        NSString *urlPath = @"/api/v1/users/123/profile";
        NSArray *segments = [urlPath componentsSeparatedByString:@"/"];
        NSArray *filteredSegments = [segments filteredArrayUsingPredicate:
                                     [NSPredicate predicateWithFormat:@"SELF != ''"]];
        NSLog(@"URL segments: %@", filteredSegments);
        // ["api", "v1", "users", "123", "profile"]
        
        // Split แบบ limit
        NSString *data = @"name:John:Doe:Extra";
        // manual split with limit
        NSMutableArray *limitedParts = [NSMutableArray array];
        NSString *remaining = data;
        NSInteger limit = 3;
        for (NSInteger i = 0; i < limit - 1; i++) {
            NSRange sep = [remaining rangeOfString:@":"];
            if (sep.location == NSNotFound) break;
            [limitedParts addObject:[remaining substringToIndex:sep.location]];
            remaining = [remaining substringFromIndex:sep.location + 1];
        }
        [limitedParts addObject:remaining];
        NSLog(@"Limited split: %@", limitedParts);
        // ["name", "John", "Doe:Extra"]
    }
    return 0;
}
```

---

## 22.7 String Joining

```objc
#import <Foundation/Foundation.h>

int main(int argc, const char * argv[]) {
    @autoreleasepool {
        // componentsJoinedByString: - join array elements
        NSArray *fruits = @[@"apple", @"banana", @"cherry"];
        
        NSString *joined1 = [fruits componentsJoinedByString:@", "];
        NSLog(@"Comma-joined: %@", joined1); // apple, banana, cherry
        
        NSString *joined2 = [fruits componentsJoinedByString:@" and "];
        NSLog(@"And-joined: %@", joined2); // apple and banana and cherry
        
        NSString *joined3 = [fruits componentsJoinedByString:@""];
        NSLog(@"No separator: %@", joined3); // applebananacherry
        
        // Join ด้วย newline
        NSArray *lines = @[@"First line", @"Second line", @"Third line"];
        NSString *text = [lines componentsJoinedByString:@"\n"];
        NSLog(@"Multi-line:\n%@", text);
        
        // Join path components
        NSArray *pathParts = @[@"home", @"user", @"documents", @"file.txt"];
        NSString *path = [NSString pathWithComponents:pathParts];
        NSLog(@"Path: %@", path); // home/user/documents/file.txt
        
        // Build CSV
        NSArray *headers = @[@"Name", @"Age", @"Email"];
        NSArray *row = @[@"สมชาย", @"30", @"somchai@example.com"];
        NSString *csvLine = [row componentsJoinedByString:@","];
        NSLog(@"CSV: %@", csvLine);
        
        // Join NSNumber array
        NSArray<NSNumber *> *numbers = @[@1, @2, @3, @4, @5];
        NSArray *numberStrings = [numbers valueForKey:@"stringValue"];
        NSString *numStr = [numberStrings componentsJoinedByString:@" + "];
        NSLog(@"Numbers: %@", numStr); // 1 + 2 + 3 + 4 + 5
    }
    return 0;
}
```

---

## 22.8 String Replacement

```objc
#import <Foundation/Foundation.h>

int main(int argc, const char * argv[]) {
    @autoreleasepool {
        NSString *text = @"The cat sat on the mat. The cat is fat.";
        
        // stringByReplacingOccurrencesOfString:withString:
        NSString *replaced1 = [text stringByReplacingOccurrencesOfString:@"cat"
                                                               withString:@"dog"];
        NSLog(@"Replace all: %@", replaced1);
        // The dog sat on the mat. The dog is fat.
        
        // Replace case-insensitive
        NSString *replaced2 = [text stringByReplacingOccurrencesOfString:@"CAT"
                                                               withString:@"bird"
                                                                  options:NSCaseInsensitiveSearch
                                                                    range:NSMakeRange(0, text.length)];
        NSLog(@"Case-insensitive: %@", replaced2);
        
        // Replace in range only
        NSString *replaced3 = [text stringByReplacingOccurrencesOfString:@"cat"
                                                               withString:@"fish"
                                                                  options:0
                                                                    range:NSMakeRange(0, 20)];
        NSLog(@"Range replace: %@", replaced3);
        // เปลี่ยนเฉพาะ "cat" ในส่วนแรก 20 ตัวอักษร
        
        // stringByReplacingCharactersInRange:withString:
        NSString *str = @"Hello World";
        NSString *replaced4 = [str stringByReplacingCharactersInRange:
                                NSMakeRange(6, 5)
                                                           withString:@"Objective-C"];
        NSLog(@"Range replace: %@", replaced4); // Hello Objective-C
        
        // แทนที่หลาย patterns
        NSMutableString *mutable = [text mutableCopy];
        [mutable replaceOccurrencesOfString:@"cat" withString:@"dog"
                                    options:0
                                      range:NSMakeRange(0, mutable.length)];
        [mutable replaceOccurrencesOfString:@"mat" withString:@"rug"
                                    options:0
                                      range:NSMakeRange(0, mutable.length)];
        NSLog(@"Multiple replace: %@", mutable);
        
        // เปลี่ยน template placeholders
        NSString *template = @"Dear {{name}}, Your order {{orderId}} is ready.";
        NSDictionary *values = @{
            @"{{name}}": @"สมชาย",
            @"{{orderId}}": @"ORD-12345"
        };
        NSMutableString *result = [template mutableCopy];
        for (NSString *key in values) {
            [result replaceOccurrencesOfString:key
                                    withString:values[key]
                                       options:0
                                         range:NSMakeRange(0, result.length)];
        }
        NSLog(@"Template: %@", result);
    }
    return 0;
}
```

---

## 22.9 Trimming Whitespace

```objc
#import <Foundation/Foundation.h>

int main(int argc, const char * argv[]) {
    @autoreleasepool {
        NSString *padded = @"   Hello World   ";
        NSString *withNewlines = @"\n\t  Hello\n\t  ";
        NSString *withMiddle = @"  Hello   World  ";
        
        // Trim whitespace และ newlines ทั้งสองด้าน
        NSString *trimmed1 = [padded stringByTrimmingCharactersInSet:
                              [NSCharacterSet whitespaceAndNewlineCharacterSet]];
        NSLog(@"Trimmed: '%@'", trimmed1); // 'Hello World'
        
        // Trim เฉพาะ whitespace (ไม่รวม newline)
        NSString *trimmed2 = [padded stringByTrimmingCharactersInSet:
                              [NSCharacterSet whitespaceCharacterSet]];
        NSLog(@"Trim whitespace: '%@'", trimmed2);
        
        // Trim newlines ด้วย
        NSString *trimmed3 = [withNewlines stringByTrimmingCharactersInSet:
                              [NSCharacterSet whitespaceAndNewlineCharacterSet]];
        NSLog(@"Trim newlines: '%@'", trimmed3);
        
        // Trim characters เฉพาะ
        NSString *quoted = @"\"Hello World\"";
        NSString *unquoted = [quoted stringByTrimmingCharactersInSet:
                              [NSCharacterSet characterSetWithCharactersInString:@"\""]];
        NSLog(@"Unquoted: '%@'", unquoted);
        
        // Trim punctuation
        NSString *punctuated = @"...Hello World!!!";
        NSString *trimmedPunct = [punctuated stringByTrimmingCharactersInSet:
                                  [NSCharacterSet punctuationCharacterSet]];
        NSLog(@"No punctuation: '%@'", trimmedPunct);
        
        // Collapse multiple spaces (trim middle spaces)
        NSArray *words = [withMiddle componentsSeparatedByCharactersInSet:
                          [NSCharacterSet whitespaceCharacterSet]];
        NSArray *filtered = [words filteredArrayUsingPredicate:
                             [NSPredicate predicateWithFormat:@"SELF != ''"]];
        NSString *collapsed = [filtered componentsJoinedByString:@" "];
        NSLog(@"Collapsed: '%@'", collapsed); // 'Hello World'
    }
    return 0;
}
```

---

## 22.10 Encoding และ Decoding

```objc
#import <Foundation/Foundation.h>

int main(int argc, const char * argv[]) {
    @autoreleasepool {
        // Supported encodings
        NSLog(@"Available string encodings:");
        NSLog(@"  UTF-8: %lu", (unsigned long)NSUTF8StringEncoding);
        NSLog(@"  UTF-16: %lu", (unsigned long)NSUTF16StringEncoding);
        NSLog(@"  ASCII: %lu", (unsigned long)NSASCIIStringEncoding);
        NSLog(@"  ISO Latin-1: %lu", (unsigned long)NSISOLatin1StringEncoding);
        NSLog(@"  TIS-620 (Thai): %lu", (unsigned long)2564); // Thai encoding
        
        // String → NSData
        NSString *str = @"สวัสดี Hello";
        NSData *utf8Data = [str dataUsingEncoding:NSUTF8StringEncoding];
        NSData *utf16Data = [str dataUsingEncoding:NSUTF16StringEncoding];
        NSLog(@"UTF-8 size: %lu bytes", (unsigned long)utf8Data.length);
        NSLog(@"UTF-16 size: %lu bytes", (unsigned long)utf16Data.length);
        
        // NSData → String
        NSString *restored = [[NSString alloc] initWithData:utf8Data
                                                   encoding:NSUTF8StringEncoding];
        NSLog(@"Restored: %@", restored);
        
        // ตรวจสอบ encoding ที่ใช้ได้
        NSStringEncoding enc;
        NSData *dataWithEnc = [str dataUsingEncoding:NSISOLatin1StringEncoding
                                allowLossyConversion:NO];
        if (dataWithEnc == nil) {
            NSLog(@"Cannot encode Thai in ISO-8859-1 without loss");
        }
        
        // URL Encoding
        NSString *url = @"https://example.com/search?q=สวัสดี โลก&lang=th";
        NSString *encoded = [url stringByAddingPercentEncodingWithAllowedCharacters:
                             [NSCharacterSet URLQueryAllowedCharacterSet]];
        NSLog(@"URL encoded: %@", encoded);
        
        NSString *decoded = [encoded stringByRemovingPercentEncoding];
        NSLog(@"URL decoded: %@", decoded);
        
        // Encode specific parts
        NSString *queryValue = @"hello world & foo=bar";
        NSString *encodedQuery = [queryValue
            stringByAddingPercentEncodingWithAllowedCharacters:
            [NSCharacterSet URLQueryAllowedCharacterSet]];
        NSLog(@"Encoded query: %@", encodedQuery);
        
        // HTML entity encoding (manual)
        NSDictionary *htmlEntities = @{
            @"<": @"&lt;",
            @">": @"&gt;",
            @"&": @"&amp;",
            @"\"": @"&quot;",
            @"'": @"&#39;"
        };
        
        NSString *html = @"<script>alert('XSS & \"attack\"');</script>";
        NSMutableString *escaped = [html mutableCopy];
        for (NSString *char_ in htmlEntities) {
            [escaped replaceOccurrencesOfString:char_
                                     withString:htmlEntities[char_]
                                        options:0
                                          range:NSMakeRange(0, escaped.length)];
        }
        NSLog(@"HTML escaped: %@", escaped);
        
        // Base64
        NSString *original = @"Hello, World! สวัสดี";
        NSData *originalData = [original dataUsingEncoding:NSUTF8StringEncoding];
        NSString *base64 = [originalData base64EncodedStringWithOptions:0];
        NSLog(@"Base64: %@", base64);
        
        NSData *decodedData = [[NSData alloc] initWithBase64EncodedString:base64
                                                                   options:0];
        NSString *decodedStr = [[NSString alloc] initWithData:decodedData
                                                     encoding:NSUTF8StringEncoding];
        NSLog(@"Decoded: %@", decodedStr);
    }
    return 0;
}
```

---

## 22.11 NSAttributedString

```objc
#import <Foundation/Foundation.h>
#if TARGET_OS_OSX
#import <AppKit/AppKit.h>
#elif TARGET_OS_IPHONE
#import <UIKit/UIKit.h>
#endif

int main(int argc, const char * argv[]) {
    @autoreleasepool {
        // สร้าง basic attributed string
        NSString *text = @"Hello, World! This is styled text.";
        NSMutableAttributedString *attrStr =
            [[NSMutableAttributedString alloc] initWithString:text];
        
        // เพิ่ม attributes ให้ "Hello"
        NSRange helloRange = [text rangeOfString:@"Hello"];
        
#if TARGET_OS_OSX
        // macOS attributes
        [attrStr addAttribute:NSForegroundColorAttributeName
                        value:[NSColor redColor]
                        range:helloRange];
        
        [attrStr addAttribute:NSFontAttributeName
                        value:[NSFont boldSystemFontOfSize:16]
                        range:helloRange];
        
        // เพิ่ม underline
        NSRange worldRange = [text rangeOfString:@"World"];
        [attrStr addAttribute:NSUnderlineStyleAttributeName
                        value:@(NSUnderlineStyleSingle)
                        range:worldRange];
        
        // เพิ่ม background color
        [attrStr addAttribute:NSBackgroundColorAttributeName
                        value:[NSColor yellowColor]
                        range:[text rangeOfString:@"styled"]];
#endif
        
        // ดู attributes ที่ตำแหน่งต่างๆ
        NSRange effectiveRange;
        NSDictionary *attrs = [attrStr attributesAtIndex:0
                                          effectiveRange:&effectiveRange];
        NSLog(@"Attributes at index 0: %@", attrs);
        NSLog(@"Effective range: {%lu, %lu}",
              (unsigned long)effectiveRange.location,
              (unsigned long)effectiveRange.length);
        
        // Enumerate attributes
        [attrStr enumerateAttributesInRange:NSMakeRange(0, attrStr.length)
                                    options:0
                                 usingBlock:^(NSDictionary *attrs,
                                              NSRange range,
                                              BOOL *stop) {
            if (attrs.count > 0) {
                NSLog(@"Range {%lu, %lu}: %@",
                      (unsigned long)range.location,
                      (unsigned long)range.length, attrs);
            }
        }];
        
        // สร้าง attributed string จาก HTML (iOS/macOS)
        NSString *htmlStr = @"<b>Bold</b> and <i>italic</i> and <u>underline</u>";
        NSData *htmlData = [htmlStr dataUsingEncoding:NSUTF8StringEncoding];
        NSAttributedString *fromHTML = [[NSAttributedString alloc]
            initWithData:htmlData
                 options:@{NSDocumentTypeDocumentAttribute:
                           NSHTMLTextDocumentType}
      documentAttributes:nil
                   error:nil];
        NSLog(@"From HTML: %@", fromHTML.string);
    }
    return 0;
}
```

### NSMutableAttributedString

```objc
// สร้าง rich text แบบ step-by-step
NSMutableAttributedString *richText = [[NSMutableAttributedString alloc] init];

// เพิ่ม parts ต่างๆ
NSAttributedString *title = [[NSAttributedString alloc]
    initWithString:@"ชื่อเรื่อง\n"
        attributes:@{
#if TARGET_OS_OSX
            NSFontAttributeName: [NSFont boldSystemFontOfSize:18],
            NSForegroundColorAttributeName: [NSColor blackColor]
#else
            NSFontAttributeName: [UIFont boldSystemFontOfSize:18],
            NSForegroundColorAttributeName: [UIColor blackColor]
#endif
        }];

[richText appendAttributedString:title];

NSAttributedString *body = [[NSAttributedString alloc]
    initWithString:@"นี่คือเนื้อหาของบทความ..."
        attributes:@{
#if TARGET_OS_OSX
            NSFontAttributeName: [NSFont systemFontOfSize:14],
            NSForegroundColorAttributeName: [NSColor darkGrayColor]
#else
            NSFontAttributeName: [UIFont systemFontOfSize:14],
            NSForegroundColorAttributeName: [UIColor darkGrayColor]
#endif
        }];

[richText appendAttributedString:body];

// แทนที่ substring
NSRange replaceRange = [[richText string] rangeOfString:@"เนื้อหา"];
if (replaceRange.location != NSNotFound) {
    NSAttributedString *replacement = [[NSAttributedString alloc]
        initWithString:@"เนื้อหาที่ถูกแทนที่"
            attributes:@{/* attributes */}];
    [richText replaceCharactersInRange:replaceRange
                  withAttributedString:replacement];
}
```

---

## 22.12 String to Number Conversions

```objc
#import <Foundation/Foundation.h>

int main(int argc, const char * argv[]) {
    @autoreleasepool {
        // integerValue, floatValue, doubleValue
        NSString *numStr1 = @"42";
        NSString *numStr2 = @"3.14159";
        NSString *numStr3 = @"  -100  "; // มี whitespace
        NSString *numStr4 = @"123abc"; // mixed
        NSString *invalid = @"abc";
        
        NSLog(@"integerValue: %ld", (long)[numStr1 integerValue]);   // 42
        NSLog(@"floatValue: %f", [numStr2 floatValue]);               // 3.14159
        NSLog(@"doubleValue: %f", [numStr2 doubleValue]);             // 3.14159
        NSLog(@"trim int: %ld", (long)[numStr3 integerValue]);        // -100
        NSLog(@"mixed: %ld", (long)[numStr4 integerValue]);           // 123
        NSLog(@"invalid: %ld", (long)[invalid integerValue]);         // 0
        NSLog(@"boolValue YES: %d", [@"YES" boolValue]);              // 1
        NSLog(@"boolValue true: %d", [@"true" boolValue]);            // 1
        NSLog(@"boolValue 1: %d", [@"1" boolValue]);                  // 1
        NSLog(@"boolValue no: %d", [@"no" boolValue]);                // 0
        
        // NSNumberFormatter - แม่นยำกว่า
        NSNumberFormatter *formatter = [[NSNumberFormatter alloc] init];
        formatter.numberStyle = NSNumberFormatterDecimalStyle;
        
        NSNumber *n1 = [formatter numberFromString:@"1,234,567"];
        NSLog(@"Formatted number: %@", n1); // 1234567
        
        NSNumber *n2 = [formatter numberFromString:@"invalid"];
        NSLog(@"Invalid: %@", n2 ? n2 : @"nil"); // nil
        
        // Number → String
        formatter.numberStyle = NSNumberFormatterCurrencyStyle;
        formatter.locale = [NSLocale localeWithLocaleIdentifier:@"th_TH"];
        NSString *thaiCurrency = [formatter stringFromNumber:@1234.50];
        NSLog(@"Thai currency: %@", thaiCurrency);
        
        formatter.numberStyle = NSNumberFormatterPercentStyle;
        NSString *percent = [formatter stringFromNumber:@0.75];
        NSLog(@"Percent: %@", percent);
        
        // Hex string to number
        NSString *hexStr = @"0xFF";
        unsigned int hexValue;
        [[NSScanner scannerWithString:hexStr] scanHexInt:&hexValue];
        NSLog(@"Hex value: %u", hexValue); // 255
        
        // Number to hex string
        NSInteger value = 255;
        NSString *hexOutput = [NSString stringWithFormat:@"0x%lX", (long)value];
        NSLog(@"Hex string: %@", hexOutput); // 0xFF
    }
    return 0;
}
```

---

## 22.13 NSScanner สำหรับ Parsing

```objc
#import <Foundation/Foundation.h>

int main(int argc, const char * argv[]) {
    @autoreleasepool {
        // NSScanner basics
        NSString *data = @"Name: John Doe, Age: 30, Score: 95.5";
        NSScanner *scanner = [NSScanner scannerWithString:data];
        
        // scanUpToString: scan จนเจอ delimiter
        NSString *name;
        [scanner scanUpToString:@":" intoString:nil]; // skip "Name"
        [scanner scanString:@": " intoString:nil];     // skip ": "
        [scanner scanUpToString:@"," intoString:&name]; // scan name
        NSLog(@"Name: %@", name); // "John Doe"
        
        // Parse age
        [scanner scanUpToString:@":" intoString:nil];
        [scanner scanString:@": " intoString:nil];
        NSInteger age;
        [scanner scanInteger:&age];
        NSLog(@"Age: %ld", (long)age); // 30
        
        // Parse float
        [scanner scanUpToString:@":" intoString:nil];
        [scanner scanString:@": " intoString:nil];
        double score;
        [scanner scanDouble:&score];
        NSLog(@"Score: %f", score); // 95.5
        
        // Parse CSV-like data
        NSLog(@"\n--- Parsing CSV ---");
        NSString *csvData = @"1,Alice,Engineering,75000\n2,Bob,Marketing,65000\n3,Charlie,HR,55000";
        NSArray *csvLines = [csvData componentsSeparatedByString:@"\n"];
        
        NSMutableArray *employees = [NSMutableArray array];
        for (NSString *line in csvLines) {
            NSArray *fields = [line componentsSeparatedByString:@","];
            if (fields.count >= 4) {
                NSDictionary *emp = @{
                    @"id": @([fields[0] integerValue]),
                    @"name": fields[1],
                    @"dept": fields[2],
                    @"salary": @([fields[3] doubleValue])
                };
                [employees addObject:emp];
            }
        }
        for (NSDictionary *emp in employees) {
            NSLog(@"Employee: %@ (%@) - $%.0f",
                  emp[@"name"], emp[@"dept"], [emp[@"salary"] doubleValue]);
        }
        
        // Parse log file entry
        NSLog(@"\n--- Parse Log ---");
        NSString *logEntry = @"2024-01-15 10:30:45 [ERROR] Connection timeout after 30s";
        NSScanner *logScanner = [NSScanner scannerWithString:logEntry];
        
        NSString *date, *time, *level, *message;
        [logScanner scanUpToString:@" " intoString:&date];
        [logScanner scanString:@" " intoString:nil];
        [logScanner scanUpToString:@" " intoString:&time];
        [logScanner scanString:@" " intoString:nil];
        [logScanner scanUpToString:@"]" intoString:&level]; // "[ERROR"
        level = [level substringFromIndex:1]; // remove "["
        [logScanner scanString:@"] " intoString:nil];
        message = [logEntry substringFromIndex:logScanner.scanLocation];
        
        NSLog(@"Date: %@", date);
        NSLog(@"Time: %@", time);
        NSLog(@"Level: %@", level);
        NSLog(@"Message: %@", message);
        
        // Scan hex values
        NSString *hexStr = @"0xDEADBEEF 0xCAFEBABE";
        NSScanner *hexScanner = [NSScanner scannerWithString:hexStr];
        
        while (![hexScanner isAtEnd]) {
            unsigned int hexValue;
            [hexScanner scanString:@"0x" intoString:nil];
            if ([hexScanner scanHexInt:&hexValue]) {
                NSLog(@"Hex: 0x%X = %u", hexValue, hexValue);
            }
            [hexScanner scanString:@" " intoString:nil];
        }
    }
    return 0;
}
```

---

## 22.14 Regular Expressions

```objc
#import <Foundation/Foundation.h>

int main(int argc, const char * argv[]) {
    @autoreleasepool {
        // NSRegularExpression พื้นฐาน
        NSString *text = @"Contact us at support@example.com or sales@company.org";
        
        // สร้าง regex
        NSError *error;
        NSRegularExpression *regex = [NSRegularExpression
            regularExpressionWithPattern:@"[a-zA-Z0-9._%+-]+@[a-zA-Z0-9.-]+\\.[a-zA-Z]{2,}"
                                 options:0
                                   error:&error];
        
        if (error) {
            NSLog(@"Regex error: %@", error.localizedDescription);
            return 1;
        }
        
        // หาทุก matches
        NSArray *matches = [regex matchesInString:text
                                          options:0
                                            range:NSMakeRange(0, text.length)];
        
        NSLog(@"Found %lu email(s):", (unsigned long)matches.count);
        for (NSTextCheckingResult *match in matches) {
            NSString *email = [text substringWithRange:match.range];
            NSLog(@"  Email: %@", email);
        }
        
        // ตรวจสอบว่า match หรือไม่ (validate)
        NSString *testEmail = @"user@example.com";
        NSUInteger matchCount = [regex numberOfMatchesInString:testEmail
                                                       options:0
                                                         range:NSMakeRange(0, testEmail.length)];
        NSLog(@"\n'%@' is valid email: %@",
              testEmail, matchCount > 0 ? @"YES" : @"NO");
        
        // แทนที่ด้วย regex
        NSString *phoneText = @"Call 123-456-7890 or (555) 123-4567";
        NSRegularExpression *phoneRegex = [NSRegularExpression
            regularExpressionWithPattern:@"[\\d\\-\\(\\) ]+"
                                 options:0
                                   error:nil];
        NSString *cleaned = [phoneRegex stringByReplacingMatchesInString:phoneText
                                                                  options:0
                                                                    range:NSMakeRange(0, phoneText.length)
                                                             withTemplate:@"[PHONE]"];
        NSLog(@"\nCleaned: %@", cleaned);
        
        // Capture groups
        NSString *dateText = @"Today is 2024-01-15 and tomorrow is 2024-01-16";
        NSRegularExpression *dateRegex = [NSRegularExpression
            regularExpressionWithPattern:@"(\\d{4})-(\\d{2})-(\\d{2})"
                                 options:0
                                   error:nil];
        
        NSLog(@"\nDates found:");
        NSArray *dateMatches = [dateRegex matchesInString:dateText
                                                   options:0
                                                     range:NSMakeRange(0, dateText.length)];
        for (NSTextCheckingResult *match in dateMatches) {
            NSString *fullDate = [dateText substringWithRange:[match range]];
            NSString *year = [dateText substringWithRange:[match rangeAtIndex:1]];
            NSString *month = [dateText substringWithRange:[match rangeAtIndex:2]];
            NSString *day = [dateText substringWithRange:[match rangeAtIndex:3]];
            NSLog(@"  Full: %@, Year: %@, Month: %@, Day: %@",
                  fullDate, year, month, day);
        }
        
        // First match only
        NSTextCheckingResult *firstMatch = [dateRegex firstMatchInString:dateText
                                                                  options:0
                                                                    range:NSMakeRange(0, dateText.length)];
        if (firstMatch) {
            NSLog(@"\nFirst date: %@",
                  [dateText substringWithRange:firstMatch.range]);
        }
        
        // Regex validation patterns
        NSDictionary *patterns = @{
            @"email": @"^[a-zA-Z0-9._%+-]+@[a-zA-Z0-9.-]+\\.[a-zA-Z]{2,}$",
            @"phone": @"^\\+?[\\d\\s\\-\\(\\)]{10,}$",
            @"thai": @"^[ก-๏\\s]+$",
            @"url": @"^https?://[^\\s/$.?#].[^\\s]*$"
        };
        
        NSArray *testValues = @[
            @"user@example.com",
            @"+66 81 234 5678",
            @"สวัสดีโลก",
            @"https://www.example.com"
        ];
        
        NSLog(@"\nValidation:");
        for (NSString *value in testValues) {
            for (NSString *patternName in patterns) {
                NSRegularExpression *r = [NSRegularExpression
                    regularExpressionWithPattern:patterns[patternName]
                                         options:0
                                           error:nil];
                BOOL matches2 = [r numberOfMatchesInString:value
                                                   options:0
                                                     range:NSMakeRange(0, value.length)] > 0;
                if (matches2) {
                    NSLog(@"  '%@' matches '%@'", value, patternName);
                }
            }
        }
    }
    return 0;
}
```

---

## 22.15 ตัวอย่างปฏิบัติ 15+ ตัวอย่าง

### ตัวอย่างที่ 1: String Tokenizer

```objc
@interface StringTokenizer : NSObject

- (NSArray<NSString *> *)tokenize:(NSString *)text;

@end

@implementation StringTokenizer

- (NSArray<NSString *> *)tokenize:(NSString *)text {
    NSMutableArray *tokens = [NSMutableArray array];
    NSScanner *scanner = [NSScanner scannerWithString:text];
    scanner.caseSensitive = NO;
    
    while (![scanner isAtEnd]) {
        NSString *token;
        if ([scanner scanCharactersFromSet:[NSCharacterSet letterCharacterSet]
                                intoString:&token]) {
            [tokens addObject:[token lowercaseString]];
        } else {
            [scanner setScanLocation:scanner.scanLocation + 1];
        }
    }
    
    return [tokens copy];
}

@end

// ใช้งาน
StringTokenizer *tokenizer = [[StringTokenizer alloc] init];
NSArray *tokens = [tokenizer tokenize:@"Hello, World! Hello Objective-C."];
NSLog(@"Tokens: %@", tokens);
// ["hello", "world", "hello", "objective", "c"]

// นับความถี่
NSMutableDictionary *freq = [NSMutableDictionary dictionary];
for (NSString *token in tokens) {
    freq[token] = @([freq[token] integerValue] + 1);
}
NSLog(@"Frequency: %@", freq);
```

### ตัวอย่างที่ 2: String Template Engine

```objc
@interface TemplateEngine : NSObject

- (NSString *)render:(NSString *)template withData:(NSDictionary *)data;

@end

@implementation TemplateEngine

- (NSString *)render:(NSString *)template withData:(NSDictionary *)data {
    NSError *error;
    // Match {{variable}} pattern
    NSRegularExpression *regex = [NSRegularExpression
        regularExpressionWithPattern:@"\\{\\{(\\w+)\\}\\}"
                             options:0
                               error:&error];
    
    NSMutableString *result = [template mutableCopy];
    
    // Process in reverse order to maintain indices
    NSArray *matches = [regex matchesInString:template
                                      options:0
                                        range:NSMakeRange(0, template.length)];
    
    for (NSTextCheckingResult *match in [matches reverseObjectEnumerator]) {
        NSString *varName = [template substringWithRange:[match rangeAtIndex:1]];
        NSString *value = [data[varName] description] ?: @"";
        [result replaceCharactersInRange:match.range withString:value];
    }
    
    return [result copy];
}

@end

// ใช้งาน
TemplateEngine *engine = [[TemplateEngine alloc] init];
NSString *template = @"สวัสดี {{name}}! คุณมีอายุ {{age}} ปี อยู่ที่ {{city}}";
NSDictionary *data = @{@"name": @"สมชาย", @"age": @30, @"city": @"กรุงเทพฯ"};
NSString *rendered = [engine render:template withData:data];
NSLog(@"%@", rendered);
// สวัสดี สมชาย! คุณมีอายุ 30 ปี อยู่ที่ กรุงเทพฯ
```

### ตัวอย่างที่ 3: String Validator

```objc
@interface StringValidator : NSObject

+ (BOOL)isValidEmail:(NSString *)email;
+ (BOOL)isValidThaiPhone:(NSString *)phone;
+ (BOOL)isValidPassword:(NSString *)password;
+ (NSString *)passwordStrength:(NSString *)password;

@end

@implementation StringValidator

+ (BOOL)isValidEmail:(NSString *)email {
    NSString *pattern = @"^[a-zA-Z0-9._%+-]+@[a-zA-Z0-9.-]+\\.[a-zA-Z]{2,}$";
    NSPredicate *pred = [NSPredicate predicateWithFormat:@"SELF MATCHES %@", pattern];
    return [pred evaluateWithObject:email];
}

+ (BOOL)isValidThaiPhone:(NSString *)phone {
    // Thai mobile: 08x, 09x, 06x; landline: 02x
    NSString *cleaned = [phone stringByReplacingOccurrencesOfString:@"[-\\s\\(\\)]"
                                                         withString:@""
                                                            options:NSRegularExpressionSearch
                                                              range:NSMakeRange(0, phone.length)];
    NSString *pattern = @"^(0[689]\\d{8}|02\\d{7})$";
    NSPredicate *pred = [NSPredicate predicateWithFormat:@"SELF MATCHES %@", pattern];
    return [pred evaluateWithObject:cleaned];
}

+ (BOOL)isValidPassword:(NSString *)password {
    // อย่างน้อย 8 ตัว, มีตัวพิมพ์เล็ก, ใหญ่, ตัวเลข
    if (password.length < 8) return NO;
    
    NSCharacterSet *upper = [NSCharacterSet uppercaseLetterCharacterSet];
    NSCharacterSet *lower = [NSCharacterSet lowercaseLetterCharacterSet];
    NSCharacterSet *digits = [NSCharacterSet decimalDigitCharacterSet];
    
    BOOL hasUpper = [password rangeOfCharacterFromSet:upper].location != NSNotFound;
    BOOL hasLower = [password rangeOfCharacterFromSet:lower].location != NSNotFound;
    BOOL hasDigit = [password rangeOfCharacterFromSet:digits].location != NSNotFound;
    
    return hasUpper && hasLower && hasDigit;
}

+ (NSString *)passwordStrength:(NSString *)password {
    NSInteger score = 0;
    if (password.length >= 8) score++;
    if (password.length >= 12) score++;
    
    NSCharacterSet *upper = [NSCharacterSet uppercaseLetterCharacterSet];
    NSCharacterSet *lower = [NSCharacterSet lowercaseLetterCharacterSet];
    NSCharacterSet *digits = [NSCharacterSet decimalDigitCharacterSet];
    NSCharacterSet *special = [NSCharacterSet punctuationCharacterSet];
    
    if ([password rangeOfCharacterFromSet:upper].location != NSNotFound) score++;
    if ([password rangeOfCharacterFromSet:lower].location != NSNotFound) score++;
    if ([password rangeOfCharacterFromSet:digits].location != NSNotFound) score++;
    if ([password rangeOfCharacterFromSet:special].location != NSNotFound) score++;
    
    if (score <= 2) return @"Weak";
    if (score <= 4) return @"Medium";
    return @"Strong";
}

@end

// ใช้งาน
NSArray *emails = @[@"user@example.com", @"invalid.email", @"test@.com"];
for (NSString *email in emails) {
    NSLog(@"%@: %@", email,
          [StringValidator isValidEmail:email] ? @"valid" : @"invalid");
}

NSArray *passwords = @[@"pass", @"Password1", @"P@ssw0rd!123"];
for (NSString *pwd in passwords) {
    NSLog(@"%@: %@", pwd, [StringValidator passwordStrength:pwd]);
}
```

### ตัวอย่างที่ 4: String Formatter

```objc
@interface StringFormatter : NSObject

+ (NSString *)formatPhoneNumber:(NSString *)phone locale:(NSString *)locale;
+ (NSString *)formatCurrency:(double)amount locale:(NSString *)locale;
+ (NSString *)formatDate:(NSDate *)date format:(NSString *)format;
+ (NSString *)truncate:(NSString *)text maxLength:(NSUInteger)maxLength;
+ (NSString *)wordWrap:(NSString *)text width:(NSUInteger)width;

@end

@implementation StringFormatter

+ (NSString *)formatPhoneNumber:(NSString *)phone locale:(NSString *)locale {
    // ลบ non-digit
    NSMutableString *digits = [NSMutableString string];
    for (NSUInteger i = 0; i < phone.length; i++) {
        unichar c = [phone characterAtIndex:i];
        if (isdigit(c)) [digits appendFormat:@"%c", c];
    }
    
    if ([locale isEqualToString:@"TH"] && digits.length == 10) {
        // Thai format: 0xx-xxx-xxxx
        return [NSString stringWithFormat:@"%@-%@-%@",
                [digits substringWithRange:NSMakeRange(0, 3)],
                [digits substringWithRange:NSMakeRange(3, 3)],
                [digits substringWithRange:NSMakeRange(6, 4)]];
    }
    
    return phone; // ส่งคืน original ถ้าไม่รู้ format
}

+ (NSString *)formatCurrency:(double)amount locale:(NSString *)locale {
    NSNumberFormatter *formatter = [[NSNumberFormatter alloc] init];
    formatter.numberStyle = NSNumberFormatterCurrencyStyle;
    formatter.locale = [NSLocale localeWithLocaleIdentifier:locale];
    return [formatter stringFromNumber:@(amount)];
}

+ (NSString *)formatDate:(NSDate *)date format:(NSString *)format {
    NSDateFormatter *formatter = [[NSDateFormatter alloc] init];
    formatter.dateFormat = format;
    formatter.locale = [NSLocale currentLocale];
    return [formatter stringFromDate:date];
}

+ (NSString *)truncate:(NSString *)text maxLength:(NSUInteger)maxLength {
    if (text.length <= maxLength) return text;
    NSString *truncated = [text substringToIndex:maxLength - 3];
    return [truncated stringByAppendingString:@"..."];
}

+ (NSString *)wordWrap:(NSString *)text width:(NSUInteger)width {
    NSArray *words = [text componentsSeparatedByString:@" "];
    NSMutableArray *lines = [NSMutableArray array];
    NSMutableString *currentLine = [NSMutableString string];
    
    for (NSString *word in words) {
        if (currentLine.length == 0) {
            [currentLine appendString:word];
        } else if (currentLine.length + 1 + word.length <= width) {
            [currentLine appendFormat:@" %@", word];
        } else {
            [lines addObject:[currentLine copy]];
            currentLine = [word mutableCopy];
        }
    }
    
    if (currentLine.length > 0) {
        [lines addObject:[currentLine copy]];
    }
    
    return [lines componentsJoinedByString:@"\n"];
}

@end

// ใช้งาน
NSLog(@"%@", [StringFormatter formatPhoneNumber:@"0812345678" locale:@"TH"]);
// 081-234-5678

NSLog(@"%@", [StringFormatter formatCurrency:1234567.89 locale:@"th_TH"]);
// ฿1,234,567.89

NSLog(@"%@", [StringFormatter formatDate:[NSDate date] format:@"dd/MM/yyyy"]);

NSString *longText = @"This is a very long text that needs to be truncated";
NSLog(@"%@", [StringFormatter truncate:longText maxLength:20]);
// This is a very lo...

NSString *paragraph = @"The quick brown fox jumps over the lazy dog in the forest";
NSLog(@"%@", [StringFormatter wordWrap:paragraph width:20]);
```

### ตัวอย่างที่ 5: Advanced String Parser

```objc
// Parse key-value pairs จาก config string
NSString *config = @"host=localhost;port=5432;database=mydb;user=admin;password=secret";
NSMutableDictionary *configDict = [NSMutableDictionary dictionary];

NSArray *pairs = [config componentsSeparatedByString:@";"];
for (NSString *pair in pairs) {
    NSArray *parts = [pair componentsSeparatedByString:@"="];
    if (parts.count == 2) {
        configDict[parts[0]] = parts[1];
    }
}
NSLog(@"Config: %@", configDict);

// Build connection string กลับ
NSMutableArray *rebuilt = [NSMutableArray array];
for (NSString *key in @[@"host", @"port", @"database"]) {
    if (configDict[key]) {
        [rebuilt addObject:[NSString stringWithFormat:@"%@=%@", key, configDict[key]]];
    }
}
NSLog(@"Connection: %@", [rebuilt componentsJoinedByString:@";"]);
```

---

## 22.16 Best Practices สำหรับ NSString

### 1. ใช้ isEqualToString: แทน isEqual:

```objc
// ดีกว่า - type-specific, เร็วกว่า
if ([str1 isEqualToString:str2]) { ... }

// หลีกเลี่ยง - ต้อง cast type ก่อน
if ([str1 isEqual:str2]) { ... }
```

### 2. ตรวจสอบ nil และ empty

```objc
// ตรวจสอบทั้ง nil และ empty
if (str == nil || str.length == 0) {
    NSLog(@"String is nil or empty");
}

// สั้นกว่า
if (!str.length) {
    NSLog(@"nil or empty");
}

// Helper macro
#define IsNilOrEmpty(s) ((s) == nil || (s).length == 0)
```

### 3. ใช้ NSMutableString สำหรับการต่อ string หลายครั้ง

```objc
// ช้า - สร้าง string ใหม่ทุกครั้ง
NSString *result = @"";
for (int i = 0; i < 1000; i++) {
    result = [result stringByAppendingFormat:@"%d", i]; // ช้า!
}

// เร็วกว่า - ใช้ NSMutableString
NSMutableString *mutableResult = [NSMutableString stringWithCapacity:5000];
for (int i = 0; i < 1000; i++) {
    [mutableResult appendFormat:@"%d", i]; // เร็วกว่ามาก
}
```

### 4. ใช้ localizedStringWithFormat: สำหรับ user-facing strings

```objc
// สำหรับแสดงผลต่อ user
NSString *message = [NSString localizedStringWithFormat:
                     NSLocalizedString(@"Hello, %@!", nil), userName];

// สำหรับ internal use (logging, etc.)
NSString *log = [NSString stringWithFormat:@"User %@ logged in", userName];
```

---

## สรุป

NSString ใน Objective-C มีความสามารถที่หลากหลายและทรงพลัง:

1. **สร้าง strings** ได้หลายวิธี ตั้งแต่ literal ไปจนถึง format
2. **Compare** ด้วย options ต่างๆ เช่น case-insensitive, numeric
3. **Range operations** สำหรับ substring extraction
4. **Split และ Join** ด้วย components methods
5. **Regular expressions** สำหรับ pattern matching
6. **NSAttributedString** สำหรับ rich text
7. **Encoding** สำหรับ URL, HTML, Base64
8. **NSScanner** สำหรับ parsing complex strings

ในบทต่อไปเราจะเรียนรู้เกี่ยวกับ **NSArray และ NSMutableArray** ซึ่งเป็น collection ที่ใช้บ่อยที่สุดใน Objective-C
