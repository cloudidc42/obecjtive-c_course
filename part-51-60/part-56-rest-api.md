# ตอนที่ 56: REST API Integration ใน Objective-C

## บทนำ

การสร้าง Layer สำหรับเชื่อมต่อ REST API เป็นทักษะสำคัญของ iOS Developer ที่ดี ในบทนี้เราจะสร้าง **Networking Manager** ที่สมบูรณ์ รองรับการยืนยันตัวตนหลายรูปแบบ, การ Retry อัตโนมัติ, Error Mapping และตัวอย่างการใช้งานจริงกับ Weather API

---

## 56.1 REST API Concepts

### REST คืออะไร?

REST (Representational State Transfer) เป็น Architectural Style สำหรับการสร้าง Web Services มีหลักการสำคัญ 6 ข้อ:

1. **Client-Server** - แยก Client และ Server ออกจากกัน
2. **Stateless** - แต่ละ Request ต้องมีข้อมูลครบสมบูรณ์ ไม่พึ่ง State เดิม
3. **Cacheable** - Response ควรระบุว่า Cacheable หรือไม่
4. **Uniform Interface** - ใช้ HTTP Methods มาตรฐาน
5. **Layered System** - สามารถมี Intermediate Layers เช่น Load Balancer
6. **Code on Demand** (Optional) - Server ส่ง Executable Code ได้

### RESTful URL Design

```
Collection Resources:
GET    /users           - ดึงทุกคน
POST   /users           - สร้างผู้ใช้ใหม่

Individual Resources:
GET    /users/{id}      - ดึงผู้ใช้คนเดียว
PUT    /users/{id}      - แก้ไขทั้งหมด
PATCH  /users/{id}      - แก้ไขบางส่วน
DELETE /users/{id}      - ลบผู้ใช้

Nested Resources:
GET    /users/{id}/orders         - Order ของผู้ใช้
POST   /users/{id}/orders         - สร้าง Order ให้ผู้ใช้
GET    /users/{id}/orders/{oid}   - Order เฉพาะของผู้ใช้

Query Parameters:
GET    /users?page=1&per_page=20&sort=name&order=asc
GET    /products?category=electronics&min_price=100&max_price=500
GET    /posts?search=keyword&tags=ios,swift
```

---

## 56.2 Building a Network Manager Class

### โครงสร้าง Network Manager

```objc
// NetworkManager.h

// Error Domain
extern NSString *const NetworkManagerErrorDomain;

// Error Codes
typedef NS_ENUM(NSInteger, NetworkManagerError) {
    NetworkManagerErrorNoNetwork = 1000,
    NetworkManagerErrorTimeout = 1001,
    NetworkManagerErrorInvalidURL = 1002,
    NetworkManagerErrorInvalidResponse = 1003,
    NetworkManagerErrorUnauthorized = 1004,
    NetworkManagerErrorForbidden = 1005,
    NetworkManagerErrorNotFound = 1006,
    NetworkManagerErrorServerError = 1007,
    NetworkManagerErrorUnknown = 1099
};

typedef void(^NetworkSuccess)(id responseObject, NSHTTPURLResponse *httpResponse);
typedef void(^NetworkFailure)(NSError *error, NSHTTPURLResponse *httpResponse);
typedef void(^NetworkProgress)(float progress);

@interface NetworkManager : NSObject

@property (nonatomic, copy) NSString *baseURL;
@property (nonatomic, assign) NSTimeInterval defaultTimeout;
@property (nonatomic, assign) BOOL loggingEnabled;

+ (instancetype)sharedManager;
+ (instancetype)managerWithBaseURL:(NSString *)baseURL;

// Auth
- (void)setAuthToken:(NSString *)token;
- (void)setBasicAuthWithUsername:(NSString *)username password:(NSString *)password;
- (void)clearAuth;

// Requests
- (NSURLSessionDataTask *)GET:(NSString *)endpoint
                  parameters:(NSDictionary *)parameters
                     success:(NetworkSuccess)success
                     failure:(NetworkFailure)failure;

- (NSURLSessionDataTask *)POST:(NSString *)endpoint
                    parameters:(NSDictionary *)parameters
                       success:(NetworkSuccess)success
                       failure:(NetworkFailure)failure;

- (NSURLSessionDataTask *)PUT:(NSString *)endpoint
                   parameters:(NSDictionary *)parameters
                      success:(NetworkSuccess)success
                      failure:(NetworkFailure)failure;

- (NSURLSessionDataTask *)PATCH:(NSString *)endpoint
                     parameters:(NSDictionary *)parameters
                        success:(NetworkSuccess)success
                        failure:(NetworkFailure)failure;

- (NSURLSessionDataTask *)DELETE:(NSString *)endpoint
                         success:(NetworkSuccess)success
                         failure:(NetworkFailure)failure;

// Upload
- (NSURLSessionUploadTask *)uploadData:(NSData *)data
                            toEndpoint:(NSString *)endpoint
                              mimeType:(NSString *)mimeType
                              progress:(NetworkProgress)progress
                               success:(NetworkSuccess)success
                               failure:(NetworkFailure)failure;

// Download
- (NSURLSessionDownloadTask *)downloadFromEndpoint:(NSString *)endpoint
                                           progress:(NetworkProgress)progress
                                         completion:(void(^)(NSURL *localURL, NSError *error))completion;

// Cancel
- (void)cancelAllRequests;
- (void)cancelRequestsWithURL:(NSString *)url;

@end
```

### Network Manager Implementation

