# ส่วนที่ 94: Open Source Contribution ใน Objective-C

## บทนำ

ชุมชน Open Source เป็นหัวใจของการพัฒนา iOS ecosystem ไลบรารีชื่อดังอย่าง AFNetworking, SDWebImage, และ Masonry ล้วนเกิดจากนักพัฒนาที่แบ่งปันโค้ดของตัวเอง การมีส่วนร่วมใน Open Source ไม่เพียงช่วยให้ชุมชน แต่ยังช่วยพัฒนาทักษะและ portfolio ของคุณด้วย

---

## 94.1 ไลบรารี Objective-C ยอดนิยม

### AFNetworking

AFNetworking เป็นไลบรารี networking ที่ได้รับความนิยมสูงที่สุดใน iOS

```objc
// ติดตั้งผ่าน CocoaPods
// pod 'AFNetworking', '~> 4.0'

#import <AFNetworking/AFNetworking.h>

// GET Request
AFHTTPSessionManager *manager = [AFHTTPSessionManager manager];
[manager GET:@"https://api.example.com/users" 
  parameters:nil 
    headers:nil 
    progress:nil 
     success:^(NSURLSessionDataTask *task, id responseObject) {
         NSLog(@"JSON: %@", responseObject);
     } 
     failure:^(NSURLSessionDataTask *task, NSError *error) {
         NSLog(@"Error: %@", error);
     }];

// POST Request
NSDictionary *parameters = @{@"username": @"john", @"password": @"secret"};
[manager POST:@"https://api.example.com/login"
   parameters:parameters
     headers:nil
    progress:nil
     success:^(NSURLSessionDataTask *task, id responseObject) {
         NSLog(@"Login successful: %@", responseObject);
     }
     failure:^(NSURLSessionDataTask *task, NSError *error) {
         NSLog(@"Login failed: %@", error);
     }];

// การใช้งาน AFNetworkReachabilityManager
[[AFNetworkReachabilityManager sharedManager] setReachabilityStatusChangeBlock:^(AFNetworkReachabilityStatus status) {
    NSLog(@"Reachability: %@", AFStringFromNetworkReachabilityStatus(status));
}];
[[AFNetworkReachabilityManager sharedManager] startMonitoring];

// การ Upload File
AFHTTPSessionManager *manager2 = [AFHTTPSessionManager manager];
[manager2 POST:@"https://api.example.com/upload"
     parameters:nil
      headers:nil
constructingBodyWithBlock:^(id<AFMultipartFormData> formData) {
    [formData appendPartWithFileData:imageData
                                name:@"photo"
                            fileName:@"photo.jpg"
                            mimeType:@"image/jpeg"];
}
       progress:^(NSProgress *uploadProgress) {
           NSLog(@"Progress: %@", uploadProgress.localizedDescription);
       }
        success:^(NSURLSessionDataTask *task, id responseObject) {
            NSLog(@"Upload successful");
        }
        failure:^(NSURLSessionDataTask *task, NSError *error) {
            NSLog(@"Upload failed: %@", error);
        }];
```

### SDWebImage

ไลบรารีสำหรับโหลดและ cache รูปภาพจาก URL

```objc
// ติดตั้งผ่าน CocoaPods
// pod 'SDWebImage', '~> 5.0'

#import <SDWebImage/SDWebImage.h>

// โหลดรูปภาพง่ายๆ
NSURL *imageURL = [NSURL URLWithString:@"https://example.com/image.jpg"];
[self.imageView sd_setImageWithURL:imageURL];

// โหลดพร้อม placeholder และ completion
[self.imageView sd_setImageWithURL:imageURL
                  placeholderImage:[UIImage imageNamed:@"placeholder"]
                         completed:^(UIImage *image, NSError *error, SDImageCacheType cacheType, NSURL *imageURL) {
    if (error) {
        NSLog(@"Error loading image: %@", error);
    } else {
        NSLog(@"Image loaded from cache: %@", (cacheType != SDImageCacheTypeNone) ? @"YES" : @"NO");
    }
}];

// โหลดพร้อม options
[self.imageView sd_setImageWithURL:imageURL
                  placeholderImage:[UIImage imageNamed:@"placeholder"]
                           options:SDWebImageRefreshCached | SDWebImageProgressiveLoad
                          progress:^(NSInteger receivedSize, NSInteger expectedSize, NSURL *targetURL) {
    CGFloat progress = (CGFloat)receivedSize / expectedSize;
    NSLog(@"Progress: %.0f%%", progress * 100);
}
                         completed:nil];

// การจัดการ Cache
// ล้าง memory cache
[[SDImageCache sharedImageCache] clearMemory];

// ล้าง disk cache
[[SDImageCache sharedImageCache] clearDiskOnCompletion:^{
    NSLog(@"Disk cache cleared");
}];

// ตรวจสอบขนาด cache
NSUInteger diskSize = [[SDImageCache sharedImageCache] totalDiskSize];
NSLog(@"Cache size: %lu bytes", diskSize);

// Prefetch images
NSArray *urls = @[[NSURL URLWithString:@"https://example.com/1.jpg"],
                  [NSURL URLWithString:@"https://example.com/2.jpg"]];
[[SDWebImagePrefetcher sharedImagePrefetcher] prefetchURLs:urls];
```

