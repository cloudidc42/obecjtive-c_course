# Part 79: Localization & Internationalization ใน Objective-C

## บทนำ

**Internationalization (i18n)** คือการออกแบบซอฟต์แวร์ให้สามารถปรับเปลี่ยนสำหรับหลายภาษาและวัฒนธรรมได้ง่าย

**Localization (l10n)** คือการปรับแต่งซอฟต์แวร์สำหรับภาษาหรือภูมิภาคเฉพาะ

ทำไมต้องสนใจ Localization?
- ขยายตลาดไปสู่ผู้ใช้ทั่วโลก
- ปรับรูปแบบตัวเลข วันที่ สกุลเงิน ตามภูมิภาค
- รองรับทิศทางข้อความ LTR (ซ้ายไปขวา) และ RTL (ขวาไปซ้าย)
- Apple มีเครื่องมือที่ครบครันสำหรับ Localization

บทนี้จะครอบคลุม:
- **NSLocalizedString** - Macro หลักสำหรับ Localization
- **Localizable.strings** - ไฟล์แปลภาษา
- **NSLocale** - ข้อมูลสถานที่และวัฒนธรรม
- **NSNumberFormatter** - จัดรูปแบบตัวเลขตาม Locale
- **NSDateFormatter** - จัดรูปแบบวันที่ตาม Locale
- **RTL Support** - ภาษาอาหรับ, ฮีบรู
- **Plural Rules** - กฎการใช้พหูพจน์ (.stringsdict)
- **Auto Layout สำหรับ Localization**

---

# หมวดที่ 1: NSLocalizedString

## 79.1 ทำความเข้าใจ NSLocalizedString

```objc
// NSLocalizedString เป็น Macro ที่ขยายเป็น:
// [[NSBundle mainBundle] localizedStringForKey:(key) value:(comment) table:nil]

// ตัวอย่างการใช้งาน
- (void)demonstrateNSLocalizedString {
    
    // 1. การใช้งานพื้นฐาน
    NSString *greeting = NSLocalizedString(@"GREETING", @"ข้อความทักทายในหน้าแรก");
    NSLog(@"%@", greeting);
    
    // 2. NSLocalizedStringWithDefaultValue - ระบุค่า Default
    NSString *title = NSLocalizedStringWithDefaultValue(
        @"NAV_TITLE",              // Key
        nil,                        // Table (nil = Localizable.strings)
        [NSBundle mainBundle],      // Bundle
        @"หน้าหลัก",               // Default Value (ถ้าไม่พบ Key)
        @"Title ของ Navigation Bar" // Comment สำหรับ Translator
    );
    
    // 3. NSLocalizedStringFromTable - ใช้ไฟล์ .strings อื่น
    // เหมาะกับแอปขนาดใหญ่ที่แยก strings หลายไฟล์
    NSString *errorMsg = NSLocalizedStringFromTable(
        @"ERROR_NETWORK",    // Key
        @"Errors",           // Table = Errors.strings
        @"ข้อความแจ้ง Network Error"
    );
    
    // 4. NSLocalizedStringFromTableInBundle - ระบุ Bundle ด้วย
    NSBundle *frameworkBundle = [NSBundle bundleWithIdentifier:@"com.company.framework"];
    NSString *frameworkString = NSLocalizedStringFromTableInBundle(
        @"FRAMEWORK_KEY",
        @"FrameworkStrings",
        frameworkBundle,
        @"String จาก Framework"
    );
    
    NSLog(@"greeting: %@", greeting);
    NSLog(@"title: %@", title);
}
```

## 79.2 โครงสร้าง Localizable.strings

```
// โครงสร้างโปรเจกต์สำหรับ Localization:
//
// MyApp/
// ├── en.lproj/
// │   ├── Localizable.strings
// │   ├── InfoPlist.strings
// │   └── Main.storyboard
// ├── th.lproj/
// │   ├── Localizable.strings
// │   ├── InfoPlist.strings
// │   └── Main.strings
// ├── ja.lproj/
// │   └── Localizable.strings
// └── ar.lproj/
//     └── Localizable.strings (RTL)
```

```objc
// en.lproj/Localizable.strings
/*
"GREETING" = "Welcome to MyApp!";
"LOGIN_TITLE" = "Sign In";
"LOGIN_BUTTON" = "Sign In";
"LOGOUT_BUTTON" = "Sign Out";
"ERROR_NETWORK" = "Network connection failed. Please try again.";
"ERROR_INVALID_EMAIL" = "Please enter a valid email address";
"ITEMS_COUNT" = "%d items";
"PRICE_FORMAT" = "Price: %@";
"NAV_HOME" = "Home";
"NAV_PROFILE" = "Profile";
"NAV_SETTINGS" = "Settings";
*/
```

```
// th.lproj/Localizable.strings
/*
"GREETING" = "ยินดีต้อนรับสู่ MyApp!";
"LOGIN_TITLE" = "เข้าสู่ระบบ";
"LOGIN_BUTTON" = "เข้าสู่ระบบ";
"LOGOUT_BUTTON" = "ออกจากระบบ";
"ERROR_NETWORK" = "การเชื่อมต่อล้มเหลว กรุณาลองใหม่อีกครั้ง";
"ERROR_INVALID_EMAIL" = "กรุณาป้อนที่อยู่อีเมลที่ถูกต้อง";
"ITEMS_COUNT" = "%d รายการ";
"PRICE_FORMAT" = "ราคา: %@";
"NAV_HOME" = "หน้าหลัก";
"NAV_PROFILE" = "โปรไฟล์";
"NAV_SETTINGS" = "การตั้งค่า";
*/
```

## 79.3 LocalizationManager - Wrapper ที่สะดวก

```objc
// LocalizationManager.h
#import <Foundation/Foundation.h>

@interface LocalizationManager : NSObject

+ (instancetype)sharedManager;

/// ดึงข้อความแปลสำหรับ Key ที่ระบุ
- (NSString *)localizedStringForKey:(NSString *)key;

/// ดึงข้อความแปลพร้อม Format Arguments
- (NSString *)localizedStringForKey:(NSString *)key arguments:(NSArray *)args;

/// เปลี่ยนภาษาของแอป (ต้อง Restart)
- (void)setLanguage:(NSString *)languageCode;

/// ภาษาปัจจุบัน
- (NSString *)currentLanguage;

/// ภาษาที่รองรับ
- (NSArray<NSString *> *)supportedLanguages;

/// ตรวจสอบว่าเป็น RTL หรือไม่
- (BOOL)isCurrentLanguageRTL;

@end
```

