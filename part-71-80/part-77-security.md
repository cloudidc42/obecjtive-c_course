# Part 77: iOS Security & Cryptography ใน Objective-C

## บทนำ

ความปลอดภัยของแอปพลิเคชัน iOS เป็นสิ่งที่นักพัฒนาต้องให้ความสำคัญเป็นอันดับแรก ในยุคที่ข้อมูลส่วนตัวของผู้ใช้มีคุณค่ามหาศาล การป้องกันข้อมูลรั่วไหลและการโจมตีจากผู้ไม่หวังดีจึงเป็นหน้าที่สำคัญของนักพัฒนาทุกคน

บทนี้จะครอบคลุม:
- **App Transport Security (ATS)** - การบังคับใช้ HTTPS
- **Keychain Services** - การจัดเก็บข้อมูลสำคัญอย่างปลอดภัย
- **CommonCrypto** - การเข้ารหัสข้อมูลด้วย AES, HMAC, RSA
- **Certificate Pinning** - การป้องกัน Man-in-the-Middle
- **Jailbreak Detection** - การตรวจจับอุปกรณ์ที่ถูก Jailbreak
- **Data Protection** - การป้องกันข้อมูลในระดับไฟล์
- **OWASP Mobile Top 10** - การป้องกันช่องโหว่ที่พบบ่อย

---

# หมวดที่ 1: App Transport Security (ATS)

## 77.1 ความเข้าใจพื้นฐาน ATS

**App Transport Security (ATS)** เป็นฟีเจอร์ของ Apple ที่บังคับให้แอปใช้การเชื่อมต่อ HTTPS ที่ปลอดภัย เริ่มตั้งแต่ iOS 9

ATS บังคับให้:
- ใช้ TLS 1.2 ขึ้นไป
- ใช้ Forward Secrecy Cipher Suites
- ใช้ SHA-256 หรือดีกว่าสำหรับ Certificate Signature
- ใช้ RSA 2048-bit key หรือ ECC 256-bit key

```objc
// ตัวอย่างการตั้งค่า ATS ใน Info.plist (ผ่านโค้ด)
// โดยปกติจะตั้งค่าใน Info.plist โดยตรง

// การตรวจสอบ ATS configuration ผ่านโค้ด
NSDictionary *infoPlist = [[NSBundle mainBundle] infoDictionary];
NSDictionary *atsConfig = infoPlist[@"NSAppTransportSecurity"];

if (atsConfig) {
    BOOL allowArbitraryLoads = [atsConfig[@"NSAllowsArbitraryLoads"] boolValue];
    NSLog(@"ATS AllowArbitraryLoads: %@", allowArbitraryLoads ? @"YES (ไม่ปลอดภัย!)" : @"NO (ปลอดภัย)");
    
    NSDictionary *exceptionDomains = atsConfig[@"NSExceptionDomains"];
    if (exceptionDomains) {
        NSLog(@"Exception Domains: %@", exceptionDomains.allKeys);
    }
}
```

## 77.2 การตั้งค่า ATS ใน Info.plist

```xml
<!-- Info.plist - การตั้งค่า ATS ที่แนะนำ -->
<key>NSAppTransportSecurity</key>
<dict>
    <!-- ไม่ควรใช้! - เปิดให้ทุก domain -->
    <!-- <key>NSAllowsArbitraryLoads</key><true/> -->
    
    <!-- สำหรับ development เท่านั้น -->
    <key>NSAllowsLocalNetworking</key>
    <true/>
    
    <!-- Exception สำหรับ domain เฉพาะ -->
    <key>NSExceptionDomains</key>
    <dict>
        <key>legacy-api.example.com</key>
        <dict>
            <key>NSExceptionAllowsInsecureHTTPLoads</key>
            <true/>
            <key>NSExceptionMinimumTLSVersion</key>
            <string>TLSv1.1</string>
            <key>NSExceptionRequiresForwardSecrecy</key>
            <false/>
            <!-- บันทึก: ต้องอธิบายให้ Apple ว่าทำไมถึง exempt domain นี้ -->
        </dict>
    </dict>
</dict>
```

## 77.3 การทดสอบ ATS Configuration

```objc
// ATSChecker.m - ตรวจสอบว่า URL ผ่าน ATS หรือไม่
@interface ATSChecker : NSObject <NSURLSessionDelegate>
@property (nonatomic, strong) NSURLSession *session;
@end

@implementation ATSChecker

- (instancetype)init {
    self = [super init];
    if (self) {
        NSURLSessionConfiguration *config = [NSURLSessionConfiguration defaultSessionConfiguration];
        self.session = [NSURLSession sessionWithConfiguration:config 
                                                     delegate:self 
                                                delegateQueue:nil];
    }
    return self;
}

- (void)checkURL:(NSString *)urlString completion:(void(^)(BOOL success, NSError *error))completion {
    NSURL *url = [NSURL URLWithString:urlString];
    NSURLRequest *request = [NSURLRequest requestWithURL:url];
    
    NSURLSessionDataTask *task = [self.session dataTaskWithRequest:request 
                                               completionHandler:^(NSData *data, NSURLResponse *response, NSError *error) {
        if (error) {
            NSLog(@"ATS Error: %@", error.localizedDescription);
            NSLog(@"Error Code: %ld", (long)error.code);
            // Error -1022 = ATS block
            completion(NO, error);
        } else {
            NSHTTPURLResponse *httpResponse = (NSHTTPURLResponse *)response;
            NSLog(@"✅ URL ผ่าน ATS: %@ (Status: %ld)", urlString, (long)httpResponse.statusCode);
            completion(YES, nil);
        }
    }];
    [task resume];
}

@end
```

---

# หมวดที่ 2: Keychain Services

## 77.4 ทำความเข้าใจ Keychain

**Keychain** คือ secure storage ของ iOS ที่เก็บข้อมูลสำคัญอย่างปลอดภัย เช่น:
- Passwords
- Cryptographic keys
- Certificates
- Tokens (OAuth, JWT)
- API Keys

Keychain มีความปลอดภัยสูงกว่า NSUserDefaults มาก เพราะ:
- ข้อมูลถูกเข้ารหัสด้วย Hardware Security Module
- ข้อมูลยังคงอยู่แม้ Uninstall App (ต้องตั้งค่าให้ลบ)
- สามารถ Share ระหว่าง App ในกลุ่มเดียวกันได้
- รองรับ biometric authentication

## 77.5 KeychainManager - Full CRUD Implementation

```objc
// KeychainManager.h
#import <Foundation/Foundation.h>
#import <Security/Security.h>

typedef NS_ENUM(NSInteger, KeychainError) {
    KeychainErrorItemNotFound = errSecItemNotFound,
    KeychainErrorDuplicateItem = errSecDuplicateItem,
    KeychainErrorUnexpected = -1,
    KeychainErrorInvalidData = -2
};

@interface KeychainManager : NSObject

// Singleton
+ (instancetype)sharedManager;

// CRUD Operations

/// บันทึก Password
- (BOOL)savePassword:(NSString *)password 
              forKey:(NSString *)key
               error:(NSError **)error;

/// อ่าน Password
- (NSString *)passwordForKey:(NSString *)key 
                       error:(NSError **)error;

/// อัปเดต Password
- (BOOL)updatePassword:(NSString *)password 
                forKey:(NSString *)key
                 error:(NSError **)error;

/// ลบ Password
- (BOOL)deletePasswordForKey:(NSString *)key 
                       error:(NSError **)error;

// Generic Data Operations

/// บันทึก Data
- (BOOL)saveData:(NSData *)data 
          forKey:(NSString *)key
           error:(NSError **)error;

/// อ่าน Data
- (NSData *)dataForKey:(NSString *)key 
                 error:(NSError **)error;

// Biometric Protected Operations

/// บันทึกข้อมูลที่ต้องใช้ Face ID / Touch ID
- (BOOL)saveBiometricProtectedPassword:(NSString *)password
                                forKey:(NSString *)key
                                 error:(NSError **)error;

/// ลบทุกรายการใน Keychain ของ App
- (BOOL)deleteAllItems:(NSError **)error;

@end
```

