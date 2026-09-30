# ส่วนที่ 96: Enterprise iOS Development

## บทนำ

Enterprise iOS Development คือการพัฒนาแอพสำหรับการใช้งานภายในองค์กร โดยมีข้อกำหนดพิเศษที่แตกต่างจากการทำแอพสำหรับ App Store เช่น การจัดการอุปกรณ์ด้วย MDM, การรักษาความปลอดภัยของข้อมูล, การ integrate กับระบบ IT ขององค์กร และการปฏิบัติตาม compliance ต่างๆ

---

## 96.1 Apple Developer Enterprise Program

### ความแตกต่างจาก Standard Program

**Apple Developer Enterprise Program:**
- ค่าสมัคร: $299 USD/ปี
- แจกจ่ายแอพได้เฉพาะพนักงานในองค์กร
- ห้ามใช้สำหรับ public distribution หรือ monetization
- ไม่ต้องผ่าน App Review
- ต้องมี D-U-N-S Number
- องค์กรต้องมีพนักงานอย่างน้อย 500 คน (Apple มีสิทธิ์ปฏิเสธองค์กรเล็กกว่า)

### วิธีสมัคร

```
1. ไปที่ developer.apple.com/programs/enterprise/
2. เลือก "Start your enrollment"
3. Sign in ด้วย Apple ID ขององค์กร
4. กรอกข้อมูลองค์กร:
   - Legal entity name
   - D-U-N-S Number
   - Address และ phone
   - Website URL
5. Apple จะ verify ข้อมูลองค์กร (อาจใช้เวลา 2-4 สัปดาห์)
6. ชำระค่าสมัคร
```

### Enterprise Distribution

```objc
// วิธีการแจกจ่ายแอพ Enterprise มี 3 แบบหลัก:

// 1. MDM (Mobile Device Management) - แนะนำ
// IT admin push แอพไปยังอุปกรณ์โดยตรงผ่าน MDM server
// ผู้ใช้ไม่ต้องทำอะไร

// 2. Managed App Distribution
// แจกจ่ายผ่าน internal app store หรือ URL
// ผู้ใช้ต้อง trust Enterprise certificate เอง
// Settings → General → VPN & Device Management → Trust

// 3. Ad Hoc Distribution (สูงสุด 100 อุปกรณ์)
// ใช้สำหรับ testing เท่านั้น
// ต้อง register UDID ทุกอุปกรณ์
```

---

## 96.2 MDM (Mobile Device Management)

### MDM คืออะไร

MDM เป็นระบบที่ให้ IT administrator จัดการอุปกรณ์ iOS จากส่วนกลาง สามารถ:
- ติดตั้ง/ถอนแอพโดยไม่ต้องให้ผู้ใช้ทำเอง
- ตั้งค่า WiFi, VPN, Email อัตโนมัติ
- บังคับ policy (เช่น ต้องใช้ passcode)
- ล้างข้อมูลอุปกรณ์จากระยะไกล (Remote Wipe)
- ดูข้อมูลอุปกรณ์ (battery, storage, installed apps)

### MDM Solutions ยอดนิยม

```
Commercial MDM:
- Jamf Pro (niche: Apple ecosystem)
- VMware Workspace ONE (เดิม AirWatch)
- Microsoft Intune
- Cisco Meraki
- Mosyle

Open Source MDM:
- MicroMDM
- NanoMDM
```

### MDM Enrollment

```
DEP (Device Enrollment Program) / Apple Business Manager:
- ซื้ออุปกรณ์ใหม่จาก Apple หรือ reseller ที่รับรอง
- อุปกรณ์ถูก enroll MDM อัตโนมัติเมื่อ setup
- ผู้ใช้ไม่สามารถ remove MDM profile ได้ (Supervised Device)

Manual Enrollment:
- ส่ง enrollment link หรือ QR code ให้ผู้ใช้
- ผู้ใช้ install MDM profile เอง
- ผู้ใช้สามารถ remove ได้
```

### การ Detect MDM Enrollment ในแอพ

```objc
// ตรวจสอบว่าอุปกรณ์ถูกจัดการโดย MDM
// (ไม่มี API โดยตรงจาก Apple แต่ใช้วิธีอ้อม)

- (BOOL)isDeviceManaged {
    // ตรวจสอบว่ามี managed configurations (ส่งมาจาก MDM)
    NSDictionary *managedConfig = [[NSUserDefaults standardUserDefaults] 
                                    dictionaryForKey:@"com.apple.configuration.managed"];
    return managedConfig != nil;
}

// การรับ Managed App Configuration จาก MDM
- (void)loadManagedConfiguration {
    NSDictionary *managedConfig = [[NSUserDefaults standardUserDefaults] 
                                    dictionaryForKey:@"com.apple.configuration.managed"];
    
    if (!managedConfig) {
        NSLog(@"No managed configuration found");
        return;
    }
    
    // อ่านค่าที่ MDM ส่งมา
    NSString *serverURL = managedConfig[@"ServerURL"];
    NSString *orgCode = managedConfig[@"OrganizationCode"];
    BOOL debugMode = [managedConfig[@"DebugMode"] boolValue];
    
    NSLog(@"Server URL: %@", serverURL);
    NSLog(@"Org Code: %@", orgCode);
    NSLog(@"Debug Mode: %@", debugMode ? @"YES" : @"NO");
    
    // Apply configuration
    [AppConfiguration sharedConfig].serverURL = serverURL;
    [AppConfiguration sharedConfig].organizationCode = orgCode;
}

// Listen for configuration changes
- (void)setupManagedConfigurationObserver {
    [[NSNotificationCenter defaultCenter] addObserver:self
                                             selector:@selector(managedConfigurationDidChange:)
                                                 name:NSUserDefaultsDidChangeNotification
                                               object:nil];
}

- (void)managedConfigurationDidChange:(NSNotification *)notification {
    [self loadManagedConfiguration];
}
```

### Managed App Feedback (ส่งข้อมูลกลับไปยัง MDM)

```objc
// ส่ง feedback กลับไปยัง MDM
- (void)sendManagedFeedback {
    NSUserDefaults *defaults = [NSUserDefaults standardUserDefaults];
    
    NSDictionary *feedback = @{
        @"AppVersion": [[NSBundle mainBundle] objectForInfoDictionaryKey:@"CFBundleShortVersionString"],
        @"LastSync": [[NSDate date] description],
        @"DataSize": @(self.calculateDataSize),
        @"UserCount": @(self.localUserCount)
    };
    
    [defaults setObject:feedback forKey:@"com.apple.feedback.managed"];
    [defaults synchronize];
}
```

---

## 96.3 Enterprise App Distribution

### การสร้าง Enterprise Build