```objc
// LocalizationManager.m
#import "LocalizationManager.h"

static NSString *const kSelectedLanguageKey = @"SelectedLanguage";

@interface LocalizationManager ()
@property (nonatomic, strong) NSBundle *currentBundle;
@property (nonatomic, copy) NSString *currentLanguageCode;
@end

@implementation LocalizationManager

+ (instancetype)sharedManager {
    static LocalizationManager *instance = nil;
    static dispatch_once_t onceToken;
    dispatch_once(&onceToken, ^{
        instance = [[self alloc] init];
    });
    return instance;
}

- (instancetype)init {
    self = [super init];
    if (self) {
        // โหลดภาษาที่บันทึกไว้ หรือใช้ภาษาของ System
        NSString *savedLanguage = [[NSUserDefaults standardUserDefaults] 
                                   objectForKey:kSelectedLanguageKey];
        
        if (savedLanguage) {
            [self loadBundleForLanguage:savedLanguage];
        } else {
            // ใช้ System Language
            NSString *preferredLanguage = [NSLocale preferredLanguages].firstObject;
            NSString *languageCode = [NSLocale componentsFromLocaleIdentifier:preferredLanguage][NSLocaleLanguageCode];
            [self loadBundleForLanguage:languageCode];
        }
    }
    return self;
}

- (void)loadBundleForLanguage:(NSString *)languageCode {
    NSString *path = [[NSBundle mainBundle] pathForResource:languageCode ofType:@"lproj"];
    
    if (path) {
        _currentBundle = [NSBundle bundleWithPath:path];
        _currentLanguageCode = languageCode;
        NSLog(@"✅ โหลดภาษา: %@", languageCode);
    } else {
        // Fallback ไปใช้ภาษา English
        NSString *enPath = [[NSBundle mainBundle] pathForResource:@"en" ofType:@"lproj"];
        _currentBundle = enPath ? [NSBundle bundleWithPath:enPath] : [NSBundle mainBundle];
        _currentLanguageCode = @"en";
        NSLog(@"⚠️ ไม่พบ Bundle สำหรับ %@, ใช้ English แทน", languageCode);
    }
}

- (NSString *)localizedStringForKey:(NSString *)key {
    NSString *value = [self.currentBundle localizedStringForKey:key 
                                                          value:nil 
                                                          table:nil];
    
    // ถ้าไม่พบ Key ให้ return Key เอง (พร้อมเตือน)
    if ([value isEqualToString:key]) {
        NSLog(@"⚠️ ไม่พบ Localization Key: %@", key);
    }
    
    return value;
}

- (NSString *)localizedStringForKey:(NSString *)key arguments:(NSArray *)args {
    NSString *format = [self localizedStringForKey:key];
    
    if (args.count == 0) {
        return format;
    }
    
    // ใช้ NSString format
    NSMutableArray *cArgs = [NSMutableArray array];
    for (id arg in args) {
        if ([arg isKindOfClass:[NSString class]]) {
            [cArgs addObject:arg];
        } else if ([arg isKindOfClass:[NSNumber class]]) {
            [cArgs addObject:arg];
        }
    }
    
    // สร้าง String ด้วย va_list
    // สำหรับ production ควรใช้ NSString stringWithFormat: กับ varargs
    NSString *result = format;
    return result;
}

- (void)setLanguage:(NSString *)languageCode {
    [self loadBundleForLanguage:languageCode];
    
    // บันทึกการตั้งค่า
    [[NSUserDefaults standardUserDefaults] setObject:languageCode 
                                              forKey:kSelectedLanguageKey];
    [[NSUserDefaults standardUserDefaults] synchronize];
    
    // แจ้ง Observers ว่าภาษาเปลี่ยน
    [[NSNotificationCenter defaultCenter] 
     postNotificationName:@"AppLanguageDidChangeNotification" 
     object:nil 
     userInfo:@{@"language": languageCode}];
}

- (NSString *)currentLanguage {
    return self.currentLanguageCode;
}

- (NSArray<NSString *> *)supportedLanguages {
    NSMutableArray *languages = [NSMutableArray array];
    NSArray *allLocalizations = [[NSBundle mainBundle] localizations];
    
    for (NSString *localization in allLocalizations) {
        if (![localization isEqualToString:@"Base"]) {
            [languages addObject:localization];
        }
    }
    
    return [languages copy];
}

- (BOOL)isCurrentLanguageRTL {
    NSLocale *locale = [NSLocale localeWithLocaleIdentifier:self.currentLanguageCode];
    NSString *characterDirection = [locale objectForKey:NSLocaleExemplarCharacterSet];
    return [NSLocale characterDirectionForLanguage:self.currentLanguageCode] 
           == NSLocaleLanguageDirectionRightToLeft;
}

@end
```

---

# หมวดที่ 2: genstrings และ Workflow

## 79.4 genstrings Command

```bash
# genstrings สแกนโค้ด ObjC/Swift หา NSLocalizedString calls
# แล้วสร้าง/อัปเดต Localizable.strings โดยอัตโนมัติ

# สแกนไฟล์ทั้งหมดใน Project
find . -name "*.m" | xargs genstrings -o en.lproj

# สแกนเฉพาะ Directory
genstrings -o en.lproj Sources/**/*.m

# ตัวเลือกเพิ่มเติม
genstrings -o en.lproj          # Output Directory
          -a                     # Append (ไม่เขียนทับ)
          -s MyLocalizedString   # Custom Macro name
          Sources/**/*.m

# สำหรับ .strings ที่แยกตาม Table
genstrings -o en.lproj -s NSLocalizedStringFromTable Sources/**/*.m
```

```objc
// ตัวอย่างโค้ดที่ genstrings จะสแกนพบ
// ต้องใส่ Comment เสมอ เพราะ genstrings ใช้เป็น Note สำหรับ Translator

NSString *s1 = NSLocalizedString(@"WELCOME", @"ข้อความยินดีต้อนรับในหน้าแรก");
NSString *s2 = NSLocalizedStringWithDefaultValue(@"TITLE", nil, [NSBundle mainBundle], @"MyApp", @"ชื่อแอป");
NSString *s3 = NSLocalizedStringFromTable(@"ERROR_404", @"Errors", @"ข้อความ Error 404");
```

## 79.5 Localization Workflow

```objc
// LocalizationWorkflow.m
// แสดงขั้นตอนการทำ Localization แบบ Professional

@interface LocalizationWorkflow : NSObject
@end

@implementation LocalizationWorkflow

/*
 ขั้นตอนการทำ Localization:
 
 1. Internationalization (i18n) - เตรียม Code
    - ใช้ NSLocalizedString ทุกครั้งที่มี User-visible Text
    - ใช้ NSNumberFormatter, NSDateFormatter ไม่ใช่ stringWithFormat
    - ใช้ Auto Layout สำหรับ Dynamic Text Size
    - อย่า Hard-code ขนาด Width สำหรับ Text
 
 2. Export Strings ด้วย genstrings
    genstrings -o en.lproj Sources/**/*.m
 
 3. สร้าง Localization สำหรับแต่ละภาษา
    - Add Language ใน Xcode Project Settings
    - Copy en.lproj/Localizable.strings ไปแต่ละ .lproj
    - แปลแต่ละ Key
 
 4. ibtool สำหรับ Storyboard/XIB
    ibtool --export-strings-file en.strings Main.storyboard
    ibtool --import-strings-file th.strings Main.storyboard --write Main-th.storyboard
 
 5. Testing
    - Scheme > Run > Options > App Language: Thai
    - ทดสอบ Pseudo-language สำหรับตรวจ Truncation
*/

- (void)showLocalizationBestPractices {
    // ✅ ดี: ใช้ NSLocalizedString
    UILabel *label = [[UILabel alloc] init];
    label.text = NSLocalizedString(@"WELCOME_MESSAGE", @"ข้อความต้อนรับ");
    
    // ❌ ไม่ดี: Hard-code ข้อความ
    // label.text = @"Welcome to MyApp!"; // ❌
    
    // ✅ ดี: ใช้ NSNumberFormatter
    NSNumberFormatter *formatter = [[NSNumberFormatter alloc] init];
    formatter.numberStyle = NSNumberFormatterCurrencyStyle;
    formatter.locale = [NSLocale currentLocale];
    NSString *price = [formatter stringFromNumber:@1500.50];
    NSLog(@"Price: %@", price); // ฿1,500.50 ในไทย, $1,500.50 ในสหรัฐ
    
    // ❌ ไม่ดี: Hard-code รูปแบบตัวเลข
    // NSString *badPrice = [NSString stringWithFormat:@"$%.2f", 1500.50]; // ❌
}

@end
```

---

# หมวดที่ 3: NSLocale

## 79.6 การใช้งาน NSLocale

