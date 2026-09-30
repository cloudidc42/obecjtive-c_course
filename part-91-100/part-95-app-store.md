# ส่วนที่ 95: App Store Submission สำหรับแอพ Objective-C

## บทนำ

การนำแอพ iOS ขึ้น App Store เป็นขั้นตอนสำคัญที่นักพัฒนาทุกคนต้องเรียนรู้ กระบวนการนี้ครอบคลุมตั้งแต่การสมัคร Apple Developer Program ไปจนถึงการผ่าน App Review และการจัดการแอพหลังจาก publish แล้ว

---

## 95.1 Apple Developer Program

### ประเภทของ Developer Account

**Individual Developer**
- ค่าสมัคร: $99 USD/ปี
- เหมาะสำหรับนักพัฒนาคนเดียวหรือ freelancer
- แอพจะแสดงชื่อบุคคลใน App Store

**Organization/Company**
- ค่าสมัคร: $99 USD/ปี
- ต้องมี D-U-N-S Number (ขอฟรีจาก Dun & Bradstreet)
- แอพจะแสดงชื่อบริษัทใน App Store
- สามารถเพิ่ม team members ได้

**Enterprise**
- ค่าสมัคร: $299 USD/ปี
- สำหรับแจกจ่ายแอพภายในองค์กรเท่านั้น
- ห้ามใช้สำหรับ public distribution

### การสมัคร Apple Developer Program

```
1. ไปที่ developer.apple.com/programs/
2. คลิก "Enroll"
3. Sign in ด้วย Apple ID (ควรเป็น ID สำหรับ business โดยเฉพาะ)
4. เลือกประเภท: Individual หรือ Organization
5. กรอกข้อมูล:
   - Legal name
   - Address
   - Phone number
   - D-U-N-S Number (สำหรับ Organization)
6. ชำระเงิน $99 USD
7. รอ approval (ปกติ 24-48 ชั่วโมง, อาจนานกว่านั้นสำหรับ Organization)
```

---

## 95.2 Certificates and Provisioning Profiles

### ประเภทของ Certificates

**Development Certificate**
- ใช้สำหรับ build และทดสอบบนอุปกรณ์
- สร้างได้สูงสุด 2 ใบต่อ developer

**Distribution Certificate**
- ใช้สำหรับ upload ขึ้น App Store
- ใช้ร่วมกันในทีมได้

**Push Notification Certificate (APNs)**
- ใช้สำหรับส่ง push notifications
- ต้องสร้างสำหรับแต่ละ app

### การสร้าง Certificate ผ่าน Xcode

```
1. Xcode → Preferences → Accounts
2. เพิ่ม Apple ID
3. เลือก Team
4. คลิก "Manage Certificates"
5. คลิก + เพื่อสร้าง Apple Distribution Certificate
```

### การสร้าง Certificate ผ่าน Developer Portal

```
1. developer.apple.com → Certificates, IDs & Profiles
2. Certificates → +
3. เลือกประเภท Certificate
4. สร้าง Certificate Signing Request (CSR):
   - เปิด Keychain Access
   - Menu: Keychain Access → Certificate Assistant → Request a Certificate From a Certificate Authority
   - กรอก email และชื่อ
   - เลือก "Saved to disk"
5. Upload CSR file
6. Download .cer file
7. Double-click เพื่อ install ใน Keychain
```

### Provisioning Profiles

**Development Profile**
- ผูก App ID + Development Certificate + Device UDIDs
- ใช้สำหรับ test บนอุปกรณ์จริง

**Ad Hoc Profile**
- ผูก App ID + Distribution Certificate + Device UDIDs (สูงสุด 100 เครื่อง)
- ใช้สำหรับ beta testing นอก TestFlight

**App Store Profile**
- ผูก App ID + Distribution Certificate
- ใช้สำหรับ upload ขึ้น App Store

**Enterprise Profile**
- ใช้สำหรับ in-house distribution

```objc
// ตรวจสอบ provisioning profile ใน code
NSDictionary *embeddedProfile = /* อ่านจาก embedded.mobileprovision */;
NSLog(@"Profile name: %@", embeddedProfile[@"Name"]);
NSLog(@"Profile type: %@", embeddedProfile[@"ProvisionsAllDevices"] ? @"Enterprise" : @"Standard");
```

