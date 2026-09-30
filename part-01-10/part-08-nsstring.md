# ตอนที่ 8: NSString - การจัดการข้อความใน Objective-C

## บทนำ

NSString เป็นคลาสพื้นฐานที่สำคัญที่สุดคลาสหนึ่งใน Objective-C สำหรับการจัดการข้อความ (string) มีฟีเจอร์มากมายตั้งแต่การสร้าง ค้นหา แก้ไข จนถึงการจัดรูปแบบข้อความ บทนี้จะครอบคลุมทุกแง่มุมของ NSString อย่างละเอียด

---

## 8.1 การสร้าง NSString

### วิธีการสร้าง NSString แบบต่างๆ

```objc
#import <Foundation/Foundation.h>

int main(int argc, const char * argv[]) {
    @autoreleasepool {
        
        // วิธีที่ 1: String literal (แนะนำที่สุด)
        NSString *str1 = @"สวัสดีชาวโลก";
        
        // วิธีที่ 2: stringWithFormat: (เหมือน printf)
        NSString *name = @"สมชาย";
        int age = 25;
        NSString *str2 = [NSString stringWithFormat:@"ชื่อ: %@, อายุ: %d", name, age];
        
        // วิธีที่ 3: initWithString: (copy string)
        NSString *str3 = [[NSString alloc] initWithString:@"Hello"];
        
        // วิธีที่ 4: จาก C string
        const char *cStr = "Hello from C string";
        NSString *str4 = [NSString stringWithUTF8String:cStr];
        NSString *str5 = [[NSString alloc] initWithCString:cStr encoding:NSUTF8StringEncoding];
        
        // วิธีที่ 5: จากไฟล์
        // NSString *str6 = [NSString stringWithContentsOfFile:@"/path/to/file"
        //                                           encoding:NSUTF8StringEncoding
        //                                              error:nil];
        
        // วิธีที่ 6: จาก Data
        NSData *data = [@"ข้อมูล" dataUsingEncoding:NSUTF8StringEncoding];
        NSString *str7 = [[NSString alloc] initWithData:data encoding:NSUTF8StringEncoding];
        
        // วิธีที่ 7: string ว่าง
        NSString *empty = @"";
        NSString *empty2 = [NSString string];
        
        NSLog(@"str1: %@", str1);
        NSLog(@"str2: %@", str2);
        NSLog(@"str3: %@", str3);
        NSLog(@"str4: %@", str4);
        NSLog(@"str7: %@", str7);
        NSLog(@"ว่างเปล่า: '%@'", empty);
        NSLog(@"ความยาว str1: %lu", str1.length);
        
        // ตรวจสอบ string ว่าง
        if ([empty length] == 0) {
            NSLog(@"\n'empty' เป็น string ว่าง");
        }
        
        // NSLog format specifiers
        NSString *formatted = [NSString stringWithFormat:
                               @"Int: %d\nFloat: %.2f\nString: %@\nChar: %c\nBool: %d",
                               42, 3.14, @"Hello", 'X', YES];
        NSLog(@"\n%@", formatted);
    }
    return 0;
}
```

### String Format Specifiers

```objc
#import <Foundation/Foundation.h>

int main(int argc, const char * argv[]) {
    @autoreleasepool {
        
        // Format specifiers ที่ใช้บ่อย
        int i = 42;
        long l = 1234567890L;
        float f = 3.14159f;
        double d = 2.71828;
        NSString *s = @"Hello";
        BOOL b = YES;
        char c = 'A';
        unsigned int u = 255;
        
        NSLog(@"%%d  (int): %d", i);
        NSLog(@"%%ld (long): %ld", l);
        NSLog(@"%%f  (float): %f", f);
        NSLog(@"%%.2f (float 2 decimal): %.2f", f);
        NSLog(@"%%e  (scientific): %e", d);
        NSLog(@"%%@  (NSObject): %@", s);
        NSLog(@"%%i  (int): %i", i);
        NSLog(@"%%u  (unsigned): %u", u);
        NSLog(@"%%x  (hex): %x", u);
        NSLog(@"%%X  (HEX): %X", u);
        NSLog(@"%%o  (octal): %o", u);
        NSLog(@"%%c  (char): %c", c);
        NSLog(@"%%s  (C string): %s", "C string");
        NSLog(@"%%p  (pointer): %p", &i);
        NSLog(@"%%@  (BOOL): %@", b ? @"YES" : @"NO");
        
        // Width and padding
        NSLog(@"\n=== การจัดรูปแบบ ===");
        NSLog(@"[%10d]  (width 10, right align)", i);
        NSLog(@"[%-10d]  (width 10, left align)", i);
        NSLog(@"[%010d]  (zero pad)", i);
        NSLog(@"[%+d]    (show sign)", i);
        NSLog(@"[%.5f]  (5 decimals)", d);
        NSLog(@"[%10.2f]  (width 10, 2 dec)", d);
    }
    return 0;
}
```

---

## 8.2 NSMutableString