```bash
# สร้าง Enterprise Distribution Profile
# 1. developer.apple.com → Certificates, IDs & Profiles
# 2. Profiles → + → In-House (Enterprise)
# 3. เลือก App ID และ Distribution Certificate
# 4. Download และ install

# Build และ Archive สำหรับ Enterprise
xcodebuild archive \
    -project MyEnterpriseApp.xcodeproj \
    -scheme MyEnterpriseApp \
    -configuration Release \
    -archivePath ./build/MyEnterpriseApp.xcarchive

# Export Enterprise IPA
xcodebuild -exportArchive \
    -archivePath ./build/MyEnterpriseApp.xcarchive \
    -exportPath ./build/enterprise \
    -exportOptionsPlist EnterpriseExportOptions.plist
```

```xml
<!-- EnterpriseExportOptions.plist -->
<?xml version="1.0" encoding="UTF-8"?>
<!DOCTYPE plist PUBLIC "-//Apple//DTD PLIST 1.0//EN" "http://www.apple.com/DTDs/PropertyList-1.0.dtd">
<plist version="1.0">
<dict>
    <key>method</key>
    <string>enterprise</string>
    <key>teamID</key>
    <string>YOURENTERPISETEAMID</string>
    <key>compileBitcode</key>
    <false/>
    <key>thinning</key>
    <string>&lt;none&gt;</string>
</dict>
</plist>
```

### Internal App Distribution Server

```objc
// manifest.plist สำหรับ OTA (Over-The-Air) installation
/*
<?xml version="1.0" encoding="UTF-8"?>
<!DOCTYPE plist PUBLIC "-//Apple//DTD PLIST 1.0//EN" "http://www.apple.com/DTDs/PropertyList-1.0.dtd">
<plist version="1.0">
<dict>
    <key>items</key>
    <array>
        <dict>
            <key>assets</key>
            <array>
                <dict>
                    <key>kind</key>
                    <string>software-package</string>
                    <key>url</key>
                    <string>https://internal.company.com/apps/MyApp.ipa</string>
                </dict>
                <dict>
                    <key>kind</key>
                    <string>display-image</string>
                    <key>url</key>
                    <string>https://internal.company.com/apps/icon-57x57.png</string>
                </dict>
            </array>
            <key>metadata</key>
            <dict>
                <key>bundle-identifier</key>
                <string>com.company.myapp</string>
                <key>bundle-version</key>
                <string>2.1.0</string>
                <key>kind</key>
                <string>software</string>
                <key>title</key>
                <string>My Enterprise App</string>
            </dict>
        </dict>
    </array>
</dict>
</plist>
*/

// ปุ่มสำหรับ install (ใน internal web portal)
// <a href="itms-services://?action=download-manifest&url=https://internal.company.com/apps/manifest.plist">
//     Install App
// </a>
```

---

## 96.4 App Thinning

App Thinning ช่วยลดขนาดแอพที่ download โดย deliver เฉพาะส่วนที่จำเป็นสำหรับแต่ละอุปกรณ์

### Slicing

```objc
// Xcode จัดการ Slicing อัตโนมัติ
// แต่ต้องใช้ Asset Catalog อย่างถูกต้อง

// ใน Asset Catalog (Assets.xcassets):
// - เพิ่มรูปสำหรับ @1x, @2x, @3x
// - ระบุ device types (iPhone, iPad, Apple Watch)
// - Xcode จะ slice ให้อัตโนมัติ

// ตรวจสอบว่าใช้ Asset Catalog แทน bundle
UIImage *image = [UIImage imageNamed:@"background"]; // ถูกต้อง - ใช้ Asset Catalog
// UIImage *image = [UIImage imageWithContentsOfFile:[[NSBundle mainBundle] pathForResource:@"background-2x" ofType:@"png"]]; // ผิด - hardcode resolution
```

### Bitcode

```objc
// Bitcode ถูก deprecate ใน Xcode 14
// สำหรับ Xcode 13 และเก่ากว่า:
// Build Settings → Enable Bitcode → YES (สำหรับ watchOS, tvOS)
// iOS: optional แต่ Apple แนะนำ

// Bitcode ช่วยให้ Apple recompile แอพสำหรับ future processors
// โดยไม่ต้อง resubmit
```

---

## 96.5 On-Demand Resources (ODR)

ODR ให้แอพ download content เพิ่มเติมเมื่อต้องการ ลดขนาด initial download

```objc
// กำหนด tags ใน Asset Catalog หรือ Build Settings
// Assets.xcassets → Asset → Attributes → On Demand Resource Tag

// การ request ODR
NSBundleResourceRequest *resourceRequest = [[NSBundleResourceRequest alloc] 
    initWithTags:[NSSet setWithObject:@"Level5Assets"]];

// Set loading priority
resourceRequest.loadingPriority = NSBundleResourceRequestLoadingPriorityUrgent;

[resourceRequest beginAccessingResourcesWithCompletionHandler:^(NSError *error) {
    if (error) {
        NSLog(@"Error loading resources: %@", error);
        return;
    }
    
    // ทรัพยากรพร้อมใช้งานแล้ว
    dispatch_async(dispatch_get_main_queue(), ^{
        [self loadLevel5WithResources];
    });
}];

// เมื่อเสร็จใช้งาน release resources
- (void)level5Finished {
    [self.resourceRequest endAccessingResources];
    self.resourceRequest = nil;
}

// ตรวจสอบว่า resources พร้อมหรือเปล่า
- (void)conditionallyLoadResources {
    NSBundleResourceRequest *request = [[NSBundleResourceRequest alloc] 
        initWithTags:[NSSet setWithObject:@"PremiumContent"]];
    
    [request conditionallyBeginAccessingResourcesWithCompletionHandler:^(BOOL resourcesAvailable) {
        if (resourcesAvailable) {
            // Resources อยู่ใน device แล้ว
            dispatch_async(dispatch_get_main_queue(), ^{
                [self showPremiumContent];
            });
        } else {
            // ต้อง download
            [self downloadAndShowPremiumContent];
        }
    }];
}
```

---

## 96.6 VPN และ Network Configurations

### การตั้งค่า VPN ผ่าน MDM Configuration Profile

```xml
<!-- VPN Configuration Profile (ส่งผ่าน MDM) -->
<dict>
    <key>PayloadType</key>
    <string>com.apple.vpn.managed</string>
    <key>PayloadIdentifier</key>
    <string>com.company.vpn.configuration</string>
    <key>UserDefinedName</key>
    <string>Corporate VPN</string>
    <key>VPNType</key>
    <string>IKEv2</string>
    <key>IKEv2</key>
    <dict>
        <key>AuthenticationMethod</key>
        <string>Certificate</string>
        <key>RemoteAddress</key>
        <string>vpn.company.com</string>
        <key>LocalIdentifier</key>
        <string>com.company.device</string>
        <key>RemoteIdentifier</key>
        <string>vpn.company.com</string>
        <key>UseConfigurationAttributeInternalIPSubnet</key>
        <integer>0</integer>
    </dict>
</dict>
```

