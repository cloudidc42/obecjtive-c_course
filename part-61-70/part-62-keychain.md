# Part 62 - Keychain Services

## บทนำ

Keychain เป็นระบบจัดเก็บข้อมูลที่ปลอดภัยที่สุดบน iOS และ macOS ออกแบบมาเพื่อเก็บ passwords, cryptographic keys, certificates และข้อมูลสำคัญอื่นๆ ข้อมูลใน Keychain ถูกเข้ารหัสด้วยฮาร์ดแวร์และมีการควบคุมการเข้าถึงอย่างเข้มงวด

ในบทนี้เราจะเรียนรู้วิธีใช้ Keychain Services API ในการจัดการข้อมูลที่ปลอดภัยบน iOS

---

## 62.1 Security Framework

ก่อนใช้ Keychain ต้อง import Security framework:

```objc
#import <Security/Security.h>
```

และเพิ่ม Security.framework ใน Build Phases → Link Binary With Libraries

### ทำไมต้องใช้ Keychain?

ข้อมูลที่ควรเก็บใน Keychain:
- Passwords และ PINs
- Access tokens (OAuth tokens)
- API keys
- Cryptographic private keys
- Certificates

ข้อมูลที่ไม่ควรเก็บใน Keychain:
- ข้อมูลทั่วไปของ user (ชื่อ, ที่อยู่) → ใช้ UserDefaults หรือ CoreData
- ไฟล์ขนาดใหญ่ → เก็บไว้ใน Documents directory แบบ encrypted
- App preferences → ใช้ UserDefaults

### เปรียบเทียบกับ UserDefaults

| คุณสมบัติ | UserDefaults | Keychain |
|----------|--------------|----------|
| ความปลอดภัย | ต่ำ (เก็บเป็น plist) | สูงมาก (encrypted) |
| ขนาดข้อมูล | ไม่จำกัด | เล็ก (~1KB/item) |
| Sharing | App เดียวกัน | แชร์ระหว่าง apps ได้ |
| Backup | iCloud backup | ควบคุมได้ |
| Device wipe | ลบหมด | ลบหมด |
| App delete | ลบหมด | คงอยู่ (ถ้าตั้งค่า) |

---

## 62.2 Keychain Services API

### Core Functions

```objc
// เพิ่ม item ใหม่
OSStatus SecItemAdd(CFDictionaryRef attributes, CFTypeRef *result);

// ค้นหา item
OSStatus SecItemCopyMatching(CFDictionaryRef query, CFTypeRef *result);

// อัปเดต item ที่มีอยู่
OSStatus SecItemUpdate(CFDictionaryRef query, CFDictionaryRef attributesToUpdate);

// ลบ item
OSStatus SecItemDelete(CFDictionaryRef query);
```

### Keychain Item Classes

```objc
// kSecClassGenericPassword  - passwords ทั่วไป
// kSecClassInternetPassword - passwords สำหรับ internet services
// kSecClassCertificate      - certificates
// kSecClassKey              - cryptographic keys
// kSecClassIdentity         - identity (certificate + private key)
```

---

## 62.3 การเก็บ Generic Passwords (kSecClassGenericPassword)

Generic password ใช้สำหรับเก็บ password ที่ไม่ได้เชื่อมกับ URL ใด เช่น PIN ของ app หรือ API token

### KeychainHelper Class

```objc
// KeychainHelper.h
#import <Foundation/Foundation.h>
#import <Security/Security.h>

typedef NS_ENUM(NSInteger, KeychainAccessibility) {
    KeychainAccessibleWhenUnlocked,           // เข้าถึงได้เฉพาะตอนปลดล็อก
    KeychainAccessibleAfterFirstUnlock,       // เข้าถึงได้หลัง unlock ครั้งแรก (background ok)
    KeychainAccessibleAlways,                 // เข้าถึงได้ตลอด (ไม่แนะนำ)
    KeychainAccessibleWhenPasscodeSetThisDeviceOnly, // ต้องมี passcode + เฉพาะ device นี้
    KeychainAccessibleWhenUnlockedThisDeviceOnly,    // เฉพาะ device นี้
    KeychainAccessibleAfterFirstUnlockThisDeviceOnly // เฉพาะ device นี้ + หลัง unlock ครั้งแรก
};

@interface KeychainHelper : NSObject

+ (instancetype)sharedHelper;

// Generic Password
- (BOOL)savePassword:(NSString *)password 
             forKey:(NSString *)key 
            service:(NSString *)service;

- (NSString *)passwordForKey:(NSString *)key 
                     service:(NSString *)service;

- (BOOL)updatePassword:(NSString *)password 
               forKey:(NSString *)key 
              service:(NSString *)service;

- (BOOL)deletePasswordForKey:(NSString *)key 
                     service:(NSString *)service;

// Data
- (BOOL)saveData:(NSData *)data 
          forKey:(NSString *)key 
         service:(NSString *)service;

- (NSData *)dataForKey:(NSString *)key 
               service:(NSString *)service;

// Convenience
- (BOOL)setString:(NSString *)string forKey:(NSString *)key;
- (NSString *)stringForKey:(NSString *)key;
- (BOOL)deleteStringForKey:(NSString *)key;

@end
```

