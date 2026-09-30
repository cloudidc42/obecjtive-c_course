# Part 29: NSDate และ NSCalendar ใน Objective-C

## บทนำ

NSDate เป็น class หลักสำหรับการจัดการวันที่และเวลาใน Objective-C / Foundation framework การทำงานกับวันที่และเวลาเป็นสิ่งที่โปรแกรมเมอร์ต้องพบบ่อยมาก ไม่ว่าจะเป็นการแสดงเวลาปัจจุบัน คำนวณอายุ นับถอยหลัง หรือจัดรูปแบบวันที่สำหรับแสดงผล

ใน part นี้เราจะเรียนรู้:
- การสร้าง NSDate
- การเปรียบเทียบวันที่
- การคำนวณวันที่
- NSCalendar และ NSDateComponents
- NSDateFormatter สำหรับ format และ parse วันที่
- NSTimeZone สำหรับจัดการ timezone
- Unix timestamp
- NSDateInterval

---

## 29.1 NSDate คืออะไร?

`NSDate` เป็น class ที่ represent จุดเวลาหนึ่งใน abstract time (โดยไม่อ้างอิง timezone ใด ๆ) ภายในเก็บค่าเป็นจำนวนวินาทีที่นับจาก **reference date** ซึ่งคือวันที่ 1 มกราคม ค.ศ. 2001 เวลา 00:00:00 UTC

```objc
#import <Foundation/Foundation.h>

int main(int argc, const char * argv[]) {
    @autoreleasepool {
        // สร้าง NSDate แทนเวลาปัจจุบัน
        NSDate *now = [NSDate date];
        NSLog(@"เวลาปัจจุบัน: %@", now);
        
        // ดูค่า timeIntervalSinceReferenceDate
        NSTimeInterval interval = [now timeIntervalSinceReferenceDate];
        NSLog(@"วินาทีนับจาก 1 ม.ค. 2001: %.2f", interval);
    }
    return 0;
}
```

**ผลลัพธ์ตัวอย่าง:**
```
เวลาปัจจุบัน: 2024-01-15 10:30:00 +0000
วินาทีนับจาก 1 ม.ค. 2001: 727000200.00
```

---

## 29.2 การสร้าง NSDate

### 29.2.1 สร้างด้วย class methods

```objc
#import <Foundation/Foundation.h>

int main(int argc, const char * argv[]) {
    @autoreleasepool {
        // วันที่ปัจจุบัน
        NSDate *now = [NSDate date];
        
        // วันที่ไกลในอดีต (distant past)
        NSDate *distantPast = [NSDate distantPast];
        
        // วันที่ไกลในอนาคต (distant future)
        NSDate *distantFuture = [NSDate distantFuture];
        
        NSLog(@"ปัจจุบัน:         %@", now);
        NSLog(@"อดีตไกล:         %@", distantPast);
        NSLog(@"อนาคตไกล:        %@", distantFuture);
        
        // สร้างจาก time interval นับจากวันนี้
        NSDate *tomorrow = [NSDate dateWithTimeIntervalSinceNow:86400]; // 24*60*60
        NSDate *yesterday = [NSDate dateWithTimeIntervalSinceNow:-86400];
        
        NSLog(@"พรุ่งนี้:         %@", tomorrow);
        NSLog(@"เมื่อวาน:        %@", yesterday);
    }
    return 0;
}
```

### 29.2.2 สร้างด้วย NSDateComponents + NSCalendar (แนะนำ)

วิธีนี้เป็นวิธีที่แนะนำสำหรับการสร้างวันที่ที่ระบุเจาะจง:

```objc
#import <Foundation/Foundation.h>

int main(int argc, const char * argv[]) {
    @autoreleasepool {
        NSCalendar *calendar = [NSCalendar currentCalendar];
        
        // สร้างวันที่ 25 ธันวาคม 2024
        NSDateComponents *components = [[NSDateComponents alloc] init];
        components.year  = 2024;
        components.month = 12;
        components.day   = 25;
        components.hour  = 0;
        components.minute = 0;
        components.second = 0;
        
        NSDate *christmas2024 = [calendar dateFromComponents:components];
        NSLog(@"คริสต์มาส 2024: %@", christmas2024);
        
        // สร้างวันที่ 1 มกราคม 2025 เวลาเที่ยงคืน
        components.year  = 2025;
        components.month = 1;
        components.day   = 1;
        NSDate *newYear2025 = [calendar dateFromComponents:components];
        NSLog(@"ปีใหม่ 2025:     %@", newYear2025);
    }
    return 0;
}
```

### 29.2.3 สร้างจาก timeIntervalSince1970 (Unix timestamp)

```objc
#import <Foundation/Foundation.h>

int main(int argc, const char * argv[]) {
    @autoreleasepool {
        // Unix timestamp = วินาทีนับจาก 1 ม.ค. 1970 00:00:00 UTC
        NSTimeInterval unixTimestamp = 1705312200; // ตัวอย่าง timestamp
        NSDate *dateFromUnix = [NSDate dateWithTimeIntervalSince1970:unixTimestamp];
        NSLog(@"วันที่จาก Unix timestamp: %@", dateFromUnix);
        
        // แปลงวันที่ปัจจุบันเป็น Unix timestamp
        NSDate *now = [NSDate date];
        NSTimeInterval nowAsUnix = [now timeIntervalSince1970];
        NSLog(@"Unix timestamp ปัจจุบัน: %.0f", nowAsUnix);
    }
    return 0;
}
```

---

## 29.3 การเปรียบเทียบวันที่

NSDate มี methods สำหรับเปรียบเทียบวันที่หลายแบบ:

```objc
#import <Foundation/Foundation.h>

int main(int argc, const char * argv[]) {
    @autoreleasepool {
        NSDate *now   = [NSDate date];
        NSDate *later = [NSDate dateWithTimeIntervalSinceNow:3600]; // 1 ชั่วโมงหลังจากนี้
        
        // เปรียบเทียบด้วย compare:
        NSComparisonResult result = [now compare:later];
        if (result == NSOrderedAscending) {
            NSLog(@"now อยู่ก่อน later (earlier)");
        } else if (result == NSOrderedDescending) {
            NSLog(@"now อยู่หลัง later (later)");
        } else {
            NSLog(@"now และ later เท่ากัน");
        }
        
        // isEqualToDate: สำหรับเปรียบเทียบว่าเท่ากันหรือไม่
        BOOL isEqual = [now isEqualToDate:later];
        NSLog(@"เท่ากัน? %@", isEqual ? @"ใช่" : @"ไม่ใช่");
        
        // earlierDate: และ laterDate:
        NSDate *earlier = [now earlierDate:later];
        NSDate *latest  = [now laterDate:later];
        NSLog(@"วันที่เร็วกว่า:  %@", earlier);
        NSLog(@"วันที่ช้ากว่า:   %@", latest);
        
        // เปรียบเทียบด้วย timeIntervalSinceDate:
        NSTimeInterval diff = [later timeIntervalSinceDate:now];
        NSLog(@"ต่างกัน %.0f วินาที (%.2f ชั่วโมง)", diff, diff / 3600.0);
    }
    return 0;
}
```

### 29.3.1 ฟังก์ชัน helper สำหรับเปรียบเทียบวันที่

```objc
#import <Foundation/Foundation.h>

// ตรวจสอบว่าวันที่ผ่านมาแล้วหรือยัง
BOOL isDateInPast(NSDate *date) {
    return [date compare:[NSDate date]] == NSOrderedAscending;
}

// ตรวจสอบว่าวันที่ยังไม่ถึงหรือเปล่า
BOOL isDateInFuture(NSDate *date) {
    return [date compare:[NSDate date]] == NSOrderedDescending;
}

// คืนค่าช่วงเวลาเป็นวินาที
NSTimeInterval secondsBetween(NSDate *start, NSDate *end) {
    return [end timeIntervalSinceDate:start];
}

int main(int argc, const char * argv[]) {
    @autoreleasepool {
        NSDate *past   = [NSDate dateWithTimeIntervalSinceNow:-10000];
        NSDate *future = [NSDate dateWithTimeIntervalSinceNow:10000];
        
        NSLog(@"past ผ่านมาแล้ว? %@",    isDateInPast(past)   ? @"ใช่" : @"ไม่");
        NSLog(@"future ยังไม่ถึง? %@",   isDateInFuture(future) ? @"ใช่" : @"ไม่");
        NSLog(@"ต่างกัน: %.0f วินาที",  secondsBetween(past, future));
    }
    return 0;
}
```