```objc
#import <Foundation/Foundation.h>

int main(int argc, const char * argv[]) {
    @autoreleasepool {
        
        // สร้าง NSMutableString
        NSMutableString *mStr = [NSMutableString string];
        NSMutableString *mStr2 = [NSMutableString stringWithString:@"Hello"];
        NSMutableString *mStr3 = [@"World" mutableCopy];
        
        NSLog(@"=== NSMutableString ===");
        
        // Append string
        [mStr appendString:@"สวัสดี"];
        [mStr appendString:@" "];
        [mStr appendString:@"ชาวโลก"];
        NSLog(@"หลัง append: %@", mStr);
        
        // Append with format
        int count = 100;
        [mStr appendFormat:@" (จำนวน %d คน)", count];
        NSLog(@"หลัง appendFormat: %@", mStr);
        
        // Insert string
        [mStr2 insertString:@", World" atIndex:5];
        NSLog(@"\nhลัง insert: %@", mStr2);
        
        // Delete characters
        NSMutableString *test = [NSMutableString stringWithString:@"Hello, World!"];
        [test deleteCharactersInRange:NSMakeRange(5, 7)];  // ลบ ", World"
        NSLog(@"\nหลัง delete: %@", test);
        
        // Replace string
        NSMutableString *replace = [NSMutableString stringWithString:
                                    @"I love cats and cats are great"];
        [replace replaceOccurrencesOfString:@"cats"
                                 withString:@"dogs"
                                    options:NSCaseInsensitiveSearch
                                      range:NSMakeRange(0, replace.length)];
        NSLog(@"\nหลัง replace: %@", replace);
        
        // Set string ใหม่ทั้งหมด
        [mStr2 setString:@"Brand new string"];
        NSLog(@"\nหลัง setString: %@", mStr2);
        
        // String building pattern
        NSMutableString *html = [NSMutableString string];
        [html appendString:@"<html>\n"];
        [html appendString:@"  <head>\n"];
        [html appendFormat:@"    <title>%@</title>\n", @"My Page"];
        [html appendString:@"  </head>\n"];
        [html appendString:@"  <body>\n"];
        [html appendFormat:@"    <h1>%@</h1>\n", @"สวัสดีโลก"];
        [html appendString:@"  </body>\n"];
        [html appendString:@"</html>"];
        
        NSLog(@"\n=== HTML ที่สร้าง ===\n%@", html);
    }
    return 0;
}
```

---

## 8.3 การเปรียบเทียบ String

```objc
#import <Foundation/Foundation.h>

int main(int argc, const char * argv[]) {
    @autoreleasepool {
        
        NSString *str1 = @"Hello";
        NSString *str2 = @"Hello";
        NSString *str3 = @"hello";  // lowercase
        NSString *str4 = @"World";
        
        NSLog(@"=== การเปรียบเทียบ String ===");
        
        // isEqualToString: - เปรียบเทียบเนื้อหา (case sensitive)
        NSLog(@"'Hello' == 'Hello': %@",
              [str1 isEqualToString:str2] ? @"YES" : @"NO");
        NSLog(@"'Hello' == 'hello': %@",
              [str1 isEqualToString:str3] ? @"YES" : @"NO");
        
        // == เปรียบเทียบ pointer (ไม่แนะนำสำหรับ value comparison)
        NSLog(@"\nstr1 == str2 (pointer): %@",
              (str1 == str2) ? @"YES" : @"NO");  // อาจเป็น YES เพราะ interning
        
        // compare: - เปรียบเทียบตาม alphabetical order
        NSLog(@"\n=== compare: ===");
        NSComparisonResult result = [str1 compare:str4];
        switch (result) {
            case NSOrderedAscending:
                NSLog(@"'%@' มาก่อน '%@'", str1, str4);
                break;
            case NSOrderedDescending:
                NSLog(@"'%@' มาหลัง '%@'", str1, str4);
                break;
            case NSOrderedSame:
                NSLog(@"'%@' เท่ากับ '%@'", str1, str4);
                break;
        }
        
        // Case insensitive compare
        NSLog(@"\n=== Case Insensitive ===");
        NSComparisonResult ciResult = [str1 compare:str3
                                            options:NSCaseInsensitiveSearch];
        NSLog(@"'Hello' vs 'hello' (case insensitive): %s",
              ciResult == NSOrderedSame ? "เท่ากัน" : "ต่างกัน");
        
        // localizedCompare: - เปรียบเทียบตาม locale ของผู้ใช้
        NSArray *names = @[@"Charlie", @"alice", @"Bob", @"DAVID"];
        NSArray *sorted = [names sortedArrayUsingSelector:
                           @selector(localizedCaseInsensitiveCompare:)];
        NSLog(@"\nชื่อเรียง (locale): %@", sorted);
        
        // hasPrefix: และ hasSuffix:
        NSString *url = @"https://www.example.com/path?query=value";
        NSLog(@"\n=== hasPrefix / hasSuffix ===");
        NSLog(@"ขึ้นต้นด้วย 'https': %@",
              [url hasPrefix:@"https"] ? @"YES" : @"NO");
        NSLog(@"ลงท้ายด้วย '.com': %@",
              [url hasSuffix:@".com"] ? @"YES" : @"NO");
        NSLog(@"ลงท้ายด้วย 'value': %@",
              [url hasSuffix:@"value"] ? @"YES" : @"NO");
        
        // เปรียบเทียบ numeric
        NSLog(@"\n=== Numeric Compare ===");
        NSString *num1 = @"10";
        NSString *num2 = @"9";
        // ตัวอักษร '1' < '9' ดังนั้น "10" < "9" ถ้าเปรียบ string ล้วน
        NSLog(@"String compare '10' vs '9': %s",
              [num1 compare:num2] == NSOrderedAscending ? "10 < 9" : "10 >= 9");
        // ใช้ NSNumericSearch เพื่อเปรียบตัวเลข
        NSLog(@"Numeric compare '10' vs '9': %s",
              [num1 compare:num2 options:NSNumericSearch] == NSOrderedDescending
              ? "10 > 9" : "10 <= 9");
    }
    return 0;
}
```

---

## 8.4 การค้นหาใน String