### Per-App VPN

```objc
// Per-App VPN ทำให้เฉพาะแอพที่กำหนดใช้ VPN
// ต้องตั้งค่าผ่าน MDM profile

// ตรวจสอบสถานะ VPN ในแอพ
#import <NetworkExtension/NetworkExtension.h>

- (void)checkVPNStatus {
    NEVPNManager *vpnManager = [NEVPNManager sharedManager];
    [vpnManager loadFromPreferencesWithCompletionHandler:^(NSError *error) {
        if (error) {
            NSLog(@"Error loading VPN prefs: %@", error);
            return;
        }
        
        NEVPNStatus status = vpnManager.connection.status;
        switch (status) {
            case NEVPNStatusConnected:
                NSLog(@"VPN Connected");
                break;
            case NEVPNStatusDisconnected:
                NSLog(@"VPN Disconnected");
                break;
            case NEVPNStatusConnecting:
                NSLog(@"VPN Connecting...");
                break;
            default:
                break;
        }
    }];
}
```

### Network Configuration Profile

```xml
<!-- WiFi Configuration ส่งผ่าน MDM -->
<dict>
    <key>PayloadType</key>
    <string>com.apple.wifi.managed</string>
    <key>SSID_STR</key>
    <string>Corporate-WiFi</string>
    <key>EncryptionType</key>
    <string>WPA2</string>
    <key>EAPClientConfiguration</key>
    <dict>
        <key>AcceptEAPTypes</key>
        <array>
            <integer>13</integer>  <!-- EAP-TLS -->
            <integer>25</integer>  <!-- PEAP -->
        </array>
        <key>UserName</key>
        <string>%email%</string>  <!-- MDM substitution variable -->
        <key>PayloadCertificateAnchorUUID</key>
        <array>
            <string>root-ca-uuid-here</string>
        </array>
    </dict>
</dict>
```

---

## 96.7 Corporate SSO และ Authentication

### SAML-based SSO

```objc
// ใช้ ASWebAuthenticationSession สำหรับ SSO
#import <AuthenticationServices/AuthenticationServices.h>

- (void)startSSOAuthentication {
    // สร้าง SSO URL
    NSURLComponents *components = [[NSURLComponents alloc] initWithString:@"https://sso.company.com/saml/auth"];
    components.queryItems = @[
        [NSURLQueryItem queryItemWithName:@"SAMLRequest" value:[self generateSAMLRequest]],
        [NSURLQueryItem queryItemWithName:@"RelayState" value:@"myapp"]
    ];
    
    NSURL *authURL = components.URL;
    NSString *callbackScheme = @"myapp";
    
    ASWebAuthenticationSession *session = [[ASWebAuthenticationSession alloc]
        initWithURL:authURL
        callbackURLScheme:callbackScheme
        completionHandler:^(NSURL *callbackURL, NSError *error) {
            if (error) {
                if (error.code != ASWebAuthenticationSessionErrorCodeCanceledLogin) {
                    [self handleSSOError:error];
                }
                return;
            }
            
            // Parse SAML response from callback URL
            [self handleSSOCallback:callbackURL];
        }];
    
    session.presentationContextProvider = self;
    session.prefersEphemeralWebBrowserSession = YES; // ไม่แชร์ cookies
    [session start];
}

// ASWebAuthenticationPresentationContextProviding
- (ASPresentationAnchor)presentationAnchorForWebAuthenticationSession:(ASWebAuthenticationSession *)session {
    return self.view.window;
}

- (void)handleSSOCallback:(NSURL *)url {
    // Parse SAMLResponse
    NSURLComponents *components = [NSURLComponents componentsWithURL:url resolvingAgainstBaseURL:NO];
    NSString *samlResponse = nil;
    
    for (NSURLQueryItem *item in components.queryItems) {
        if ([item.name isEqualToString:@"SAMLResponse"]) {
            samlResponse = item.value;
            break;
        }
    }
    
    if (samlResponse) {
        // Decode และ validate SAML assertion
        [self validateSAMLResponse:samlResponse];
    }
}
```

### OAuth 2.0 / OpenID Connect

```objc
// ใช้ AppAuth library สำหรับ OAuth 2.0
// pod 'AppAuth'

#import <AppAuth/AppAuth.h>

@interface AuthManager : NSObject

@property (nonatomic, strong) id<OIDExternalUserAgentSession> currentAuthorizationFlow;

- (void)authenticateWithConfiguration:(OIDServiceConfiguration *)config 
                           completion:(void(^)(OIDAuthState *authState, NSError *error))completion;

@end

@implementation AuthManager

- (void)authenticateWithConfiguration:(OIDServiceConfiguration *)config 
                           completion:(void(^)(OIDAuthState *authState, NSError *error))completion {
    OIDAuthorizationRequest *request = [[OIDAuthorizationRequest alloc] 
        initWithConfiguration:config
                     clientId:@"your-client-id"
                       scopes:@[OIDScopeOpenID, OIDScopeProfile, OIDScopeEmail]
                  redirectURL:[NSURL URLWithString:@"com.company.myapp://oauth/callback"]
                 responseType:OIDResponseTypeCode
         additionalParameters:@{@"login_hint": @"employee"}];
    
    UIViewController *presentingVC = [self topViewController];
    
    self.currentAuthorizationFlow = [OIDAuthState 
        authStateByPresentingAuthorizationRequest:request
        presentingViewController:presentingVC
        callback:^(OIDAuthState *authState, NSError *error) {
            if (authState) {
                NSLog(@"Got tokens: %@", authState.lastTokenResponse.accessToken);
                // Store securely in Keychain
                [self storeAuthState:authState];
                completion(authState, nil);
            } else {
                completion(nil, error);
            }
        }];
}

- (void)refreshTokenIfNeeded:(OIDAuthState *)authState 
                  completion:(void(^)(NSString *accessToken, NSError *error))completion {
    [authState performActionWithFreshTokens:^(NSString *accessToken, NSString *idToken, NSError *error) {
        if (error) {
            completion(nil, error);
            return;
        }
        completion(accessToken, nil);
    }];
}

@end
```

---

## 96.8 LDAP/Active Directory Integration