### App ID

```
App ID Format: TEAMID.bundleidentifier
ตัวอย่าง: ABC123.com.yourcompany.yourapp

Explicit App ID:
- ใช้สำหรับ app เฉพาะ
- ตรงกับ Bundle Identifier ใน Xcode

Wildcard App ID:
- ABC123.com.yourcompany.*
- ใช้กับ app หลายตัวในองค์กร
- ไม่รองรับ Push Notifications, In-App Purchase บางประเภท
```

---

## 95.3 App Store Connect Setup

### การสร้าง App ใหม่ใน App Store Connect

```
1. ไปที่ appstoreconnect.apple.com
2. Apps → + (New App)
3. กรอกข้อมูล:
   - Platform: iOS
   - Name: ชื่อแอพ (สูงสุด 30 ตัวอักษร)
   - Primary Language
   - Bundle ID (ต้องตรงกับ Xcode)
   - SKU: unique identifier (ใช้ภายในเท่านั้น)
   - User Access: Full Access หรือ Limited Access
```

### App Store Connect Roles

```
Account Holder: สิทธิ์สูงสุด ทำได้ทุกอย่าง
Admin: จัดการทุกอย่างยกเว้น legal/financial
App Manager: จัดการ app ได้
Developer: สร้าง build, manage TestFlight
Marketer: ดู analytics, ไม่สามารถ submit
Finance: ดูข้อมูล financial เท่านั้น
Customer Support: จัดการ customer reviews
```

---

## 95.4 App Metadata

### Screenshots

**ขนาดที่ต้องการ (2024):**

```
iPhone 6.7" (iPhone 14 Pro Max):
- Portrait: 1290 x 2796 px
- Landscape: 2796 x 1290 px

iPhone 6.5" (iPhone 14 Plus):
- Portrait: 1284 x 2778 px

iPhone 5.5" (iPhone 8 Plus) - Required if supporting iOS 12:
- Portrait: 1242 x 2208 px

iPad Pro 12.9" (6th gen):
- Portrait: 2048 x 2732 px

iPad Pro 11" (4th gen):
- Portrait: 1668 x 2388 px
```

**Best Practices สำหรับ Screenshots:**
```
1. แสดง key features ของแอพ
2. ใช้ภาษาของ target market
3. เพิ่ม caption/text overlay ที่อธิบาย features
4. ใช้ device frame เพื่อให้ดูเป็นธรรมชาติ
5. Screenshot แรกสำคัญที่สุด - แสดง core value
6. อย่าใช้รูปจาก device จริงที่มีข้อมูลส่วนตัว
```

### App Preview Video

```
- ความยาว: 15-30 วินาที
- Format: .mov, .m4v, .mp4
- Resolution: ตรงกับ screenshot size
- ไม่ควรมี Apple logo, Apple device trademarks
- แสดง actual app usage
- ไม่ควรมี pricing information ที่อาจเปลี่ยนแปลง
```

### App Description

```markdown
# ตัวอย่าง App Description (ภาษาไทย)

[ส่วนแรก - สำคัญมาก เพราะแสดงก่อน "More"]
MyApp - แอพจัดการงานที่ดีที่สุดสำหรับทีมของคุณ
ออกแบบมาเพื่อให้ทีมทำงานร่วมกันได้อย่างมีประสิทธิภาพ

✨ คุณสมบัติหลัก:
• จัดการ task และ project ได้ง่ายดาย
• แชร์ไฟล์และ collaborate แบบ real-time
• แจ้งเตือนอัจฉริยะที่ไม่รบกวน
• รองรับ offline mode

[ส่วนต่อมา - รายละเอียดเพิ่มเติม]
WHY MYAPP?
MyApp ถูกสร้างมาเพื่อแก้ปัญหา...

FEATURES:
- Project Management: สร้างและจัดการโปรเจกต์
- Task Tracking: ติดตามงานแบบ real-time
- Team Chat: สื่อสารในทีมได้ทันที
- File Sharing: แชร์ไฟล์ได้ถึง 1GB
- Calendar Integration: ซิงค์กับ Calendar อัตโนมัติ

SUBSCRIPTION:
- Free: ใช้งานได้ 3 projects, 5 members
- Pro ($9.99/เดือน): Unlimited projects และ members
- Business ($29.99/เดือน): Advanced analytics, Priority support

SUPPORT:
support@myapp.com | myapp.com/support
```