```objc
#import <Foundation/Foundation.h>

int main(int argc, const char * argv[]) {
    @autoreleasepool {
        
        NSString *text = @"The quick brown fox jumps over the lazy dog";
        
        NSLog(@"ข้อความ: %@\n", text);
        
        // containsString: - ตรวจสอบว่ามี substring หรือไม่
        NSLog(@"=== containsString: ===");
        NSLog(@"มี 'fox': %@",
              [text containsString:@"fox"] ? @"YES" : @"NO");
        NSLog(@"มี 'cat': %@",
              [text containsString:@"cat"] ? @"YES" : @"NO");
        
        // rangeOfString: - หาตำแหน่งของ substring
        NSLog(@"\n=== rangeOfString: ===");
        NSRange range = [text rangeOfString:@"fox"];
        if (range.location != NSNotFound) {
            NSLog(@"พบ 'fox' ที่ตำแหน่ง %lu, ความยาว %lu",
                  range.location, range.length);
        }
        
        // ไม่พบ
        NSRange notFound = [text rangeOfString:@"cat"];
        NSLog(@"ค้นหา 'cat': %s",
              notFound.location == NSNotFound ? "ไม่พบ" : "พบ");
        
        // Case insensitive search
        NSRange ciRange = [text rangeOfString:@"THE"
                                      options:NSCaseInsensitiveSearch];
        NSLog(@"\n'THE' (case insensitive) ตำแหน่ง: %lu", ciRange.location);
        
        // ค้นหาจากท้าย
        NSString *repeated = @"cat dog cat bird cat";
        NSRange backRange = [repeated rangeOfString:@"cat"
                                            options:NSBackwardsSearch];
        NSLog(@"\n'cat' ตำแหน่งสุดท้ายใน '%@': %lu",
              repeated, backRange.location);
        
        // ค้นหาทุกตำแหน่ง
        NSLog(@"\n=== ค้นหาทุกตำแหน่งของ 'cat' ===");
        NSString *searchStr = @"cat";
        NSRange searchRange = NSMakeRange(0, [repeated length]);
        NSUInteger foundCount = 0;
        
        while (searchRange.location < [repeated length]) {
            NSRange found = [repeated rangeOfString:searchStr
                                            options:0
                                              range:searchRange];
            if (found.location == NSNotFound) break;
            
            foundCount++;
            NSLog(@"พบ '%@' ที่ตำแหน่ง %lu", searchStr, found.location);
            
            searchRange.location = found.location + found.length;
            searchRange.length = [repeated length] - searchRange.location;
        }
        NSLog(@"พบทั้งหมด %lu ครั้ง", foundCount);
        
        // rangeOfCharacterFromSet:
        NSLog(@"\n=== rangeOfCharacterFromSet: ===");
        NSCharacterSet *vowels = [NSCharacterSet characterSetWithCharactersInString:@"aeiouAEIOU"];
        NSRange firstVowel = [text rangeOfCharacterFromSet:vowels];
        NSLog(@"สระแรกใน text: '%@' ที่ตำแหน่ง %lu",
              [text substringWithRange:firstVowel], firstVowel.location);
        
        // นับ occurrences
        NSString *countText = @"banana";
        NSString *countChar = @"a";
        NSUInteger occurrences = 0;
        NSRange countRange = NSMakeRange(0, [countText length]);
        
        while (YES) {
            NSRange found = [countText rangeOfString:countChar
                                             options:0
                                               range:countRange];
            if (found.location == NSNotFound) break;
            occurrences++;
            countRange.location = found.location + found.length;
            countRange.length = [countText length] - countRange.location;
        }
        NSLog(@"\nจำนวน 'a' ใน 'banana': %lu", occurrences);
    }
    return 0;
}
```

---

## 8.5 การตัด Substring

```objc
#import <Foundation/Foundation.h>

int main(int argc, const char * argv[]) {
    @autoreleasepool {
        
        NSString *str = @"Hello, World! How are you?";
        NSLog(@"String เดิม: %@\n", str);
        
        // substringFromIndex: - ตัดจาก index ไปจนสุด
        NSString *fromIndex7 = [str substringFromIndex:7];
        NSLog(@"substringFromIndex:7 → '%@'", fromIndex7);
        
        // substringToIndex: - ตัดจากต้นไปถึง index
        NSString *toIndex5 = [str substringToIndex:5];
        NSLog(@"substringToIndex:5 → '%@'", toIndex5);
        
        // substringWithRange: - ตัดตาม range
        NSRange range = NSMakeRange(7, 5);  // เริ่มที่ 7, ยาว 5
        NSString *sub = [str substringWithRange:range];
        NSLog(@"substringWithRange:(7,5) → '%@'", sub);
        
        // ตัดด้วย componentsSeparatedByString:
        NSString *csv = @"John,25,Engineer,Bangkok";
        NSArray *parts = [csv componentsSeparatedByString:@","];
        NSLog(@"\nแยก CSV:");
        for (int i = 0; i < parts.count; i++) {
            NSLog(@"  [%d]: %@", i, parts[i]);
        }
        
        // แยกด้วย CharacterSet
        NSString *sentence = @"Hello World  Test   Multiple";
        NSArray *words = [sentence componentsSeparatedByCharactersInSet:
                         [NSCharacterSet whitespaceCharacterSet]];
        NSLog(@"\nแยกด้วย space: %@", words);
        
        // กรองค่าว่างออก
        NSArray *nonEmpty = [words filteredArrayUsingPredicate:
                            [NSPredicate predicateWithFormat:@"length > 0"]];
        NSLog(@"กรองค่าว่างออก: %@", nonEmpty);
        
        // หา substring ระหว่าง patterns
        NSString *html = @"<title>My Page Title</title>";
        NSRange startRange = [html rangeOfString:@"<title>"];
        NSRange endRange = [html rangeOfString:@"</title>"];
        
        if (startRange.location != NSNotFound && endRange.location != NSNotFound) {
            NSUInteger start = startRange.location + startRange.length;
            NSUInteger length = endRange.location - start;
            NSString *title = [html substringWithRange:NSMakeRange(start, length)];
            NSLog(@"\nTitle tag content: %@", title);
        }
        
        // ตัด prefix และ suffix
        NSString *email = @"user@example.com";
        if ([email hasSuffix:@".com"]) {
            NSString *withoutCom = [email substringToIndex:email.length - 4];
            NSLog(@"\nEmail ไม่มี .com: %@", withoutCom);
        }
        
        // ดึงตัวอักษรทีละตัว
        NSString *word = @"Hello";
        NSLog(@"\nตัวอักษรแต่ละตัวใน '%@':", word);
        for (NSUInteger i = 0; i < word.length; i++) {
            unichar c = [word characterAtIndex:i];
            NSLog(@"  [%lu]: %C (ASCII: %d)", i, c, c);
        }
    }
    return 0;
}
```