```objc
// การ Authenticate กับ LDAP ผ่าน API
// โดยปกติจะไม่ connect LDAP โดยตรงจาก iOS แต่จะผ่าน backend API

@interface LDAPAuthService : NSObject

- (void)authenticateUser:(NSString *)username 
                password:(NSString *)password 
              completion:(void(^)(LDAPUser *user, NSError *error))completion;

- (void)fetchUserGroups:(NSString *)username 
              accessToken:(NSString *)token 
              completion:(void(^)(NSArray<NSString *> *groups, NSError *error))completion;

@end

@implementation LDAPAuthService

- (void)authenticateUser:(NSString *)username 
                password:(NSString *)password 
              completion:(void(^)(LDAPUser *user, NSError *error))completion {
    
    // ส่ง credentials ไปยัง backend API ที่ connect กับ LDAP
    NSURL *url = [NSURL URLWithString:@"https://api.company.com/auth/ldap"];
    NSMutableURLRequest *request = [NSMutableURLRequest requestWithURL:url];
    request.HTTPMethod = @"POST";
    [request setValue:@"application/json" forHTTPHeaderField:@"Content-Type"];
    
    NSDictionary *body = @{
        @"username": username,
        @"password": password,
        @"domain": @"company.local"
    };
    
    NSError *jsonError;
    request.HTTPBody = [NSJSONSerialization dataWithJSONObject:body options:0 error:&jsonError];
    
    NSURLSessionDataTask *task = [[NSURLSession sharedSession] 
        dataTaskWithRequest:request 
        completionHandler:^(NSData *data, NSURLResponse *response, NSError *error) {
            if (error) {
                dispatch_async(dispatch_get_main_queue(), ^{
                    completion(nil, error);
                });
                return;
            }
            
            NSHTTPURLResponse *httpResponse = (NSHTTPURLResponse *)response;
            if (httpResponse.statusCode == 200) {
                NSDictionary *json = [NSJSONSerialization JSONObjectWithData:data options:0 error:nil];
                LDAPUser *user = [LDAPUser userFromDictionary:json];
                
                // เก็บ token ใน Keychain
                [KeychainManager saveToken:json[@"access_token"] forKey:@"ldap_token"];
                
                dispatch_async(dispatch_get_main_queue(), ^{
                    completion(user, nil);
                });
            } else {
                NSError *authError = [NSError errorWithDomain:@"LDAPAuthError"
                                                         code:httpResponse.statusCode
                                                     userInfo:@{NSLocalizedDescriptionKey: @"Authentication failed"}];
                dispatch_async(dispatch_get_main_queue(), ^{
                    completion(nil, authError);
                });
            }
        }];
    [task resume];
}

@end

// ตรวจสอบ user permissions ตาม AD Groups
- (void)checkPermissionsForUser:(LDAPUser *)user {
    if ([user.groups containsObject:@"iOS-Admin"]) {
        [self enableAdminFeatures];
    } else if ([user.groups containsObject:@"iOS-Manager"]) {
        [self enableManagerFeatures];
    } else {
        [self enableStandardFeatures];
    }
}
```

---

## 96.9 Certificate-based Authentication

```objc
// Client Certificate Authentication
#import <Security/Security.h>

@interface CertificateAuthManager : NSObject

- (void)importClientCertificate:(NSData *)p12Data 
                       password:(NSString *)password 
                     completion:(void(^)(BOOL success, NSError *error))completion;

- (void)makeAuthenticatedRequest:(NSURLRequest *)request 
                      completion:(void(^)(NSData *data, NSError *error))completion;

@end

@implementation CertificateAuthManager {
    SecIdentityRef _clientIdentity;
}

- (void)importClientCertificate:(NSData *)p12Data 
                       password:(NSString *)password 
                     completion:(void(^)(BOOL success, NSError *error))completion {
    
    NSDictionary *options = @{
        (id)kSecImportExportPassphrase: password
    };
    
    CFArrayRef items = NULL;
    OSStatus status = SecPKCS12Import((__bridge CFDataRef)p12Data, 
                                       (__bridge CFDictionaryRef)options, 
                                       &items);
    
    if (status == errSecSuccess && CFArrayGetCount(items) > 0) {
        CFDictionaryRef item = CFArrayGetValueAtIndex(items, 0);
        SecIdentityRef identity = (SecIdentityRef)CFDictionaryGetValue(item, kSecImportItemIdentity);
        
        if (identity) {
            _clientIdentity = (SecIdentityRef)CFRetain(identity);
            
            // เก็บใน Keychain
            NSDictionary *addQuery = @{
                (id)kSecClass: (id)kSecClassIdentity,
                (id)kSecValueRef: (__bridge id)_clientIdentity,
                (id)kSecAttrAccessible: (id)kSecAttrAccessibleWhenUnlockedThisDeviceOnly
            };
            
            SecItemAdd((__bridge CFDictionaryRef)addQuery, NULL);
            
            if (items) CFRelease(items);
            completion(YES, nil);
        }
    } else {
        if (items) CFRelease(items);
        NSError *error = [NSError errorWithDomain:NSOSStatusErrorDomain 
                                             code:status 
                                         userInfo:nil];
        completion(NO, error);
    }
}

// NSURLSessionDelegate สำหรับ Client Certificate
- (void)URLSession:(NSURLSession *)session 
didReceiveChallenge:(NSURLAuthenticationChallenge *)challenge 
 completionHandler:(void (^)(NSURLSessionAuthChallengeDisposition, NSURLCredential *))completionHandler {
    
    if ([challenge.protectionSpace.authenticationMethod isEqualToString:NSURLAuthenticationMethodClientCertificate]) {
        if (_clientIdentity) {
            // ดึง certificate จาก identity
            SecCertificateRef certificate = NULL;
            SecIdentityCopyCertificate(_clientIdentity, &certificate);
            
            NSArray *certificates = @[(__bridge id)certificate];
            NSURLCredential *credential = [NSURLCredential credentialWithIdentity:_clientIdentity
                                                                     certificates:certificates
                                                                      persistence:NSURLCredentialPersistenceForSession];
            
            if (certificate) CFRelease(certificate);
            completionHandler(NSURLSessionAuthChallengeUseCredential, credential);
        } else {
            completionHandler(NSURLSessionAuthChallengeCancelAuthenticationChallenge, nil);
        }
    } else if ([challenge.protectionSpace.authenticationMethod isEqualToString:NSURLAuthenticationMethodServerTrust]) {
        // Server Certificate Validation
        [self validateServerTrust:challenge.protectionSpace.serverTrust 
                   completionHandler:completionHandler];
    } else {
        completionHandler(NSURLSessionAuthChallengePerformDefaultHandling, nil);
    }
}

- (void)validateServerTrust:(SecTrustRef)serverTrust 
          completionHandler:(void(^)(NSURLSessionAuthChallengeDisposition, NSURLCredential *))completionHandler {
    // Certificate Pinning
    NSString *certPath = [[NSBundle mainBundle] pathForResource:@"server_cert" ofType:@"cer"];
    NSData *certData = [NSData dataWithContentsOfFile:certPath];
    SecCertificateRef pinnedCert = SecCertificateCreateWithData(NULL, (__bridge CFDataRef)certData);
    
    SecTrustSetAnchorCertificates(serverTrust, (__bridge CFArrayRef)@[(__bridge id)pinnedCert]);
    
    SecTrustResultType result;
    SecTrustEvaluate(serverTrust, &result);
    
    if (result == kSecTrustResultUnspecified || result == kSecTrustResultProceed) {
        NSURLCredential *credential = [NSURLCredential credentialForTrust:serverTrust];
        completionHandler(NSURLSessionAuthChallengeUseCredential, credential);
    } else {
        completionHandler(NSURLSessionAuthChallengeCancelAuthenticationChallenge, nil);
    }
    
    if (pinnedCert) CFRelease(pinnedCert);
}

@end
```

