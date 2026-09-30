# ตอนที่ 90: CI/CD และ Deployment สำหรับ iOS

## บทนำ

การพัฒนา iOS app ไม่ได้จบแค่การเขียนโค้ด การส่งมอบแอปให้ถึงมือผู้ใช้อย่างรวดเร็วและเชื่อถือได้เป็นสิ่งสำคัญมาก **Continuous Integration (CI)** และ **Continuous Delivery/Deployment (CD)** คือกระบวนการที่ช่วยให้ทีมพัฒนาสามารถ build, test, และ deploy แอปได้โดยอัตโนมัติ

บทนี้จะครอบคลุมทุกอย่างตั้งแต่ Xcode build system พื้นฐานไปจนถึง CI/CD pipeline แบบสมบูรณ์

---

## 90.1 Xcode Build System

### ภาพรวมของ Xcode Build System

```
Xcode Build Process:
1. Preprocessing
   ├── Process header search paths
   └── Evaluate build settings

2. Compilation
   ├── Compile Objective-C/Swift files
   ├── Generate Swift header (for ObjC interop)
   └── Check dependencies

3. Linking
   ├── Link object files
   ├── Link frameworks
   └── Apply code signing

4. Packaging
   ├── Copy resources
   ├── Process Info.plist
   └── Create .app bundle

5. Code Signing
   ├── Sign app bundle
   └── Verify provisioning profile
```

### Build Phases ใน Xcode

```
Target Build Phases:
├── Dependencies (ต้อง build ก่อน)
├── Compile Sources (ไฟล์โค้ด)
├── Link Binary With Libraries (frameworks, libraries)
├── Copy Bundle Resources (assets, plists, storyboards)
├── Run Script (custom scripts)
└── Embed Frameworks (สำหรับ dynamic frameworks)
```

### Build Rules

```
Build Rules กำหนดว่า input file type ถูก build อย่างไร
├── Swift source files → Swift compiler
├── Objective-C source files → clang compiler
├── Asset catalogs → actool
├── Interface Builder files → ibtool
└── Custom rules (เพิ่มได้)
```

---

## 90.2 Schemes, Targets, Configurations

### Targets

Target คือผลลัพธ์ที่ต้องการ build (app, framework, test bundle)

```
Project Structure:
MyApp.xcodeproj
├── MyApp (Target - แอปหลัก)
│   ├── MyApp source files
│   └── MyApp resources
├── MyAppTests (Target - Unit Tests)
├── MyAppUITests (Target - UI Tests)
├── MyAppExtension (Target - Widget Extension)
└── MyFramework (Target - Custom Framework)
```

### Configurations

Configuration คือชุดของ Build Settings สำหรับสภาพแวดล้อมต่างๆ

```
Default Configurations:
├── Debug
│   ├── Optimization Level: None [-O0]
│   ├── Debug Information Format: DWARF with dSYM
│   └── Swift Optimization: -Onone
└── Release
    ├── Optimization Level: Fastest [-O2]
    ├── Debug Information Format: DWARF with dSYM file
    └── Swift Optimization: -O

Custom Configurations:
├── Staging (คล้าย Debug แต่ใช้ staging server)
├── Production (= Release)
└── Beta (สำหรับ TestFlight)
```

### เพิ่ม Custom Configuration

```
1. ไปที่ Project > Info > Configurations
2. คลิก + เพื่อเพิ่ม Configuration ใหม่
3. Duplicate "Release" เป็น "Staging"
4. ตั้งค่า Build Settings ต่างๆ สำหรับ Staging
```

### Schemes

Scheme กำหนดว่าจะ build, run, test, profile อย่างไร

```
Scheme Actions:
├── Build     - จะ build targets อะไร
├── Run       - ใช้ configuration ไหน, launch arguments อะไร
├── Test      - ใช้ configuration ไหน, test plans อะไร
├── Profile   - สำหรับ Instruments
├── Analyze   - Static analysis
└── Archive   - สำหรับ distribution/submission
```

```
# xcodebuild commands
# Build ด้วย specific scheme และ configuration

# Build สำหรับ simulator
xcodebuild -project MyApp.xcodeproj \
           -scheme MyApp \
           -configuration Debug \
           -destination 'platform=iOS Simulator,name=iPhone 15,OS=17.0' \
           build

# Build สำหรับ device
xcodebuild -project MyApp.xcodeproj \
           -scheme MyApp \
           -configuration Release \
           -destination 'generic/platform=iOS' \
           build

# Run tests
xcodebuild test \
           -project MyApp.xcodeproj \
           -scheme MyAppTests \
           -destination 'platform=iOS Simulator,name=iPhone 15'

# Archive สำหรับ distribution
xcodebuild archive \
           -project MyApp.xcodeproj \
           -scheme MyApp \
           -configuration Release \
           -archivePath ./build/MyApp.xcarchive

# Export IPA
xcodebuild -exportArchive \
           -archivePath ./build/MyApp.xcarchive \
           -exportOptionsPlist ExportOptions.plist \
           -exportPath ./build/IPA
```

---

## 90.3 xcconfig Files

xcconfig (Xcode Configuration) files คือไฟล์ text ที่เก็บ Build Settings ทำให้ version control ได้และแชร์ข้ามโปรเจกต์ได้

### โครงสร้าง xcconfig

```
Configurations/
├── Base.xcconfig           (shared settings)
├── Debug.xcconfig          (debug-specific)
├── Release.xcconfig        (release-specific)
├── Staging.xcconfig        (staging-specific)
└── Pods/
    ├── Pods.debug.xcconfig
    └── Pods.release.xcconfig
```

### ตัวอย่าง xcconfig Files

```bash
# Base.xcconfig - Settings ที่ใช้ร่วมกันทุก configuration

// Product Information
PRODUCT_BUNDLE_IDENTIFIER = com.company.$(PRODUCT_NAME:rfc1034identifier)
PRODUCT_NAME = MyApp
MARKETING_VERSION = 1.0.0

// Build Information
SWIFT_VERSION = 5.9
IPHONEOS_DEPLOYMENT_TARGET = 16.0
TARGETED_DEVICE_FAMILY = 1,2

// Code Signing
CODE_SIGN_STYLE = Automatic
DEVELOPMENT_TEAM = ABCD1234EF

// Swift Settings
SWIFT_OPTIMIZATION_LEVEL = -Onone
ENABLE_BITCODE = NO

// Include other xcconfig
#include "Pods/Pods.debug.xcconfig"
```