### Keywords

```
- ใส่ได้สูงสุด 100 ตัวอักษร
- แยกด้วย comma
- ไม่ต้องใส่ app name (ระบบเพิ่มให้อัตโนมัติ)
- ไม่ต้องใส่ชื่อคู่แข่ง
- ใช้ keyword ที่ผู้ใช้จะค้นหา

ตัวอย่าง:
task,project,team,collaborate,productivity,todo,work,manage,planner,organizer
```

### What's New (Release Notes)

```markdown
เวอร์ชัน 2.1.0:
✨ ฟีเจอร์ใหม่:
• เพิ่ม Dark Mode รองรับ iOS 16
• เพิ่ม Widget สำหรับ Home Screen
• เพิ่ม Siri Shortcuts

🐛 แก้ไขบั๊ก:
• แก้ปัญหา crash เมื่อ sync กับ iCloud
• แก้ปัญหา notification ไม่แสดงผล

⚡ ปรับปรุงประสิทธิภาพ:
• โหลดเร็วขึ้น 30%
• ลดการใช้ battery
```

---

## 95.5 App Review Guidelines

### หมวดหมู่สำคัญที่ต้องทราบ

**Safety**
```
- ห้ามมี content ที่ promote ความรุนแรง
- ห้ามมี cyberbullying features
- ต้องมีวิธีให้ user report inappropriate content
- แอพสำหรับเด็กต้องปฏิบัติตาม COPPA
```

**Performance**
```
- แอพต้องไม่ crash
- ต้องทำงานตามที่อธิบายไว้
- ต้องโหลดและทำงานได้บน supported devices
- ห้ามมี placeholder content
```

**Business**
```
- In-App Purchase ต้องใช้ Apple's payment system
- ห้าม direct link ไปยัง external payment outside sandbox
- Subscription pricing ต้องชัดเจน
- ห้าม incentivize ratings
```

**Design**
```
- ต้องออกแบบตาม iOS HIG (Human Interface Guidelines)
- ต้องรองรับ Dynamic Type
- ต้องรองรับ both portrait และ landscape (iPad)
- ไม่ควรใช้ Apple icons ในทางที่ผิด
```

**Legal**
```
- ต้องมี Privacy Policy
- ห้าม collect data โดยไม่แจ้ง user
- ต้องปฏิบัติตาม GDPR สำหรับ EU users
- Intellectual property ต้องถูกต้อง
```

---

## 95.6 Age Rating

### การกำหนด Age Rating

```
4+   : ไม่มี content ที่ไม่เหมาะสม
9+   : มี content เล็กน้อยที่ไม่เหมาะกับเด็กเล็ก
12+  : มี content สำหรับวัยรุ่น
17+  : มี content สำหรับผู้ใหญ่ (ต้องยืนยันอายุ)

Categories ที่กระทบ Rating:
- Cartoon or Fantasy Violence
- Realistic Violence
- Sexual Content or Nudity
- Profanity or Crude Humor
- Mature/Suggestive Themes
- Horror/Fear Themes
- Medical/Treatment Information
- Alcohol, Tobacco, or Drug Use or References
- Simulated Gambling
- Unrestricted Web Access
```

```objc
// ตัวอย่างการ check age restriction ในแอพ
- (void)checkAgeRestriction {
    // ถ้าแอพมี content สำหรับผู้ใหญ่
    NSUserDefaults *defaults = [NSUserDefaults standardUserDefaults];
    BOOL hasVerifiedAge = [defaults boolForKey:@"hasVerifiedAge"];
    
    if (!hasVerifiedAge) {
        [self presentAgeVerificationAlert];
    }
}

- (void)presentAgeVerificationAlert {
    UIAlertController *alert = [UIAlertController 
        alertControllerWithTitle:@"ยืนยันอายุ"
        message:@"คุณต้องมีอายุ 18 ปีขึ้นไปเพื่อใช้งานฟีเจอร์นี้"
        preferredStyle:UIAlertControllerStyleAlert];
    
    [alert addAction:[UIAlertAction actionWithTitle:@"ฉันอายุ 18 ปีขึ้นไป"
                                             style:UIAlertActionStyleDefault
                                           handler:^(UIAlertAction *action) {
        [[NSUserDefaults standardUserDefaults] setBool:YES forKey:@"hasVerifiedAge"];
    }]];
    
    [alert addAction:[UIAlertAction actionWithTitle:@"ยกเลิก"
                                             style:UIAlertActionStyleCancel
                                           handler:nil]];
    
    [self presentViewController:alert animated:YES completion:nil];
}
```