```objc
// KeychainManager.m
#import "KeychainManager.h"
#import <LocalAuthentication/LocalAuthentication.h>

@interface KeychainManager ()
@property (nonatomic, copy) NSString *serviceName;
@property (nonatomic, copy) NSString *accessGroup;
@end

@implementation KeychainManager

+ (instancetype)sharedManager {
    static KeychainManager *instance = nil;
    static dispatch_once_t onceToken;
    dispatch_once(&onceToken, ^{
        instance = [[self alloc] initPrivate];
    });
    return instance;
}

- (instancetype)initPrivate {
    self = [super init];
    if (self) {
        _serviceName = [[NSBundle mainBundle] bundleIdentifier] ?: @"com.app.keychain";
        // _accessGroup = @"TeamID.com.company.shared"; // สำหรับ shared keychain
    }
    return self;
}

#pragma mark - Private Helper Methods

- (NSMutableDictionary *)baseQueryForKey:(NSString *)key {
    NSMutableDictionary *query = [NSMutableDictionary dictionary];
    query[(__bridge id)kSecClass] = (__bridge id)kSecClassGenericPassword;
    query[(__bridge id)kSecAttrService] = self.serviceName;
    query[(__bridge id)kSecAttrAccount] = key;
    
    if (self.accessGroup) {
        query[(__bridge id)kSecAttrAccessGroup] = self.accessGroup;
    }
    
    return query;
}

- (NSError *)errorFromStatus:(OSStatus)status {
    NSString *message;
    switch (status) {
        case errSecItemNotFound:
            message = @"ไม่พบรายการใน Keychain";
            break;
        case errSecDuplicateItem:
            message = @"มีรายการนี้อยู่แล้วใน Keychain";
            break;
        case errSecAuthFailed:
            message = @"การยืนยันตัวตนล้มเหลว";
            break;
        case errSecUserCanceled:
            message = @"ผู้ใช้ยกเลิกการดำเนินการ";
            break;
        default:
            message = [NSString stringWithFormat:@"Keychain Error: %d", (int)status];
    }
    return [NSError errorWithDomain:@"KeychainErrorDomain" 
                               code:status 
                           userInfo:@{NSLocalizedDescriptionKey: message}];
}

#pragma mark - Password CRUD

- (BOOL)savePassword:(NSString *)password 
              forKey:(NSString *)key
               error:(NSError **)error {
    
    if (!password || !key) {
        if (error) {
            *error = [NSError errorWithDomain:@"KeychainErrorDomain" 
                                         code:KeychainErrorInvalidData 
                                     userInfo:@{NSLocalizedDescriptionKey: @"Password หรือ Key ไม่ถูกต้อง"}];
        }
        return NO;
    }
    
    NSData *passwordData = [password dataUsingEncoding:NSUTF8StringEncoding];
    
    // ตรวจสอบว่ามีอยู่แล้วหรือไม่
    NSError *fetchError = nil;
    NSString *existing = [self passwordForKey:key error:&fetchError];
    
    if (existing) {
        // มีอยู่แล้ว - Update
        return [self updatePassword:password forKey:key error:error];
    }
    
    // สร้างใหม่
    NSMutableDictionary *query = [self baseQueryForKey:key];
    query[(__bridge id)kSecValueData] = passwordData;
    
    // ตั้งค่า Access Control
    query[(__bridge id)kSecAttrAccessible] = (__bridge id)kSecAttrAccessibleWhenUnlockedThisDeviceOnly;
    
    OSStatus status = SecItemAdd((__bridge CFDictionaryRef)query, NULL);
    
    if (status != errSecSuccess) {
        if (error) {
            *error = [self errorFromStatus:status];
        }
        return NO;
    }
    
    NSLog(@"✅ บันทึก Keychain สำเร็จ: %@", key);
    return YES;
}

- (NSString *)passwordForKey:(NSString *)key 
                       error:(NSError **)error {
    
    NSMutableDictionary *query = [self baseQueryForKey:key];
    query[(__bridge id)kSecReturnData] = @YES;
    query[(__bridge id)kSecMatchLimit] = (__bridge id)kSecMatchLimitOne;
    
    CFTypeRef result = NULL;
    OSStatus status = SecItemCopyMatching((__bridge CFDictionaryRef)query, &result);
    
    if (status == errSecSuccess) {
        NSData *data = (__bridge_transfer NSData *)result;
        NSString *password = [[NSString alloc] initWithData:data encoding:NSUTF8StringEncoding];
        return password;
    } else if (status == errSecItemNotFound) {
        // ไม่พบ - ไม่ถือว่า error
        return nil;
    } else {
        if (error) {
            *error = [self errorFromStatus:status];
        }
        return nil;
    }
}

- (BOOL)updatePassword:(NSString *)password 
                forKey:(NSString *)key
                 error:(NSError **)error {
    
    NSData *passwordData = [password dataUsingEncoding:NSUTF8StringEncoding];
    
    NSMutableDictionary *query = [self baseQueryForKey:key];
    NSDictionary *attributes = @{
        (__bridge id)kSecValueData: passwordData
    };
    
    OSStatus status = SecItemUpdate((__bridge CFDictionaryRef)query, 
                                    (__bridge CFDictionaryRef)attributes);
    
    if (status != errSecSuccess) {
        if (error) {
            *error = [self errorFromStatus:status];
        }
        return NO;
    }
    
    NSLog(@"✅ อัปเดต Keychain สำเร็จ: %@", key);
    return YES;
}

- (BOOL)deletePasswordForKey:(NSString *)key 
                       error:(NSError **)error {
    
    NSMutableDictionary *query = [self baseQueryForKey:key];
    
    OSStatus status = SecItemDelete((__bridge CFDictionaryRef)query);
    
    if (status != errSecSuccess && status != errSecItemNotFound) {
        if (error) {
            *error = [self errorFromStatus:status];
        }
        return NO;
    }
    
    NSLog(@"✅ ลบ Keychain สำเร็จ: %@", key);
    return YES;
}

#pragma mark - Generic Data CRUD

- (BOOL)saveData:(NSData *)data 
          forKey:(NSString *)key
           error:(NSError **)error {
    
    if (!data || !key) {
        if (error) {
            *error = [NSError errorWithDomain:@"KeychainErrorDomain" 
                                         code:KeychainErrorInvalidData 
                                     userInfo:@{NSLocalizedDescriptionKey: @"Data หรือ Key ไม่ถูกต้อง"}];
        }
        return NO;
    }
    
    // ลบของเก่าก่อน
    [self deletePasswordForKey:key error:nil];
    
    NSMutableDictionary *query = [self baseQueryForKey:key];
    query[(__bridge id)kSecValueData] = data;
    query[(__bridge id)kSecAttrAccessible] = (__bridge id)kSecAttrAccessibleWhenUnlockedThisDeviceOnly;
    
    OSStatus status = SecItemAdd((__bridge CFDictionaryRef)query, NULL);
    
    if (status != errSecSuccess) {
        if (error) {
            *error = [self errorFromStatus:status];
        }
        return NO;
    }
    
    return YES;
}

- (NSData *)dataForKey:(NSString *)key 
                 error:(NSError **)error {
    
    NSMutableDictionary *query = [self baseQueryForKey:key];
    query[(__bridge id)kSecReturnData] = @YES;
    query[(__bridge id)kSecMatchLimit] = (__bridge id)kSecMatchLimitOne;
    
    CFTypeRef result = NULL;
    OSStatus status = SecItemCopyMatching((__bridge CFDictionaryRef)query, &result);
    
    if (status == errSecSuccess) {
        return (__bridge_transfer NSData *)result;
    } else if (status != errSecItemNotFound) {
        if (error) {
            *error = [self errorFromStatus:status];
        }
    }
    
    return nil;
}

#pragma mark - Biometric Protected

- (BOOL)saveBiometricProtectedPassword:(NSString *)password
                                forKey:(NSString *)key
                                 error:(NSError **)error {
    
    // สร้าง Access Control ที่ต้องใช้ Biometric
    CFErrorRef cfError = NULL;
    SecAccessControlRef accessControl = SecAccessControlCreateWithFlags(
        kCFAllocatorDefault,
        kSecAttrAccessibleWhenUnlockedThisDeviceOnly,
        kSecAccessControlBiometryCurrentSet | kSecAccessControlOr | kSecAccessControlDevicePasscode,
        &cfError
    );
    
    if (cfError) {
        if (error) {
            *error = (__bridge_transfer NSError *)cfError;
        }
        return NO;
    }
    
    NSData *passwordData = [password dataUsingEncoding:NSUTF8StringEncoding];
    
    // ลบของเก่าก่อน
    [self deletePasswordForKey:key error:nil];
    
    NSMutableDictionary *query = [self baseQueryForKey:key];
    query[(__bridge id)kSecValueData] = passwordData;
    query[(__bridge id)kSecAttrAccessControl] = (__bridge_transfer id)accessControl;
    
    // ต้องรันบน main thread สำหรับ biometric prompt
    LAContext *context = [[LAContext alloc] init];
    query[(__bridge id)kSecUseAuthenticationContext] = context;
    
    OSStatus status = SecItemAdd((__bridge CFDictionaryRef)query, NULL);
    
    if (status != errSecSuccess) {
        if (error) {
            *error = [self errorFromStatus:status];
        }
        return NO;
    }
    
    NSLog(@"✅ บันทึก Biometric Protected Keychain: %@", key);
    return YES;
}

#pragma mark - Utility

- (BOOL)deleteAllItems:(NSError **)error {
    NSMutableDictionary *query = [NSMutableDictionary dictionary];
    query[(__bridge id)kSecClass] = (__bridge id)kSecClassGenericPassword;
    query[(__bridge id)kSecAttrService] = self.serviceName;
    
    OSStatus status = SecItemDelete((__bridge CFDictionaryRef)query);
    
    if (status != errSecSuccess && status != errSecItemNotFound) {
        if (error) {
            *error = [self errorFromStatus:status];
        }
        return NO;
    }
    
    NSLog(@"✅ ลบ Keychain ทั้งหมดสำเร็จ");
    return YES;
}

@end
```

## 77.6 การใช้งาน KeychainManager

```objc
// ViewController.m - ตัวอย่างการใช้งาน
#import "KeychainManager.h"

@implementation SecurityViewController

- (void)demonstrateKeychain {
    KeychainManager *keychain = [KeychainManager sharedManager];
    NSError *error = nil;
    
    // บันทึก API Token
    BOOL saved = [keychain savePassword:@"eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9..."
                                 forKey:@"api_token"
                                  error:&error];
    if (saved) {
        NSLog(@"✅ บันทึก Token สำเร็จ");
    } else {
        NSLog(@"❌ Error: %@", error.localizedDescription);
    }
    
    // อ่าน Token
    NSString *token = [keychain passwordForKey:@"api_token" error:&error];
    if (token) {
        NSLog(@"Token: %@", [token substringToIndex:20]);
    }
    
    // อัปเดต Token
    [keychain updatePassword:@"new_token_value"
                      forKey:@"api_token"
                       error:&error];
    
    // บันทึก User Credentials
    NSDictionary *credentials = @{
        @"username": @"john_doe",
        @"password": @"SecureP@ss123"
    };
    NSData *credData = [NSJSONSerialization dataWithJSONObject:credentials 
                                                       options:0 
                                                         error:nil];
    [keychain saveData:credData forKey:@"user_credentials" error:&error];
    
    // อ่าน User Credentials
    NSData *savedCredData = [keychain dataForKey:@"user_credentials" error:&error];
    if (savedCredData) {
        NSDictionary *savedCreds = [NSJSONSerialization JSONObjectWithData:savedCredData 
                                                                   options:0 
                                                                     error:nil];
        NSLog(@"Username: %@", savedCreds[@"username"]);
    }
    
    // ลบ Token
    [keychain deletePasswordForKey:@"api_token" error:&error];
}

@end
```

---

# หมวดที่ 3: CommonCrypto - การเข้ารหัส

## 77.7 AES Encryption/Decryption

**AES (Advanced Encryption Standard)** เป็น symmetric encryption ที่ใช้กันแพร่หลายที่สุด