```bash
# Debug.xcconfig
#include "Base.xcconfig"

// Override for Debug
SWIFT_OPTIMIZATION_LEVEL = -Onone
DEBUG_INFORMATION_FORMAT = dwarf
GCC_PREPROCESSOR_DEFINITIONS = DEBUG=1 $(inherited)

// Custom build settings
API_BASE_URL = https://api-dev.example.com
ANALYTICS_ENABLED = NO
LOGGING_LEVEL = verbose
```

```bash
# Release.xcconfig
#include "Base.xcconfig"

// Override for Release
SWIFT_OPTIMIZATION_LEVEL = -O
DEBUG_INFORMATION_FORMAT = dwarf-with-dsym
VALIDATE_PRODUCT = YES

// Custom build settings
API_BASE_URL = https://api.example.com
ANALYTICS_ENABLED = YES
LOGGING_LEVEL = error
```

```bash
# Staging.xcconfig
#include "Base.xcconfig"

// Staging is like Release but different server
SWIFT_OPTIMIZATION_LEVEL = -O
DEBUG_INFORMATION_FORMAT = dwarf-with-dsym

// Custom build settings
API_BASE_URL = https://api-staging.example.com
ANALYTICS_ENABLED = YES
LOGGING_LEVEL = warning

// Different bundle ID for staging
PRODUCT_BUNDLE_IDENTIFIER = com.company.$(PRODUCT_NAME:rfc1034identifier).staging
```

### การใช้ Custom Build Settings ใน Code

```objc
// Info.plist
// เพิ่ม custom key ที่ map กับ xcconfig variable:
// API_BASE_URL = $(API_BASE_URL)
// ANALYTICS_ENABLED = $(ANALYTICS_ENABLED)

// ใน Objective-C code
NSString *baseURL = [[NSBundle mainBundle] objectForInfoDictionaryKey:@"API_BASE_URL"];
NSString *analyticsStr = [[NSBundle mainBundle] objectForInfoDictionaryKey:@"ANALYTICS_ENABLED"];
BOOL analyticsEnabled = [analyticsStr boolValue];

NSLog(@"API Base URL: %@", baseURL);
NSLog(@"Analytics enabled: %@", analyticsEnabled ? @"YES" : @"NO");
```

```swift
// ใน Swift code
enum AppConfig {
    static let apiBaseURL: String = {
        guard let url = Bundle.main.object(forInfoDictionaryKey: "API_BASE_URL") as? String else {
            fatalError("API_BASE_URL not configured")
        }
        return url
    }()
    
    static let isAnalyticsEnabled: Bool = {
        let value = Bundle.main.object(forInfoDictionaryKey: "ANALYTICS_ENABLED") as? String
        return value == "YES"
    }()
}
```

---

## 90.4 Code Signing และ Provisioning Profiles

### ความเข้าใจพื้นฐาน

```
Code Signing Components:
├── Certificate (ใบรับรอง)
│   ├── Development Certificate
│   └── Distribution Certificate
├── Provisioning Profile
│   ├── Development Profile (สำหรับ testing บน device)
│   ├── Ad Hoc Profile (สำหรับ beta distribution)
│   ├── App Store Profile (สำหรับ App Store)
│   └── Enterprise Profile (สำหรับ in-house distribution)
└── Entitlements (permissions)
    ├── Push Notifications
    ├── iCloud
    ├── In-App Purchase
    └── etc.
```

### Provisioning Profile ประกอบด้วย

```
Provisioning Profile:
├── App ID (Bundle Identifier)
├── Certificates (กำหนดว่า developer คนไหน sign ได้)
├── Devices (กำหนดว่า device ไหนรันได้ - สำหรับ Dev และ Ad Hoc)
├── Entitlements (permissions ที่แอปมี)
└── Expiration Date
```

### Automatic vs Manual Signing

```
Automatic Signing:
├── Xcode จัดการ certificate และ profile อัตโนมัติ
├── ดีสำหรับ development
└── ต้องการ internet connection และ Apple Developer account

Manual Signing:
├── ต้องสร้าง certificate และ profile ด้วยตัวเอง
├── ดีสำหรับ CI/CD
└── ควบคุมได้มากกว่า
```

### Export Options Plist

```xml
<!-- ExportOptions.plist -->
<?xml version="1.0" encoding="UTF-8"?>
<!DOCTYPE plist PUBLIC "-//Apple//DTD PLIST 1.0//EN" 
  "http://www.apple.com/DTDs/PropertyList-1.0.dtd">
<plist version="1.0">
<dict>
    <!-- App Store Connect -->
    <key>method</key>
    <string>app-store</string>
    
    <!-- สำหรับ Ad Hoc -->
    <!-- <key>method</key>
    <string>ad-hoc</string> -->
    
    <key>teamID</key>
    <string>ABCD1234EF</string>
    
    <key>provisioningProfiles</key>
    <dict>
        <key>com.company.myapp</key>
        <string>MyApp AppStore Distribution</string>
    </dict>
    
    <key>signingCertificate</key>
    <string>Apple Distribution</string>
    
    <key>signingStyle</key>
    <string>manual</string>
    
    <key>stripSwiftSymbols</key>
    <true/>
    
    <key>uploadBitcode</key>
    <false/>
    
    <key>uploadSymbols</key>
    <true/>
    
    <!-- สำหรับ TestFlight -->
    <key>destination</key>
    <string>upload</string>
</dict>
</plist>
```

---

## 90.5 Fastlane Setup และ Usage

### Fastlane คืออะไร?

Fastlane เป็น open-source tool ที่ automate การ build, test, และ release iOS/Android apps ช่วยลดงานซ้ำๆ และลด human error

### การติดตั้ง Fastlane

```bash
# ติดตั้งผ่าน Homebrew (แนะนำ)
brew install fastlane

# หรือผ่าน RubyGems
sudo gem install fastlane

# ตรวจสอบ version
fastlane --version

# Initialize fastlane ใน project
cd /path/to/project
fastlane init
```

### โครงสร้าง Fastlane

```
fastlane/
├── Fastfile        (lanes - automation scripts)
├── Appfile         (app configuration)
├── Matchfile       (certificate/profile management)
├── Gymfile         (build settings)
├── Deliverfile     (App Store metadata)
├── metadata/       (App Store metadata files)
│   ├── en-US/
│   │   ├── name.txt
│   │   ├── description.txt
│   │   ├── keywords.txt
│   │   └── release_notes.txt
│   └── th/
│       └── ...
└── screenshots/    (App Store screenshots)
```

### Appfile