---

## 96.10 Data Security และ Compliance

### HIPAA Compliance สำหรับ Healthcare Apps

```objc
// HIPAA ต้องการ:
// 1. Data Encryption at rest
// 2. Data Encryption in transit
// 3. Access control
// 4. Audit logging
// 5. Auto-logout

// 1. Encryption at Rest - ใช้ NSFileProtectionComplete
- (void)saveSecureData:(NSData *)data toFile:(NSString *)filename {
    NSString *documentsPath = [NSSearchPathForDirectoriesInDomains(NSDocumentDirectory, 
                                                                     NSUserDomainMask, YES) firstObject];
    NSString *filePath = [documentsPath stringByAppendingPathComponent:filename];
    
    NSDictionary *attributes = @{
        NSFileProtectionKey: NSFileProtectionComplete
        // NSFileProtectionComplete: file ถูกเข้ารหัส เข้าถึงได้เฉพาะตอน device unlock
        // NSFileProtectionCompleteUnlessOpen: เข้าถึงได้แม้ device lock ถ้า file เปิดอยู่แล้ว
        // NSFileProtectionCompleteUntilFirstUserAuthentication: เข้าถึงได้หลัง first unlock
    };
    
    [data writeToFile:filePath options:NSDataWritingAtomic error:nil];
    [[NSFileManager defaultManager] setAttributes:attributes ofItemAtPath:filePath error:nil];
}

// 2. Keychain สำหรับ sensitive data
- (void)savePatientID:(NSString *)patientID {
    NSData *data = [patientID dataUsingEncoding:NSUTF8StringEncoding];
    
    NSDictionary *query = @{
        (id)kSecClass: (id)kSecClassGenericPassword,
        (id)kSecAttrService: @"com.company.healthcare",
        (id)kSecAttrAccount: @"patientID",
        (id)kSecValueData: data,
        (id)kSecAttrAccessible: (id)kSecAttrAccessibleWhenPasscodeSetThisDeviceOnly
        // kSecAttrAccessibleWhenPasscodeSetThisDeviceOnly: 
        // เข้าถึงได้เฉพาะเมื่อ device มี passcode และไม่ migrate ระหว่าง devices
    };
    
    SecItemDelete((__bridge CFDictionaryRef)query);
    SecItemAdd((__bridge CFDictionaryRef)query, NULL);
}

// 3. Auto-logout
@interface AutoLogoutManager : NSObject

@property (nonatomic, assign) NSTimeInterval timeoutInterval; // default 15 minutes for HIPAA
@property (nonatomic, copy) void (^logoutHandler)(void);

- (void)resetTimer;
- (void)invalidate;

@end

@implementation AutoLogoutManager {
    NSTimer *_timer;
}

- (instancetype)init {
    self = [super init];
    if (self) {
        _timeoutInterval = 15 * 60; // 15 minutes
        [self setupUserActivityTracking];
    }
    return self;
}

- (void)setupUserActivityTracking {
    // Track user interactions
    [[NSNotificationCenter defaultCenter] addObserver:self
                                             selector:@selector(userActivity:)
                                                 name:@"UserActivityDetected"
                                               object:nil];
    [self resetTimer];
}

- (void)resetTimer {
    [_timer invalidate];
    _timer = [NSTimer scheduledTimerWithTimeInterval:self.timeoutInterval
                                              target:self
                                            selector:@selector(timerExpired)
                                            userInfo:nil
                                             repeats:NO];
}

- (void)timerExpired {
    if (self.logoutHandler) {
        dispatch_async(dispatch_get_main_queue(), ^{
            self.logoutHandler();
        });
    }
}

- (void)invalidate {
    [_timer invalidate];
    _timer = nil;
}

@end

// 4. Audit Log
@interface AuditLogger : NSObject

+ (void)logEvent:(NSString *)eventType 
          userId:(NSString *)userId 
          details:(NSDictionary *)details;

@end

@implementation AuditLogger

+ (void)logEvent:(NSString *)eventType 
          userId:(NSString *)userId 
          details:(NSDictionary *)details {
    
    NSDictionary *logEntry = @{
        @"timestamp": [[NSDate date] description],
        @"eventType": eventType,
        @"userId": userId ?: @"anonymous",
        @"deviceId": [UIDevice currentDevice].identifierForVendor.UUIDString,
        @"appVersion": [[NSBundle mainBundle] objectForInfoDictionaryKey:@"CFBundleShortVersionString"],
        @"details": details ?: @{}
    };
    
    // ส่งไปยัง audit log server
    [self sendAuditLog:logEntry];
    
    // เก็บ local copy (encrypted)
    [self saveLocalAuditLog:logEntry];
}

+ (void)sendAuditLog:(NSDictionary *)log {
    // ส่งผ่าน secure HTTPS endpoint
    NSURL *url = [NSURL URLWithString:@"https://api.company.com/audit-logs"];
    NSMutableURLRequest *request = [NSMutableURLRequest requestWithURL:url];
    request.HTTPMethod = @"POST";
    [request setValue:@"application/json" forHTTPHeaderField:@"Content-Type"];
    [request setValue:[self authorizationHeader] forHTTPHeaderField:@"Authorization"];
    request.HTTPBody = [NSJSONSerialization dataWithJSONObject:log options:0 error:nil];
    
    [[NSURLSession sharedSession] dataTaskWithRequest:request completionHandler:^(NSData *data, NSURLResponse *response, NSError *error) {
        if (error) {
            NSLog(@"Audit log failed: %@", error);
            // Queue for retry
        }
    }];
}

@end
```

### GDPR Compliance