---

## 29.4 การคำนวณวันที่

### 29.4.1 dateByAddingTimeInterval:

วิธีง่ายที่สุดคือบวก/ลบวินาที:

```objc
#import <Foundation/Foundation.h>

int main(int argc, const char * argv[]) {
    @autoreleasepool {
        NSDate *now = [NSDate date];
        
        // บวก/ลบด้วย timeInterval (วินาที)
        NSTimeInterval oneDay    = 24 * 60 * 60;    // 86400 วินาที
        NSTimeInterval oneWeek   = 7  * oneDay;
        NSTimeInterval oneHour   = 3600;
        
        NSDate *tomorrow       = [now dateByAddingTimeInterval:oneDay];
        NSDate *nextWeek       = [now dateByAddingTimeInterval:oneWeek];
        NSDate *oneHourLater   = [now dateByAddingTimeInterval:oneHour];
        NSDate *oneHourEarlier = [now dateByAddingTimeInterval:-oneHour];
        
        NSLog(@"ตอนนี้:           %@", now);
        NSLog(@"พรุ่งนี้:         %@", tomorrow);
        NSLog(@"สัปดาห์หน้า:     %@", nextWeek);
        NSLog(@"1 ชั่วโมงหลังจากนี้: %@", oneHourLater);
        NSLog(@"1 ชั่วโมงก่อนหน้า:  %@", oneHourEarlier);
    }
    return 0;
}
```

### 29.4.2 NSDateComponents สำหรับการคำนวณที่ซับซ้อน

การบวก/ลบด้วย TimeInterval ไม่รองรับการ leap year หรือ daylight saving time ได้ดีนัก วิธีที่ถูกต้องกว่าคือใช้ NSCalendar:

```objc
#import <Foundation/Foundation.h>

int main(int argc, const char * argv[]) {
    @autoreleasepool {
        NSCalendar *calendar = [NSCalendar currentCalendar];
        NSDate *now = [NSDate date];
        
        // บวก 3 เดือน
        NSDateComponents *threeMonths = [[NSDateComponents alloc] init];
        threeMonths.month = 3;
        NSDate *in3Months = [calendar dateByAddingComponents:threeMonths
                                                       toDate:now
                                                      options:0];
        
        // บวก 1 ปี 2 เดือน 15 วัน
        NSDateComponents *offset = [[NSDateComponents alloc] init];
        offset.year  = 1;
        offset.month = 2;
        offset.day   = 15;
        NSDate *inTheFuture = [calendar dateByAddingComponents:offset
                                                         toDate:now
                                                        options:0];
        
        // ลบ 6 เดือน (ใส่ค่าติดลบ)
        NSDateComponents *minus6Months = [[NSDateComponents alloc] init];
        minus6Months.month = -6;
        NSDate *sixMonthsAgo = [calendar dateByAddingComponents:minus6Months
                                                          toDate:now
                                                         options:0];
        
        NSLog(@"ตอนนี้:              %@", now);
        NSLog(@"อีก 3 เดือน:        %@", in3Months);
        NSLog(@"อีก 1 ปี 2 เดือน 15 วัน: %@", inTheFuture);
        NSLog(@"6 เดือนที่แล้ว:      %@", sixMonthsAgo);
    }
    return 0;
}
```

---

## 29.5 NSCalendar

NSCalendar เป็น class ที่ใช้แปลง NSDate เป็น date components และในทางกลับกัน รวมถึงคำนวณวันที่ในรูปแบบต่าง ๆ

### 29.5.1 ชนิดของ Calendar

```objc
#import <Foundation/Foundation.h>

int main(int argc, const char * argv[]) {
    @autoreleasepool {
        // Calendar ปัจจุบันของ device
        NSCalendar *currentCal = [NSCalendar currentCalendar];
        
        // Gregorian calendar (ปฏิทินสากล)
        NSCalendar *gregorian = [NSCalendar calendarWithIdentifier:NSCalendarIdentifierGregorian];
        
        // Buddhist calendar (ปฏิทินพุทธศักราช)
        NSCalendar *buddhist = [NSCalendar calendarWithIdentifier:NSCalendarIdentifierBuddhist];
        
        // ISO 8601 calendar
        NSCalendar *iso = [NSCalendar calendarWithIdentifier:NSCalendarIdentifierISO8601];
        
        NSDate *now = [NSDate date];
        
        // ดึง year จาก calendar ต่างกัน
        NSInteger gregorianYear = [gregorian component:NSCalendarUnitYear fromDate:now];
        NSInteger buddhistYear  = [buddhist  component:NSCalendarUnitYear fromDate:now];
        
        NSLog(@"ปีคริสต์ศักราช:   %ld", (long)gregorianYear);
        NSLog(@"ปีพุทธศักราช:     %ld", (long)buddhistYear); // gregorian + 543
    }
    return 0;
}
```

### 29.5.2 ดึง Date Components

```objc
#import <Foundation/Foundation.h>

int main(int argc, const char * argv[]) {
    @autoreleasepool {
        NSCalendar *calendar = [NSCalendar calendarWithIdentifier:NSCalendarIdentifierGregorian];
        NSDate *now = [NSDate date];
        
        // ดึงแต่ละ component แยกกัน
        NSInteger year   = [calendar component:NSCalendarUnitYear   fromDate:now];
        NSInteger month  = [calendar component:NSCalendarUnitMonth  fromDate:now];
        NSInteger day    = [calendar component:NSCalendarUnitDay    fromDate:now];
        NSInteger hour   = [calendar component:NSCalendarUnitHour   fromDate:now];
        NSInteger minute = [calendar component:NSCalendarUnitMinute fromDate:now];
        NSInteger second = [calendar component:NSCalendarUnitSecond fromDate:now];
        NSInteger weekday = [calendar component:NSCalendarUnitWeekday fromDate:now];
        
        NSLog(@"ปี: %ld", (long)year);
        NSLog(@"เดือน: %ld", (long)month);
        NSLog(@"วัน: %ld", (long)day);
        NSLog(@"ชั่วโมง: %ld", (long)hour);
        NSLog(@"นาที: %ld", (long)minute);
        NSLog(@"วินาที: %ld", (long)second);
        NSLog(@"วันในสัปดาห์: %ld (1=อาทิตย์, 7=เสาร์)", (long)weekday);
        
        // ดึงหลาย component พร้อมกัน
        NSDateComponents *comps = [calendar components:(NSCalendarUnitYear |
                                                         NSCalendarUnitMonth |
                                                         NSCalendarUnitDay)
                                              fromDate:now];
        NSLog(@"วันที่: %ld/%ld/%ld", (long)comps.day, (long)comps.month, (long)comps.year);
    }
    return 0;
}
```

### 29.5.3 การเปรียบเทียบวันที่ด้วย Calendar

```objc
#import <Foundation/Foundation.h>

int main(int argc, const char * argv[]) {
    @autoreleasepool {
        NSCalendar *calendar = [NSCalendar currentCalendar];
        NSDate *now = [NSDate date];
        
        // ตรวจสอบว่าเป็นวันเดียวกันหรือไม่
        NSDate *todayLater = [NSDate dateWithTimeIntervalSinceNow:3600]; // 1 ชั่วโมงข้างหน้า
        NSDate *tomorrow   = [NSDate dateWithTimeIntervalSinceNow:86400];
        
        BOOL sameDay1 = [calendar isDate:now inSameDayAsDate:todayLater];
        BOOL sameDay2 = [calendar isDate:now inSameDayAsDate:tomorrow];
        
        NSLog(@"now และ todayLater เป็นวันเดียวกัน? %@", sameDay1 ? @"ใช่" : @"ไม่");
        NSLog(@"now และ tomorrow เป็นวันเดียวกัน? %@", sameDay2 ? @"ใช่" : @"ไม่");
        
        // ตรวจสอบว่าเป็นวันนี้หรือไม่
        BOOL isToday = [calendar isDateInToday:now];
        BOOL isTomorrow = [calendar isDateInTomorrow:tomorrow];
        BOOL isYesterday = [calendar isDateInYesterday:[NSDate dateWithTimeIntervalSinceNow:-86400]];
        
        NSLog(@"now เป็นวันนี้? %@",     isToday     ? @"ใช่" : @"ไม่");
        NSLog(@"tomorrow เป็นพรุ่งนี้? %@", isTomorrow  ? @"ใช่" : @"ไม่");
        NSLog(@"yesterday เป็นเมื่อวาน? %@", isYesterday ? @"ใช่" : @"ไม่");
        
        // ตรวจสอบวันสุดสัปดาห์
        BOOL isWeekend = [calendar isDateInWeekend:now];
        NSLog(@"วันนี้เป็นวันหยุด? %@", isWeekend ? @"ใช่" : @"ไม่");
    }
    return 0;
}
```