```objc
// KeychainHelper.m
#import "KeychainHelper.h"

static NSString *const kDefaultService = @"com.myapp.keychain";

@implementation KeychainHelper

+ (instancetype)sharedHelper {
    static KeychainHelper *instance = nil;
    static dispatch_once_t onceToken;
    dispatch_once(&onceToken, ^{
        instance = [[KeychainHelper alloc] init];
    });
    return instance;
}

// สร้าง base dictionary สำหรับ query
- (NSMutableDictionary *)baseQueryForKey:(NSString *)key service:(NSString *)service {
    return [@{
        (__bridge id)kSecClass: (__bridge id)kSecClassGenericPassword,
        (__bridge id)kSecAttrAccount: key,
        (__bridge id)kSecAttrService: service ?: kDefaultService
    } mutableCopy];
}

- (BOOL)savePassword:(NSString *)password 
             forKey:(NSString *)key 
            service:(NSString *)service {
    
    NSData *passwordData = [password dataUsingEncoding:NSUTF8StringEncoding];
    return [self saveData:passwordData forKey:key service:service];
}

- (BOOL)saveData:(NSData *)data 
          forKey:(NSString *)key 
         service:(NSString *)service {
    
    // ลบ item เดิมก่อน (ถ้ามี)
    [self deletePasswordForKey:key service:service];
    
    NSMutableDictionary *query = [self baseQueryForKey:key service:service];
    query[(__bridge id)kSecValueData] = data;
    // กำหนด accessibility
    query[(__bridge id)kSecAttrAccessible] = (__bridge id)kSecAttrAccessibleWhenUnlockedThisDeviceOnly;
    
    OSStatus status = SecItemAdd((__bridge CFDictionaryRef)query, NULL);
    
    if (status != errSecSuccess) {
        NSLog(@"Keychain save failed for key '%@': %d", key, (int)status);
        return NO;
    }
    
    return YES;
}

- (NSString *)passwordForKey:(NSString *)key service:(NSString *)service {
    NSData *data = [self dataForKey:key service:service];
    if (!data) return nil;
    return [[NSString alloc] initWithData:data encoding:NSUTF8StringEncoding];
}

- (NSData *)dataForKey:(NSString *)key service:(NSString *)service {
    NSMutableDictionary *query = [self baseQueryForKey:key service:service];
    query[(__bridge id)kSecReturnData] = @YES;
    query[(__bridge id)kSecMatchLimit] = (__bridge id)kSecMatchLimitOne;
    
    CFTypeRef result = NULL;
    OSStatus status = SecItemCopyMatching((__bridge CFDictionaryRef)query, &result);
    
    if (status != errSecSuccess) {
        if (status != errSecItemNotFound) {
            NSLog(@"Keychain read failed for key '%@': %d", key, (int)status);
        }
        return nil;
    }
    
    NSData *data = (__bridge_transfer NSData *)result;
    return data;
}

- (BOOL)updatePassword:(NSString *)password 
               forKey:(NSString *)key 
              service:(NSString *)service {
    
    NSMutableDictionary *query = [self baseQueryForKey:key service:service];
    
    NSData *passwordData = [password dataUsingEncoding:NSUTF8StringEncoding];
    NSDictionary *attributes = @{
        (__bridge id)kSecValueData: passwordData
    };
    
    OSStatus status = SecItemUpdate((__bridge CFDictionaryRef)query, 
                                    (__bridge CFDictionaryRef)attributes);
    
    if (status == errSecItemNotFound) {
        // Item ไม่มี ให้ create ใหม่แทน
        return [self savePassword:password forKey:key service:service];
    }
    
    return status == errSecSuccess;
}

- (BOOL)deletePasswordForKey:(NSString *)key service:(NSString *)service {
    NSMutableDictionary *query = [self baseQueryForKey:key service:service];
    
    OSStatus status = SecItemDelete((__bridge CFDictionaryRef)query);
    return (status == errSecSuccess || status == errSecItemNotFound);
}

// Convenience methods ที่ใช้ bundle identifier เป็น service
- (BOOL)setString:(NSString *)string forKey:(NSString *)key {
    NSString *service = [[NSBundle mainBundle] bundleIdentifier];
    return [self savePassword:string forKey:key service:service];
}

- (NSString *)stringForKey:(NSString *)key {
    NSString *service = [[NSBundle mainBundle] bundleIdentifier];
    return [self passwordForKey:key service:service];
}

- (BOOL)deleteStringForKey:(NSString *)key {
    NSString *service = [[NSBundle mainBundle] bundleIdentifier];
    return [self deletePasswordForKey:key service:service];
}

@end
```