```objc
// CryptoManager.h
#import <Foundation/Foundation.h>
#import <CommonCrypto/CommonCrypto.h>

@interface CryptoManager : NSObject

+ (instancetype)sharedManager;

#pragma mark - AES Operations

/// เข้ารหัสด้วย AES-256 CBC
- (NSData *)encryptData:(NSData *)data 
               withKey:(NSData *)key
                    iv:(NSData *)iv
                 error:(NSError **)error;

/// ถอดรหัส AES-256 CBC
- (NSData *)decryptData:(NSData *)data 
               withKey:(NSData *)key
                    iv:(NSData *)iv
                 error:(NSError **)error;

/// เข้ารหัส String ด้วย AES-256
- (NSString *)encryptString:(NSString *)plaintext 
                   password:(NSString *)password
                      error:(NSError **)error;

/// ถอดรหัส String จาก AES-256
- (NSString *)decryptString:(NSString *)ciphertext 
                   password:(NSString *)password
                      error:(NSError **)error;

#pragma mark - Key Generation

/// สร้าง Random Key
- (NSData *)generateRandomKeyOfLength:(NSUInteger)length;

/// Derive Key จาก Password ด้วย PBKDF2
- (NSData *)deriveKeyFromPassword:(NSString *)password 
                             salt:(NSData *)salt
                       iterations:(NSUInteger)iterations
                        keyLength:(NSUInteger)keyLength;

#pragma mark - HMAC Operations

/// คำนวณ HMAC-SHA256
- (NSData *)hmacSHA256:(NSData *)data withKey:(NSData *)key;

/// ตรวจสอบ HMAC
- (BOOL)verifyHMACSHA256:(NSData *)hmac 
                 forData:(NSData *)data 
                 withKey:(NSData *)key;

#pragma mark - Hashing

/// คำนวณ SHA-256 Hash
- (NSData *)sha256Hash:(NSData *)data;

/// คำนวณ SHA-256 Hash ของ String
- (NSString *)sha256HashString:(NSString *)input;

@end
```

```objc
// CryptoManager.m
#import "CryptoManager.h"
#import <CommonCrypto/CommonCrypto.h>

static const NSUInteger kAESKeyLength = kCCKeySizeAES256;  // 32 bytes
static const NSUInteger kAESIVLength = kCCBlockSizeAES128;  // 16 bytes
static const NSUInteger kPBKDF2Iterations = 100000;

@implementation CryptoManager

+ (instancetype)sharedManager {
    static CryptoManager *instance = nil;
    static dispatch_once_t onceToken;
    dispatch_once(&onceToken, ^{
        instance = [[self alloc] init];
    });
    return instance;
}

#pragma mark - AES Operations

- (NSData *)encryptData:(NSData *)data 
               withKey:(NSData *)key
                    iv:(NSData *)iv
                 error:(NSError **)error {
    
    if (!data || !key || !iv) {
        if (error) {
            *error = [NSError errorWithDomain:@"CryptoErrorDomain" 
                                         code:-1 
                                     userInfo:@{NSLocalizedDescriptionKey: @"ข้อมูล, Key หรือ IV ไม่ถูกต้อง"}];
        }
        return nil;
    }
    
    // ตรวจสอบขนาด Key
    if (key.length != kCCKeySizeAES256) {
        if (error) {
            *error = [NSError errorWithDomain:@"CryptoErrorDomain" 
                                         code:-2 
                                     userInfo:@{NSLocalizedDescriptionKey: [NSString stringWithFormat:@"Key ต้องมีขนาด %lu bytes", kCCKeySizeAES256]}];
        }
        return nil;
    }
    
    // คำนวณขนาด Output Buffer
    size_t bufferSize = data.length + kCCBlockSizeAES128;
    void *buffer = malloc(bufferSize);
    
    if (!buffer) {
        if (error) {
            *error = [NSError errorWithDomain:@"CryptoErrorDomain" 
                                         code:-3 
                                     userInfo:@{NSLocalizedDescriptionKey: @"ไม่สามารถจัดสรรหน่วยความจำ"}];
        }
        return nil;
    }
    
    size_t numBytesEncrypted = 0;
    
    CCCryptorStatus cryptStatus = CCCrypt(
        kCCEncrypt,                    // Operation: Encrypt
        kCCAlgorithmAES,               // Algorithm: AES
        kCCOptionPKCS7Padding,         // Options: PKCS7 Padding
        key.bytes,                     // Key
        key.length,                    // Key Length
        iv.bytes,                      // IV
        data.bytes,                    // Input
        data.length,                   // Input Length
        buffer,                        // Output
        bufferSize,                    // Output Buffer Size
        &numBytesEncrypted             // Output Size
    );
    
    if (cryptStatus == kCCSuccess) {
        NSData *result = [NSData dataWithBytesNoCopy:buffer 
                                              length:numBytesEncrypted 
                                        freeWhenDone:YES];
        return result;
    } else {
        free(buffer);
        if (error) {
            *error = [NSError errorWithDomain:@"CryptoErrorDomain" 
                                         code:cryptStatus 
                                     userInfo:@{NSLocalizedDescriptionKey: [NSString stringWithFormat:@"การเข้ารหัสล้มเหลว: %d", cryptStatus]}];
        }
        return nil;
    }
}

- (NSData *)decryptData:(NSData *)data 
               withKey:(NSData *)key
                    iv:(NSData *)iv
                 error:(NSError **)error {
    
    if (!data || !key || !iv) {
        if (error) {
            *error = [NSError errorWithDomain:@"CryptoErrorDomain" 
                                         code:-1 
                                     userInfo:@{NSLocalizedDescriptionKey: @"ข้อมูล, Key หรือ IV ไม่ถูกต้อง"}];
        }
        return nil;
    }
    
    size_t bufferSize = data.length + kCCBlockSizeAES128;
    void *buffer = malloc(bufferSize);
    
    if (!buffer) {
        if (error) {
            *error = [NSError errorWithDomain:@"CryptoErrorDomain" 
                                         code:-3 
                                     userInfo:@{NSLocalizedDescriptionKey: @"ไม่สามารถจัดสรรหน่วยความจำ"}];
        }
        return nil;
    }
    
    size_t numBytesDecrypted = 0;
    
    CCCryptorStatus cryptStatus = CCCrypt(
        kCCDecrypt,                    // Operation: Decrypt
        kCCAlgorithmAES,               // Algorithm: AES
        kCCOptionPKCS7Padding,         // Options: PKCS7 Padding
        key.bytes,                     // Key
        key.length,                    // Key Length
        iv.bytes,                      // IV
        data.bytes,                    // Input
        data.length,                   // Input Length
        buffer,                        // Output
        bufferSize,                    // Output Buffer Size
        &numBytesDecrypted             // Output Size
    );
    
    if (cryptStatus == kCCSuccess) {
        NSData *result = [NSData dataWithBytesNoCopy:buffer 
                                              length:numBytesDecrypted 
                                        freeWhenDone:YES];
        return result;
    } else {
        free(buffer);
        if (error) {
            *error = [NSError errorWithDomain:@"CryptoErrorDomain" 
                                         code:cryptStatus 
                                     userInfo:@{NSLocalizedDescriptionKey: [NSString stringWithFormat:@"การถอดรหัสล้มเหลว: %d", cryptStatus]}];
        }
        return nil;
    }
}

- (NSString *)encryptString:(NSString *)plaintext 
                   password:(NSString *)password
                      error:(NSError **)error {
    
    // สร้าง Salt แบบ Random
    NSData *salt = [self generateRandomKeyOfLength:16];
    
    // Derive Key จาก Password
    NSData *key = [self deriveKeyFromPassword:password 
                                         salt:salt 
                                   iterations:kPBKDF2Iterations 
                                    keyLength:kAESKeyLength];
    
    // สร้าง Random IV
    NSData *iv = [self generateRandomKeyOfLength:kAESIVLength];
    
    // เข้ารหัส
    NSData *plaintextData = [plaintext dataUsingEncoding:NSUTF8StringEncoding];
    NSData *encryptedData = [self encryptData:plaintextData 
                                      withKey:key 
                                           iv:iv 
                                        error:error];
    
    if (!encryptedData) {
        return nil;
    }
    
    // รวม salt + iv + encrypted data และ encode เป็น Base64
    NSMutableData *combined = [NSMutableData data];
    [combined appendData:salt];          // 16 bytes
    [combined appendData:iv];            // 16 bytes
    [combined appendData:encryptedData]; // n bytes
    
    return [combined base64EncodedStringWithOptions:0];
}

- (NSString *)decryptString:(NSString *)ciphertext 
                   password:(NSString *)password
                      error:(NSError **)error {
    
    // Decode Base64
    NSData *combined = [[NSData alloc] initWithBase64EncodedString:ciphertext 
                                                           options:0];
    if (!combined || combined.length < 32) {
        if (error) {
            *error = [NSError errorWithDomain:@"CryptoErrorDomain" 
                                         code:-4 
                                     userInfo:@{NSLocalizedDescriptionKey: @"Ciphertext ไม่ถูกต้อง"}];
        }
        return nil;
    }
    
    // แยก salt, iv, encrypted data
    NSData *salt = [combined subdataWithRange:NSMakeRange(0, 16)];
    NSData *iv = [combined subdataWithRange:NSMakeRange(16, 16)];
    NSData *encryptedData = [combined subdataWithRange:NSMakeRange(32, combined.length - 32)];
    
    // Derive Key จาก Password
    NSData *key = [self deriveKeyFromPassword:password 
                                         salt:salt 
                                   iterations:kPBKDF2Iterations 
                                    keyLength:kAESKeyLength];
    
    // ถอดรหัส
    NSData *decryptedData = [self decryptData:encryptedData 
                                      withKey:key 
                                           iv:iv 
                                        error:error];
    
    if (!decryptedData) {
        return nil;
    }
    
    return [[NSString alloc] initWithData:decryptedData encoding:NSUTF8StringEncoding];
}

#pragma mark - Key Generation

- (NSData *)generateRandomKeyOfLength:(NSUInteger)length {
    NSMutableData *key = [NSMutableData dataWithLength:length];
    int result = SecRandomCopyBytes(kSecRandomDefault, length, key.mutableBytes);
    if (result != 0) {
        NSLog(@"❌ ไม่สามารถสร้าง Random Key ได้");
        return nil;
    }
    return [key copy];
}

- (NSData *)deriveKeyFromPassword:(NSString *)password 
                             salt:(NSData *)salt
                       iterations:(NSUInteger)iterations
                        keyLength:(NSUInteger)keyLength {
    
    NSData *passwordData = [password dataUsingEncoding:NSUTF8StringEncoding];
    NSMutableData *derivedKey = [NSMutableData dataWithLength:keyLength];
    
    int result = CCKeyDerivationPBKDF(
        kCCPBKDF2,                              // Algorithm
        passwordData.bytes,                      // Password
        passwordData.length,                     // Password Length
        salt.bytes,                              // Salt
        salt.length,                             // Salt Length
        kCCPRFHmacAlgSHA256,                    // PRF: HMAC-SHA256
        (unsigned int)iterations,                // Iterations
        derivedKey.mutableBytes,                 // Derived Key
        keyLength                                // Derived Key Length
    );
    
    if (result != kCCSuccess) {
        NSLog(@"❌ Key Derivation ล้มเหลว: %d", result);
        return nil;
    }
    
    return [derivedKey copy];
}

#pragma mark - HMAC Operations

- (NSData *)hmacSHA256:(NSData *)data withKey:(NSData *)key {
    NSMutableData *hmac = [NSMutableData dataWithLength:CC_SHA256_DIGEST_LENGTH];
    
    CCHmac(kCCHmacAlgSHA256, 
           key.bytes, 
           key.length, 
           data.bytes, 
           data.length, 
           hmac.mutableBytes);
    
    return [hmac copy];
}

- (BOOL)verifyHMACSHA256:(NSData *)hmac 
                 forData:(NSData *)data 
                 withKey:(NSData *)key {
    
    NSData *computedHmac = [self hmacSHA256:data withKey:key];
    
    // Constant-time comparison เพื่อป้องกัน timing attack
    if (computedHmac.length != hmac.length) {
        return NO;
    }
    
    const uint8_t *a = computedHmac.bytes;
    const uint8_t *b = hmac.bytes;
    uint8_t diff = 0;
    
    for (NSUInteger i = 0; i < computedHmac.length; i++) {
        diff |= a[i] ^ b[i];
    }
    
    return diff == 0;
}

#pragma mark - Hashing

- (NSData *)sha256Hash:(NSData *)data {
    NSMutableData *hash = [NSMutableData dataWithLength:CC_SHA256_DIGEST_LENGTH];
    CC_SHA256(data.bytes, (CC_LONG)data.length, hash.mutableBytes);
    return [hash copy];
}

- (NSString *)sha256HashString:(NSString *)input {
    NSData *data = [input dataUsingEncoding:NSUTF8StringEncoding];
    NSData *hash = [self sha256Hash:data];
    
    // แปลงเป็น Hex String
    const uint8_t *bytes = hash.bytes;
    NSMutableString *hexString = [NSMutableString stringWithCapacity:hash.length * 2];
    for (NSUInteger i = 0; i < hash.length; i++) {
        [hexString appendFormat:@"%02x", bytes[i]];
    }
    return [hexString copy];
}

@end
```