### 29.5.4 คำนวณความต่างระหว่างวันที่

```objc
#import <Foundation/Foundation.h>

int main(int argc, const char * argv[]) {
    @autoreleasepool {
        NSCalendar *calendar = [NSCalendar calendarWithIdentifier:NSCalendarIdentifierGregorian];
        
        // สร้างวันที่เกิด
        NSDateComponents *birthComps = [[NSDateComponents alloc] init];
        birthComps.year  = 1990;
        birthComps.month = 5;
        birthComps.day   = 15;
        NSDate *birthDate = [calendar dateFromComponents:birthComps];
        
        NSDate *now = [NSDate date];
        
        // คำนวณความต่างเป็นปี เดือน วัน
        NSDateComponents *diff = [calendar components:(NSCalendarUnitYear |
                                                        NSCalendarUnitMonth |
                                                        NSCalendarUnitDay)
                                             fromDate:birthDate
                                               toDate:now
                                              options:0];
        
        NSLog(@"อายุ: %ld ปี %ld เดือน %ld วัน",
              (long)diff.year, (long)diff.month, (long)diff.day);
        
        // ความต่างเป็นวันเท่านั้น
        NSDateComponents *daysOnly = [calendar components:NSCalendarUnitDay
                                                 fromDate:birthDate
                                                   toDate:now
                                                  options:0];
        NSLog(@"หรือ %ld วัน ที่มีชีวิตอยู่", (long)daysOnly.day);
    }
    return 0;
}
```

---

## 29.6 NSDateFormatter

NSDateFormatter ใช้สำหรับแปลง NSDate เป็น String และ String เป็น NSDate

### 29.6.1 Format พื้นฐาน

```objc
#import <Foundation/Foundation.h>

int main(int argc, const char * argv[]) {
    @autoreleasepool {
        NSDate *now = [NSDate date];
        NSDateFormatter *formatter = [[NSDateFormatter alloc] init];
        
        // รูปแบบ วัน/เดือน/ปี
        formatter.dateFormat = @"dd/MM/yyyy";
        NSLog(@"รูปแบบ 1: %@", [formatter stringFromDate:now]);
        
        // รูปแบบพร้อมเวลา
        formatter.dateFormat = @"dd/MM/yyyy HH:mm:ss";
        NSLog(@"รูปแบบ 2: %@", [formatter stringFromDate:now]);
        
        // รูปแบบ ISO 8601
        formatter.dateFormat = @"yyyy-MM-dd'T'HH:mm:ssZ";
        NSLog(@"รูปแบบ ISO: %@", [formatter stringFromDate:now]);
        
        // รูปแบบภาษาไทย
        formatter.dateFormat = @"d MMMM yyyy";
        formatter.locale = [NSLocale localeWithLocaleIdentifier:@"th_TH"];
        NSLog(@"รูปแบบไทย: %@", [formatter stringFromDate:now]);
        
        // รูปแบบภาษาอังกฤษ
        formatter.locale = [NSLocale localeWithLocaleIdentifier:@"en_US"];
        formatter.dateFormat = @"EEEE, MMMM d, yyyy";
        NSLog(@"รูปแบบอังกฤษ: %@", [formatter stringFromDate:now]);
    }
    return 0;
}
```

### 29.6.2 Date Style และ Time Style

```objc
#import <Foundation/Foundation.h>

int main(int argc, const char * argv[]) {
    @autoreleasepool {
        NSDate *now = [NSDate date];
        NSDateFormatter *formatter = [[NSDateFormatter alloc] init];
        
        // ใช้ dateStyle และ timeStyle แทน format string
        // NSDateFormatterNoStyle, NSDateFormatterShortStyle,
        // NSDateFormatterMediumStyle, NSDateFormatterLongStyle, NSDateFormatterFullStyle
        
        formatter.dateStyle = NSDateFormatterShortStyle;
        formatter.timeStyle = NSDateFormatterNoStyle;
        NSLog(@"Short date:  %@", [formatter stringFromDate:now]);
        
        formatter.dateStyle = NSDateFormatterMediumStyle;
        formatter.timeStyle = NSDateFormatterShortStyle;
        NSLog(@"Medium date + Short time: %@", [formatter stringFromDate:now]);
        
        formatter.dateStyle = NSDateFormatterLongStyle;
        formatter.timeStyle = NSDateFormatterMediumStyle;
        NSLog(@"Long date + Medium time: %@", [formatter stringFromDate:now]);
        
        formatter.dateStyle = NSDateFormatterFullStyle;
        formatter.timeStyle = NSDateFormatterFullStyle;
        NSLog(@"Full date + Full time: %@", [formatter stringFromDate:now]);
        
        // เปลี่ยน locale เป็น Thai
        formatter.locale = [NSLocale localeWithLocaleIdentifier:@"th_TH"];
        formatter.dateStyle = NSDateFormatterFullStyle;
        formatter.timeStyle = NSDateFormatterNoStyle;
        NSLog(@"Thai full date: %@", [formatter stringFromDate:now]);
    }
    return 0;
}
```

### 29.6.3 การ Parse String เป็น NSDate

```objc
#import <Foundation/Foundation.h>

int main(int argc, const char * argv[]) {
    @autoreleasepool {
        NSDateFormatter *formatter = [[NSDateFormatter alloc] init];
        
        // parse วันที่จาก string รูปแบบต่าง ๆ
        formatter.dateFormat = @"dd/MM/yyyy";
        NSDate *date1 = [formatter dateFromString:@"25/12/2024"];
        NSLog(@"parse 1: %@", date1);
        
        formatter.dateFormat = @"yyyy-MM-dd";
        NSDate *date2 = [formatter dateFromString:@"2024-12-25"];
        NSLog(@"parse 2: %@", date2);
        
        formatter.dateFormat = @"MM/dd/yyyy HH:mm";
        NSDate *date3 = [formatter dateFromString:@"12/25/2024 10:30"];
        NSLog(@"parse 3: %@", date3);
        
        // กรณี parse ไม่สำเร็จ จะได้ nil
        formatter.dateFormat = @"dd/MM/yyyy";
        NSDate *invalid = [formatter dateFromString:@"not-a-date"];
        if (invalid == nil) {
            NSLog(@"parse ไม่สำเร็จ ได้ nil");
        }
        
        // ISO 8601 parsing
        NSISO8601DateFormatter *isoFormatter = [[NSISO8601DateFormatter alloc] init];
        NSDate *isoDate = [isoFormatter dateFromString:@"2024-12-25T10:30:00Z"];
        NSLog(@"ISO parse: %@", isoDate);
    }
    return 0;
}
```

### 29.6.4 Format String Symbols

| Symbol | ความหมาย              | ตัวอย่าง         |
|--------|----------------------|----------------|
| `yyyy` | ปี 4 หลัก             | 2024           |
| `yy`   | ปี 2 หลัก             | 24             |
| `MM`   | เดือน 2 หลัก          | 01, 12         |
| `M`    | เดือน (ไม่นำหน้า 0)   | 1, 12          |
| `MMMM` | ชื่อเดือนเต็ม           | January        |
| `MMM`  | ชื่อเดือนย่อ            | Jan            |
| `dd`   | วัน 2 หลัก             | 01, 31         |
| `d`    | วัน (ไม่นำหน้า 0)     | 1, 31          |
| `EEEE` | ชื่อวันเต็ม             | Monday         |
| `EEE`  | ชื่อวันย่อ              | Mon            |
| `HH`   | ชั่วโมง 24h 2 หลัก     | 00, 23         |
| `hh`   | ชั่วโมง 12h 2 หลัก     | 01, 12         |
| `mm`   | นาที 2 หลัก            | 00, 59         |
| `ss`   | วินาที 2 หลัก          | 00, 59         |
| `a`    | AM/PM                | AM, PM         |
| `z`    | Timezone abbreviation | GMT+7          |
| `Z`    | RFC 822 timezone     | +0700          |
| `ZZZZZ`| ISO 8601 timezone   | +07:00         |