```objc
// GDPR Requirements:
// 1. Explicit consent ก่อนเก็บข้อมูล
// 2. Right to access ข้อมูลของตัวเอง
// 3. Right to erasure (Right to be forgotten)
// 4. Data portability
// 5. Privacy by design

// ConsentManager
@interface ConsentManager : NSObject

+ (BOOL)hasUserConsented;
+ (void)setUserConsent:(BOOL)consented;
+ (NSDate *)consentDate;
+ (void)clearConsent;
+ (void)exportUserData:(NSString *)userId completion:(void(^)(NSDictionary *data))completion;
+ (void)deleteUserData:(NSString *)userId completion:(void(^)(BOOL success))completion;

@end

@implementation ConsentManager

static NSString * const kConsentKey = @"GDPR_UserConsent";
static NSString * const kConsentDateKey = @"GDPR_ConsentDate";

+ (BOOL)hasUserConsented {
    return [[NSUserDefaults standardUserDefaults] boolForKey:kConsentKey];
}

+ (void)setUserConsent:(BOOL)consented {
    NSUserDefaults *defaults = [NSUserDefaults standardUserDefaults];
    [defaults setBool:consented forKey:kConsentKey];
    [defaults setObject:[NSDate date] forKey:kConsentDateKey];
    [defaults synchronize];
    
    if (!consented) {
        // หยุดการเก็บ analytics ทันที
        [AnalyticsManager disableTracking];
    }
}

+ (void)exportUserData:(NSString *)userId completion:(void(^)(NSDictionary *))completion {
    // ดึงข้อมูลทั้งหมดของ user สำหรับ GDPR Data Access Request
    dispatch_group_t group = dispatch_group_create();
    NSMutableDictionary *allData = [NSMutableDictionary dictionary];
    
    dispatch_group_enter(group);
    [self fetchUserProfileData:userId completion:^(NSDictionary *data) {
        allData[@"profile"] = data;
        dispatch_group_leave(group);
    }];
    
    dispatch_group_enter(group);
    [self fetchUserActivityData:userId completion:^(NSArray *data) {
        allData[@"activity"] = data;
        dispatch_group_leave(group);
    }];
    
    dispatch_group_notify(group, dispatch_get_main_queue(), ^{
        completion([allData copy]);
    });
}

+ (void)deleteUserData:(NSString *)userId completion:(void(^)(BOOL))completion {
    // ลบข้อมูลทั้งหมดของ user (GDPR Right to Erasure)
    [[NSURLSession sharedSession] dataTaskWithRequest:[self deleteRequestForUser:userId] 
                                   completionHandler:^(NSData *data, NSURLResponse *response, NSError *error) {
        BOOL success = !error && [(NSHTTPURLResponse *)response statusCode] == 200;
        
        if (success) {
            // ล้าง local data ด้วย
            [NSUserDefaults standardUserDefaults];
            // ล้าง Keychain
            [KeychainManager clearAllForUser:userId];
            // ล้าง Core Data
            [CoreDataManager deleteAllForUser:userId];
        }
        
        dispatch_async(dispatch_get_main_queue(), ^{
            completion(success);
        });
    }];
}

@end
```

---

## 96.11 App Wrapping vs MAM

### App Wrapping

App Wrapping คือการ wrap existing IPA ด้วย MDM SDK โดยไม่ต้องเปลี่ยน source code

```
เครื่องมือที่ใช้:
- Intune App Wrapping Tool
- Jamf App Wrapping
- BlackBerry Dynamics SDK

ข้อดี:
- ไม่ต้องเปลี่ยน source code
- ทำได้กับ third-party apps

ข้อเสีย:
- ไม่สามารถ customize พฤติกรรมได้มาก
- อาจทำให้ app ทำงานผิดปกติ
- ไม่รองรับ Swift apps บางกรณี
```

### MAM (Mobile Application Management) SDK

```objc
// ตัวอย่างการ integrate Microsoft Intune SDK
// pod 'MSAL'

#import <MSAL/MSAL.h>
#import <IntuneMAM/IntuneMAM.h>

@interface AppDelegate : UIResponder <UIApplicationDelegate, IntuneMAMPolicyDelegate>

@end

@implementation AppDelegate

- (BOOL)application:(UIApplication *)application 
    didFinishLaunchingWithOptions:(NSDictionary *)launchOptions {
    
    // Initialize Intune MAM
    IntuneMAMPolicyManager *policyManager = [IntuneMAMPolicyManager instance];
    policyManager.delegate = self;
    
    return YES;
}

// IntuneMAMPolicyDelegate
- (void)identitySwitchRequired:(NSString *)upn 
                        reason:(IntuneMAMIdentitySwitchReason)reason 
              completionHandler:(void (^)(IntuneMAMAddIdentityResult))completionHandler {
    // Handle identity switch
    completionHandler(IntuneMAMAddIdentityResultSuccess);
}

// ตรวจสอบ policy ก่อนทำ action
- (void)copyDataToClipboard:(NSString *)data {
    IntuneMAMPolicy *policy = [[IntuneMAMPolicyManager instance] policy];
    
    if ([policy isCutCopyToClipboardBlocked]) {
        [self showAlert:@"การ copy ถูกปิดใช้งานโดยนโยบายขององค์กร"];
        return;
    }
    
    [[UIPasteboard generalPasteboard] setString:data];
}

@end
```

---

## 96.12 Crash Reporting ด้วย Firebase Crashlytics

```objc
// ติดตั้ง
// pod 'Firebase/Crashlytics'
// pod 'Firebase/Analytics'

// AppDelegate.m
#import <Firebase/Firebase.h>

- (BOOL)application:(UIApplication *)application 
    didFinishLaunchingWithOptions:(NSDictionary *)launchOptions {
    
    [FIRApp configure];
    
    // Enable crash reporting
    [[FIRCrashlytics crashlytics] setCrashlyticsCollectionEnabled:YES];
    
    return YES;
}

// การใช้งาน Crashlytics
@implementation CrashReporter

// Log ข้อมูล user สำหรับ debugging (ไม่ใส่ข้อมูลส่วนตัว)
+ (void)setUserIdentifier:(NSString *)userId {
    [[FIRCrashlytics crashlytics] setUserID:userId];
}

// Log custom key-value pairs
+ (void)setCustomKeys {
    [[FIRCrashlytics crashlytics] setCustomValue:@"premium" forKey:@"user_type"];
    [[FIRCrashlytics crashlytics] setCustomValue:@"v2.1.0" forKey:@"api_version"];
}

// Log ข้อความ
+ (void)logMessage:(NSString *)message {
    [[FIRCrashlytics crashlytics] log:message];
}

// Record non-fatal errors
+ (void)recordError:(NSError *)error context:(NSString *)context {
    [[FIRCrashlytics crashlytics] log:[NSString stringWithFormat:@"Context: %@", context]];
    [[FIRCrashlytics crashlytics] recordError:error];
}

// Force crash สำหรับ testing (ห้ามใส่ใน production!)
#ifdef DEBUG
+ (void)testCrash {
    @[][1]; // Index out of bounds
}
#endif

@end

// การใช้งานใน code จริง
- (void)fetchUserData:(NSString *)userId {
    [[FIRCrashlytics crashlytics] log:@"Fetching user data"];
    
    [self.service fetchUser:userId completion:^(User *user, NSError *error) {
        if (error) {
            [[FIRCrashlytics crashlytics] log:
                [NSString stringWithFormat:@"Failed to fetch user %@", userId]];
            [CrashReporter recordError:error context:@"fetchUserData"];
            return;
        }
        
        [[FIRCrashlytics crashlytics] log:@"User data fetched successfully"];
        [self updateUIWithUser:user];
    }];
}
```