```objc
// NSLocaleDemo.m
@implementation NSLocaleDemo

- (void)demonstrateLocale {
    // 1. Current Locale (ของ Device)
    NSLocale *currentLocale = [NSLocale currentLocale];
    NSLog(@"Current Locale ID: %@", currentLocale.localeIdentifier); // เช่น th_TH
    NSLog(@"Language Code: %@", currentLocale.languageCode);         // th
    NSLog(@"Country Code: %@", currentLocale.countryCode);           // TH
    NSLog(@"Currency Code: %@", currentLocale.currencyCode);         // THB
    NSLog(@"Currency Symbol: %@", currentLocale.currencySymbol);     // ฿
    
    // 2. Locale Identifier ต่างๆ
    NSLocale *usLocale = [NSLocale localeWithLocaleIdentifier:@"en_US"];
    NSLocale *thLocale = [NSLocale localeWithLocaleIdentifier:@"th_TH"];
    NSLocale *jpLocale = [NSLocale localeWithLocaleIdentifier:@"ja_JP"];
    NSLocale *arLocale = [NSLocale localeWithLocaleIdentifier:@"ar_SA"];
    
    // 3. ข้อมูลจาก Locale
    NSDictionary *components = [NSLocale componentsFromLocaleIdentifier:@"th_TH_u_ca-buddhist"];
    NSLog(@"Language: %@", components[NSLocaleLanguageCode]);     // th
    NSLog(@"Country: %@", components[NSLocaleCountryCode]);       // TH
    NSLog(@"Calendar: %@", components[NSLocaleCalendar]);         // buddhist
    
    // 4. ตรวจสอบทิศทางภาษา
    NSLocaleLanguageDirection direction = [NSLocale characterDirectionForLanguage:@"ar"];
    BOOL isRTL = (direction == NSLocaleLanguageDirectionRightToLeft);
    NSLog(@"Arabic is RTL: %@", isRTL ? @"YES" : @"NO");
    
    // 5. Available Locales
    NSArray *availableLocales = [NSLocale availableLocaleIdentifiers];
    NSLog(@"จำนวน Locale ที่รองรับ: %lu", availableLocales.count);
    
    // 6. Locale Display Names
    NSString *thaiName = [currentLocale displayNameForKey:NSLocaleIdentifier 
                                                    value:@"th_TH"];
    NSString *japanName = [currentLocale displayNameForKey:NSLocaleIdentifier 
                                                     value:@"ja_JP"];
    NSLog(@"ชื่อ Thai ใน Current Locale: %@", thaiName);
    NSLog(@"ชื่อ Japan ใน Current Locale: %@", japanName);
    
    // 7. Preferred Languages ของผู้ใช้
    NSArray *preferred = [NSLocale preferredLanguages];
    NSLog(@"Preferred Languages: %@", preferred);
    
    // 8. System Locale vs Current Locale
    // System Locale = ค่าของ Device
    // Current Locale = อาจถูก Override โดยแอป
    NSLocale *systemLocale = [NSLocale systemLocale];
    NSLog(@"System Locale: %@", systemLocale.localeIdentifier);
    NSLog(@"Current Locale: %@", [NSLocale currentLocale].localeIdentifier);
}

@end
```

---

# หมวดที่ 4: NSNumberFormatter

## 79.7 การ Format ตัวเลขตาม Locale

```objc
// NumberFormatterDemo.m
@implementation NumberFormatterDemo

- (void)demonstrateNumberFormatting {
    NSNumber *number = @1234567.89;
    NSNumber *percentage = @0.1547;
    NSNumber *price = @2999.00;
    
    // 1. Decimal Style (ทั่วไป)
    [self formatNumber:number style:NSNumberFormatterDecimalStyle];
    // th_TH: 1,234,567.89
    // de_DE: 1.234.567,89 (จุดและคอมมาสลับกัน!)
    
    // 2. Currency Style (สกุลเงิน)
    [self formatCurrency:price locale:[NSLocale localeWithLocaleIdentifier:@"th_TH"]];
    [self formatCurrency:price locale:[NSLocale localeWithLocaleIdentifier:@"en_US"]];
    [self formatCurrency:price locale:[NSLocale localeWithLocaleIdentifier:@"ja_JP"]];
    [self formatCurrency:price locale:[NSLocale localeWithLocaleIdentifier:@"de_DE"]];
    
    // 3. Percent Style
    [self formatPercent:percentage];
    
    // 4. Scientific Style
    [self formatScientific:@0.000000001234];
    
    // 5. Spell Out Style
    [self formatSpellOut:@42];
    // en_US: "forty-two"
    // th_TH: "สี่สิบสอง"
}

- (void)formatNumber:(NSNumber *)number style:(NSNumberFormatterStyle)style {
    NSNumberFormatter *formatter = [[NSNumberFormatter alloc] init];
    formatter.numberStyle = style;
    
    // แสดงผลสำหรับ Locales ต่างๆ
    NSArray *locales = @[@"th_TH", @"en_US", @"de_DE", @"ja_JP", @"fr_FR"];
    
    for (NSString *localeID in locales) {
        formatter.locale = [NSLocale localeWithLocaleIdentifier:localeID];
        NSString *formatted = [formatter stringFromNumber:number];
        NSLog(@"[%@] %@", localeID, formatted);
    }
}

- (void)formatCurrency:(NSNumber *)amount locale:(NSLocale *)locale {
    NSNumberFormatter *formatter = [[NSNumberFormatter alloc] init];
    formatter.numberStyle = NSNumberFormatterCurrencyStyle;
    formatter.locale = locale;
    
    NSString *formatted = [formatter stringFromNumber:amount];
    NSLog(@"[%@] %@", locale.localeIdentifier, formatted);
}

- (void)formatPercent:(NSNumber *)number {
    NSNumberFormatter *formatter = [[NSNumberFormatter alloc] init];
    formatter.numberStyle = NSNumberFormatterPercentStyle;
    formatter.locale = [NSLocale currentLocale];
    formatter.maximumFractionDigits = 2;
    
    NSString *formatted = [formatter stringFromNumber:number];
    NSLog(@"Percent: %@", formatted); // 15.47%
}

- (void)formatScientific:(NSNumber *)number {
    NSNumberFormatter *formatter = [[NSNumberFormatter alloc] init];
    formatter.numberStyle = NSNumberFormatterScientificStyle;
    
    NSString *formatted = [formatter stringFromNumber:number];
    NSLog(@"Scientific: %@", formatted); // 1.234E-9
}

- (void)formatSpellOut:(NSNumber *)number {
    NSNumberFormatter *formatter = [[NSNumberFormatter alloc] init];
    formatter.numberStyle = NSNumberFormatterSpellOutStyle;
    
    NSArray *locales = @[@"th_TH", @"en_US", @"ja_JP"];
    for (NSString *localeID in locales) {
        formatter.locale = [NSLocale localeWithLocaleIdentifier:localeID];
        NSLog(@"[%@] %@: %@", localeID, number, [formatter stringFromNumber:number]);
    }
}

// ตัวอย่างการตั้งค่า Formatter แบบละเอียด
- (NSNumberFormatter *)createCurrencyFormatterForThai {
    NSNumberFormatter *formatter = [[NSNumberFormatter alloc] init];
    formatter.numberStyle = NSNumberFormatterCurrencyStyle;
    formatter.locale = [NSLocale localeWithLocaleIdentifier:@"th_TH"];
    formatter.currencyCode = @"THB";
    formatter.minimumFractionDigits = 0;
    formatter.maximumFractionDigits = 2;
    formatter.groupingSeparator = @",";
    formatter.decimalSeparator = @".";
    
    return formatter;
}

// การ Parse ตัวเลขจาก String (Locale-aware)
- (void)parseNumberFromString:(NSString *)numberString {
    NSNumberFormatter *formatter = [[NSNumberFormatter alloc] init];
    formatter.numberStyle = NSNumberFormatterDecimalStyle;
    formatter.locale = [NSLocale currentLocale];
    
    NSNumber *number = [formatter numberFromString:numberString];
    if (number) {
        NSLog(@"Parsed: %@", number);
    } else {
        NSLog(@"ไม่สามารถ Parse '%@' ได้สำหรับ Locale %@", 
              numberString, formatter.locale.localeIdentifier);
    }
}

@end
```

---

# หมวดที่ 5: NSDateFormatter

## 79.8 การ Format วันที่และเวลาตาม Locale