---

## 95.7 Privacy Policy และ App Privacy

### Privacy Policy Requirements

Privacy Policy จำเป็นสำหรับทุกแอพที่:
- Collect user data
- ใช้ social login
- มี in-app purchase
- ใช้ analytics
- แสดง ads

**โครงสร้าง Privacy Policy พื้นฐาน:**

```
1. ข้อมูลที่เก็บรวบรวม
   - ข้อมูลที่ user ให้ (ชื่อ, email)
   - ข้อมูลที่เก็บอัตโนมัติ (device info, usage data)
   - Cookies และ tracking

2. วิธีใช้ข้อมูล
   - ให้บริการและปรับปรุงแอพ
   - ส่งการแจ้งเตือน
   - Analytics และ performance

3. การแบ่งปันข้อมูล
   - Third-party service providers
   - ไม่ขายข้อมูลให้บุคคลที่สาม

4. Data Security
   - วิธีการเข้ารหัสข้อมูล
   - SSL/TLS

5. User Rights
   - สิทธิ์ในการเข้าถึง ลบ แก้ไขข้อมูล
   - Opt-out options

6. Contact Information
   - วิธีติดต่อสำหรับ privacy concerns
```

### App Privacy Nutrition Labels

ใน App Store Connect ต้องกรอกข้อมูล Privacy ก่อน submit:

**Categories หลัก:**

```
Data Used to Track You:
- ข้อมูลที่ใช้ข้ามแอพเพื่อ track user
- เช่น Advertising ID

Data Linked to You:
- ข้อมูลที่ผูกกับ identity ของ user
- เช่น Email, Name, Location, Photos

Data Not Linked to You:
- ข้อมูลที่เก็บแต่ไม่ผูกกับ user identity
- เช่น Crash data, Device ID

No Data Collected:
- แอพไม่เก็บข้อมูลใดๆ
```

**ตัวอย่างการระบุ Privacy practices:**

```objc
// ตัวอย่าง code ที่ต้องระบุใน Privacy Labels

// 1. Contact Info - Email
NSString *userEmail = /* collected during sign up */;
// ต้องระบุ: "Email Address" - "Contact Info" - "Data Linked to You"

// 2. Location
CLLocationManager *locationManager = [[CLLocationManager alloc] init];
[locationManager requestWhenInUseAuthorization];
// ต้องระบุ: "Precise Location" หรือ "Coarse Location"

// 3. Photos
PHPhotoLibrary *photoLibrary = [PHPhotoLibrary sharedPhotoLibrary];
// ต้องระบุ: "Photos or Videos" ถ้า upload ไปยัง server

// 4. Analytics
// Firebase Analytics เก็บ:
// - Device ID (Data Not Linked to You)
// - Usage Data (Data Not Linked to You)
```

---

## 95.8 Building for Release

### การตั้งค่า Build Configuration

```objc
// ตรวจสอบ build configuration ใน code
#ifdef DEBUG
    #define DLog(fmt, ...) NSLog((@"[DEBUG] %s line %d: " fmt), __PRETTY_FUNCTION__, __LINE__, ##__VA_ARGS__)
#else
    #define DLog(...)
#endif

// ตั้งค่า different endpoints
static NSString * const APIBaseURL = 
#ifdef DEBUG
    @"https://dev.api.example.com";
#else
    @"https://api.example.com";
#endif
```

**Build Settings ที่ควรตรวจสอบก่อน Release:**