### MBProgressHUD

ไลบรารีสำหรับแสดง loading indicator และ progress

```objc
// ติดตั้งผ่าน CocoaPods
// pod 'MBProgressHUD', '~> 1.2'

#import <MBProgressHUD/MBProgressHUD.h>

// แสดง Loading
MBProgressHUD *hud = [MBProgressHUD showHUDAddedTo:self.view animated:YES];
hud.label.text = @"กำลังโหลด...";

// ซ่อน Loading
[MBProgressHUD hideHUDForView:self.view animated:YES];

// แสดงพร้อม completion
[MBProgressHUD showHUDAddedTo:self.view animated:YES];

dispatch_async(dispatch_get_global_queue(DISPATCH_QUEUE_PRIORITY_DEFAULT, 0), ^{
    // ทำงาน background
    [self performHeavyTask];
    
    dispatch_async(dispatch_get_main_queue(), ^{
        [MBProgressHUD hideHUDForView:self.view animated:YES];
    });
});

// Custom mode - Progress
MBProgressHUD *progressHUD = [MBProgressHUD showHUDAddedTo:self.view animated:YES];
progressHUD.mode = MBProgressHUDModeDeterminate;
progressHUD.label.text = @"กำลัง Upload...";

// อัพเดต progress
progressHUD.progress = 0.5; // 50%

// แสดงแบบ Checkmark
MBProgressHUD *successHUD = [MBProgressHUD showHUDAddedTo:self.view animated:YES];
successHUD.mode = MBProgressHUDModeCustomView;
UIImage *checkmark = [UIImage imageNamed:@"checkmark"];
successHUD.customView = [[UIImageView alloc] initWithImage:checkmark];
successHUD.label.text = @"บันทึกสำเร็จ!";
[successHUD hideAnimated:YES afterDelay:2.0];
```

### Masonry

ไลบรารีสำหรับ Auto Layout ที่ใช้งานง่าย

```objc
// ติดตั้งผ่าน CocoaPods
// pod 'Masonry'

#import <Masonry/Masonry.h>

// แทนที่ NSLayoutConstraint ที่ยาวและซับซ้อน
// ก่อนใช้ Masonry:
[NSLayoutConstraint activateConstraints:@[
    [view.topAnchor constraintEqualToAnchor:self.view.topAnchor constant:20],
    [view.leadingAnchor constraintEqualToAnchor:self.view.leadingAnchor constant:20],
    [view.trailingAnchor constraintEqualToAnchor:self.view.trailingAnchor constant:-20],
    [view.heightAnchor constraintEqualToConstant:50]
]];

// หลังใช้ Masonry:
[view mas_makeConstraints:^(MASConstraintMaker *make) {
    make.top.equalTo(self.view).offset(20);
    make.left.equalTo(self.view).offset(20);
    make.right.equalTo(self.view).offset(-20);
    make.height.equalTo(@50);
}];

// ตัวอย่างการใช้งานเพิ่มเติม
UIView *container = [[UIView alloc] init];
[self.view addSubview:container];

[container mas_makeConstraints:^(MASConstraintMaker *make) {
    make.edges.equalTo(self.view).insets(UIEdgeInsetsMake(20, 20, 20, 20));
}];

// Center in superview
UIImageView *logoView = [[UIImageView alloc] init];
[self.view addSubview:logoView];
[logoView mas_makeConstraints:^(MASConstraintMaker *make) {
    make.center.equalTo(self.view);
    make.size.mas_equalTo(CGSizeMake(100, 100));
}];

// อัพเดต constraint
[view mas_updateConstraints:^(MASConstraintMaker *make) {
    make.top.equalTo(self.view).offset(40); // เปลี่ยนจาก 20 เป็น 40
}];

// สร้าง grid layout
NSArray *buttons = @[button1, button2, button3];
UIView *previous = nil;
for (UIButton *button in buttons) {
    [self.view addSubview:button];
    [button mas_makeConstraints:^(MASConstraintMaker *make) {
        make.left.right.equalTo(self.view).insets(UIEdgeInsetsMake(0, 20, 0, 20));
        make.height.equalTo(@44);
        if (previous) {
            make.top.equalTo(previous.mas_bottom).offset(10);
        } else {
            make.top.equalTo(self.view).offset(100);
        }
    }];
    previous = button;
}
```