```objc
// NetworkManager.m
#import "NetworkManager.h"

NSString *const NetworkManagerErrorDomain = @"NetworkManagerErrorDomain";

@interface NetworkManager () <NSURLSessionDelegate, NSURLSessionTaskDelegate>

@property (nonatomic, strong) NSURLSession *session;
@property (nonatomic, copy) NSString *authToken;
@property (nonatomic, copy) NSString *basicAuthHeader;
@property (nonatomic, strong) NSMutableDictionary *defaultHeaders;
@property (nonatomic, strong) dispatch_queue_t processingQueue;

@end

@implementation NetworkManager

+ (instancetype)sharedManager {
    static NetworkManager *manager = nil;
    static dispatch_once_t onceToken;
    dispatch_once(&onceToken, ^{
        manager = [[NetworkManager alloc] init];
    });
    return manager;
}

+ (instancetype)managerWithBaseURL:(NSString *)baseURL {
    NetworkManager *manager = [[NetworkManager alloc] init];
    manager.baseURL = baseURL;
    return manager;
}

- (instancetype)init {
    self = [super init];
    if (self) {
        _defaultTimeout = 30.0;
        _loggingEnabled = NO;
        _defaultHeaders = [@{
            @"Accept": @"application/json",
            @"Content-Type": @"application/json"
        } mutableCopy];
        _processingQueue = dispatch_queue_create("com.networkmanager.processing",
                                                  DISPATCH_QUEUE_CONCURRENT);
        [self setupSession];
    }
    return self;
}

- (void)setupSession {
    NSURLSessionConfiguration *config = [NSURLSessionConfiguration defaultSessionConfiguration];
    config.timeoutIntervalForRequest = self.defaultTimeout;
    config.waitsForConnectivity = YES;
    config.HTTPAdditionalHeaders = self.defaultHeaders;
    
    self.session = [NSURLSession sessionWithConfiguration:config
                                                 delegate:self
                                            delegateQueue:nil];
}

#pragma mark - Auth

- (void)setAuthToken:(NSString *)token {
    self.authToken = token;
    self.basicAuthHeader = nil;
}

- (void)setBasicAuthWithUsername:(NSString *)username password:(NSString *)password {
    NSString *credentials = [NSString stringWithFormat:@"%@:%@", username, password];
    NSData *credentialsData = [credentials dataUsingEncoding:NSUTF8StringEncoding];
    NSString *base64 = [credentialsData base64EncodedStringWithOptions:0];
    self.basicAuthHeader = [NSString stringWithFormat:@"Basic %@", base64];
    self.authToken = nil;
}

- (void)clearAuth {
    self.authToken = nil;
    self.basicAuthHeader = nil;
}

#pragma mark - Request Building

- (NSMutableURLRequest *)buildRequestWithEndpoint:(NSString *)endpoint
                                           method:(NSString *)method
                                       parameters:(NSDictionary *)parameters {
    NSString *urlString = [self.baseURL stringByAppendingString:endpoint];
    
    // Add Query Parameters for GET/DELETE
    if ([@[@"GET", @"DELETE"] containsObject:method] && parameters.count > 0) {
        NSURLComponents *components = [NSURLComponents componentsWithString:urlString];
        NSMutableArray *queryItems = [NSMutableArray array];
        [parameters enumerateKeysAndObjectsUsingBlock:^(NSString *key, id value, BOOL *stop) {
            NSString *stringValue = [value isKindOfClass:[NSString class]] ? 
                value : [value description];
            [queryItems addObject:[NSURLQueryItem queryItemWithName:key value:stringValue]];
        }];
        components.queryItems = queryItems;
        urlString = components.URL.absoluteString;
    }
    
    NSURL *url = [NSURL URLWithString:urlString];
    if (!url) {
        NSLog(@"Invalid URL: %@", urlString);
        return nil;
    }
    
    NSMutableURLRequest *request = [NSMutableURLRequest requestWithURL:url];
    [request setHTTPMethod:method];
    [request setTimeoutInterval:self.defaultTimeout];
    
    // Auth Headers
    if (self.authToken) {
        [request setValue:[NSString stringWithFormat:@"Bearer %@", self.authToken]
                   forHTTPHeaderField:@"Authorization"];
    } else if (self.basicAuthHeader) {
        [request setValue:self.basicAuthHeader forHTTPHeaderField:@"Authorization"];
    }
    
    // Default Headers
    [self.defaultHeaders enumerateKeysAndObjectsUsingBlock:^(NSString *key, 
                                                              NSString *value, 
                                                              BOOL *stop) {
        [request setValue:value forHTTPHeaderField:key];
    }];
    
    // Body for POST/PUT/PATCH
    if ([@[@"POST", @"PUT", @"PATCH"] containsObject:method] && parameters.count > 0) {
        NSError *jsonError;
        NSData *bodyData = [NSJSONSerialization dataWithJSONObject:parameters
                                                           options:0
                                                             error:&jsonError];
        if (!jsonError) {
            [request setHTTPBody:bodyData];
        }
    }
    
    // Logging
    if (self.loggingEnabled) {
        NSLog(@"[NetworkManager] %@ %@", method, urlString);
        if (parameters.count > 0) {
            NSLog(@"[NetworkManager] Parameters: %@", parameters);
        }
    }
    
    return request;
}

#pragma mark - HTTP Methods

- (NSURLSessionDataTask *)GET:(NSString *)endpoint
                  parameters:(NSDictionary *)parameters
                     success:(NetworkSuccess)success
                     failure:(NetworkFailure)failure {
    
    NSMutableURLRequest *request = [self buildRequestWithEndpoint:endpoint
                                                           method:@"GET"
                                                       parameters:parameters];
    return [self executeRequest:request success:success failure:failure];
}

- (NSURLSessionDataTask *)POST:(NSString *)endpoint
                    parameters:(NSDictionary *)parameters
                       success:(NetworkSuccess)success
                       failure:(NetworkFailure)failure {
    
    NSMutableURLRequest *request = [self buildRequestWithEndpoint:endpoint
                                                           method:@"POST"
                                                       parameters:parameters];
    return [self executeRequest:request success:success failure:failure];
}

- (NSURLSessionDataTask *)PUT:(NSString *)endpoint
                   parameters:(NSDictionary *)parameters
                      success:(NetworkSuccess)success
                      failure:(NetworkFailure)failure {
    
    NSMutableURLRequest *request = [self buildRequestWithEndpoint:endpoint
                                                           method:@"PUT"
                                                       parameters:parameters];
    return [self executeRequest:request success:success failure:failure];
}

- (NSURLSessionDataTask *)PATCH:(NSString *)endpoint
                     parameters:(NSDictionary *)parameters
                        success:(NetworkSuccess)success
                        failure:(NetworkFailure)failure {
    
    NSMutableURLRequest *request = [self buildRequestWithEndpoint:endpoint
                                                           method:@"PATCH"
                                                       parameters:parameters];
    return [self executeRequest:request success:success failure:failure];
}

- (NSURLSessionDataTask *)DELETE:(NSString *)endpoint
                         success:(NetworkSuccess)success
                         failure:(NetworkFailure)failure {
    
    NSMutableURLRequest *request = [self buildRequestWithEndpoint:endpoint
                                                           method:@"DELETE"
                                                       parameters:nil];
    return [self executeRequest:request success:success failure:failure];
}

#pragma mark - Execute Request

- (NSURLSessionDataTask *)executeRequest:(NSURLRequest *)request
                                 success:(NetworkSuccess)success
                                 failure:(NetworkFailure)failure {
    if (!request) {
        NSError *error = [self errorWithCode:NetworkManagerErrorInvalidURL
                                     message:@"Invalid URL"];
        if (failure) {
            dispatch_async(dispatch_get_main_queue(), ^{ failure(error, nil); });
        }
        return nil;
    }
    
    NSDate *startTime = [NSDate date];
    
    NSURLSessionDataTask *task = [self.session dataTaskWithRequest:request
                                                completionHandler:^(NSData *data,
                                                                    NSURLResponse *response,
                                                                    NSError *error) {
        NSHTTPURLResponse *httpResponse = (NSHTTPURLResponse *)response;
        NSTimeInterval duration = [[NSDate date] timeIntervalSinceDate:startTime];
        
        if (self.loggingEnabled) {
            NSLog(@"[NetworkManager] Response: %ld in %.3fs", 
                  (long)httpResponse.statusCode, duration);
        }
        
        if (error) {
            NSError *mappedError = [self mapNetworkError:error];
            dispatch_async(dispatch_get_main_queue(), ^{
                if (failure) failure(mappedError, httpResponse);
            });
            return;
        }
        
        // Check HTTP Status
        NSInteger statusCode = httpResponse.statusCode;
        if (statusCode < 200 || statusCode >= 300) {
            NSError *httpError = [self mapHTTPError:statusCode data:data];
            dispatch_async(dispatch_get_main_queue(), ^{
                if (failure) failure(httpError, httpResponse);
            });
            return;
        }
        
        // Parse Response
        dispatch_async(self.processingQueue, ^{
            id responseObject = nil;
            if (data.length > 0) {
                NSError *jsonError;
                responseObject = [NSJSONSerialization JSONObjectWithData:data
                                                                options:NSJSONReadingAllowFragments
                                                                  error:&jsonError];
                if (jsonError) {
                    dispatch_async(dispatch_get_main_queue(), ^{
                        if (failure) failure(jsonError, httpResponse);
                    });
                    return;
                }
            }
            
            dispatch_async(dispatch_get_main_queue(), ^{
                if (success) success(responseObject, httpResponse);
            });
        });
    }];
    
    [task resume];
    return task;
}

#pragma mark - Error Mapping

- (NSError *)mapNetworkError:(NSError *)error {
    NSInteger code = NetworkManagerErrorUnknown;
    
    if (error.code == NSURLErrorNotConnectedToInternet ||
        error.code == NSURLErrorNetworkConnectionLost) {
        code = NetworkManagerErrorNoNetwork;
    } else if (error.code == NSURLErrorTimedOut) {
        code = NetworkManagerErrorTimeout;
    }
    
    return [NSError errorWithDomain:NetworkManagerErrorDomain
                               code:code
                           userInfo:@{
        NSLocalizedDescriptionKey: error.localizedDescription,
        NSUnderlyingErrorKey: error
    }];
}

- (NSError *)mapHTTPError:(NSInteger)statusCode data:(NSData *)data {
    NSInteger code = NetworkManagerErrorUnknown;
    NSString *message = @"Request failed";
    
    switch (statusCode) {
        case 401:
            code = NetworkManagerErrorUnauthorized;
            message = @"Unauthorized - Please login again";
            break;
        case 403:
            code = NetworkManagerErrorForbidden;
            message = @"Forbidden - Insufficient permissions";
            break;
        case 404:
            code = NetworkManagerErrorNotFound;
            message = @"Resource not found";
            break;
        case 500 ... 599:
            code = NetworkManagerErrorServerError;
            message = @"Server error - Please try again later";
            break;
        default:
            message = [NSString stringWithFormat:@"HTTP Error: %ld", (long)statusCode];
            break;
    }
    
    // ลอง Parse Error Message จาก Response
    if (data.length > 0) {
        NSError *jsonError;
        NSDictionary *errorDict = [NSJSONSerialization JSONObjectWithData:data
                                                                  options:0
                                                                    error:&jsonError];
        if (!jsonError && [errorDict isKindOfClass:[NSDictionary class]]) {
            NSString *serverMessage = errorDict[@"message"] ?: errorDict[@"error"];
            if (serverMessage) message = serverMessage;
        }
    }
    
    return [NSError errorWithDomain:NetworkManagerErrorDomain
                               code:code
                           userInfo:@{NSLocalizedDescriptionKey: message}];
}

- (NSError *)errorWithCode:(NetworkManagerError)code message:(NSString *)message {
    return [NSError errorWithDomain:NetworkManagerErrorDomain
                               code:code
                           userInfo:@{NSLocalizedDescriptionKey: message}];
}

- (void)cancelAllRequests {
    [self.session getTasksWithCompletionHandler:^(NSArray *dataTasks,
                                                   NSArray *uploadTasks,
                                                   NSArray *downloadTasks) {
        NSArray *allTasks = [[dataTasks arrayByAddingObjectsFromArray:uploadTasks]
                              arrayByAddingObjectsFromArray:downloadTasks];
        for (NSURLSessionTask *task in allTasks) {
            [task cancel];
        }
    }];
}

@end
```