```
1. Bundle Version (Build Number): ต้องเพิ่มขึ้นทุก submission
   CFBundleVersion = "42"

2. Bundle Short Version String (App Version): 
   CFBundleShortVersionString = "2.1.0"

3. Deployment Target: iOS version ต่ำสุดที่รองรับ
   IPHONEOS_DEPLOYMENT_TARGET = "14.0"

4. Bitcode: ปิดสำหรับ Xcode 14+
   ENABLE_BITCODE = NO

5. Strip Debug Symbols: เปิดสำหรับ Release
   STRIP_INSTALLED_PRODUCT = YES

6. Code Signing: ใช้ Distribution certificate
   CODE_SIGN_IDENTITY = "Apple Distribution"
   PROVISIONING_PROFILE_SPECIFIER = "MyApp App Store"
```

### การสร้าง Archive

```
1. เลือก Target device: Any iOS Device (arm64)
   - ห้ามเลือก Simulator
   
2. Product → Archive
   - Xcode จะ build และสร้าง archive
   - ใช้เวลาสักครู่

3. Archive จะเปิดใน Organizer
   - เห็น archive ที่สร้าง
   - สามารถ validate และ distribute ได้

4. Validate App (ทำก่อน Upload เสมอ):
   - คลิก "Validate App"
   - ตรวจสอบ errors และ warnings
   - แก้ไขปัญหาก่อน upload

5. Distribute App:
   - คลิก "Distribute App"
   - เลือก "App Store Connect"
   - เลือก "Upload" หรือ "Export"
```

### Command Line Archive

```bash
# Clean build folder
xcodebuild clean \
    -project MyApp.xcodeproj \
    -scheme MyApp \
    -configuration Release

# Archive
xcodebuild archive \
    -project MyApp.xcodeproj \
    -scheme MyApp \
    -configuration Release \
    -archivePath ./build/MyApp.xcarchive \
    CODE_SIGN_IDENTITY="Apple Distribution" \
    PROVISIONING_PROFILE_SPECIFIER="MyApp App Store"

# Export IPA
xcodebuild -exportArchive \
    -archivePath ./build/MyApp.xcarchive \
    -exportPath ./build/ipa \
    -exportOptionsPlist ExportOptions.plist
```

```xml
<!-- ExportOptions.plist -->
<?xml version="1.0" encoding="UTF-8"?>
<!DOCTYPE plist PUBLIC "-//Apple//DTD PLIST 1.0//EN" "http://www.apple.com/DTDs/PropertyList-1.0.dtd">
<plist version="1.0">
<dict>
    <key>method</key>
    <string>app-store</string>
    <key>teamID</key>
    <string>YOURTEAMID</string>
    <key>uploadSymbols</key>
    <true/>
    <key>compileBitcode</key>
    <false/>
</dict>
</plist>
```

---

## 95.9 Uploading to App Store Connect

### การ Upload ด้วย Xcode Organizer

```
1. Product → Archive (ถ้ายังไม่มี archive)
2. Window → Organizer → Archives tab
3. เลือก archive ล่าสุด
4. คลิก "Distribute App"
5. เลือก "App Store Connect"
6. เลือก "Upload"
7. เลือก options:
   ✅ Include bitcode (optional, Xcode 14+ ปิด)
   ✅ Upload your app's symbols
8. Review
9. Upload
```

### การ Upload ด้วย Transporter App

```
1. Download Transporter จาก Mac App Store (ฟรี)
2. Sign in ด้วย Apple ID ที่เป็น Developer
3. ลาก .ipa file เข้าไป
4. คลิก Deliver
```

### การ Upload ด้วย altool (Command Line)

```bash
# Validate ก่อน upload
xcrun altool --validate-app \
    -f MyApp.ipa \
    -t ios \
    -u your@appleid.com \
    -p @keychain:"Application Loader: your@appleid.com"

# Upload
xcrun altool --upload-app \
    -f MyApp.ipa \
    -t ios \
    -u your@appleid.com \
    -p @keychain:"Application Loader: your@appleid.com"

# หรือใช้ App-specific password
xcrun altool --upload-app \
    -f MyApp.ipa \
    -t ios \
    -u your@appleid.com \
    -p "xxxx-xxxx-xxxx-xxxx"
```

---

## 95.10 TestFlight for Beta Testing

### Internal Testing

```
- นักทดสอบสูงสุด 100 คน
- เฉพาะ members ใน App Store Connect team
- ไม่ต้องผ่าน App Review
- Build หมดอายุ 90 วัน
```