---

## 96.13 Analytics Integration

```objc
// Firebase Analytics
#import <FirebaseAnalytics/FirebaseAnalytics.h>

@interface AnalyticsManager : NSObject

+ (void)logEvent:(NSString *)eventName parameters:(NSDictionary *)parameters;
+ (void)setUserProperty:(NSString *)value forName:(NSString *)name;
+ (void)setUserId:(NSString *)userId;

@end

@implementation AnalyticsManager

+ (void)logEvent:(NSString *)eventName parameters:(NSDictionary *)parameters {
    [FIRAnalytics logEventWithName:eventName parameters:parameters];
}

+ (void)setUserProperty:(NSString *)value forName:(NSString *)name {
    [FIRAnalytics setUserPropertyString:value forName:name];
}

+ (void)setUserId:(NSString *)userId {
    [FIRAnalytics setUserID:userId];
}

@end

// Constants สำหรับ event names
static NSString * const kEventLoginSuccess = @"login_success";
static NSString * const kEventLoginFailed = @"login_failed";
static NSString * const kEventProjectCreated = @"project_created";
static NSString * const kEventFeatureUsed = @"feature_used";

// การใช้งาน
- (void)userDidLogin:(User *)user {
    [AnalyticsManager logEvent:kEventLoginSuccess parameters:@{
        @"user_type": user.isAdmin ? @"admin" : @"standard",
        @"login_method": @"password"
    }];
    
    [AnalyticsManager setUserId:user.userId];
    [AnalyticsManager setUserProperty:user.department forName:@"department"];
}

- (void)userDidCreateProject:(Project *)project {
    [AnalyticsManager logEvent:kEventProjectCreated parameters:@{
        @"project_type": project.type,
        @"member_count": @(project.members.count),
        @"has_deadline": project.deadline ? @YES : @NO
    }];
}
```

---

## 96.14 A/B Testing

```objc
// Firebase Remote Config สำหรับ A/B Testing
#import <FirebaseRemoteConfig/FirebaseRemoteConfig.h>

@interface ABTestManager : NSObject

+ (instancetype)sharedManager;
- (void)fetchConfigWithCompletion:(void(^)(void))completion;
- (NSString *)stringValueForKey:(NSString *)key;
- (BOOL)boolValueForKey:(NSString *)key;

@end

@implementation ABTestManager {
    FIRRemoteConfig *_remoteConfig;
}

+ (instancetype)sharedManager {
    static ABTestManager *instance = nil;
    static dispatch_once_t onceToken;
    dispatch_once(&onceToken, ^{
        instance = [[ABTestManager alloc] init];
    });
    return instance;
}

- (instancetype)init {
    self = [super init];
    if (self) {
        _remoteConfig = [FIRRemoteConfig remoteConfig];
        
        // Set defaults
        [_remoteConfig setDefaults:@{
            @"onboarding_variant": @"control",
            @"show_new_dashboard": @NO,
            @"checkout_button_color": @"blue",
            @"max_items_in_cart": @10
        }];
        
        // Set fetch interval
        FIRRemoteConfigSettings *settings = [[FIRRemoteConfigSettings alloc] init];
        settings.minimumFetchInterval = 3600; // 1 hour
        _remoteConfig.configSettings = settings;
    }
    return self;
}

- (void)fetchConfigWithCompletion:(void(^)(void))completion {
    [_remoteConfig fetchAndActivateWithCompletionHandler:^(FIRRemoteConfigFetchAndActivateStatus status, NSError *error) {
        if (error) {
            NSLog(@"Remote config fetch failed: %@", error);
        }
        dispatch_async(dispatch_get_main_queue(), ^{
            completion();
        });
    }];
}

- (NSString *)stringValueForKey:(NSString *)key {
    return [_remoteConfig configValueForKey:key].stringValue;
}

- (BOOL)boolValueForKey:(NSString *)key {
    return [_remoteConfig configValueForKey:key].boolValue;
}

@end

// การใช้งาน A/B Test
- (void)setupOnboarding {
    NSString *variant = [[ABTestManager sharedManager] stringValueForKey:@"onboarding_variant"];
    
    if ([variant isEqualToString:@"variant_a"]) {
        [self showVideoOnboarding];
    } else if ([variant isEqualToString:@"variant_b"]) {
        [self showInteractiveOnboarding];
    } else {
        [self showStandardOnboarding]; // control
    }
    
    // Track which variant was shown
    [AnalyticsManager logEvent:@"onboarding_started" parameters:@{
        @"variant": variant
    }];
}

- (void)setupNewDashboard {
    BOOL showNewDashboard = [[ABTestManager sharedManager] boolValueForKey:@"show_new_dashboard"];
    
    if (showNewDashboard) {
        [self loadNewDashboardViewController];
    } else {
        [self loadClassicDashboardViewController];
    }
}
```

---

## 96.15 Feature Flags

