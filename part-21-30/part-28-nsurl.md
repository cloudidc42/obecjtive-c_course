# Part 28: NSURL

## บทนำ

**NSURL** (Uniform Resource Locator) คือคลาสที่ใช้แทน URL ใน Objective-C ซึ่งสามารถชี้ไปยัง:
- ไฟล์บนเครื่อง (file:///Users/john/Documents/file.txt)
- ทรัพยากรบนอินเทอร์เน็ต (https://www.example.com/api/data)
- ทรัพยากรภายในแอป (app://settings/profile)

NSURL ไม่เพียงแค่เก็บ string แต่ยังช่วย parse, validate, และ manipulate URL components ต่าง ๆ ได้อย่างถูกต้อง

---

## 28.1 NSURL Creation - การสร้าง URL

### สร้าง HTTP/HTTPS URLs

```objc
#import <Foundation/Foundation.h>

int main(int argc, const char * argv[]) {
    @autoreleasepool {
        
        // วิธีที่ 1: URLWithString:
        NSURL *url1 = [NSURL URLWithString:@"https://www.apple.com"];
        NSLog(@"URL 1: %@", url1);
        
        // วิธีที่ 2: initWithString:
        NSURL *url2 = [[NSURL alloc] initWithString:@"https://api.github.com/users/octocat"];
        NSLog(@"URL 2: %@", url2);
        
        // วิธีที่ 3: URL พร้อม path และ query
        NSURL *url3 = [NSURL URLWithString:@"https://www.google.com/search?q=objective+c&lang=en"];
        NSLog(@"URL 3: %@", url3);
        
        // ตรวจสอบ URL ที่ไม่ถูกต้อง
        NSURL *invalid = [NSURL URLWithString:@"not a valid url with spaces"];
        NSLog(@"\nInvalid URL: %@", invalid); // จะได้ nil
        
        // URL ว่าง
        NSURL *empty = [NSURL URLWithString:@""];
        NSLog(@"Empty URL: %@", empty);
        
        // URL ที่มี Fragment (#)
        NSURL *withFragment = [NSURL URLWithString:@"https://developer.apple.com/documentation#overview"];
        NSLog(@"\nWith fragment: %@", withFragment);
        
        // URL ที่มี Port
        NSURL *withPort = [NSURL URLWithString:@"https://localhost:8080/api/v1"];
        NSLog(@"With port: %@", withPort);
        
    }
    return 0;
}
```

### สร้าง File URLs

```objc
#import <Foundation/Foundation.h>

int main(int argc, const char * argv[]) {
    @autoreleasepool {
        
        // File URL จาก path
        NSURL *fileURL1 = [NSURL fileURLWithPath:@"/Users/john/Documents/file.txt"];
        NSLog(@"File URL: %@", fileURL1);
        NSLog(@"Is file URL: %@", fileURL1.isFileURL ? @"YES" : @"NO");
        
        // File URL directory (ลงท้ายด้วย /)
        NSURL *dirURL = [NSURL fileURLWithPath:@"/Users/john/Documents" isDirectory:YES];
        NSLog(@"Dir URL: %@", dirURL);
        
        // File URL จาก NSFileManager paths
        NSString *homeDir = NSHomeDirectory();
        NSURL *homeURL = [NSURL fileURLWithPath:homeDir];
        NSLog(@"\nHome URL: %@", homeURL);
        
        // Temp directory
        NSURL *tempURL = [NSURL fileURLWithPath:NSTemporaryDirectory()];
        NSLog(@"Temp URL: %@", tempURL);
        
        // Documents directory (iOS-style)
        NSArray *paths = NSSearchPathForDirectoriesInDomains(NSDocumentDirectory, 
                                                              NSUserDomainMask, 
                                                              YES);
        if (paths.count > 0) {
            NSURL *docsURL = [NSURL fileURLWithPath:paths.firstObject];
            NSLog(@"Documents URL: %@", docsURL);
        }
        
        // NSFileManager URL methods (macOS 10.6+)
        NSFileManager *fm = [NSFileManager defaultManager];
        NSURL *desktopURL = [fm URLForDirectory:NSDesktopDirectory
                                       inDomain:NSUserDomainMask
                              appropriateForURL:nil
                                         create:NO
                                          error:nil];
        NSLog(@"Desktop URL: %@", desktopURL);
        
        // แปลง path -> URL -> path
        NSString *originalPath = @"/tmp/test.txt";
        NSURL *url = [NSURL fileURLWithPath:originalPath];
        NSString *backToPath = url.path;
        NSLog(@"\nPath round-trip: %@ == %@ : %@",
              originalPath, backToPath, 
              [originalPath isEqualToString:backToPath] ? @"YES" : @"NO");
        
    }
    return 0;
}
```

---

## 28.2 URL Components - ส่วนประกอบของ URL

URL มีโครงสร้างดังนี้:
```
https://user:password@www.example.com:8080/path/to/page?key=value&key2=value2#section
  ↑        ↑      ↑          ↑          ↑       ↑              ↑                ↑
scheme   user  password    host       port    path           query           fragment
```

```objc
#import <Foundation/Foundation.h>

int main(int argc, const char * argv[]) {
    @autoreleasepool {
        
        NSURL *url = [NSURL URLWithString:
            @"https://john:secret@api.example.com:8443/v2/users/profile?format=json&lang=th#details"];
        
        NSLog(@"Full URL: %@", url);
        NSLog(@"---");
        
        // แยกส่วนประกอบ
        NSLog(@"scheme:          %@", url.scheme);         // https
        NSLog(@"user:            %@", url.user);           // john
        NSLog(@"password:        %@", url.password);       // secret
        NSLog(@"host:            %@", url.host);           // api.example.com
        NSLog(@"port:            %@", url.port);           // 8443
        NSLog(@"path:            %@", url.path);           // /v2/users/profile
        NSLog(@"query:           %@", url.query);          // format=json&lang=th
        NSLog(@"fragment:        %@", url.fragment);       // details
        NSLog(@"relativePath:    %@", url.relativePath);   // /v2/users/profile
        
        // Path components
        NSLog(@"\npathComponents:  %@", url.pathComponents);
        NSLog(@"lastPathComponent: %@", url.lastPathComponent); // profile
        NSLog(@"pathExtension:   %@", url.pathExtension);    // (none)
        
        // File URL examples
        NSURL *fileURL = [NSURL fileURLWithPath:@"/Users/john/Documents/report.pdf"];
        NSLog(@"\nFile components:");
        NSLog(@"path:              %@", fileURL.path);
        NSLog(@"lastPathComponent: %@", fileURL.lastPathComponent); // report.pdf
        NSLog(@"pathExtension:     %@", fileURL.pathExtension);     // pdf
        NSLog(@"pathComponents:    %@", fileURL.pathComponents);
        
        // absoluteString vs relativeString
        NSLog(@"\nabsoluteString: %@", url.absoluteString);
        NSLog(@"relativeString: %@", url.relativeString);
        
    }
    return 0;
}
```

---

## 28.3 NSURLComponents - การสร้าง URL แบบ Component

NSURLComponents เป็นวิธีที่ถูกต้องในการสร้าง URL โดยกำหนดทีละส่วน และจัดการ percent encoding ให้อัตโนมัติ

```objc
#import <Foundation/Foundation.h>

int main(int argc, const char * argv[]) {
    @autoreleasepool {
        
        // สร้าง URL ด้วย NSURLComponents
        NSURLComponents *components = [[NSURLComponents alloc] init];
        components.scheme = @"https";
        components.host   = @"api.example.com";
        components.port   = @443;
        components.path   = @"/v1/search";
        
        // Query items
        components.queryItems = @[
            [NSURLQueryItem queryItemWithName:@"q" value:@"objective-c tutorial"],
            [NSURLQueryItem queryItemWithName:@"page" value:@"1"],
            [NSURLQueryItem queryItemWithName:@"limit" value:@"20"]
        ];
        
        NSURL *url = components.URL;
        NSLog(@"Built URL: %@", url);
        
        // Parse URL ด้วย NSURLComponents
        NSURL *existingURL = [NSURL URLWithString:
            @"https://search.example.com/results?q=swift+programming&lang=en&page=2"];
        
        NSURLComponents *parsed = [NSURLComponents componentsWithURL:existingURL
                                             resolvingAgainstBaseURL:NO];
        
        NSLog(@"\nParsed URL: %@", existingURL);
        NSLog(@"Scheme: %@", parsed.scheme);
        NSLog(@"Host: %@", parsed.host);
        NSLog(@"Path: %@", parsed.path);
        
        NSLog(@"\nQuery items:");
        for (NSURLQueryItem *item in parsed.queryItems) {
            NSLog(@"  %@ = %@", item.name, item.value);
        }
        
        // แก้ไข component และสร้าง URL ใหม่
        parsed.path = @"/new-results";
        
        NSMutableArray *newItems = [parsed.queryItems mutableCopy];
        NSURLQueryItem *pageItem = [NSURLQueryItem queryItemWithName:@"page" value:@"3"];
        // แทนที่ page item
        for (NSUInteger i = 0; i < newItems.count; i++) {
            if ([newItems[i].name isEqualToString:@"page"]) {
                newItems[i] = pageItem;
            }
        }
        parsed.queryItems = newItems;
        
        NSLog(@"\nModified URL: %@", parsed.URL);
        
    }
    return 0;
}
```

### สร้าง API URL อย่างถูกต้อง

```objc
#import <Foundation/Foundation.h>

@interface APIURLBuilder : NSObject
@property (nonatomic, strong) NSString *baseURL;
@property (nonatomic, strong) NSString *version;

- (instancetype)initWithBaseURL:(NSString *)baseURL version:(NSString *)version;
- (NSURL *)buildURL:(NSString *)endpoint params:(NSDictionary *)params;
@end

@implementation APIURLBuilder

- (instancetype)initWithBaseURL:(NSString *)baseURL version:(NSString *)version {
    self = [super init];
    if (self) {
        _baseURL = baseURL;
        _version = version;
    }
    return self;
}

- (NSURL *)buildURL:(NSString *)endpoint params:(NSDictionary *)params {
    NSURLComponents *components = [NSURLComponents componentsWithString:self.baseURL];
    components.path = [NSString stringWithFormat:@"/%@%@", self.version, endpoint];
    
    if (params.count > 0) {
        NSMutableArray *queryItems = [NSMutableArray array];
        // เรียงลำดับ key เพื่อให้ URL สม่ำเสมอ
        NSArray *sortedKeys = [params.allKeys sortedArrayUsingSelector:@selector(compare:)];
        for (NSString *key in sortedKeys) {
            [queryItems addObject:[NSURLQueryItem queryItemWithName:key 
                                                             value:[params[key] description]]];
        }
        components.queryItems = queryItems;
    }
    
    return components.URL;
}

@end

int main(int argc, const char * argv[]) {
    @autoreleasepool {
        
        APIURLBuilder *builder = [[APIURLBuilder alloc] 
                                   initWithBaseURL:@"https://api.myapp.com"
                                           version:@"v2"];
        
        // Build URLs
        NSURL *usersURL = [builder buildURL:@"/users" params:@{
            @"page": @1,
            @"limit": @50,
            @"sort": @"created_at"
        }];
        NSLog(@"Users URL: %@", usersURL);
        
        NSURL *searchURL = [builder buildURL:@"/search" params:@{
            @"q": @"objective c",
            @"type": @"code",
            @"lang": @"objc"
        }];
        NSLog(@"Search URL: %@", searchURL);
        
        NSURL *profileURL = [builder buildURL:@"/users/123/profile" params:nil];
        NSLog(@"Profile URL: %@", profileURL);
        
    }
    return 0;
}
```

---

## 28.4 File System URLs

### การทำงานกับ File System

```objc
#import <Foundation/Foundation.h>

int main(int argc, const char * argv[]) {
    @autoreleasepool {
        
        NSFileManager *fm = [NSFileManager defaultManager];
        
        // ดึง URL สำหรับ system directories
        NSError *error = nil;
        
        // Documents directory
        NSURL *docsURL = [fm URLForDirectory:NSDocumentDirectory
                                    inDomain:NSUserDomainMask
                           appropriateForURL:nil
                                      create:YES
                                       error:&error];
        NSLog(@"Documents: %@", docsURL);
        
        // Application Support
        NSURL *appSupportURL = [fm URLForDirectory:NSApplicationSupportDirectory
                                          inDomain:NSUserDomainMask
                                 appropriateForURL:nil
                                            create:YES
                                             error:&error];
        NSLog(@"App Support: %@", appSupportURL);
        
        // Caches
        NSURL *cachesURL = [fm URLForDirectory:NSCachesDirectory
                                      inDomain:NSUserDomainMask
                             appropriateForURL:nil
                                        create:YES
                                         error:&error];
        NSLog(@"Caches: %@", cachesURL);
        
        // Temp directory
        NSURL *tempURL = [NSURL fileURLWithPath:NSTemporaryDirectory()];
        NSLog(@"Temp: %@", tempURL);
        
        // สร้าง URL สำหรับไฟล์ใน directories เหล่านี้
        NSURL *fileURL = [docsURL URLByAppendingPathComponent:@"myfile.txt"];
        NSLog(@"\nFile in docs: %@", fileURL);
        
        NSURL *subDir = [docsURL URLByAppendingPathComponent:@"images" isDirectory:YES];
        NSLog(@"Subdir: %@", subDir);
        
        NSURL *imageFile = [subDir URLByAppendingPathComponent:@"photo.jpg"];
        NSLog(@"Image file: %@", imageFile);
        
    }
    return 0;
}
```

### File URL Path Manipulation

```objc
#import <Foundation/Foundation.h>

int main(int argc, const char * argv[]) {
    @autoreleasepool {
        
        NSURL *url = [NSURL fileURLWithPath:@"/Users/john/Documents/report.pdf"];
        
        NSLog(@"Original: %@", url);
        
        // ขึ้นไปหนึ่งระดับ
        NSURL *parent = [url URLByDeletingLastPathComponent];
        NSLog(@"Parent: %@", parent);
        
        // เพิ่ม path component
        NSURL *sibling = [[url URLByDeletingLastPathComponent] 
                          URLByAppendingPathComponent:@"summary.txt"];
        NSLog(@"Sibling: %@", sibling);
        
        // เปลี่ยน extension
        NSURL *docx = [url URLByDeletingPathExtension];
        docx = [docx URLByAppendingPathExtension:@"docx"];
        NSLog(@"Changed extension: %@", docx);
        
        // ตรวจสอบว่ามีไฟล์จริงไหม
        NSLog(@"\nFile exists: %@", 
              [url checkResourceIsReachableAndReturnError:nil] ? @"YES" : @"NO");
        
        // เปรียบเทียบ URLs
        NSURL *url2 = [NSURL fileURLWithPath:@"/Users/john/Documents/report.pdf"];
        NSLog(@"URLs equal: %@", [url isEqual:url2] ? @"YES" : @"NO");
        
        // Standardize URL (resolve symlinks, .., .)
        NSURL *weird = [NSURL fileURLWithPath:@"/tmp/../tmp/./test.txt"];
        NSURL *standard = [weird standardizedURL];
        NSLog(@"\nOriginal: %@", weird);
        NSLog(@"Standardized: %@", standard);
        
        // Resolve symlinks
        NSURL *resolved = [weird URLByResolvingSymlinksInPath];
        NSLog(@"Resolved: %@", resolved);
        
    }
    return 0;
}
```

---

## 28.5 Relative vs Absolute URLs

```objc
#import <Foundation/Foundation.h>

int main(int argc, const char * argv[]) {
    @autoreleasepool {
        
        // Base URL
        NSURL *baseURL = [NSURL URLWithString:@"https://www.example.com/api/v1/"];
        
        // สร้าง relative URL
        NSURL *relativeURL = [NSURL URLWithString:@"users" relativeToURL:baseURL];
        NSLog(@"Relative URL: %@", relativeURL);
        NSLog(@"Absolute URL: %@", relativeURL.absoluteURL);
        NSLog(@"Absolute string: %@", relativeURL.absoluteString);
        NSLog(@"Relative string: %@", relativeURL.relativeString);
        
        // ตัวอย่างต่าง ๆ ของ relative URLs
        NSArray *relatives = @[
            @"users",              // https://www.example.com/api/v1/users
            @"../v2/users",        // https://www.example.com/api/v2/users
            @"/search",            // https://www.example.com/search
            @"//cdn.example.com",  // https://cdn.example.com
            @"?q=test",            // https://www.example.com/api/v1/?q=test
            @"#section"            // https://www.example.com/api/v1/#section
        ];
        
        NSLog(@"\nRelative resolution from: %@", baseURL);
        for (NSString *rel in relatives) {
            NSURL *resolved = [NSURL URLWithString:rel relativeToURL:baseURL];
            NSLog(@"  '%@' -> '%@'", rel, resolved.absoluteString);
        }
        
        // isFileURL check
        NSURL *fileURL  = [NSURL fileURLWithPath:@"/tmp/file.txt"];
        NSURL *httpURL  = [NSURL URLWithString:@"https://example.com"];
        
        NSLog(@"\nfileURL isFileURL: %@", fileURL.isFileURL ? @"YES" : @"NO");
        NSLog(@"httpURL isFileURL: %@", httpURL.isFileURL ? @"YES" : @"NO");
        
    }
    return 0;
}
```

---

## 28.6 URL Encoding/Decoding (Percent Encoding)

URL ไม่สามารถมีตัวอักษรพิเศษบางตัวได้ จึงต้อง "percent encode" เช่น space เป็น %20, ภาษาไทยเป็น %E0%B8%...

```objc
#import <Foundation/Foundation.h>

int main(int argc, const char * argv[]) {
    @autoreleasepool {
        
        // Encode URL string
        NSString *rawString = @"Hello World! search=สวัสดี&type=greeting";
        
        // สำหรับ query string
        NSString *encoded = [rawString stringByAddingPercentEncodingWithAllowedCharacters:
                             [NSCharacterSet URLQueryAllowedCharacterSet]];
        NSLog(@"Original: %@", rawString);
        NSLog(@"Encoded:  %@", encoded);
        
        // Decode กลับ
        NSString *decoded = [encoded stringByRemovingPercentEncoding];
        NSLog(@"Decoded:  %@", decoded);
        NSLog(@"Match: %@", [rawString isEqualToString:decoded] ? @"YES" : @"NO");
        
        // Character sets ต่าง ๆ
        NSString *pathString = @"path/to/my file (1).pdf";
        NSString *pathEncoded = [pathString stringByAddingPercentEncodingWithAllowedCharacters:
                                 [NSCharacterSet URLPathAllowedCharacterSet]];
        NSLog(@"\nPath encoded: %@", pathEncoded);
        
        NSString *hostString = @"my host name";
        NSString *hostEncoded = [hostString stringByAddingPercentEncodingWithAllowedCharacters:
                                 [NSCharacterSet URLHostAllowedCharacterSet]];
        NSLog(@"Host encoded: %@", hostEncoded);
        
        // Fragment
        NSString *fragString = @"section 1 & 2";
        NSString *fragEncoded = [fragString stringByAddingPercentEncodingWithAllowedCharacters:
                                 [NSCharacterSet URLFragmentAllowedCharacterSet]];
        NSLog(@"Fragment encoded: %@", fragEncoded);
        
        // ตรวจสอบ characters ที่ไม่ต้อง encode
        NSCharacterSet *queryAllowed = [NSCharacterSet URLQueryAllowedCharacterSet];
        NSString *testChars = @"ABCDabcd0123-._~:@!$&'()*+,;=/?";
        NSMutableString *result = [NSMutableString string];
        for (NSUInteger i = 0; i < testChars.length; i++) {
            unichar c = [testChars characterAtIndex:i];
            if ([queryAllowed characterIsMember:c]) {
                [result appendFormat:@"%c", c];
            }
        }
        NSLog(@"\nAllowed in query: %@", result);
        
    }
    return 0;
}
```

### URL ที่มีภาษาไทยและ Unicode

```objc
#import <Foundation/Foundation.h>

int main(int argc, const char * argv[]) {
    @autoreleasepool {
        
        // ค้นหาด้วยภาษาไทย
        NSString *thaiQuery = @"โปรแกรม Objective-C สำหรับมือใหม่";
        
        NSURLComponents *components = [[NSURLComponents alloc] init];
        components.scheme = @"https";
        components.host   = @"search.example.com";
        components.path   = @"/results";
        components.queryItems = @[
            [NSURLQueryItem queryItemWithName:@"q" value:thaiQuery],
            [NSURLQueryItem queryItemWithName:@"lang" value:@"th"]
        ];
        
        NSURL *searchURL = components.URL;
        NSLog(@"Thai search URL: %@", searchURL);
        NSLog(@"Absolute string: %@", searchURL.absoluteString);
        
        // Parse กลับ
        NSURLComponents *parsed = [NSURLComponents componentsWithURL:searchURL
                                             resolvingAgainstBaseURL:NO];
        for (NSURLQueryItem *item in parsed.queryItems) {
            NSLog(@"  %@: %@", item.name, item.value);
        }
        
        // IDN (International Domain Names)
        // ภาษาไทยใน domain name
        // ใช้ Punycode encoding (ทำโดย system)
        NSURL *thaiDomain = [NSURL URLWithString:@"http://xn--12cfi8ixb8l.xn--o3cw4h"]; // ตัวอย่าง
        NSLog(@"\nThai domain (punycode): %@", thaiDomain);
        
        // URL ที่ถูกต้องจะผ่าน validation
        NSArray *testURLs = @[
            @"https://example.com",
            @"ftp://files.example.com/file.zip",
            @"mailto:user@example.com",
            @"tel:+66812345678",
            @"https://example.com/path?q=hello world", // space ไม่ถูกต้อง
        ];
        
        NSLog(@"\nURL Validation:");
        for (NSString *urlStr in testURLs) {
            NSURL *url = [NSURL URLWithString:urlStr];
            NSLog(@"  '%@': %@", urlStr, url ? @"valid" : @"invalid");
        }
        
    }
    return 0;
}
```

---

## 28.7 NSURLRequest Basics

NSURLRequest คือ request object ที่ใช้กับ NSURLSession เพื่อดึงข้อมูลจาก URL

```objc
#import <Foundation/Foundation.h>

int main(int argc, const char * argv[]) {
    @autoreleasepool {
        
        // GET Request
        NSURL *url = [NSURL URLWithString:@"https://httpbin.org/get"];
        NSURLRequest *getRequest = [NSURLRequest requestWithURL:url];
        
        NSLog(@"URL: %@", getRequest.URL);
        NSLog(@"Method: %@", getRequest.HTTPMethod);
        NSLog(@"Cache Policy: %lu", (unsigned long)getRequest.cachePolicy);
        NSLog(@"Timeout: %.1f", getRequest.timeoutInterval);
        
        // NSMutableURLRequest - แก้ไขได้
        NSMutableURLRequest *mutableRequest = [NSMutableURLRequest requestWithURL:url];
        mutableRequest.HTTPMethod = @"POST";
        mutableRequest.timeoutInterval = 30.0;
        [mutableRequest setValue:@"application/json" forHTTPHeaderField:@"Content-Type"];
        [mutableRequest setValue:@"Bearer mytoken123" forHTTPHeaderField:@"Authorization"];
        [mutableRequest setValue:@"MyApp/1.0" forHTTPHeaderField:@"User-Agent"];
        
        NSLog(@"\nPOST Request:");
        NSLog(@"URL: %@", mutableRequest.URL);
        NSLog(@"Method: %@", mutableRequest.HTTPMethod);
        NSLog(@"Headers: %@", mutableRequest.allHTTPHeaderFields);
        
        // เพิ่ม body
        NSDictionary *body = @{@"name": @"Alice", @"email": @"alice@example.com"};
        NSData *bodyData = [NSJSONSerialization dataWithJSONObject:body options:0 error:nil];
        mutableRequest.HTTPBody = bodyData;
        
        NSLog(@"Body size: %lu bytes", (unsigned long)mutableRequest.HTTPBody.length);
        
        // Cache policy
        NSMutableURLRequest *cachedRequest = [NSMutableURLRequest requestWithURL:url
                                                                     cachePolicy:NSURLRequestReturnCacheDataElseLoad
                                                                 timeoutInterval:60.0];
        NSLog(@"\nCache policy: %lu", (unsigned long)cachedRequest.cachePolicy);
        
    }
    return 0;
}
```

### ประเภท Cache Policy

```objc
#import <Foundation/Foundation.h>

int main(int argc, const char * argv[]) {
    @autoreleasepool {
        
        NSURL *url = [NSURL URLWithString:@"https://api.example.com/data"];
        
        // NSURLRequestUseProtocolCachePolicy (default)
        // ใช้ cache ตามที่ server กำหนดใน headers
        NSURLRequest *r1 = [NSURLRequest requestWithURL:url
                                            cachePolicy:NSURLRequestUseProtocolCachePolicy
                                        timeoutInterval:30];
        
        // NSURLRequestReloadIgnoringLocalCacheData
        // ไม่ใช้ cache เลย ดึงใหม่เสมอ
        NSURLRequest *r2 = [NSURLRequest requestWithURL:url
                                            cachePolicy:NSURLRequestReloadIgnoringLocalCacheData
                                        timeoutInterval:30];
        
        // NSURLRequestReturnCacheDataElseLoad
        // ใช้ cache ถ้ามี ถ้าไม่มีค่อยดึงใหม่
        NSURLRequest *r3 = [NSURLRequest requestWithURL:url
                                            cachePolicy:NSURLRequestReturnCacheDataElseLoad
                                        timeoutInterval:30];
        
        // NSURLRequestReturnCacheDataDontLoad
        // ใช้ cache เท่านั้น ถ้าไม่มีให้ fail
        NSURLRequest *r4 = [NSURLRequest requestWithURL:url
                                            cachePolicy:NSURLRequestReturnCacheDataDontLoad
                                        timeoutInterval:30];
        
        NSLog(@"r1 cachePolicy: %lu (UseProtocol)", (unsigned long)r1.cachePolicy);
        NSLog(@"r2 cachePolicy: %lu (ReloadIgnoring)", (unsigned long)r2.cachePolicy);
        NSLog(@"r3 cachePolicy: %lu (ReturnCacheElseLoad)", (unsigned long)r3.cachePolicy);
        NSLog(@"r4 cachePolicy: %lu (ReturnCacheDontLoad)", (unsigned long)r4.cachePolicy);
        
    }
    return 0;
}
```

---

## 28.8 Query String Parsing

```objc
#import <Foundation/Foundation.h>

// Parse query string เป็น Dictionary
NSDictionary *parseQueryString(NSString *queryString) {
    NSMutableDictionary *dict = [NSMutableDictionary dictionary];
    
    if (queryString.length == 0) return dict;
    
    // ลบ ? ถ้ามี
    if ([queryString hasPrefix:@"?"]) {
        queryString = [queryString substringFromIndex:1];
    }
    
    NSArray *pairs = [queryString componentsSeparatedByString:@"&"];
    for (NSString *pair in pairs) {
        NSArray *parts = [pair componentsSeparatedByString:@"="];
        if (parts.count == 2) {
            NSString *key   = [parts[0] stringByRemovingPercentEncoding];
            NSString *value = [parts[1] stringByRemovingPercentEncoding];
            dict[key] = value;
        } else if (parts.count == 1) {
            NSString *key = [parts[0] stringByRemovingPercentEncoding];
            dict[key] = @"";
        }
    }
    
    return [dict copy];
}

// Build query string จาก Dictionary
NSString *buildQueryString(NSDictionary *params) {
    if (params.count == 0) return @"";
    
    NSMutableArray *pairs = [NSMutableArray array];
    NSArray *keys = [params.allKeys sortedArrayUsingSelector:@selector(compare:)];
    
    for (NSString *key in keys) {
        NSString *value = [params[key] description];
        NSString *encodedKey = [key stringByAddingPercentEncodingWithAllowedCharacters:
                                [NSCharacterSet URLQueryAllowedCharacterSet]];
        NSString *encodedValue = [value stringByAddingPercentEncodingWithAllowedCharacters:
                                  [NSCharacterSet URLQueryAllowedCharacterSet]];
        [pairs addObject:[NSString stringWithFormat:@"%@=%@", encodedKey, encodedValue]];
    }
    
    return [pairs componentsJoinedByString:@"&"];
}

int main(int argc, const char * argv[]) {
    @autoreleasepool {
        
        // Parse query string
        NSString *query = @"name=Alice&age=30&city=Bangkok&hobby=coding";
        NSDictionary *parsed = parseQueryString(query);
        NSLog(@"Parsed query:");
        for (NSString *key in [parsed.allKeys sortedArrayUsingSelector:@selector(compare:)]) {
            NSLog(@"  %@ = %@", key, parsed[key]);
        }
        
        // Parse จาก URL
        NSURL *url = [NSURL URLWithString:@"https://example.com/search?q=hello+world&type=article&page=1"];
        NSURLComponents *comps = [NSURLComponents componentsWithURL:url 
                                             resolvingAgainstBaseURL:NO];
        
        NSMutableDictionary *queryParams = [NSMutableDictionary dictionary];
        for (NSURLQueryItem *item in comps.queryItems) {
            queryParams[item.name] = item.value ?: @"";
        }
        
        NSLog(@"\nURL query params:");
        for (NSString *key in [queryParams.allKeys sortedArrayUsingSelector:@selector(compare:)]) {
            NSLog(@"  %@ = %@", key, queryParams[key]);
        }
        
        // Build query string
        NSDictionary *params = @{
            @"search": @"Objective-C tutorial",
            @"lang": @"th",
            @"page": @"1",
            @"limit": @"10"
        };
        NSString *built = buildQueryString(params);
        NSLog(@"\nBuilt query: %@", built);
        
        // ใช้ NSURLQueryItem approach (ดีกว่า)
        NSMutableArray *queryItems = [NSMutableArray array];
        for (NSString *key in params) {
            [queryItems addObject:[NSURLQueryItem queryItemWithName:key 
                                                             value:params[key]]];
        }
        
        NSURLComponents *buildComps = [[NSURLComponents alloc] init];
        buildComps.queryItems = queryItems;
        NSLog(@"Built query (URLComponents): %@", buildComps.query);
        
    }
    return 0;
}
```

---

## 28.9 Building API URLs

```objc
#import <Foundation/Foundation.h>

// API URL Builder class
@interface APIClient : NSObject

@property (nonatomic, strong) NSString *baseURL;
@property (nonatomic, strong) NSString *apiKey;
@property (nonatomic, strong) NSString *apiVersion;

- (instancetype)initWithBaseURL:(NSString *)baseURL 
                         apiKey:(NSString *)apiKey
                     apiVersion:(NSString *)apiVersion;

- (NSURL *)endpointURL:(NSString *)endpoint;
- (NSURL *)endpointURL:(NSString *)endpoint queryParams:(NSDictionary *)params;
- (NSURLRequest *)GETRequest:(NSString *)endpoint params:(NSDictionary *)params;
- (NSURLRequest *)POSTRequest:(NSString *)endpoint body:(NSDictionary *)body;
- (NSURLRequest *)PUTRequest:(NSString *)endpoint body:(NSDictionary *)body;
- (NSURLRequest *)DELETERequest:(NSString *)endpoint;

@end

@implementation APIClient

- (instancetype)initWithBaseURL:(NSString *)baseURL 
                         apiKey:(NSString *)apiKey
                     apiVersion:(NSString *)apiVersion {
    self = [super init];
    if (self) {
        _baseURL    = baseURL;
        _apiKey     = apiKey;
        _apiVersion = apiVersion;
    }
    return self;
}

- (NSURL *)endpointURL:(NSString *)endpoint {
    return [self endpointURL:endpoint queryParams:nil];
}

- (NSURL *)endpointURL:(NSString *)endpoint queryParams:(NSDictionary *)params {
    NSURLComponents *comps = [NSURLComponents componentsWithString:self.baseURL];
    comps.path = [NSString stringWithFormat:@"/%@%@", self.apiVersion, endpoint];
    
    NSMutableArray *queryItems = [NSMutableArray array];
    // เพิ่ม API key
    if (self.apiKey) {
        [queryItems addObject:[NSURLQueryItem queryItemWithName:@"api_key" 
                                                         value:self.apiKey]];
    }
    // เพิ่ม params
    for (NSString *key in params) {
        [queryItems addObject:[NSURLQueryItem queryItemWithName:key 
                                                         value:[params[key] description]]];
    }
    
    if (queryItems.count > 0) {
        comps.queryItems = queryItems;
    }
    
    return comps.URL;
}

- (NSMutableURLRequest *)baseRequest:(NSString *)endpoint {
    NSURL *url = [self endpointURL:endpoint];
    NSMutableURLRequest *request = [NSMutableURLRequest requestWithURL:url];
    [request setValue:@"application/json" forHTTPHeaderField:@"Accept"];
    [request setValue:@"application/json" forHTTPHeaderField:@"Content-Type"];
    return request;
}

- (NSURLRequest *)GETRequest:(NSString *)endpoint params:(NSDictionary *)params {
    NSURL *url = [self endpointURL:endpoint queryParams:params];
    NSMutableURLRequest *request = [NSMutableURLRequest requestWithURL:url];
    request.HTTPMethod = @"GET";
    [request setValue:@"application/json" forHTTPHeaderField:@"Accept"];
    return request;
}

- (NSURLRequest *)POSTRequest:(NSString *)endpoint body:(NSDictionary *)body {
    NSMutableURLRequest *request = [self baseRequest:endpoint];
    request.HTTPMethod = @"POST";
    
    if (body) {
        NSError *error = nil;
        request.HTTPBody = [NSJSONSerialization dataWithJSONObject:body
                                                           options:0
                                                             error:&error];
    }
    return request;
}

- (NSURLRequest *)PUTRequest:(NSString *)endpoint body:(NSDictionary *)body {
    NSMutableURLRequest *request = [self baseRequest:endpoint];
    request.HTTPMethod = @"PUT";
    
    if (body) {
        NSError *error = nil;
        request.HTTPBody = [NSJSONSerialization dataWithJSONObject:body
                                                           options:0
                                                             error:&error];
    }
    return request;
}

- (NSURLRequest *)DELETERequest:(NSString *)endpoint {
    NSMutableURLRequest *request = [self baseRequest:endpoint];
    request.HTTPMethod = @"DELETE";
    return request;
}

@end

int main(int argc, const char * argv[]) {
    @autoreleasepool {
        
        APIClient *client = [[APIClient alloc] 
                             initWithBaseURL:@"https://api.myservice.com"
                                      apiKey:@"secret_key_123"
                                  apiVersion:@"v1"];
        
        // GET /users
        NSURLRequest *getUsersReq = [client GETRequest:@"/users" 
                                                params:@{@"page": @1, @"limit": @20}];
        NSLog(@"GET Users: %@", getUsersReq.URL);
        
        // GET /users/123
        NSURLRequest *getUserReq = [client GETRequest:@"/users/123" params:nil];
        NSLog(@"GET User: %@", getUserReq.URL);
        
        // POST /users
        NSURLRequest *createUserReq = [client POSTRequest:@"/users" 
                                                     body:@{@"name": @"Alice", 
                                                            @"email": @"alice@example.com"}];
        NSLog(@"POST User URL: %@", createUserReq.URL);
        NSLog(@"POST Method: %@", createUserReq.HTTPMethod);
        
        // PUT /users/123
        NSURLRequest *updateUserReq = [client PUTRequest:@"/users/123"
                                                    body:@{@"name": @"Alice Smith"}];
        NSLog(@"PUT User: %@", updateUserReq.URL);
        
        // DELETE /users/123
        NSURLRequest *deleteUserReq = [client DELETERequest:@"/users/123"];
        NSLog(@"DELETE User: %@", deleteUserReq.URL);
        
        // Search endpoint
        NSURLRequest *searchReq = [client GETRequest:@"/search" 
                                              params:@{
                                                  @"q": @"objective c",
                                                  @"type": @"tutorial",
                                                  @"lang": @"th"
                                              }];
        NSLog(@"\nSearch: %@", searchReq.URL);
        
    }
    return 0;
}
```

---

## 28.10 NSURL Resource Values (File Attributes)

```objc
#import <Foundation/Foundation.h>

int main(int argc, const char * argv[]) {
    @autoreleasepool {
        
        // สร้างไฟล์ทดสอบ
        NSString *testPath = @"/tmp/test_resource.txt";
        [@"Hello, NSURL!" writeToFile:testPath atomically:YES 
                           encoding:NSUTF8StringEncoding error:nil];
        
        NSURL *fileURL = [NSURL fileURLWithPath:testPath];
        
        // ดึง resource values
        NSError *error = nil;
        NSDictionary *values = [fileURL resourceValuesForKeys:@[
            NSURLFileSizeKey,
            NSURLCreationDateKey,
            NSURLContentModificationDateKey,
            NSURLIsDirectoryKey,
            NSURLIsReadableKey,
            NSURLIsWritableKey,
            NSURLIsHiddenKey,
            NSURLFileResourceTypeKey
        ] error:&error];
        
        if (!error) {
            NSLog(@"File: %@", testPath);
            NSLog(@"Size: %@ bytes", values[NSURLFileSizeKey]);
            NSLog(@"Created: %@", values[NSURLCreationDateKey]);
            NSLog(@"Modified: %@", values[NSURLContentModificationDateKey]);
            NSLog(@"Is directory: %@", values[NSURLIsDirectoryKey]);
            NSLog(@"Is readable: %@", values[NSURLIsReadableKey]);
            NSLog(@"Is writable: %@", values[NSURLIsWritableKey]);
            NSLog(@"Is hidden: %@", values[NSURLIsHiddenKey]);
            NSLog(@"Type: %@", values[NSURLFileResourceTypeKey]);
        }
        
        // ตรวจสอบ specific values
        NSNumber *isDir;
        [fileURL getResourceValue:&isDir forKey:NSURLIsDirectoryKey error:nil];
        NSLog(@"\nIs directory: %@", isDir.boolValue ? @"YES" : @"NO");
        
        NSNumber *fileSize;
        [fileURL getResourceValue:&fileSize forKey:NSURLFileSizeKey error:nil];
        NSLog(@"File size: %lu bytes", (unsigned long)fileSize.unsignedLongValue);
        
        // Directory listing
        NSURL *dirURL = [NSURL fileURLWithPath:@"/tmp"];
        NSDirectoryEnumerator *enumerator = [[NSFileManager defaultManager]
                                             enumeratorAtURL:dirURL
                                         includingPropertiesForKeys:@[NSURLFileSizeKey, NSURLIsDirectoryKey]
                                                            options:NSDirectoryEnumerationSkipsHiddenFiles | 
                                                                    NSDirectoryEnumerationSkipsSubdirectoryDescendants
                                                       errorHandler:nil];
        
        NSLog(@"\nFiles in /tmp:");
        NSUInteger count = 0;
        for (NSURL *itemURL in enumerator) {
            if (count++ >= 5) break; // แสดงแค่ 5 รายการ
            
            NSNumber *isDirectory;
            [itemURL getResourceValue:&isDirectory forKey:NSURLIsDirectoryKey error:nil];
            
            NSLog(@"  %@ %@", isDirectory.boolValue ? @"[DIR]" : @"[FILE]",
                  itemURL.lastPathComponent);
        }
        
    }
    return 0;
}
```

---

## 28.11 URL Scheme และ Deep Links

```objc
#import <Foundation/Foundation.h>

int main(int argc, const char * argv[]) {
    @autoreleasepool {
        
        // ตรวจสอบ scheme ต่าง ๆ
        NSArray *urls = @[
            @"https://www.apple.com",
            @"http://example.com",
            @"ftp://files.example.com",
            @"mailto:user@example.com",
            @"tel:+66812345678",
            @"sms:+66812345678",
            @"file:///tmp/file.txt",
            @"myapp://settings/profile",
            @"myapp://products/123?ref=email"
        ];
        
        NSLog(@"URL Schemes:");
        for (NSString *urlStr in urls) {
            NSURL *url = [NSURL URLWithString:urlStr];
            NSLog(@"  %-45s scheme: %@", urlStr.UTF8String, url.scheme);
        }
        
        // Deep Link parsing
        NSURL *deepLink = [NSURL URLWithString:@"myapp://product/detail/456?source=push&campaign=summer_sale"];
        
        NSLog(@"\nDeep Link: %@", deepLink);
        NSLog(@"Scheme: %@", deepLink.scheme);
        NSLog(@"Host: %@", deepLink.host);   // "product" (first path component)
        NSLog(@"Path: %@", deepLink.path);
        NSLog(@"PathComponents: %@", deepLink.pathComponents);
        
        NSURLComponents *deepComponents = [NSURLComponents componentsWithURL:deepLink
                                                     resolvingAgainstBaseURL:NO];
        NSLog(@"Query items:");
        for (NSURLQueryItem *item in deepComponents.queryItems) {
            NSLog(@"  %@ = %@", item.name, item.value);
        }
        
        // Router สำหรับ deep links
        NSMutableDictionary *routes = [NSMutableDictionary dictionary];
        routes[@"product/detail"] = ^(NSDictionary *params) {
            NSLog(@"Show product detail, ID: %@", params[@"id"]);
        };
        routes[@"settings"] = ^(NSDictionary *params) {
            NSLog(@"Show settings");
        };
        
        // Match route
        NSString *host = deepLink.host; // product
        NSArray *pathParts = [deepLink.path componentsSeparatedByString:@"/"];
        NSString *routeKey = [NSString stringWithFormat:@"%@/%@", 
                              host, pathParts.count > 1 ? pathParts[1] : @""];
        
        void(^handler)(NSDictionary *) = routes[routeKey];
        if (handler) {
            NSMutableDictionary *params = [NSMutableDictionary dictionary];
            for (NSURLQueryItem *item in deepComponents.queryItems) {
                params[item.name] = item.value;
            }
            // ดึง ID จาก path
            if (pathParts.count > 2) {
                params[@"id"] = pathParts[2];
            }
            handler(params);
        }
        
    }
    return 0;
}
```

---

## 28.12 URL Validation และ Sanitization

```objc
#import <Foundation/Foundation.h>

@interface URLValidator : NSObject

+ (BOOL)isValidURL:(NSString *)urlString;
+ (BOOL)isValidHTTPURL:(NSString *)urlString;
+ (BOOL)isValidFileURL:(NSString *)urlString;
+ (NSString *)sanitizeURL:(NSString *)urlString;
+ (NSURL *)safeURLFromString:(NSString *)urlString;

@end

@implementation URLValidator

+ (BOOL)isValidURL:(NSString *)urlString {
    if (urlString.length == 0) return NO;
    NSURL *url = [NSURL URLWithString:urlString];
    return url != nil && url.scheme != nil;
}

+ (BOOL)isValidHTTPURL:(NSString *)urlString {
    if (![self isValidURL:urlString]) return NO;
    NSURL *url = [NSURL URLWithString:urlString];
    return [@[@"http", @"https"] containsObject:url.scheme.lowercaseString];
}

+ (BOOL)isValidFileURL:(NSString *)urlString {
    if (![self isValidURL:urlString]) return NO;
    NSURL *url = [NSURL URLWithString:urlString];
    return url.isFileURL;
}

+ (NSString *)sanitizeURL:(NSString *)urlString {
    // trim whitespace
    urlString = [urlString stringByTrimmingCharactersInSet:
                 [NSCharacterSet whitespaceAndNewlineCharacterSet]];
    
    // เพิ่ม scheme ถ้าไม่มี
    if (urlString.length > 0 && 
        ![urlString hasPrefix:@"http://"] && 
        ![urlString hasPrefix:@"https://"] &&
        ![urlString hasPrefix:@"ftp://"] &&
        ![urlString hasPrefix:@"file://"]) {
        urlString = [@"https://" stringByAppendingString:urlString];
    }
    
    return urlString;
}

+ (NSURL *)safeURLFromString:(NSString *)urlString {
    // Percent-encode ตัวอักษรที่ไม่ถูกต้อง
    NSString *encoded = [urlString stringByAddingPercentEncodingWithAllowedCharacters:
                         [NSCharacterSet URLQueryAllowedCharacterSet]];
    
    // ลองสร้าง URL
    NSURL *url = [NSURL URLWithString:encoded];
    
    // ถ้าไม่ได้ ลองใช้ NSURLComponents
    if (!url) {
        NSURLComponents *comps = [[NSURLComponents alloc] init];
        // parse แบบ manual
        NSRange schemeRange = [urlString rangeOfString:@"://"];
        if (schemeRange.location != NSNotFound) {
            comps.scheme = [urlString substringToIndex:schemeRange.location];
            NSString *rest = [urlString substringFromIndex:schemeRange.location + 3];
            NSRange pathStart = [rest rangeOfString:@"/"];
            if (pathStart.location != NSNotFound) {
                comps.host = [rest substringToIndex:pathStart.location];
                comps.path = [rest substringFromIndex:pathStart.location];
            } else {
                comps.host = rest;
            }
        }
        url = comps.URL;
    }
    
    return url;
}

@end

int main(int argc, const char * argv[]) {
    @autoreleasepool {
        
        NSArray *testURLs = @[
            @"https://www.apple.com",
            @"http://example.com/path?q=test",
            @"ftp://files.com/file.zip",
            @"not a url",
            @"",
            @"   https://trimmed.com   ",
            @"example.com",           // ขาด scheme
            @"https://example.com/path with spaces",
        ];
        
        NSLog(@"URL Validation:");
        for (NSString *url in testURLs) {
            BOOL valid = [URLValidator isValidHTTPURL:url];
            NSLog(@"  %-45s -> %@", url.UTF8String, valid ? @"VALID" : @"INVALID");
        }
        
        // Sanitize
        NSLog(@"\nURL Sanitization:");
        NSArray *rawInputs = @[@"  apple.com  ", @"google.com/search", @"https://existing.com"];
        for (NSString *input in rawInputs) {
            NSString *sanitized = [URLValidator sanitizeURL:input];
            NSLog(@"  '%@' -> '%@'", input, sanitized);
        }
        
        // Safe URL creation
        NSLog(@"\nSafe URL creation:");
        NSArray *tricky = @[
            @"https://example.com/path?q=hello world",
            @"https://example.com/สวัสดี",
        ];
        for (NSString *t in tricky) {
            NSURL *safe = [URLValidator safeURLFromString:t];
            NSLog(@"  '%@' -> %@", t, safe);
        }
        
    }
    return 0;
}
```

---

## 28.13 NSURL กับ NSFileManager

```objc
#import <Foundation/Foundation.h>

int main(int argc, const char * argv[]) {
    @autoreleasepool {
        
        NSFileManager *fm = [NSFileManager defaultManager];
        NSError *error = nil;
        
        // สร้าง directory
        NSURL *tempDir = [NSURL fileURLWithPath:NSTemporaryDirectory()];
        NSURL *newDir = [tempDir URLByAppendingPathComponent:@"myapp_temp" isDirectory:YES];
        
        [fm createDirectoryAtURL:newDir 
     withIntermediateDirectories:YES 
                      attributes:nil 
                           error:&error];
        if (!error) NSLog(@"Created dir: %@", newDir.path);
        
        // สร้างไฟล์
        NSURL *fileURL = [newDir URLByAppendingPathComponent:@"data.json"];
        NSString *json = @"{\"hello\": \"world\"}";
        [json writeToURL:fileURL atomically:YES encoding:NSUTF8StringEncoding error:&error];
        if (!error) NSLog(@"Created file: %@", fileURL.path);
        
        // ตรวจสอบว่ามีอยู่
        BOOL exists = [fm fileExistsAtPath:fileURL.path];
        NSLog(@"File exists: %@", exists ? @"YES" : @"NO");
        
        // Copy
        NSURL *copyURL = [newDir URLByAppendingPathComponent:@"data_backup.json"];
        [fm copyItemAtURL:fileURL toURL:copyURL error:&error];
        if (!error) NSLog(@"Copied to: %@", copyURL.path);
        
        // Move/Rename
        NSURL *renamedURL = [newDir URLByAppendingPathComponent:@"data_renamed.json"];
        [fm moveItemAtURL:fileURL toURL:renamedURL error:&error];
        if (!error) NSLog(@"Moved to: %@", renamedURL.path);
        
        // List directory contents
        NSArray *contents = [fm contentsOfDirectoryAtURL:newDir
                                  includingPropertiesForKeys:@[NSURLFileSizeKey]
                                                     options:0
                                                       error:&error];
        NSLog(@"\nDirectory contents:");
        for (NSURL *item in contents) {
            NSNumber *size;
            [item getResourceValue:&size forKey:NSURLFileSizeKey error:nil];
            NSLog(@"  %@ (%@ bytes)", item.lastPathComponent, size);
        }
        
        // Delete
        [fm removeItemAtURL:newDir error:&error];
        if (!error) NSLog(@"\nDeleted: %@", newDir.path);
        
    }
    return 0;
}
```

---

## 28.14 URL Bookmarks (เก็บ reference ไปยังไฟล์อย่างถาวร)

```objc
#import <Foundation/Foundation.h>

int main(int argc, const char * argv[]) {
    @autoreleasepool {
        
        // สร้างไฟล์ทดสอบ
        NSString *testPath = @"/tmp/bookmark_test.txt";
        [@"Bookmark test" writeToFile:testPath atomically:YES 
                           encoding:NSUTF8StringEncoding error:nil];
        
        NSURL *originalURL = [NSURL fileURLWithPath:testPath];
        
        // สร้าง Bookmark Data
        NSError *error = nil;
        NSData *bookmarkData = [originalURL bookmarkDataWithOptions:NSURLBookmarkCreationSuitableForBookmarkFile
                                     includingResourceValuesForKeys:@[NSURLFileSizeKey, NSURLContentModificationDateKey]
                                                      relativeToURL:nil
                                                              error:&error];
        
        if (!error) {
            NSLog(@"Bookmark created: %lu bytes", (unsigned long)bookmarkData.length);
            
            // บันทึก bookmark
            NSString *bookmarkPath = @"/tmp/myfile.bookmark";
            [bookmarkData writeToFile:bookmarkPath atomically:YES];
            
            // อ่าน bookmark กลับมา
            NSData *loadedBookmark = [NSData dataWithContentsOfFile:bookmarkPath];
            
            BOOL isStale = NO;
            NSURL *resolvedURL = [NSURL URLByResolvingBookmarkData:loadedBookmark
                                                           options:NSURLBookmarkResolutionWithoutUI
                                                     relativeToURL:nil
                                               bookmarkDataIsStale:&isStale
                                                             error:&error];
            
            if (!error) {
                NSLog(@"Resolved URL: %@", resolvedURL.path);
                NSLog(@"Is stale: %@", isStale ? @"YES" : @"NO");
                
                // ตรวจสอบว่าชี้ไปไฟล์เดียวกัน
                NSLog(@"Same file: %@", 
                      [resolvedURL.path isEqualToString:originalURL.path] ? @"YES" : @"NO");
            }
        }
        
    }
    return 0;
}
```

---

## 28.15 แบบฝึกหัด (Practice Exercises)

### แบบฝึกหัดที่ 1: URL Parser

```objc
#import <Foundation/Foundation.h>

@interface URLInfo : NSObject
@property (nonatomic, strong) NSString *scheme;
@property (nonatomic, strong) NSString *host;
@property (nonatomic, strong) NSNumber *port;
@property (nonatomic, strong) NSString *path;
@property (nonatomic, strong) NSDictionary *queryParams;
@property (nonatomic, strong) NSString *fragment;
@property (nonatomic, assign) BOOL isValid;
@end

@implementation URLInfo
@end

URLInfo *parseURL(NSString *urlString) {
    URLInfo *info = [[URLInfo alloc] init];
    
    NSURL *url = [NSURL URLWithString:urlString];
    if (!url) {
        info.isValid = NO;
        return info;
    }
    
    info.isValid  = YES;
    info.scheme   = url.scheme;
    info.host     = url.host;
    info.port     = url.port;
    info.path     = url.path;
    info.fragment = url.fragment;
    
    // Parse query
    NSURLComponents *comps = [NSURLComponents componentsWithURL:url 
                                          resolvingAgainstBaseURL:NO];
    NSMutableDictionary *params = [NSMutableDictionary dictionary];
    for (NSURLQueryItem *item in comps.queryItems) {
        params[item.name] = item.value ?: @"";
    }
    info.queryParams = [params copy];
    
    return info;
}

int main(int argc, const char * argv[]) {
    @autoreleasepool {
        
        NSArray *testURLs = @[
            @"https://api.example.com:8443/v2/users?page=1&limit=20&sort=name#results",
            @"ftp://files.example.com/public/data.zip",
            @"mailto:user@example.com",
            @"invalid url!!",
        ];
        
        for (NSString *urlStr in testURLs) {
            URLInfo *info = parseURL(urlStr);
            NSLog(@"\nURL: %@", urlStr);
            if (info.isValid) {
                NSLog(@"  Scheme:   %@", info.scheme);
                NSLog(@"  Host:     %@", info.host);
                NSLog(@"  Port:     %@", info.port ?: @"(default)");
                NSLog(@"  Path:     %@", info.path);
                NSLog(@"  Query:    %@", info.queryParams);
                NSLog(@"  Fragment: %@", info.fragment ?: @"(none)");
            } else {
                NSLog(@"  INVALID URL");
            }
        }
        
    }
    return 0;
}
```

### แบบฝึกหัดที่ 2: File URL Organizer

```objc
#import <Foundation/Foundation.h>

// จัดระเบียบไฟล์ตาม extension
NSMutableDictionary *organizeFilesByExtension(NSArray *fileURLs) {
    NSMutableDictionary *organized = [NSMutableDictionary dictionary];
    
    for (NSURL *url in fileURLs) {
        NSString *ext = url.pathExtension.lowercaseString;
        if (ext.length == 0) ext = @"no_extension";
        
        NSMutableArray *files = organized[ext];
        if (!files) {
            files = [NSMutableArray array];
            organized[ext] = files;
        }
        [files addObject:url];
    }
    
    return organized;
}

int main(int argc, const char * argv[]) {
    @autoreleasepool {
        
        // สร้างรายการไฟล์สมมติ
        NSArray *filePaths = @[
            @"/docs/report.pdf",
            @"/docs/notes.txt",
            @"/images/photo1.jpg",
            @"/images/photo2.png",
            @"/images/icon.svg",
            @"/code/main.m",
            @"/code/utils.h",
            @"/code/Makefile",
            @"/data/users.json",
            @"/data/config.json",
            @"/docs/presentation.pdf",
        ];
        
        NSArray *fileURLs = [filePaths valueForKey:@"self"];
        NSMutableArray *urlArray = [NSMutableArray array];
        for (NSString *path in filePaths) {
            [urlArray addObject:[NSURL fileURLWithPath:path]];
        }
        
        NSDictionary *organized = organizeFilesByExtension(urlArray);
        
        NSLog(@"Files organized by extension:");
        NSArray *extensions = [[organized.allKeys sortedArrayUsingSelector:@selector(compare:)] copy];
        for (NSString *ext in extensions) {
            NSArray *files = organized[ext];
            NSLog(@"\n.%@ (%lu files):", ext, (unsigned long)files.count);
            for (NSURL *fileURL in files) {
                NSLog(@"  %@", fileURL.lastPathComponent);
            }
        }
        
        // หาไฟล์ทั้งหมดใน /docs
        NSArray *docsFiles = [urlArray filteredArrayUsingPredicate:
                              [NSPredicate predicateWithBlock:^BOOL(NSURL *url, NSDictionary *bindings) {
            return [url.path hasPrefix:@"/docs/"];
        }]];
        NSLog(@"\nFiles in /docs: %lu", (unsigned long)docsFiles.count);
        
        // หาขนาดรวม (สมมติ)
        NSLog(@"\nTotal files: %lu", (unsigned long)urlArray.count);
        NSLog(@"Extensions: %@", extensions);
        
    }
    return 0;
}
```

### แบบฝึกหัดที่ 3: REST API URL Builder

```objc
#import <Foundation/Foundation.h>

typedef NS_ENUM(NSInteger, HTTPMethod) {
    HTTPMethodGET,
    HTTPMethodPOST,
    HTTPMethodPUT,
    HTTPMethodPATCH,
    HTTPMethodDELETE
};

@interface RESTRequest : NSObject

@property (nonatomic, strong) NSURL *url;
@property (nonatomic, assign) HTTPMethod method;
@property (nonatomic, strong) NSDictionary *headers;
@property (nonatomic, strong) NSData *body;

+ (instancetype)GETUrl:(NSURL *)url headers:(NSDictionary *)headers;
+ (instancetype)POSTUrl:(NSURL *)url body:(NSDictionary *)body headers:(NSDictionary *)headers;
+ (instancetype)PUTUrl:(NSURL *)url body:(NSDictionary *)body headers:(NSDictionary *)headers;
+ (instancetype)DELETEUrl:(NSURL *)url headers:(NSDictionary *)headers;

- (NSURLRequest *)toURLRequest;
- (NSString *)methodString;

@end

@implementation RESTRequest

+ (instancetype)GETUrl:(NSURL *)url headers:(NSDictionary *)headers {
    RESTRequest *req = [[RESTRequest alloc] init];
    req.url = url;
    req.method = HTTPMethodGET;
    req.headers = headers;
    return req;
}

+ (instancetype)POSTUrl:(NSURL *)url body:(NSDictionary *)body headers:(NSDictionary *)headers {
    RESTRequest *req = [[RESTRequest alloc] init];
    req.url = url;
    req.method = HTTPMethodPOST;
    req.headers = headers;
    req.body = [NSJSONSerialization dataWithJSONObject:body options:0 error:nil];
    return req;
}

+ (instancetype)PUTUrl:(NSURL *)url body:(NSDictionary *)body headers:(NSDictionary *)headers {
    RESTRequest *req = [[RESTRequest alloc] init];
    req.url = url;
    req.method = HTTPMethodPUT;
    req.headers = headers;
    req.body = [NSJSONSerialization dataWithJSONObject:body options:0 error:nil];
    return req;
}

+ (instancetype)DELETEUrl:(NSURL *)url headers:(NSDictionary *)headers {
    RESTRequest *req = [[RESTRequest alloc] init];
    req.url = url;
    req.method = HTTPMethodDELETE;
    req.headers = headers;
    return req;
}

- (NSString *)methodString {
    switch (self.method) {
        case HTTPMethodGET:    return @"GET";
        case HTTPMethodPOST:   return @"POST";
        case HTTPMethodPUT:    return @"PUT";
        case HTTPMethodPATCH:  return @"PATCH";
        case HTTPMethodDELETE: return @"DELETE";
    }
    return @"GET";
}

- (NSURLRequest *)toURLRequest {
    NSMutableURLRequest *request = [NSMutableURLRequest requestWithURL:self.url];
    request.HTTPMethod = [self methodString];
    
    // Default headers
    [request setValue:@"application/json" forHTTPHeaderField:@"Content-Type"];
    [request setValue:@"application/json" forHTTPHeaderField:@"Accept"];
    
    // Custom headers
    for (NSString *key in self.headers) {
        [request setValue:self.headers[key] forHTTPHeaderField:key];
    }
    
    if (self.body) {
        request.HTTPBody = self.body;
        [request setValue:[NSString stringWithFormat:@"%lu", (unsigned long)self.body.length]
       forHTTPHeaderField:@"Content-Length"];
    }
    
    return [request copy];
}

@end

int main(int argc, const char * argv[]) {
    @autoreleasepool {
        
        NSString *apiBase = @"https://api.todoapp.com/v1";
        NSDictionary *authHeaders = @{@"Authorization": @"Bearer token123"};
        
        // GET /todos
        NSURL *todosURL = [NSURL URLWithString:[apiBase stringByAppendingString:@"/todos"]];
        NSURLRequest *getTodos = [[RESTRequest GETUrl:todosURL headers:authHeaders] toURLRequest];
        NSLog(@"GET %@", getTodos.URL);
        NSLog(@"Headers: %@", getTodos.allHTTPHeaderFields);
        
        // POST /todos
        NSDictionary *newTodo = @{@"title": @"Buy groceries", @"done": @NO};
        NSURLRequest *createTodo = [[RESTRequest POSTUrl:todosURL 
                                                   body:newTodo 
                                                headers:authHeaders] toURLRequest];
        NSLog(@"\nPOST %@", createTodo.URL);
        NSString *bodyStr = [[NSString alloc] initWithData:createTodo.HTTPBody 
                                                  encoding:NSUTF8StringEncoding];
        NSLog(@"Body: %@", bodyStr);
        
        // PUT /todos/1
        NSURL *todoURL = [NSURL URLWithString:[apiBase stringByAppendingString:@"/todos/1"]];
        NSDictionary *updateTodo = @{@"title": @"Buy groceries", @"done": @YES};
        NSURLRequest *updateReq = [[RESTRequest PUTUrl:todoURL 
                                                  body:updateTodo 
                                               headers:authHeaders] toURLRequest];
        NSLog(@"\nPUT %@", updateReq.URL);
        
        // DELETE /todos/1
        NSURLRequest *deleteReq = [[RESTRequest DELETEUrl:todoURL 
                                                   headers:authHeaders] toURLRequest];
        NSLog(@"\nDELETE %@", deleteReq.URL);
        
    }
    return 0;
}
```

### แบบฝึกหัดที่ 4: URL Normalizer

```objc
#import <Foundation/Foundation.h>

NSString *normalizeURL(NSString *urlString) {
    // 1. Trim whitespace
    urlString = [urlString stringByTrimmingCharactersInSet:
                 [NSCharacterSet whitespaceAndNewlineCharacterSet]];
    
    // 2. Lowercase scheme และ host
    NSURL *url = [NSURL URLWithString:urlString];
    if (!url) return nil;
    
    NSURLComponents *comps = [NSURLComponents componentsWithURL:url 
                                          resolvingAgainstBaseURL:NO];
    
    comps.scheme = comps.scheme.lowercaseString;
    comps.host   = comps.host.lowercaseString;
    
    // 3. ลบ port default (80 สำหรับ http, 443 สำหรับ https)
    if ([comps.scheme isEqualToString:@"http"] && comps.port.intValue == 80) {
        comps.port = nil;
    } else if ([comps.scheme isEqualToString:@"https"] && comps.port.intValue == 443) {
        comps.port = nil;
    }
    
    // 4. Normalize path (ลบ trailing slash ถ้าเป็น root)
    if ([comps.path isEqualToString:@"/"]) {
        comps.path = @"";
    }
    
    // 5. เรียง query params
    NSArray *sortedItems = [comps.queryItems sortedArrayUsingComparator:
        ^NSComparisonResult(NSURLQueryItem *a, NSURLQueryItem *b) {
            return [a.name compare:b.name];
        }];
    comps.queryItems = sortedItems.count > 0 ? sortedItems : nil;
    
    return comps.URL.absoluteString;
}

int main(int argc, const char * argv[]) {
    @autoreleasepool {
        
        NSArray *urls = @[
            @"  HTTPS://WWW.EXAMPLE.COM/  ",
            @"http://example.com:80/path",
            @"https://example.com:443/path",
            @"https://example.com/search?z=last&a=first&m=middle",
            @"HTTP://Example.COM/Path?B=2&A=1",
        ];
        
        NSLog(@"URL Normalization:");
        for (NSString *url in urls) {
            NSString *normalized = normalizeURL(url);
            NSLog(@"\nOriginal:   '%@'", url);
            NSLog(@"Normalized: '%@'", normalized);
        }
        
    }
    return 0;
}
```

---

## 28.16 NSURL กับ NSURLSession (ตัวอย่างสมบูรณ์)

```objc
#import <Foundation/Foundation.h>

@interface NetworkManager : NSObject
@property (nonatomic, strong) NSURLSession *session;
+ (instancetype)shared;
- (void)fetchJSON:(NSURL *)url completion:(void(^)(NSDictionary *json, NSError *error))completion;
- (void)downloadFile:(NSURL *)url toPath:(NSString *)path completion:(void(^)(NSURL *localURL, NSError *error))completion;
- (void)uploadData:(NSData *)data toURL:(NSURL *)url completion:(void(^)(NSData *response, NSError *error))completion;
@end

@implementation NetworkManager

+ (instancetype)shared {
    static NetworkManager *instance;
    static dispatch_once_t token;
    dispatch_once(&token, ^{ instance = [[NetworkManager alloc] init]; });
    return instance;
}

- (instancetype)init {
    self = [super init];
    if (self) {
        NSURLSessionConfiguration *config = [NSURLSessionConfiguration defaultSessionConfiguration];
        config.timeoutIntervalForRequest = 30.0;
        config.timeoutIntervalForResource = 60.0;
        _session = [NSURLSession sessionWithConfiguration:config];
    }
    return self;
}

- (void)fetchJSON:(NSURL *)url completion:(void(^)(NSDictionary *, NSError *))completion {
    NSURLSessionDataTask *task = [self.session dataTaskWithURL:url
        completionHandler:^(NSData *data, NSURLResponse *response, NSError *error) {
            
        if (error) { completion(nil, error); return; }
        
        NSHTTPURLResponse *httpResponse = (NSHTTPURLResponse *)response;
        if (httpResponse.statusCode < 200 || httpResponse.statusCode >= 300) {
            NSError *statusError = [NSError errorWithDomain:@"HTTP"
                                                       code:httpResponse.statusCode
                                                   userInfo:nil];
            completion(nil, statusError);
            return;
        }
        
        NSError *jsonError = nil;
        NSDictionary *json = [NSJSONSerialization JSONObjectWithData:data
                                                             options:0
                                                               error:&jsonError];
        completion(json, jsonError);
    }];
    [task resume];
}

- (void)downloadFile:(NSURL *)url toPath:(NSString *)path completion:(void(^)(NSURL *, NSError *))completion {
    NSURLSessionDownloadTask *task = [self.session downloadTaskWithURL:url
        completionHandler:^(NSURL *tempURL, NSURLResponse *response, NSError *error) {
            
        if (error) { completion(nil, error); return; }
        
        NSURL *destURL = [NSURL fileURLWithPath:path];
        NSError *moveError = nil;
        NSFileManager *fm = [NSFileManager defaultManager];
        
        // ลบไฟล์เก่าถ้ามี
        if ([fm fileExistsAtPath:path]) {
            [fm removeItemAtURL:destURL error:nil];
        }
        
        [fm moveItemAtURL:tempURL toURL:destURL error:&moveError];
        completion(moveError ? nil : destURL, moveError);
    }];
    [task resume];
}

- (void)uploadData:(NSData *)data toURL:(NSURL *)url completion:(void(^)(NSData *, NSError *))completion {
    NSMutableURLRequest *request = [NSMutableURLRequest requestWithURL:url];
    request.HTTPMethod = @"POST";
    [request setValue:@"application/octet-stream" forHTTPHeaderField:@"Content-Type"];
    
    NSURLSessionUploadTask *task = [self.session uploadTaskWithRequest:request
                                                             fromData:data
                                                    completionHandler:^(NSData *responseData, 
                                                                       NSURLResponse *response, 
                                                                       NSError *error) {
        completion(responseData, error);
    }];
    [task resume];
}

@end

int main(int argc, const char * argv[]) {
    @autoreleasepool {
        
        dispatch_semaphore_t semaphore = dispatch_semaphore_create(0);
        
        // Fetch JSON
        NSURL *jsonURL = [NSURL URLWithString:@"https://httpbin.org/json"];
        
        [[NetworkManager shared] fetchJSON:jsonURL completion:^(NSDictionary *json, NSError *error) {
            if (error) {
                NSLog(@"Fetch error: %@", error.localizedDescription);
            } else {
                NSLog(@"Fetched JSON: %@", json.allKeys);
            }
            dispatch_semaphore_signal(semaphore);
        }];
        
        dispatch_semaphore_wait(semaphore, dispatch_time(DISPATCH_TIME_NOW, 15 * NSEC_PER_SEC));
        
        NSLog(@"Done!");
        
    }
    return 0;
}
```

---

## สรุปบทที่ 28

ในบทนี้เราได้เรียนรู้:

1. **NSURL** - แทน URL ทุกประเภท (HTTP, File, Mailto, Custom Scheme)
2. **URL Components** - scheme, host, port, path, query, fragment
3. **NSURLComponents** - สร้างและแก้ไข URL ทีละส่วน
4. **File System URLs** - ทำงานกับไฟล์และ directories
5. **Relative URLs** - URL ที่อิงจาก base URL
6. **Percent Encoding** - encode ตัวอักษรพิเศษใน URL
7. **NSURLRequest** - สร้าง HTTP requests
8. **Query String** - parse และ build query parameters
9. **API URL Builder** - สร้าง URL สำหรับ REST APIs
10. **URL Validation** - ตรวจสอบและ sanitize URLs
11. **NSFileManager + NSURL** - จัดการไฟล์ด้วย URLs
12. **NSURLSession** - ดึงข้อมูลจากเน็ต, download, upload

### คำถามทบทวน

1. ทำไมควรใช้ NSURLComponents แทนที่จะสร้าง URL string โดยตรง?
2. ความแตกต่างระหว่าง `url.path` และ `url.relativePath` คืออะไร?
3. Percent encoding สำคัญอย่างไรใน URL?
4. ทำไม NSURLSession ถึงดีกว่า dataWithContentsOfURL: สำหรับ network requests?
5. `isFileURL` ตรวจสอบอะไร?

---

**ต่อไป:** Part 29 - NSDate และ NSCalendar