---

## 56.3 Base URL และ Endpoint Construction

```objc
// Endpoint Builder
@interface EndpointBuilder : NSObject

@property (nonatomic, copy) NSString *baseURL;
@property (nonatomic, strong) NSMutableDictionary *defaultQueryParams;

- (instancetype)initWithBaseURL:(NSString *)baseURL;
- (NSString *)buildEndpoint:(NSString *)path;
- (NSString *)buildEndpoint:(NSString *)path queryParams:(NSDictionary *)params;
- (NSString *)buildEndpointWithPathComponents:(NSArray *)components;

@end

@implementation EndpointBuilder

- (instancetype)initWithBaseURL:(NSString *)baseURL {
    self = [super init];
    if (self) {
        _baseURL = baseURL;
        _defaultQueryParams = [NSMutableDictionary dictionary];
    }
    return self;
}

- (NSString *)buildEndpoint:(NSString *)path {
    return [self buildEndpoint:path queryParams:nil];
}

- (NSString *)buildEndpoint:(NSString *)path queryParams:(NSDictionary *)params {
    NSString *fullURL = [self.baseURL stringByAppendingString:path];
    
    NSMutableDictionary *allParams = [NSMutableDictionary dictionaryWithDictionary:self.defaultQueryParams];
    if (params) [allParams addEntriesFromDictionary:params];
    
    if (allParams.count == 0) return fullURL;
    
    NSURLComponents *components = [NSURLComponents componentsWithString:fullURL];
    NSMutableArray *queryItems = [NSMutableArray array];
    [allParams enumerateKeysAndObjectsUsingBlock:^(NSString *key, id value, BOOL *stop) {
        if (![value isKindOfClass:[NSNull class]]) {
            [queryItems addObject:[NSURLQueryItem queryItemWithName:key 
                                                             value:[value description]]];
        }
    }];
    components.queryItems = queryItems;
    return components.URL.absoluteString;
}

- (NSString *)buildEndpointWithPathComponents:(NSArray *)components {
    NSString *path = [components componentsJoinedByString:@"/"];
    return [self buildEndpoint:[@"/" stringByAppendingString:path]];
}

@end

// API Endpoints Constants
@interface APIEndpoints : NSObject

+ (NSString *)users;
+ (NSString *)userWithID:(NSInteger)userID;
+ (NSString *)ordersForUserID:(NSInteger)userID;
+ (NSString *)orderWithID:(NSInteger)orderID forUserID:(NSInteger)userID;

@end

@implementation APIEndpoints

+ (NSString *)users { return @"/users"; }
+ (NSString *)userWithID:(NSInteger)userID {
    return [NSString stringWithFormat:@"/users/%ld", (long)userID];
}
+ (NSString *)ordersForUserID:(NSInteger)userID {
    return [NSString stringWithFormat:@"/users/%ld/orders", (long)userID];
}
+ (NSString *)orderWithID:(NSInteger)orderID forUserID:(NSInteger)userID {
    return [NSString stringWithFormat:@"/users/%ld/orders/%ld", 
            (long)userID, (long)orderID];
}

@end

// การใช้งาน
EndpointBuilder *builder = [[EndpointBuilder alloc] initWithBaseURL:@"https://api.example.com"];
NSString *url = [builder buildEndpoint:[APIEndpoints userWithID:123]
                           queryParams:@{@"include": @"orders,addresses"}];
// Result: https://api.example.com/users/123?include=orders%2Caddresses
```

---

## 56.4 Authentication: Basic Auth, Bearer Token, API Key

```objc
// AuthManager.h
typedef NS_ENUM(NSInteger, AuthType) {
    AuthTypeNone,
    AuthTypeBasic,
    AuthTypeBearer,
    AuthTypeAPIKey,
    AuthTypeOAuth2
};

@interface AuthManager : NSObject

@property (nonatomic, assign, readonly) AuthType authType;
@property (nonatomic, copy, readonly) NSString *token;

+ (instancetype)shared;

// Basic Auth
- (void)setBasicAuthWithUsername:(NSString *)username password:(NSString *)password;

// Bearer Token
- (void)setBearerToken:(NSString *)token;

// API Key
- (void)setAPIKey:(NSString *)key headerName:(NSString *)headerName;

// OAuth 2.0
- (void)setOAuth2Token:(NSString *)accessToken refreshToken:(NSString *)refreshToken;

// Apply to Request
- (void)applyAuthToRequest:(NSMutableURLRequest *)request;

// Clear
- (void)clearAuth;

// Check
- (BOOL)isAuthenticated;

@end

// AuthManager.m
@interface AuthManager ()
@property (nonatomic, assign) AuthType authType;
@property (nonatomic, copy) NSString *token;
@property (nonatomic, copy) NSString *apiKeyHeaderName;
@property (nonatomic, copy) NSString *refreshToken;
@end

@implementation AuthManager

+ (instancetype)shared {
    static AuthManager *instance = nil;
    static dispatch_once_t onceToken;
    dispatch_once(&onceToken, ^{ instance = [[AuthManager alloc] init]; });
    return instance;
}

- (void)setBasicAuthWithUsername:(NSString *)username password:(NSString *)password {
    NSString *credentials = [NSString stringWithFormat:@"%@:%@", username, password];
    NSData *data = [credentials dataUsingEncoding:NSUTF8StringEncoding];
    self.token = [NSString stringWithFormat:@"Basic %@",
                  [data base64EncodedStringWithOptions:0]];
    self.authType = AuthTypeBasic;
    [self saveToKeychain];
}

- (void)setBearerToken:(NSString *)token {
    self.token = token;
    self.authType = AuthTypeBearer;
    [self saveToKeychain];
}

- (void)setAPIKey:(NSString *)key headerName:(NSString *)headerName {
    self.token = key;
    self.apiKeyHeaderName = headerName ?: @"X-API-Key";
    self.authType = AuthTypeAPIKey;
}

- (void)setOAuth2Token:(NSString *)accessToken refreshToken:(NSString *)refreshToken {
    self.token = accessToken;
    self.refreshToken = refreshToken;
    self.authType = AuthTypeOAuth2;
    [self saveToKeychain];
}

- (void)applyAuthToRequest:(NSMutableURLRequest *)request {
    if (!self.token || self.authType == AuthTypeNone) return;
    
    switch (self.authType) {
        case AuthTypeBasic:
            [request setValue:self.token forHTTPHeaderField:@"Authorization"];
            break;
            
        case AuthTypeBearer:
        case AuthTypeOAuth2:
            [request setValue:[NSString stringWithFormat:@"Bearer %@", self.token]
                       forHTTPHeaderField:@"Authorization"];
            break;
            
        case AuthTypeAPIKey:
            [request setValue:self.token 
                       forHTTPHeaderField:self.apiKeyHeaderName ?: @"X-API-Key"];
            break;
            
        case AuthTypeNone:
        default:
            break;
    }
}

- (void)clearAuth {
    self.token = nil;
    self.refreshToken = nil;
    self.authType = AuthTypeNone;
    [self deleteFromKeychain];
}

- (BOOL)isAuthenticated {
    return self.token != nil && self.authType != AuthTypeNone;
}

// Keychain Storage
- (void)saveToKeychain {
    NSDictionary *keychainItem = @{
        (__bridge id)kSecClass: (__bridge id)kSecClassGenericPassword,
        (__bridge id)kSecAttrService: @"com.myapp.auth",
        (__bridge id)kSecAttrAccount: @"authToken",
        (__bridge id)kSecValueData: [self.token dataUsingEncoding:NSUTF8StringEncoding]
    };
    
    SecItemDelete((__bridge CFDictionaryRef)keychainItem);
    OSStatus status = SecItemAdd((__bridge CFDictionaryRef)keychainItem, nil);
    if (status != errSecSuccess) {
        NSLog(@"Failed to save to Keychain: %d", (int)status);
    }
}

- (void)deleteFromKeychain {
    NSDictionary *query = @{
        (__bridge id)kSecClass: (__bridge id)kSecClassGenericPassword,
        (__bridge id)kSecAttrService: @"com.myapp.auth"
    };
    SecItemDelete((__bridge CFDictionaryRef)query);
}

@end
```