```objc
// DateFormatterDemo.m
@implementation DateFormatterDemo

- (void)demonstrateDateFormatting {
    NSDate *now = [NSDate date];
    
    // 1. การใช้งาน Style ต่างๆ
    [self formatDate:now style:NSDateFormatterShortStyle];
    [self formatDate:now style:NSDateFormatterMediumStyle];
    [self formatDate:now style:NSDateFormatterLongStyle];
    [self formatDate:now style:NSDateFormatterFullStyle];
    
    // 2. Date + Time
    [self formatDateTime:now];
    
    // 3. Custom Format
    [self formatWithCustomPattern:now];
    
    // 4. Calendar ต่างๆ
    [self demonstrateCalendars:now];
    
    // 5. Relative Date
    [self demonstrateRelativeDate];
}

- (void)formatDate:(NSDate *)date style:(NSDateFormatterStyle)style {
    NSDateFormatter *formatter = [[NSDateFormatter alloc] init];
    formatter.dateStyle = style;
    formatter.timeStyle = NSDateFormatterNoStyle;
    
    NSArray *locales = @[@"th_TH", @"en_US", @"ja_JP", @"de_DE", @"ar_SA"];
    
    NSLog(@"--- Style: %ld ---", (long)style);
    for (NSString *localeID in locales) {
        formatter.locale = [NSLocale localeWithLocaleIdentifier:localeID];
        
        // สำคัญ: ต้องตั้ง Locale ก่อน timeZone
        formatter.timeZone = [NSTimeZone localTimeZone];
        
        NSString *formatted = [formatter stringFromDate:date];
        NSLog(@"[%@] %@", localeID, formatted);
    }
}

- (void)formatDateTime:(NSDate *)date {
    NSDateFormatter *formatter = [[NSDateFormatter alloc] init];
    formatter.dateStyle = NSDateFormatterMediumStyle;
    formatter.timeStyle = NSDateFormatterShortStyle;
    formatter.locale = [NSLocale currentLocale];
    
    NSLog(@"DateTime: %@", [formatter stringFromDate:date]);
}

- (void)formatWithCustomPattern:(NSDate *)date {
    NSDateFormatter *formatter = [[NSDateFormatter alloc] init];
    formatter.locale = [NSLocale localeWithLocaleIdentifier:@"th_TH"];
    
    // ⚠️ ระวัง: Pattern ตาม Unicode Date Format Patterns
    // ไม่ใช่รูปแบบเดียวกับ strftime ใน C
    
    // วันที่แบบไทย
    formatter.dateFormat = @"วันEEEEที่ d MMMM yyyy";
    NSLog(@"Thai Date: %@", [formatter stringFromDate:date]);
    // ผลลัพธ์: "วันพุธที่ 30 กันยายน 2568"
    
    // รูปแบบ ISO 8601
    formatter.dateFormat = @"yyyy-MM-dd'T'HH:mm:ssZ";
    formatter.locale = [NSLocale localeWithLocaleIdentifier:@"en_US_POSIX"];
    NSLog(@"ISO 8601: %@", [formatter stringFromDate:date]);
    
    // รูปแบบ Custom ภาษาอังกฤษ
    formatter.locale = [NSLocale localeWithLocaleIdentifier:@"en_US"];
    formatter.dateFormat = @"MMM dd, yyyy 'at' h:mm a";
    NSLog(@"Custom EN: %@", [formatter stringFromDate:date]);
}

- (void)demonstrateCalendars:(NSDate *)date {
    NSDateFormatter *formatter = [[NSDateFormatter alloc] init];
    formatter.dateStyle = NSDateFormatterLongStyle;
    formatter.timeStyle = NSDateFormatterNoStyle;
    
    // 1. Buddhist Calendar (ปฏิทินพุทธศักราช - ใช้ในไทย)
    formatter.locale = [NSLocale localeWithLocaleIdentifier:@"th_TH_u_ca-buddhist"];
    NSLog(@"Buddhist Calendar: %@", [formatter stringFromDate:date]);
    // ผลลัพธ์: "30 กันยายน 2568"
    
    // 2. Gregorian Calendar (สากล)
    formatter.locale = [NSLocale localeWithLocaleIdentifier:@"en_US"];
    formatter.calendar = [NSCalendar calendarWithIdentifier:NSCalendarIdentifierGregorian];
    NSLog(@"Gregorian: %@", [formatter stringFromDate:date]);
    
    // 3. Japanese Imperial Calendar
    formatter.locale = [NSLocale localeWithLocaleIdentifier:@"ja_JP_u_ca-japanese"];
    NSLog(@"Japanese Imperial: %@", [formatter stringFromDate:date]);
    
    // 4. Islamic Calendar
    formatter.locale = [NSLocale localeWithLocaleIdentifier:@"ar_SA_u_ca-islamic"];
    NSLog(@"Islamic: %@", [formatter stringFromDate:date]);
    
    // 5. Hebrew Calendar
    formatter.locale = [NSLocale localeWithLocaleIdentifier:@"he_IL_u_ca-hebrew"];
    NSLog(@"Hebrew: %@", [formatter stringFromDate:date]);
}

- (void)demonstrateRelativeDate {
    NSDateFormatter *formatter = [[NSDateFormatter alloc] init];
    formatter.locale = [NSLocale currentLocale];
    formatter.doesRelativeDateFormatting = YES;
    formatter.dateStyle = NSDateFormatterMediumStyle;
    formatter.timeStyle = NSDateFormatterNoStyle;
    
    NSDate *yesterday = [NSDate dateWithTimeIntervalSinceNow:-86400];
    NSDate *tomorrow = [NSDate dateWithTimeIntervalSinceNow:86400];
    NSDate *twoDaysAgo = [NSDate dateWithTimeIntervalSinceNow:-172800];
    
    NSLog(@"เมื่อวาน: %@", [formatter stringFromDate:yesterday]);
    NSLog(@"พรุ่งนี้: %@", [formatter stringFromDate:tomorrow]);
    NSLog(@"สองวันที่แล้ว: %@", [formatter stringFromDate:twoDaysAgo]);
    // th: "เมื่อวาน", "พรุ่งนี้"
    // en: "Yesterday", "Tomorrow"
}

// ⚠️ Best Practice: Cache NSDateFormatter เพราะ init มีค่าใช้จ่ายสูง
+ (NSDateFormatter *)cachedFormatterForStyle:(NSDateFormatterStyle)style 
                                  localeID:(NSString *)localeID {
    static NSMutableDictionary *cache;
    static dispatch_once_t onceToken;
    dispatch_once(&onceToken, ^{
        cache = [NSMutableDictionary dictionary];
    });
    
    NSString *key = [NSString stringWithFormat:@"%ld-%@", (long)style, localeID];
    
    NSDateFormatter *formatter = cache[key];
    if (!formatter) {
        formatter = [[NSDateFormatter alloc] init];
        formatter.dateStyle = style;
        formatter.timeStyle = NSDateFormatterNoStyle;
        formatter.locale = [NSLocale localeWithLocaleIdentifier:localeID];
        cache[key] = formatter;
    }
    
    return formatter;
}

@end
```

---

# หมวดที่ 6: RTL Language Support

## 79.9 การรองรับภาษา RTL (Right-to-Left)