---

## 94.2 การใช้งาน CocoaPods

CocoaPods เป็น dependency manager ที่นิยมใช้มากที่สุดใน iOS

### การติดตั้ง CocoaPods

```bash
# ติดตั้ง CocoaPods
sudo gem install cocoapods

# ตรวจสอบ version
pod --version

# อัพเดต repo
pod repo update
```

### การสร้าง Podfile

```ruby
# Podfile - ตัวอย่างที่สมบูรณ์

# กำหนด platform และ minimum version
platform :ios, '13.0'

# ปิด warning จาก pod dependencies
inhibit_all_warnings!

# ใช้ framework แทน static library (แนะนำสำหรับ Swift integration)
use_frameworks!

# Main target
target 'MyApp' do
    # Networking
    pod 'AFNetworking', '~> 4.0'
    
    # Image Loading
    pod 'SDWebImage', '~> 5.0'
    
    # UI Components
    pod 'MBProgressHUD', '~> 1.2'
    pod 'Masonry', '~> 1.1'
    
    # JSON
    pod 'YYModel', '~> 1.0.4'
    
    # Database
    pod 'FMDB', '~> 2.7'
    
    # Analytics
    pod 'Firebase/Analytics'
    pod 'Firebase/Crashlytics'
    
    # Testing
    target 'MyAppTests' do
        inherit! :search_paths
        pod 'OCMock', '~> 3.9'
        pod 'Expecta', '~> 1.0'
    end
    
    target 'MyAppUITests' do
        inherit! :search_paths
    end
end

# Configurations สำหรับ multiple targets
target 'MyAppLite' do
    pod 'AFNetworking', '~> 4.0'
    pod 'SDWebImage', '~> 5.0'
end

# Post-install hook สำหรับการ customize
post_install do |installer|
    installer.pods_project.targets.each do |target|
        target.build_configurations.each do |config|
            config.build_settings['IPHONEOS_DEPLOYMENT_TARGET'] = '13.0'
        end
    end
end
```

### CocoaPods Commands สำคัญ

```bash
# Initialize Podfile ใหม่
pod init

# ติดตั้ง dependencies
pod install

# อัพเดต dependencies
pod update

# อัพเดต pod เฉพาะตัว
pod update AFNetworking

# ตรวจสอบ dependencies ที่ outdated
pod outdated

# ค้นหา pod
pod search SDWebImage

# ดูข้อมูล pod
pod spec cat AFNetworking

# ล้าง cache
pod cache clean --all

# Deintegrate CocoaPods
pod deintegrate
```

---

## 94.3 การใช้งาน Carthage

Carthage เป็น dependency manager อีกตัวที่ไม่ต้อง integrate เข้า Xcode project โดยตรง

### การติดตั้ง Carthage

```bash
# ติดตั้งผ่าน Homebrew
brew install carthage

# ตรวจสอบ version
carthage version
```

### การสร้าง Cartfile

```ruby
# Cartfile
github "Alamofire/Alamofire" ~> 5.6
github "SDWebImage/SDWebImage" ~> 5.0
github "nicklockwood/SwipeView" >= 1.3.2

# จาก GitLab หรือ custom git
git "https://enterprise.example.com/ios/SomeLibrary" "v1.0"

# จาก binary
binary "https://example.com/framework.json" ~> 1.0
```

### การ Build Dependencies