---

## 29.7 NSTimeZone

```objc
#import <Foundation/Foundation.h>

int main(int argc, const char * argv[]) {
    @autoreleasepool {
        // Timezone ปัจจุบันของ device
        NSTimeZone *localTZ = [NSTimeZone localTimeZone];
        NSLog(@"Timezone ปัจจุบัน: %@", localTZ.name);
        NSLog(@"Timezone offset: %ld วินาที", (long)localTZ.secondsFromGMT);
        
        // Timezone ของกรุงเทพ
        NSTimeZone *bangkokTZ = [NSTimeZone timeZoneWithName:@"Asia/Bangkok"];
        NSLog(@"Bangkok timezone: %@", bangkokTZ.name);
        NSLog(@"Bangkok offset: %ld วินาที (= %ld ชั่วโมง)",
              (long)bangkokTZ.secondsFromGMT,
              (long)(bangkokTZ.secondsFromGMT / 3600));
        
        // ดู timezone ทั้งหมดที่รองรับ
        NSArray *allTZNames = [NSTimeZone knownTimeZoneNames];
        NSLog(@"Timezone ทั้งหมด: %lu รายการ", (unsigned long)allTZNames.count);
        
        // timezone บางส่วนที่น่าสนใจ
        NSArray *interesting = @[@"Asia/Bangkok",
                                  @"Asia/Tokyo",
                                  @"America/New_York",
                                  @"Europe/London",
                                  @"UTC"];
        for (NSString *tzName in interesting) {
            NSTimeZone *tz = [NSTimeZone timeZoneWithName:tzName];
            NSLog(@"%-30s offset: %+.0f ชั่วโมง",
                  tzName.UTF8String,
                  tz.secondsFromGMT / 3600.0);
        }
    }
    return 0;
}
```

### 29.7.1 แสดงเวลาเดียวกันใน Timezone ต่าง ๆ

```objc
#import <Foundation/Foundation.h>

int main(int argc, const char * argv[]) {
    @autoreleasepool {
        NSDate *now = [NSDate date];
        NSDateFormatter *formatter = [[NSDateFormatter alloc] init];
        formatter.dateFormat = @"yyyy-MM-dd HH:mm:ss";
        
        NSArray *cities = @[
            @{@"name": @"กรุงเทพ",      @"tz": @"Asia/Bangkok"},
            @{@"name": @"โตเกียว",      @"tz": @"Asia/Tokyo"},
            @{@"name": @"ลอนดอน",       @"tz": @"Europe/London"},
            @{@"name": @"นิวยอร์ค",     @"tz": @"America/New_York"},
            @{@"name": @"ลอสแองเจลีส", @"tz": @"America/Los_Angeles"},
        ];
        
        NSLog(@"เวลาปัจจุบันใน timezone ต่าง ๆ:");
        for (NSDictionary *city in cities) {
            formatter.timeZone = [NSTimeZone timeZoneWithName:city[@"tz"]];
            NSLog(@"  %-18s: %@", [city[@"name"] UTF8String],
                  [formatter stringFromDate:now]);
        }
    }
    return 0;
}
```

---

## 29.8 Unix Timestamps

Unix timestamp (หรือ POSIX time) คือจำนวนวินาทีนับจาก 1 มกราคม ค.ศ. 1970 00:00:00 UTC

```objc
#import <Foundation/Foundation.h>

int main(int argc, const char * argv[]) {
    @autoreleasepool {
        // ดึง Unix timestamp ปัจจุบัน
        NSDate *now = [NSDate date];
        NSTimeInterval unixNow = [now timeIntervalSince1970];
        NSLog(@"Unix timestamp ปัจจุบัน: %.0f", unixNow);
        
        // แปลงจาก Unix timestamp กลับเป็น NSDate
        NSTimeInterval specificTimestamp = 1735084800; // 25 Dec 2024 00:00:00 UTC
        NSDate *christmas = [NSDate dateWithTimeIntervalSince1970:specificTimestamp];
        
        NSDateFormatter *formatter = [[NSDateFormatter alloc] init];
        formatter.dateFormat = @"dd MMMM yyyy HH:mm:ss zzz";
        formatter.locale = [NSLocale localeWithLocaleIdentifier:@"th_TH"];
        NSLog(@"จาก timestamp %ld: %@",
              (long)specificTimestamp,
              [formatter stringFromDate:christmas]);
        
        // เปรียบเทียบ timestamp สองค่า
        NSTimeInterval ts1 = 1735084800;
        NSTimeInterval ts2 = 1735171200; // 86400 วินาทีถัดมา
        NSDate *date1 = [NSDate dateWithTimeIntervalSince1970:ts1];
        NSDate *date2 = [NSDate dateWithTimeIntervalSince1970:ts2];
        
        NSTimeInterval diffSeconds = [date2 timeIntervalSinceDate:date1];
        NSLog(@"ต่างกัน: %.0f วินาที = %.0f วัน", diffSeconds, diffSeconds / 86400);
        
        // ใช้ time() function จาก C standard library
        #include <time.h>
        time_t cTimestamp = time(NULL);
        NSLog(@"C time() timestamp: %ld", (long)cTimestamp);
        NSDate *dateFromCTime = [NSDate dateWithTimeIntervalSince1970:(NSTimeInterval)cTimestamp];
        NSLog(@"จาก C time: %@", [formatter stringFromDate:dateFromCTime]);
    }
    return 0;
}
```

---

## 29.9 NSDateInterval

`NSDateInterval` (iOS 10+ / macOS 10.12+) ใช้แทนช่วงเวลาระหว่างวันที่สองวัน

```objc
#import <Foundation/Foundation.h>

int main(int argc, const char * argv[]) {
    @autoreleasepool {
        NSDate *start = [NSDate date];
        NSDate *end   = [NSDate dateWithTimeIntervalSinceNow:7 * 86400]; // 7 วัน
        
        // สร้าง NSDateInterval
        NSDateInterval *interval = [[NSDateInterval alloc] initWithStartDate:start
                                                                      endDate:end];
        
        NSLog(@"เริ่ม:    %@", interval.startDate);
        NSLog(@"สิ้นสุด:  %@", interval.endDate);
        NSLog(@"ระยะเวลา: %.0f วินาที (%.1f วัน)",
              interval.duration,
              interval.duration / 86400.0);
        
        // ตรวจสอบว่าวันที่หนึ่ง ๆ อยู่ในช่วงนี้หรือไม่
        NSDate *middle = [NSDate dateWithTimeIntervalSinceNow:3 * 86400];
        NSDate *outside = [NSDate dateWithTimeIntervalSinceNow:10 * 86400];
        
        BOOL middleIn  = [interval containsDate:middle];
        BOOL outsideIn = [interval containsDate:outside];
        NSLog(@"middle อยู่ในช่วง? %@",  middleIn  ? @"ใช่" : @"ไม่");
        NSLog(@"outside อยู่ในช่วง? %@", outsideIn ? @"ใช่" : @"ไม่");
        
        // ตรวจสอบการ intersect ระหว่างสอง interval
        NSDate *start2 = [NSDate dateWithTimeIntervalSinceNow:3 * 86400];
        NSDate *end2   = [NSDate dateWithTimeIntervalSinceNow:14 * 86400];
        NSDateInterval *interval2 = [[NSDateInterval alloc] initWithStartDate:start2 endDate:end2];
        
        BOOL intersects = [interval intersectsDateInterval:interval2];
        NSLog(@"สอง interval ซ้อนกัน? %@", intersects ? @"ใช่" : @"ไม่");
        
        if (intersects) {
            NSDateInterval *overlap = [interval intersectionWithDateInterval:interval2];
            NSLog(@"ส่วนที่ซ้อนกัน: %.0f วินาที (%.1f วัน)",
                  overlap.duration, overlap.duration / 86400.0);
        }
    }
    return 0;
}
```