```objc
// Feature Flag Manager
@interface FeatureFlagManager : NSObject

+ (instancetype)sharedManager;
- (BOOL)isFeatureEnabled:(NSString *)featureName;
- (void)enableFeature:(NSString *)featureName forGroups:(NSArray<NSString *> *)groups;

@end

// Feature Flag Constants
static NSString * const kFeatureNewChatUI = @"new_chat_ui";
static NSString * const kFeatureBiometricLogin = @"biometric_login";
static NSString * const kFeatureOfflineMode = @"offline_mode";
static NSString * const kFeatureAdvancedAnalytics = @"advanced_analytics";

@implementation FeatureFlagManager {
    NSMutableDictionary *_flags;
    User *_currentUser;
}

+ (instancetype)sharedManager {
    static FeatureFlagManager *instance = nil;
    static dispatch_once_t onceToken;
    dispatch_once(&onceToken, ^{
        instance = [[FeatureFlagManager alloc] init];
    });
    return instance;
}

- (instancetype)init {
    self = [super init];
    if (self) {
        [self loadFlagsFromServer];
    }
    return self;
}

- (BOOL)isFeatureEnabled:(NSString *)featureName {
    // 1. Check local override (for testing)
    NSNumber *localOverride = [[NSUserDefaults standardUserDefaults] 
                                objectForKey:[NSString stringWithFormat:@"ff_%@", featureName]];
    if (localOverride) {
        return [localOverride boolValue];
    }
    
    // 2. Check server flags
    NSDictionary *featureConfig = _flags[featureName];
    if (!featureConfig) {
        return NO; // Default: disabled
    }
    
    BOOL globalEnabled = [featureConfig[@"enabled"] boolValue];
    if (!globalEnabled) {
        return NO;
    }
    
    // 3. Check rollout percentage
    NSNumber *rolloutPercent = featureConfig[@"rollout_percent"];
    if (rolloutPercent) {
        // ใช้ user ID เพื่อให้ consistent (user เดิมเห็น feature เดิมเสมอ)
        NSInteger userHash = [[self currentUserHashString] hash] % 100;
        if (userHash >= [rolloutPercent integerValue]) {
            return NO;
        }
    }
    
    // 4. Check user groups
    NSArray *allowedGroups = featureConfig[@"allowed_groups"];
    if (allowedGroups && allowedGroups.count > 0) {
        for (NSString *group in _currentUser.groups) {
            if ([allowedGroups containsObject:group]) {
                return YES;
            }
        }
        return NO;
    }
    
    return YES;
}

// Toggle feature สำหรับ testing (Debug only)
#ifdef DEBUG
- (void)setLocalOverride:(BOOL)enabled forFeature:(NSString *)featureName {
    [[NSUserDefaults standardUserDefaults] 
        setObject:@(enabled) 
           forKey:[NSString stringWithFormat:@"ff_%@", featureName]];
}

- (void)clearLocalOverrides {
    NSUserDefaults *defaults = [NSUserDefaults standardUserDefaults];
    NSArray *keys = [[defaults dictionaryRepresentation] allKeys];
    for (NSString *key in keys) {
        if ([key hasPrefix:@"ff_"]) {
            [defaults removeObjectForKey:key];
        }
    }
}
#endif

@end

// การใช้งาน Feature Flags ในแอพ
- (void)setupFeatures {
    FeatureFlagManager *flags = [FeatureFlagManager sharedManager];
    
    if ([flags isFeatureEnabled:kFeatureNewChatUI]) {
        [self loadNewChatViewController];
    } else {
        [self loadLegacyChatViewController];
    }
    
    self.biometricLoginButton.hidden = ![flags isFeatureEnabled:kFeatureBiometricLogin];
    
    if ([flags isFeatureEnabled:kFeatureAdvancedAnalytics]) {
        [self enableAdvancedTracking];
    }
}
```

---

## 96.16 แบบฝึกหัด

### แบบฝึกหัดที่ 1: Managed Configuration

สร้างแอพที่รับ configuration จาก MDM:

```objc
// สร้าง ConfigurationManager ที่:
// 1. อ่านค่าจาก NSUserDefaults key: "com.apple.configuration.managed"
// 2. ถ้าไม่มี managed config ใช้ default values
// 3. อัพเดต UI เมื่อ config เปลี่ยน

// ตั้งค่า test configuration ใน Simulator:
// Settings → หา app ของคุณ → ตั้งค่าต่างๆ

// หรือสร้าง plist สำหรับ test:
NSDictionary *testConfig = @{
    @"ServerURL": @"https://test.company.com",
    @"MaxUploadSize": @(10),
    @"AllowExternalSharing": @NO
};
[[NSUserDefaults standardUserDefaults] setObject:testConfig 
                                          forKey:@"com.apple.configuration.managed"];
```

### แบบฝึกหัดที่ 2: Secure Storage

สร้าง SecureStorageManager ที่:
1. บันทึก sensitive data ใน Keychain
2. เข้ารหัส less sensitive data ด้วย AES-256
3. ใช้ NSFileProtectionComplete สำหรับ files
4. Auto-wipe เมื่อ tamper detection พบ jailbreak

### แบบฝึกหัดที่ 3: Feature Flags Dashboard

สร้าง debug screen ใน Settings ที่:
1. แสดง feature flags ทั้งหมด
2. Allow toggle แต่ละ flag (debug mode เท่านั้น)
3. แสดงว่า flag มาจากไหน (local, server, MDM)

```objc
// Debug Feature Flags ViewController
@interface FeatureFlagsDebugViewController : UITableViewController

@end

@implementation FeatureFlagsDebugViewController

- (void)viewDidLoad {
    [super viewDidLoad];
    self.title = @"Feature Flags (Debug)";
    
    // Warning banner
    UILabel *warningLabel = [[UILabel alloc] init];
    warningLabel.text = @"⚠️ Debug Mode Only - Changes affect this device only";
    warningLabel.textAlignment = NSTextAlignmentCenter;
    warningLabel.backgroundColor = [UIColor systemYellowColor];
    self.tableView.tableHeaderView = warningLabel;
}

// ... implement table view to show/toggle flags

@end
```

### แบบฝึกหัดที่ 4: Crash Reporting Integration

ติดตั้ง Firebase Crashlytics และ:
1. ตั้งค่า user identifier (anonymized)
2. Log key actions ก่อน crash-prone operations
3. Record non-fatal errors
4. สร้าง test crash button (debug only)

---

## สรุป

ในส่วนนี้เราได้เรียนรู้:
- Apple Developer Enterprise Program และการแจกจ่ายแอพภายในองค์กร
- MDM (Mobile Device Management) และ Managed App Configuration
- Enterprise Distribution ทั้ง 3 แบบ
- App Thinning: Slicing, Bitcode, On-Demand Resources
- VPN และ Network Configurations ผ่าน MDM
- Corporate SSO ด้วย SAML และ OAuth 2.0/OIDC
- LDAP/Active Directory Integration ผ่าน backend API
- Certificate-based Authentication และ Certificate Pinning
- Data Security Compliance (HIPAA, GDPR)
- App Wrapping vs MAM SDK
- Crash Reporting ด้วย Firebase Crashlytics
- Analytics Integration
- A/B Testing ด้วย Firebase Remote Config
- Feature Flags สำหรับ controlled rollout

Enterprise iOS Development ต้องการความรู้ที่หลากหลายทั้ง iOS programming, security, และ IT infrastructure เป็นสาขาที่ท้าทายแต่มีความต้องการในตลาดสูงมาก

---

*จบ Series: Enterprise iOS Development*
*ย้อนกลับไป: Part 95 - App Store Submission*