### External Testing

```
- นักทดสอบสูงสุด 10,000 คน
- เชิญได้โดยใช้ email หรือ public link
- ต้องผ่าน Beta App Review ครั้งแรก
- Build หมดอายุ 90 วัน
```

### การตั้งค่า TestFlight

```
1. App Store Connect → TestFlight tab
2. เลือก build ที่ upload แล้ว
3. กรอก "What to Test" (บอกผู้ทดสอบว่าต้องทดสอบอะไร)
4. Internal Testing:
   - Add internal testers
   - Enable build สำหรับ testing
5. External Testing:
   - Create group
   - Add testers หรือสร้าง public link
   - Submit for Beta Review
```

```objc
// การตรวจสอบว่ากำลังรันจาก TestFlight
+ (BOOL)isRunningFromTestFlight {
    NSURL *receiptURL = [[NSBundle mainBundle] appStoreReceiptURL];
    NSString *receiptPath = receiptURL.path;
    return [receiptPath containsString:@"sandboxReceipt"];
}

// การใช้งาน
if ([AppUtils isRunningFromTestFlight]) {
    // แสดง beta features
    [self enableBetaFeatures];
}
```

### TestFlight Feedback

```objc
// TestFlight ให้ผู้ทดสอบ capture screenshot และส่ง feedback ได้อัตโนมัติ
// แต่คุณสามารถเพิ่ม custom feedback mechanism ได้

- (IBAction)sendFeedbackTapped:(id)sender {
    if ([MFMailComposeViewController canSendMail]) {
        MFMailComposeViewController *mail = [[MFMailComposeViewController alloc] init];
        mail.mailComposeDelegate = self;
        [mail setToRecipients:@[@"feedback@yourapp.com"]];
        [mail setSubject:@"MyApp Beta Feedback"];
        
        // เพิ่ม device info
        NSString *body = [NSString stringWithFormat:@"\n\n---\nDevice: %@\niOS: %@\nApp: %@",
                         [[UIDevice currentDevice] model],
                         [[UIDevice currentDevice] systemVersion],
                         [[NSBundle mainBundle] objectForInfoDictionaryKey:@"CFBundleShortVersionString"]];
        [mail setMessageBody:body isHTML:NO];
        
        [self presentViewController:mail animated:YES completion:nil];
    }
}
```

---

## 95.11 App Review Process

### ขั้นตอน App Review

```
1. Prepare for Submission:
   - ตั้งค่า pricing, availability
   - กรอก metadata ให้ครบ
   - เพิ่ม screenshots
   - กำหนด ratings
   - กรอก App Privacy info
   - เพิ่ม review notes (ถ้าจำเป็น)

2. Submit for Review:
   - เลือก build จาก TestFlight/upload
   - คลิก "Submit for Review"
   - ตอบคำถาม:
     * Uses advertising identifier?
     * Content rights?
     * Export compliance?

3. Review Process:
   - In Review: กำลังถูก review (ปกติ 24-48 ชั่วโมง)
   - Approved: ผ่าน review แล้ว
   - Rejected: ไม่ผ่าน (จะมี rejection reason)
   - Pending Release: รอ release ตามวันที่กำหนด

4. Timeline:
   - ปกติ: 1-2 วันทำการ
   - เร่งด่วน (Expedite Review): ขอได้ในกรณีฉุกเฉิน
   - วันหยุดนักขัตฤกษ์: อาจนานขึ้น
```

### Demo Account / Instructions for Review

```objc
// ถ้าแอพต้องการ login เพื่อใช้งาน ต้องให้ demo account แก่ reviewers
/*
Review Notes ตัวอย่าง:

Demo Account:
Username: reviewer@example.com
Password: TestPassword123!

Steps to test main feature:
1. Login with demo account above
2. Tap "Create New Project"
3. Enter project name and tap "Create"
4. Add tasks by tapping the + button
5. Swipe left on a task to see delete option

Note: The app requires camera permission to scan QR codes.
To test: tap "Scan QR" button and use the camera to scan 
the QR code provided in the image below.
*/
```

---

## 95.12 Responding to Rejections

### ประเภทของ Rejection