```ruby
# fastlane/Appfile
app_identifier("com.company.myapp")  # Bundle Identifier
apple_id("developer@company.com")    # Apple ID

# สำหรับหลาย environment
for_platform :ios do
  for_lane :staging do
    app_identifier "com.company.myapp.staging"
  end
  
  for_lane :production do
    app_identifier "com.company.myapp"
  end
end
```

### Fastfile - Basic Structure

```ruby
# fastlane/Fastfile
# frozen_string_literal: true

# สิ่งที่ต้องทำก่อน lane ทั้งหมด
before_all do |lane|
  ensure_git_branch(branch: 'main') if lane == :release
  git_pull if lane == :release
end

# Lane สำหรับ run tests
lane :test do
  run_tests(
    project: "MyApp.xcodeproj",
    scheme: "MyAppTests",
    devices: ["iPhone 15 (17.0)"],
    code_coverage: true,
    output_directory: "./test_output"
  )
end

# Lane สำหรับ build สำหรับ TestFlight
lane :beta do
  # Increment build number
  increment_build_number(
    build_number: latest_testflight_build_number + 1
  )
  
  # Build app
  gym(
    scheme: "MyApp",
    configuration: "Release",
    export_method: "app-store"
  )
  
  # Upload to TestFlight
  pilot(
    skip_waiting_for_build_processing: true
  )
  
  # Notify team
  slack(
    message: "Beta build uploaded to TestFlight! :rocket:",
    channel: "#ios-releases"
  ) if ENV["SLACK_URL"]
end

# Lane สำหรับ App Store release
lane :release do |options|
  version = options[:version]
  
  # Ensure clean git state
  ensure_git_status_clean
  
  # Set version
  increment_version_number(version_number: version) if version
  
  # Run tests first
  test
  
  # Build
  gym(
    scheme: "MyApp",
    configuration: "Release"
  )
  
  # Upload to App Store
  deliver(
    submit_for_review: true,
    automatic_release: false,
    force: true
  )
  
  # Tag the release
  git_tag(tag: "v#{version || get_version_number}")
  push_git_tags
  
  UI.success "Successfully released version #{version}!"
end

# Lane สำหรับ screenshot automation
lane :screenshots do
  capture_screenshots
  frame_screenshots
  upload_to_app_store(skip_binary_upload: true)
end

# Error handling
error do |lane, exception|
  slack(
    message: "Error in lane #{lane}: #{exception.message}",
    success: false
  ) if ENV["SLACK_URL"]
end
```

---

## 90.6 Fastlane Match สำหรับ Certificates

### Match คืออะไร?

Match (fastlane match) เป็น tool สำหรับ sync certificates และ provisioning profiles ระหว่างทีม โดยเก็บไว้ใน Git repository ที่ encrypted

### ทำไมต้องใช้ Match?

```
ปัญหาทั่วไปของ Team Code Signing:
├── Developer แต่ละคนมี certificate ของตัวเอง
├── Provisioning profiles ที่ไม่ sync กัน
├── "It works on my machine" syndrome
└── ยุ่งยากเมื่อ CI server ต้อง build

วิธีแก้ด้วย Match:
├── Certificate เดียวสำหรับทั้งทีม
├── Profiles เก็บใน Git (encrypted)
├── CI server ดึง certificates/profiles ได้อัตโนมัติ
└── Developer ใหม่ setup ได้ง่าย
```

### Setup Match

```bash
# Initialize match
fastlane match init

# เลือก storage type:
# 1. git (แนะนำ - เก็บใน private Git repo)
# 2. google_cloud
# 3. s3
# 4. gitlab_secure_files

# สร้าง development certificates/profiles
fastlane match development

# สร้าง distribution (App Store) certificates/profiles
fastlane match appstore

# สร้าง ad-hoc certificates/profiles
fastlane match adhoc
```

### Matchfile Configuration

```ruby
# fastlane/Matchfile

# Storage location
git_url("https://github.com/your-org/certificates-private.git")
# หรือ
# storage_mode("google_cloud")
# google_cloud_bucket_name("your-bucket")

# App-specific settings
app_identifier(["com.company.myapp"])
username("developer@company.com")

# Git branch (สำหรับแยก production กับ staging)
git_branch("main")

# ใช้ keychain name เฉพาะ (สำคัญสำหรับ CI)
keychain_name("CI_KEYCHAIN")
```

### Match ใน Fastfile

```ruby
# fastlane/Fastfile

lane :setup_certificates do
  # ดึง certificates จาก match
  match(
    type: "development",
    readonly: true  # ไม่สร้างใหม่ แค่ดึงที่มีอยู่
  )
end

lane :build_for_store do
  # sync certificates สำหรับ App Store
  match(
    type: "appstore",
    readonly: true,
    keychain_name: ENV["KEYCHAIN_NAME"] || "login",
    keychain_password: ENV["KEYCHAIN_PASSWORD"]
  )
  
  gym(
    scheme: "MyApp",
    export_method: "app-store",
    export_options: {
      provisioningProfiles: {
        "com.company.myapp" => "match AppStore com.company.myapp"
      }
    }
  )
end
```

### CI/CD สำหรับ Match

```bash
# Script สำหรับ CI ที่ต้องการ setup keychain

#!/bin/bash
# ci_setup_keychain.sh

KEYCHAIN_NAME="CI_TEMP_KEYCHAIN"
KEYCHAIN_PASSWORD="temporary_password_for_ci"

# สร้าง temporary keychain
security create-keychain -p "$KEYCHAIN_PASSWORD" "$KEYCHAIN_NAME"
security set-keychain-settings -t 3600 -l "$KEYCHAIN_NAME"
security unlock-keychain -p "$KEYCHAIN_PASSWORD" "$KEYCHAIN_NAME"

# เพิ่ม keychain เข้า search list
security list-keychains -d user -s "$KEYCHAIN_NAME" $(security list-keychains -d user | xargs)

# ตั้งค่า environment variables
export MATCH_KEYCHAIN_NAME="$KEYCHAIN_NAME"
export MATCH_KEYCHAIN_PASSWORD="$KEYCHAIN_PASSWORD"

# รัน match
bundle exec fastlane match appstore --readonly true
```

---

## 90.7 Fastlane Gym สำหรับ Building

### Gymfile Configuration