---

## 29.10 ตัวอย่างจริง: เครื่องคิดอายุ (Age Calculator)

```objc
#import <Foundation/Foundation.h>

// ฟังก์ชันสำหรับสร้างวันที่เกิด
NSDate* createBirthdate(NSInteger year, NSInteger month, NSInteger day) {
    NSCalendar *cal = [NSCalendar calendarWithIdentifier:NSCalendarIdentifierGregorian];
    NSDateComponents *comps = [[NSDateComponents alloc] init];
    comps.year  = year;
    comps.month = month;
    comps.day   = day;
    return [cal dateFromComponents:comps];
}

// โครงสร้างเก็บข้อมูลอายุ
typedef struct {
    NSInteger years;
    NSInteger months;
    NSInteger days;
    NSInteger totalDays;
} AgeInfo;

// ฟังก์ชันคำนวณอายุ
AgeInfo calculateAge(NSDate *birthdate) {
    NSCalendar *cal = [NSCalendar calendarWithIdentifier:NSCalendarIdentifierGregorian];
    NSDate *today = [NSDate date];
    
    NSDateComponents *comps = [cal components:(NSCalendarUnitYear |
                                                NSCalendarUnitMonth |
                                                NSCalendarUnitDay)
                                     fromDate:birthdate
                                       toDate:today
                                      options:0];
    
    NSDateComponents *daysComps = [cal components:NSCalendarUnitDay
                                         fromDate:birthdate
                                           toDate:today
                                          options:0];
    
    AgeInfo info;
    info.years     = comps.year;
    info.months    = comps.month;
    info.days      = comps.day;
    info.totalDays = daysComps.day;
    return info;
}

// ตรวจสอบว่าวันนี้เป็นวันเกิดหรือไม่
BOOL isBirthdayToday(NSDate *birthdate) {
    NSCalendar *cal = [NSCalendar calendarWithIdentifier:NSCalendarIdentifierGregorian];
    NSDate *today = [NSDate date];
    
    NSInteger birthMonth = [cal component:NSCalendarUnitMonth fromDate:birthdate];
    NSInteger birthDay   = [cal component:NSCalendarUnitDay   fromDate:birthdate];
    NSInteger todayMonth = [cal component:NSCalendarUnitMonth fromDate:today];
    NSInteger todayDay   = [cal component:NSCalendarUnitDay   fromDate:today];
    
    return birthMonth == todayMonth && birthDay == todayDay;
}

// หาวันเกิดครั้งถัดไป
NSDate* nextBirthday(NSDate *birthdate) {
    NSCalendar *cal = [NSCalendar calendarWithIdentifier:NSCalendarIdentifierGregorian];
    NSDate *today = [NSDate date];
    
    NSInteger birthMonth = [cal component:NSCalendarUnitMonth fromDate:birthdate];
    NSInteger birthDay   = [cal component:NSCalendarUnitDay   fromDate:birthdate];
    NSInteger currentYear = [cal component:NSCalendarUnitYear fromDate:today];
    
    NSDateComponents *nextBdayComps = [[NSDateComponents alloc] init];
    nextBdayComps.year  = currentYear;
    nextBdayComps.month = birthMonth;
    nextBdayComps.day   = birthDay;
    
    NSDate *thisBirthday = [cal dateFromComponents:nextBdayComps];
    
    // ถ้าผ่านไปแล้ว ใช้ปีหน้า
    if ([thisBirthday compare:today] == NSOrderedAscending) {
        nextBdayComps.year = currentYear + 1;
        thisBirthday = [cal dateFromComponents:nextBdayComps];
    }
    
    return thisBirthday;
}

int main(int argc, const char * argv[]) {
    @autoreleasepool {
        // ทดสอบกับวันที่เกิดต่าง ๆ
        NSDate *birthdate1 = createBirthdate(1990, 5, 15);
        NSDate *birthdate2 = createBirthdate(2000, 1, 1);
        
        NSDateFormatter *df = [[NSDateFormatter alloc] init];
        df.dateFormat = @"dd/MM/yyyy";
        df.locale = [NSLocale localeWithLocaleIdentifier:@"th_TH"];
        
        NSArray *people = @[
            @{@"name": @"คนที่ 1", @"bday": birthdate1},
            @{@"name": @"คนที่ 2", @"bday": birthdate2},
        ];
        
        for (NSDictionary *person in people) {
            NSDate *bday = person[@"bday"];
            AgeInfo age = calculateAge(bday);
            
            NSLog(@"\n=== %@ ===", person[@"name"]);
            NSLog(@"วันเกิด: %@", [df stringFromDate:bday]);
            NSLog(@"อายุ: %ld ปี %ld เดือน %ld วัน",
                  (long)age.years, (long)age.months, (long)age.days);
            NSLog(@"รวมทั้งหมด: %ld วัน", (long)age.totalDays);
            
            if (isBirthdayToday(bday)) {
                NSLog(@"สุขสันต์วันเกิด! 🎂");
            } else {
                NSDate *nextBday = nextBirthday(bday);
                NSCalendar *cal = [NSCalendar currentCalendar];
                NSDateComponents *daysUntil = [cal components:NSCalendarUnitDay
                                                     fromDate:[NSDate date]
                                                       toDate:nextBday
                                                      options:0];
                NSLog(@"วันเกิดครั้งถัดไป: %@ (อีก %ld วัน)",
                      [df stringFromDate:nextBday],
                      (long)daysUntil.day);
            }
        }
    }
    return 0;
}
```

---

## 29.11 ตัวอย่างจริง: Countdown Timer

```objc
#import <Foundation/Foundation.h>

// โครงสร้างเก็บข้อมูล countdown
typedef struct {
    NSInteger days;
    NSInteger hours;
    NSInteger minutes;
    NSInteger seconds;
    BOOL isExpired;
} CountdownInfo;

CountdownInfo countdownTo(NSDate *targetDate) {
    NSDate *now = [NSDate date];
    NSTimeInterval diff = [targetDate timeIntervalSinceDate:now];
    
    CountdownInfo info;
    
    if (diff <= 0) {
        info.isExpired = YES;
        info.days = info.hours = info.minutes = info.seconds = 0;
        return info;
    }
    
    info.isExpired = NO;
    
    NSInteger totalSeconds = (NSInteger)diff;
    info.seconds = totalSeconds % 60;
    info.minutes = (totalSeconds / 60) % 60;
    info.hours   = (totalSeconds / 3600) % 24;
    info.days    = totalSeconds / 86400;
    
    return info;
}

NSDate* createEventDate(NSInteger year, NSInteger month, NSInteger day,
                         NSInteger hour, NSInteger minute) {
    NSCalendar *cal = [NSCalendar calendarWithIdentifier:NSCalendarIdentifierGregorian];
    NSDateComponents *comps = [[NSDateComponents alloc] init];
    comps.year   = year;
    comps.month  = month;
    comps.day    = day;
    comps.hour   = hour;
    comps.minute = minute;
    comps.second = 0;
    return [cal dateFromComponents:comps];
}

int main(int argc, const char * argv[]) {
    @autoreleasepool {
        NSDateFormatter *df = [[NSDateFormatter alloc] init];
        df.dateFormat = @"dd/MM/yyyy HH:mm";
        
        // กำหนด events
        NSArray *events = @[
            @{@"name": @"ปีใหม่ 2026",
              @"date": createEventDate(2026, 1, 1, 0, 0)},
            @{@"name": @"วันวาเลนไทน์ 2026",
              @"date": createEventDate(2026, 2, 14, 0, 0)},
            @{@"name": @"สงกรานต์ 2026",
              @"date": createEventDate(2026, 4, 13, 0, 0)},
        ];
        
        NSLog(@"=== Countdown Timer ===");
        NSLog(@"เวลาปัจจุบัน: %@\n", [df stringFromDate:[NSDate date]]);
        
        for (NSDictionary *event in events) {
            NSDate *eventDate = event[@"date"];
            CountdownInfo cd = countdownTo(eventDate);
            
            NSLog(@"Event: %@", event[@"name"]);
            NSLog(@"วันที่:  %@", [df stringFromDate:eventDate]);
            
            if (cd.isExpired) {
                NSLog(@"สถานะ: ผ่านไปแล้ว");
            } else {
                NSLog(@"เหลืออีก: %ld วัน %ld ชั่วโมง %ld นาที %ld วินาที",
                      (long)cd.days, (long)cd.hours,
                      (long)cd.minutes, (long)cd.seconds);
            }
            NSLog(@"---");
        }
    }
    return 0;
}
```