```objc
// RTLSupportViewController.m
// รองรับภาษาอาหรับ, ฮีบรู ที่เขียนจากขวาไปซ้าย

@interface RTLSupportViewController : UIViewController
@end

@implementation RTLSupportViewController

- (void)viewDidLoad {
    [super viewDidLoad];
    [self setupRTLAwareUI];
}

- (void)setupRTLAwareUI {
    BOOL isRTL = [UIApplication sharedApplication].userInterfaceLayoutDirection == UIUserInterfaceLayoutDirectionRightToLeft;
    NSLog(@"Layout Direction: %@", isRTL ? @"RTL" : @"LTR");
    
    // 1. Auto Layout จะ Mirror อัตโนมัติสำหรับ RTL
    // ใช้ Leading/Trailing แทน Left/Right เสมอ
    
    UILabel *titleLabel = [[UILabel alloc] init];
    titleLabel.translatesAutoresizingMaskIntoConstraints = NO;
    titleLabel.text = NSLocalizedString(@"TITLE", @"หัวข้อ");
    titleLabel.textAlignment = NSTextAlignmentNatural; // ปรับตามภาษา
    [self.view addSubview:titleLabel];
    
    // ✅ ใช้ Leading/Trailing (จะ Flip สำหรับ RTL)
    [NSLayoutConstraint activateConstraints:@[
        [titleLabel.leadingAnchor constraintEqualToAnchor:self.view.leadingAnchor constant:16],
        [titleLabel.trailingAnchor constraintEqualToAnchor:self.view.trailingAnchor constant:-16],
        [titleLabel.topAnchor constraintEqualToAnchor:self.view.safeAreaLayoutGuide.topAnchor constant:16],
    ]];
    
    // ❌ อย่าใช้ Left/Right (จะไม่ Flip)
    // [titleLabel.leftAnchor constraintEqualToAnchor:self.view.leftAnchor constant:16]
    
    // 2. UIStackView ที่ Flip อัตโนมัติ
    UIStackView *buttonStack = [[UIStackView alloc] init];
    buttonStack.axis = UILayoutConstraintAxisHorizontal;
    buttonStack.spacing = 8;
    // UIStackView จะ Flip ลำดับใน RTL โดยอัตโนมัติ
    
    // 3. Image ที่ต้อง Flip ใน RTL
    UIImage *arrowImage = [[UIImage imageNamed:@"arrow_right"] 
                            imageWithHorizontallyFlippedOrientation];
    // ใช้ imageFlippedForRightToLeftLayoutDirection ใน iOS 9+
    UIImage *rtlArrow = [UIImage imageNamed:@"arrow_right"];
    rtlArrow = rtlArrow.imageFlippedForRightToLeftLayoutDirection;
    
    // 4. Text Alignment ที่ Natural
    UITextField *textField = [[UITextField alloc] init];
    textField.textAlignment = NSTextAlignmentNatural; // จัดตามภาษา
    // RTL: จัดชิดขวา
    // LTR: จัดชิดซ้าย
    
    // 5. ตรวจสอบ RTL ด้วยตนเอง
    if (isRTL) {
        // ปรับ Custom Drawing หรือ Manual Layout
        NSLog(@"ปรับ Layout สำหรับ RTL");
    }
    
    // 6. Semantic Content Attribute สำหรับ Views ที่ไม่ควร Flip
    UIView *mapView = [[UIView alloc] init];
    // แผนที่ไม่ควร Mirror
    mapView.semanticContentAttribute = UISemanticContentAttributeSpatial;
    
    UIView *videoPlayer = [[UIView alloc] init];
    // Video Player ไม่ควร Mirror
    videoPlayer.semanticContentAttribute = UISemanticContentAttributePlayback;
    
    // 7. Force LTR สำหรับ Specific View
    UIView *codeBlock = [[UIView alloc] init];
    codeBlock.semanticContentAttribute = UISemanticContentAttributeForceLeftToRight;
}

// ตรวจสอบว่า Device ใช้ RTL หรือไม่
- (BOOL)isRTLLanguage {
    return [[UIApplication sharedApplication] userInterfaceLayoutDirection] 
           == UIUserInterfaceLayoutDirectionRightToLeft;
}

// ปรับ Auto Layout สำหรับ RTL
- (NSLayoutConstraint *)leadingConstraintForView:(UIView *)view 
                                       toAnchor:(NSLayoutAnchor *)anchor 
                                       constant:(CGFloat)constant {
    // ใช้ leadingAnchor เสมอ - Auto Layout จะ flip ให้เอง
    return [view.leadingAnchor constraintEqualToAnchor:(NSLayoutXAxisAnchor *)anchor constant:constant];
}

@end
```

---

# หมวดที่ 7: Plural Rules ด้วย .stringsdict

## 79.10 การจัดการ Plural Rules

ภาษาต่างๆ มีกฎ Plural ต่างกัน เช่น:
- ภาษาไทย: ไม่มีกฎ Plural (1 รายการ, 2 รายการ)
- ภาษาอังกฤษ: singular vs plural (1 item, 2 items)
- ภาษารัสเซีย: มีหลายรูปแบบ (1 рубль, 2 рубля, 5 рублей)
- ภาษาอาหรับ: มีถึง 6 รูปแบบ

```xml
<!-- en.lproj/Localizable.stringsdict -->
<?xml version="1.0" encoding="UTF-8"?>
<!DOCTYPE plist PUBLIC "-//Apple//DTD PLIST 1.0//EN" 
    "http://www.apple.com/DTDs/PropertyList-1.0.dtd">
<plist version="1.0">
<dict>
    <!-- Key สำหรับนับจำนวนรายการ -->
    <key>ITEMS_COUNT</key>
    <dict>
        <key>NSStringLocalizedFormatKey</key>
        <string>%#@items@</string>
        <key>items</key>
        <dict>
            <key>NSStringFormatSpecTypeKey</key>
            <string>NSStringPluralRuleType</string>
            <key>NSStringFormatValueTypeKey</key>
            <string>d</string>
            <key>zero</key>
            <string>No items</string>
            <key>one</key>
            <string>%d item</string>
            <key>other</key>
            <string>%d items</string>
        </dict>
    </dict>
    
    <!-- Key ซับซ้อน: "%d messages from %d people" -->
    <key>MESSAGES_FROM_PEOPLE</key>
    <dict>
        <key>NSStringLocalizedFormatKey</key>
        <string>%1$#@messages@ from %2$#@people@</string>
        <key>messages</key>
        <dict>
            <key>NSStringFormatSpecTypeKey</key>
            <string>NSStringPluralRuleType</string>
            <key>NSStringFormatValueTypeKey</key>
            <string>d</string>
            <key>one</key>
            <string>%d message</string>
            <key>other</key>
            <string>%d messages</string>
        </dict>
        <key>people</key>
        <dict>
            <key>NSStringFormatSpecTypeKey</key>
            <string>NSStringPluralRuleType</string>
            <key>NSStringFormatValueTypeKey</key>
            <string>d</string>
            <key>one</key>
            <string>%d person</string>
            <key>other</key>
            <string>%d people</string>
        </dict>
    </dict>
</dict>
</plist>
```

```xml
<!-- th.lproj/Localizable.stringsdict - ภาษาไทยไม่มี Plural -->
<?xml version="1.0" encoding="UTF-8"?>
<!DOCTYPE plist PUBLIC "-//Apple//DTD PLIST 1.0//EN" 
    "http://www.apple.com/DTDs/PropertyList-1.0.dtd">
<plist version="1.0">
<dict>
    <key>ITEMS_COUNT</key>
    <dict>
        <key>NSStringLocalizedFormatKey</key>
        <string>%#@items@</string>
        <key>items</key>
        <dict>
            <key>NSStringFormatSpecTypeKey</key>
            <string>NSStringPluralRuleType</string>
            <key>NSStringFormatValueTypeKey</key>
            <string>d</string>
            <key>zero</key>
            <string>ไม่มีรายการ</string>
            <key>other</key>
            <string>%d รายการ</string>
        </dict>
    </dict>
    
    <key>MESSAGES_FROM_PEOPLE</key>
    <dict>
        <key>NSStringLocalizedFormatKey</key>
        <string>%1$#@messages@ จาก %2$#@people@</string>
        <key>messages</key>
        <dict>
            <key>NSStringFormatSpecTypeKey</key>
            <string>NSStringPluralRuleType</string>
            <key>NSStringFormatValueTypeKey</key>
            <string>d</string>
            <key>other</key>
            <string>%d ข้อความ</string>
        </dict>
        <key>people</key>
        <dict>
            <key>NSStringFormatSpecTypeKey</key>
            <string>NSStringPluralRuleType</string>
            <key>NSStringFormatValueTypeKey</key>
            <string>d</string>
            <key>other</key>
            <string>%d คน</string>
        </dict>
    </dict>
</dict>
</plist>
```

```objc
// การใช้งาน .stringsdict
- (void)demonstratePluralRules {
    // ใช้ NSLocalizedString กับ stringsdict เหมือนกันเลย
    // แต่ต้องใช้ [NSString localizedStringWithFormat:]
    
    NSInteger count = 3;
    
    // ✅ วิธีที่ถูก: ใช้ localizedStringWithFormat:
    NSString *itemsText = [NSString localizedStringWithFormat:
                           NSLocalizedString(@"ITEMS_COUNT", @"จำนวนรายการ"), 
                           count];
    NSLog(@"%@", itemsText); // EN: "3 items" / TH: "3 รายการ"
    
    // ตัวอย่าง Complex Plural
    NSInteger messages = 5;
    NSInteger people = 1;
    NSString *complexText = [NSString localizedStringWithFormat:
                             NSLocalizedString(@"MESSAGES_FROM_PEOPLE", @"ข้อความจากผู้ส่ง"),
                             messages, people];
    NSLog(@"%@", complexText);
    // EN: "5 messages from 1 person"
    // TH: "5 ข้อความ จาก 1 คน"
    
    // ทดสอบ Edge Cases
    for (NSInteger i = 0; i <= 5; i++) {
        NSString *text = [NSString localizedStringWithFormat:
                          NSLocalizedString(@"ITEMS_COUNT", @""), i];
        NSLog(@"%ld -> %@", (long)i, text);
    }
}
```