---

## 56.5 OAuth 2.0 Flow

```objc
// OAuth2Manager.h
@interface OAuth2Manager : NSObject

@property (nonatomic, copy) NSString *clientID;
@property (nonatomic, copy) NSString *clientSecret;
@property (nonatomic, copy) NSString *redirectURI;
@property (nonatomic, copy) NSString *authorizationURL;
@property (nonatomic, copy) NSString *tokenURL;
@property (nonatomic, copy) NSArray *scopes;

+ (instancetype)shared;

// Authorization URL
- (NSURL *)authorizationURLWithState:(NSString *)state;

// Exchange Code for Token
- (void)exchangeCode:(NSString *)code
          completion:(void(^)(NSDictionary *tokens, NSError *error))completion;

// Refresh Token
- (void)refreshAccessToken:(NSString *)refreshToken
                completion:(void(^)(NSString *newToken, NSError *error))completion;

// Handle Redirect URL
- (BOOL)handleRedirectURL:(NSURL *)url completion:(void(^)(NSDictionary *tokens, NSError *error))completion;

@end

// OAuth2Manager.m
@interface OAuth2Manager ()
@property (nonatomic, strong) NSURLSession *session;
@property (nonatomic, copy) NSString *pendingState;
@property (nonatomic, copy) void(^pendingCompletion)(NSDictionary *, NSError *);
@end

@implementation OAuth2Manager

+ (instancetype)shared {
    static OAuth2Manager *instance = nil;
    static dispatch_once_t onceToken;
    dispatch_once(&onceToken, ^{ instance = [[OAuth2Manager alloc] init]; });
    return instance;
}

- (instancetype)init {
    self = [super init];
    if (self) {
        _session = [NSURLSession sharedSession];
    }
    return self;
}

- (NSURL *)authorizationURLWithState:(NSString *)state {
    self.pendingState = state;
    
    NSURLComponents *components = [NSURLComponents componentsWithString:self.authorizationURL];
    NSMutableArray *queryItems = [NSMutableArray array];
    
    [queryItems addObject:[NSURLQueryItem queryItemWithName:@"client_id" 
                                                     value:self.clientID]];
    [queryItems addObject:[NSURLQueryItem queryItemWithName:@"redirect_uri" 
                                                     value:self.redirectURI]];
    [queryItems addObject:[NSURLQueryItem queryItemWithName:@"response_type" 
                                                     value:@"code"]];
    [queryItems addObject:[NSURLQueryItem queryItemWithName:@"state" 
                                                     value:state]];
    
    if (self.scopes.count > 0) {
        NSString *scopeString = [self.scopes componentsJoinedByString:@" "];
        [queryItems addObject:[NSURLQueryItem queryItemWithName:@"scope" 
                                                         value:scopeString]];
    }
    
    components.queryItems = queryItems;
    return components.URL;
}

- (BOOL)handleRedirectURL:(NSURL *)url 
               completion:(void(^)(NSDictionary *, NSError *))completion {
    
    NSURLComponents *components = [NSURLComponents componentsWithURL:url 
                                             resolvingAgainstBaseURL:NO];
    NSDictionary *params = [self parseQueryItems:components.queryItems];
    
    // ตรวจสอบ State เพื่อป้องกัน CSRF
    NSString *returnedState = params[@"state"];
    if (![returnedState isEqualToString:self.pendingState]) {
        NSError *error = [NSError errorWithDomain:@"OAuth2Error" 
                                             code:1001
                                         userInfo:@{NSLocalizedDescriptionKey: @"State mismatch - possible CSRF attack"}];
        if (completion) completion(nil, error);
        return NO;
    }
    
    // ตรวจสอบ Error
    if (params[@"error"]) {
        NSError *error = [NSError errorWithDomain:@"OAuth2Error"
                                             code:1002
                                         userInfo:@{NSLocalizedDescriptionKey: params[@"error_description"] ?: params[@"error"]}];
        if (completion) completion(nil, error);
        return YES;
    }
    
    // Exchange Code
    NSString *code = params[@"code"];
    if (code) {
        [self exchangeCode:code completion:completion];
        return YES;
    }
    
    return NO;
}

- (void)exchangeCode:(NSString *)code 
          completion:(void(^)(NSDictionary *, NSError *))completion {
    
    NSURL *url = [NSURL URLWithString:self.tokenURL];
    NSMutableURLRequest *request = [NSMutableURLRequest requestWithURL:url];
    [request setHTTPMethod:@"POST"];
    [request setValue:@"application/x-www-form-urlencoded" 
               forHTTPHeaderField:@"Content-Type"];
    
    NSDictionary *params = @{
        @"grant_type": @"authorization_code",
        @"code": code,
        @"redirect_uri": self.redirectURI,
        @"client_id": self.clientID,
        @"client_secret": self.clientSecret
    };
    
    NSString *bodyString = [self urlEncodeParams:params];
    [request setHTTPBody:[bodyString dataUsingEncoding:NSUTF8StringEncoding]];
    
    NSURLSessionDataTask *task = [self.session dataTaskWithRequest:request
                                                completionHandler:^(NSData *data,
                                                                    NSURLResponse *response,
                                                                    NSError *error) {
        if (error) {
            if (completion) {
                dispatch_async(dispatch_get_main_queue(), ^{ completion(nil, error); });
            }
            return;
        }
        
        NSError *jsonError;
        NSDictionary *tokens = [NSJSONSerialization JSONObjectWithData:data
                                                               options:0
                                                                 error:&jsonError];
        
        dispatch_async(dispatch_get_main_queue(), ^{
            if (jsonError) {
                if (completion) completion(nil, jsonError);
            } else {
                // บันทึก Tokens
                if (tokens[@"access_token"]) {
                    [[AuthManager shared] setOAuth2Token:tokens[@"access_token"]
                                           refreshToken:tokens[@"refresh_token"]];
                }
                if (completion) completion(tokens, nil);
            }
        });
    }];
    [task resume];
}

- (void)refreshAccessToken:(NSString *)refreshToken
                completion:(void(^)(NSString *, NSError *))completion {
    
    NSURL *url = [NSURL URLWithString:self.tokenURL];
    NSMutableURLRequest *request = [NSMutableURLRequest requestWithURL:url];
    [request setHTTPMethod:@"POST"];
    [request setValue:@"application/x-www-form-urlencoded" 
               forHTTPHeaderField:@"Content-Type"];
    
    NSDictionary *params = @{
        @"grant_type": @"refresh_token",
        @"refresh_token": refreshToken,
        @"client_id": self.clientID,
        @"client_secret": self.clientSecret
    };
    
    NSString *bodyString = [self urlEncodeParams:params];
    [request setHTTPBody:[bodyString dataUsingEncoding:NSUTF8StringEncoding]];
    
    NSURLSessionDataTask *task = [self.session dataTaskWithRequest:request
                                                completionHandler:^(NSData *data,
                                                                    NSURLResponse *response,
                                                                    NSError *error) {
        if (error) {
            if (completion) {
                dispatch_async(dispatch_get_main_queue(), ^{ completion(nil, error); });
            }
            return;
        }
        
        NSHTTPURLResponse *httpResponse = (NSHTTPURLResponse *)response;
        if (httpResponse.statusCode == 401) {
            // Refresh Token หมดอายุ - ต้อง Login ใหม่
            [[AuthManager shared] clearAuth];
            NSError *authError = [NSError errorWithDomain:@"OAuth2Error"
                                                     code:401
                                                 userInfo:@{NSLocalizedDescriptionKey: @"Session expired, please login again"}];
            dispatch_async(dispatch_get_main_queue(), ^{
                if (completion) completion(nil, authError);
                [[NSNotificationCenter defaultCenter] postNotificationName:@"UserSessionExpired" object:nil];
            });
            return;
        }
        
        NSError *jsonError;
        NSDictionary *result = [NSJSONSerialization JSONObjectWithData:data options:0 error:&jsonError];
        NSString *newToken = result[@"access_token"];
        
        if (newToken) {
            [[AuthManager shared] setBearerToken:newToken];
        }
        
        dispatch_async(dispatch_get_main_queue(), ^{
            if (completion) completion(newToken, jsonError);
        });
    }];
    [task resume];
}

- (NSDictionary *)parseQueryItems:(NSArray<NSURLQueryItem *> *)items {
    NSMutableDictionary *dict = [NSMutableDictionary dictionary];
    for (NSURLQueryItem *item in items) {
        if (item.value) dict[item.name] = item.value;
    }
    return [dict copy];
}

- (NSString *)urlEncodeParams:(NSDictionary *)params {
    NSMutableArray *parts = [NSMutableArray array];
    [params enumerateKeysAndObjectsUsingBlock:^(NSString *key, NSString *value, BOOL *stop) {
        NSString *encodedKey = [key stringByAddingPercentEncodingWithAllowedCharacters:
                                [NSCharacterSet URLQueryAllowedCharacterSet]];
        NSString *encodedValue = [value stringByAddingPercentEncodingWithAllowedCharacters:
                                  [NSCharacterSet URLQueryAllowedCharacterSet]];
        [parts addObject:[NSString stringWithFormat:@"%@=%@", encodedKey, encodedValue]];
    }];
    return [parts componentsJoinedByString:@"&"];
}

@end
```