---

## 8.6 String Manipulation

### Case Conversion และ Trimming

```objc
#import <Foundation/Foundation.h>

int main(int argc, const char * argv[]) {
    @autoreleasepool {
        
        NSString *mixed = @"Hello World! This Is A Test.";
        NSLog(@"ต้นฉบับ: %@\n", mixed);
        
        // Case conversion
        NSLog(@"=== Case Conversion ===");
        NSLog(@"uppercase: %@", [mixed uppercaseString]);
        NSLog(@"lowercase: %@", [mixed lowercaseString]);
        NSLog(@"capitalized: %@", [mixed capitalizedString]);
        
        // Locale-aware
        NSLog(@"\nlocale uppercase: %@",
              [mixed uppercaseStringWithLocale:[NSLocale currentLocale]]);
        
        // Trimming
        NSLog(@"\n=== Trimming ===");
        NSString *padded = @"   Hello World!   ";
        NSLog(@"ก่อน trim: '%@'", padded);
        
        NSString *trimmed = [padded stringByTrimmingCharactersInSet:
                            [NSCharacterSet whitespaceCharacterSet]];
        NSLog(@"หลัง trim (whitespace): '%@'", trimmed);
        
        // Trim whitespace and newlines
        NSString *withNewlines = @"\n\n  Hello\n  World  \n\n";
        NSString *trimmedNL = [withNewlines stringByTrimmingCharactersInSet:
                              [NSCharacterSet whitespaceAndNewlineCharacterSet]];
        NSLog(@"\nหลัง trim (whitespace+newline): '%@'", trimmedNL);
        
        // Trim custom characters
        NSString *dashes = @"---Hello---";
        NSString *noDashes = [dashes stringByTrimmingCharactersInSet:
                             [NSCharacterSet characterSetWithCharactersInString:@"-"]];
        NSLog(@"\nหลัง trim '-': %@", noDashes);
        
        // Replace string
        NSLog(@"\n=== String Replacement ===");
        NSString *original = @"I have a cat, you have a cat, we all have cats";
        NSString *replaced = [original stringByReplacingOccurrencesOfString:@"cat"
                                                                 withString:@"dog"];
        NSLog(@"Replace 'cat' with 'dog': %@", replaced);
        
        // Replace ใน range เฉพาะ
        NSString *partialReplace = [original stringByReplacingOccurrencesOfString:@"cat"
                                                                        withString:@"dog"
                                                                           options:0
                                                                             range:NSMakeRange(0, 20)];
        NSLog(@"Replace (partial, first 20): %@", partialReplace);
        
        // Replace characters in set
        NSString *mixed2 = @"Hello 123 World 456!";
        NSMutableString *result = [NSMutableString stringWithString:mixed2];
        for (NSUInteger i = result.length; i > 0; i--) {
            unichar c = [result characterAtIndex:i - 1];
            if (isdigit(c)) {
                [result deleteCharactersInRange:NSMakeRange(i - 1, 1)];
            }
        }
        NSLog(@"\nลบตัวเลขออก: %@", result);
        
        // Padding (ไม่มี built-in ต้องทำเอง)
        NSString *short_str = @"Hello";
        NSInteger targetLength = 10;
        NSInteger padding = targetLength - (NSInteger)short_str.length;
        NSString *padRight = [short_str stringByPaddingToLength:targetLength
                                                     withString:@" "
                                                startingAtIndex:0];
        NSLog(@"\npad right: '[%@]'", padRight);
        
        NSString *padLeft = [@"" stringByPaddingToLength:padding
                                              withString:@" "
                                         startingAtIndex:0];
        padLeft = [padLeft stringByAppendingString:short_str];
        NSLog(@"pad left:  '[%@]'", padLeft);
    }
    return 0;
}
```

---

## 8.7 String Conversion

```objc
#import <Foundation/Foundation.h>

int main(int argc, const char * argv[]) {
    @autoreleasepool {
        
        NSLog(@"=== String to Number ===");
        
        NSString *intStr = @"42";
        NSString *floatStr = @"3.14159";
        NSString *boolStr = @"YES";
        NSString *invalidStr = @"abc";
        
        // ตัวเลข
        NSInteger intVal = [intStr integerValue];
        int intVal2 = [intStr intValue];
        long long llVal = [intStr longLongValue];
        double dblVal = [floatStr doubleValue];
        float fltVal = [floatStr floatValue];
        
        NSLog(@"integerValue: %ld", (long)intVal);
        NSLog(@"intValue: %d", intVal2);
        NSLog(@"longLongValue: %lld", llVal);
        NSLog(@"doubleValue: %.5f", dblVal);
        NSLog(@"floatValue: %.5f", fltVal);
        NSLog(@"boolValue: %d", [boolStr boolValue]);
        
        // invalid string คืน 0
        NSLog(@"\n'abc' integerValue: %ld", [invalidStr integerValue]);
        
        // NSScanner - parse ที่ซับซ้อนกว่า
        NSLog(@"\n=== NSScanner ===");
        NSString *data = @"Price: $99.99 Units: 5";
        NSScanner *scanner = [NSScanner scannerWithString:data];
        
        [scanner scanUpToString:@"$" intoString:nil];
        [scanner scanString:@"$" intoString:nil];
        double price;
        [scanner scanDouble:&price];
        NSLog(@"Price: %.2f", price);
        
        [scanner scanUpToString:@"Units: " intoString:nil];
        [scanner scanString:@"Units: " intoString:nil];
        int units;
        [scanner scanInt:&units];
        NSLog(@"Units: %d", units);
        
        // Number to String
        NSLog(@"\n=== Number to String ===");
        int num = 42;
        float fnum = 3.14f;
        
        NSString *fromInt = [NSString stringWithFormat:@"%d", num];
        NSString *fromFloat = [NSString stringWithFormat:@"%.2f", fnum];
        NSString *fromNSNumber = [@(num) stringValue];
        
        NSLog(@"int to string: %@", fromInt);
        NSLog(@"float to string: %@", fromFloat);
        NSLog(@"NSNumber to string: %@", fromNSNumber);
        
        // NSNumberFormatter - จัดรูปแบบตัวเลข
        NSLog(@"\n=== NSNumberFormatter ===");
        NSNumberFormatter *formatter = [[NSNumberFormatter alloc] init];
        formatter.numberStyle = NSNumberFormatterDecimalStyle;
        formatter.groupingSeparator = @",";
        formatter.minimumFractionDigits = 2;
        formatter.maximumFractionDigits = 2;
        
        NSString *formatted = [formatter stringFromNumber:@1234567.89];
        NSLog(@"Formatted: %@", formatted);
        
        // Currency format
        NSNumberFormatter *currencyFmt = [[NSNumberFormatter alloc] init];
        currencyFmt.numberStyle = NSNumberFormatterCurrencyStyle;
        currencyFmt.currencyCode = @"THB";
        NSString *price_str = [currencyFmt stringFromNumber:@99999.99];
        NSLog(@"Currency (THB): %@", price_str);
        
        // Parse ตัวเลขจาก formatted string
        NSNumber *parsed = [formatter numberFromString:@"1,234,567.89"];
        NSLog(@"\nParsed: %@", parsed);
        
        // Convert to C string
        NSLog(@"\n=== to C String ===");
        NSString *nsStr = @"Hello World";
        const char *cStr = [nsStr UTF8String];
        NSLog(@"C string: %s", cStr);
        NSLog(@"Length (C): %lu", strlen(cStr));
        
        // NSURL conversion
        NSString *urlString = @"https://www.example.com/path?q=hello world";
        NSString *encoded = [urlString stringByAddingPercentEncodingWithAllowedCharacters:
                            [NSCharacterSet URLQueryAllowedCharacterSet]];
        NSLog(@"\nURL encoded: %@", encoded);
    }
    return 0;
}
```