```ruby
# fastlane/Gymfile

# Project/Workspace
workspace("MyApp.xcworkspace")  # ถ้าใช้ CocoaPods
# project("MyApp.xcodeproj")  # ถ้าไม่มี workspace

# Scheme
scheme("MyApp")

# Configuration
configuration("Release")

# Build destination
destination("generic/platform=iOS")

# Output settings
output_directory("./build")
output_name("MyApp")

# Code signing (สำหรับ CI)
export_method("app-store")
export_xcargs("-allowProvisioningUpdates")

# Clean build
clean(true)

# Disable bitcode
include_bitcode(false)

# Silent output (สำหรับ CI logs)
silent(false)
```

### ตัวอย่าง Advanced Gym Configuration

```ruby
# fastlane/Fastfile
lane :build_app do |options|
  environment = options[:environment] || "production"
  
  # ตั้งค่า build number
  build_number = ENV["BUILD_NUMBER"] || "1"
  increment_build_number(build_number: build_number)
  
  # Build ตาม environment
  case environment
  when "staging"
    gym(
      workspace: "MyApp.xcworkspace",
      scheme: "MyApp-Staging",
      configuration: "Staging",
      export_method: "ad-hoc",
      export_options: {
        provisioningProfiles: {
          "com.company.myapp.staging" => "match AdHoc com.company.myapp.staging"
        }
      },
      output_directory: "./build/staging"
    )
  when "production"
    gym(
      workspace: "MyApp.xcworkspace",
      scheme: "MyApp",
      configuration: "Release",
      export_method: "app-store",
      export_options: {
        provisioningProfiles: {
          "com.company.myapp" => "match AppStore com.company.myapp"
        },
        uploadBitcode: false,
        uploadSymbols: true
      },
      output_directory: "./build/production",
      include_symbols: true
    )
  end
  
  # บันทึก build path
  build_path = lane_context[SharedValues::IPA_OUTPUT_PATH]
  UI.message "Build completed: #{build_path}"
end
```

---

## 90.8 Fastlane Deliver สำหรับ App Store Upload

### Deliverfile Configuration

```ruby
# fastlane/Deliverfile

# App identification
username("developer@company.com")
app_identifier("com.company.myapp")

# Metadata
app_version("2.1.0")  # Optional: กำหนด version ที่จะ upload

# App Store metadata paths
metadata_path("./fastlane/metadata")
screenshots_path("./fastlane/screenshots")

# App Store options
submit_for_review(false)  # ไม่ submit อัตโนมัติ
automatic_release(false)  # ต้อง release เอง

# Price
price_tier(0)  # Free

# Languages
languages(["en-US", "th"])

# Skip some options
skip_screenshots(false)
skip_metadata(false)
skip_binary_upload(false)

# Submission options
reject_if_possible(true)  # Reject existing pending review if possible
```

### Metadata Structure

```
fastlane/metadata/
├── en-US/
│   ├── name.txt              # App name
│   ├── subtitle.txt          # Subtitle
│   ├── description.txt       # Full description
│   ├── keywords.txt          # Keywords (comma separated)
│   ├── release_notes.txt     # What's new
│   ├── support_url.txt
│   ├── marketing_url.txt
│   └── privacy_url.txt
├── th/
│   ├── name.txt
│   ├── description.txt
│   ├── keywords.txt
│   └── release_notes.txt
└── review_information/
    ├── first_name.txt
    ├── last_name.txt
    ├── phone_number.txt
    ├── email_address.txt
    └── demo_user.txt (ถ้ามี)
```

### ตัวอย่าง Deliver ใน Fastfile

```ruby
# fastlane/Fastfile
lane :deploy_to_appstore do |options|
  version = options[:version]
  
  UI.user_error!("Version is required") unless version
  
  # Update metadata ก่อน upload
  deliver(
    app_version: version,
    ipa: "./build/production/MyApp.ipa",
    
    # Metadata
    name: {
      "en-US" => "My Awesome App",
      "th" => "แอปสุดเจ๋ง"
    },
    
    release_notes: {
      "en-US" => File.read("./CHANGELOG.md"),
      "th" => File.read("./CHANGELOG_TH.md")
    },
    
    # Screenshots  
    screenshots_path: "./fastlane/screenshots",
    
    # Submission
    submit_for_review: true,
    submission_information: {
      add_id_info_serves_ads: false,
      add_id_info_tracks_action: true,
      add_id_info_tracks_install: false,
      add_id_info_uses_idfa: false,
      content_rights_has_rights: true,
      export_compliance_uses_encryption: false
    },
    
    # Review info
    app_review_information: {
      first_name: "John",
      last_name: "Doe",
      phone_number: "+1 555-1234",
      email_address: "appreviewer@company.com",
      demo_user: "reviewer@test.com",
      demo_password: "ReviewPassword123"
    },
    
    force: true  # ไม่ถาม confirmation
  )
end
```

---

## 90.9 GitHub Actions สำหรับ iOS CI

### ภาพรวม GitHub Actions

```
.github/
└── workflows/
    ├── ci.yml              (pull request checks)
    ├── beta.yml            (upload to TestFlight)
    ├── release.yml         (App Store release)
    └── nightly.yml         (nightly builds)
```

### CI Workflow (Pull Request)

```yaml
# .github/workflows/ci.yml
name: iOS CI

on:
  pull_request:
    branches: [main, develop]
  push:
    branches: [main]

env:
  XCODE_VERSION: "15.2"
  RUBY_VERSION: "3.2"

jobs:
  test:
    name: Build & Test
    runs-on: macos-14  # macOS Sonoma with Xcode 15

    steps:
      - name: Checkout code
        uses: actions/checkout@v4

      - name: Setup Ruby
        uses: ruby/setup-ruby@v1
        with:
          ruby-version: ${{ env.RUBY_VERSION }}
          bundler-cache: true  # cache gems

      - name: Select Xcode version
        run: sudo xcode-select -s /Applications/Xcode_${{ env.XCODE_VERSION }}.app

      - name: Install CocoaPods
        run: bundle exec pod install --repo-update
        
      - name: Cache CocoaPods
        uses: actions/cache@v4
        with:
          path: Pods
          key: ${{ runner.os }}-pods-${{ hashFiles('**/Podfile.lock') }}

      - name: Run tests
        run: |
          bundle exec fastlane test
        env:
          DEVELOPER_DIR: /Applications/Xcode_${{ env.XCODE_VERSION }}.app/Contents/Developer

      - name: Upload test results
        uses: actions/upload-artifact@v4
        if: always()
        with:
          name: test-results
          path: test_output/

      - name: Code coverage report
        uses: codecov/codecov-action@v4
        with:
          file: test_output/coverage.xml
          token: ${{ secrets.CODECOV_TOKEN }}

  lint:
    name: Lint
    runs-on: macos-14
    steps:
      - uses: actions/checkout@v4
      
      - name: Setup Ruby
        uses: ruby/setup-ruby@v1
        with:
          ruby-version: ${{ env.RUBY_VERSION }}
          bundler-cache: true

      - name: Run SwiftLint
        run: |
          if which swiftlint >/dev/null; then
            swiftlint lint --reporter github-actions-logging
          else
            brew install swiftlint
            swiftlint lint --reporter github-actions-logging
          fi
```