---

## 56.6 Retry Logic

```objc
// RetryPolicy.h
@interface RetryPolicy : NSObject

@property (nonatomic, assign) NSInteger maxRetries;
@property (nonatomic, assign) NSTimeInterval initialDelay;
@property (nonatomic, assign) NSTimeInterval maxDelay;
@property (nonatomic, assign) double backoffMultiplier;
@property (nonatomic, strong) NSArray<NSNumber *> *retryableStatusCodes;

+ (instancetype)defaultPolicy;
+ (instancetype)aggressivePolicy;
+ (instancetype)conservativePolicy;

- (BOOL)shouldRetryWithError:(NSError *)error attempt:(NSInteger)attempt;
- (NSTimeInterval)delayForAttempt:(NSInteger)attempt;

@end

@implementation RetryPolicy

+ (instancetype)defaultPolicy {
    RetryPolicy *policy = [[RetryPolicy alloc] init];
    policy.maxRetries = 3;
    policy.initialDelay = 1.0;
    policy.maxDelay = 30.0;
    policy.backoffMultiplier = 2.0;
    policy.retryableStatusCodes = @[@408, @429, @500, @502, @503, @504];
    return policy;
}

+ (instancetype)aggressivePolicy {
    RetryPolicy *policy = [[RetryPolicy alloc] init];
    policy.maxRetries = 5;
    policy.initialDelay = 0.5;
    policy.maxDelay = 10.0;
    policy.backoffMultiplier = 1.5;
    policy.retryableStatusCodes = @[@408, @429, @500, @502, @503, @504];
    return policy;
}

+ (instancetype)conservativePolicy {
    RetryPolicy *policy = [[RetryPolicy alloc] init];
    policy.maxRetries = 2;
    policy.initialDelay = 2.0;
    policy.maxDelay = 60.0;
    policy.backoffMultiplier = 3.0;
    policy.retryableStatusCodes = @[@503, @504];
    return policy;
}

- (BOOL)shouldRetryWithError:(NSError *)error attempt:(NSInteger)attempt {
    if (attempt >= self.maxRetries) return NO;
    
    // Network Errors ที่ควร Retry
    if ([error.domain isEqualToString:NSURLErrorDomain]) {
        NSArray *retryableURLErrors = @[
            @(NSURLErrorTimedOut),
            @(NSURLErrorNetworkConnectionLost),
            @(NSURLErrorNotConnectedToInternet),
            @(NSURLErrorCannotConnectToHost),
            @(NSURLErrorDNSLookupFailed)
        ];
        return [retryableURLErrors containsObject:@(error.code)];
    }
    
    // HTTP Status Codes
    if ([error.domain isEqualToString:NetworkManagerErrorDomain]) {
        if (error.code == NetworkManagerErrorTimeout) return YES;
        if (error.code == NetworkManagerErrorServerError) return YES;
    }
    
    return NO;
}

- (NSTimeInterval)delayForAttempt:(NSInteger)attempt {
    // Exponential Backoff with Jitter
    double delay = self.initialDelay * pow(self.backoffMultiplier, attempt);
    delay = MIN(delay, self.maxDelay);
    
    // เพิ่ม Jitter (Random ±25%)
    double jitter = delay * 0.25 * ((double)arc4random() / UINT32_MAX - 0.5) * 2;
    delay += jitter;
    
    return MAX(0, delay);
}

@end

// RetryableNetworkManager
@interface RetryableNetworkManager : NSObject

@property (nonatomic, strong) NetworkManager *networkManager;
@property (nonatomic, strong) RetryPolicy *retryPolicy;

- (void)GET:(NSString *)endpoint
 parameters:(NSDictionary *)parameters
    success:(NetworkSuccess)success
    failure:(NetworkFailure)failure;

@end

@implementation RetryableNetworkManager

- (void)GET:(NSString *)endpoint
 parameters:(NSDictionary *)parameters
    success:(NetworkSuccess)success
    failure:(NetworkFailure)failure {
    
    [self executeWithRetry:0 
                 endpoint:endpoint 
                   method:@"GET" 
               parameters:parameters 
                  success:success 
                  failure:failure];
}

- (void)executeWithRetry:(NSInteger)attempt
               endpoint:(NSString *)endpoint
                 method:(NSString *)method
             parameters:(NSDictionary *)parameters
                success:(NetworkSuccess)success
                failure:(NetworkFailure)failure {
    
    [self.networkManager GET:endpoint 
                 parameters:parameters
                    success:success
                    failure:^(NSError *error, NSHTTPURLResponse *response) {
        
        if ([self.retryPolicy shouldRetryWithError:error attempt:attempt]) {
            NSTimeInterval delay = [self.retryPolicy delayForAttempt:attempt];
            NSLog(@"Retrying request (attempt %ld/%ld) in %.1f seconds...",
                  (long)(attempt + 1), (long)self.retryPolicy.maxRetries, delay);
            
            dispatch_after(dispatch_time(DISPATCH_TIME_NOW, (int64_t)(delay * NSEC_PER_SEC)),
                          dispatch_get_main_queue(), ^{
                [self executeWithRetry:attempt + 1
                             endpoint:endpoint
                               method:method
                           parameters:parameters
                              success:success
                              failure:failure];
            });
        } else {
            if (failure) failure(error, response);
        }
    }];
}

@end
```

---

## 56.7 Complete Example: Weather API Integration

### OpenWeatherMap API Client