## 77.8 การใช้งาน CryptoManager

```objc
// ตัวอย่างการใช้งานจริง
CryptoManager *crypto = [CryptoManager sharedManager];
NSError *error = nil;

// 1. เข้ารหัส/ถอดรหัส String
NSString *secret = @"ข้อมูลลับสุดยอด - My Super Secret Data";
NSString *password = @"MyStrongPassword123!";

NSString *encrypted = [crypto encryptString:secret 
                                    password:password 
                                       error:&error];
NSLog(@"Encrypted: %@", [encrypted substringToIndex:30]);

NSString *decrypted = [crypto decryptString:encrypted 
                                    password:password 
                                       error:&error];
NSLog(@"Decrypted: %@", decrypted);

// 2. HMAC สำหรับ Message Integrity
NSData *messageData = [@"Important Message" dataUsingEncoding:NSUTF8StringEncoding];
NSData *hmacKey = [crypto generateRandomKeyOfLength:32];

NSData *messageHmac = [crypto hmacSHA256:messageData withKey:hmacKey];
BOOL isValid = [crypto verifyHMACSHA256:messageHmac forData:messageData withKey:hmacKey];
NSLog(@"HMAC Valid: %@", isValid ? @"✅ Valid" : @"❌ Invalid");

// 3. SHA-256 Hashing
NSString *hashString = [crypto sha256HashString:@"Hello World"];
NSLog(@"SHA-256: %@", hashString);
```

---

# หมวดที่ 4: Certificate Pinning

## 77.9 ทำความเข้าใจ Certificate Pinning

**Certificate Pinning** คือเทคนิคป้องกัน Man-in-the-Middle (MITM) attack โดยการ "ตรึง" certificate ที่ยอมรับได้ไว้ในแอป

มี 2 แบบ:
1. **Certificate Pinning** - ตรึง Certificate ทั้งใบ (แนะนำน้อยกว่า เพราะ Cert หมดอายุต้องอัปเดตแอป)
2. **Public Key Pinning** - ตรึง Public Key เท่านั้น (แนะนำ เพราะ Key อยู่นานกว่า Cert)

```objc
// CertificatePinningManager.h
#import <Foundation/Foundation.h>

@interface CertificatePinningManager : NSObject <NSURLSessionDelegate>

+ (instancetype)sharedManager;

/// สร้าง NSURLSession ที่มี Certificate Pinning
- (NSURLSession *)createPinnedSession;

/// ดึงข้อมูลพร้อม Pinning
- (void)fetchURL:(NSURL *)url 
      completion:(void(^)(NSData *data, NSURLResponse *response, NSError *error))completion;

@end
```

```objc
// CertificatePinningManager.m
#import "CertificatePinningManager.h"
#import <CommonCrypto/CommonCrypto.h>

// Public Key Hashes ของ Server ที่เชื่อถือ (SHA-256 Base64)
// สร้างด้วย: openssl s_client -connect api.example.com:443 | openssl x509 -pubkey -noout | openssl pkey -pubin -outform der | openssl dgst -sha256 -binary | base64
static NSArray *trustedPublicKeyHashes;

@interface CertificatePinningManager ()
@property (nonatomic, strong) NSURLSession *pinnedSession;
@end

@implementation CertificatePinningManager

+ (void)initialize {
    if (self == [CertificatePinningManager class]) {
        // ใส่ hash ของ public key ที่ต้องการ pin
        trustedPublicKeyHashes = @[
            @"BBBBBBBBBBBBBBBBBBBBBBBBBBBBBBBBBBBBBBBBBBB=",  // Primary Cert
            @"CCCCCCCCCCCCCCCCCCCCCCCCCCCCCCCCCCCCCCCCCCC=",  // Backup Cert
        ];
    }
}

+ (instancetype)sharedManager {
    static CertificatePinningManager *instance = nil;
    static dispatch_once_t onceToken;
    dispatch_once(&onceToken, ^{
        instance = [[self alloc] init];
    });
    return instance;
}

- (instancetype)init {
    self = [super init];
    if (self) {
        _pinnedSession = [self createPinnedSession];
    }
    return self;
}

- (NSURLSession *)createPinnedSession {
    NSURLSessionConfiguration *config = [NSURLSessionConfiguration defaultSessionConfiguration];
    config.TLSMinimumSupportedProtocolVersion = tls_protocol_version_TLSv13;
    
    return [NSURLSession sessionWithConfiguration:config 
                                         delegate:self 
                                    delegateQueue:[NSOperationQueue mainQueue]];
}

- (void)fetchURL:(NSURL *)url 
      completion:(void(^)(NSData *data, NSURLResponse *response, NSError *error))completion {
    
    NSURLSessionDataTask *task = [self.pinnedSession dataTaskWithURL:url 
                                                  completionHandler:completion];
    [task resume];
}

#pragma mark - NSURLSessionDelegate (Certificate Pinning)

- (void)URLSession:(NSURLSession *)session 
didReceiveChallenge:(NSURLAuthenticationChallenge *)challenge
 completionHandler:(void(^)(NSURLSessionAuthChallengeDisposition, NSURLCredential *))completionHandler {
    
    // ตรวจสอบว่าเป็น Server Trust Challenge
    if (![challenge.protectionSpace.authenticationMethod 
          isEqualToString:NSURLAuthenticationMethodServerTrust]) {
        completionHandler(NSURLSessionAuthChallengePerformDefaultHandling, nil);
        return;
    }
    
    SecTrustRef serverTrust = challenge.protectionSpace.serverTrust;
    
    // ตรวจสอบ Certificate Chain ก่อน
    if (![self validateServerTrust:serverTrust forHost:challenge.protectionSpace.host]) {
        NSLog(@"❌ Certificate Validation ล้มเหลว");
        completionHandler(NSURLSessionAuthChallengeCancelAuthenticationChallenge, nil);
        return;
    }
    
    // ตรวจสอบ Public Key Pinning
    if ([self publicKeyPinsMatchForServerTrust:serverTrust]) {
        NSLog(@"✅ Public Key Pinning สำเร็จ");
        NSURLCredential *credential = [NSURLCredential credentialForTrust:serverTrust];
        completionHandler(NSURLSessionAuthChallengeUseCredential, credential);
    } else {
        NSLog(@"❌ Public Key Pinning ล้มเหลว - อาจเป็น MITM Attack!");
        completionHandler(NSURLSessionAuthChallengeCancelAuthenticationChallenge, nil);
        
        // แจ้งเตือน Security Team
        [self reportPotentialMITMAttack:challenge.protectionSpace.host];
    }
}

#pragma mark - Certificate Validation

- (BOOL)validateServerTrust:(SecTrustRef)serverTrust 
                    forHost:(NSString *)host {
    
    // สร้าง Policy สำหรับ SSL
    SecPolicyRef policy = SecPolicyCreateSSL(true, (__bridge CFStringRef)host);
    
    // ตั้งค่า Policy ให้ Trust
    OSStatus status = SecTrustSetPolicies(serverTrust, policy);
    CFRelease(policy);
    
    if (status != errSecSuccess) {
        return NO;
    }
    
    // ประเมิน Trust
    CFErrorRef error = NULL;
    bool isTrusted = SecTrustEvaluateWithError(serverTrust, &error);
    
    if (error) {
        NSLog(@"Trust Evaluation Error: %@", (__bridge NSError *)error);
        CFRelease(error);
    }
    
    return isTrusted;
}

#pragma mark - Public Key Pinning

- (BOOL)publicKeyPinsMatchForServerTrust:(SecTrustRef)serverTrust {
    // ดึง Certificate จาก Server
    CFIndex certCount = SecTrustGetCertificateCount(serverTrust);
    
    for (CFIndex i = 0; i < certCount; i++) {
        SecCertificateRef certificate = SecTrustGetCertificateAtIndex(serverTrust, i);
        
        // ดึง Public Key จาก Certificate
        NSData *publicKeyHash = [self publicKeyHashForCertificate:certificate];
        
        if (!publicKeyHash) {
            continue;
        }
        
        NSString *publicKeyHashBase64 = [publicKeyHash base64EncodedStringWithOptions:0];
        
        // ตรวจสอบกับ Pinned Hashes
        for (NSString *pinnedHash in trustedPublicKeyHashes) {
            if ([publicKeyHashBase64 isEqualToString:pinnedHash]) {
                return YES;
            }
        }
    }
    
    return NO;
}

- (NSData *)publicKeyHashForCertificate:(SecCertificateRef)certificate {
    // ดึง Public Key จาก Certificate
    SecKeyRef publicKey = NULL;
    
    // iOS 12+ method
    if (@available(iOS 12.0, *)) {
        publicKey = SecCertificateCopyKey(certificate);
    }
    
    if (!publicKey) {
        return nil;
    }
    
    // Export Public Key เป็น Data
    CFErrorRef error = NULL;
    NSData *publicKeyData = (__bridge_transfer NSData *)SecKeyCopyExternalRepresentation(publicKey, &error);
    CFRelease(publicKey);
    
    if (!publicKeyData) {
        return nil;
    }
    
    // คำนวณ SHA-256 Hash
    NSMutableData *hash = [NSMutableData dataWithLength:CC_SHA256_DIGEST_LENGTH];
    
    // เพิ่ม ASN.1 Header สำหรับ RSA Public Key
    // (ต้องเพิ่ม header ก่อน hash ถ้าเป็น RSA key)
    CC_SHA256(publicKeyData.bytes, (CC_LONG)publicKeyData.length, hash.mutableBytes);
    
    return [hash copy];
}

- (void)reportPotentialMITMAttack:(NSString *)host {
    NSLog(@"⚠️ Potential MITM Attack detected for host: %@", host);
    // TODO: ส่ง alert ไปยัง Security Monitoring System
    // TODO: บันทึก Log พร้อม timestamp
    // TODO: แจ้ง User
}

@end
```