### Beta Distribution Workflow

```yaml
# .github/workflows/beta.yml
name: Beta Distribution

on:
  push:
    branches: [develop]
  workflow_dispatch:
    inputs:
      notes:
        description: "Release notes for testers"
        required: false

jobs:
  build-and-upload:
    name: Build and Upload to TestFlight
    runs-on: macos-14
    
    steps:
      - name: Checkout
        uses: actions/checkout@v4
        with:
          fetch-depth: 0  # สำหรับ git operations

      - name: Setup Ruby
        uses: ruby/setup-ruby@v1
        with:
          ruby-version: "3.2"
          bundler-cache: true

      - name: Setup Xcode
        run: sudo xcode-select -s /Applications/Xcode_15.2.app

      - name: Setup keychain
        run: |
          security create-keychain -p "${{ secrets.KEYCHAIN_PASSWORD }}" build.keychain
          security set-keychain-settings -t 3600 -l ~/Library/Keychains/build.keychain
          security unlock-keychain -p "${{ secrets.KEYCHAIN_PASSWORD }}" ~/Library/Keychains/build.keychain
          security list-keychains -d user -s ~/Library/Keychains/build.keychain $(security list-keychains -d user | xargs)

      - name: Install pods
        run: bundle exec pod install

      - name: Build and upload beta
        run: bundle exec fastlane beta
        env:
          MATCH_PASSWORD: ${{ secrets.MATCH_PASSWORD }}
          MATCH_GIT_BASIC_AUTHORIZATION: ${{ secrets.MATCH_GIT_TOKEN }}
          FASTLANE_APPLE_APPLICATION_SPECIFIC_PASSWORD: ${{ secrets.APP_SPECIFIC_PASSWORD }}
          DEVELOPER_PORTAL_TEAM_ID: ${{ secrets.TEAM_ID }}
          KEYCHAIN_NAME: "build.keychain"
          KEYCHAIN_PASSWORD: ${{ secrets.KEYCHAIN_PASSWORD }}
          SLACK_URL: ${{ secrets.SLACK_WEBHOOK_URL }}

      - name: Cleanup keychain
        if: always()
        run: security delete-keychain build.keychain
```

### Release Workflow

```yaml
# .github/workflows/release.yml
name: App Store Release

on:
  push:
    tags:
      - 'v*.*.*'  # Trigger เมื่อ push tag เช่น v2.1.0

jobs:
  release:
    name: Release to App Store
    runs-on: macos-14
    environment: production  # ต้อง approve ก่อน
    
    steps:
      - name: Checkout
        uses: actions/checkout@v4
        with:
          fetch-depth: 0

      - name: Get version from tag
        id: get_version
        run: echo "VERSION=${GITHUB_REF#refs/tags/v}" >> $GITHUB_OUTPUT

      - name: Setup Ruby
        uses: ruby/setup-ruby@v1
        with:
          ruby-version: "3.2"
          bundler-cache: true

      - name: Setup Xcode
        run: sudo xcode-select -s /Applications/Xcode_15.2.app

      - name: Setup keychain
        run: |
          security create-keychain -p "${{ secrets.KEYCHAIN_PASSWORD }}" release.keychain
          security set-keychain-settings -t 3600 -l ~/Library/Keychains/release.keychain
          security unlock-keychain -p "${{ secrets.KEYCHAIN_PASSWORD }}" ~/Library/Keychains/release.keychain
          security list-keychains -d user -s ~/Library/Keychains/release.keychain $(security list-keychains -d user | xargs)

      - name: Install CocoaPods
        run: bundle exec pod install

      - name: Release to App Store
        run: |
          bundle exec fastlane release version:${{ steps.get_version.outputs.VERSION }}
        env:
          MATCH_PASSWORD: ${{ secrets.MATCH_PASSWORD }}
          MATCH_GIT_BASIC_AUTHORIZATION: ${{ secrets.MATCH_GIT_TOKEN }}
          FASTLANE_APPLE_APPLICATION_SPECIFIC_PASSWORD: ${{ secrets.APP_SPECIFIC_PASSWORD }}
          DEVELOPER_PORTAL_TEAM_ID: ${{ secrets.TEAM_ID }}
          KEYCHAIN_NAME: "release.keychain"
          KEYCHAIN_PASSWORD: ${{ secrets.KEYCHAIN_PASSWORD }}

      - name: Create GitHub Release
        uses: actions/create-release@v1
        env:
          GITHUB_TOKEN: ${{ secrets.GITHUB_TOKEN }}
        with:
          tag_name: ${{ github.ref }}
          release_name: Release ${{ steps.get_version.outputs.VERSION }}
          body: |
            See [CHANGELOG.md](CHANGELOG.md) for details.
          draft: false
          prerelease: false
          
      - name: Cleanup keychain
        if: always()
        run: security delete-keychain release.keychain
```

---

## 90.10 Xcode Cloud

Xcode Cloud เป็น CI/CD service ของ Apple ที่ integrate กับ Xcode โดยตรง

### Setup Xcode Cloud

```
1. ใน Xcode: Product > Xcode Cloud > Create Workflow
2. เลือก App (ต้องมี App Store Connect account)
3. กำหนด Workflow:
   - Name: "Build and Test"
   - Start Condition: Source Branch Changes (branch: main)
   - Actions:
     - Build (scheme, platform)
     - Test (test plans)
     - Archive (ถ้าต้องการ)
   - Post-Actions:
     - TestFlight (ถ้าต้องการ upload)
     - Notify (email/Slack)
```

### ci_scripts สำหรับ Xcode Cloud

```bash
# ci_scripts/ci_post_clone.sh
# รันหลัง clone ก่อน build

#!/bin/sh

set -e

echo "Post clone script running..."

# Install CocoaPods ถ้าใช้
if [ -f "Podfile" ]; then
    echo "Installing CocoaPods dependencies..."
    pod install --repo-update
fi

# หรือ Swift Package Manager (อัตโนมัติ แต่สามารถ custom ได้)

echo "Post clone completed"
```