```objc
// WeatherModels.h
@interface WeatherCondition : NSObject
@property (nonatomic, copy) NSString *main;
@property (nonatomic, copy) NSString *description;
@property (nonatomic, copy) NSString *iconCode;
@end

@interface Temperature : NSObject
@property (nonatomic, assign) double current;
@property (nonatomic, assign) double feelsLike;
@property (nonatomic, assign) double min;
@property (nonatomic, assign) double max;
@property (nonatomic, assign) NSInteger humidity;
@end

@interface Wind : NSObject
@property (nonatomic, assign) double speed;
@property (nonatomic, assign) NSInteger degrees;
@end

@interface WeatherData : NSObject
@property (nonatomic, copy) NSString *cityName;
@property (nonatomic, strong) NSArray<WeatherCondition *> *conditions;
@property (nonatomic, strong) Temperature *temperature;
@property (nonatomic, strong) Wind *wind;
@property (nonatomic, assign) NSInteger visibility;
@property (nonatomic, strong) NSDate *timestamp;
@property (nonatomic, assign) double latitude;
@property (nonatomic, assign) double longitude;

+ (instancetype)weatherFromDictionary:(NSDictionary *)dict;

// Computed Properties
- (NSString *)temperatureCelsius;
- (NSString *)temperatureFahrenheit;
- (NSString *)windDescription;
- (NSString *)primaryConditionDescription;
- (NSString *)weatherIconURL;

@end

// WeatherModels.m
@implementation WeatherCondition

+ (instancetype)conditionFromDictionary:(NSDictionary *)dict {
    WeatherCondition *condition = [[WeatherCondition alloc] init];
    condition.main = dict[@"main"];
    condition.description = dict[@"description"];
    condition.iconCode = dict[@"icon"];
    return condition;
}

@end

@implementation Temperature

+ (instancetype)temperatureFromDictionary:(NSDictionary *)dict {
    Temperature *temp = [[Temperature alloc] init];
    temp.current = [dict[@"temp"] doubleValue];
    temp.feelsLike = [dict[@"feels_like"] doubleValue];
    temp.min = [dict[@"temp_min"] doubleValue];
    temp.max = [dict[@"temp_max"] doubleValue];
    temp.humidity = [dict[@"humidity"] integerValue];
    return temp;
}

@end

@implementation Wind

+ (instancetype)windFromDictionary:(NSDictionary *)dict {
    Wind *wind = [[Wind alloc] init];
    wind.speed = [dict[@"speed"] doubleValue];
    wind.degrees = [dict[@"deg"] integerValue];
    return wind;
}

@end

@implementation WeatherData

+ (instancetype)weatherFromDictionary:(NSDictionary *)dict {
    WeatherData *weather = [[WeatherData alloc] init];
    weather.cityName = dict[@"name"];
    weather.timestamp = [NSDate dateWithTimeIntervalSince1970:[dict[@"dt"] doubleValue]];
    
    NSDictionary *coord = dict[@"coord"];
    weather.latitude = [coord[@"lat"] doubleValue];
    weather.longitude = [coord[@"lon"] doubleValue];
    weather.visibility = [dict[@"visibility"] integerValue];
    
    // Parse Conditions
    NSArray *condArray = dict[@"weather"];
    NSMutableArray *conditions = [NSMutableArray array];
    for (NSDictionary *condDict in condArray) {
        [conditions addObject:[WeatherCondition conditionFromDictionary:condDict]];
    }
    weather.conditions = [conditions copy];
    
    // Parse Temperature (Kelvin -> Celsius)
    NSDictionary *mainDict = dict[@"main"];
    weather.temperature = [Temperature temperatureFromDictionary:mainDict];
    
    // Parse Wind
    NSDictionary *windDict = dict[@"wind"];
    if (windDict) weather.wind = [Wind windFromDictionary:windDict];
    
    return weather;
}

- (NSString *)temperatureCelsius {
    // OpenWeatherMap ส่งมาเป็น Kelvin
    double celsius = self.temperature.current - 273.15;
    return [NSString stringWithFormat:@"%.1f°C", celsius];
}

- (NSString *)temperatureFahrenheit {
    double celsius = self.temperature.current - 273.15;
    double fahrenheit = (celsius * 9/5) + 32;
    return [NSString stringWithFormat:@"%.1f°F", fahrenheit];
}

- (NSString *)windDescription {
    if (!self.wind) return @"N/A";
    NSArray *directions = @[@"N", @"NNE", @"NE", @"ENE", @"E", @"ESE", @"SE", @"SSE",
                            @"S", @"SSW", @"SW", @"WSW", @"W", @"WNW", @"NW", @"NNW"];
    NSInteger index = ((NSInteger)((self.wind.degrees + 11.25) / 22.5)) % 16;
    return [NSString stringWithFormat:@"%.1f m/s %@", self.wind.speed, directions[index]];
}

- (NSString *)primaryConditionDescription {
    return self.conditions.firstObject.description ?: @"Unknown";
}

- (NSString *)weatherIconURL {
    NSString *iconCode = self.conditions.firstObject.iconCode;
    if (!iconCode) return nil;
    return [NSString stringWithFormat:@"https://openweathermap.org/img/wn/%@@2x.png", iconCode];
}

@end

// WeatherAPIClient.h
@interface WeatherAPIClient : NSObject

+ (instancetype)shared;

// Current Weather
- (void)currentWeatherForCity:(NSString *)city
                   completion:(void(^)(WeatherData *weather, NSError *error))completion;

- (void)currentWeatherForLatitude:(double)lat
                        longitude:(double)lon
                       completion:(void(^)(WeatherData *weather, NSError *error))completion;

// Forecast
- (void)forecastForCity:(NSString *)city
                   days:(NSInteger)days
             completion:(void(^)(NSArray<WeatherData *> *forecasts, NSError *error))completion;

@end

// WeatherAPIClient.m
static NSString *const WeatherAPIBaseURL = @"https://api.openweathermap.org/data/2.5";
static NSString *const WeatherAPIKey = @"YOUR_API_KEY_HERE";

@interface WeatherAPIClient ()
@property (nonatomic, strong) NetworkManager *networkManager;
@end

@implementation WeatherAPIClient

+ (instancetype)shared {
    static WeatherAPIClient *instance = nil;
    static dispatch_once_t onceToken;
    dispatch_once(&onceToken, ^{ instance = [[WeatherAPIClient alloc] init]; });
    return instance;
}

- (instancetype)init {
    self = [super init];
    if (self) {
        self.networkManager = [NetworkManager managerWithBaseURL:WeatherAPIBaseURL];
        self.networkManager.loggingEnabled = YES;
    }
    return self;
}

- (void)currentWeatherForCity:(NSString *)city
                   completion:(void(^)(WeatherData *, NSError *))completion {
    
    NSDictionary *params = @{
        @"q": city,
        @"appid": WeatherAPIKey,
        @"lang": @"th"
    };
    
    [self.networkManager GET:@"/weather"
                 parameters:params
                    success:^(id responseObject, NSHTTPURLResponse *httpResponse) {
        WeatherData *weather = [WeatherData weatherFromDictionary:responseObject];
        if (completion) completion(weather, nil);
    }
                    failure:^(NSError *error, NSHTTPURLResponse *httpResponse) {
        NSError *mappedError = error;
        if (httpResponse.statusCode == 404) {
            mappedError = [NSError errorWithDomain:@"WeatherAPIError"
                                             code:404
                                         userInfo:@{NSLocalizedDescriptionKey: 
                                             [NSString stringWithFormat:@"ไม่พบเมือง '%@'", city]}];
        } else if (httpResponse.statusCode == 401) {
            mappedError = [NSError errorWithDomain:@"WeatherAPIError"
                                             code:401
                                         userInfo:@{NSLocalizedDescriptionKey: @"API Key ไม่ถูกต้อง"}];
        }
        if (completion) completion(nil, mappedError);
    }];
}

- (void)currentWeatherForLatitude:(double)lat
                        longitude:(double)lon
                       completion:(void(^)(WeatherData *, NSError *))completion {
    
    NSDictionary *params = @{
        @"lat": [NSString stringWithFormat:@"%.6f", lat],
        @"lon": [NSString stringWithFormat:@"%.6f", lon],
        @"appid": WeatherAPIKey,
        @"lang": @"th"
    };
    
    [self.networkManager GET:@"/weather"
                 parameters:params
                    success:^(id responseObject, NSHTTPURLResponse *httpResponse) {
        WeatherData *weather = [WeatherData weatherFromDictionary:responseObject];
        if (completion) completion(weather, nil);
    }
                    failure:^(NSError *error, NSHTTPURLResponse *httpResponse) {
        if (completion) completion(nil, error);
    }];
}

- (void)forecastForCity:(NSString *)city
                   days:(NSInteger)days
             completion:(void(^)(NSArray<WeatherData *> *, NSError *))completion {
    
    NSDictionary *params = @{
        @"q": city,
        @"appid": WeatherAPIKey,
        @"cnt": @(days * 8), // 8 intervals per day (3 hours each)
        @"lang": @"th"
    };
    
    [self.networkManager GET:@"/forecast"
                 parameters:params
                    success:^(id responseObject, NSHTTPURLResponse *httpResponse) {
        NSArray *list = responseObject[@"list"];
        
        NSMutableArray *forecasts = [NSMutableArray array];
        for (NSDictionary *item in list) {
            WeatherData *weather = [WeatherData weatherFromDictionary:item];
            if (weather) [forecasts addObject:weather];
        }
        
        if (completion) completion([forecasts copy], nil);
    }
                    failure:^(NSError *error, NSHTTPURLResponse *httpResponse) {
        if (completion) completion(nil, error);
    }];
}

@end

// WeatherViewController.m - การใช้งาน
@implementation WeatherViewController

- (void)viewDidLoad {
    [super viewDidLoad];
    [self loadWeather];
}

- (void)loadWeather {
    [self showLoading:YES];
    
    [[WeatherAPIClient shared] currentWeatherForCity:@"Bangkok"
                                          completion:^(WeatherData *weather, NSError *error) {
        [self showLoading:NO];
        
        if (error) {
            [self showError:error.localizedDescription];
            return;
        }
        
        [self updateUIWithWeather:weather];
    }];
}

- (void)updateUIWithWeather:(WeatherData *)weather {
    self.cityLabel.text = weather.cityName;
    self.temperatureLabel.text = weather.temperatureCelsius;
    self.descriptionLabel.text = [weather.primaryConditionDescription capitalizedString];
    self.humidityLabel.text = [NSString stringWithFormat:@"💧 %ld%%", 
                                (long)weather.temperature.humidity];
    self.windLabel.text = [NSString stringWithFormat:@"💨 %@", weather.windDescription];
    
    // โหลดรูป Weather Icon
    NSString *iconURL = weather.weatherIconURL;
    if (iconURL) {
        [self loadImageFromURL:iconURL intoImageView:self.weatherIconView];
    }
    
    // อัปเดต Background ตาม Condition
    NSString *mainCondition = weather.conditions.firstObject.main;
    [self updateBackgroundForCondition:mainCondition];
}

- (void)updateBackgroundForCondition:(NSString *)condition {
    UIColor *backgroundColor;
    
    if ([condition isEqualToString:@"Clear"]) {
        backgroundColor = [UIColor colorWithRed:0.29 green:0.56 blue:0.89 alpha:1.0];
    } else if ([condition isEqualToString:@"Rain"] || [condition isEqualToString:@"Drizzle"]) {
        backgroundColor = [UIColor colorWithRed:0.27 green:0.35 blue:0.43 alpha:1.0];
    } else if ([condition isEqualToString:@"Snow"]) {
        backgroundColor = [UIColor colorWithRed:0.80 green:0.87 blue:0.94 alpha:1.0];
    } else if ([condition isEqualToString:@"Thunderstorm"]) {
        backgroundColor = [UIColor colorWithRed:0.16 green:0.17 blue:0.20 alpha:1.0];
    } else {
        backgroundColor = [UIColor colorWithRed:0.58 green:0.65 blue:0.73 alpha:1.0];
    }
    
    [UIView animateWithDuration:0.5 animations:^{
        self.view.backgroundColor = backgroundColor;
    }];
}

- (void)loadImageFromURL:(NSString *)urlString intoImageView:(UIImageView *)imageView {
    NSURL *url = [NSURL URLWithString:urlString];
    NSURLSessionDataTask *task = [[NSURLSession sharedSession] dataTaskWithURL:url
                                                             completionHandler:^(NSData *data,
                                                                                 NSURLResponse *response,
                                                                                 NSError *error) {
        if (!error && data) {
            UIImage *image = [UIImage imageWithData:data];
            dispatch_async(dispatch_get_main_queue(), ^{
                imageView.image = image;
            });
        }
    }];
    [task resume];
}

- (void)showLoading:(BOOL)loading {
    if (loading) {
        [self.activityIndicator startAnimating];
        self.contentView.alpha = 0.5;
    } else {
        [self.activityIndicator stopAnimating];
        self.contentView.alpha = 1.0;
    }
}

- (void)showError:(NSString *)message {
    UIAlertController *alert = [UIAlertController 
        alertControllerWithTitle:@"เกิดข้อผิดพลาด"
                         message:message
                  preferredStyle:UIAlertControllerStyleAlert];
    
    [alert addAction:[UIAlertAction actionWithTitle:@"ลองอีกครั้ง"
                                             style:UIAlertActionStyleDefault
                                           handler:^(UIAlertAction *action) {
        [self loadWeather];
    }]];
    
    [alert addAction:[UIAlertAction actionWithTitle:@"ตกลง"
                                             style:UIAlertActionStyleCancel
                                           handler:nil]];
    
    [self presentViewController:alert animated:YES completion:nil];
}

@end
```