---

# หมวดที่ 5: Jailbreak Detection

## 77.10 เทคนิคการตรวจจับ Jailbreak

การตรวจจับ Jailbreak ช่วยปกป้องแอปจากการถูก Reverse Engineering และการโจมตีในระดับ Runtime

```objc
// JailbreakDetector.h
#import <Foundation/Foundation.h>

@interface JailbreakDetector : NSObject

/// ตรวจสอบว่า Device ถูก Jailbreak หรือไม่
+ (BOOL)isDeviceJailbroken;

/// ตรวจสอบไฟล์ที่บ่งบอก Jailbreak
+ (BOOL)checkJailbreakFiles;

/// ตรวจสอบว่าเขียนไฟล์นอก Sandbox ได้หรือไม่
+ (BOOL)checkSandboxViolation;

/// ตรวจสอบ Symbolic Links ที่น่าสงสัย
+ (BOOL)checkSymbolicLinks;

/// ตรวจสอบ Cydia URL Scheme
+ (BOOL)checkCydiaURLScheme;

/// ตรวจสอบ Dynamic Libraries ที่น่าสงสัย
+ (BOOL)checkSuspiciousLibraries;

/// ตรวจสอบ Fork syscall
+ (BOOL)checkForkBehavior;

@end
```

```objc
// JailbreakDetector.m
#import "JailbreakDetector.h"
#import <sys/stat.h>
#include <dlfcn.h>

@implementation JailbreakDetector

+ (BOOL)isDeviceJailbroken {
    // ใน Simulator ไม่ต้องตรวจ
#if TARGET_IPHONE_SIMULATOR
    return NO;
#endif
    
    // รวมผลการตรวจสอบทุกวิธี
    if ([self checkJailbreakFiles]) {
        NSLog(@"⚠️ Jailbreak ตรวจพบ: Jailbreak Files");
        return YES;
    }
    
    if ([self checkSandboxViolation]) {
        NSLog(@"⚠️ Jailbreak ตรวจพบ: Sandbox Violation");
        return YES;
    }
    
    if ([self checkSymbolicLinks]) {
        NSLog(@"⚠️ Jailbreak ตรวจพบ: Suspicious Symlinks");
        return YES;
    }
    
    if ([self checkCydiaURLScheme]) {
        NSLog(@"⚠️ Jailbreak ตรวจพบ: Cydia URL Scheme");
        return YES;
    }
    
    if ([self checkSuspiciousLibraries]) {
        NSLog(@"⚠️ Jailbreak ตรวจพบ: Suspicious Libraries");
        return YES;
    }
    
    return NO;
}

+ (BOOL)checkJailbreakFiles {
    NSArray *jailbreakPaths = @[
        @"/Applications/Cydia.app",
        @"/Applications/blackra1n.app",
        @"/Applications/FakeCarrier.app",
        @"/Applications/Icy.app",
        @"/Applications/IntelliScreen.app",
        @"/Applications/MxTube.app",
        @"/Applications/RockApp.app",
        @"/Applications/SBSettings.app",
        @"/Applications/WinterBoard.app",
        @"/Library/MobileSubstrate/MobileSubstrate.dylib",
        @"/Library/MobileSubstrate/DynamicLibraries",
        @"/private/var/lib/apt",
        @"/private/var/lib/cydia",
        @"/private/var/mobile/Library/SBSettings/Themes",
        @"/private/var/stash",
        @"/private/var/tmp/cydia.log",
        @"/usr/bin/sshd",
        @"/usr/libexec/sftp-server",
        @"/usr/sbin/sshd",
        @"/bin/bash",
        @"/bin/sh",
        @"/etc/apt",
        @"/etc/ssh/sshd_config",
        @"/var/cache/apt",
        @"/var/lib/apt",
        @"/var/log/syslog",
    ];
    
    NSFileManager *fileManager = [NSFileManager defaultManager];
    
    for (NSString *path in jailbreakPaths) {
        // ใช้ stat แทน fileExistsAtPath เพื่อป้องกัน hook
        struct stat statInfo;
        if (stat([path UTF8String], &statInfo) == 0) {
            NSLog(@"⚠️ พบไฟล์ Jailbreak: %@", path);
            return YES;
        }
    }
    
    return NO;
}

+ (BOOL)checkSandboxViolation {
    // พยายามเขียนไฟล์นอก Sandbox
    NSString *testPath = @"/private/jailbreak_test.txt";
    NSString *testContent = @"jailbreak_test";
    
    NSError *error = nil;
    BOOL success = [testContent writeToFile:testPath 
                                 atomically:YES 
                                   encoding:NSUTF8StringEncoding 
                                      error:&error];
    
    if (success) {
        // ลบไฟล์ทดสอบ
        [[NSFileManager defaultManager] removeItemAtPath:testPath error:nil];
        return YES;  // Device ถูก Jailbreak
    }
    
    return NO;
}

+ (BOOL)checkSymbolicLinks {
    NSArray *suspiciousSymlinks = @[
        @"/Library/Ringtones",
        @"/Library/Wallpaper",
        @"/usr/arm-apple-darwin9",
        @"/usr/include",
        @"/usr/libexec",
        @"/usr/share",
    ];
    
    NSFileManager *fileManager = [NSFileManager defaultManager];
    
    for (NSString *path in suspiciousSymlinks) {
        NSDictionary *attrs = [fileManager attributesOfItemAtPath:path error:nil];
        if (attrs && [attrs[NSFileType] isEqualToString:NSFileTypeSymbolicLink]) {
            return YES;
        }
    }
    
    return NO;
}

+ (BOOL)checkCydiaURLScheme {
    NSURL *cydiaURL = [NSURL URLWithString:@"cydia://package/com.example.package"];
    return [[UIApplication sharedApplication] canOpenURL:cydiaURL];
}

+ (BOOL)checkSuspiciousLibraries {
    NSArray *suspiciousLibraries = @[
        @"MobileSubstrate",
        @"CydiaSubstrate",
        @"libcycript",
        @"SSLKillSwitch",
        @"TweakName",
        @"Substrate",
        @"SubstrateLoader",
    ];
    
    for (NSString *lib in suspiciousLibraries) {
        if (dlopen([lib UTF8String], RTLD_NOW | RTLD_NOLOAD) != NULL) {
            return YES;
        }
    }
    
    return NO;
}

+ (BOOL)checkForkBehavior {
    // ใน iOS ปกติ fork() จะ return -1 เพราะ Sandbox
    // ใน Jailbreak device fork() อาจทำงานได้
    int pid = fork();
    if (pid >= 0) {
        // fork() สำเร็จ - อาจถูก Jailbreak
        if (pid > 0) {
            kill(pid, SIGTERM);
        }
        return YES;
    }
    return NO;
}

@end
```

---

# หมวดที่ 6: Data Protection Classes

## 77.11 iOS Data Protection

iOS มี Data Protection ระดับไฟล์โดยใช้ Encryption Key ที่ผูกกับ Device Passcode