```bash
# ci_scripts/ci_pre_xcodebuild.sh
# รันก่อน xcodebuild

#!/bin/sh

echo "Pre build script..."

# Set build number จาก CI build number
if [ -n "$CI_BUILD_NUMBER" ]; then
    /usr/libexec/PlistBuddy -c "Set :CFBundleVersion $CI_BUILD_NUMBER" \
        "$CI_PRIMARY_REPOSITORY_PATH/MyApp/Info.plist"
fi

# อื่นๆ ที่ต้องทำก่อน build
echo "Pre build completed"
```

```bash
# ci_scripts/ci_post_xcodebuild.sh
# รันหลัง xcodebuild

#!/bin/sh

echo "Post build script..."

# ส่ง notification ไป Slack
if [ -n "$SLACK_WEBHOOK_URL" ]; then
    if [ "$CI_WORKFLOW_ID" == "build-and-test" ]; then
        curl -X POST -H 'Content-type: application/json' \
            --data '{"text":"iOS build completed successfully! :white_check_mark:"}' \
            "$SLACK_WEBHOOK_URL"
    fi
fi

echo "Post build completed"
```

### Xcode Cloud Environment Variables

```
Built-in Variables:
├── CI_BUILD_ID          (unique build ID)
├── CI_BUILD_NUMBER      (incrementing number)
├── CI_COMMIT            (git commit hash)
├── CI_BRANCH            (current branch)
├── CI_TAG               (current tag if any)
├── CI_WORKFLOW_ID       (workflow name)
├── CI_PRODUCT_PLATFORM  (iOS, macOS, etc.)
└── CI_ARCHIVE_PATH      (path to .xcarchive)

Custom Variables:
เพิ่มได้ใน Xcode Cloud workflow settings
(ต้องการ secrets จะ mask ออกจาก logs)
```

---

## 90.11 TestFlight

### TestFlight คืออะไร?

TestFlight เป็น Apple's beta testing platform ที่ช่วยให้ distribute แอปให้ testers ก่อน submit ไป App Store

### การ Upload ไป TestFlight

```ruby
# Fastfile - Upload to TestFlight
lane :upload_testflight do |options|
  # Build ก่อน (ถ้าจำเป็น)
  ipa_path = options[:ipa] || "./build/MyApp.ipa"
  
  pilot(
    ipa: ipa_path,
    
    # TestFlight notes
    changelog: options[:notes] || "Bug fixes and improvements",
    
    # Testers
    groups: ["Internal", "Beta Testers"],
    
    # Options
    skip_waiting_for_build_processing: false,  # รอ processing (สำมหรับ immediate testing)
    skip_submission: false,
    
    # Demo account (สำหรับ review)
    demo_account_required: false,
    
    # Beta app description
    beta_app_description: "This is a beta version for testing",
    beta_app_feedback_email: "feedback@company.com"
  )
end
```

### Managing TestFlight Testers

```ruby
# เพิ่ม tester
lane :add_tester do |options|
  pilot(
    add_new_testers: true,
    testers_file_path: "./testers.csv"  # CSV with name, email
  )
end

# ลบ tester
lane :remove_tester do |options|
  pilot(
    remove_from_group: true,
    first_name: options[:first_name],
    last_name: options[:last_name],
    email: options[:email]
  )
end
```

---

## 90.12 App Store Connect API

### Authentication

```bash
# App Store Connect API ใช้ JWT authentication

# สร้าง key ใน App Store Connect:
# 1. ไปที่ App Store Connect > Users and Access > Keys
# 2. สร้าง API Key ใหม่
# 3. Download .p8 file (download ได้ครั้งเดียว!)
# 4. จดบันทึก Key ID และ Issuer ID
```

### ใช้กับ Fastlane

```ruby
# Fastfile - ใช้ App Store Connect API
lane :upload_metadata do
  app_store_connect_api_key(
    key_id: ENV["ASC_KEY_ID"],
    issuer_id: ENV["ASC_ISSUER_ID"],
    key_filepath: ENV["ASC_KEY_PATH"],  # path ไปยัง .p8 file
    duration: 1200,
    in_house: false
  )
  
  deliver(
    skip_binary_upload: true,
    metadata_path: "./fastlane/metadata"
  )
end
```

### App Store Connect API โดยตรง

```bash
# Generate JWT token
# ใช้ library หรือเขียน script เอง

# Python example
python3 << 'EOF'
import jwt
import time
import pathlib

ISSUER_ID = "your-issuer-id"
KEY_ID = "your-key-id"
KEY_FILE = "AuthKey_XXXXXXXXXX.p8"

key = pathlib.Path(KEY_FILE).read_text()

token = jwt.encode(
    {
        "iss": ISSUER_ID,
        "iat": int(time.time()),
        "exp": int(time.time()) + 1200,
        "aud": "appstoreconnect-v1"
    },
    key,
    algorithm="ES256",
    headers={"kid": KEY_ID}
)

print(token)
EOF

# ใช้ token เรียก API
TOKEN="your-jwt-token"

# List apps
curl -H "Authorization: Bearer $TOKEN" \
     "https://api.appstoreconnect.apple.com/v1/apps"

# Get builds
curl -H "Authorization: Bearer $TOKEN" \
     "https://api.appstoreconnect.apple.com/v1/builds?filter[app]=YOUR_APP_ID"
```

---

## 90.13 Versioning และ Build Numbers

### Version Numbering Strategy

```
Version Format: MAJOR.MINOR.PATCH (Semantic Versioning)

MAJOR: Breaking changes (1.0.0 → 2.0.0)
MINOR: New features (1.0.0 → 1.1.0)
PATCH: Bug fixes (1.0.0 → 1.0.1)

Build Number: Unique number ที่เพิ่มขึ้นเรื่อยๆ
ใช้สำหรับ App Store Connect ในการแยก builds
```

### Fastlane สำหรับ Versioning

```ruby
# Fastfile
lane :bump_version do |options|
  # อ่าน version ปัจจุบัน
  current_version = get_version_number(target: "MyApp")
  UI.message "Current version: #{current_version}"
  
  case options[:type]
  when "major"
    increment_version_number(bump_type: "major")
  when "minor"
    increment_version_number(bump_type: "minor")
  when "patch"
    increment_version_number(bump_type: "patch")
  else
    increment_version_number(version_number: options[:version])
  end
  
  new_version = get_version_number(target: "MyApp")
  UI.success "Version bumped to: #{new_version}"
  
  # Commit version change
  git_commit(
    path: ["MyApp/Info.plist"],
    message: "Bump version to #{new_version}"
  )
end

lane :bump_build_number do
  # ใช้ TestFlight build number + 1
  latest_build = latest_testflight_build_number(
    app_identifier: "com.company.myapp"
  )
  
  new_build = latest_build + 1
  increment_build_number(build_number: new_build)
  
  UI.message "Build number: #{new_build}"
end
```