---

## 8.8 Unicode Support

```objc
#import <Foundation/Foundation.h>

int main(int argc, const char * argv[]) {
    @autoreleasepool {
        
        NSLog(@"=== Unicode Support ===");
        
        // Thai text
        NSString *thai = @"สวัสดีชาวโลก";
        NSLog(@"ภาษาไทย: %@", thai);
        NSLog(@"ความยาว (characters): %lu", thai.length);  // นับ UTF-16 code units
        
        // Emoji
        NSString *emoji = @"Hello 👋 World 🌍";
        NSLog(@"\nEmoji: %@", emoji);
        NSLog(@"ความยาว (UTF-16): %lu", emoji.length);  // อาจไม่ตรงกับจำนวนที่เห็น
        
        // Unicode escape
        NSString *heart = @"♥";   // ♥
        NSString *star = @"★";    // ★
        NSString *arrow = @"→";   // →
        NSLog(@"\nSymbols: %@ %@ %@", heart, star, arrow);
        
        // Chinese, Japanese, Korean
        NSString *chinese = @"你好世界";
        NSString *japanese = @"こんにちは世界";
        NSString *korean = @"안녕하세요 세계";
        NSLog(@"\nChinese: %@", chinese);
        NSLog(@"Japanese: %@", japanese);
        NSLog(@"Korean: %@", korean);
        
        // enumerate ด้วย composed character sequences
        NSLog(@"\n=== Enumerate Characters ===");
        NSString *testStr = @"Hello";
        [testStr enumerateSubstringsInRange:NSMakeRange(0, testStr.length)
                                    options:NSStringEnumerationByComposedCharacterSequences
                                 usingBlock:^(NSString *substring, NSRange substringRange,
                                              NSRange enclosingRange, BOOL *stop) {
            NSLog(@"ตัวอักษร: %@ (index: %lu)", substring, substringRange.location);
        }];
        
        // เปรียบเทียบ Unicode อย่างถูกต้อง
        // é สามารถแทนด้วย 2 วิธี: e + combining accent, หรือ precomposed é
        NSString *e1 = @"é";          // precomposed é
        NSString *e2 = @"é";         // e + combining accent
        NSLog(@"\ne1: %@, e2: %@", e1, e2);
        NSLog(@"e1 == e2 (isEqualToString): %@",
              [e1 isEqualToString:e2] ? @"YES" : @"NO");
        NSLog(@"e1.length: %lu, e2.length: %lu", e1.length, e2.length);
        
        // NSLocale
        NSLog(@"\n=== NSLocale ===");
        NSString *greeting = @"hello world";
        NSLog(@"Uppercase (Thai locale): %@",
              [greeting uppercaseStringWithLocale:[NSLocale localeWithLocaleIdentifier:@"th_TH"]]);
        NSLog(@"Uppercase (English locale): %@",
              [greeting uppercaseStringWithLocale:[NSLocale localeWithLocaleIdentifier:@"en_US"]]);
    }
    return 0;
}
```

---

## 8.9 Regular Expressions ด้วย NSRegularExpression