```objc
// DataProtectionManager.m
#import <Foundation/Foundation.h>

@interface DataProtectionManager : NSObject

/// บันทึกไฟล์พร้อม Protection Class
+ (BOOL)saveData:(NSData *)data 
          toPath:(NSString *)path 
 protectionClass:(NSString *)protectionClass
           error:(NSError **)error;

/// ตรวจสอบ Protection Class ของไฟล์
+ (NSString *)protectionClassForPath:(NSString *)path;

/// แสดงตัวอย่าง Protection Classes ต่างๆ
+ (void)demonstrateProtectionClasses;

@end

@implementation DataProtectionManager

+ (BOOL)saveData:(NSData *)data 
          toPath:(NSString *)path 
 protectionClass:(NSString *)protectionClass
           error:(NSError **)error {
    
    NSDictionary *attributes = @{
        NSFileProtectionKey: protectionClass
    };
    
    // บันทึกไฟล์ก่อน
    BOOL success = [data writeToFile:path options:NSDataWritingAtomic error:error];
    
    if (!success) {
        return NO;
    }
    
    // ตั้งค่า Protection Class
    NSFileManager *fm = [NSFileManager defaultManager];
    success = [fm setAttributes:attributes ofItemAtPath:path error:error];
    
    return success;
}

+ (NSString *)protectionClassForPath:(NSString *)path {
    NSFileManager *fm = [NSFileManager defaultManager];
    NSDictionary *attrs = [fm attributesOfItemAtPath:path error:nil];
    return attrs[NSFileProtectionKey];
}

+ (void)demonstrateProtectionClasses {
    NSString *docsPath = [NSSearchPathForDirectoriesInDomains(NSDocumentDirectory, NSUserDomainMask, YES) firstObject];
    NSData *testData = [@"Test Data" dataUsingEncoding:NSUTF8StringEncoding];
    NSError *error = nil;
    
    // 1. NSFileProtectionComplete
    // ไฟล์เข้าถึงได้เฉพาะตอน Device Unlocked
    // เหมาะกับ: ข้อมูลส่วนตัว, พาสเวิร์ด
    NSString *completeFile = [docsPath stringByAppendingPathComponent:@"complete.dat"];
    [self saveData:testData 
            toPath:completeFile 
   protectionClass:NSFileProtectionComplete 
             error:&error];
    NSLog(@"✅ NSFileProtectionComplete: เข้าถึงได้เฉพาะตอน Unlocked");
    
    // 2. NSFileProtectionCompleteUnlessOpen
    // เข้าถึงได้ขณะที่ไฟล์ถูกเปิดอยู่แม้ Lock
    // เหมาะกับ: ไฟล์ที่ต้องเขียนในขณะ Background
    NSString *unlessOpenFile = [docsPath stringByAppendingPathComponent:@"unless_open.dat"];
    [self saveData:testData 
            toPath:unlessOpenFile 
   protectionClass:NSFileProtectionCompleteUnlessOpen 
             error:&error];
    NSLog(@"✅ NSFileProtectionCompleteUnlessOpen: เปิดได้แม้ Lock ถ้าเปิดอยู่");
    
    // 3. NSFileProtectionCompleteUntilFirstUserAuthentication
    // เข้าถึงได้หลัง Boot + Unlock ครั้งแรก
    // เหมาะกับ: Background tasks, Push notification processing
    NSString *firstAuthFile = [docsPath stringByAppendingPathComponent:@"first_auth.dat"];
    [self saveData:testData 
            toPath:firstAuthFile 
   protectionClass:NSFileProtectionCompleteUntilFirstUserAuthentication 
             error:&error];
    NSLog(@"✅ NSFileProtectionCompleteUntilFirstUserAuthentication: หลัง Unlock ครั้งแรก");
    
    // 4. NSFileProtectionNone (ไม่แนะนำ!)
    // ไม่มีการป้องกัน - อย่าใช้สำหรับข้อมูลสำคัญ
    NSString *noneFile = [docsPath stringByAppendingPathComponent:@"no_protection.dat"];
    [self saveData:testData 
            toPath:noneFile 
   protectionClass:NSFileProtectionNone 
             error:&error];
    NSLog(@"⚠️ NSFileProtectionNone: ไม่แนะนำสำหรับข้อมูลสำคัญ!");
    
    // ตรวจสอบ Protection Class
    NSLog(@"Protection Class ของไฟล์: %@", [self protectionClassForPath:completeFile]);
}

@end
```

## 77.12 NSUserDefaults vs Keychain - เปรียบเทียบความปลอดภัย

```objc
// SecurityComparison.m
@interface SecurityComparison : NSObject
+ (void)compareStorageOptions;
@end

@implementation SecurityComparison

+ (void)compareStorageOptions {
    // ❌ ไม่ควรใช้ NSUserDefaults สำหรับข้อมูลสำคัญ
    [[NSUserDefaults standardUserDefaults] setObject:@"my_password" forKey:@"password"];
    // ปัญหา: ไม่ได้รับการเข้ารหัส, ใครก็อ่านได้ด้วย iExplorer หรือ backup
    
    // ✅ ควรใช้ Keychain แทน
    KeychainManager *keychain = [KeychainManager sharedManager];
    [keychain savePassword:@"my_password" forKey:@"password" error:nil];
    
    // เปรียบเทียบ
    NSLog(@"NSUserDefaults: ❌ ไม่ปลอดภัย - ไม่เข้ารหัส, อ่านได้ผ่าน Backup");
    NSLog(@"Keychain: ✅ ปลอดภัย - เข้ารหัสด้วย Hardware, ป้องกันด้วย Passcode");
    NSLog(@"NSFileProtectionComplete: ✅ ปลอดภัย - เข้ารหัสไฟล์ระดับ OS");
    
    // สรุปการใช้งาน
    NSDictionary *usageGuide = @{
        @"Password/Token": @"Keychain",
        @"User Preferences": @"NSUserDefaults (ข้อมูลไม่สำคัญ)",
        @"Sensitive Files": @"NSFileProtectionComplete",
        @"Biometric Data": @"Keychain + kSecAccessControlBiometryCurrentSet",
        @"Offline Data": @"Core Data + NSFileProtectionComplete",
    };
    
    [usageGuide enumerateKeysAndObjectsUsingBlock:^(NSString *dataType, NSString *storage, BOOL *stop) {
        NSLog(@"%-25@ -> %@", dataType, storage);
    }];
}

@end
```

---

# หมวดที่ 7: Secure Coding Practices

## 77.13 หลักการ Secure Coding

```objc
// SecureCodingExamples.m

// ❌ ไม่ปลอดภัย - SQL Injection
- (NSArray *)unsafeGetUsersWithName:(NSString *)name {
    NSString *query = [NSString stringWithFormat:
                       @"SELECT * FROM users WHERE name = '%@'", name];
    // ถ้า name = "'; DROP TABLE users;--" จะเกิด SQL Injection!
    return @[];
}

// ✅ ปลอดภัย - Parameterized Query
- (NSArray *)safeGetUsersWithName:(NSString *)name {
    // ใช้ Prepared Statement
    // const char *sql = "SELECT * FROM users WHERE name = ?";
    // sqlite3_bind_text(stmt, 1, [name UTF8String], -1, SQLITE_TRANSIENT);
    NSLog(@"Using parameterized query - safe from SQL injection");
    return @[];
}

// ❌ ไม่ปลอดภัย - Logging Sensitive Data
- (void)unsafeLogin:(NSString *)username password:(NSString *)password {
    NSLog(@"Login: username=%@, password=%@", username, password);
    // Log จะถูกเก็บไว้ใน Console และอ่านได้
}

// ✅ ปลอดภัย - ไม่ Log ข้อมูลสำคัญ
- (void)safeLogin:(NSString *)username password:(NSString *)password {
    NSLog(@"Login attempt for user: %@", username);
    // ไม่ Log password เด็ดขาด
    
    // ใน Production ควรใช้ Structured Logging ที่กรองข้อมูลสำคัญ
}

// ❌ ไม่ปลอดภัย - เก็บ Secret ใน Code
NSString *const kAPIKey = @"sk-live-1234567890abcdef"; // ❌ Hard-coded!

// ✅ ปลอดภัย - ดึงจาก Keychain หรือ Secure Config
- (NSString *)getAPIKey {
    NSError *error = nil;
    NSString *apiKey = [[KeychainManager sharedManager] passwordForKey:@"api_key" error:&error];
    if (!apiKey) {
        // ดึงจาก Server พร้อม Authentication
        NSLog(@"API Key not found, need to authenticate first");
    }
    return apiKey;
}

// ❌ ไม่ปลอดภัย - ใช้ HTTP
- (void)unsafeSendData:(NSDictionary *)data {
    NSURL *url = [NSURL URLWithString:@"http://api.example.com/data"];
    // HTTP ไม่เข้ารหัส - ข้อมูลถูกดักฟังได้
}

// ✅ ปลอดภัย - ใช้ HTTPS เสมอ
- (void)safeSendData:(NSDictionary *)data {
    NSURL *url = [NSURL URLWithString:@"https://api.example.com/data"];
    // HTTPS เข้ารหัสข้อมูลระหว่างส่ง
}
```

## 77.14 Input Validation

```objc
// InputValidator.m
@interface InputValidator : NSObject

+ (BOOL)isValidEmail:(NSString *)email;
+ (BOOL)isValidPassword:(NSString *)password error:(NSString **)errorMessage;
+ (NSString *)sanitizeInput:(NSString *)input;
+ (BOOL)isValidURL:(NSString *)urlString;

@end

@implementation InputValidator

+ (BOOL)isValidEmail:(NSString *)email {
    if (!email || email.length == 0) return NO;
    
    // ใช้ NSPredicate สำหรับ Email Validation
    NSString *emailRegex = @"[A-Z0-9a-z._%+-]+@[A-Za-z0-9.-]+\\.[A-Za-z]{2,64}";
    NSPredicate *emailTest = [NSPredicate predicateWithFormat:@"SELF MATCHES %@", emailRegex];
    return [emailTest evaluateWithObject:email];
}

+ (BOOL)isValidPassword:(NSString *)password error:(NSString **)errorMessage {
    if (password.length < 8) {
        if (errorMessage) *errorMessage = @"Password ต้องมีอย่างน้อย 8 ตัวอักษร";
        return NO;
    }
    
    // ตรวจสอบ uppercase
    NSCharacterSet *uppercaseSet = [NSCharacterSet uppercaseLetterCharacterSet];
    if ([password rangeOfCharacterFromSet:uppercaseSet].location == NSNotFound) {
        if (errorMessage) *errorMessage = @"Password ต้องมีตัวพิมพ์ใหญ่อย่างน้อย 1 ตัว";
        return NO;
    }
    
    // ตรวจสอบ lowercase
    NSCharacterSet *lowercaseSet = [NSCharacterSet lowercaseLetterCharacterSet];
    if ([password rangeOfCharacterFromSet:lowercaseSet].location == NSNotFound) {
        if (errorMessage) *errorMessage = @"Password ต้องมีตัวพิมพ์เล็กอย่างน้อย 1 ตัว";
        return NO;
    }
    
    // ตรวจสอบ digit
    NSCharacterSet *digitSet = [NSCharacterSet decimalDigitCharacterSet];
    if ([password rangeOfCharacterFromSet:digitSet].location == NSNotFound) {
        if (errorMessage) *errorMessage = @"Password ต้องมีตัวเลขอย่างน้อย 1 ตัว";
        return NO;
    }
    
    // ตรวจสอบ special character
    NSCharacterSet *specialSet = [NSCharacterSet characterSetWithCharactersInString:@"!@#$%^&*()_+-=[]{}|;':\",./<>?"];
    if ([password rangeOfCharacterFromSet:specialSet].location == NSNotFound) {
        if (errorMessage) *errorMessage = @"Password ต้องมีอักขระพิเศษอย่างน้อย 1 ตัว";
        return NO;
    }
    
    return YES;
}

+ (NSString *)sanitizeInput:(NSString *)input {
    if (!input) return @"";
    
    // ลบ HTML tags
    NSRegularExpression *htmlTagRegex = [NSRegularExpression 
                                         regularExpressionWithPattern:@"<[^>]+>" 
                                         options:NSRegularExpressionCaseInsensitive 
                                         error:nil];
    NSString *sanitized = [htmlTagRegex stringByReplacingMatchesInString:input 
                                                                 options:0 
                                                                   range:NSMakeRange(0, input.length) 
                                                            withTemplate:@""];
    
    // Trim whitespace
    sanitized = [sanitized stringByTrimmingCharactersInSet:[NSCharacterSet whitespaceAndNewlineCharacterSet]];
    
    return sanitized;
}

+ (BOOL)isValidURL:(NSString *)urlString {
    NSURL *url = [NSURL URLWithString:urlString];
    if (!url) return NO;
    
    // ตรวจสอบว่าเป็น HTTPS
    if (![url.scheme isEqualToString:@"https"]) {
        NSLog(@"⚠️ URL ไม่ใช่ HTTPS: %@", urlString);
        return NO;
    }
    
    return YES;
}

@end
```