---

## 29.12 ตัวอย่างจริง: ตารางเวลา (Schedule Checker)

```objc
#import <Foundation/Foundation.h>

typedef NS_ENUM(NSInteger, DayOfWeek) {
    Sunday    = 1,
    Monday    = 2,
    Tuesday   = 3,
    Wednesday = 4,
    Thursday  = 5,
    Friday    = 6,
    Saturday  = 7
};

NSString* dayName(DayOfWeek day) {
    switch (day) {
        case Sunday:    return @"อาทิตย์";
        case Monday:    return @"จันทร์";
        case Tuesday:   return @"อังคาร";
        case Wednesday: return @"พุธ";
        case Thursday:  return @"พฤหัสบดี";
        case Friday:    return @"ศุกร์";
        case Saturday:  return @"เสาร์";
    }
    return @"ไม่รู้จัก";
}

// ตรวจสอบว่าตอนนี้อยู่ในช่วงเวลาทำการหรือไม่ (จ-ศ 9:00-17:00)
BOOL isBusinessHours(void) {
    NSCalendar *cal = [NSCalendar calendarWithIdentifier:NSCalendarIdentifierGregorian];
    NSDate *now = [NSDate date];
    
    NSInteger weekday = [cal component:NSCalendarUnitWeekday fromDate:now];
    NSInteger hour    = [cal component:NSCalendarUnitHour    fromDate:now];
    NSInteger minute  = [cal component:NSCalendarUnitMinute  fromDate:now];
    
    // จันทร์(2) ถึง ศุกร์(6)
    if (weekday < 2 || weekday > 6) return NO;
    
    // 9:00 ถึง 17:00
    NSInteger timeInMinutes = hour * 60 + minute;
    return timeInMinutes >= 9*60 && timeInMinutes < 17*60;
}

// หาวันทำงานถัดไป
NSDate* nextWorkday(void) {
    NSCalendar *cal = [NSCalendar calendarWithIdentifier:NSCalendarIdentifierGregorian];
    NSDate *candidate = [NSDate date];
    
    NSDateComponents *oneDayOffset = [[NSDateComponents alloc] init];
    oneDayOffset.day = 1;
    
    // เริ่มจากพรุ่งนี้
    candidate = [cal dateByAddingComponents:oneDayOffset toDate:candidate options:0];
    
    // วนหา weekday ที่เป็น จ-ศ
    for (int i = 0; i < 7; i++) {
        NSInteger weekday = [cal component:NSCalendarUnitWeekday fromDate:candidate];
        if (weekday >= 2 && weekday <= 6) {
            break;
        }
        candidate = [cal dateByAddingComponents:oneDayOffset toDate:candidate options:0];
    }
    
    return candidate;
}

int main(int argc, const char * argv[]) {
    @autoreleasepool {
        NSCalendar *cal = [NSCalendar calendarWithIdentifier:NSCalendarIdentifierGregorian];
        NSDate *now = [NSDate date];
        
        NSInteger weekday = [cal component:NSCalendarUnitWeekday fromDate:now];
        NSInteger hour    = [cal component:NSCalendarUnitHour    fromDate:now];
        NSInteger minute  = [cal component:NSCalendarUnitMinute  fromDate:now];
        
        NSLog(@"วันนี้: วัน%@  เวลา %02ld:%02ld",
              dayName((DayOfWeek)weekday), (long)hour, (long)minute);
        NSLog(@"เวลาทำการ: %@", isBusinessHours() ? @"ใช่ (เปิดทำการ)" : @"ไม่ใช่ (ปิดทำการ)");
        
        if (!isBusinessHours()) {
            NSDate *nextWD = nextWorkday();
            NSDateFormatter *df = [[NSDateFormatter alloc] init];
            df.dateFormat = @"EEEE dd/MM/yyyy";
            df.locale = [NSLocale localeWithLocaleIdentifier:@"th_TH"];
            NSLog(@"วันทำการถัดไป: %@", [df stringFromDate:nextWD]);
        }
    }
    return 0;
}
```

---

## 29.13 Performance: Reusing NSDateFormatter

NSDateFormatter มีค่าใช้จ่ายสูงในการสร้าง ควร reuse หรือ cache ไว้:

```objc
#import <Foundation/Foundation.h>

// Singleton formatter ที่ reuse ได้
@interface DateFormatterCache : NSObject
+ (NSDateFormatter *)formatterWithFormat:(NSString *)format;
+ (NSDateFormatter *)iso8601Formatter;
@end

@implementation DateFormatterCache

static NSMutableDictionary<NSString *, NSDateFormatter *> *_cache;

+ (void)initialize {
    if (self == [DateFormatterCache class]) {
        _cache = [NSMutableDictionary dictionary];
    }
}

+ (NSDateFormatter *)formatterWithFormat:(NSString *)format {
    NSDateFormatter *formatter = _cache[format];
    if (!formatter) {
        formatter = [[NSDateFormatter alloc] init];
        formatter.dateFormat = format;
        formatter.locale = [NSLocale localeWithLocaleIdentifier:@"en_US_POSIX"];
        _cache[format] = formatter;
    }
    return formatter;
}

+ (NSDateFormatter *)iso8601Formatter {
    return [self formatterWithFormat:@"yyyy-MM-dd'T'HH:mm:ssZ"];
}

@end

int main(int argc, const char * argv[]) {
    @autoreleasepool {
        NSDate *now = [NSDate date];
        
        // ใช้ formatter ที่ cache ไว้
        NSDateFormatter *df = [DateFormatterCache formatterWithFormat:@"dd/MM/yyyy"];
        NSLog(@"วันที่: %@", [df stringFromDate:now]);
        
        NSDateFormatter *isoDF = [DateFormatterCache iso8601Formatter];
        NSLog(@"ISO: %@", [isoDF stringFromDate:now]);
    }
    return 0;
}
```

---

## 29.14 สรุป Class ที่ใช้ในการจัดการวันที่

| Class                | หน้าที่                                    |
|----------------------|------------------------------------------|
| `NSDate`             | เก็บจุดเวลา (abstract, ไม่มี timezone)    |
| `NSCalendar`         | แปลง NSDate ↔ components, คำนวณวันที่     |
| `NSDateComponents`   | เก็บ year, month, day, hour, minute, etc |
| `NSDateFormatter`    | แปลง NSDate ↔ String                     |
| `NSTimeZone`         | จัดการ timezone                           |
| `NSDateInterval`     | แทนช่วงเวลาระหว่างสองจุด                  |
| `NSISO8601DateFormatter` | parse/format ISO 8601 โดยเฉพาะ     |

---

## แบบฝึกหัด (Practice Exercises)

### แบบฝึกหัดที่ 1: NSDate พื้นฐาน
สร้าง NSDate สำหรับวันที่ 1 มกราคม 1970 00:00:00 UTC (Unix epoch) และแสดงค่า timeIntervalSinceReferenceDate

```objc
// เฉลย
NSDate *epoch = [NSDate dateWithTimeIntervalSince1970:0];
NSLog(@"Unix epoch: %@", epoch);
NSLog(@"Reference interval: %.0f", [epoch timeIntervalSinceReferenceDate]);
// ควรได้ -978307200 (เพราะ reference date คือ 1 ม.ค. 2001)
```

### แบบฝึกหัดที่ 2: เปรียบเทียบวันที่
เขียนฟังก์ชัน `latestDate(NSArray *dates)` ที่รับ array ของ NSDate และคืนค่าวันที่ล่าสุด

```objc
NSDate* latestDate(NSArray *dates) {
    if (dates.count == 0) return nil;
    
    NSDate *latest = dates[0];
    for (NSDate *date in dates) {
        if ([date compare:latest] == NSOrderedDescending) {
            latest = date;
        }
    }
    return latest;
}

// ทดสอบ
NSArray *dates = @[
    [NSDate dateWithTimeIntervalSinceNow:-3600],
    [NSDate date],
    [NSDate dateWithTimeIntervalSinceNow:3600],
    [NSDate dateWithTimeIntervalSinceNow:-86400],
];
NSDate *latest = latestDate(dates);
NSLog(@"วันที่ล่าสุด: %@", latest);
```