```objc
#import <Foundation/Foundation.h>

int main(int argc, const char * argv[]) {
    @autoreleasepool {
        
        NSLog(@"=== NSRegularExpression ===");
        
        NSError *error = nil;
        
        // Basic matching - ตรวจสอบ email format
        NSString *emailPattern = @"[A-Za-z0-9._%+-]+@[A-Za-z0-9.-]+\\.[A-Za-z]{2,}";
        NSRegularExpression *emailRegex = [NSRegularExpression
                                           regularExpressionWithPattern:emailPattern
                                           options:0
                                           error:&error];
        
        if (error) {
            NSLog(@"Error: %@", error.localizedDescription);
            return 1;
        }
        
        NSArray *testEmails = @[
            @"user@example.com",
            @"invalid-email",
            @"test.user+tag@domain.co.th",
            @"no@domain",
            @"valid123@test.org"
        ];
        
        for (NSString *email in testEmails) {
            NSUInteger matches = [emailRegex numberOfMatchesInString:email
                                                             options:0
                                                               range:NSMakeRange(0, email.length)];
            NSLog(@"%@: %@", email, matches > 0 ? @"Valid ✓" : @"Invalid ✗");
        }
        
        // การหาข้อมูลทั้งหมดที่ match
        NSLog(@"\n=== หาตัวเลขทั้งหมดใน text ===");
        NSString *text = @"I have 3 cats, 5 dogs, and 12 fish in my 2 ponds.";
        NSRegularExpression *numberRegex = [NSRegularExpression
                                            regularExpressionWithPattern:@"\\d+"
                                            options:0
                                            error:nil];
        
        NSArray *matches = [numberRegex matchesInString:text
                                                options:0
                                                  range:NSMakeRange(0, text.length)];
        
        NSLog(@"พบตัวเลข %lu ตัว:", matches.count);
        for (NSTextCheckingResult *match in matches) {
            NSString *found = [text substringWithRange:match.range];
            NSLog(@"  %@ ที่ตำแหน่ง %lu", found, match.range.location);
        }
        
        // Capture groups
        NSLog(@"\n=== Capture Groups ===");
        NSString *dateText = @"Today is 2024-01-15 and tomorrow is 2024-01-16";
        NSRegularExpression *dateRegex = [NSRegularExpression
                                          regularExpressionWithPattern:@"(\\d{4})-(\\d{2})-(\\d{2})"
                                          options:0
                                          error:nil];
        
        NSArray *dateMatches = [dateRegex matchesInString:dateText
                                                  options:0
                                                    range:NSMakeRange(0, dateText.length)];
        
        for (NSTextCheckingResult *match in dateMatches) {
            NSString *fullDate = [dateText substringWithRange:[match range]];
            NSString *year = [dateText substringWithRange:[match rangeAtIndex:1]];
            NSString *month = [dateText substringWithRange:[match rangeAtIndex:2]];
            NSString *day = [dateText substringWithRange:[match rangeAtIndex:3]];
            NSLog(@"วันที่: %@ (ปี: %@, เดือน: %@, วัน: %@)",
                  fullDate, year, month, day);
        }
        
        // การ replace ด้วย regex
        NSLog(@"\n=== Replace ด้วย Regex ===");
        NSString *phoneText = @"Call us at 081-234-5678 or 02-555-9999";
        NSRegularExpression *phoneRegex = [NSRegularExpression
                                           regularExpressionWithPattern:@"\\d{2,3}-\\d{3}-\\d{4}"
                                           options:0
                                           error:nil];
        
        NSString *masked = [phoneRegex stringByReplacingMatchesInString:phoneText
                                                                options:0
                                                                  range:NSMakeRange(0, phoneText.length)
                                                           withTemplate:@"XXX-XXX-XXXX"];
        NSLog(@"ปิด phone: %@", masked);
        
        // First match
        NSLog(@"\n=== First Match ===");
        NSRegularExpression *urlRegex = [NSRegularExpression
                                         regularExpressionWithPattern:@"https?://[\\w./-]+"
                                         options:0
                                         error:nil];
        
        NSString *webText = @"Visit https://www.example.com or http://test.org for more info";
        NSTextCheckingResult *firstMatch = [urlRegex firstMatchInString:webText
                                                                options:0
                                                                  range:NSMakeRange(0, webText.length)];
        
        if (firstMatch) {
            NSLog(@"URL แรก: %@", [webText substringWithRange:firstMatch.range]);
        }
        
        // Case insensitive
        NSLog(@"\n=== Case Insensitive ===");
        NSRegularExpression *ciRegex = [NSRegularExpression
                                        regularExpressionWithPattern:@"hello"
                                        options:NSRegularExpressionCaseInsensitive
                                        error:nil];
        
        NSString *ciText = @"Hello HELLO hello HeLLo";
        NSUInteger ciCount = [ciRegex numberOfMatchesInString:ciText
                                                      options:0
                                                        range:NSMakeRange(0, ciText.length)];
        NSLog(@"พบ 'hello' (case insensitive) %lu ครั้ง", ciCount);
    }
    return 0;
}
```

---

## 8.10 NSString vs C String

```objc
#import <Foundation/Foundation.h>
#include <string.h>

int main(int argc, const char * argv[]) {
    @autoreleasepool {
        
        NSLog(@"=== NSString vs C String ===\n");
        
        // C String (char array, null-terminated)
        char cStr[] = "Hello, World!";
        char *cPtr = "Hello, World!";
        
        // NSString
        NSString *nsStr = @"Hello, World!";
        
        // การแปลงระหว่างกัน
        // C to NSString
        NSString *fromC = [NSString stringWithUTF8String:cStr];
        NSString *fromC2 = [[NSString alloc] initWithBytes:cStr
                                                    length:strlen(cStr)
                                                  encoding:NSUTF8StringEncoding];
        
        // NSString to C
        const char *toC = [nsStr UTF8String];
        const char *toC2 = [nsStr cStringUsingEncoding:NSUTF8StringEncoding];
        
        NSLog(@"C string: %s", cStr);
        NSLog(@"NSString จาก C: %@", fromC);
        NSLog(@"C จาก NSString: %s", toC);
        
        // ข้อแตกต่าง
        NSLog(@"\n=== ข้อแตกต่าง ===");
        
        // 1. Encoding: C string ขึ้นกับ charset, NSString เป็น Unicode เสมอ
        NSString *thai = @"สวัสดี";
        const char *thaiC = [thai UTF8String];
        NSLog(@"Thai NSString: %@", thai);
        NSLog(@"Thai C string: %s", thaiC);
        NSLog(@"NSString length: %lu", thai.length);
        NSLog(@"C string length: %lu", strlen(thaiC));  // bytes ไม่ใช่ characters
        
        // 2. Immutability: NSString ไม่สามารถแก้ไขได้ตรงๆ
        // แต่ C string สามารถแก้ไขได้ (ถ้าไม่ใช่ const)
        cStr[0] = 'h';  // OK
        NSLog(@"\nC string หลังแก้: %s", cStr);
        // nsStr[0] = 'h';  // Error! NSString ไม่รองรับ subscript assignment
        
        // 3. Methods: NSString มี methods มากมาย, C string ใช้ <string.h>
        
        // C string operations
        char s1[50] = "Hello";
        char s2[] = " World";
        strcat(s1, s2);     // ต่อ string
        NSLog(@"\nC strcat: %s", s1);
        NSLog(@"C strlen: %lu", strlen(s1));
        
        int cmp = strcmp("ABC", "ABC");  // เปรียบเทียบ
        NSLog(@"C strcmp result: %d", cmp);
        
        char *found = strstr(s1, "World");  // ค้นหา
        if (found) {
            NSLog(@"C strstr: พบ 'World' ที่ offset %ld", found - s1);
        }
        
        // NSString equivalent
        NSString *ns1 = @"Hello World";
        NSLog(@"\nNSString length: %lu", ns1.length);
        NSLog(@"NSString contains: %@", [ns1 containsString:@"World"] ? @"YES" : @"NO");
        
        // char buffer - สำหรับงานที่ต้องการ performance สูง
        NSLog(@"\n=== Char Buffer ===");
        NSString *source = @"Performance critical string";
        NSMutableData *buffer = [NSMutableData dataWithLength:source.length + 1];
        char *buf = (char *)[buffer mutableBytes];
        [source getCString:buf maxLength:source.length + 1 encoding:NSUTF8StringEncoding];
        NSLog(@"Buffer: %s", buf);
    }
    return 0;
}
```