### Automatic Build Numbering จาก Git

```bash
# git_build_number.sh
# ใช้ git commit count เป็น build number

#!/bin/bash
BUILD_NUMBER=$(git rev-list --count HEAD)
echo "Build number: $BUILD_NUMBER"

# Set ใน Info.plist
/usr/libexec/PlistBuddy -c "Set :CFBundleVersion $BUILD_NUMBER" \
    "${SRCROOT}/${TARGETNAME}/Info.plist"
```

```ruby
# หรือใน Fastfile
lane :set_build_number_from_git do
  build_number = sh("git rev-list --count HEAD").strip.to_i
  increment_build_number(build_number: build_number)
  UI.message "Build number set to: #{build_number}"
end
```

---

## 90.14 Beta Testing Distribution

### ช่องทางการ Distribute Beta

```
Distribution Options:
├── TestFlight (Apple's official)
│   ├── Internal Testing (ทีม, max 100 คน)
│   └── External Testing (max 10,000 คน, ต้องผ่าน review)
├── Ad Hoc (ไฟล์ IPA โดยตรง, max 100 devices)
├── Firebase App Distribution
└── Diawi / AppCenter
```

### TestFlight Internal Testing

```ruby
# ไม่ต้องผ่าน review - internal team เท่านั้น
lane :internal_beta do
  build_app(
    scheme: "MyApp",
    export_method: "app-store"
  )
  
  pilot(
    groups: ["Internal Beta"],
    changelog: "New features for internal testing",
    distribute_external: false  # Internal only
  )
end
```

### TestFlight External Testing

```ruby
# ต้องผ่าน review ครั้งแรก แต่ครั้งต่อๆ ไปเร็วกว่า App Store review
lane :external_beta do
  build_app(
    scheme: "MyApp",
    export_method: "app-store"
  )
  
  pilot(
    groups: ["External Beta", "Power Users"],
    changelog: "Beta version - please report bugs",
    distribute_external: true,
    notify_external_testers: true,
    
    # Demo account สำหรับ review
    demo_account_required: true,
    demo_account_name: "demo@company.com",
    demo_account_password: "DemoPassword123!"
  )
end
```

---

## 90.15 App Store Submission Checklist

### Pre-submission Checklist

```
ก่อน Submit ไป App Store:

Technical Requirements:
□ Build ผ่าน iPad และ iPhone simulators ทุก size
□ Test บน physical devices
□ Memory usage ไม่ excessive
□ App ไม่ crash ใน edge cases
□ Support latest iOS version (อย่างน้อย 2 versions ล่าสุด)
□ 64-bit support
□ No private APIs usage
□ App Transport Security compliance
□ Privacy manifest (PrivacyInfo.xcprivacy) ครบถ้วน

Code Quality:
□ No compiler warnings (อย่างน้อยใน Release build)
□ Static analyzer ผ่าน
□ Unit tests pass (coverage > 70%)
□ UI tests pass
□ Memory leaks ไม่มี (checked with Instruments)

App Store Requirements:
□ App name (30 chars max)
□ Subtitle (30 chars max)
□ Description (4000 chars max)
□ Keywords (100 chars max)
□ Support URL ใช้งานได้
□ Privacy Policy URL ถ้าแอปเก็บข้อมูล
□ Screenshots สำหรับทุก device size ที่รองรับ
□ App Preview video (optional)
□ Age rating กรอกครบ
□ Copyright information
□ Review notes (ถ้า app ต้องการ sign in)
```

### Fastlane Precheck

```ruby
# ตรวจสอบ metadata ก่อน submit
lane :precheck do
  precheck(
    app_identifier: "com.company.myapp",
    
    # ตรวจสอบสิ่งเหล่านี้
    default_rule_level: :error,
    
    # Custom rules
    include_in_app_purchases: true,
    
    # ตรวจสอบ
    negative_apple_sentiment: :warn,
    placeholder_text: :error,
    curse_words: :error,
    other_platforms: :warn
  )
end
```

---

## 90.16 Complete CI/CD Pipeline

### ตัวอย่าง Pipeline ครบวงจร

```ruby
# fastlane/Fastfile - Complete Pipeline

default_platform(:ios)

platform :ios do
  
  # ===== SETUP =====
  
  before_all do |lane|
    setup_circle_ci if is_ci
  end
  
  # ===== TESTING =====
  
  lane :lint do
    swiftlint(
      mode: :lint,
      config_file: ".swiftlint.yml",
      raise_if_swiftlint_error: true,
      reporter: "emoji"
    )
  end
  
  lane :unit_tests do
    run_tests(
      workspace: "MyApp.xcworkspace",
      scheme: "MyAppTests",
      devices: ["iPhone 15 Pro (17.2)"],
      code_coverage: true,
      output_directory: "./test_output/unit",
      output_files: "report.junit",
      xcargs: "-resultBundlePath ./test_output/unit/TestResults.xcresult"
    )
  end
  
  lane :ui_tests do
    run_tests(
      workspace: "MyApp.xcworkspace",
      scheme: "MyAppUITests",
      devices: ["iPhone 15 Pro (17.2)", "iPhone SE (3rd generation) (17.2)"],
      output_directory: "./test_output/ui"
    )
  end
  
  lane :test do
    lint
    unit_tests
    # ui_tests  # optional, ช้ากว่า
  end
  
  # ===== BUILDING =====
  
  lane :build_debug do
    gym(
      workspace: "MyApp.xcworkspace",
      scheme: "MyApp",
      configuration: "Debug",
      destination: "generic/platform=iOS Simulator"
    )
  end
  
  private_lane :build_for_distribution do |options|
    match(
      type: options[:export_method] == "app-store" ? "appstore" : "adhoc",
      readonly: is_ci,
      keychain_name: ENV["KEYCHAIN_NAME"],
      keychain_password: ENV["KEYCHAIN_PASSWORD"]
    )
    
    increment_build_number(
      build_number: options[:build_number] || latest_testflight_build_number + 1
    )
    
    gym(
      workspace: "MyApp.xcworkspace",
      scheme: options[:scheme] || "MyApp",
      configuration: options[:configuration] || "Release",
      export_method: options[:export_method] || "app-store",
      output_directory: "./build",
      output_name: "MyApp_#{options[:configuration] || 'Release'}"
    )
  end
  
  # ===== DISTRIBUTION =====
  
  lane :beta do |options|
    test unless options[:skip_tests]
    
    build_for_distribution(
      scheme: "MyApp",
      configuration: "Release",
      export_method: "app-store"
    )
    
    pilot(
      ipa: "./build/MyApp_Release.ipa",
      changelog: options[:notes] || git_branch_name_or_default,
      skip_waiting_for_build_processing: true,
      groups: options[:groups] || ["Internal"]
    )
    
    notify_slack(
      message: ":rocket: Beta #{get_version_number}.#{get_build_number} uploaded to TestFlight"
    )
  end
  
  lane :release do |options|
    UI.user_error!("Version number required") unless options[:version]
    
    ensure_git_status_clean
    ensure_git_branch(branch: "main")
    
    increment_version_number(version_number: options[:version])
    
    beta(skip_tests: false, notes: "Release candidate #{options[:version]}")
    
    deliver(
      submit_for_review: options[:submit] || false,
      automatic_release: false,
      force: true
    )
    
    git_commit(
      path: ["."],
      message: "Release version #{options[:version]}"
    )
    
    add_git_tag(tag: "v#{options[:version]}")
    push_to_git_remote
    
    notify_slack(
      message: ":tada: Version #{options[:version]} submitted to App Store!"
    )
  end
  
  # ===== UTILITIES =====
  
  private_lane :notify_slack do |options|
    slack(
      message: options[:message],
      channel: "#ios-releases",
      slack_url: ENV["SLACK_WEBHOOK_URL"]
    ) if ENV["SLACK_WEBHOOK_URL"]
  end
  
  private_lane :git_branch_name_or_default do
    `git rev-parse --abbrev-ref HEAD`.strip rescue "unknown"
  end
  
  # ===== ERROR HANDLING =====
  
  error do |lane, exception, options|
    notify_slack(
      message: ":x: Lane #{lane} failed: #{exception.message}"
    )
  end
  
  after_all do |lane|
    # Cleanup
    clean_build_artifacts if is_ci
  end
end
```