### ตัวอย่างการใช้งาน

```objc
KeychainHelper *keychain = [KeychainHelper sharedHelper];

// บันทึก password
[keychain savePassword:@"mySecretPassword123" 
               forKey:@"user_password" 
              service:@"com.myapp.auth"];

// อ่าน password
NSString *password = [keychain passwordForKey:@"user_password" 
                                      service:@"com.myapp.auth"];
NSLog(@"Password: %@", password);

// อัปเดต password
[keychain updatePassword:@"newPassword456" 
                 forKey:@"user_password" 
                service:@"com.myapp.auth"];

// ลบ password
[keychain deletePasswordForKey:@"user_password" 
                       service:@"com.myapp.auth"];

// Convenience API
[[KeychainHelper sharedHelper] setString:@"abc123token" forKey:@"auth_token"];
NSString *token = [[KeychainHelper sharedHelper] stringForKey:@"auth_token"];
```

---

## 62.4 Internet Passwords (kSecClassInternetPassword)

Internet passwords ใช้สำหรับเก็บ credentials ที่เชื่อมกับ URL เฉพาะ

```objc
@interface InternetPasswordHelper : NSObject

- (BOOL)savePassword:(NSString *)password
         forUsername:(NSString *)username
                 url:(NSURL *)url;

- (NSString *)passwordForUsername:(NSString *)username
                               url:(NSURL *)url;

- (BOOL)deletePasswordForUsername:(NSString *)username
                               url:(NSURL *)url;

@end

@implementation InternetPasswordHelper

- (NSMutableDictionary *)queryForUsername:(NSString *)username url:(NSURL *)url {
    NSMutableDictionary *query = [@{
        (__bridge id)kSecClass: (__bridge id)kSecClassInternetPassword,
        (__bridge id)kSecAttrAccount: username,
        (__bridge id)kSecAttrServer: url.host ?: @"",
        (__bridge id)kSecAttrPath: url.path ?: @"",
        (__bridge id)kSecAttrPort: url.port ?: @(443),
        (__bridge id)kSecAttrProtocol: url.scheme.lowercaseString
    } mutableCopy];
    
    // กำหนด protocol ที่ถูกต้อง
    if ([url.scheme.lowercaseString isEqualToString:@"https"]) {
        query[(__bridge id)kSecAttrProtocol] = (__bridge id)kSecAttrProtocolHTTPS;
    } else {
        query[(__bridge id)kSecAttrProtocol] = (__bridge id)kSecAttrProtocolHTTP;
    }
    
    return query;
}

- (BOOL)savePassword:(NSString *)password
         forUsername:(NSString *)username
                 url:(NSURL *)url {
    
    [self deletePasswordForUsername:username url:url];
    
    NSMutableDictionary *query = [self queryForUsername:username url:url];
    query[(__bridge id)kSecValueData] = [password dataUsingEncoding:NSUTF8StringEncoding];
    query[(__bridge id)kSecAttrAccessible] = (__bridge id)kSecAttrAccessibleWhenUnlocked;
    
    OSStatus status = SecItemAdd((__bridge CFDictionaryRef)query, NULL);
    return status == errSecSuccess;
}

- (NSString *)passwordForUsername:(NSString *)username url:(NSURL *)url {
    NSMutableDictionary *query = [self queryForUsername:username url:url];
    query[(__bridge id)kSecReturnData] = @YES;
    query[(__bridge id)kSecMatchLimit] = (__bridge id)kSecMatchLimitOne;
    
    CFTypeRef result = NULL;
    OSStatus status = SecItemCopyMatching((__bridge CFDictionaryRef)query, &result);
    
    if (status != errSecSuccess) return nil;
    
    NSData *data = (__bridge_transfer NSData *)result;
    return [[NSString alloc] initWithData:data encoding:NSUTF8StringEncoding];
}

- (BOOL)deletePasswordForUsername:(NSString *)username url:(NSURL *)url {
    NSMutableDictionary *query = [self queryForUsername:username url:url];
    OSStatus status = SecItemDelete((__bridge CFDictionaryRef)query);
    return (status == errSecSuccess || status == errSecItemNotFound);
}

@end

// การใช้งาน
InternetPasswordHelper *helper = [[InternetPasswordHelper alloc] init];
NSURL *url = [NSURL URLWithString:@"https://api.example.com/v1"];

[helper savePassword:@"myPassword" 
         forUsername:@"john@example.com" 
                 url:url];

NSString *pw = [helper passwordForUsername:@"john@example.com" url:url];
NSLog(@"Retrieved password: %@", pw);
```

---

## 62.5 การเก็บ Certificates