```bash
# Build สำหรับ iOS
carthage update --platform iOS

# Build เฉพาะ library
carthage update SDWebImage --platform iOS

# Build พร้อม XCFramework (รองรับ Apple Silicon)
carthage update --platform iOS --use-xcframeworks

# ดู dependencies ทั้งหมด
carthage version

# ตรวจสอบ build
carthage build --no-skip-current
```

### การเพิ่ม Frameworks ใน Xcode

```
1. ไปที่ Project → Target → General → Frameworks, Libraries, and Embedded Content
2. ลาก .framework จาก Carthage/Build/iOS เข้าไป
3. ไปที่ Build Phases → เพิ่ม New Run Script Phase
4. ใส่ script:
   /usr/local/bin/carthage copy-frameworks
5. เพิ่ม Input Files:
   $(SRCROOT)/Carthage/Build/iOS/AFNetworking.framework
6. เพิ่ม Output Files:
   $(BUILT_PRODUCTS_DIR)/$(FRAMEWORKS_FOLDER_PATH)/AFNetworking.framework
```

---

## 94.4 Swift Package Manager สำหรับ Objective-C

Swift Package Manager (SPM) ยังรองรับ Objective-C libraries ด้วย

### การเพิ่ม Dependencies ผ่าน SPM

```swift
// Package.swift (ถ้าสร้าง package ของตัวเอง)
// swift-tools-version:5.5
import PackageDescription

let package = Package(
    name: "MyApp",
    platforms: [
        .iOS(.v13)
    ],
    dependencies: [
        // เพิ่ม SPM-compatible libraries
        .package(url: "https://github.com/SDWebImage/SDWebImage.git", from: "5.0.0"),
        .package(url: "https://github.com/Alamofire/Alamofire.git", from: "5.6.0"),
    ],
    targets: [
        .target(
            name: "MyApp",
            dependencies: [
                "SDWebImage",
                "Alamofire"
            ]
        )
    ]
)
```

**การเพิ่ม Package ผ่าน Xcode UI:**
1. File → Add Packages...
2. ใส่ URL ของ GitHub repository
3. เลือก version rule
4. เลือก target ที่ต้องการ

---

## 94.5 การอ่านและเข้าใจโค้ด Open Source

### เทคนิคการอ่านโค้ด Open Source

**1. เริ่มจาก README**
```
- อ่าน README.md อย่างละเอียด
- ดู badges (CI status, coverage, version)
- ดู installation instructions
- ดู quick start examples
```

**2. ดู Structure ของ Project**
```
AFNetworking/
├── AFNetworking/
│   ├── AFNetworking.h          # Main header (umbrella)
│   ├── AFURLSessionManager.h   # Core class
│   ├── AFHTTPSessionManager.h  # HTTP convenience class
│   ├── AFURLRequestSerialization.h
│   ├── AFURLResponseSerialization.h
│   └── AFSecurityPolicy.h
├── Tests/
│   └── Tests/
├── Podspec Metadata/
│   └── AFNetworking.podspec
└── README.md
```

**3. อ่าน Main Interface File**
```objc
// เริ่มจาก .h files ก่อนเสมอ
// AFHTTPSessionManager.h - เข้าใจ public API

@interface AFHTTPSessionManager : AFURLSessionManager

// Factory methods
+ (instancetype)manager;

// Initializers
- (instancetype)initWithBaseURL:(nullable NSURL *)url;

// HTTP Methods
- (NSURLSessionDataTask *)GET:(NSString *)URLString
                   parameters:(nullable id)parameters
                      headers:(nullable NSDictionary<NSString *, NSString *> *)headers
                     progress:(nullable void (^)(NSProgress *downloadProgress))downloadProgress
                      success:(nullable void (^)(NSURLSessionDataTask *task, id _Nullable responseObject))success
                      failure:(nullable void (^)(NSURLSessionDataTask * _Nullable task, NSError *error))failure;

// Serializers
@property (nonatomic, strong) AFHTTPRequestSerializer <AFURLRequestSerialization> *requestSerializer;
@property (nonatomic, strong) AFHTTPResponseSerializer <AFURLResponseSerialization> *responseSerializer;

@end
```