---

## 8.11 NSDateFormatter และ NSDate

```objc
#import <Foundation/Foundation.h>

int main(int argc, const char * argv[]) {
    @autoreleasepool {
        
        NSLog(@"=== NSDateFormatter ===");
        
        // สร้าง formatter
        NSDateFormatter *formatter = [[NSDateFormatter alloc] init];
        
        // รูปแบบมาตรฐาน
        formatter.dateStyle = NSDateFormatterLongStyle;
        formatter.timeStyle = NSDateFormatterMediumStyle;
        NSLog(@"วันที่ปัจจุบัน: %@", [formatter stringFromDate:[NSDate date]]);
        
        // Custom format
        formatter.dateFormat = @"dd/MM/yyyy HH:mm:ss";
        NSString *dateStr = [formatter stringFromDate:[NSDate date]];
        NSLog(@"Custom format: %@", dateStr);
        
        // Parse date from string
        formatter.dateFormat = @"yyyy-MM-dd";
        NSDate *date = [formatter dateFromString:@"2024-01-15"];
        formatter.dateFormat = @"dd MMMM yyyy";
        formatter.locale = [NSLocale localeWithLocaleIdentifier:@"th_TH"];
        NSLog(@"วันที่ (ไทย): %@", [formatter stringFromDate:date]);
        
        // ISO 8601
        NSISO8601DateFormatter *isoFormatter = [[NSISO8601DateFormatter alloc] init];
        NSString *isoStr = [isoFormatter stringFromDate:[NSDate date]];
        NSLog(@"\nISO 8601: %@", isoStr);
    }
    return 0;
}
```

---

## 8.12 String การทำงานร่วมกับไฟล์

```objc
#import <Foundation/Foundation.h>

int main(int argc, const char * argv[]) {
    @autoreleasepool {
        
        NSLog(@"=== String จากไฟล์ ===");
        
        // อ่านไฟล์ (ตัวอย่าง - ต้องมีไฟล์จริง)
        NSString *path = @"/tmp/test.txt";
        NSError *writeError = nil;
        
        // เขียนไฟล์
        NSString *content = @"Hello, World!\nบรรทัดที่ 2\nบรรทัดที่ 3";
        BOOL success = [content writeToFile:path
                                 atomically:YES
                                   encoding:NSUTF8StringEncoding
                                      error:&writeError];
        
        if (success) {
            NSLog(@"เขียนไฟล์สำเร็จ: %@", path);
            
            // อ่านไฟล์
            NSError *readError = nil;
            NSString *read = [NSString stringWithContentsOfFile:path
                                                       encoding:NSUTF8StringEncoding
                                                          error:&readError];
            if (read) {
                NSLog(@"เนื้อหาในไฟล์:\n%@", read);
                
                // แยกเป็น lines
                NSArray *lines = [read componentsSeparatedByString:@"\n"];
                NSLog(@"\nจำนวนบรรทัด: %lu", lines.count);
                for (NSUInteger i = 0; i < lines.count; i++) {
                    NSLog(@"บรรทัด %lu: %@", i + 1, lines[i]);
                }
            }
        }
        
        // Path manipulation
        NSLog(@"\n=== Path Manipulation ===");
        NSString *filePath = @"/Users/user/Documents/project/source.m";
        
        NSLog(@"lastPathComponent: %@", [filePath lastPathComponent]);
        NSLog(@"pathExtension: %@", [filePath pathExtension]);
        NSLog(@"stringByDeletingLastPathComponent: %@",
              [filePath stringByDeletingLastPathComponent]);
        NSLog(@"stringByDeletingPathExtension: %@",
              [filePath stringByDeletingPathExtension]);
        
        // สร้าง path
        NSString *dir = @"/Users/user/Documents";
        NSString *newPath = [dir stringByAppendingPathComponent:@"newfile.txt"];
        NSLog(@"\nAppend path: %@", newPath);
        
        NSString *withExt = [dir stringByAppendingPathExtension:@"backup"];
        NSLog(@"Append extension: %@", withExt);
    }
    return 0;
}
```

---

## 8.13 สรุปเนื้อหา