```objc
// นำเข้า certificate จาก DER-encoded data
- (SecCertificateRef)importCertificateFromData:(NSData *)certData {
    SecCertificateRef cert = SecCertificateCreateWithData(NULL, (__bridge CFDataRef)certData);
    if (!cert) {
        NSLog(@"Failed to create certificate from data");
        return NULL;
    }
    return cert;
}

// บันทึก certificate ลง Keychain
- (BOOL)saveCertificate:(SecCertificateRef)certificate withLabel:(NSString *)label {
    NSDictionary *query = @{
        (__bridge id)kSecClass: (__bridge id)kSecClassCertificate,
        (__bridge id)kSecValueRef: (__bridge id)certificate,
        (__bridge id)kSecAttrLabel: label,
        (__bridge id)kSecAttrAccessible: (__bridge id)kSecAttrAccessibleWhenUnlocked
    };
    
    OSStatus status = SecItemAdd((__bridge CFDictionaryRef)query, NULL);
    
    if (status == errSecDuplicateItem) {
        NSLog(@"Certificate already exists in keychain");
        return YES; // ถือว่าสำเร็จ
    }
    
    return status == errSecSuccess;
}

// อ่าน certificate จาก Keychain
- (SecCertificateRef)certificateWithLabel:(NSString *)label {
    NSDictionary *query = @{
        (__bridge id)kSecClass: (__bridge id)kSecClassCertificate,
        (__bridge id)kSecAttrLabel: label,
        (__bridge id)kSecReturnRef: @YES,
        (__bridge id)kSecMatchLimit: (__bridge id)kSecMatchLimitOne
    };
    
    CFTypeRef result = NULL;
    OSStatus status = SecItemCopyMatching((__bridge CFDictionaryRef)query, &result);
    
    if (status != errSecSuccess) {
        NSLog(@"Failed to retrieve certificate: %d", (int)status);
        return NULL;
    }
    
    return (SecCertificateRef)result;
}
```

---

## 62.6 การเก็บ Cryptographic Keys

```objc
// สร้างและบันทึก RSA key pair
- (BOOL)generateRSAKeyPairWithTag:(NSString *)tag keySize:(int)keySize {
    NSData *tagData = [tag dataUsingEncoding:NSUTF8StringEncoding];
    
    // ลบ key เดิมถ้ามี
    NSDictionary *deleteQuery = @{
        (__bridge id)kSecClass: (__bridge id)kSecClassKey,
        (__bridge id)kSecAttrApplicationTag: tagData,
    };
    SecItemDelete((__bridge CFDictionaryRef)deleteQuery);
    
    // สร้าง key pair
    NSDictionary *privateKeyAttrs = @{
        (__bridge id)kSecAttrIsPermanent: @YES,
        (__bridge id)kSecAttrApplicationTag: tagData,
        (__bridge id)kSecAttrAccessible: (__bridge id)kSecAttrAccessibleWhenUnlockedThisDeviceOnly
    };
    
    NSDictionary *publicKeyAttrs = @{
        (__bridge id)kSecAttrIsPermanent: @YES,
        (__bridge id)kSecAttrApplicationTag: [tag stringByAppendingString:@".public"]
    };
    
    NSDictionary *keyAttributes = @{
        (__bridge id)kSecAttrKeyType: (__bridge id)kSecAttrKeyTypeRSA,
        (__bridge id)kSecAttrKeySizeInBits: @(keySize),
        (__bridge id)kSecPrivateKeyAttrs: privateKeyAttrs,
        (__bridge id)kSecPublicKeyAttrs: publicKeyAttrs
    };
    
    SecKeyRef publicKey = NULL;
    SecKeyRef privateKey = NULL;
    
    OSStatus status = SecKeyGeneratePair((__bridge CFDictionaryRef)keyAttributes,
                                          &publicKey, &privateKey);
    
    if (publicKey) CFRelease(publicKey);
    if (privateKey) CFRelease(privateKey);
    
    if (status != errSecSuccess) {
        NSLog(@"Failed to generate key pair: %d", (int)status);
        return NO;
    }
    
    NSLog(@"RSA key pair generated successfully");
    return YES;
}

// ดึง private key จาก Keychain
- (SecKeyRef)privateKeyWithTag:(NSString *)tag {
    NSData *tagData = [tag dataUsingEncoding:NSUTF8StringEncoding];
    
    NSDictionary *query = @{
        (__bridge id)kSecClass: (__bridge id)kSecClassKey,
        (__bridge id)kSecAttrKeyClass: (__bridge id)kSecAttrKeyClassPrivate,
        (__bridge id)kSecAttrApplicationTag: tagData,
        (__bridge id)kSecReturnRef: @YES,
        (__bridge id)kSecMatchLimit: (__bridge id)kSecMatchLimitOne
    };
    
    CFTypeRef result = NULL;
    OSStatus status = SecItemCopyMatching((__bridge CFDictionaryRef)query, &result);
    
    if (status != errSecSuccess) {
        NSLog(@"Failed to retrieve private key: %d", (int)status);
        return NULL;
    }
    
    return (SecKeyRef)result;
}

// เข้ารหัสด้วย RSA Public Key
- (NSData *)encryptData:(NSData *)data withPublicKey:(SecKeyRef)publicKey {
    size_t bufferSize = SecKeyGetBlockSize(publicKey);
    NSMutableData *buffer = [NSMutableData dataWithLength:bufferSize];
    
    OSStatus status = SecKeyEncrypt(
        publicKey,
        kSecPaddingPKCS1,
        (const uint8_t *)[data bytes],
        [data length],
        (uint8_t *)[buffer mutableBytes],
        &bufferSize
    );
    
    if (status != errSecSuccess) {
        NSLog(@"Encryption failed: %d", (int)status);
        return nil;
    }
    
    return [NSData dataWithBytes:[buffer bytes] length:bufferSize];
}
```