**4. ดู Tests เพื่อเข้าใจ Usage**
```objc
// Tests บอกเราถึงวิธีใช้งานที่ถูกต้อง
- (void)testGETRequest {
    XCTestExpectation *expectation = [self expectationWithDescription:@"GET request"];
    
    [self.manager GET:@"/get" 
           parameters:nil 
             headers:nil 
             progress:nil 
              success:^(NSURLSessionDataTask *task, id responseObject) {
        XCTAssertNotNil(responseObject);
        [expectation fulfill];
    } failure:^(NSURLSessionDataTask *task, NSError *error) {
        XCTFail(@"Error: %@", error);
    }];
    
    [self waitForExpectationsWithTimeout:10.0 handler:nil];
}
```

**5. ติดตาม CHANGELOG**
```
CHANGELOG.md บอกว่า:
- version นี้เพิ่มอะไร
- แก้บั๊กอะไร
- มี breaking changes ไหม
- ใครมีส่วนร่วม
```

---

## 94.6 การมีส่วนร่วมใน Open Source

### Workflow การ Contribute

```bash
# 1. Fork repository บน GitHub
# (คลิก Fork button บน GitHub.com)

# 2. Clone fork ของคุณ
git clone https://github.com/YOUR_USERNAME/AFNetworking.git
cd AFNetworking

# 3. เพิ่ม upstream remote
git remote add upstream https://github.com/AFNetworking/AFNetworking.git

# 4. อัพเดตกับ upstream
git fetch upstream
git checkout main
git merge upstream/main

# 5. สร้าง branch ใหม่
git checkout -b fix/memory-leak-in-request-serializer

# 6. ทำการเปลี่ยนแปลง
# ... เขียนโค้ด, แก้บั๊ก ...

# 7. ทดสอบ
xcodebuild test -scheme AFNetworking -destination 'platform=iOS Simulator,name=iPhone 14'

# 8. Commit
git add -A
git commit -m "Fix memory leak in AFURLRequestSerialization

The request serializer was not properly releasing the copied URL
components in error cases. This patch adds appropriate cleanup.

Fixes #1234"

# 9. Push ไปยัง fork
git push origin fix/memory-leak-in-request-serializer

# 10. สร้าง Pull Request บน GitHub
```

### การเขียน Good Commit Message

```
Format:
<type>(<scope>): <short description>

<body>

<footer>

Types: fix, feat, docs, style, refactor, test, chore
Scopes: เช่น serializer, session, security

ตัวอย่าง:
fix(serializer): fix memory leak when encoding multipart form data

AFURLRequestSerialization was not releasing temporary data buffers
when an error occurred during multipart encoding. This led to memory
leaks in error paths.

Fixes #456
```

### การเขียน Pull Request ที่ดี

```markdown
## Description
แก้ไข memory leak ใน AFURLRequestSerialization เมื่อ encode multipart form data ผิดพลาด

## Motivation and Context
Issue #456 รายงานว่าแอพมี memory usage สูงขึ้นเมื่อ upload ไฟล์ล้มเหลว

## How Has This Been Tested?
- [ ] Unit tests ผ่านทั้งหมด
- [ ] ทดสอบบน iPhone 13 iOS 16
- [ ] ทดสอบ upload ทั้ง success และ failure cases

## Types of changes
- [x] Bug fix (non-breaking change which fixes an issue)
- [ ] New feature (non-breaking change which adds functionality)
- [ ] Breaking change (fix or feature that would cause existing functionality to change)

## Checklist
- [x] My code follows the code style of this project
- [x] My change requires a change to the documentation
- [x] I have updated the documentation accordingly
- [x] I have read the CONTRIBUTING document
- [x] I have added tests to cover my changes
- [x] All new and existing tests passed
```

---

## 94.7 การสร้าง Library ของตัวเอง

### โครงสร้าง Library

```
TKNetworkKit/
├── TKNetworkKit/
│   ├── TKNetworkKit.h          # Umbrella header
│   ├── TKHTTPClient.h
│   ├── TKHTTPClient.m
│   ├── TKRequest.h
│   ├── TKRequest.m
│   ├── TKResponse.h
│   ├── TKResponse.m
│   └── Categories/
│       ├── NSString+TKNetworking.h
│       └── NSString+TKNetworking.m
├── TKNetworkKitTests/
│   ├── TKHTTPClientTests.m
│   └── TKRequestTests.m
├── Example/
│   ├── TKNetworkKitExample.xcodeproj
│   └── TKNetworkKitExample/
├── TKNetworkKit.podspec
├── Cartfile
├── Package.swift
├── LICENSE
├── CHANGELOG.md
└── README.md
```