---

# หมวดที่ 8: NSAttributedString กับ Localization

## 79.11 Localized Attributed Strings

```objc
// LocalizedAttributedStrings.m
@implementation LocalizedAttributedStrings

// สร้าง Attributed String ที่ Localized
- (NSAttributedString *)createAttributedWelcomeMessage {
    // Pattern: "ยินดีต้อนรับ, {ชื่อผู้ใช้}! คุณมี {จำนวน} ข้อความ"
    // เราต้องการให้ ชื่อผู้ใช้ เป็น Bold และ จำนวน เป็นสีแดง
    
    NSString *username = @"สมชาย";
    NSInteger messageCount = 5;
    
    // Template ที่มี Placeholders
    // ใส่ใน Localizable.strings:
    // "WELCOME_FORMAT" = "ยินดีต้อนรับ, %1$@! คุณมี %2$ld ข้อความใหม่";
    
    NSString *template = NSLocalizedString(@"WELCOME_FORMAT", 
                                           @"ข้อความต้อนรับพร้อมชื่อและจำนวนข้อความ");
    NSString *fullText = [NSString stringWithFormat:template, username, messageCount];
    
    NSMutableAttributedString *attrString = [[NSMutableAttributedString alloc] 
                                             initWithString:fullText];
    
    // หา Range ของ username
    NSRange usernameRange = [fullText rangeOfString:username];
    if (usernameRange.location != NSNotFound) {
        [attrString addAttribute:NSFontAttributeName 
                           value:[UIFont boldSystemFontOfSize:17]
                           range:usernameRange];
    }
    
    // หา Range ของตัวเลข
    NSString *countString = [NSString stringWithFormat:@"%ld", (long)messageCount];
    NSRange countRange = [fullText rangeOfString:countString];
    if (countRange.location != NSNotFound) {
        [attrString addAttribute:NSForegroundColorAttributeName 
                           value:[UIColor systemRedColor]
                           range:countRange];
        [attrString addAttribute:NSFontAttributeName 
                           value:[UIFont boldSystemFontOfSize:17]
                           range:countRange];
    }
    
    return [attrString copy];
}

// สร้าง Attributed String สำหรับ Terms & Conditions
- (NSAttributedString *)createTermsAndConditionsText {
    // "โดยการลงทะเบียน คุณยอมรับ ข้อกำหนดการใช้งาน และ นโยบายความเป็นส่วนตัว ของเรา"
    // ต้องการให้ "ข้อกำหนดการใช้งาน" และ "นโยบายความเป็นส่วนตัว" เป็น Link
    
    NSString *termsText = NSLocalizedString(@"TERMS_LINK_TEXT", @"ข้อกำหนดการใช้งาน");
    NSString *privacyText = NSLocalizedString(@"PRIVACY_LINK_TEXT", @"นโยบายความเป็นส่วนตัว");
    NSString *fullText = NSLocalizedString(@"TERMS_FULL_TEXT", 
                                           @"ข้อความเต็มพร้อม Terms และ Privacy");
    
    NSMutableAttributedString *attrString = [[NSMutableAttributedString alloc] 
                                             initWithString:fullText];
    
    // Base Style
    [attrString addAttribute:NSFontAttributeName 
                       value:[UIFont preferredFontForTextStyle:UIFontTextStyleFootnote]
                       range:NSMakeRange(0, fullText.length)];
    [attrString addAttribute:NSForegroundColorAttributeName 
                       value:[UIColor labelColor]
                       range:NSMakeRange(0, fullText.length)];
    
    // Terms Link
    NSRange termsRange = [fullText rangeOfString:termsText];
    if (termsRange.location != NSNotFound) {
        [attrString addAttribute:NSLinkAttributeName 
                           value:[NSURL URLWithString:@"https://example.com/terms"]
                           range:termsRange];
        [attrString addAttribute:NSForegroundColorAttributeName 
                           value:[UIColor systemBlueColor]
                           range:termsRange];
    }
    
    // Privacy Link
    NSRange privacyRange = [fullText rangeOfString:privacyText];
    if (privacyRange.location != NSNotFound) {
        [attrString addAttribute:NSLinkAttributeName 
                           value:[NSURL URLWithString:@"https://example.com/privacy"]
                           range:privacyRange];
        [attrString addAttribute:NSForegroundColorAttributeName 
                           value:[UIColor systemBlueColor]
                           range:privacyRange];
    }
    
    return [attrString copy];
}

@end
```

---

# หมวดที่ 9: Auto Layout สำหรับ Localized UI

## 79.12 การออกแบบ Layout ที่รองรับข้อความหลายภาษา

```objc
// LocalizedLayoutViewController.m
// Layout ที่รองรับข้อความที่มีความยาวต่างกัน

@interface LocalizedLayoutViewController : UIViewController
@end

@implementation LocalizedLayoutViewController

- (void)viewDidLoad {
    [super viewDidLoad];
    [self setupFlexibleLayout];
}

- (void)setupFlexibleLayout {
    // ❌ ปัญหา: Fixed Width ทำให้ข้อความถูกตัดในบางภาษา
    // UILabel *badLabel = [[UILabel alloc] initWithFrame:CGRectMake(0, 0, 100, 30)];
    // badLabel.text = NSLocalizedString(@"LOGIN_BUTTON", @"");
    // "Sign In" = OK, "เข้าสู่ระบบ" = อาจเต็ม, "Inloggen" (Dutch) = เต็มแน่
    
    // ✅ วิธีที่ดี: ใช้ Auto Layout กับ Intrinsic Content Size
    
    // Button ที่ขยายตาม Text
    UIButton *loginButton = [UIButton buttonWithType:UIButtonTypeSystem];
    loginButton.translatesAutoresizingMaskIntoConstraints = NO;
    [loginButton setTitle:NSLocalizedString(@"LOGIN_BUTTON", @"ปุ่มเข้าสู่ระบบ") 
                 forState:UIControlStateNormal];
    loginButton.backgroundColor = [UIColor systemBlueColor];
    [loginButton setTitleColor:[UIColor whiteColor] forState:UIControlStateNormal];
    loginButton.layer.cornerRadius = 8;
    loginButton.contentEdgeInsets = UIEdgeInsetsMake(12, 24, 12, 24); // Padding
    [self.view addSubview:loginButton];
    
    // ✅ ไม่กำหนด Width - ปล่อยให้ขยายตาม Text
    [NSLayoutConstraint activateConstraints:@[
        [loginButton.centerXAnchor constraintEqualToAnchor:self.view.centerXAnchor],
        [loginButton.topAnchor constraintEqualToAnchor:self.view.safeAreaLayoutGuide.topAnchor constant:20],
        // ✅ กำหนด min width เท่านั้น
        [loginButton.widthAnchor constraintGreaterThanOrEqualToConstant:120],
        // ✅ ไม่เกิน screen width
        [loginButton.leadingAnchor constraintGreaterThanOrEqualToAnchor:self.view.leadingAnchor constant:20],
        [loginButton.trailingAnchor constraintLessThanOrEqualToAnchor:self.view.trailingAnchor constant:-20],
    ]];
    
    // Label ที่รองรับหลายบรรทัด
    UILabel *descLabel = [[UILabel alloc] init];
    descLabel.translatesAutoresizingMaskIntoConstraints = NO;
    descLabel.text = NSLocalizedString(@"WELCOME_DESC", @"คำอธิบายการต้อนรับ");
    descLabel.numberOfLines = 0; // ✅ ให้ขยายเป็นหลายบรรทัดได้
    descLabel.font = [UIFont preferredFontForTextStyle:UIFontTextStyleBody];
    descLabel.adjustsFontForContentSizeCategory = YES;
    [self.view addSubview:descLabel];
    
    [NSLayoutConstraint activateConstraints:@[
        [descLabel.topAnchor constraintEqualToAnchor:loginButton.bottomAnchor constant:20],
        [descLabel.leadingAnchor constraintEqualToAnchor:self.view.leadingAnchor constant:16],
        [descLabel.trailingAnchor constraintEqualToAnchor:self.view.trailingAnchor constant:-16],
    ]];
    
    // Stack View ที่ปรับทิศทางตามภาษา
    UILabel *nameLabel = [[UILabel alloc] init];
    nameLabel.text = NSLocalizedString(@"USERNAME_LABEL", @"Label สำหรับชื่อผู้ใช้");
    
    UITextField *nameField = [[UITextField alloc] init];
    nameField.placeholder = NSLocalizedString(@"USERNAME_PLACEHOLDER", @"");
    nameField.borderStyle = UITextBorderStyleRoundedRect;
    
    UIStackView *nameStack = [[UIStackView alloc] initWithArrangedSubviews:@[nameLabel, nameField]];
    nameStack.axis = UILayoutConstraintAxisHorizontal;
    nameStack.spacing = 8;
    nameStack.distribution = UIStackViewDistributionFill;
    nameStack.translatesAutoresizingMaskIntoConstraints = NO;
    [self.view addSubview:nameStack];
    
    // Priority สำหรับ Content Hugging
    [nameLabel setContentHuggingPriority:UILayoutPriorityDefaultHigh 
                                 forAxis:UILayoutConstraintAxisHorizontal];
    
    [NSLayoutConstraint activateConstraints:@[
        [nameStack.topAnchor constraintEqualToAnchor:descLabel.bottomAnchor constant:16],
        [nameStack.leadingAnchor constraintEqualToAnchor:self.view.leadingAnchor constant:16],
        [nameStack.trailingAnchor constraintEqualToAnchor:self.view.trailingAnchor constant:-16],
    ]];
}

// ปรับ Layout เมื่อภาษาเปลี่ยน
- (void)languageDidChange:(NSNotification *)notification {
    // อัปเดต Text ทั้งหมด
    [self updateLocalizedTexts];
    
    // ถ้าเปลี่ยนจาก LTR เป็น RTL หรือกลับกัน ต้อง Reload View
    [self.view setNeedsLayout];
    [self.view layoutIfNeeded];
}

- (void)updateLocalizedTexts {
    // Update ทุก UI Element ที่มี Localized Text
    // (ควรใช้ Data Binding หรือ ViewModel)
}

@end
```