### แบบฝึกหัดที่ 3: นับวันทำงาน
เขียนฟังก์ชันนับจำนวนวันทำงาน (จ-ศ) ระหว่างวันที่สองวัน (ไม่นับวันหยุดสาธารณะ)

```objc
NSInteger workdaysBetween(NSDate *startDate, NSDate *endDate) {
    NSCalendar *cal = [NSCalendar calendarWithIdentifier:NSCalendarIdentifierGregorian];
    NSDateComponents *oneDayOffset = [[NSDateComponents alloc] init];
    oneDayOffset.day = 1;
    
    NSInteger count = 0;
    NSDate *current = startDate;
    
    while ([current compare:endDate] == NSOrderedAscending) {
        NSInteger weekday = [cal component:NSCalendarUnitWeekday fromDate:current];
        if (weekday >= 2 && weekday <= 6) { // จ-ศ
            count++;
        }
        current = [cal dateByAddingComponents:oneDayOffset toDate:current options:0];
    }
    
    return count;
}

// ทดสอบ
NSCalendar *cal = [NSCalendar calendarWithIdentifier:NSCalendarIdentifierGregorian];
NSDateComponents *start = [[NSDateComponents alloc] init];
start.year = 2024; start.month = 1; start.day = 1;
NSDateComponents *end = [[NSDateComponents alloc] init];
end.year = 2024; end.month = 1; end.day = 31;

NSInteger days = workdaysBetween([cal dateFromComponents:start],
                                  [cal dateFromComponents:end]);
NSLog(@"วันทำงานในเดือน ม.ค. 2024: %ld วัน", (long)days);
```

### แบบฝึกหัดที่ 4: Time Since
เขียนฟังก์ชัน `timeSinceString(NSDate *date)` ที่คืนค่า string อธิบายเวลาที่ผ่านมา เช่น "เมื่อ 5 นาทีที่แล้ว", "2 ชั่วโมงที่แล้ว", "3 วันที่แล้ว"

```objc
NSString* timeSinceString(NSDate *date) {
    NSTimeInterval diff = [[NSDate date] timeIntervalSinceDate:date];
    
    if (diff < 0) return @"อีกสักพัก";
    if (diff < 60) return [NSString stringWithFormat:@"%.0f วินาทีที่แล้ว", diff];
    if (diff < 3600) return [NSString stringWithFormat:@"%.0f นาทีที่แล้ว", diff/60];
    if (diff < 86400) return [NSString stringWithFormat:@"%.0f ชั่วโมงที่แล้ว", diff/3600];
    if (diff < 86400*7) return [NSString stringWithFormat:@"%.0f วันที่แล้ว", diff/86400];
    if (diff < 86400*30) return [NSString stringWithFormat:@"%.0f สัปดาห์ที่แล้ว", diff/(86400*7)];
    if (diff < 86400*365) return [NSString stringWithFormat:@"%.0f เดือนที่แล้ว", diff/(86400*30)];
    return [NSString stringWithFormat:@"%.0f ปีที่แล้ว", diff/(86400*365)];
}

// ทดสอบ
NSLog(@"%@", timeSinceString([NSDate dateWithTimeIntervalSinceNow:-30]));
NSLog(@"%@", timeSinceString([NSDate dateWithTimeIntervalSinceNow:-300]));
NSLog(@"%@", timeSinceString([NSDate dateWithTimeIntervalSinceNow:-7200]));
NSLog(@"%@", timeSinceString([NSDate dateWithTimeIntervalSinceNow:-86400*3]));
```

### แบบฝึกหัดที่ 5: Calendar Grid
สร้าง calendar grid สำหรับเดือนปัจจุบัน (แสดงเป็น array of arrays ของวัน)

```objc
NSArray* calendarGridForMonth(NSInteger month, NSInteger year) {
    NSCalendar *cal = [NSCalendar calendarWithIdentifier:NSCalendarIdentifierGregorian];
    
    NSDateComponents *comps = [[NSDateComponents alloc] init];
    comps.year  = year;
    comps.month = month;
    comps.day   = 1;
    NSDate *firstDay = [cal dateFromComponents:comps];
    
    NSInteger firstWeekday = [cal component:NSCalendarUnitWeekday fromDate:firstDay];
    NSRange daysRange = [cal rangeOfUnit:NSCalendarUnitDay
                                  inUnit:NSCalendarUnitMonth
                                 forDate:firstDay];
    NSInteger daysInMonth = daysRange.length;
    
    NSMutableArray *weeks = [NSMutableArray array];
    NSMutableArray *currentWeek = [NSMutableArray array];
    
    // เติม padding ก่อนวันแรก
    for (NSInteger i = 1; i < firstWeekday; i++) {
        [currentWeek addObject:@0];
    }
    
    for (NSInteger day = 1; day <= daysInMonth; day++) {
        [currentWeek addObject:@(day)];
        if (currentWeek.count == 7) {
            [weeks addObject:[currentWeek copy]];
            currentWeek = [NSMutableArray array];
        }
    }
    
    // เติม padding หลังวันสุดท้าย
    while (currentWeek.count > 0 && currentWeek.count < 7) {
        [currentWeek addObject:@0];
    }
    if (currentWeek.count > 0) {
        [weeks addObject:currentWeek];
    }
    
    return weeks;
}

// ทดสอบ - แสดง calendar เดือนนี้
NSCalendar *cal = [NSCalendar currentCalendar];
NSDate *now = [NSDate date];
NSInteger month = [cal component:NSCalendarUnitMonth fromDate:now];
NSInteger year  = [cal component:NSCalendarUnitYear  fromDate:now];

NSArray *grid = calendarGridForMonth(month, year);
NSLog(@"อา  จ   อ   พ   พฤ  ศ   ส");
for (NSArray *week in grid) {
    NSMutableString *row = [NSMutableString string];
    for (NSNumber *day in week) {
        if ([day integerValue] == 0) {
            [row appendString:@"    "];
        } else {
            [row appendFormat:@"%3ld ", (long)[day integerValue]];
        }
    }
    NSLog(@"%@", row);
}
```

### แบบฝึกหัดที่ 6: รวม Timezone Converter
เขียน function ที่แปลงเวลาจาก timezone หนึ่งไปอีก timezone หนึ่ง

```objc
NSDate* convertTime(NSDate *date,
                    NSString *fromTimezone,
                    NSString *toTimezone) {
    // NSDate เก็บเวลาเป็น UTC เสมอ ดังนั้นการแปลง timezone
    // ทำได้แค่ตอน format เป็น string เท่านั้น
    // ฟังก์ชันนี้ return NSDate เดิม แต่แสดงผลต่างกัน
    return date; // NSDate is timezone-agnostic
}

NSString* timeString(NSDate *date, NSString *timezone) {
    NSDateFormatter *df = [[NSDateFormatter alloc] init];
    df.dateFormat = @"yyyy-MM-dd HH:mm:ss zzz";
    df.timeZone = [NSTimeZone timeZoneWithName:timezone];
    return [df stringFromDate:date];
}

// ทดสอบ
NSDate *now = [NSDate date];
NSLog(@"Bangkok:      %@", timeString(now, @"Asia/Bangkok"));
NSLog(@"Tokyo:        %@", timeString(now, @"Asia/Tokyo"));
NSLog(@"London:       %@", timeString(now, @"Europe/London"));
NSLog(@"New York:     %@", timeString(now, @"America/New_York"));
NSLog(@"Los Angeles:  %@", timeString(now, @"America/Los_Angeles"));
```

### แบบฝึกหัดที่ 7: Recurring Event Checker
เขียน function ตรวจสอบว่าวันนี้เป็น event ประจำปีหรือไม่ (เช่น วันเกิด, วันครบรอบ)