### การสร้าง Umbrella Header

```objc
// TKNetworkKit.h - รวม public headers ทั้งหมด
#import <Foundation/Foundation.h>

//! Project version number for TKNetworkKit.
FOUNDATION_EXPORT double TKNetworkKitVersionNumber;

//! Project version string for TKNetworkKit.
FOUNDATION_EXPORT const unsigned char TKNetworkKitVersionString[];

// Public Headers
#import <TKNetworkKit/TKHTTPClient.h>
#import <TKNetworkKit/TKRequest.h>
#import <TKNetworkKit/TKResponse.h>
#import <TKNetworkKit/NSString+TKNetworking.h>
```

### การเขียน Podspec

```ruby
# TKNetworkKit.podspec

Pod::Spec.new do |spec|
  # ข้อมูลพื้นฐาน
  spec.name         = "TKNetworkKit"
  spec.version      = "1.0.0"
  spec.summary      = "A lightweight HTTP networking library for iOS"
  spec.description  = <<-DESC
    TKNetworkKit is a simple, lightweight HTTP networking library 
    built on top of NSURLSession. It provides an easy-to-use API
    for making HTTP requests with built-in support for JSON,
    retry logic, and request queuing.
  DESC
  
  # Metadata
  spec.homepage     = "https://github.com/yourusername/TKNetworkKit"
  spec.license      = { :type => "MIT", :file => "LICENSE" }
  spec.author       = { "Your Name" => "your@email.com" }
  
  # Platform
  spec.ios.deployment_target = "13.0"
  
  # Source
  spec.source = { 
    :git => "https://github.com/yourusername/TKNetworkKit.git", 
    :tag => "#{spec.version}" 
  }
  
  # Source files
  spec.source_files = "TKNetworkKit/**/*.{h,m}"
  
  # Public headers
  spec.public_header_files = "TKNetworkKit/**/*.h"
  
  # Dependencies
  spec.dependency "AFNetworking", "~> 4.0"
  
  # Frameworks
  spec.frameworks = "Foundation", "UIKit"
  
  # Subspec (optional) สำหรับ modular distribution
  spec.subspec 'Core' do |core|
    core.source_files = "TKNetworkKit/Core/**/*.{h,m}"
  end
  
  spec.subspec 'Cache' do |cache|
    cache.source_files = "TKNetworkKit/Cache/**/*.{h,m}"
    cache.dependency 'TKNetworkKit/Core'
  end
  
  # Test spec
  spec.test_spec 'Tests' do |test_spec|
    test_spec.source_files = "TKNetworkKitTests/**/*.{h,m}"
    test_spec.dependency 'OCMock', '~> 3.9'
  end
  
  # Build settings
  spec.pod_target_xcconfig = { 
    'SWIFT_VERSION' => '5.0',
    'CLANG_ALLOW_NON_MODULAR_INCLUDES_IN_FRAMEWORK_MODULES' => 'YES'
  }
end
```

### การ Publish Pod ไปยัง CocoaPods Trunk

```bash
# 1. ลงทะเบียน session
pod trunk register your@email.com 'Your Name' --description='MacBook Pro'

# 2. ตรวจสอบ email ที่ได้รับ
# คลิก link ใน email เพื่อ verify

# 3. Validate podspec
pod spec lint TKNetworkKit.podspec

# 4. Validate กับ remote source
pod spec lint TKNetworkKit.podspec --allow-warnings

# 5. Push ไปยัง trunk
pod trunk push TKNetworkKit.podspec

# 6. ตรวจสอบว่า pod พร้อมใช้งาน
pod search TKNetworkKit
```

---

## 94.8 Versioning และ Changelog

### Semantic Versioning (SemVer)

```
MAJOR.MINOR.PATCH

MAJOR: เปลี่ยน API ที่ incompatible (breaking changes)
MINOR: เพิ่ม functionality ใหม่ที่ backward compatible
PATCH: แก้บั๊กที่ backward compatible

ตัวอย่าง:
1.0.0 - Initial release
1.0.1 - Fix crash when nil URL passed
1.1.0 - Add support for multipart form data
1.1.1 - Fix memory leak in multipart encoding
2.0.0 - Rewrite API (breaking changes)

Pre-release:
1.0.0-alpha.1
1.0.0-beta.2
1.0.0-rc.1
```