---

## 90.17 แบบฝึกหัด

### แบบฝึกหัดที่ 1: Setup xcconfig

```
งาน:
1. สร้าง xcconfig files สำหรับ 3 environments (Debug, Staging, Release)
2. กำหนด custom build settings:
   - API_BASE_URL
   - FEATURE_FLAGS (JSON string)
   - ANALYTICS_KEY
3. ใช้ settings เหล่านี้ใน Info.plist
4. อ่านค่าจาก Info.plist ใน code
5. ทดสอบว่าแต่ละ build configuration ใช้ค่าที่ถูกต้อง
```

### แบบฝึกหัดที่ 2: Fastlane Basic Setup

```ruby
# งาน: สร้าง Fastlane lanes สำหรับ:
# 1. run_tests - รัน unit tests
# 2. build_debug - build debug configuration
# 3. screenshot - capture screenshots (ถ้ามี snapshot tests)
# 4. upload_beta - upload ไป TestFlight (mockup ก็ได้)

# Hints:
# - ใช้ xcodebuild หรือ gym action
# - ใช้ run_tests action
# - ใช้ pilot action สำหรับ TestFlight

lane :my_test do
  run_tests(
    project: "MyApp.xcodeproj",  # หรือ workspace
    scheme: "MyApp",
    # เพิ่ม options ตามต้องการ
  )
end
```

### แบบฝึกหัดที่ 3: GitHub Actions CI

```yaml
# งาน: สร้าง GitHub Actions workflow ที่:
# 1. Build app เมื่อมี push ไป main branch
# 2. Run tests เมื่อมี pull request
# 3. Upload test results เป็น artifact
# 4. Post status comment บน PR

# Template:
name: iOS CI
on:
  # เพิ่ม triggers
  
jobs:
  test:
    runs-on: macos-14
    steps:
      # เพิ่ม steps
      - name: Checkout
        uses: actions/checkout@v4
      
      # Setup steps...
      
      # Test steps...
      
      # Upload results...
```

### แบบฝึกหัดที่ 4: Version Automation

```ruby
# งาน: สร้าง lane ที่:
# 1. อ่าน current version จาก Info.plist
# 2. Increment patch version
# 3. Update build number จาก git commit count
# 4. Commit changes
# 5. Create git tag

lane :version_bump_patch do
  # อ่าน current version
  current_version = get_version_number(target: "MyApp")
  parts = current_version.split(".").map(&:to_i)
  
  # Increment patch
  parts[2] += 1
  new_version = parts.join(".")
  
  # Update version
  increment_version_number(version_number: new_version)
  
  # Update build number
  build_number = sh("git rev-list --count HEAD").strip
  increment_build_number(build_number: build_number)
  
  # Commit
  git_commit(
    path: ["MyApp/Info.plist"],
    message: "chore: bump version to #{new_version} (#{build_number})"
  )
  
  UI.success "Version bumped to #{new_version} (build #{build_number})"
end
```

---

## สรุป

ในบทนี้เราได้เรียนรู้:

1. **Xcode Build System** - Build phases, rules, configurations
2. **Schemes, Targets, Configurations** - จัดการ build environments
3. **xcconfig Files** - การใช้ text files แทน GUI settings
4. **Code Signing** - Certificates, Provisioning Profiles
5. **Fastlane** - Automation ทั้ง build, test, deploy
6. **Fastlane Match** - Certificate/Profile management แบบ team
7. **Fastlane Gym** - Build automation
8. **Fastlane Deliver** - App Store upload
9. **GitHub Actions** - CI/CD บน cloud
10. **Xcode Cloud** - Apple's native CI/CD
11. **TestFlight** - Beta testing distribution
12. **App Store Connect API** - Programmatic access
13. **Versioning** - Semantic versioning, build numbers
14. **Beta Distribution** - ช่องทางต่างๆ
15. **App Store Checklist** - ก่อน submission

---

## แหล่งข้อมูลเพิ่มเติม

- [Fastlane Documentation](https://docs.fastlane.tools/)
- [GitHub Actions for iOS](https://docs.github.com/en/actions)
- [Xcode Cloud Documentation](https://developer.apple.com/documentation/xcode/xcode-cloud)
- [App Store Connect API](https://developer.apple.com/documentation/appstoreconnectapi)
- [TestFlight Guide](https://developer.apple.com/testflight/)

---

*บทต่อไป: ตอนที่ 91 - โปรเจกต์สมบูรณ์: Social Media App*