**Guideline 2.1 - Performance: App Completeness**
```
ปัญหา: แอพมีฟีเจอร์ที่ยังไม่เสร็จ, placeholder content, crash
วิธีแก้: แก้บั๊ก, เติม content, ทดสอบให้ครบ
```

**Guideline 3.1.1 - Business: Payments - In-App Purchase**
```
ปัญหา: ใช้ payment method อื่นที่ไม่ใช่ Apple In-App Purchase
วิธีแก้: ใช้ StoreKit, ห้าม link ไป external payment
```

**Guideline 4.0 - Design: Copycats**
```
ปัญหา: แอพ copy app อื่นอย่างชัดเจน
วิธีแก้: ต้องมี unique value proposition
```

**Guideline 5.1.1 - Privacy: Data Collection and Storage**
```
ปัญหา: ไม่ได้ขอ permission ก่อน collect data, Privacy Policy ไม่ครบ
วิธีแก้: เพิ่ม permission request, อัพเดต Privacy Policy
```

### วิธี Appeal Rejection

```
1. อ่าน rejection reason อย่างละเอียด
2. หา guideline ที่เกี่ยวข้องและอ่านให้ครบ
3. ถ้าเข้าใจปัญหา:
   - แก้ไขและ resubmit
   - ใส่ reply อธิบายว่าแก้ไขอะไรแล้ว
   
4. ถ้าไม่เห็นด้วยกับ rejection:
   - คลิก "Reply" ใน Resolution Center
   - อธิบายอย่างสุภาพว่าทำไมคิดว่าแอพสอดคล้องกับ guideline
   - ให้หลักฐาน (screenshots, video)
   
5. ถ้ายังไม่ได้รับการแก้ไข:
   - ขอ appeal ผ่าน App Review Board
   - developer.apple.com/contact/app-store/?topic=appeal
```

```objc
// ตัวอย่าง reply ที่ดี:
/*
Thank you for reviewing MyApp.

Regarding guideline 5.1.1:

We believe our app is in compliance because:
1. We request location permission with clear explanation of usage
2. Location data is only used to show nearby services and is not stored
3. Our Privacy Policy (https://myapp.com/privacy) clearly states 
   how we handle location data

Screenshot attached showing our permission request with explanation.

If there's specific behavior that violated the guideline, 
please let us know so we can address it immediately.

Best regards,
Developer Name
*/
```

---

## 95.13 App Store Optimization (ASO)

### ปัจจัยที่กระทบ Ranking

**On-Metadata Factors (คุณควบคุมได้):**
```
1. Title (30 ตัวอักษร)
   - ใส่ keyword สำคัญที่สุด
   - ตัวอย่าง: "TodoMaster - Task Manager" ดีกว่า "TodoMaster"

2. Subtitle (30 ตัวอักษร)
   - ใส่ keywords เพิ่มเติม
   - ตัวอย่าง: "Organize Projects & Goals"

3. Keywords (100 ตัวอักษร)
   - อย่าซ้ำกับ Title/Subtitle
   - แยกด้วย comma
   - ไม่ต้องเว้นวรรค

4. Description (4000 ตัวอักษร)
   - Keyword density
   - Localization
```

**Off-Metadata Factors (ควบคุมได้บางส่วน):**
```
1. Ratings & Reviews
   - ขอ review ในเวลาที่เหมาะสม
   - ตอบ review ทุกอัน

2. Downloads & Engagement
   - Conversion rate
   - Retention rate

3. Updates
   - Update บ่อยๆ แสดงว่ายังมีการพัฒนา
```

### การขอ Review ใน App

```objc
// ใช้ SKStoreReviewController อย่างถูกต้อง
#import <StoreKit/StoreKit.h>

- (void)requestReviewIfAppropriate {
    // ขอ review หลังจาก user ทำ action สำเร็จ 3 ครั้ง
    NSInteger completedActions = [[NSUserDefaults standardUserDefaults] 
                                   integerForKey:@"completedActionsCount"];
    
    if (completedActions >= 3) {
        // Apple สามารถ show หรือไม่ show ก็ได้ตาม algorithm ของ Apple
        if (@available(iOS 14.0, *)) {
            if (let scene = UIApplication.sharedApplication.connectedScenes.firstObject as? UIWindowScene {
                [SKStoreReviewController requestReviewInScene:scene];
            }
        } else {
            [SKStoreReviewController requestReview];
        }
        
        // Reset counter
        [[NSUserDefaults standardUserDefaults] setInteger:0 forKey:@"completedActionsCount"];
    }
}

// เรียกเมื่อ user ทำ action สำเร็จ
- (void)userCompletedImportantAction {
    NSInteger count = [[NSUserDefaults standardUserDefaults] 
                        integerForKey:@"completedActionsCount"];
    [[NSUserDefaults standardUserDefaults] setInteger:count + 1 
                                               forKey:@"completedActionsCount"];
    [self requestReviewIfAppropriate];
}
```