---

# หมวดที่ 10: ibtool สำหรับ Storyboard

## 79.13 การ Localize Storyboard ด้วย ibtool

```bash
# Export strings จาก Storyboard
ibtool --export-strings-file en.strings Main.storyboard

# ผลลัพธ์ en.strings:
# /* Class = "UILabel"; text = "Welcome"; ObjectID = "abc-123-def"; */
# "abc-123-def.text" = "Welcome";
# /* Class = "UIButton"; normalTitle = "Sign In"; ObjectID = "xyz-456-ghi"; */
# "xyz-456-ghi.normalTitle" = "Sign In";

# แปลเป็นภาษาไทย (th.strings):
# "abc-123-def.text" = "ยินดีต้อนรับ";
# "xyz-456-ghi.normalTitle" = "เข้าสู่ระบบ";

# Import strings กลับเข้า Storyboard
ibtool --import-strings-file th.strings \
       --write th.lproj/Main.storyboard \
       Main.storyboard

# ตรวจสอบ Localized Storyboard
ibtool --print-strings Main.storyboard
```

```objc
// NSBundle Localization
// โหลด Resource จาก Bundle ที่ถูกต้องสำหรับ Language ปัจจุบัน

@implementation NSBundleLocalizationDemo

- (void)demonstrateBundleLocalization {
    // 1. โหลดรูปภาพตาม Locale
    // iOS จะค้นหาใน Language Bundle ก่อน แล้วค่อย Fallback ไป Base
    UIImage *localizedImage = [UIImage imageNamed:@"welcome_banner"];
    // ถ้ามี th.lproj/welcome_banner.png จะโหลดภาษาไทย
    // ถ้าไม่มีก็จะ Fallback ไป en.lproj/welcome_banner.png หรือ Assets
    
    // 2. โหลด Nib/XIB ตาม Locale
    // Xcode จะสร้าง th.lproj/MyView.xib เมื่อ Localize
    NSArray *nib = [[NSBundle mainBundle] loadNibNamed:@"ProfileCard" 
                                                 owner:nil 
                                               options:nil];
    
    // 3. โหลด Resource File ตาม Locale
    NSString *helpFilePath = [[NSBundle mainBundle] pathForResource:@"help" 
                                                             ofType:@"html"];
    // iOS จะหาใน th.lproj/help.html ก่อน
    
    // 4. โหลดจาก Specific Bundle
    NSString *language = [[LocalizationManager sharedManager] currentLanguage];
    NSString *bundlePath = [[NSBundle mainBundle] pathForResource:language ofType:@"lproj"];
    NSBundle *languageBundle = bundlePath ? [NSBundle bundleWithPath:bundlePath] : [NSBundle mainBundle];
    
    NSString *key = @"HELP_TITLE";
    NSString *localizedString = [languageBundle localizedStringForKey:key 
                                                               value:key 
                                                               table:nil];
    NSLog(@"Localized: %@", localizedString);
}

@end
```

---

# แบบฝึกหัดท้ายบท

## แบบฝึกหัดที่ 1: Multi-Language Settings Screen

สร้าง Settings Screen ที่:
- แสดงภาษาที่รองรับทั้งหมด
- เปลี่ยนภาษาของแอปได้ทันที
- บันทึกการตั้งค่าภาษาไว้

```objc
// LanguageSettingsViewController.m
@interface LanguageSettingsViewController : UITableViewController

@property (nonatomic, strong) NSArray<NSDictionary *> *languages;

@end

@implementation LanguageSettingsViewController

- (void)viewDidLoad {
    [super viewDidLoad];
    
    self.title = NSLocalizedString(@"SETTINGS_LANGUAGE", @"หัวข้อหน้า Language Settings");
    
    // ภาษาที่รองรับ
    self.languages = @[
        @{@"code": @"th", @"name": @"ภาษาไทย", @"nativeName": @"ภาษาไทย"},
        @{@"code": @"en", @"name": @"English", @"nativeName": @"English"},
        @{@"code": @"ja", @"name": @"Japanese", @"nativeName": @"日本語"},
        @{@"code": @"zh", @"name": @"Chinese", @"nativeName": @"中文"},
        @{@"code": @"ar", @"name": @"Arabic", @"nativeName": @"العربية"},
    ];
    
    [self.tableView registerClass:[UITableViewCell class] forCellReuseIdentifier:@"LanguageCell"];
}

- (NSInteger)tableView:(UITableView *)tableView numberOfRowsInSection:(NSInteger)section {
    return self.languages.count;
}

- (UITableViewCell *)tableView:(UITableView *)tableView cellForRowAtIndexPath:(NSIndexPath *)indexPath {
    UITableViewCell *cell = [tableView dequeueReusableCellWithIdentifier:@"LanguageCell" 
                                                             forIndexPath:indexPath];
    
    NSDictionary *language = self.languages[indexPath.row];
    NSString *currentLanguage = [[LocalizationManager sharedManager] currentLanguage];
    
    // แสดงชื่อภาษาในภาษาปัจจุบัน + ชื่อภาษาในภาษานั้นเอง
    NSString *langCode = language[@"code"];
    NSLocale *langLocale = [NSLocale localeWithLocaleIdentifier:langCode];
    NSString *displayName = [langLocale displayNameForKey:NSLocaleLanguageCode value:langCode];
    
    cell.textLabel.text = displayName ?: language[@"nativeName"];
    cell.detailTextLabel.text = language[@"nativeName"];
    
    // Checkmark สำหรับภาษาที่เลือก
    cell.accessoryType = [langCode isEqualToString:currentLanguage] ? 
                         UITableViewCellAccessoryCheckmark : UITableViewCellAccessoryNone;
    
    return cell;
}

- (void)tableView:(UITableView *)tableView didSelectRowAtIndexPath:(NSIndexPath *)indexPath {
    [tableView deselectRowAtIndexPath:indexPath animated:YES];
    
    NSDictionary *selectedLang = self.languages[indexPath.row];
    NSString *langCode = selectedLang[@"code"];
    
    // เปลี่ยนภาษา
    [[LocalizationManager sharedManager] setLanguage:langCode];
    
    // แจ้งผู้ใช้ว่าต้อง Restart
    UIAlertController *alert = [UIAlertController 
        alertControllerWithTitle:NSLocalizedString(@"LANGUAGE_CHANGED_TITLE", @"")
        message:NSLocalizedString(@"LANGUAGE_CHANGED_MESSAGE", @"ต้อง Restart แอป")
        preferredStyle:UIAlertControllerStyleAlert];
    
    [alert addAction:[UIAlertAction actionWithTitle:NSLocalizedString(@"OK", @"") 
                                             style:UIAlertActionStyleDefault 
                                           handler:^(UIAlertAction *action) {
        // Reload UI หรือ Restart
        [self.tableView reloadData];
    }]];
    
    [self presentViewController:alert animated:YES completion:nil];
}

@end
```