| Method | การใช้งาน |
|--------|-----------|
| `@"text"` | สร้าง NSString literal |
| `stringWithFormat:` | สร้างด้วย format |
| `length` | ความยาว string |
| `isEqualToString:` | เปรียบเทียบเนื้อหา |
| `compare:` | เปรียบเทียบ order |
| `containsString:` | ตรวจมี substring |
| `rangeOfString:` | หาตำแหน่ง substring |
| `substringFromIndex:` | ตัด substring จาก index |
| `substringWithRange:` | ตัด substring ตาม range |
| `uppercaseString` | แปลงเป็น uppercase |
| `lowercaseString` | แปลงเป็น lowercase |
| `stringByTrimmingCharactersInSet:` | ตัด whitespace |
| `stringByReplacingOccurrencesOfString:withString:` | แทน string |
| `componentsSeparatedByString:` | แยก string |
| `integerValue`, `doubleValue` | แปลงเป็นตัวเลข |
| `UTF8String` | แปลงเป็น C string |

---

## แบบฝึกหัด

### ข้อที่ 1: String Statistics
```objc
// รับ string ยาวๆ แล้วนับ:
// - จำนวน vowels (a,e,i,o,u)
// - จำนวน consonants
// - จำนวน spaces
// - จำนวน digits
// - จำนวน special characters
NSString *text = @"Hello World! I have 3 cats.";
```

### ข้อที่ 2: Palindrome Check
```objc
// ตรวจสอบว่า string เป็น palindrome หรือไม่
// (อ่านหน้าหรือหลังได้เหมือนกัน)
// เช่น "racecar", "level", "A man a plan a canal Panama"
BOOL isPalindrome(NSString *str);
```

### ข้อที่ 3: Word Count
```objc
// นับจำนวนคำใน text
// และแสดง word frequency (คำไหนพบบ่อยที่สุด)
NSString *paragraph = @"the cat sat on the mat the cat is fat";
// ผลลัพธ์: "the" = 3, "cat" = 2, ...
```

### ข้อที่ 4: String Reverse
```objc
// เขียนฟังก์ชัน reverse string
// ต้องรองรับ Unicode characters (emoji, Thai)
NSString *reverse(NSString *str);
// "Hello" → "olleH"
// "สวัสดี" → "ีดัสวส"
```

### ข้อที่ 5: Email Validator
```objc
// ตรวจสอบ email ว่า valid หรือไม่ ด้วย NSRegularExpression
// ต้องมีรูปแบบ xxx@xxx.xxx
BOOL isValidEmail(NSString *email);
```

### ข้อที่ 6: Template Engine
```objc
// สร้าง simple template engine
// แทน {{name}}, {{age}}, {{city}} ด้วยค่าจาก dictionary
NSString *template = @"สวัสดีคุณ {{name}} อายุ {{age}} ปี อาศัยอยู่ที่ {{city}}";
NSDictionary *data = @{@"name": @"สมชาย", @"age": @"25", @"city": @"กรุงเทพ"};
// ผลลัพธ์: "สวัสดีคุณ สมชาย อายุ 25 ปี อาศัยอยู่ที่ กรุงเทพ"
```

### ข้อที่ 7: Thai Number Format
```objc
// แปลงตัวเลขเป็นข้อความภาษาไทย
// เช่น 42 → "สี่สิบสอง"
// เช่น 1000 → "หนึ่งพัน"
NSString *thaiNumber(NSInteger number);
```

### ข้อที่ 8: String Compress
```objc
// Compress string โดยนับตัวอักษรซ้ำ
// "aabbbcccc" → "a2b3c4"
// "abc" → "abc" (ถ้าไม่ compress ได้ คืน original)
NSString *compress(NSString *str);
```

### ข้อที่ 9: CSV Parser
```objc
// Parse CSV string เป็น array ของ dictionary
NSString *csv = @"Name,Age,City\nSomchai,25,Bangkok\nManee,30,Chiang Mai";
// ผลลัพธ์: array ของ dict {Name, Age, City}
NSArray *parseCSV(NSString *csv);
```

### ข้อที่ 10: Phone Number Formatter
```objc
// จัดรูปแบบเบอร์โทรศัพท์
// "0812345678" → "081-234-5678"
// "0812345678" → "(081) 234-5678"
// "0812345678" → "+66 81 234 5678"
NSString *formatPhone(NSString *phone, NSString *format);
```

### ข้อที่ 11: String Tokenizer
```objc
// แบ่ง string เป็น tokens ด้วย delimiters หลายตัว
// "Hello,World;How are.you?" → ["Hello", "World", "How", "are", "you?"]
NSArray *tokenize(NSString *str, NSString *delimiters);
```

---

## เฉลยตัวอย่าง (ข้อที่ 2: Palindrome)

```objc
#import <Foundation/Foundation.h>

BOOL isPalindrome(NSString *str) {
    // ทำให้เป็น lowercase และลบ non-alphanumeric
    NSString *lower = [str lowercaseString];
    NSMutableString *cleaned = [NSMutableString string];
    
    NSCharacterSet *alphaNumeric = [NSCharacterSet alphanumericCharacterSet];
    for (NSUInteger i = 0; i < lower.length; i++) {
        unichar c = [lower characterAtIndex:i];
        if ([alphaNumeric characterIsMember:c]) {
            [cleaned appendFormat:@"%C", c];
        }
    }
    
    // ตรวจสอบ palindrome
    NSUInteger len = cleaned.length;
    for (NSUInteger i = 0; i < len / 2; i++) {
        if ([cleaned characterAtIndex:i] != [cleaned characterAtIndex:len - 1 - i]) {
            return NO;
        }
    }
    return YES;
}

int main(int argc, const char * argv[]) {
    @autoreleasepool {
        
        NSArray *tests = @[@"racecar", @"level", @"hello",
                           @"A man a plan a canal Panama", @"Was it a car or a cat I saw"];
        
        for (NSString *test in tests) {
            NSLog(@"'%@': %@", test, isPalindrome(test) ? @"Palindrome ✓" : @"Not Palindrome ✗");
        }
    }
    return 0;
}
```

---

*จบบทที่ 8: NSString Deep Dive* | ไปยัง [บทที่ 9: Memory Management →](part-09-memory-management.md)