---

## 62.7 Keychain Access Groups

Keychain Access Groups ทำให้หลาย apps แชร์ keychain items ได้ โดยต้องมี App ID Prefix (Team ID) เดียวกัน

### การตั้งค่า Entitlements

ใน `MyApp.entitlements`:
```xml
<?xml version="1.0" encoding="UTF-8"?>
<!DOCTYPE plist PUBLIC "-//Apple//DTD PLIST 1.0//EN" "http://www.apple.com/DTDs/PropertyList-1.0.dtd">
<plist version="1.0">
<dict>
    <key>keychain-access-groups</key>
    <array>
        <string>$(AppIdentifierPrefix)com.mycompany.shared</string>
        <string>$(AppIdentifierPrefix)com.myapp.private</string>
    </array>
</dict>
</plist>
```

### การใช้ Access Group ใน Code

```objc
// บันทึก item ที่แชร์ได้กับ apps อื่น
- (BOOL)saveSharedPassword:(NSString *)password forKey:(NSString *)key {
    // ลบ item เดิมก่อน
    NSDictionary *deleteQuery = @{
        (__bridge id)kSecClass: (__bridge id)kSecClassGenericPassword,
        (__bridge id)kSecAttrAccount: key,
        (__bridge id)kSecAttrAccessGroup: @"TEAMID.com.mycompany.shared"
    };
    SecItemDelete((__bridge CFDictionaryRef)deleteQuery);
    
    // บันทึก item ใหม่พร้อม access group
    NSDictionary *addQuery = @{
        (__bridge id)kSecClass: (__bridge id)kSecClassGenericPassword,
        (__bridge id)kSecAttrAccount: key,
        (__bridge id)kSecAttrAccessGroup: @"TEAMID.com.mycompany.shared",
        (__bridge id)kSecValueData: [password dataUsingEncoding:NSUTF8StringEncoding],
        (__bridge id)kSecAttrAccessible: (__bridge id)kSecAttrAccessibleWhenUnlocked
    };
    
    OSStatus status = SecItemAdd((__bridge CFDictionaryRef)addQuery, NULL);
    return status == errSecSuccess;
}

// อ่าน item จาก shared access group
- (NSString *)sharedPasswordForKey:(NSString *)key {
    NSDictionary *query = @{
        (__bridge id)kSecClass: (__bridge id)kSecClassGenericPassword,
        (__bridge id)kSecAttrAccount: key,
        (__bridge id)kSecAttrAccessGroup: @"TEAMID.com.mycompany.shared",
        (__bridge id)kSecReturnData: @YES,
        (__bridge id)kSecMatchLimit: (__bridge id)kSecMatchLimitOne
    };
    
    CFTypeRef result = NULL;
    OSStatus status = SecItemCopyMatching((__bridge CFDictionaryRef)query, &result);
    
    if (status != errSecSuccess) return nil;
    
    NSData *data = (__bridge_transfer NSData *)result;
    return [[NSString alloc] initWithData:data encoding:NSUTF8StringEncoding];
}
```

---

## 62.8 Keychain Sharing กับ App Extensions

App Extensions (Today Widget, Keyboard extension ฯลฯ) ต้องใช้ Access Groups เพื่อแชร์ Keychain กับ main app:

```objc
// ใน main app: บันทึก auth token สำหรับแชร์กับ extension
- (void)saveAuthTokenForExtension:(NSString *)token {
    NSString *groupId = @"group.com.mycompany.myapp"; // App Group
    
    NSDictionary *query = @{
        (__bridge id)kSecClass: (__bridge id)kSecClassGenericPassword,
        (__bridge id)kSecAttrAccount: @"auth_token",
        (__bridge id)kSecAttrService: @"com.mycompany.myapp",
        (__bridge id)kSecAttrAccessGroup: @"TEAMID.com.mycompany.shared",
        (__bridge id)kSecValueData: [token dataUsingEncoding:NSUTF8StringEncoding],
        (__bridge id)kSecAttrAccessible: (__bridge id)kSecAttrAccessibleAfterFirstUnlock
    };
    
    // ลบเดิมก่อน
    NSMutableDictionary *deleteQuery = [query mutableCopy];
    [deleteQuery removeObjectForKey:(__bridge id)kSecValueData];
    SecItemDelete((__bridge CFDictionaryRef)deleteQuery);
    
    SecItemAdd((__bridge CFDictionaryRef)query, NULL);
}

// ใน extension: อ่าน auth token
- (NSString *)readAuthTokenFromMainApp {
    NSDictionary *query = @{
        (__bridge id)kSecClass: (__bridge id)kSecClassGenericPassword,
        (__bridge id)kSecAttrAccount: @"auth_token",
        (__bridge id)kSecAttrService: @"com.mycompany.myapp",
        (__bridge id)kSecAttrAccessGroup: @"TEAMID.com.mycompany.shared",
        (__bridge id)kSecReturnData: @YES,
        (__bridge id)kSecMatchLimit: (__bridge id)kSecMatchLimitOne
    };
    
    CFTypeRef result = NULL;
    OSStatus status = SecItemCopyMatching((__bridge CFDictionaryRef)query, &result);
    
    if (status != errSecSuccess) return nil;
    
    NSData *data = (__bridge_transfer NSData *)result;
    return [[NSString alloc] initWithData:data encoding:NSUTF8StringEncoding];
}
```

---

## 62.9 Biometric Authentication (Touch ID / Face ID)

iOS รองรับการใช้ biometric authentication ร่วมกับ Keychain เพื่อเพิ่มความปลอดภัย

### การตั้งค่า Local Authentication

```objc
#import <LocalAuthentication/LocalAuthentication.h>

// ตรวจสอบว่า device รองรับ biometric หรือไม่
- (BOOL)isBiometricAvailable {
    LAContext *context = [[LAContext alloc] init];
    NSError *error;
    return [context canEvaluatePolicy:LAPolicyDeviceOwnerAuthenticationWithBiometrics error:&error];
}

// ประเภท biometric
- (NSString *)biometricType {
    LAContext *context = [[LAContext alloc] init];
    NSError *error;
    
    if (![context canEvaluatePolicy:LAPolicyDeviceOwnerAuthenticationWithBiometrics error:&error]) {
        return @"Not Available";
    }
    
    if (@available(iOS 11.0, *)) {
        switch (context.biometryType) {
            case LABiometryTypeFaceID:
                return @"Face ID";
            case LABiometryTypeTouchID:
                return @"Touch ID";
            default:
                return @"None";
        }
    }
    return @"Touch ID"; // iOS 10 และต่ำกว่า
}
```

### การบันทึก Password ที่ต้องใช้ Biometric

```objc
#import <LocalAuthentication/LocalAuthentication.h>

// บันทึก item ที่ต้องการ Touch ID หรือ Face ID ในการเข้าถึง
- (BOOL)saveBiometricProtectedPassword:(NSString *)password forKey:(NSString *)key {
    // สร้าง Access Control ที่ต้องการ biometric
    CFErrorRef accessError = NULL;
    SecAccessControlRef accessControl = SecAccessControlCreateWithFlags(
        NULL,
        kSecAttrAccessibleWhenUnlockedThisDeviceOnly,
        kSecAccessControlBiometryCurrentSet, // ต้องการ biometric ที่ลงทะเบียนปัจจุบัน
        &accessError
    );
    
    if (accessError) {
        NSLog(@"Failed to create access control: %@", (__bridge NSError *)accessError);
        if (accessControl) CFRelease(accessControl);
        return NO;
    }
    
    LAContext *context = [[LAContext alloc] init];
    
    NSDictionary *query = @{
        (__bridge id)kSecClass: (__bridge id)kSecClassGenericPassword,
        (__bridge id)kSecAttrAccount: key,
        (__bridge id)kSecAttrService: @"BiometricApp",
        (__bridge id)kSecValueData: [password dataUsingEncoding:NSUTF8StringEncoding],
        (__bridge id)kSecAttrAccessControl: (__bridge id)accessControl,
        (__bridge id)kSecUseAuthenticationContext: context
    };
    
    // ลบ item เดิมก่อน
    NSMutableDictionary *deleteQuery = @{
        (__bridge id)kSecClass: (__bridge id)kSecClassGenericPassword,
        (__bridge id)kSecAttrAccount: key,
        (__bridge id)kSecAttrService: @"BiometricApp"
    }.mutableCopy;
    SecItemDelete((__bridge CFDictionaryRef)deleteQuery);
    
    OSStatus status = SecItemAdd((__bridge CFDictionaryRef)query, NULL);
    
    CFRelease(accessControl);
    
    return status == errSecSuccess;
}

// อ่าน password ที่ต้องใช้ biometric (แสดง prompt อัตโนมัติ)
- (void)readBiometricProtectedPassword:(NSString *)key 
                            completion:(void (^)(NSString *password, NSError *error))completion {
    
    LAContext *context = [[LAContext alloc] init];
    context.localizedReason = [NSString stringWithFormat:@"ยืนยันตัวตนเพื่อเข้าถึง '%@'", key];
    
    NSDictionary *query = @{
        (__bridge id)kSecClass: (__bridge id)kSecClassGenericPassword,
        (__bridge id)kSecAttrAccount: key,
        (__bridge id)kSecAttrService: @"BiometricApp",
        (__bridge id)kSecReturnData: @YES,
        (__bridge id)kSecMatchLimit: (__bridge id)kSecMatchLimitOne,
        (__bridge id)kSecUseAuthenticationContext: context,
        (__bridge id)kSecUseOperationPrompt: @"กรุณายืนยันตัวตนเพื่อดู password"
    };
    
    dispatch_async(dispatch_get_global_queue(DISPATCH_QUEUE_PRIORITY_DEFAULT, 0), ^{
        CFTypeRef result = NULL;
        OSStatus status = SecItemCopyMatching((__bridge CFDictionaryRef)query, &result);
        
        dispatch_async(dispatch_get_main_queue(), ^{
            if (status == errSecSuccess) {
                NSData *data = (__bridge_transfer NSData *)result;
                NSString *password = [[NSString alloc] initWithData:data 
                                                           encoding:NSUTF8StringEncoding];
                completion(password, nil);
            } else {
                NSError *error = [NSError errorWithDomain:NSOSStatusErrorDomain
                                                     code:status
                                                 userInfo:nil];
                completion(nil, error);
            }
        });
    });
}
```