## แบบฝึกหัดที่ 2: Localized Product Listing

สร้าง Product List ที่แสดง:
- ราคาในสกุลเงินของ Locale ปัจจุบัน
- วันที่ตาม Format ของ Locale
- จำนวนรีวิวด้วย Plural Rules ที่ถูกต้อง

```objc
@interface Product : NSObject
@property (nonatomic, copy) NSString *nameKey;     // Localization Key
@property (nonatomic, strong) NSDecimalNumber *price;
@property (nonatomic, strong) NSDate *addedDate;
@property (nonatomic, assign) NSInteger reviewCount;
@end

@implementation Product
@end

@interface LocalizedProductCell : UITableViewCell

@property (nonatomic, strong) UILabel *nameLabel;
@property (nonatomic, strong) UILabel *priceLabel;
@property (nonatomic, strong) UILabel *dateLabel;
@property (nonatomic, strong) UILabel *reviewsLabel;

@end

@implementation LocalizedProductCell

- (void)configureWithProduct:(Product *)product {
    // ชื่อสินค้าใน Locale ปัจจุบัน
    self.nameLabel.text = NSLocalizedString(product.nameKey, @"ชื่อสินค้า");
    
    // ราคาในรูปแบบสกุลเงิน Local
    NSNumberFormatter *currencyFormatter = [[NSNumberFormatter alloc] init];
    currencyFormatter.numberStyle = NSNumberFormatterCurrencyStyle;
    currencyFormatter.locale = [NSLocale currentLocale];
    self.priceLabel.text = [currencyFormatter stringFromNumber:product.price];
    
    // วันที่ในรูปแบบ Local
    NSDateFormatter *dateFormatter = [[NSDateFormatter alloc] init];
    dateFormatter.dateStyle = NSDateFormatterMediumStyle;
    dateFormatter.timeStyle = NSDateFormatterNoStyle;
    dateFormatter.locale = [NSLocale currentLocale];
    dateFormatter.doesRelativeDateFormatting = YES;
    self.dateLabel.text = [dateFormatter stringFromDate:product.addedDate];
    
    // จำนวนรีวิว พร้อม Plural Rules
    self.reviewsLabel.text = [NSString localizedStringWithFormat:
                              NSLocalizedString(@"REVIEWS_COUNT", @""),
                              product.reviewCount];
}

@end
```

## แบบฝึกหัดที่ 3: Localization Testing Helper

```objc
// LocalizationTestHelper.m
@interface LocalizationTestHelper : NSObject

+ (void)testAllLocalizations;
+ (void)findMissingLocalizations;
+ (void)findExtraLongStrings;

@end

@implementation LocalizationTestHelper

+ (void)testAllLocalizations {
    NSArray *languages = @[@"th", @"en", @"ja"];
    
    // อ่าน Keys จาก Base (English)
    NSString *enPath = [[NSBundle mainBundle] pathForResource:@"en" ofType:@"lproj"];
    NSBundle *enBundle = [NSBundle bundleWithPath:enPath];
    
    NSDictionary *enStrings = [NSDictionary dictionaryWithContentsOfFile:
                               [enPath stringByAppendingPathComponent:@"Localizable.strings"]];
    
    for (NSString *lang in languages) {
        NSString *langPath = [[NSBundle mainBundle] pathForResource:lang ofType:@"lproj"];
        if (!langPath) {
            NSLog(@"❌ ไม่พบ Bundle สำหรับ: %@", lang);
            continue;
        }
        
        NSDictionary *langStrings = [NSDictionary dictionaryWithContentsOfFile:
                                    [langPath stringByAppendingPathComponent:@"Localizable.strings"]];
        
        NSInteger missing = 0;
        for (NSString *key in enStrings) {
            if (!langStrings[key]) {
                NSLog(@"⚠️ [%@] Missing Key: %@", lang, key);
                missing++;
            }
        }
        
        NSLog(@"[%@] Missing: %ld/%lu", lang, (long)missing, (unsigned long)enStrings.count);
    }
}

+ (void)findMissingLocalizations {
    NSLog(@"ตรวจสอบ Localization Keys ที่หายไป...");
    [self testAllLocalizations];
}

+ (void)findExtraLongStrings {
    // หาข้อความที่ยาวเกิน ซึ่งอาจทำให้ UI Overflow
    NSString *enPath = [[NSBundle mainBundle] pathForResource:@"en" ofType:@"lproj"];
    NSDictionary *enStrings = [NSDictionary dictionaryWithContentsOfFile:
                               [enPath stringByAppendingPathComponent:@"Localizable.strings"]];
    
    for (NSString *key in enStrings) {
        NSString *enValue = enStrings[key];
        
        NSArray *otherLanguages = @[@"th", @"ja", @"de"];
        for (NSString *lang in otherLanguages) {
            NSString *langPath = [[NSBundle mainBundle] pathForResource:lang ofType:@"lproj"];
            NSDictionary *langStrings = [NSDictionary dictionaryWithContentsOfFile:
                                        [langPath stringByAppendingPathComponent:@"Localizable.strings"]];
            
            NSString *langValue = langStrings[key];
            if (!langValue) continue;
            
            CGFloat ratio = (CGFloat)langValue.length / enValue.length;
            if (ratio > 2.0) {
                NSLog(@"⚠️ [%@] Key '%@' ยาวกว่า EN %.1fx: '%@'", 
                      lang, key, ratio, langValue);
            }
        }
    }
}

@end
```

---

## สรุปบทที่ 79

| หัวข้อ | สิ่งสำคัญที่ต้องจำ |
|--------|-------------------|
| NSLocalizedString | ใส่ Comment ทุกครั้ง เพื่อช่วย Translator |
| genstrings | รัน ทุกครั้งที่เพิ่ม Key ใหม่ |
| NSNumberFormatter | Cache Instance เพราะ init มีค่าใช้จ่ายสูง |
| NSDateFormatter | ใช้ en_US_POSIX สำหรับ ISO 8601 |
| RTL Support | ใช้ Leading/Trailing ไม่ใช้ Left/Right |
| .stringsdict | ใช้สำหรับ Plural Rules ในทุกภาษา |
| Auto Layout | numberOfLines = 0 และ Dynamic Type |
| Testing | ทดสอบด้วย Pseudolanguage ใน Xcode |

```bash
# Xcode Pseudolanguage Testing
# Scheme > Run > Options > App Language:
# - Double-Length Pseudolanguage (ทดสอบ Truncation)
# - Right-to-Left Pseudolanguage (ทดสอบ RTL)
# - Accented Pseudolanguage (ทดสอบ Special Characters)
```

> **คำแนะนำ**: เริ่มต้นด้วย Internationalization ตั้งแต่ต้น อย่ารอแปลทีหลัง การ Refactor ทีหลังมีค่าใช้จ่ายสูงมาก และเสี่ยงต่อการ Break UI