---

## 56.8 Generic Response Parsing

```objc
// GenericAPIClient.h - Type-safe API Client
@interface APIResult<T> : NSObject
@property (nonatomic, strong) T value;
@property (nonatomic, strong) NSError *error;
@property (nonatomic, strong) NSHTTPURLResponse *httpResponse;
@property (nonatomic, assign) BOOL isSuccess;
+ (instancetype)successWithValue:(T)value httpResponse:(NSHTTPURLResponse *)response;
+ (instancetype)failureWithError:(NSError *)error httpResponse:(NSHTTPURLResponse *)response;
@end

@interface GenericAPIClient : NSObject

- (void)fetchModel:(Class)modelClass
       fromEndpoint:(NSString *)endpoint
         parameters:(NSDictionary *)parameters
         completion:(void(^)(id model, NSHTTPURLResponse *response, NSError *error))completion;

- (void)fetchModels:(Class)modelClass
       fromEndpoint:(NSString *)endpoint
         parameters:(NSDictionary *)parameters
         completion:(void(^)(NSArray *models, NSHTTPURLResponse *response, NSError *error))completion;

@end

@implementation GenericAPIClient

- (void)fetchModel:(Class)modelClass
       fromEndpoint:(NSString *)endpoint
         parameters:(NSDictionary *)parameters
         completion:(void(^)(id, NSHTTPURLResponse *, NSError *))completion {
    
    [[NetworkManager sharedManager] GET:endpoint
                             parameters:parameters
                                success:^(id responseObject, NSHTTPURLResponse *httpResponse) {
        
        if (![responseObject isKindOfClass:[NSDictionary class]]) {
            NSError *error = [NSError errorWithDomain:@"GenericAPIError"
                                                 code:400
                                             userInfo:@{NSLocalizedDescriptionKey: 
                                                 @"Expected dictionary response"}];
            if (completion) completion(nil, httpResponse, error);
            return;
        }
        
        // Create Model
        SEL selector = @selector(modelFromDictionary:);
        if (![modelClass respondsToSelector:selector]) {
            NSError *error = [NSError errorWithDomain:@"GenericAPIError"
                                                 code:500
                                             userInfo:@{NSLocalizedDescriptionKey: 
                                                 [NSString stringWithFormat:@"%@ does not implement modelFromDictionary:", 
                                                  NSStringFromClass(modelClass)]}];
            if (completion) completion(nil, httpResponse, error);
            return;
        }
        
        id model = [modelClass performSelector:selector withObject:responseObject];
        if (completion) completion(model, httpResponse, nil);
        
    } failure:^(NSError *error, NSHTTPURLResponse *httpResponse) {
        if (completion) completion(nil, httpResponse, error);
    }];
}

- (void)fetchModels:(Class)modelClass
       fromEndpoint:(NSString *)endpoint
         parameters:(NSDictionary *)parameters
         completion:(void(^)(NSArray *, NSHTTPURLResponse *, NSError *))completion {
    
    [[NetworkManager sharedManager] GET:endpoint
                             parameters:parameters
                                success:^(id responseObject, NSHTTPURLResponse *httpResponse) {
        
        NSArray *rawArray;
        if ([responseObject isKindOfClass:[NSArray class]]) {
            rawArray = responseObject;
        } else if ([responseObject isKindOfClass:[NSDictionary class]]) {
            // บาง API Wrap Array ด้วย Key
            for (NSString *key in @[@"data", @"results", @"items", @"list"]) {
                id value = [(NSDictionary *)responseObject objectForKey:key];
                if ([value isKindOfClass:[NSArray class]]) {
                    rawArray = value;
                    break;
                }
            }
        }
        
        if (!rawArray) {
            NSError *error = [NSError errorWithDomain:@"GenericAPIError"
                                                 code:400
                                             userInfo:@{NSLocalizedDescriptionKey: @"Expected array response"}];
            if (completion) completion(nil, httpResponse, error);
            return;
        }
        
        NSMutableArray *models = [NSMutableArray arrayWithCapacity:rawArray.count];
        SEL selector = @selector(modelFromDictionary:);
        
        for (NSDictionary *dict in rawArray) {
            if ([dict isKindOfClass:[NSDictionary class]] && 
                [modelClass respondsToSelector:selector]) {
                id model = [modelClass performSelector:selector withObject:dict];
                if (model) [models addObject:model];
            }
        }
        
        if (completion) completion([models copy], httpResponse, nil);
        
    } failure:^(NSError *error, NSHTTPURLResponse *httpResponse) {
        if (completion) completion(nil, httpResponse, error);
    }];
}

@end

// การใช้งาน
GenericAPIClient *client = [[GenericAPIClient alloc] init];

[client fetchModel:[User class]
       fromEndpoint:@"/users/123"
         parameters:nil
         completion:^(User *user, NSHTTPURLResponse *response, NSError *error) {
    NSLog(@"User: %@", user.name);
}];

[client fetchModels:[Product class]
       fromEndpoint:@"/products"
         parameters:@{@"category": @"electronics", @"page": @1}
         completion:^(NSArray<Product *> *products, NSHTTPURLResponse *response, NSError *error) {
    NSLog(@"Products: %lu", (unsigned long)products.count);
}];
```