---

# หมวดที่ 8: OWASP Mobile Top 10 Mitigations

## 77.15 OWASP Mobile Top 10 และการป้องกัน

```objc
// OWASPMitigations.m

/*
 OWASP Mobile Top 10 (2023):
 
 M1: Improper Credential Usage
 M2: Inadequate Supply Chain Security
 M3: Insecure Authentication/Authorization
 M4: Insufficient Input/Output Validation
 M5: Insecure Communication
 M6: Inadequate Privacy Controls
 M7: Insufficient Binary Protections
 M8: Security Misconfiguration
 M9: Insecure Data Storage
 M10: Insufficient Cryptography
*/

@interface OWASPMitigations : NSObject
@end

@implementation OWASPMitigations

// M1: Improper Credential Usage
// ✅ Mitigation: ใช้ Keychain เสมอ ห้าม Hard-code credentials
- (void)m1_credentialUsage {
    // ❌ ไม่ควร
    // NSString *apiKey = @"hardcoded_api_key";
    
    // ✅ ควร
    NSString *apiKey = [[KeychainManager sharedManager] passwordForKey:@"api_key" error:nil];
    NSLog(@"API Key: %@", apiKey ? @"Retrieved from Keychain" : @"Not set");
}

// M3: Insecure Authentication
// ✅ Mitigation: ใช้ Biometric + Keychain, Token Rotation
- (void)m3_authentication {
    LAContext *context = [[LAContext alloc] init];
    NSError *error = nil;
    
    if ([context canEvaluatePolicy:LAPolicyDeviceOwnerAuthenticationWithBiometrics error:&error]) {
        [context evaluatePolicy:LAPolicyDeviceOwnerAuthenticationWithBiometrics
                localizedReason:@"ยืนยันตัวตนเพื่อเข้าถึงข้อมูล"
                          reply:^(BOOL success, NSError *error) {
            if (success) {
                NSLog(@"✅ Authentication สำเร็จ");
            } else {
                NSLog(@"❌ Authentication ล้มเหลว: %@", error.localizedDescription);
            }
        }];
    }
}

// M4: Input Validation
// ✅ Mitigation: ตรวจสอบ Input ทุกครั้ง
- (void)m4_inputValidation:(NSString *)userInput {
    // Sanitize
    NSString *sanitized = [InputValidator sanitizeInput:userInput];
    
    // Validate length
    if (sanitized.length > 255) {
        NSLog(@"❌ Input ยาวเกินไป");
        return;
    }
    
    // ใช้ sanitized input เท่านั้น
    NSLog(@"✅ Sanitized input: %@", sanitized);
}

// M5: Insecure Communication
// ✅ Mitigation: ใช้ HTTPS + Certificate Pinning
- (void)m5_secureCommunication {
    // ดูตัวอย่าง Certificate Pinning ในหัวข้อ 77.9
    CertificatePinningManager *pinManager = [CertificatePinningManager sharedManager];
    NSURL *url = [NSURL URLWithString:@"https://api.example.com/data"];
    
    [pinManager fetchURL:url completion:^(NSData *data, NSURLResponse *response, NSError *error) {
        if (!error) {
            NSLog(@"✅ Secure communication with certificate pinning");
        }
    }];
}

// M9: Insecure Data Storage
// ✅ Mitigation: ใช้ Keychain สำหรับ Sensitive Data, Data Protection สำหรับไฟล์
- (void)m9_dataStorage {
    // Keychain สำหรับ credentials
    [[KeychainManager sharedManager] savePassword:@"user_token" 
                                           forKey:@"auth_token" 
                                            error:nil];
    
    // NSFileProtectionComplete สำหรับไฟล์ sensitive
    NSString *docsPath = [NSSearchPathForDirectoriesInDomains(NSDocumentDirectory, NSUserDomainMask, YES) firstObject];
    NSString *sensitiveFile = [docsPath stringByAppendingPathComponent:@"sensitive.dat"];
    
    [@"Sensitive Data" writeToFile:sensitiveFile 
                        atomically:YES 
                          encoding:NSUTF8StringEncoding 
                             error:nil];
    
    [[NSFileManager defaultManager] setAttributes:@{NSFileProtectionKey: NSFileProtectionComplete}
                                     ofItemAtPath:sensitiveFile 
                                            error:nil];
    
    NSLog(@"✅ Data stored securely");
}

// M10: Insufficient Cryptography
// ✅ Mitigation: ใช้ AES-256, SHA-256, PBKDF2 เสมอ
- (void)m10_cryptography {
    CryptoManager *crypto = [CryptoManager sharedManager];
    NSError *error = nil;
    
    // ✅ ใช้ AES-256 พร้อม PBKDF2 Key Derivation
    NSString *encrypted = [crypto encryptString:@"Sensitive Data" 
                                       password:@"StrongPassword123!" 
                                          error:&error];
    
    // ❌ อย่าใช้ MD5, SHA1, DES, RC4 - ล้าสมัยและไม่ปลอดภัย
    // NSLog(@"MD5: %@", [self md5Hash:data]); // ❌
    
    NSLog(@"✅ AES-256 encryption: %@", encrypted ? @"Success" : error.localizedDescription);
}

@end
```

---

# แบบฝึกหัดท้ายบท

## แบบฝึกหัดที่ 1: Secure Note Application

สร้าง SecureNoteManager ที่:
- เข้ารหัส Note ด้วย AES-256 ก่อนบันทึก
- ใช้ Keychain เก็บ Encryption Key
- ใช้ NSFileProtectionComplete สำหรับไฟล์
- ตรวจสอบ Integrity ด้วย HMAC

```objc
// แนวทางคำตอบ
@interface SecureNoteManager : NSObject

- (BOOL)saveNote:(NSString *)note 
          withId:(NSString *)noteId
           error:(NSError **)error;

- (NSString *)loadNoteWithId:(NSString *)noteId 
                       error:(NSError **)error;

- (BOOL)deleteNoteWithId:(NSString *)noteId 
                   error:(NSError **)error;

@end

@implementation SecureNoteManager {
    CryptoManager *_crypto;
    KeychainManager *_keychain;
}

- (instancetype)init {
    self = [super init];
    if (self) {
        _crypto = [CryptoManager sharedManager];
        _keychain = [KeychainManager sharedManager];
        [self ensureEncryptionKey];
    }
    return self;
}

- (void)ensureEncryptionKey {
    // ตรวจสอบว่ามี Key อยู่ใน Keychain หรือไม่
    NSError *error = nil;
    NSData *existingKey = [_keychain dataForKey:@"note_encryption_key" error:&error];
    
    if (!existingKey) {
        // สร้าง Key ใหม่
        NSData *newKey = [_crypto generateRandomKeyOfLength:32];
        [_keychain saveData:newKey forKey:@"note_encryption_key" error:&error];
        NSLog(@"✅ สร้าง Encryption Key ใหม่");
    }
}

- (BOOL)saveNote:(NSString *)note 
          withId:(NSString *)noteId
           error:(NSError **)error {
    
    // ดึง Key จาก Keychain
    NSError *keychainError = nil;
    NSData *key = [_keychain dataForKey:@"note_encryption_key" error:&keychainError];
    
    if (!key) {
        if (error) *error = keychainError;
        return NO;
    }
    
    // สร้าง IV
    NSData *iv = [_crypto generateRandomKeyOfLength:16];
    
    // เข้ารหัส Note
    NSData *noteData = [note dataUsingEncoding:NSUTF8StringEncoding];
    NSData *encryptedNote = [_crypto encryptData:noteData withKey:key iv:iv error:error];
    
    if (!encryptedNote) {
        return NO;
    }
    
    // คำนวณ HMAC สำหรับ Integrity Check
    NSData *hmac = [_crypto hmacSHA256:encryptedNote withKey:key];
    
    // รวมข้อมูล: iv (16) + hmac (32) + encryptedNote
    NSMutableData *combined = [NSMutableData data];
    [combined appendData:iv];
    [combined appendData:hmac];
    [combined appendData:encryptedNote];
    
    // บันทึกลงไฟล์พร้อม Protection
    NSString *docsPath = [NSSearchPathForDirectoriesInDomains(NSDocumentDirectory, NSUserDomainMask, YES) firstObject];
    NSString *notePath = [docsPath stringByAppendingPathComponent:[NSString stringWithFormat:@"note_%@.enc", noteId]];
    
    BOOL success = [combined writeToFile:notePath options:NSDataWritingAtomic error:error];
    
    if (success) {
        // ตั้งค่า File Protection
        [[NSFileManager defaultManager] setAttributes:@{NSFileProtectionKey: NSFileProtectionComplete}
                                         ofItemAtPath:notePath 
                                                error:nil];
        NSLog(@"✅ บันทึก Secure Note สำเร็จ: %@", noteId);
    }
    
    return success;
}

- (NSString *)loadNoteWithId:(NSString *)noteId 
                       error:(NSError **)error {
    
    NSString *docsPath = [NSSearchPathForDirectoriesInDomains(NSDocumentDirectory, NSUserDomainMask, YES) firstObject];
    NSString *notePath = [docsPath stringByAppendingPathComponent:[NSString stringWithFormat:@"note_%@.enc", noteId]];
    
    NSData *combined = [NSData dataWithContentsOfFile:notePath options:0 error:error];
    if (!combined || combined.length < 48) {
        return nil;
    }
    
    // แยกข้อมูล
    NSData *iv = [combined subdataWithRange:NSMakeRange(0, 16)];
    NSData *storedHmac = [combined subdataWithRange:NSMakeRange(16, 32)];
    NSData *encryptedNote = [combined subdataWithRange:NSMakeRange(48, combined.length - 48)];
    
    // ดึง Key
    NSData *key = [_keychain dataForKey:@"note_encryption_key" error:error];
    if (!key) return nil;
    
    // ตรวจสอบ Integrity
    if (![_crypto verifyHMACSHA256:storedHmac forData:encryptedNote withKey:key]) {
        if (error) {
            *error = [NSError errorWithDomain:@"SecureNoteError" 
                                         code:-1 
                                     userInfo:@{NSLocalizedDescriptionKey: @"Data Integrity Check ล้มเหลว!"}];
        }
        NSLog(@"❌ HMAC Verification ล้มเหลว - ข้อมูลอาจถูกแก้ไข!");
        return nil;
    }
    
    // ถอดรหัส
    NSData *noteData = [_crypto decryptData:encryptedNote withKey:key iv:iv error:error];
    if (!noteData) return nil;
    
    return [[NSString alloc] initWithData:noteData encoding:NSUTF8StringEncoding];
}

- (BOOL)deleteNoteWithId:(NSString *)noteId 
                   error:(NSError **)error {
    NSString *docsPath = [NSSearchPathForDirectoriesInDomains(NSDocumentDirectory, NSUserDomainMask, YES) firstObject];
    NSString *notePath = [docsPath stringByAppendingPathComponent:[NSString stringWithFormat:@"note_%@.enc", noteId]];
    
    return [[NSFileManager defaultManager] removeItemAtPath:notePath error:error];
}

@end
```