### การสร้าง Git Tags

```bash
# สร้าง annotated tag
git tag -a 1.0.0 -m "Release version 1.0.0"

# Push tag ไปยัง remote
git push origin 1.0.0

# Push ทุก tags
git push origin --tags

# ดู tags ทั้งหมด
git tag -l

# ลบ tag
git tag -d 1.0.0
git push origin :refs/tags/1.0.0
```

### การเขียน CHANGELOG

```markdown
# Changelog
All notable changes to TKNetworkKit will be documented in this file.

The format is based on [Keep a Changelog](https://keepachangelog.com/en/1.0.0/),
and this project adheres to [Semantic Versioning](https://semver.org/spec/v2.0.0.html).

## [Unreleased]

## [1.1.0] - 2024-01-15
### Added
- เพิ่ม support สำหรับ multipart form data upload
- เพิ่ม retry logic พร้อม exponential backoff
- เพิ่ม request queuing

### Changed
- ปรับปรุง error messages ให้ชัดเจนขึ้น
- อัพเดต minimum deployment target เป็น iOS 13

### Fixed
- แก้ memory leak ใน response serializer
- แก้ crash เมื่อ network ไม่พร้อมใช้งาน

### Deprecated
- `makeRequest:` method - ใช้ `sendRequest:completion:` แทน

## [1.0.1] - 2023-12-01
### Fixed
- แก้ crash เมื่อ pass nil URL
- แก้ race condition ใน concurrent requests

## [1.0.0] - 2023-11-15
### Added
- Initial release
- GET, POST, PUT, DELETE support
- JSON serialization/deserialization
- Request/response interceptors

[Unreleased]: https://github.com/user/TKNetworkKit/compare/1.1.0...HEAD
[1.1.0]: https://github.com/user/TKNetworkKit/compare/1.0.1...1.1.0
[1.0.1]: https://github.com/user/TKNetworkKit/compare/1.0.0...1.0.1
[1.0.0]: https://github.com/user/TKNetworkKit/releases/tag/1.0.0
```

---

## 94.9 Documentation

### appledoc

appledoc ใช้สร้าง documentation จาก Javadoc-style comments

```bash
# ติดตั้ง
brew install appledoc

# สร้าง documentation
appledoc \
    --project-name "TKNetworkKit" \
    --project-company "Your Company" \
    --company-id "com.yourcompany" \
    --output "~/help" \
    --keep-undocumented-objects \
    --keep-undocumented-members \
    --no-repeat-first-par \
    --ignore ".m" \
    .
```

### การเขียน Header Doc Comments

```objc
/**
 TKHTTPClient เป็น class หลักสำหรับทำ HTTP requests
 
 ตัวอย่างการใช้งาน:
 
 @code
 TKHTTPClient *client = [TKHTTPClient clientWithBaseURL:[NSURL URLWithString:@"https://api.example.com"]];
 [client GET:@"/users" parameters:nil completion:^(id response, NSError *error) {
     if (!error) {
         NSLog(@"Users: %@", response);
     }
 }];
 @endcode
 
 @see TKRequest
 @see TKResponse
 */
@interface TKHTTPClient : NSObject

/**
 สร้าง TKHTTPClient instance ด้วย base URL
 
 @param baseURL URL พื้นฐานที่จะใช้เป็น prefix ของทุก request
 @return instance ใหม่ของ TKHTTPClient
 
 @warning baseURL ต้องไม่เป็น nil
 */
+ (instancetype)clientWithBaseURL:(NSURL *)baseURL;

/**
 ส่ง GET request ไปยัง endpoint ที่กำหนด
 
 @param path Path ของ endpoint (จะถูก append ต่อจาก baseURL)
 @param parameters NSDictionary ของ query parameters (nullable)
 @param completion Callback ที่เรียกเมื่อ request เสร็จสิ้น
                   - response: Response object (NSDictionary หรือ NSArray)
                   - error: NSError ถ้า request ล้มเหลว
 
 @discussion เมธอดนี้ทำงานแบบ asynchronous completion จะถูกเรียกบน main queue เสมอ
 
 @note ถ้า parameters เป็น nil จะไม่มี query string
 */
- (void)GET:(NSString *)path
 parameters:(nullable NSDictionary *)parameters
 completion:(void(^)(id _Nullable response, NSError * _Nullable error))completion;

/**
 Timeout interval สำหรับ request (default: 30 วินาที)
 */
@property (nonatomic, assign) NSTimeInterval timeoutInterval;

/**
 Headers เพิ่มเติมที่จะส่งใน request ทุกครั้ง
 */
@property (nonatomic, copy) NSDictionary<NSString *, NSString *> *defaultHeaders;

@end
```