### การแสดง Biometric Prompt โดยตรง

```objc
- (void)authenticateWithBiometrics:(void (^)(BOOL success, NSError *error))completion {
    LAContext *context = [[LAContext alloc] init];
    NSError *error;
    
    if (![context canEvaluatePolicy:LAPolicyDeviceOwnerAuthenticationWithBiometrics 
                              error:&error]) {
        completion(NO, error);
        return;
    }
    
    NSString *reason = @"ยืนยันตัวตนเพื่อเข้าใช้งาน";
    
    [context evaluatePolicy:LAPolicyDeviceOwnerAuthenticationWithBiometrics
            localizedReason:reason
                      reply:^(BOOL success, NSError *authError) {
        dispatch_async(dispatch_get_main_queue(), ^{
            completion(success, authError);
        });
    }];
}

// ตัวอย่างการใช้งาน
[self authenticateWithBiometrics:^(BOOL success, NSError *error) {
    if (success) {
        NSLog(@"Authentication successful!");
        // แสดงข้อมูลที่ปลอดภัย
    } else {
        if (error.code == LAErrorUserCancel) {
            NSLog(@"User cancelled authentication");
        } else if (error.code == LAErrorAuthenticationFailed) {
            NSLog(@"Authentication failed");
        } else if (error.code == LAErrorBiometryNotAvailable) {
            NSLog(@"Biometry not available");
        }
    }
}];
```

---

## 62.10 Best Practices สำหรับข้อมูลสำคัญ

### 1. เลือก Accessibility ที่เหมาะสม

```objc
// ใช้ ThisDeviceOnly สำหรับข้อมูลที่ไม่ควร backup
kSecAttrAccessibleWhenUnlockedThisDeviceOnly   // แนะนำสำหรับ passwords
kSecAttrAccessibleAfterFirstUnlockThisDeviceOnly // สำหรับ background access

// หลีกเลี่ยง:
kSecAttrAccessibleAlways              // ไม่ปลอดภัย
kSecAttrAccessibleAlwaysThisDeviceOnly // ไม่ปลอดภัย
```

### 2. ใช้ Access Control สำหรับข้อมูลสำคัญมาก

```objc
// ต้องการ biometric + device passcode
CFErrorRef error = NULL;
SecAccessControlRef acl = SecAccessControlCreateWithFlags(
    NULL,
    kSecAttrAccessibleWhenPasscodeSetThisDeviceOnly,
    kSecAccessControlBiometryAny | kSecAccessControlDevicePasscode,
    &error
);
```

### 3. ป้องกัน Keychain ถูกย้ายไป device อื่น

```objc
// ใช้ ThisDeviceOnly variants
// ข้อมูลจะถูกเข้ารหัสด้วย hardware key ของ device นั้นๆ
// ไม่สามารถ restore ไปยัง device อื่นได้
kSecAttrAccessibleWhenUnlockedThisDeviceOnly
```

### 4. Error Handling ที่ดี