## แบบฝึกหัดที่ 2: Network Security Audit

เขียนฟังก์ชันที่ตรวจสอบ:
1. ATS Configuration ของแอป
2. ตรวจสอบว่า API Endpoints ทั้งหมดใช้ HTTPS
3. ตรวจสอบว่า Certificate ยังไม่หมดอายุ
4. สร้าง Security Report

```objc
// SecurityAuditor.m - ตรวจสอบความปลอดภัยของแอป
@interface SecurityAuditor : NSObject

- (void)runAuditWithCompletion:(void(^)(NSDictionary *report))completion;

@end

@implementation SecurityAuditor

- (void)runAuditWithCompletion:(void(^)(NSDictionary *report))completion {
    NSMutableDictionary *report = [NSMutableDictionary dictionary];
    dispatch_group_t group = dispatch_group_create();
    
    // 1. ตรวจสอบ ATS
    report[@"ats_configured"] = @([self checkATSConfiguration]);
    
    // 2. ตรวจสอบ Jailbreak
    report[@"jailbreak_detected"] = @([JailbreakDetector isDeviceJailbroken]);
    
    // 3. ตรวจสอบ Keychain
    report[@"keychain_accessible"] = @([self checkKeychainAccessibility]);
    
    // 4. สรุปผล
    report[@"audit_date"] = [NSDate date].description;
    report[@"overall_status"] = [self calculateOverallStatus:report];
    
    if (completion) {
        completion([report copy]);
    }
}

- (BOOL)checkATSConfiguration {
    NSDictionary *ats = [[NSBundle mainBundle] infoDictionary][@"NSAppTransportSecurity"];
    if (!ats) return YES; // ไม่มี exception = ปลอดภัย
    
    BOOL allowArbitrary = [ats[@"NSAllowsArbitraryLoads"] boolValue];
    return !allowArbitrary;
}

- (BOOL)checkKeychainAccessibility {
    NSError *error = nil;
    BOOL success = [[KeychainManager sharedManager] savePassword:@"test" 
                                                          forKey:@"audit_test" 
                                                           error:&error];
    [[KeychainManager sharedManager] deletePasswordForKey:@"audit_test" error:nil];
    return success;
}

- (NSString *)calculateOverallStatus:(NSDictionary *)report {
    BOOL jailbroken = [report[@"jailbreak_detected"] boolValue];
    BOOL atsOk = [report[@"ats_configured"] boolValue];
    
    if (jailbroken) return @"CRITICAL";
    if (!atsOk) return @"WARNING";
    return @"PASS";
}

@end
```

## แบบฝึกหัดที่ 3: Password Manager

สร้าง Simple Password Manager ที่ใช้ Keychain เก็บรหัสผ่านหลายรายการ

```objc
@interface PasswordEntry : NSObject
@property (nonatomic, copy) NSString *service;
@property (nonatomic, copy) NSString *username;
@property (nonatomic, copy) NSString *password;
@property (nonatomic, copy) NSDate *createdAt;
@property (nonatomic, copy) NSDate *lastModified;
@end

@implementation PasswordEntry

- (NSData *)toData {
    NSDictionary *dict = @{
        @"service": self.service ?: @"",
        @"username": self.username ?: @"",
        @"password": self.password ?: @"",
        @"createdAt": self.createdAt ? [NSString stringWithFormat:@"%f", self.createdAt.timeIntervalSince1970] : @"0",
        @"lastModified": self.lastModified ? [NSString stringWithFormat:@"%f", self.lastModified.timeIntervalSince1970] : @"0",
    };
    return [NSJSONSerialization dataWithJSONObject:dict options:0 error:nil];
}

+ (instancetype)fromData:(NSData *)data {
    NSDictionary *dict = [NSJSONSerialization JSONObjectWithData:data options:0 error:nil];
    if (!dict) return nil;
    
    PasswordEntry *entry = [[PasswordEntry alloc] init];
    entry.service = dict[@"service"];
    entry.username = dict[@"username"];
    entry.password = dict[@"password"];
    entry.createdAt = [NSDate dateWithTimeIntervalSince1970:[dict[@"createdAt"] doubleValue]];
    entry.lastModified = [NSDate dateWithTimeIntervalSince1970:[dict[@"lastModified"] doubleValue]];
    return entry;
}

@end

@interface SimplePasswordManager : NSObject

- (BOOL)addEntry:(PasswordEntry *)entry error:(NSError **)error;
- (PasswordEntry *)getEntryForService:(NSString *)service error:(NSError **)error;
- (BOOL)updateEntry:(PasswordEntry *)entry error:(NSError **)error;
- (BOOL)deleteEntryForService:(NSString *)service error:(NSError **)error;
- (NSArray<PasswordEntry *> *)allEntries;

@end

@implementation SimplePasswordManager {
    KeychainManager *_keychain;
    NSMutableArray<NSString *> *_serviceIndex;
}

- (instancetype)init {
    self = [super init];
    if (self) {
        _keychain = [KeychainManager sharedManager];
        _serviceIndex = [NSMutableArray array];
        [self loadServiceIndex];
    }
    return self;
}

- (void)loadServiceIndex {
    NSError *error = nil;
    NSData *indexData = [_keychain dataForKey:@"pm_service_index" error:&error];
    if (indexData) {
        NSArray *index = [NSJSONSerialization JSONObjectWithData:indexData options:0 error:nil];
        if (index) {
            [_serviceIndex addObjectsFromArray:index];
        }
    }
}

- (void)saveServiceIndex {
    NSData *indexData = [NSJSONSerialization dataWithJSONObject:[_serviceIndex copy] options:0 error:nil];
    [_keychain saveData:indexData forKey:@"pm_service_index" error:nil];
}

- (BOOL)addEntry:(PasswordEntry *)entry error:(NSError **)error {
    entry.createdAt = [NSDate date];
    entry.lastModified = [NSDate date];
    
    NSString *key = [NSString stringWithFormat:@"pm_%@", entry.service];
    BOOL success = [_keychain saveData:[entry toData] forKey:key error:error];
    
    if (success) {
        [_serviceIndex addObject:entry.service];
        [self saveServiceIndex];
    }
    
    return success;
}

- (PasswordEntry *)getEntryForService:(NSString *)service error:(NSError **)error {
    NSString *key = [NSString stringWithFormat:@"pm_%@", service];
    NSData *data = [_keychain dataForKey:key error:error];
    if (!data) return nil;
    return [PasswordEntry fromData:data];
}

- (BOOL)updateEntry:(PasswordEntry *)entry error:(NSError **)error {
    entry.lastModified = [NSDate date];
    NSString *key = [NSString stringWithFormat:@"pm_%@", entry.service];
    return [_keychain saveData:[entry toData] forKey:key error:error];
}

- (BOOL)deleteEntryForService:(NSString *)service error:(NSError **)error {
    NSString *key = [NSString stringWithFormat:@"pm_%@", service];
    BOOL success = [_keychain deletePasswordForKey:key error:error];
    
    if (success) {
        [_serviceIndex removeObject:service];
        [self saveServiceIndex];
    }
    
    return success;
}

- (NSArray<PasswordEntry *> *)allEntries {
    NSMutableArray *entries = [NSMutableArray array];
    for (NSString *service in _serviceIndex) {
        PasswordEntry *entry = [self getEntryForService:service error:nil];
        if (entry) {
            [entries addObject:entry];
        }
    }
    return [entries copy];
}

@end
```

---

## สรุปบทที่ 77

| หัวข้อ | สิ่งสำคัญที่ต้องจำ |
|--------|-------------------|
| ATS | บังคับ HTTPS, ไม่ใช้ NSAllowsArbitraryLoads |
| Keychain | ใช้เก็บ Password/Token แทน NSUserDefaults |
| AES-256 | ใช้ PBKDF2 Derive Key, Random Salt/IV |
| Certificate Pinning | ป้องกัน MITM, ใช้ Public Key Pinning |
| Jailbreak Detection | ตรวจหลายวิธี, ไม่มีวิธีเดียวที่สมบูรณ์ |
| Data Protection | NSFileProtectionComplete สำหรับข้อมูลสำคัญ |
| Secure Coding | Input Validation, ไม่ Log ข้อมูลสำคัญ |
| OWASP Top 10 | ทำความเข้าใจและป้องกันทุกข้อ |

> **คำแนะนำ**: ความปลอดภัยไม่ใช่สิ่งที่ add-on ทีหลัง ต้องคิดตั้งแต่ออกแบบ Architecture พิจารณาใช้ Security Review ก่อน Release ทุกครั้ง