### Jazzy สำหรับ HTML Documentation

```bash
# ติดตั้ง
gem install jazzy

# สร้าง documentation
jazzy \
    --objc \
    --umbrella-header TKNetworkKit/TKNetworkKit.h \
    --framework-root . \
    --module TKNetworkKit \
    --output docs/ \
    --clean

# หรือสร้าง .jazzy.yaml
```

```yaml
# .jazzy.yaml
module: TKNetworkKit
module_version: 1.0.0
author: Your Name
author_url: https://example.com
github_url: https://github.com/user/TKNetworkKit
github_file_prefix: https://github.com/user/TKNetworkKit/blob/main
min_acl: public
objc_document_prefix: TK
umbrella_header: TKNetworkKit/TKNetworkKit.h
framework_root: .
output: docs
clean: true
theme: fullwidth

custom_categories:
  - name: Networking
    children:
      - TKHTTPClient
      - TKRequest
      - TKResponse
  - name: Configuration
    children:
      - TKConfiguration
      - TKSSLPinning
```

---

## 94.10 แบบฝึกหัด

### แบบฝึกหัดที่ 1: ใช้งาน CocoaPods

สร้าง iOS project ใหม่และเพิ่ม dependencies ต่อไปนี้:

```ruby
# Podfile
platform :ios, '14.0'
use_frameworks!

target 'PracticeApp' do
    pod 'AFNetworking', '~> 4.0'
    pod 'SDWebImage', '~> 5.0'
    pod 'MBProgressHUD', '~> 1.2'
    
    target 'PracticeAppTests' do
        inherit! :search_paths
        pod 'OCMock'
    end
end
```

จากนั้นสร้าง ViewController ที่:
1. โหลด list of users จาก JSONPlaceholder API
2. แสดงใน UITableView พร้อมรูปโปรไฟล์ (ใช้ SDWebImage)
3. แสดง loading indicator (ใช้ MBProgressHUD)

### แบบฝึกหัดที่ 2: สร้าง Simple Pod

สร้าง pod สำหรับ utility functions:

```objc
// TKStringUtils.h
@interface TKStringUtils : NSObject

/**
 ตรวจสอบว่า string เป็น valid email หรือเปล่า
 */
+ (BOOL)isValidEmail:(NSString *)email;

/**
 ตรวจสอบว่า string เป็น valid Thai phone number หรือเปล่า
 */
+ (BOOL)isValidThaiPhoneNumber:(NSString *)phoneNumber;

/**
 แปลง NSString เป็น slug
 @example "Hello World" -> "hello-world"
 */
+ (NSString *)slugFromString:(NSString *)string;

@end
```

### แบบฝึกหัดที่ 3: อ่านและ Contribute Open Source

1. Fork repository ของ SDWebImage
2. อ่านโค้ดใน `SDWebImageDownloader.m`
3. สร้าง unit test สำหรับ method ที่ยังไม่มี test
4. สร้าง Pull Request

### แบบฝึกหัดที่ 4: Documentation

เพิ่ม documentation ให้กับไลบรารีที่สร้างในแบบฝึกหัดที่ 2 และสร้าง HTML docs ด้วย jazzy

---

## สรุป

ในส่วนนี้เราได้เรียนรู้:
- ไลบรารี Objective-C ยอดนิยม (AFNetworking, SDWebImage, MBProgressHUD, Masonry)
- การใช้ CocoaPods, Carthage และ Swift Package Manager
- การอ่านและเข้าใจโค้ด Open Source
- Workflow การ Contribute ไปยัง Open Source (Fork, Branch, PR)
- การสร้าง Library และ Podspec ของตัวเอง
- Versioning และ Changelog
- การสร้าง Documentation

---

*ส่วนถัดไป: Part 95 - App Store Submission*