```objc
- (BOOL)saveSecureData:(NSData *)data 
               forKey:(NSString *)key 
                error:(NSError **)outError {
    // ลบเดิมก่อน
    [self deleteDataForKey:key error:nil];
    
    NSDictionary *query = @{
        (__bridge id)kSecClass: (__bridge id)kSecClassGenericPassword,
        (__bridge id)kSecAttrAccount: key,
        (__bridge id)kSecValueData: data,
        (__bridge id)kSecAttrAccessible: (__bridge id)kSecAttrAccessibleWhenUnlockedThisDeviceOnly
    };
    
    OSStatus status = SecItemAdd((__bridge CFDictionaryRef)query, NULL);
    
    if (status != errSecSuccess) {
        if (outError) {
            NSDictionary *userInfo = @{
                NSLocalizedDescriptionKey: [self descriptionForStatus:status],
                @"OSStatus": @(status)
            };
            *outError = [NSError errorWithDomain:@"KeychainError" 
                                           code:status 
                                       userInfo:userInfo];
        }
        return NO;
    }
    
    return YES;
}

- (NSString *)descriptionForStatus:(OSStatus)status {
    switch (status) {
        case errSecSuccess:         return @"No error";
        case errSecItemNotFound:    return @"Item not found";
        case errSecDuplicateItem:   return @"Duplicate item";
        case errSecAuthFailed:      return @"Authentication failed";
        case errSecUserCanceled:    return @"User canceled";
        case errSecInteractionNotAllowed: return @"Interaction not allowed";
        default:
            return [NSString stringWithFormat:@"Keychain error: %d", (int)status];
    }
}
```

### 5. ลบข้อมูลเมื่อ User Logout

```objc
- (void)clearAllUserCredentials {
    NSArray *itemClasses = @[
        (__bridge id)kSecClassGenericPassword,
        (__bridge id)kSecClassInternetPassword,
        (__bridge id)kSecClassKey
    ];
    
    for (id itemClass in itemClasses) {
        NSDictionary *query = @{
            (__bridge id)kSecClass: itemClass
        };
        SecItemDelete((__bridge CFDictionaryRef)query);
    }
    
    NSLog(@"All keychain items cleared for logout.");
}
```

### 6. ไม่ Log ข้อมูลสำคัญ

```objc
// ไม่ควรทำ
NSString *password = [keychain passwordForKey:@"user_password" service:@"MyApp"];
NSLog(@"Password: %@", password); // อันตราย!

// ควรทำ
BOOL hasPassword = [keychain passwordForKey:@"user_password" service:@"MyApp"] != nil;
NSLog(@"Password exists: %@", hasPassword ? @"YES" : @"NO");
```

---

## 62.11 แบบฝึกหัด

### แบบฝึกหัดที่ 1: Secure Login Manager

สร้าง `LoginManager` class ที่:
- บันทึก username และ password ลง Keychain
- รองรับ Remember Me functionality
- ใช้ biometric authentication ถ้า available
- Auto-logout หลัง idle time

```objc
@interface LoginManager : NSObject
@property (nonatomic, readonly) BOOL isLoggedIn;
- (void)loginWithUsername:(NSString *)username 
                 password:(NSString *)password 
                 remember:(BOOL)remember
               completion:(void (^)(BOOL success, NSError *error))completion;
- (void)loginWithBiometrics:(void (^)(BOOL success, NSError *error))completion;
- (void)logout;
@end
```

### แบบฝึกหัดที่ 2: Token Manager

สร้าง class สำหรับจัดการ OAuth tokens:
- เก็บ access token และ refresh token
- ตรวจสอบ expiry
- Auto-refresh เมื่อหมดอายุ
- Thread-safe access

### แบบฝึกหัดที่ 3: Encrypted Preferences

สร้าง wrapper ที่คล้าย `NSUserDefaults` แต่เก็บข้อมูลใน Keychain:
- รองรับ NSString, NSData, NSNumber, BOOL
- Thread-safe
- รองรับ default values

```objc
@interface SecurePreferences : NSObject
+ (instancetype)standardPreferences;
- (void)setString:(NSString *)value forKey:(NSString *)key;
- (NSString *)stringForKey:(NSString *)key;
- (void)setBool:(BOOL)value forKey:(NSString *)key;
- (BOOL)boolForKey:(NSString *)key;
- (void)removeObjectForKey:(NSString *)key;
@end
```

### แบบฝึกหัดที่ 4: Certificate Pinning

ใช้ Keychain เก็บ certificates สำหรับ SSL Pinning:
- โหลด certificates จาก bundle
- เปรียบเทียบกับ server certificate
- Reject connections ที่ certificate ไม่ตรง

---

## สรุป

Keychain Services เป็น API ที่สำคัญสำหรับการจัดการข้อมูลที่ปลอดภัยบน iOS เราได้เรียนรู้:

1. **Generic Passwords**: เก็บ passwords และ tokens ทั่วไป
2. **Internet Passwords**: เก็บ credentials สำหรับ web services
3. **Certificates และ Keys**: เก็บ cryptographic materials
4. **Access Groups**: แชร์ keychain items ระหว่าง apps
5. **Biometric Authentication**: เพิ่มความปลอดภัยด้วย Touch ID/Face ID
6. **Best Practices**: การใช้งานที่ปลอดภัยและถูกต้อง

บทต่อไปเราจะเรียนรู้เกี่ยวกับ **iCloud** ซึ่งทำให้ data ของ user sync ระหว่าง devices ได้