```objc
BOOL isAnniversaryToday(NSDate *anniversaryDate) {
    NSCalendar *cal = [NSCalendar calendarWithIdentifier:NSCalendarIdentifierGregorian];
    NSDate *today = [NSDate date];
    
    NSInteger aMonth = [cal component:NSCalendarUnitMonth fromDate:anniversaryDate];
    NSInteger aDay   = [cal component:NSCalendarUnitDay   fromDate:anniversaryDate];
    NSInteger tMonth = [cal component:NSCalendarUnitMonth fromDate:today];
    NSInteger tDay   = [cal component:NSCalendarUnitDay   fromDate:today];
    
    return aMonth == tMonth && aDay == tDay;
}

NSInteger yearsOfAnniversary(NSDate *anniversaryDate) {
    NSCalendar *cal = [NSCalendar calendarWithIdentifier:NSCalendarIdentifierGregorian];
    NSDateComponents *diff = [cal components:NSCalendarUnitYear
                                    fromDate:anniversaryDate
                                      toDate:[NSDate date]
                                     options:0];
    return diff.year;
}

// ทดสอบ
NSCalendar *cal = [NSCalendar calendarWithIdentifier:NSCalendarIdentifierGregorian];
NSDateComponents *weddingComps = [[NSDateComponents alloc] init];
weddingComps.year  = 2010;
weddingComps.month = 9;
weddingComps.day   = 30; // วันนี้!

NSDate *weddingDate = [cal dateFromComponents:weddingComps];
if (isAnniversaryToday(weddingDate)) {
    NSLog(@"ฉลองครบรอบปีที่ %ld!", (long)yearsOfAnniversary(weddingDate));
} else {
    NSLog(@"ยังไม่ถึงวันครบรอบ (%ld ปี)", (long)yearsOfAnniversary(weddingDate));
}
```

### แบบฝึกหัดที่ 8: Date Range Generator
สร้าง function ที่ generate array ของวันที่ทั้งหมดในช่วง startDate ถึง endDate

```objc
NSArray<NSDate *>* datesBetween(NSDate *startDate, NSDate *endDate) {
    NSCalendar *cal = [NSCalendar calendarWithIdentifier:NSCalendarIdentifierGregorian];
    NSMutableArray *dates = [NSMutableArray array];
    
    NSDateComponents *oneDayOffset = [[NSDateComponents alloc] init];
    oneDayOffset.day = 1;
    
    NSDate *current = startDate;
    while ([current compare:endDate] != NSOrderedDescending) {
        [dates addObject:current];
        current = [cal dateByAddingComponents:oneDayOffset toDate:current options:0];
    }
    
    return [dates copy];
}

// ทดสอบ
NSCalendar *cal = [NSCalendar calendarWithIdentifier:NSCalendarIdentifierGregorian];
NSDateComponents *startComps = [[NSDateComponents alloc] init];
startComps.year = 2024; startComps.month = 12; startComps.day = 25;
NSDateComponents *endComps = [[NSDateComponents alloc] init];
endComps.year = 2025; endComps.month = 1; endComps.day = 5;

NSArray *range = datesBetween([cal dateFromComponents:startComps],
                               [cal dateFromComponents:endComps]);
NSDateFormatter *df = [[NSDateFormatter alloc] init];
df.dateFormat = @"dd/MM/yyyy EEE";
df.locale = [NSLocale localeWithLocaleIdentifier:@"th_TH"];

NSLog(@"วันที่ในช่วง:");
for (NSDate *d in range) {
    NSLog(@"  %@", [df stringFromDate:d]);
}
```

### แบบฝึกหัดที่ 9: Quarter of Year
เขียน function หา quarter ของปีจากวันที่ (Q1=ม.ค.-มี.ค., Q2=เม.ย.-มิ.ย., Q3=ก.ค.-ก.ย., Q4=ต.ค.-ธ.ค.)

```objc
NSInteger quarterOfYear(NSDate *date) {
    NSCalendar *cal = [NSCalendar calendarWithIdentifier:NSCalendarIdentifierGregorian];
    NSInteger month = [cal component:NSCalendarUnitMonth fromDate:date];
    return (month - 1) / 3 + 1;
}

NSDate* startOfQuarter(NSInteger quarter, NSInteger year) {
    NSCalendar *cal = [NSCalendar calendarWithIdentifier:NSCalendarIdentifierGregorian];
    NSDateComponents *comps = [[NSDateComponents alloc] init];
    comps.year  = year;
    comps.month = (quarter - 1) * 3 + 1;
    comps.day   = 1;
    return [cal dateFromComponents:comps];
}

NSDate* endOfQuarter(NSInteger quarter, NSInteger year) {
    NSCalendar *cal = [NSCalendar calendarWithIdentifier:NSCalendarIdentifierGregorian];
    NSDateComponents *comps = [[NSDateComponents alloc] init];
    comps.year  = year;
    comps.month = quarter * 3;
    comps.day   = 1;
    NSDate *firstDayOfLastMonth = [cal dateFromComponents:comps];
    NSRange range = [cal rangeOfUnit:NSCalendarUnitDay
                              inUnit:NSCalendarUnitMonth
                             forDate:firstDayOfLastMonth];
    comps.day = range.length;
    return [cal dateFromComponents:comps];
}

// ทดสอบ
NSDate *today = [NSDate date];
NSCalendar *cal = [NSCalendar calendarWithIdentifier:NSCalendarIdentifierGregorian];
NSInteger year = [cal component:NSCalendarUnitYear fromDate:today];
NSInteger q = quarterOfYear(today);

NSDateFormatter *df = [[NSDateFormatter alloc] init];
df.dateFormat = @"dd/MM/yyyy";

NSLog(@"วันนี้อยู่ใน Q%ld ของปี %ld", (long)q, (long)year);
NSLog(@"Q%ld เริ่ม: %@", (long)q, [df stringFromDate:startOfQuarter(q, year)]);
NSLog(@"Q%ld สิ้นสุด: %@", (long)q, [df stringFromDate:endOfQuarter(q, year)]);
```

### แบบฝึกหัดที่ 10: Flexible Date Parser
เขียน function ที่ลอง parse string ด้วย format หลาย ๆ แบบ

```objc
NSDate* flexibleDateParse(NSString *dateString) {
    NSArray *formats = @[
        @"yyyy-MM-dd",
        @"dd/MM/yyyy",
        @"MM/dd/yyyy",
        @"dd-MM-yyyy",
        @"d MMM yyyy",
        @"MMMM d, yyyy",
        @"yyyy-MM-dd'T'HH:mm:ss",
        @"yyyy-MM-dd HH:mm:ss",
    ];
    
    NSDateFormatter *df = [[NSDateFormatter alloc] init];
    df.locale = [NSLocale localeWithLocaleIdentifier:@"en_US_POSIX"];
    
    for (NSString *format in formats) {
        df.dateFormat = format;
        NSDate *result = [df dateFromString:dateString];
        if (result) {
            NSLog(@"Parse สำเร็จด้วย format: %@", format);
            return result;
        }
    }
    
    return nil;
}

// ทดสอบ
NSArray *testStrings = @[
    @"2024-12-25",
    @"25/12/2024",
    @"12/25/2024",
    @"25 Dec 2024",
    @"December 25, 2024",
];

for (NSString *s in testStrings) {
    NSDate *parsed = flexibleDateParse(s);
    if (parsed) {
        NSLog(@"'%@' => %@", s, parsed);
    } else {
        NSLog(@"'%@' => ไม่สามารถ parse ได้", s);
    }
}
```

---

## สรุป

ในบทนี้เราได้เรียนรู้:

1. **NSDate** - การสร้างและพื้นฐานของวันที่และเวลาใน Objective-C
2. **การเปรียบเทียบวันที่** - ด้วย compare:, isEqualToDate:, earlierDate:, laterDate:
3. **การคำนวณวันที่** - ด้วย dateByAddingTimeInterval: และ NSCalendar + NSDateComponents
4. **NSCalendar** - การดึง components และคำนวณความต่างระหว่างวันที่
5. **NSDateFormatter** - การ format และ parse วันที่
6. **NSTimeZone** - การจัดการ timezone
7. **Unix timestamps** - การแปลงระหว่าง NSDate และ Unix time
8. **NSDateInterval** - การแทนช่วงเวลา
9. **ตัวอย่างจริง** - Age calculator, countdown timer, schedule checker

ในบทถัดไปเราจะเรียนรู้เรื่อง **NSError และ Exception Handling** สำหรับการจัดการข้อผิดพลาดใน Objective-C