### ตอบ Reviews

```
Tips สำหรับการตอบ Reviews:
1. ตอบทุก review ทั้งดีและไม่ดี
2. ขอบคุณสำหรับ positive reviews
3. สำหรับ negative reviews:
   - ขอโทษสำหรับประสบการณ์ที่ไม่ดี
   - อธิบายว่ากำลังแก้ไข
   - ให้ contact info เพื่อช่วยเพิ่มเติม
4. อย่าเถียงหรือ defensive
5. ใช้ภาษาสุภาพเสมอ

ตัวอย่าง:
"ขอบคุณสำหรับ feedback ครับ เราได้รับทราบปัญหาที่คุณพบแล้ว 
และกำลังแก้ไขใน version ถัดไปที่จะออกเร็วๆ นี้ 
ถ้ามีปัญหาเพิ่มเติม ติดต่อเราได้ที่ support@myapp.com ครับ"
```

---

## 95.14 Practice Checklist

### Pre-Submission Checklist

**Technical:**
```
□ ทดสอบบน device จริง (ไม่ใช่แค่ Simulator)
□ ทดสอบบน iOS versions ทั้งหมดที่ support
□ ทดสอบบน device sizes ต่างๆ (iPhone SE, iPhone 14 Pro Max)
□ ทดสอบ network conditions (offline, slow network)
□ ทดสอบ memory warnings
□ ไม่มี crashes ใน Simulator และ device
□ Clang Analyzer ไม่พบ issues
□ Bundle ID ถูกต้อง
□ Version number เพิ่มขึ้น (Build number ต้องสูงกว่า build ก่อน)
□ Signing certificate ถูกต้อง (Distribution, ไม่ใช่ Development)
□ Provisioning profile เป็น App Store type
□ Info.plist มีข้อมูลครบ
□ Icon ครบทุกขนาด
□ Launch screen ทำงานถูกต้อง
```

**Content:**
```
□ App Name ไม่เกิน 30 ตัวอักษร
□ Screenshots ครบทุก size (6.7", 5.5" required)
□ App Preview video (optional but recommended)
□ Description เขียนดี (ไม่มีการพิมพ์ผิด)
□ Keywords ครบ 100 ตัวอักษร
□ What's New อัพเดตแล้ว
□ Rating ถูกต้อง
□ Privacy Policy URL ใช้งานได้
□ Support URL ใช้งานได้
□ App Privacy Nutrition Labels กรอกครบ
□ Review Notes (Demo account ถ้าจำเป็น)
□ Content Rights ยืนยันแล้ว
□ Export Compliance ตอบแล้ว
□ Advertising Identifier ตอบแล้ว
```

**TestFlight:**
```
□ Build ผ่าน Internal Testing แล้ว
□ ได้รับ feedback จาก beta testers
□ แก้ไข issues ที่พบใน beta แล้ว
```

---

## สรุป

ในส่วนนี้เราได้เรียนรู้:
- การสมัครและใช้งาน Apple Developer Program
- การสร้าง Certificates และ Provisioning Profiles
- การตั้งค่า App Store Connect
- การเตรียม App Metadata ที่ดี
- App Review Guidelines ที่สำคัญ
- Privacy Policy และ App Privacy Nutrition Labels
- การ Build และ Archive สำหรับ Release
- การ Upload ด้วย Xcode, Transporter และ Command Line
- TestFlight สำหรับ Beta Testing
- วิธีรับมือกับ Rejection
- App Store Optimization (ASO)

---

*ส่วนถัดไป: Part 96 - Enterprise iOS Development*