---

## 56.9 Refresh Token Implementation

```objc
@interface TokenRefreshManager : NSObject

+ (instancetype)shared;

- (void)refreshIfNeededWithCompletion:(void(^)(BOOL success, NSError *error))completion;
- (BOOL)isTokenExpired;

@end

@interface TokenRefreshManager ()

@property (nonatomic, strong) NSDate *tokenExpiryDate;
@property (nonatomic, assign) BOOL isRefreshing;
@property (nonatomic, strong) NSMutableArray *pendingRequests;

@end

@implementation TokenRefreshManager

+ (instancetype)shared {
    static TokenRefreshManager *instance = nil;
    static dispatch_once_t onceToken;
    dispatch_once(&onceToken, ^{ instance = [[TokenRefreshManager alloc] init]; });
    return instance;
}

- (instancetype)init {
    self = [super init];
    if (self) {
        _pendingRequests = [NSMutableArray array];
        _isRefreshing = NO;
    }
    return self;
}

- (BOOL)isTokenExpired {
    if (!self.tokenExpiryDate) return YES;
    // หมดอายุแล้วหรืออีก 5 นาทีจะหมดอายุ
    return [self.tokenExpiryDate timeIntervalSinceNow] < 300;
}

- (void)refreshIfNeededWithCompletion:(void(^)(BOOL, NSError *))completion {
    if (![self isTokenExpired]) {
        if (completion) completion(YES, nil);
        return;
    }
    
    // ถ้ากำลัง Refresh อยู่ - เพิ่ม Request ลง Queue
    if (self.isRefreshing) {
        if (completion) {
            [self.pendingRequests addObject:[completion copy]];
        }
        return;
    }
    
    self.isRefreshing = YES;
    
    NSString *refreshToken = [[NSUserDefaults standardUserDefaults] 
                              stringForKey:@"refreshToken"];
    if (!refreshToken) {
        NSError *error = [NSError errorWithDomain:@"AuthError"
                                             code:401
                                         userInfo:@{NSLocalizedDescriptionKey: @"No refresh token"}];
        [self notifyPendingRequestsWithSuccess:NO error:error];
        if (completion) completion(NO, error);
        return;
    }
    
    [[OAuth2Manager shared] refreshAccessToken:refreshToken
                                    completion:^(NSString *newToken, NSError *error) {
        self.isRefreshing = NO;
        
        BOOL success = (newToken != nil);
        
        if (success) {
            // อัปเดต Token Expiry (สมมติ 1 ชั่วโมง)
            self.tokenExpiryDate = [NSDate dateWithTimeIntervalSinceNow:3600];
        }
        
        [self notifyPendingRequestsWithSuccess:success error:error];
        if (completion) completion(success, error);
    }];
}

- (void)notifyPendingRequestsWithSuccess:(BOOL)success error:(NSError *)error {
    NSArray *pending = [self.pendingRequests copy];
    [self.pendingRequests removeAllObjects];
    
    for (void(^pendingCompletion)(BOOL, NSError *) in pending) {
        dispatch_async(dispatch_get_main_queue(), ^{
            pendingCompletion(success, error);
        });
    }
}

@end
```

---

## 56.10 แบบฝึกหัด

### แบบฝึกหัดที่ 1: Complete REST Client

สร้าง REST Client สำหรับ JSONPlaceholder API:
```
Base URL: https://jsonplaceholder.typicode.com

Endpoints:
- GET /posts (ดึงทุก Post)
- GET /posts/{id} (ดึง Post เดียว)
- POST /posts (สร้าง Post)
- PUT /posts/{id} (แก้ไข Post)
- DELETE /posts/{id} (ลบ Post)
- GET /users/{id}/posts (ดึง Post ของ User)
```

### แบบฝึกหัดที่ 2: Authentication Flow

สร้าง Auth Flow ที่:
1. Login ด้วย Username/Password
2. บันทึก Token ใน Keychain
3. Auto-refresh เมื่อ Token ใกล้หมดอายุ
4. Logout และลบ Token ทั้งหมด
5. Handle Session Expiry

### แบบฝึกหัดที่ 3: Retry with Backoff

ทดสอบ Retry Logic โดย:
1. Mock Server ที่ตอบ 503 ครั้งแรก 2 ครั้ง แล้วจึง 200
2. ตรวจสอบว่า Client Retry ถูกต้อง
3. ตรวจสอบ Exponential Backoff Timing

### แบบฝึกหัดที่ 4: Weather Dashboard

สร้าง Weather App ที่:
1. แสดงอากาศปัจจุบันของ 5 เมืองในประเทศไทย
2. แสดง 5-Day Forecast
3. Cache ข้อมูล 30 นาที
4. รองรับ Offline Mode
5. แสดง Error Message เป็นภาษาไทย

```objc
// เฉลยโครงสร้างหลัก
@interface WeatherDashboardViewController : UIViewController

@property (nonatomic, strong) NSArray<NSString *> *cities;
@property (nonatomic, strong) NSMutableDictionary<NSString *, WeatherData *> *cachedWeather;
@property (nonatomic, strong) NSDate *lastFetchDate;

- (void)refreshAllCities;
- (BOOL)isCacheValid;
- (void)loadFromCache;
- (void)saveToCache;

@end
```

---

## สรุปบทที่ 56

ในบทนี้เราได้สร้างและเรียนรู้:

1. **REST API Concepts** - หลักการ RESTful และ URL Design
2. **Network Manager** - Class ที่ครบถ้วนสำหรับ HTTP Requests
3. **Endpoint Builder** - การสร้าง URL แบบ Type-safe
4. **Authentication** - Basic Auth, Bearer Token, API Key
5. **OAuth 2.0** - Authorization Code Flow พร้อม State และ PKCE
6. **Refresh Token** - จัดการ Token Expiry อัตโนมัติ
7. **Retry Logic** - Exponential Backoff with Jitter
8. **Error Mapping** - แปลง HTTP Errors เป็น Domain Errors
9. **Generic Parsing** - Type-safe Response Parsing
10. **Weather API** - ตัวอย่างสมบูรณ์กับ OpenWeatherMap API

### สิ่งที่ควรจำ

- ใช้ **Background Thread** สำหรับ JSON Parsing
- กลับมา **Main Thread** เสมอก่อนอัปเดต UI
- บันทึ **Auth Tokens** ใน **Keychain** เท่านั้น
- ใช้ **HTTPS** เสมอในการส่งข้อมูลสำคัญ
- Handle **Error** ทุกกรณีเพื่อ UX ที่ดี
- ใช้ **Cache** เพื่อลด Network Traffic
- ทดสอบกับ **Poor Network** Conditions

---

*จบบทที่ 56 - ถัดไป: ตอนที่ 57 - Core Data*
