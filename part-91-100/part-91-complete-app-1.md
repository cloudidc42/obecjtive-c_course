# ตอนที่ 91: โปรเจกต์สมบูรณ์ - Social Media App

## บทนำ

ในบทนี้เราจะสร้าง Social Media App แบบสมบูรณ์ด้วย Objective-C ที่ครอบคลุมฟีเจอร์หลักของ social network เช่น Facebook หรือ Instagram แอปนี้จะใช้ทุกความรู้ที่เรียนมาตลอดหลักสูตรนี้

---

## 91.1 Architecture Design

### โครงสร้างโปรเจกต์

```
SocialApp/
├── AppDelegate.h / .m
├── SceneDelegate.h / .m
├── Models/
│   ├── SAUser.h / .m
│   ├── SAPost.h / .m
│   ├── SAComment.h / .m
│   ├── SANotification.h / .m
│   └── SAMessage.h / .m
├── Views/
│   ├── Cells/
│   │   ├── SAPostCell.h / .m
│   │   ├── SAUserCell.h / .m
│   │   ├── SACommentCell.h / .m
│   │   └── SANotificationCell.h / .m
│   ├── SAProfileHeaderView.h / .m
│   └── SAImagePickerView.h / .m
├── ViewControllers/
│   ├── Auth/
│   │   ├── SALoginViewController.h / .m
│   │   └── SARegisterViewController.h / .m
│   ├── Feed/
│   │   ├── SAFeedViewController.h / .m
│   │   └── SAPostDetailViewController.h / .m
│   ├── Profile/
│   │   └── SAProfileViewController.h / .m
│   ├── Create/
│   │   └── SACreatePostViewController.h / .m
│   ├── Search/
│   │   └── SASearchViewController.h / .m
│   ├── Notifications/
│   │   └── SANotificationsViewController.h / .m
│   └── Chat/
│       ├── SAChatListViewController.h / .m
│       └── SAChatViewController.h / .m
├── Services/
│   ├── SANetworkManager.h / .m
│   ├── SAAuthService.h / .m
│   ├── SAPostService.h / .m
│   ├── SAUserService.h / .m
│   ├── SANotificationService.h / .m
│   └── SAChatService.h / .m
├── Managers/
│   ├── SAImageManager.h / .m
│   ├── SACacheManager.h / .m
│   └── SASessionManager.h / .m
├── Utilities/
│   ├── SAConstants.h
│   ├── SAHelpers.h / .m
│   └── UIColor+SocialApp.h / .m
└── Resources/
    ├── Assets.xcassets
    └── Main.storyboard
```

### Architecture Pattern: MVC + Service Layer

```
Flow:
ViewController → Service → NetworkManager → API Server
     ↑                          ↓
     └──── Model ←── JSON Parsing ←──────────┘

Cache Layer:
CacheManager ← Service (reads/writes)
     ↓
NSUserDefaults / NSCache / CoreData
```

---

## 91.2 Data Models

### SAUser Model

```objc
// SAUser.h
#import <Foundation/Foundation.h>
#import <UIKit/UIKit.h>

NS_ASSUME_NONNULL_BEGIN

@interface SAUser : NSObject <NSCoding, NSCopying>

@property (nonatomic, copy) NSString *userID;
@property (nonatomic, copy) NSString *username;
@property (nonatomic, copy) NSString *email;
@property (nonatomic, copy, nullable) NSString *displayName;
@property (nonatomic, copy, nullable) NSString *bio;
@property (nonatomic, copy, nullable) NSString *avatarURL;
@property (nonatomic, strong) NSDate *createdAt;
@property (nonatomic, assign) NSInteger followersCount;
@property (nonatomic, assign) NSInteger followingCount;
@property (nonatomic, assign) NSInteger postsCount;
@property (nonatomic, assign, getter=isFollowing) BOOL following;
@property (nonatomic, assign, getter=isPrivate) BOOL private;

+ (instancetype)userFromDictionary:(NSDictionary *)dictionary;
- (NSDictionary *)toDictionary;

@end

NS_ASSUME_NONNULL_END
```

```objc
// SAUser.m
#import "SAUser.h"

@implementation SAUser

+ (instancetype)userFromDictionary:(NSDictionary *)dictionary {
    if (!dictionary || ![dictionary isKindOfClass:[NSDictionary class]]) return nil;
    
    SAUser *user = [[SAUser alloc] init];
    user.userID = dictionary[@"id"] ?: @"";
    user.username = dictionary[@"username"] ?: @"";
    user.email = dictionary[@"email"] ?: @"";
    user.displayName = dictionary[@"display_name"];
    user.bio = dictionary[@"bio"];
    user.avatarURL = dictionary[@"avatar_url"];
    
    NSTimeInterval createdTimestamp = [dictionary[@"created_at"] doubleValue];
    user.createdAt = [NSDate dateWithTimeIntervalSince1970:createdTimestamp];
    
    user.followersCount = [dictionary[@"followers_count"] integerValue];
    user.followingCount = [dictionary[@"following_count"] integerValue];
    user.postsCount = [dictionary[@"posts_count"] integerValue];
    user.following = [dictionary[@"is_following"] boolValue];
    user.private = [dictionary[@"is_private"] boolValue];
    
    return user;
}

- (NSDictionary *)toDictionary {
    NSMutableDictionary *dict = [NSMutableDictionary dictionary];
    dict[@"id"] = self.userID ?: @"";
    dict[@"username"] = self.username ?: @"";
    dict[@"email"] = self.email ?: @"";
    if (self.displayName) dict[@"display_name"] = self.displayName;
    if (self.bio) dict[@"bio"] = self.bio;
    if (self.avatarURL) dict[@"avatar_url"] = self.avatarURL;
    dict[@"created_at"] = @([self.createdAt timeIntervalSince1970]);
    dict[@"followers_count"] = @(self.followersCount);
    dict[@"following_count"] = @(self.followingCount);
    dict[@"posts_count"] = @(self.postsCount);
    dict[@"is_following"] = @(self.following);
    dict[@"is_private"] = @(self.private);
    return [dict copy];
}

// NSCoding
- (void)encodeWithCoder:(NSCoder *)coder {
    [coder encodeObject:self.userID forKey:@"userID"];
    [coder encodeObject:self.username forKey:@"username"];
    [coder encodeObject:self.email forKey:@"email"];
    [coder encodeObject:self.displayName forKey:@"displayName"];
    [coder encodeObject:self.bio forKey:@"bio"];
    [coder encodeObject:self.avatarURL forKey:@"avatarURL"];
    [coder encodeObject:self.createdAt forKey:@"createdAt"];
    [coder encodeInteger:self.followersCount forKey:@"followersCount"];
    [coder encodeInteger:self.followingCount forKey:@"followingCount"];
    [coder encodeInteger:self.postsCount forKey:@"postsCount"];
    [coder encodeBool:self.following forKey:@"following"];
    [coder encodeBool:self.private forKey:@"private"];
}

- (instancetype)initWithCoder:(NSCoder *)coder {
    if (self = [super init]) {
        _userID = [coder decodeObjectForKey:@"userID"];
        _username = [coder decodeObjectForKey:@"username"];
        _email = [coder decodeObjectForKey:@"email"];
        _displayName = [coder decodeObjectForKey:@"displayName"];
        _bio = [coder decodeObjectForKey:@"bio"];
        _avatarURL = [coder decodeObjectForKey:@"avatarURL"];
        _createdAt = [coder decodeObjectForKey:@"createdAt"];
        _followersCount = [coder decodeIntegerForKey:@"followersCount"];
        _followingCount = [coder decodeIntegerForKey:@"followingCount"];
        _postsCount = [coder decodeIntegerForKey:@"postsCount"];
        _following = [coder decodeBoolForKey:@"following"];
        _private = [coder decodeBoolForKey:@"private"];
    }
    return self;
}

// NSCopying
- (id)copyWithZone:(NSZone *)zone {
    SAUser *copy = [[SAUser allocWithZone:zone] init];
    copy.userID = self.userID;
    copy.username = self.username;
    copy.email = self.email;
    copy.displayName = self.displayName;
    copy.bio = self.bio;
    copy.avatarURL = self.avatarURL;
    copy.createdAt = self.createdAt;
    copy.followersCount = self.followersCount;
    copy.followingCount = self.followingCount;
    copy.postsCount = self.postsCount;
    copy.following = self.following;
    copy.private = self.private;
    return copy;
}

@end
```

### SAPost Model

```objc
// SAPost.h
#import <Foundation/Foundation.h>
#import "SAUser.h"

NS_ASSUME_NONNULL_BEGIN

typedef NS_ENUM(NSInteger, SAPostMediaType) {
    SAPostMediaTypeNone,
    SAPostMediaTypeImage,
    SAPostMediaTypeVideo
};

@interface SAPost : NSObject <NSCoding>

@property (nonatomic, copy) NSString *postID;
@property (nonatomic, strong) SAUser *author;
@property (nonatomic, copy) NSString *content;
@property (nonatomic, copy, nullable) NSArray<NSString *> *imageURLs;
@property (nonatomic, copy, nullable) NSString *videoURL;
@property (nonatomic, assign) SAPostMediaType mediaType;
@property (nonatomic, strong) NSDate *createdAt;
@property (nonatomic, assign) NSInteger likesCount;
@property (nonatomic, assign) NSInteger commentsCount;
@property (nonatomic, assign) NSInteger sharesCount;
@property (nonatomic, assign, getter=isLiked) BOOL liked;
@property (nonatomic, assign, getter=isBookmarked) BOOL bookmarked;
@property (nonatomic, copy, nullable) NSArray<NSString *> *hashtags;
@property (nonatomic, copy, nullable) NSArray<SAUser *> *taggedUsers;

+ (instancetype)postFromDictionary:(NSDictionary *)dictionary;
- (NSDictionary *)toDictionary;
- (NSString *)timeAgoString;

@end

NS_ASSUME_NONNULL_END
```

```objc
// SAPost.m
#import "SAPost.h"

@implementation SAPost

+ (instancetype)postFromDictionary:(NSDictionary *)dictionary {
    if (!dictionary) return nil;
    
    SAPost *post = [[SAPost alloc] init];
    post.postID = dictionary[@"id"] ?: @"";
    
    NSDictionary *authorDict = dictionary[@"author"];
    post.author = authorDict ? [SAUser userFromDictionary:authorDict] : [[SAUser alloc] init];
    
    post.content = dictionary[@"content"] ?: @"";
    post.imageURLs = dictionary[@"image_urls"];
    post.videoURL = dictionary[@"video_url"];
    
    NSString *mediaTypeStr = dictionary[@"media_type"];
    if ([mediaTypeStr isEqualToString:@"image"]) {
        post.mediaType = SAPostMediaTypeImage;
    } else if ([mediaTypeStr isEqualToString:@"video"]) {
        post.mediaType = SAPostMediaTypeVideo;
    } else {
        post.mediaType = SAPostMediaTypeNone;
    }
    
    NSTimeInterval timestamp = [dictionary[@"created_at"] doubleValue];
    post.createdAt = [NSDate dateWithTimeIntervalSince1970:timestamp];
    
    post.likesCount = [dictionary[@"likes_count"] integerValue];
    post.commentsCount = [dictionary[@"comments_count"] integerValue];
    post.sharesCount = [dictionary[@"shares_count"] integerValue];
    post.liked = [dictionary[@"is_liked"] boolValue];
    post.bookmarked = [dictionary[@"is_bookmarked"] boolValue];
    post.hashtags = dictionary[@"hashtags"];
    
    NSArray *taggedUsersData = dictionary[@"tagged_users"];
    if (taggedUsersData) {
        NSMutableArray<SAUser *> *taggedUsers = [NSMutableArray array];
        for (NSDictionary *userData in taggedUsersData) {
            SAUser *user = [SAUser userFromDictionary:userData];
            if (user) [taggedUsers addObject:user];
        }
        post.taggedUsers = [taggedUsers copy];
    }
    
    return post;
}

- (NSDictionary *)toDictionary {
    NSMutableDictionary *dict = [NSMutableDictionary dictionary];
    dict[@"id"] = self.postID;
    dict[@"author"] = [self.author toDictionary];
    dict[@"content"] = self.content;
    if (self.imageURLs) dict[@"image_urls"] = self.imageURLs;
    if (self.videoURL) dict[@"video_url"] = self.videoURL;
    dict[@"created_at"] = @([self.createdAt timeIntervalSince1970]);
    dict[@"likes_count"] = @(self.likesCount);
    dict[@"comments_count"] = @(self.commentsCount);
    dict[@"shares_count"] = @(self.sharesCount);
    dict[@"is_liked"] = @(self.liked);
    dict[@"is_bookmarked"] = @(self.bookmarked);
    if (self.hashtags) dict[@"hashtags"] = self.hashtags;
    return [dict copy];
}

- (NSString *)timeAgoString {
    NSTimeInterval seconds = -[self.createdAt timeIntervalSinceNow];
    
    if (seconds < 60) {
        return @"เมื่อกี้";
    } else if (seconds < 3600) {
        NSInteger minutes = (NSInteger)(seconds / 60);
        return [NSString stringWithFormat:@"%ld นาทีที่แล้ว", (long)minutes];
    } else if (seconds < 86400) {
        NSInteger hours = (NSInteger)(seconds / 3600);
        return [NSString stringWithFormat:@"%ld ชั่วโมงที่แล้ว", (long)hours];
    } else if (seconds < 604800) {
        NSInteger days = (NSInteger)(seconds / 86400);
        return [NSString stringWithFormat:@"%ld วันที่แล้ว", (long)days];
    } else {
        NSDateFormatter *formatter = [[NSDateFormatter alloc] init];
        [formatter setDateFormat:@"d MMM yyyy"];
        [formatter setLocale:[[NSLocale alloc] initWithLocaleIdentifier:@"th_TH"]];
        return [formatter stringFromDate:self.createdAt];
    }
}

// NSCoding
- (void)encodeWithCoder:(NSCoder *)coder {
    [coder encodeObject:self.postID forKey:@"postID"];
    [coder encodeObject:self.author forKey:@"author"];
    [coder encodeObject:self.content forKey:@"content"];
    [coder encodeObject:self.imageURLs forKey:@"imageURLs"];
    [coder encodeObject:self.videoURL forKey:@"videoURL"];
    [coder encodeInteger:self.mediaType forKey:@"mediaType"];
    [coder encodeObject:self.createdAt forKey:@"createdAt"];
    [coder encodeInteger:self.likesCount forKey:@"likesCount"];
    [coder encodeInteger:self.commentsCount forKey:@"commentsCount"];
    [coder encodeInteger:self.sharesCount forKey:@"sharesCount"];
    [coder encodeBool:self.liked forKey:@"liked"];
    [coder encodeBool:self.bookmarked forKey:@"bookmarked"];
    [coder encodeObject:self.hashtags forKey:@"hashtags"];
}

- (instancetype)initWithCoder:(NSCoder *)coder {
    if (self = [super init]) {
        _postID = [coder decodeObjectForKey:@"postID"];
        _author = [coder decodeObjectForKey:@"author"];
        _content = [coder decodeObjectForKey:@"content"];
        _imageURLs = [coder decodeObjectForKey:@"imageURLs"];
        _videoURL = [coder decodeObjectForKey:@"videoURL"];
        _mediaType = [coder decodeIntegerForKey:@"mediaType"];
        _createdAt = [coder decodeObjectForKey:@"createdAt"];
        _likesCount = [coder decodeIntegerForKey:@"likesCount"];
        _commentsCount = [coder decodeIntegerForKey:@"commentsCount"];
        _sharesCount = [coder decodeIntegerForKey:@"sharesCount"];
        _liked = [coder decodeBoolForKey:@"liked"];
        _bookmarked = [coder decodeBoolForKey:@"bookmarked"];
        _hashtags = [coder decodeObjectForKey:@"hashtags"];
    }
    return self;
}

@end
```

---

## 91.3 Networking Layer

### SANetworkManager

```objc
// SANetworkManager.h
#import <Foundation/Foundation.h>

NS_ASSUME_NONNULL_BEGIN

typedef void (^SANetworkCompletion)(NSDictionary * _Nullable response, NSError * _Nullable error);
typedef void (^SANetworkArrayCompletion)(NSArray * _Nullable response, NSError * _Nullable error);
typedef void (^SANetworkUploadCompletion)(NSString * _Nullable urlString, NSError * _Nullable error);

typedef NS_ENUM(NSInteger, SAHTTPMethod) {
    SAHTTPMethodGET,
    SAHTTPMethodPOST,
    SAHTTPMethodPUT,
    SAHTTPMethodDELETE,
    SAHTTPMethodPATCH
};

@interface SANetworkManager : NSObject

@property (nonatomic, copy) NSString *baseURL;
@property (nonatomic, copy, nullable) NSString *authToken;

+ (instancetype)sharedManager;

- (NSURLSessionDataTask *)requestWithPath:(NSString *)path
                                   method:(SAHTTPMethod)method
                               parameters:(nullable NSDictionary *)parameters
                               completion:(SANetworkCompletion)completion;

- (NSURLSessionDataTask *)uploadImage:(UIImage *)image
                                 path:(NSString *)path
                            completion:(SANetworkUploadCompletion)completion;

@end

NS_ASSUME_NONNULL_END
```

```objc
// SANetworkManager.m
#import "SANetworkManager.h"
#import "SAConstants.h"
#import <UIKit/UIKit.h>

@interface SANetworkManager ()
@property (nonatomic, strong) NSURLSession *session;
@property (nonatomic, strong) dispatch_queue_t processingQueue;
@end

@implementation SANetworkManager

+ (instancetype)sharedManager {
    static SANetworkManager *instance = nil;
    static dispatch_once_t onceToken;
    dispatch_once(&onceToken, ^{
        instance = [[SANetworkManager alloc] init];
    });
    return instance;
}

- (instancetype)init {
    if (self = [super init]) {
        NSURLSessionConfiguration *config = [NSURLSessionConfiguration defaultSessionConfiguration];
        config.timeoutIntervalForRequest = 30.0;
        config.timeoutIntervalForResource = 60.0;
        config.HTTPAdditionalHeaders = @{
            @"Content-Type": @"application/json",
            @"Accept": @"application/json",
            @"App-Version": [[NSBundle mainBundle] objectForInfoDictionaryKey:@"CFBundleShortVersionString"] ?: @"1.0"
        };
        
        _session = [NSURLSession sessionWithConfiguration:config];
        _baseURL = SA_API_BASE_URL;
        _processingQueue = dispatch_queue_create("com.socialapp.networking", DISPATCH_QUEUE_CONCURRENT);
    }
    return self;
}

- (NSURLSessionDataTask *)requestWithPath:(NSString *)path
                                   method:(SAHTTPMethod)method
                               parameters:(nullable NSDictionary *)parameters
                               completion:(SANetworkCompletion)completion {
    NSURL *url = [NSURL URLWithString:[self.baseURL stringByAppendingString:path]];
    NSMutableURLRequest *request = [NSMutableURLRequest requestWithURL:url];
    
    // Set method
    switch (method) {
        case SAHTTPMethodGET:    request.HTTPMethod = @"GET"; break;
        case SAHTTPMethodPOST:   request.HTTPMethod = @"POST"; break;
        case SAHTTPMethodPUT:    request.HTTPMethod = @"PUT"; break;
        case SAHTTPMethodDELETE: request.HTTPMethod = @"DELETE"; break;
        case SAHTTPMethodPATCH:  request.HTTPMethod = @"PATCH"; break;
    }
    
    // Auth header
    if (self.authToken) {
        [request setValue:[NSString stringWithFormat:@"Bearer %@", self.authToken] 
       forHTTPHeaderField:@"Authorization"];
    }
    
    // Parameters
    if (parameters) {
        if (method == SAHTTPMethodGET) {
            // GET: parameters ใน query string
            NSURLComponents *components = [NSURLComponents componentsWithURL:url resolvingAgainstBaseURL:NO];
            NSMutableArray *queryItems = [NSMutableArray array];
            [parameters enumerateKeysAndObjectsUsingBlock:^(id key, id value, BOOL *stop) {
                [queryItems addObject:[NSURLQueryItem queryItemWithName:[key description] 
                                                                  value:[value description]]];
            }];
            components.queryItems = queryItems;
            request.URL = components.URL;
        } else {
            // POST/PUT/PATCH: parameters ใน body
            NSData *bodyData = [NSJSONSerialization dataWithJSONObject:parameters 
                                                               options:0 
                                                                 error:nil];
            request.HTTPBody = bodyData;
        }
    }
    
    NSURLSessionDataTask *task = [self.session dataTaskWithRequest:request 
                                                 completionHandler:^(NSData *data, 
                                                                    NSURLResponse *response, 
                                                                    NSError *error) {
        dispatch_async(dispatch_get_main_queue(), ^{
            if (error) {
                if (completion) completion(nil, error);
                return;
            }
            
            NSHTTPURLResponse *httpResponse = (NSHTTPURLResponse *)response;
            
            // Handle HTTP error status codes
            if (httpResponse.statusCode < 200 || httpResponse.statusCode >= 300) {
                NSDictionary *errorResponse = nil;
                if (data) {
                    errorResponse = [NSJSONSerialization JSONObjectWithData:data 
                                                                    options:0 
                                                                      error:nil];
                }
                
                NSString *message = errorResponse[@"message"] ?: 
                                   [NSHTTPURLResponse localizedStringForStatusCode:httpResponse.statusCode];
                NSError *httpError = [NSError errorWithDomain:@"SANetworkError"
                                                         code:httpResponse.statusCode
                                                     userInfo:@{NSLocalizedDescriptionKey: message}];
                if (completion) completion(nil, httpError);
                return;
            }
            
            if (!data) {
                if (completion) completion(nil, nil);
                return;
            }
            
            NSError *parseError = nil;
            id jsonObject = [NSJSONSerialization JSONObjectWithData:data 
                                                            options:0 
                                                              error:&parseError];
            
            if (parseError) {
                if (completion) completion(nil, parseError);
                return;
            }
            
            if ([jsonObject isKindOfClass:[NSDictionary class]]) {
                if (completion) completion(jsonObject, nil);
            } else {
                NSError *typeError = [NSError errorWithDomain:@"SANetworkError"
                                                         code:-1
                                                     userInfo:@{NSLocalizedDescriptionKey: @"Invalid response format"}];
                if (completion) completion(nil, typeError);
            }
        });
    }];
    
    [task resume];
    return task;
}

- (NSURLSessionDataTask *)uploadImage:(UIImage *)image
                                 path:(NSString *)path
                            completion:(SANetworkUploadCompletion)completion {
    NSURL *url = [NSURL URLWithString:[self.baseURL stringByAppendingString:path]];
    
    // สร้าง multipart form data
    NSString *boundary = [NSUUID UUID].UUIDString;
    NSMutableURLRequest *request = [NSMutableURLRequest requestWithURL:url];
    request.HTTPMethod = @"POST";
    [request setValue:[NSString stringWithFormat:@"multipart/form-data; boundary=%@", boundary] 
   forHTTPHeaderField:@"Content-Type"];
    
    if (self.authToken) {
        [request setValue:[NSString stringWithFormat:@"Bearer %@", self.authToken] 
       forHTTPHeaderField:@"Authorization"];
    }
    
    NSMutableData *bodyData = [NSMutableData data];
    
    // Image data
    NSData *imageData = UIImageJPEGRepresentation(image, 0.8);
    if (!imageData) {
        NSError *error = [NSError errorWithDomain:@"SANetworkError"
                                             code:-1
                                         userInfo:@{NSLocalizedDescriptionKey: @"Cannot convert image"}];
        if (completion) completion(nil, error);
        return nil;
    }
    
    [bodyData appendData:[[NSString stringWithFormat:@"--%@\r\n", boundary] 
                           dataUsingEncoding:NSUTF8StringEncoding]];
    [bodyData appendData:[@"Content-Disposition: form-data; name=\"image\"; filename=\"photo.jpg\"\r\n" 
                           dataUsingEncoding:NSUTF8StringEncoding]];
    [bodyData appendData:[@"Content-Type: image/jpeg\r\n\r\n" 
                           dataUsingEncoding:NSUTF8StringEncoding]];
    [bodyData appendData:imageData];
    [bodyData appendData:[[NSString stringWithFormat:@"\r\n--%@--\r\n", boundary] 
                           dataUsingEncoding:NSUTF8StringEncoding]];
    
    request.HTTPBody = bodyData;
    
    NSURLSessionDataTask *task = [self.session dataTaskWithRequest:request 
                                                 completionHandler:^(NSData *data, 
                                                                    NSURLResponse *response, 
                                                                    NSError *error) {
        dispatch_async(dispatch_get_main_queue(), ^{
            if (error) {
                if (completion) completion(nil, error);
                return;
            }
            
            NSError *parseError = nil;
            NSDictionary *json = [NSJSONSerialization JSONObjectWithData:data 
                                                                 options:0 
                                                                   error:&parseError];
            
            if (parseError || !json[@"url"]) {
                NSError *respError = [NSError errorWithDomain:@"SANetworkError"
                                                         code:-1
                                                     userInfo:@{NSLocalizedDescriptionKey: @"Upload failed"}];
                if (completion) completion(nil, respError);
                return;
            }
            
            if (completion) completion(json[@"url"], nil);
        });
    }];
    
    [task resume];
    return task;
}

@end
```

---

## 91.4 Authentication

### SAAuthService

```objc
// SAAuthService.h
#import <Foundation/Foundation.h>
#import "SAUser.h"

NS_ASSUME_NONNULL_BEGIN

extern NSNotificationName const SAAuthDidLoginNotification;
extern NSNotificationName const SAAuthDidLogoutNotification;

@interface SAAuthService : NSObject

@property (nonatomic, strong, readonly, nullable) SAUser *currentUser;
@property (nonatomic, assign, readonly, getter=isAuthenticated) BOOL authenticated;

+ (instancetype)sharedService;

- (void)loginWithEmail:(NSString *)email
              password:(NSString *)password
            completion:(void (^)(SAUser * _Nullable user, NSError * _Nullable error))completion;

- (void)registerWithUsername:(NSString *)username
                       email:(NSString *)email
                    password:(NSString *)password
                  completion:(void (^)(SAUser * _Nullable user, NSError * _Nullable error))completion;

- (void)logoutWithCompletion:(nullable void (^)(void))completion;
- (BOOL)restoreSession;

@end

NS_ASSUME_NONNULL_END
```

```objc
// SAAuthService.m
#import "SAAuthService.h"
#import "SANetworkManager.h"
#import "SACacheManager.h"

NSNotificationName const SAAuthDidLoginNotification = @"SAAuthDidLoginNotification";
NSNotificationName const SAAuthDidLogoutNotification = @"SAAuthDidLogoutNotification";

static NSString * const kAuthTokenKey = @"SAAuthToken";
static NSString * const kCurrentUserKey = @"SACurrentUser";

@interface SAAuthService ()
@property (nonatomic, strong, readwrite) SAUser *currentUser;
@property (nonatomic, assign, readwrite, getter=isAuthenticated) BOOL authenticated;
@end

@implementation SAAuthService

+ (instancetype)sharedService {
    static SAAuthService *instance = nil;
    static dispatch_once_t onceToken;
    dispatch_once(&onceToken, ^{
        instance = [[SAAuthService alloc] init];
    });
    return instance;
}

- (instancetype)init {
    if (self = [super init]) {
        [self restoreSession];
    }
    return self;
}

- (BOOL)restoreSession {
    NSString *token = [[NSUserDefaults standardUserDefaults] stringForKey:kAuthTokenKey];
    NSData *userData = [[NSUserDefaults standardUserDefaults] dataForKey:kCurrentUserKey];
    
    if (token && userData) {
        NSError *error = nil;
        SAUser *user = [NSKeyedUnarchiver unarchivedObjectOfClass:[SAUser class] 
                                                         fromData:userData 
                                                           error:&error];
        if (user && !error) {
            self.currentUser = user;
            self.authenticated = YES;
            [SANetworkManager sharedManager].authToken = token;
            return YES;
        }
    }
    
    return NO;
}

- (void)loginWithEmail:(NSString *)email
              password:(NSString *)password
            completion:(void (^)(SAUser * _Nullable user, NSError * _Nullable error))completion {
    
    NSDictionary *params = @{
        @"email": email,
        @"password": password
    };
    
    [[SANetworkManager sharedManager] requestWithPath:@"/auth/login"
                                               method:SAHTTPMethodPOST
                                           parameters:params
                                           completion:^(NSDictionary *response, NSError *error) {
        if (error) {
            if (completion) completion(nil, error);
            return;
        }
        
        NSString *token = response[@"token"];
        NSDictionary *userData = response[@"user"];
        
        if (!token || !userData) {
            NSError *parseError = [NSError errorWithDomain:@"SAAuthError"
                                                      code:-1
                                                  userInfo:@{NSLocalizedDescriptionKey: @"Invalid response"}];
            if (completion) completion(nil, parseError);
            return;
        }
        
        SAUser *user = [SAUser userFromDictionary:userData];
        if (!user) {
            NSError *userError = [NSError errorWithDomain:@"SAAuthError"
                                                     code:-1
                                                 userInfo:@{NSLocalizedDescriptionKey: @"Failed to parse user"}];
            if (completion) completion(nil, userError);
            return;
        }
        
        // Save session
        [[NSUserDefaults standardUserDefaults] setObject:token forKey:kAuthTokenKey];
        NSData *encodedUser = [NSKeyedArchiver archivedDataWithRootObject:user 
                                                    requiringSecureCoding:NO 
                                                                    error:nil];
        if (encodedUser) {
            [[NSUserDefaults standardUserDefaults] setObject:encodedUser forKey:kCurrentUserKey];
        }
        [[NSUserDefaults standardUserDefaults] synchronize];
        
        // Update state
        self.currentUser = user;
        self.authenticated = YES;
        [SANetworkManager sharedManager].authToken = token;
        
        // Notify
        [[NSNotificationCenter defaultCenter] postNotificationName:SAAuthDidLoginNotification 
                                                            object:user];
        
        if (completion) completion(user, nil);
    }];
}

- (void)registerWithUsername:(NSString *)username
                       email:(NSString *)email
                    password:(NSString *)password
                  completion:(void (^)(SAUser * _Nullable user, NSError * _Nullable error))completion {
    
    NSDictionary *params = @{
        @"username": username,
        @"email": email,
        @"password": password
    };
    
    [[SANetworkManager sharedManager] requestWithPath:@"/auth/register"
                                               method:SAHTTPMethodPOST
                                           parameters:params
                                           completion:^(NSDictionary *response, NSError *error) {
        if (error) {
            if (completion) completion(nil, error);
            return;
        }
        
        // Auto-login after registration
        [self loginWithEmail:email password:password completion:completion];
    }];
}

- (void)logoutWithCompletion:(nullable void (^)(void))completion {
    // Clear local session
    [[NSUserDefaults standardUserDefaults] removeObjectForKey:kAuthTokenKey];
    [[NSUserDefaults standardUserDefaults] removeObjectForKey:kCurrentUserKey];
    [[NSUserDefaults standardUserDefaults] synchronize];
    
    [SANetworkManager sharedManager].authToken = nil;
    self.currentUser = nil;
    self.authenticated = NO;
    
    // Notify
    [[NSNotificationCenter defaultCenter] postNotificationName:SAAuthDidLogoutNotification 
                                                        object:nil];
    
    if (completion) completion();
}

@end
```

### Login ViewController

```objc
// SALoginViewController.h
#import <UIKit/UIKit.h>

@interface SALoginViewController : UIViewController
@end
```

```objc
// SALoginViewController.m
#import "SALoginViewController.h"
#import "SAAuthService.h"
#import "SARegisterViewController.h"

@interface SALoginViewController () <UITextFieldDelegate>

@property (nonatomic, strong) UIScrollView *scrollView;
@property (nonatomic, strong) UIView *containerView;
@property (nonatomic, strong) UIImageView *logoImageView;
@property (nonatomic, strong) UILabel *titleLabel;
@property (nonatomic, strong) UITextField *emailTextField;
@property (nonatomic, strong) UITextField *passwordTextField;
@property (nonatomic, strong) UIButton *loginButton;
@property (nonatomic, strong) UIButton *registerButton;
@property (nonatomic, strong) UIActivityIndicatorView *activityIndicator;
@property (nonatomic, strong) NSLayoutConstraint *containerBottomConstraint;

@end

@implementation SALoginViewController

- (void)viewDidLoad {
    [super viewDidLoad];
    [self setupUI];
    [self setupKeyboardNotifications];
}

- (void)setupUI {
    self.view.backgroundColor = [UIColor systemBackgroundColor];
    
    // Scroll View
    self.scrollView = [[UIScrollView alloc] init];
    self.scrollView.translatesAutoresizingMaskIntoConstraints = NO;
    [self.view addSubview:self.scrollView];
    
    // Container
    self.containerView = [[UIView alloc] init];
    self.containerView.translatesAutoresizingMaskIntoConstraints = NO;
    [self.scrollView addSubview:self.containerView];
    
    // Logo
    self.logoImageView = [[UIImageView alloc] initWithImage:[UIImage systemImageNamed:@"person.circle.fill"]];
    self.logoImageView.translatesAutoresizingMaskIntoConstraints = NO;
    self.logoImageView.tintColor = [UIColor systemBlueColor];
    self.logoImageView.contentMode = UIViewContentModeScaleAspectFit;
    [self.containerView addSubview:self.logoImageView];
    
    // Title
    self.titleLabel = [[UILabel alloc] init];
    self.titleLabel.translatesAutoresizingMaskIntoConstraints = NO;
    self.titleLabel.text = @"SocialApp";
    self.titleLabel.font = [UIFont boldSystemFontOfSize:32];
    self.titleLabel.textAlignment = NSTextAlignmentCenter;
    [self.containerView addSubview:self.titleLabel];
    
    // Email Field
    self.emailTextField = [self createTextFieldWithPlaceholder:@"อีเมล" 
                                                    keyboardType:UIKeyboardTypeEmailAddress];
    [self.containerView addSubview:self.emailTextField];
    
    // Password Field
    self.passwordTextField = [self createTextFieldWithPlaceholder:@"รหัสผ่าน" 
                                                       keyboardType:UIKeyboardTypeDefault];
    self.passwordTextField.secureTextEntry = YES;
    [self.containerView addSubview:self.passwordTextField];
    
    // Login Button
    self.loginButton = [UIButton buttonWithType:UIButtonTypeSystem];
    self.loginButton.translatesAutoresizingMaskIntoConstraints = NO;
    [self.loginButton setTitle:@"เข้าสู่ระบบ" forState:UIControlStateNormal];
    self.loginButton.titleLabel.font = [UIFont boldSystemFontOfSize:17];
    self.loginButton.backgroundColor = [UIColor systemBlueColor];
    [self.loginButton setTitleColor:[UIColor whiteColor] forState:UIControlStateNormal];
    self.loginButton.layer.cornerRadius = 12;
    [self.loginButton addTarget:self action:@selector(loginTapped) 
               forControlEvents:UIControlEventTouchUpInside];
    [self.containerView addSubview:self.loginButton];
    
    // Register Button
    self.registerButton = [UIButton buttonWithType:UIButtonTypeSystem];
    self.registerButton.translatesAutoresizingMaskIntoConstraints = NO;
    [self.registerButton setTitle:@"ยังไม่มีบัญชี? สมัครสมาชิก" forState:UIControlStateNormal];
    [self.registerButton addTarget:self action:@selector(registerTapped) 
                  forControlEvents:UIControlEventTouchUpInside];
    [self.containerView addSubview:self.registerButton];
    
    // Activity Indicator
    self.activityIndicator = [[UIActivityIndicatorView alloc] initWithActivityIndicatorStyle:UIActivityIndicatorViewStyleMedium];
    self.activityIndicator.translatesAutoresizingMaskIntoConstraints = NO;
    self.activityIndicator.hidesWhenStopped = YES;
    [self.containerView addSubview:self.activityIndicator];
    
    // Constraints
    [NSLayoutConstraint activateConstraints:@[
        // ScrollView
        [self.scrollView.topAnchor constraintEqualToAnchor:self.view.safeAreaLayoutGuide.topAnchor],
        [self.scrollView.leadingAnchor constraintEqualToAnchor:self.view.leadingAnchor],
        [self.scrollView.trailingAnchor constraintEqualToAnchor:self.view.trailingAnchor],
        [self.scrollView.bottomAnchor constraintEqualToAnchor:self.view.bottomAnchor],
        
        // Container
        [self.containerView.topAnchor constraintEqualToAnchor:self.scrollView.topAnchor],
        [self.containerView.leadingAnchor constraintEqualToAnchor:self.scrollView.leadingAnchor],
        [self.containerView.trailingAnchor constraintEqualToAnchor:self.scrollView.trailingAnchor],
        [self.containerView.bottomAnchor constraintEqualToAnchor:self.scrollView.bottomAnchor],
        [self.containerView.widthAnchor constraintEqualToAnchor:self.scrollView.widthAnchor],
        [self.containerView.heightAnchor constraintGreaterThanOrEqualToAnchor:self.view.heightAnchor],
        
        // Logo
        [self.logoImageView.topAnchor constraintEqualToAnchor:self.containerView.topAnchor constant:80],
        [self.logoImageView.centerXAnchor constraintEqualToAnchor:self.containerView.centerXAnchor],
        [self.logoImageView.widthAnchor constraintEqualToConstant:80],
        [self.logoImageView.heightAnchor constraintEqualToConstant:80],
        
        // Title
        [self.titleLabel.topAnchor constraintEqualToAnchor:self.logoImageView.bottomAnchor constant:16],
        [self.titleLabel.centerXAnchor constraintEqualToAnchor:self.containerView.centerXAnchor],
        
        // Email
        [self.emailTextField.topAnchor constraintEqualToAnchor:self.titleLabel.bottomAnchor constant:40],
        [self.emailTextField.leadingAnchor constraintEqualToAnchor:self.containerView.leadingAnchor constant:24],
        [self.emailTextField.trailingAnchor constraintEqualToAnchor:self.containerView.trailingAnchor constant:-24],
        [self.emailTextField.heightAnchor constraintEqualToConstant:52],
        
        // Password
        [self.passwordTextField.topAnchor constraintEqualToAnchor:self.emailTextField.bottomAnchor constant:16],
        [self.passwordTextField.leadingAnchor constraintEqualToAnchor:self.containerView.leadingAnchor constant:24],
        [self.passwordTextField.trailingAnchor constraintEqualToAnchor:self.containerView.trailingAnchor constant:-24],
        [self.passwordTextField.heightAnchor constraintEqualToConstant:52],
        
        // Login Button
        [self.loginButton.topAnchor constraintEqualToAnchor:self.passwordTextField.bottomAnchor constant:24],
        [self.loginButton.leadingAnchor constraintEqualToAnchor:self.containerView.leadingAnchor constant:24],
        [self.loginButton.trailingAnchor constraintEqualToAnchor:self.containerView.trailingAnchor constant:-24],
        [self.loginButton.heightAnchor constraintEqualToConstant:52],
        
        // Register Button
        [self.registerButton.topAnchor constraintEqualToAnchor:self.loginButton.bottomAnchor constant:16],
        [self.registerButton.centerXAnchor constraintEqualToAnchor:self.containerView.centerXAnchor],
        
        // Activity Indicator
        [self.activityIndicator.centerXAnchor constraintEqualToAnchor:self.loginButton.centerXAnchor],
        [self.activityIndicator.centerYAnchor constraintEqualToAnchor:self.loginButton.centerYAnchor]
    ]];
}

- (UITextField *)createTextFieldWithPlaceholder:(NSString *)placeholder 
                                    keyboardType:(UIKeyboardType)keyboardType {
    UITextField *textField = [[UITextField alloc] init];
    textField.translatesAutoresizingMaskIntoConstraints = NO;
    textField.placeholder = placeholder;
    textField.borderStyle = UITextBorderStyleRoundedRect;
    textField.keyboardType = keyboardType;
    textField.autocapitalizationType = UITextAutocapitalizationTypeNone;
    textField.autocorrectionType = UITextAutocorrectionTypeNo;
    textField.returnKeyType = UIReturnKeyNext;
    textField.delegate = self;
    textField.font = [UIFont systemFontOfSize:16];
    return textField;
}

- (void)setupKeyboardNotifications {
    [[NSNotificationCenter defaultCenter] addObserver:self 
                                             selector:@selector(keyboardWillShow:) 
                                                 name:UIKeyboardWillShowNotification 
                                               object:nil];
    [[NSNotificationCenter defaultCenter] addObserver:self 
                                             selector:@selector(keyboardWillHide:) 
                                                 name:UIKeyboardWillHideNotification 
                                               object:nil];
}

- (void)keyboardWillShow:(NSNotification *)notification {
    NSDictionary *info = notification.userInfo;
    CGRect keyboardFrame = [info[UIKeyboardFrameEndUserInfoKey] CGRectValue];
    NSTimeInterval duration = [info[UIKeyboardAnimationDurationUserInfoKey] doubleValue];
    
    [UIView animateWithDuration:duration animations:^{
        self.scrollView.contentInset = UIEdgeInsetsMake(0, 0, keyboardFrame.size.height, 0);
    }];
}

- (void)keyboardWillHide:(NSNotification *)notification {
    NSTimeInterval duration = [notification.userInfo[UIKeyboardAnimationDurationUserInfoKey] doubleValue];
    [UIView animateWithDuration:duration animations:^{
        self.scrollView.contentInset = UIEdgeInsetsZero;
    }];
}

- (void)loginTapped {
    NSString *email = self.emailTextField.text;
    NSString *password = self.passwordTextField.text;
    
    // Validation
    if (email.length == 0 || password.length == 0) {
        [self showAlert:@"กรุณากรอกข้อมูลให้ครบ" message:@"กรุณากรอกอีเมลและรหัสผ่าน"];
        return;
    }
    
    if (![self isValidEmail:email]) {
        [self showAlert:@"อีเมลไม่ถูกต้อง" message:@"กรุณากรอกอีเมลที่ถูกต้อง"];
        return;
    }
    
    if (password.length < 6) {
        [self showAlert:@"รหัสผ่านสั้นเกินไป" message:@"รหัสผ่านต้องมีอย่างน้อย 6 ตัวอักษร"];
        return;
    }
    
    // Loading state
    [self setLoading:YES];
    [self.view endEditing:YES];
    
    [[SAAuthService sharedService] loginWithEmail:email 
                                         password:password 
                                       completion:^(SAUser *user, NSError *error) {
        [self setLoading:NO];
        
        if (error) {
            [self showAlert:@"เข้าสู่ระบบล้มเหลว" message:error.localizedDescription];
            return;
        }
        
        // ไปหน้า main
        // ใช้ notification หรือ coordinator pattern
        [[NSNotificationCenter defaultCenter] postNotificationName:SAAuthDidLoginNotification 
                                                            object:user];
    }];
}

- (void)registerTapped {
    SARegisterViewController *vc = [[SARegisterViewController alloc] init];
    [self.navigationController pushViewController:vc animated:YES];
}

- (void)setLoading:(BOOL)loading {
    self.loginButton.enabled = !loading;
    [self.loginButton setTitle:loading ? @"" : @"เข้าสู่ระบบ" forState:UIControlStateNormal];
    
    if (loading) {
        [self.activityIndicator startAnimating];
    } else {
        [self.activityIndicator stopAnimating];
    }
}

- (BOOL)isValidEmail:(NSString *)email {
    NSString *pattern = @"[A-Z0-9a-z._%+-]+@[A-Za-z0-9.-]+\\.[A-Za-z]{2,}";
    NSPredicate *predicate = [NSPredicate predicateWithFormat:@"SELF MATCHES %@", pattern];
    return [predicate evaluateWithObject:email];
}

- (void)showAlert:(NSString *)title message:(NSString *)message {
    UIAlertController *alert = [UIAlertController alertControllerWithTitle:title
                                                                   message:message
                                                            preferredStyle:UIAlertControllerStyleAlert];
    [alert addAction:[UIAlertAction actionWithTitle:@"ตกลง" 
                                              style:UIAlertActionStyleDefault 
                                            handler:nil]];
    [self presentViewController:alert animated:YES completion:nil];
}

// UITextFieldDelegate
- (BOOL)textFieldShouldReturn:(UITextField *)textField {
    if (textField == self.emailTextField) {
        [self.passwordTextField becomeFirstResponder];
    } else {
        [textField resignFirstResponder];
        [self loginTapped];
    }
    return YES;
}

- (void)dealloc {
    [[NSNotificationCenter defaultCenter] removeObserver:self];
}

@end
```

---

## 91.5 Feed/Timeline with UITableView

### SAFeedViewController

```objc
// SAFeedViewController.h
#import <UIKit/UIKit.h>

@interface SAFeedViewController : UIViewController
@end
```

```objc
// SAFeedViewController.m
#import "SAFeedViewController.h"
#import "SAPostCell.h"
#import "SAPost.h"
#import "SAPostService.h"
#import "SAPostDetailViewController.h"
#import "SACreatePostViewController.h"

@interface SAFeedViewController () <UITableViewDataSource, UITableViewDelegate, SAPostCellDelegate>

@property (nonatomic, strong) UITableView *tableView;
@property (nonatomic, strong) UIRefreshControl *refreshControl;
@property (nonatomic, strong) NSMutableArray<SAPost *> *posts;
@property (nonatomic, strong) UIActivityIndicatorView *loadingIndicator;
@property (nonatomic, assign) BOOL isLoading;
@property (nonatomic, assign) BOOL hasMorePosts;
@property (nonatomic, strong) NSString *nextCursor;

@end

@implementation SAFeedViewController

- (void)viewDidLoad {
    [super viewDidLoad];
    
    self.title = @"Feed";
    self.posts = [NSMutableArray array];
    self.hasMorePosts = YES;
    
    [self setupNavigationBar];
    [self setupTableView];
    [self loadInitialFeed];
}

- (void)setupNavigationBar {
    UIBarButtonItem *createButton = [[UIBarButtonItem alloc] 
                                     initWithImage:[UIImage systemImageNamed:@"plus.circle.fill"]
                                     style:UIBarButtonItemStylePlain
                                     target:self 
                                     action:@selector(createPostTapped)];
    self.navigationItem.rightBarButtonItem = createButton;
    
    UIBarButtonItem *logoButton = [[UIBarButtonItem alloc]
                                   initWithTitle:@"SocialApp"
                                   style:UIBarButtonItemStylePlain
                                   target:nil
                                   action:nil];
    logoButton.tintColor = [UIColor labelColor];
    UIFont *boldFont = [UIFont boldSystemFontOfSize:20];
    [logoButton setTitleTextAttributes:@{NSFontAttributeName: boldFont} forState:UIControlStateNormal];
    self.navigationItem.leftBarButtonItem = logoButton;
}

- (void)setupTableView {
    self.tableView = [[UITableView alloc] initWithFrame:CGRectZero style:UITableViewStylePlain];
    self.tableView.translatesAutoresizingMaskIntoConstraints = NO;
    self.tableView.dataSource = self;
    self.tableView.delegate = self;
    self.tableView.separatorStyle = UITableViewCellSeparatorStyleNone;
    self.tableView.estimatedRowHeight = 300;
    self.tableView.rowHeight = UITableViewAutomaticDimension;
    [self.tableView registerClass:[SAPostCell class] forCellReuseIdentifier:@"PostCell"];
    [self.view addSubview:self.tableView];
    
    // Refresh Control
    self.refreshControl = [[UIRefreshControl alloc] init];
    [self.refreshControl addTarget:self action:@selector(refreshFeed) 
                  forControlEvents:UIControlEventValueChanged];
    self.tableView.refreshControl = self.refreshControl;
    
    // Loading footer
    self.loadingIndicator = [[UIActivityIndicatorView alloc] initWithActivityIndicatorStyle:UIActivityIndicatorViewStyleMedium];
    self.loadingIndicator.frame = CGRectMake(0, 0, self.view.bounds.size.width, 50);
    self.tableView.tableFooterView = self.loadingIndicator;
    
    // Constraints
    [NSLayoutConstraint activateConstraints:@[
        [self.tableView.topAnchor constraintEqualToAnchor:self.view.safeAreaLayoutGuide.topAnchor],
        [self.tableView.leadingAnchor constraintEqualToAnchor:self.view.leadingAnchor],
        [self.tableView.trailingAnchor constraintEqualToAnchor:self.view.trailingAnchor],
        [self.tableView.bottomAnchor constraintEqualToAnchor:self.view.bottomAnchor]
    ]];
}

- (void)loadInitialFeed {
    self.isLoading = YES;
    [self.loadingIndicator startAnimating];
    
    [[SAPostService sharedService] getFeedWithCursor:nil 
                                              limit:20
                                         completion:^(NSArray<SAPost *> *posts, 
                                                     NSString *nextCursor, 
                                                     NSError *error) {
        self.isLoading = NO;
        [self.loadingIndicator stopAnimating];
        [self.refreshControl endRefreshing];
        
        if (error) {
            [self showError:error];
            return;
        }
        
        [self.posts removeAllObjects];
        [self.posts addObjectsFromArray:posts];
        self.nextCursor = nextCursor;
        self.hasMorePosts = nextCursor != nil;
        
        [self.tableView reloadData];
    }];
}

- (void)refreshFeed {
    self.nextCursor = nil;
    self.hasMorePosts = YES;
    [self loadInitialFeed];
}

- (void)loadMorePosts {
    if (self.isLoading || !self.hasMorePosts || !self.nextCursor) return;
    
    self.isLoading = YES;
    [self.loadingIndicator startAnimating];
    
    [[SAPostService sharedService] getFeedWithCursor:self.nextCursor
                                              limit:20
                                         completion:^(NSArray<SAPost *> *posts, 
                                                     NSString *nextCursor, 
                                                     NSError *error) {
        self.isLoading = NO;
        [self.loadingIndicator stopAnimating];
        
        if (error) {
            [self showError:error];
            return;
        }
        
        NSInteger startIndex = self.posts.count;
        [self.posts addObjectsFromArray:posts];
        self.nextCursor = nextCursor;
        self.hasMorePosts = nextCursor != nil;
        
        // Animate new rows
        NSMutableArray<NSIndexPath *> *indexPaths = [NSMutableArray array];
        for (NSInteger i = startIndex; i < self.posts.count; i++) {
            [indexPaths addObject:[NSIndexPath indexPathForRow:i inSection:0]];
        }
        
        [self.tableView insertRowsAtIndexPaths:indexPaths 
                              withRowAnimation:UITableViewRowAnimationAutomatic];
    }];
}

// MARK: - UITableViewDataSource

- (NSInteger)tableView:(UITableView *)tableView numberOfRowsInSection:(NSInteger)section {
    return self.posts.count;
}

- (UITableViewCell *)tableView:(UITableView *)tableView cellForRowAtIndexPath:(NSIndexPath *)indexPath {
    SAPostCell *cell = [tableView dequeueReusableCellWithIdentifier:@"PostCell" 
                                                       forIndexPath:indexPath];
    cell.delegate = self;
    [cell configureWithPost:self.posts[indexPath.row]];
    return cell;
}

// MARK: - UITableViewDelegate

- (void)tableView:(UITableView *)tableView didSelectRowAtIndexPath:(NSIndexPath *)indexPath {
    [tableView deselectRowAtIndexPath:indexPath animated:YES];
    
    SAPost *post = self.posts[indexPath.row];
    SAPostDetailViewController *detailVC = [[SAPostDetailViewController alloc] initWithPost:post];
    [self.navigationController pushViewController:detailVC animated:YES];
}

- (void)scrollViewDidScroll:(UIScrollView *)scrollView {
    CGFloat offsetY = scrollView.contentOffset.y;
    CGFloat contentHeight = scrollView.contentSize.height;
    CGFloat scrollViewHeight = scrollView.frame.size.height;
    
    // Load more when near bottom
    if (offsetY > contentHeight - scrollViewHeight - 200) {
        [self loadMorePosts];
    }
}

// MARK: - SAPostCellDelegate

- (void)postCell:(SAPostCell *)cell didTapLikeForPost:(SAPost *)post {
    BOOL wasLiked = post.liked;
    
    // Optimistic update
    post.liked = !wasLiked;
    post.likesCount += wasLiked ? -1 : 1;
    [self.tableView reloadRowsAtIndexPaths:@[[self indexPathForPost:post]]
                          withRowAnimation:UITableViewRowAnimationNone];
    
    [[SAPostService sharedService] toggleLikeForPost:post.postID
                                          completion:^(BOOL liked, NSError *error) {
        if (error) {
            // Revert optimistic update
            post.liked = wasLiked;
            post.likesCount += wasLiked ? 1 : -1;
            [self.tableView reloadRowsAtIndexPaths:@[[self indexPathForPost:post]]
                                  withRowAnimation:UITableViewRowAnimationNone];
        }
    }];
}

- (void)postCell:(SAPostCell *)cell didTapCommentForPost:(SAPost *)post {
    SAPostDetailViewController *detailVC = [[SAPostDetailViewController alloc] initWithPost:post];
    detailVC.shouldFocusComment = YES;
    [self.navigationController pushViewController:detailVC animated:YES];
}

- (NSIndexPath *)indexPathForPost:(SAPost *)post {
    NSInteger index = [self.posts indexOfObjectPassingTest:^BOOL(SAPost *obj, NSUInteger idx, BOOL *stop) {
        return [obj.postID isEqualToString:post.postID];
    }];
    
    if (index == NSNotFound) return [NSIndexPath indexPathForRow:0 inSection:0];
    return [NSIndexPath indexPathForRow:index inSection:0];
}

- (void)createPostTapped {
    SACreatePostViewController *createVC = [[SACreatePostViewController alloc] init];
    __weak typeof(self) weakSelf = self;
    createVC.onPostCreated = ^(SAPost *newPost) {
        [weakSelf.posts insertObject:newPost atIndex:0];
        [weakSelf.tableView insertRowsAtIndexPaths:@[[NSIndexPath indexPathForRow:0 inSection:0]]
                                  withRowAnimation:UITableViewRowAnimationTop];
    };
    UINavigationController *nav = [[UINavigationController alloc] initWithRootViewController:createVC];
    [self presentViewController:nav animated:YES completion:nil];
}

- (void)showError:(NSError *)error {
    UIAlertController *alert = [UIAlertController alertControllerWithTitle:@"เกิดข้อผิดพลาด"
                                                                   message:error.localizedDescription
                                                            preferredStyle:UIAlertControllerStyleAlert];
    [alert addAction:[UIAlertAction actionWithTitle:@"ตกลง" style:UIAlertActionStyleDefault handler:nil]];
    [self presentViewController:alert animated:YES completion:nil];
}

@end
```

---

## 91.6 Post Cell

```objc
// SAPostCell.h
#import <UIKit/UIKit.h>
@class SAPost, SAPostCell;

@protocol SAPostCellDelegate <NSObject>
- (void)postCell:(SAPostCell *)cell didTapLikeForPost:(SAPost *)post;
- (void)postCell:(SAPostCell *)cell didTapCommentForPost:(SAPost *)post;
@optional
- (void)postCell:(SAPostCell *)cell didTapShareForPost:(SAPost *)post;
- (void)postCell:(SAPostCell *)cell didTapAuthorForPost:(SAPost *)post;
@end

@interface SAPostCell : UITableViewCell

@property (nonatomic, weak) id<SAPostCellDelegate> delegate;

- (void)configureWithPost:(SAPost *)post;

@end
```

```objc
// SAPostCell.m
#import "SAPostCell.h"
#import "SAPost.h"
#import "SAUser.h"

@interface SAPostCell ()

@property (nonatomic, strong) UIView *containerView;
@property (nonatomic, strong) UIButton *authorButton;
@property (nonatomic, strong) UIImageView *avatarImageView;
@property (nonatomic, strong) UILabel *authorNameLabel;
@property (nonatomic, strong) UILabel *timeLabel;
@property (nonatomic, strong) UILabel *contentLabel;
@property (nonatomic, strong) UIImageView *postImageView;
@property (nonatomic, strong) UIStackView *actionStack;
@property (nonatomic, strong) UIButton *likeButton;
@property (nonatomic, strong) UIButton *commentButton;
@property (nonatomic, strong) UIButton *shareButton;
@property (nonatomic, strong) UILabel *likesLabel;
@property (nonatomic, strong) UILabel *commentsLabel;
@property (nonatomic, strong) NSLayoutConstraint *imageHeightConstraint;
@property (nonatomic, strong) SAPost *currentPost;

@end

@implementation SAPostCell

- (instancetype)initWithStyle:(UITableViewCellStyle)style reuseIdentifier:(NSString *)reuseIdentifier {
    if (self = [super initWithStyle:style reuseIdentifier:reuseIdentifier]) {
        [self setupUI];
    }
    return self;
}

- (void)setupUI {
    self.selectionStyle = UITableViewCellSelectionStyleNone;
    self.backgroundColor = [UIColor systemBackgroundColor];
    
    // Container
    self.containerView = [[UIView alloc] init];
    self.containerView.translatesAutoresizingMaskIntoConstraints = NO;
    self.containerView.backgroundColor = [UIColor secondarySystemBackgroundColor];
    self.containerView.layer.cornerRadius = 12;
    self.containerView.clipsToBounds = YES;
    [self.contentView addSubview:self.containerView];
    
    // Avatar
    self.avatarImageView = [[UIImageView alloc] init];
    self.avatarImageView.translatesAutoresizingMaskIntoConstraints = NO;
    self.avatarImageView.backgroundColor = [UIColor systemGray4Color];
    self.avatarImageView.layer.cornerRadius = 20;
    self.avatarImageView.clipsToBounds = YES;
    self.avatarImageView.contentMode = UIViewContentModeScaleAspectFill;
    [self.containerView addSubview:self.avatarImageView];
    
    // Author Name
    self.authorNameLabel = [[UILabel alloc] init];
    self.authorNameLabel.translatesAutoresizingMaskIntoConstraints = NO;
    self.authorNameLabel.font = [UIFont boldSystemFontOfSize:15];
    [self.containerView addSubview:self.authorNameLabel];
    
    // Time Label
    self.timeLabel = [[UILabel alloc] init];
    self.timeLabel.translatesAutoresizingMaskIntoConstraints = NO;
    self.timeLabel.font = [UIFont systemFontOfSize:12];
    self.timeLabel.textColor = [UIColor systemGrayColor];
    [self.containerView addSubview:self.timeLabel];
    
    // Content Label
    self.contentLabel = [[UILabel alloc] init];
    self.contentLabel.translatesAutoresizingMaskIntoConstraints = NO;
    self.contentLabel.font = [UIFont systemFontOfSize:15];
    self.contentLabel.numberOfLines = 0;
    [self.containerView addSubview:self.contentLabel];
    
    // Post Image
    self.postImageView = [[UIImageView alloc] init];
    self.postImageView.translatesAutoresizingMaskIntoConstraints = NO;
    self.postImageView.contentMode = UIViewContentModeScaleAspectFill;
    self.postImageView.clipsToBounds = YES;
    self.postImageView.backgroundColor = [UIColor systemGray5Color];
    [self.containerView addSubview:self.postImageView];
    
    // Action Stack
    self.likeButton = [self createActionButtonWithSystemName:@"heart" title:@"0"];
    self.commentButton = [self createActionButtonWithSystemName:@"bubble.left" title:@"0"];
    self.shareButton = [self createActionButtonWithSystemName:@"square.and.arrow.up" title:@"แชร์"];
    
    [self.likeButton addTarget:self action:@selector(likeTapped) forControlEvents:UIControlEventTouchUpInside];
    [self.commentButton addTarget:self action:@selector(commentTapped) forControlEvents:UIControlEventTouchUpInside];
    [self.shareButton addTarget:self action:@selector(shareTapped) forControlEvents:UIControlEventTouchUpInside];
    
    self.actionStack = [[UIStackView alloc] initWithArrangedSubviews:@[
        self.likeButton, self.commentButton, self.shareButton
    ]];
    self.actionStack.translatesAutoresizingMaskIntoConstraints = NO;
    self.actionStack.axis = UILayoutConstraintAxisHorizontal;
    self.actionStack.distribution = UIStackViewDistributionFillEqually;
    [self.containerView addSubview:self.actionStack];
    
    // Image height constraint (ซ่อนได้)
    self.imageHeightConstraint = [self.postImageView.heightAnchor constraintEqualToConstant:0];
    
    // Constraints
    [NSLayoutConstraint activateConstraints:@[
        // Container
        [self.containerView.topAnchor constraintEqualToAnchor:self.contentView.topAnchor constant:8],
        [self.containerView.leadingAnchor constraintEqualToAnchor:self.contentView.leadingAnchor constant:16],
        [self.containerView.trailingAnchor constraintEqualToAnchor:self.contentView.trailingAnchor constant:-16],
        [self.containerView.bottomAnchor constraintEqualToAnchor:self.contentView.bottomAnchor constant:-8],
        
        // Avatar
        [self.avatarImageView.topAnchor constraintEqualToAnchor:self.containerView.topAnchor constant:12],
        [self.avatarImageView.leadingAnchor constraintEqualToAnchor:self.containerView.leadingAnchor constant:12],
        [self.avatarImageView.widthAnchor constraintEqualToConstant:40],
        [self.avatarImageView.heightAnchor constraintEqualToConstant:40],
        
        // Author Name
        [self.authorNameLabel.topAnchor constraintEqualToAnchor:self.avatarImageView.topAnchor],
        [self.authorNameLabel.leadingAnchor constraintEqualToAnchor:self.avatarImageView.trailingAnchor constant:8],
        [self.authorNameLabel.trailingAnchor constraintEqualToAnchor:self.containerView.trailingAnchor constant:-12],
        
        // Time
        [self.timeLabel.topAnchor constraintEqualToAnchor:self.authorNameLabel.bottomAnchor constant:2],
        [self.timeLabel.leadingAnchor constraintEqualToAnchor:self.authorNameLabel.leadingAnchor],
        
        // Content
        [self.contentLabel.topAnchor constraintEqualToAnchor:self.avatarImageView.bottomAnchor constant:8],
        [self.contentLabel.leadingAnchor constraintEqualToAnchor:self.containerView.leadingAnchor constant:12],
        [self.contentLabel.trailingAnchor constraintEqualToAnchor:self.containerView.trailingAnchor constant:-12],
        
        // Post Image
        [self.postImageView.topAnchor constraintEqualToAnchor:self.contentLabel.bottomAnchor constant:8],
        [self.postImageView.leadingAnchor constraintEqualToAnchor:self.containerView.leadingAnchor],
        [self.postImageView.trailingAnchor constraintEqualToAnchor:self.containerView.trailingAnchor],
        self.imageHeightConstraint,
        
        // Action Stack
        [self.actionStack.topAnchor constraintEqualToAnchor:self.postImageView.bottomAnchor constant:8],
        [self.actionStack.leadingAnchor constraintEqualToAnchor:self.containerView.leadingAnchor],
        [self.actionStack.trailingAnchor constraintEqualToAnchor:self.containerView.trailingAnchor],
        [self.actionStack.bottomAnchor constraintEqualToAnchor:self.containerView.bottomAnchor constant:-8],
        [self.actionStack.heightAnchor constraintEqualToConstant:44]
    ]];
}

- (UIButton *)createActionButtonWithSystemName:(NSString *)systemName title:(NSString *)title {
    UIButton *button = [UIButton buttonWithType:UIButtonTypeSystem];
    button.translatesAutoresizingMaskIntoConstraints = NO;
    
    UIImage *image = [UIImage systemImageNamed:systemName];
    UIImageSymbolConfiguration *config = [UIImageSymbolConfiguration configurationWithScale:UIImageSymbolScaleMedium];
    image = [image imageByApplyingSymbolConfiguration:config];
    
    [button setImage:image forState:UIControlStateNormal];
    [button setTitle:[NSString stringWithFormat:@" %@", title] forState:UIControlStateNormal];
    button.tintColor = [UIColor systemGrayColor];
    
    return button;
}

- (void)configureWithPost:(SAPost *)post {
    self.currentPost = post;
    
    // Author info
    self.authorNameLabel.text = post.author.displayName ?: post.author.username;
    self.timeLabel.text = [post timeAgoString];
    
    // Content
    self.contentLabel.text = post.content;
    
    // Like button state
    UIImage *heartImage = [UIImage systemImageNamed:post.liked ? @"heart.fill" : @"heart"];
    UIImageSymbolConfiguration *config = [UIImageSymbolConfiguration configurationWithScale:UIImageSymbolScaleMedium];
    heartImage = [heartImage imageByApplyingSymbolConfiguration:config];
    [self.likeButton setImage:heartImage forState:UIControlStateNormal];
    self.likeButton.tintColor = post.liked ? [UIColor systemRedColor] : [UIColor systemGrayColor];
    [self.likeButton setTitle:[NSString stringWithFormat:@" %ld", (long)post.likesCount] 
                     forState:UIControlStateNormal];
    
    // Comment count
    [self.commentButton setTitle:[NSString stringWithFormat:@" %ld", (long)post.commentsCount] 
                        forState:UIControlStateNormal];
    
    // Image
    if (post.mediaType == SAPostMediaTypeImage && post.imageURLs.count > 0) {
        self.imageHeightConstraint.constant = 250;
        // Load image ด้วย URL loading (simplified)
        [self loadImageFromURL:post.imageURLs.firstObject];
    } else {
        self.imageHeightConstraint.constant = 0;
        self.postImageView.image = nil;
    }
    
    // Avatar
    if (post.author.avatarURL) {
        [self loadAvatarFromURL:post.author.avatarURL];
    } else {
        self.avatarImageView.image = [UIImage systemImageNamed:@"person.circle.fill"];
        self.avatarImageView.tintColor = [UIColor systemGray3Color];
    }
}

- (void)loadImageFromURL:(NSString *)urlString {
    NSURL *url = [NSURL URLWithString:urlString];
    if (!url) return;
    
    self.postImageView.image = nil;
    
    NSURLSessionDataTask *task = [[NSURLSession sharedSession] dataTaskWithURL:url 
                                                             completionHandler:^(NSData *data, 
                                                                               NSURLResponse *response, 
                                                                               NSError *error) {
        if (!error && data) {
            UIImage *image = [UIImage imageWithData:data];
            dispatch_async(dispatch_get_main_queue(), ^{
                self.postImageView.image = image;
            });
        }
    }];
    [task resume];
}

- (void)loadAvatarFromURL:(NSString *)urlString {
    NSURL *url = [NSURL URLWithString:urlString];
    if (!url) return;
    
    NSURLSessionDataTask *task = [[NSURLSession sharedSession] dataTaskWithURL:url 
                                                             completionHandler:^(NSData *data, 
                                                                               NSURLResponse *response, 
                                                                               NSError *error) {
        if (!error && data) {
            UIImage *image = [UIImage imageWithData:data];
            dispatch_async(dispatch_get_main_queue(), ^{
                self.avatarImageView.image = image;
            });
        }
    }];
    [task resume];
}

- (void)likeTapped {
    if ([self.delegate respondsToSelector:@selector(postCell:didTapLikeForPost:)]) {
        [self.delegate postCell:self didTapLikeForPost:self.currentPost];
    }
}

- (void)commentTapped {
    if ([self.delegate respondsToSelector:@selector(postCell:didTapCommentForPost:)]) {
        [self.delegate postCell:self didTapCommentForPost:self.currentPost];
    }
}

- (void)shareTapped {
    if ([self.delegate respondsToSelector:@selector(postCell:didTapShareForPost:)]) {
        [self.delegate postCell:self didTapShareForPost:self.currentPost];
    }
}

- (void)prepareForReuse {
    [super prepareForReuse];
    self.avatarImageView.image = nil;
    self.postImageView.image = nil;
    self.authorNameLabel.text = nil;
    self.timeLabel.text = nil;
    self.contentLabel.text = nil;
    self.imageHeightConstraint.constant = 0;
}

@end
```

---

## 91.7 User Profile

```objc
// SAProfileViewController.m
#import "SAProfileViewController.h"
#import "SAPostCell.h"
#import "SAUserService.h"
#import "SAPostService.h"
#import "SAAuthService.h"

@interface SAProfileViewController () <UITableViewDataSource, UITableViewDelegate, SAPostCellDelegate>

@property (nonatomic, strong) UITableView *tableView;
@property (nonatomic, strong) SAUser *user;
@property (nonatomic, strong) NSMutableArray<SAPost *> *posts;
@property (nonatomic, assign) BOOL isCurrentUser;
@property (nonatomic, assign) BOOL isLoading;
@property (nonatomic, strong) NSString *userID;

@end

@implementation SAProfileViewController

- (instancetype)initWithUserID:(NSString *)userID {
    if (self = [super init]) {
        _userID = userID;
    }
    return self;
}

- (void)viewDidLoad {
    [super viewDidLoad];
    self.posts = [NSMutableArray array];
    
    [self setupTableView];
    [self loadUserProfile];
}

- (void)setupTableView {
    self.tableView = [[UITableView alloc] initWithFrame:self.view.bounds style:UITableViewStylePlain];
    self.tableView.translatesAutoresizingMaskIntoConstraints = NO;
    self.tableView.dataSource = self;
    self.tableView.delegate = self;
    self.tableView.separatorStyle = UITableViewCellSeparatorStyleNone;
    self.tableView.estimatedRowHeight = 300;
    self.tableView.rowHeight = UITableViewAutomaticDimension;
    [self.tableView registerClass:[SAPostCell class] forCellReuseIdentifier:@"PostCell"];
    [self.view addSubview:self.tableView];
    
    [NSLayoutConstraint activateConstraints:@[
        [self.tableView.topAnchor constraintEqualToAnchor:self.view.safeAreaLayoutGuide.topAnchor],
        [self.tableView.leadingAnchor constraintEqualToAnchor:self.view.leadingAnchor],
        [self.tableView.trailingAnchor constraintEqualToAnchor:self.view.trailingAnchor],
        [self.tableView.bottomAnchor constraintEqualToAnchor:self.view.bottomAnchor]
    ]];
}

- (void)loadUserProfile {
    [[SAUserService sharedService] getUserWithID:self.userID 
                                      completion:^(SAUser *user, NSError *error) {
        if (error || !user) return;
        
        self.user = user;
        self.isCurrentUser = [user.userID isEqualToString:[SAAuthService sharedService].currentUser.userID];
        self.title = user.username;
        
        // Setup header
        [self setupProfileHeader];
        
        // Load posts
        [self loadUserPosts];
    }];
}

- (void)setupProfileHeader {
    SAProfileHeaderView *header = [[SAProfileHeaderView alloc] initWithFrame:CGRectMake(0, 0, self.view.bounds.size.width, 280)];
    [header configureWithUser:self.user isCurrentUser:self.isCurrentUser];
    
    __weak typeof(self) weakSelf = self;
    header.onFollowTapped = ^{
        [weakSelf toggleFollow];
    };
    header.onEditTapped = ^{
        [weakSelf editProfile];
    };
    
    self.tableView.tableHeaderView = header;
}

- (void)loadUserPosts {
    [[SAPostService sharedService] getPostsForUserID:self.userID
                                          completion:^(NSArray<SAPost *> *posts, NSError *error) {
        if (error) return;
        
        [self.posts removeAllObjects];
        [self.posts addObjectsFromArray:posts];
        [self.tableView reloadData];
    }];
}

- (void)toggleFollow {
    if (!self.user) return;
    
    BOOL wasFollowing = self.user.following;
    self.user.following = !wasFollowing;
    self.user.followersCount += wasFollowing ? -1 : 1;
    
    // Update header
    [self setupProfileHeader];
    
    [[SAUserService sharedService] toggleFollowForUserID:self.userID
                                              completion:^(BOOL following, NSError *error) {
        if (error) {
            // Revert
            self.user.following = wasFollowing;
            self.user.followersCount += wasFollowing ? 1 : -1;
            [self setupProfileHeader];
        }
    }];
}

- (void)editProfile {
    // Present edit profile view controller
    // SAEditProfileViewController *editVC = [[SAEditProfileViewController alloc] init];
    // [self presentViewController:editVC animated:YES completion:nil];
}

// UITableViewDataSource
- (NSInteger)tableView:(UITableView *)tableView numberOfRowsInSection:(NSInteger)section {
    return self.posts.count;
}

- (UITableViewCell *)tableView:(UITableView *)tableView cellForRowAtIndexPath:(NSIndexPath *)indexPath {
    SAPostCell *cell = [tableView dequeueReusableCellWithIdentifier:@"PostCell" forIndexPath:indexPath];
    cell.delegate = self;
    [cell configureWithPost:self.posts[indexPath.row]];
    return cell;
}

// SAPostCellDelegate
- (void)postCell:(SAPostCell *)cell didTapLikeForPost:(SAPost *)post {
    BOOL wasLiked = post.liked;
    post.liked = !wasLiked;
    post.likesCount += wasLiked ? -1 : 1;
    
    NSInteger index = [self.posts indexOfObject:post];
    if (index != NSNotFound) {
        [self.tableView reloadRowsAtIndexPaths:@[[NSIndexPath indexPathForRow:index inSection:0]]
                              withRowAnimation:UITableViewRowAnimationNone];
    }
    
    [[SAPostService sharedService] toggleLikeForPost:post.postID 
                                          completion:^(BOOL liked, NSError *error) {
        if (error) {
            post.liked = wasLiked;
            post.likesCount += wasLiked ? 1 : -1;
            NSInteger idx = [self.posts indexOfObject:post];
            if (idx != NSNotFound) {
                [self.tableView reloadRowsAtIndexPaths:@[[NSIndexPath indexPathForRow:idx inSection:0]]
                                      withRowAnimation:UITableViewRowAnimationNone];
            }
        }
    }];
}

- (void)postCell:(SAPostCell *)cell didTapCommentForPost:(SAPost *)post {
    // Navigate to post detail
}

@end
```

---

## 91.8 Search Users

```objc
// SASearchViewController.m
#import "SASearchViewController.h"
#import "SAUserCell.h"
#import "SAUserService.h"
#import "SAProfileViewController.h"

@interface SASearchViewController () <UISearchResultsUpdating, UITableViewDataSource, UITableViewDelegate>

@property (nonatomic, strong) UISearchController *searchController;
@property (nonatomic, strong) UITableView *tableView;
@property (nonatomic, strong) NSMutableArray<SAUser *> *searchResults;
@property (nonatomic, strong) NSTimer *searchTimer;

@end

@implementation SASearchViewController

- (void)viewDidLoad {
    [super viewDidLoad];
    self.title = @"ค้นหา";
    self.searchResults = [NSMutableArray array];
    
    [self setupSearchController];
    [self setupTableView];
}

- (void)setupSearchController {
    self.searchController = [[UISearchController alloc] initWithSearchResultsController:nil];
    self.searchController.searchResultsUpdater = self;
    self.searchController.obscuresBackgroundDuringPresentation = NO;
    self.searchController.searchBar.placeholder = @"ค้นหาผู้ใช้...";
    
    self.navigationItem.searchController = self.searchController;
    self.navigationItem.hidesSearchBarWhenScrolling = NO;
    self.definesPresentationContext = YES;
}

- (void)setupTableView {
    self.tableView = [[UITableView alloc] initWithFrame:self.view.bounds style:UITableViewStylePlain];
    self.tableView.translatesAutoresizingMaskIntoConstraints = NO;
    self.tableView.dataSource = self;
    self.tableView.delegate = self;
    [self.tableView registerClass:[SAUserCell class] forCellReuseIdentifier:@"UserCell"];
    [self.view addSubview:self.tableView];
    
    [NSLayoutConstraint activateConstraints:@[
        [self.tableView.topAnchor constraintEqualToAnchor:self.view.safeAreaLayoutGuide.topAnchor],
        [self.tableView.leadingAnchor constraintEqualToAnchor:self.view.leadingAnchor],
        [self.tableView.trailingAnchor constraintEqualToAnchor:self.view.trailingAnchor],
        [self.tableView.bottomAnchor constraintEqualToAnchor:self.view.bottomAnchor]
    ]];
}

// UISearchResultsUpdating
- (void)updateSearchResultsForSearchController:(UISearchController *)searchController {
    NSString *query = searchController.searchBar.text;
    
    // Debounce - รอ 0.5 วินาทีก่อน search
    [self.searchTimer invalidate];
    
    if (query.length < 2) {
        [self.searchResults removeAllObjects];
        [self.tableView reloadData];
        return;
    }
    
    self.searchTimer = [NSTimer scheduledTimerWithTimeInterval:0.5 
                                                       target:self 
                                                     selector:@selector(performSearch:) 
                                                     userInfo:query 
                                                      repeats:NO];
}

- (void)performSearch:(NSTimer *)timer {
    NSString *query = timer.userInfo;
    
    [[SAUserService sharedService] searchUsersWithQuery:query 
                                             completion:^(NSArray<SAUser *> *users, NSError *error) {
        if (error) return;
        
        [self.searchResults removeAllObjects];
        [self.searchResults addObjectsFromArray:users];
        [self.tableView reloadData];
    }];
}

// UITableViewDataSource
- (NSInteger)tableView:(UITableView *)tableView numberOfRowsInSection:(NSInteger)section {
    return self.searchResults.count;
}

- (UITableViewCell *)tableView:(UITableView *)tableView cellForRowAtIndexPath:(NSIndexPath *)indexPath {
    SAUserCell *cell = [tableView dequeueReusableCellWithIdentifier:@"UserCell" forIndexPath:indexPath];
    [cell configureWithUser:self.searchResults[indexPath.row]];
    return cell;
}

// UITableViewDelegate
- (void)tableView:(UITableView *)tableView didSelectRowAtIndexPath:(NSIndexPath *)indexPath {
    [tableView deselectRowAtIndexPath:indexPath animated:YES];
    
    SAUser *user = self.searchResults[indexPath.row];
    SAProfileViewController *profileVC = [[SAProfileViewController alloc] initWithUserID:user.userID];
    [self.navigationController pushViewController:profileVC animated:YES];
}

@end
```

---

## 91.9 Local Caching Strategy

### SACacheManager

```objc
// SACacheManager.h
#import <Foundation/Foundation.h>
#import "SAPost.h"
#import "SAUser.h"

NS_ASSUME_NONNULL_BEGIN

@interface SACacheManager : NSObject

+ (instancetype)sharedManager;

// User caching
- (void)cacheUser:(SAUser *)user;
- (nullable SAUser *)cachedUserForID:(NSString *)userID;
- (void)invalidateUserCache:(NSString *)userID;

// Post caching
- (void)cachePosts:(NSArray<SAPost *> *)posts forKey:(NSString *)key;
- (nullable NSArray<SAPost *> *)cachedPostsForKey:(NSString *)key;
- (void)invalidatePostsCacheForKey:(NSString *)key;

// Image caching
- (void)cacheImageData:(NSData *)data forURL:(NSString *)urlString;
- (nullable NSData *)cachedImageDataForURL:(NSString *)urlString;

// Clear all
- (void)clearAllCache;

@end

NS_ASSUME_NONNULL_END
```

```objc
// SACacheManager.m
#import "SACacheManager.h"

static const NSInteger kMaxMemoryCacheCount = 100;
static const NSTimeInterval kCacheExpiryTime = 300; // 5 minutes

@interface SACacheManager ()
@property (nonatomic, strong) NSCache<NSString *, SAUser *> *userCache;
@property (nonatomic, strong) NSCache<NSString *, NSArray *> *postsCache;
@property (nonatomic, strong) NSCache<NSString *, NSData *> *imageCache;
@property (nonatomic, strong) NSMutableDictionary<NSString *, NSDate *> *cacheTimestamps;
@property (nonatomic, strong) dispatch_queue_t cacheQueue;
@end

@implementation SACacheManager

+ (instancetype)sharedManager {
    static SACacheManager *instance = nil;
    static dispatch_once_t onceToken;
    dispatch_once(&onceToken, ^{
        instance = [[SACacheManager alloc] init];
    });
    return instance;
}

- (instancetype)init {
    if (self = [super init]) {
        _userCache = [[NSCache alloc] init];
        _userCache.countLimit = kMaxMemoryCacheCount;
        
        _postsCache = [[NSCache alloc] init];
        _postsCache.countLimit = 50;
        
        _imageCache = [[NSCache alloc] init];
        _imageCache.totalCostLimit = 50 * 1024 * 1024; // 50MB
        
        _cacheTimestamps = [NSMutableDictionary dictionary];
        _cacheQueue = dispatch_queue_create("com.socialapp.cache", DISPATCH_QUEUE_CONCURRENT);
    }
    return self;
}

- (void)cacheUser:(SAUser *)user {
    dispatch_barrier_async(self.cacheQueue, ^{
        [self.userCache setObject:user forKey:user.userID];
        self.cacheTimestamps[user.userID] = [NSDate date];
    });
}

- (nullable SAUser *)cachedUserForID:(NSString *)userID {
    __block SAUser *user = nil;
    dispatch_sync(self.cacheQueue, ^{
        NSDate *timestamp = self.cacheTimestamps[userID];
        if (timestamp && -[timestamp timeIntervalSinceNow] < kCacheExpiryTime) {
            user = [self.userCache objectForKey:userID];
        }
    });
    return user;
}

- (void)invalidateUserCache:(NSString *)userID {
    dispatch_barrier_async(self.cacheQueue, ^{
        [self.userCache removeObjectForKey:userID];
        [self.cacheTimestamps removeObjectForKey:userID];
    });
}

- (void)cachePosts:(NSArray<SAPost *> *)posts forKey:(NSString *)key {
    dispatch_barrier_async(self.cacheQueue, ^{
        [self.postsCache setObject:posts forKey:key];
        self.cacheTimestamps[key] = [NSDate date];
    });
}

- (nullable NSArray<SAPost *> *)cachedPostsForKey:(NSString *)key {
    __block NSArray *posts = nil;
    dispatch_sync(self.cacheQueue, ^{
        NSDate *timestamp = self.cacheTimestamps[key];
        if (timestamp && -[timestamp timeIntervalSinceNow] < kCacheExpiryTime) {
            posts = [self.postsCache objectForKey:key];
        }
    });
    return posts;
}

- (void)invalidatePostsCacheForKey:(NSString *)key {
    dispatch_barrier_async(self.cacheQueue, ^{
        [self.postsCache removeObjectForKey:key];
        [self.cacheTimestamps removeObjectForKey:key];
    });
}

- (void)cacheImageData:(NSData *)data forURL:(NSString *)urlString {
    dispatch_barrier_async(self.cacheQueue, ^{
        [self.imageCache setObject:data forKey:urlString cost:data.length];
    });
}

- (nullable NSData *)cachedImageDataForURL:(NSString *)urlString {
    __block NSData *data = nil;
    dispatch_sync(self.cacheQueue, ^{
        data = [self.imageCache objectForKey:urlString];
    });
    return data;
}

- (void)clearAllCache {
    dispatch_barrier_async(self.cacheQueue, ^{
        [self.userCache removeAllObjects];
        [self.postsCache removeAllObjects];
        [self.imageCache removeAllObjects];
        [self.cacheTimestamps removeAllObjects];
    });
}

@end
```

---

## 91.10 Create Post

```objc
// SACreatePostViewController.m
#import "SACreatePostViewController.h"
#import "SAPostService.h"
#import "SAImageManager.h"

@interface SACreatePostViewController () <UITextViewDelegate, UIImagePickerControllerDelegate, UINavigationControllerDelegate>

@property (nonatomic, strong) UIScrollView *scrollView;
@property (nonatomic, strong) UITextView *contentTextView;
@property (nonatomic, strong) UILabel *placeholderLabel;
@property (nonatomic, strong) UIImageView *selectedImageView;
@property (nonatomic, strong) UIButton *addImageButton;
@property (nonatomic, strong) UIButton *removeImageButton;
@property (nonatomic, strong) UIBarButtonItem *postButton;
@property (nonatomic, strong) UIImage *selectedImage;
@property (nonatomic, assign) BOOL isPosting;

@end

@implementation SACreatePostViewController

- (void)viewDidLoad {
    [super viewDidLoad];
    self.title = @"สร้างโพสต์ใหม่";
    self.view.backgroundColor = [UIColor systemBackgroundColor];
    
    [self setupNavigationBar];
    [self setupUI];
}

- (void)setupNavigationBar {
    UIBarButtonItem *cancelButton = [[UIBarButtonItem alloc] 
                                     initWithBarButtonSystemItem:UIBarButtonSystemItemCancel 
                                     target:self 
                                     action:@selector(cancelTapped)];
    self.navigationItem.leftBarButtonItem = cancelButton;
    
    self.postButton = [[UIBarButtonItem alloc] 
                       initWithTitle:@"โพสต์" 
                       style:UIBarButtonItemStyleDone 
                       target:self 
                       action:@selector(postTapped)];
    self.postButton.enabled = NO;
    self.navigationItem.rightBarButtonItem = self.postButton;
}

- (void)setupUI {
    self.scrollView = [[UIScrollView alloc] init];
    self.scrollView.translatesAutoresizingMaskIntoConstraints = NO;
    [self.view addSubview:self.scrollView];
    
    // Text View
    self.contentTextView = [[UITextView alloc] init];
    self.contentTextView.translatesAutoresizingMaskIntoConstraints = NO;
    self.contentTextView.font = [UIFont systemFontOfSize:16];
    self.contentTextView.delegate = self;
    self.contentTextView.backgroundColor = [UIColor clearColor];
    [self.scrollView addSubview:self.contentTextView];
    
    // Placeholder
    self.placeholderLabel = [[UILabel alloc] init];
    self.placeholderLabel.translatesAutoresizingMaskIntoConstraints = NO;
    self.placeholderLabel.text = @"แบ่งปันอะไรบางอย่าง...";
    self.placeholderLabel.textColor = [UIColor placeholderTextColor];
    self.placeholderLabel.font = [UIFont systemFontOfSize:16];
    [self.contentTextView addSubview:self.placeholderLabel];
    
    // Image View
    self.selectedImageView = [[UIImageView alloc] init];
    self.selectedImageView.translatesAutoresizingMaskIntoConstraints = NO;
    self.selectedImageView.contentMode = UIViewContentModeScaleAspectFill;
    self.selectedImageView.clipsToBounds = YES;
    self.selectedImageView.layer.cornerRadius = 8;
    self.selectedImageView.hidden = YES;
    [self.scrollView addSubview:self.selectedImageView];
    
    // Add Image Button
    self.addImageButton = [UIButton buttonWithType:UIButtonTypeSystem];
    self.addImageButton.translatesAutoresizingMaskIntoConstraints = NO;
    [self.addImageButton setImage:[UIImage systemImageNamed:@"photo.on.rectangle"] forState:UIControlStateNormal];
    [self.addImageButton setTitle:@" เพิ่มรูปภาพ" forState:UIControlStateNormal];
    self.addImageButton.tintColor = [UIColor systemBlueColor];
    [self.addImageButton addTarget:self action:@selector(addImageTapped) 
                  forControlEvents:UIControlEventTouchUpInside];
    [self.scrollView addSubview:self.addImageButton];
    
    // Constraints
    [NSLayoutConstraint activateConstraints:@[
        [self.scrollView.topAnchor constraintEqualToAnchor:self.view.safeAreaLayoutGuide.topAnchor],
        [self.scrollView.leadingAnchor constraintEqualToAnchor:self.view.leadingAnchor],
        [self.scrollView.trailingAnchor constraintEqualToAnchor:self.view.trailingAnchor],
        [self.scrollView.bottomAnchor constraintEqualToAnchor:self.view.keyboardLayoutGuide.topAnchor],
        
        [self.contentTextView.topAnchor constraintEqualToAnchor:self.scrollView.topAnchor constant:16],
        [self.contentTextView.leadingAnchor constraintEqualToAnchor:self.view.leadingAnchor constant:16],
        [self.contentTextView.trailingAnchor constraintEqualToAnchor:self.view.trailingAnchor constant:-16],
        [self.contentTextView.heightAnchor constraintGreaterThanOrEqualToConstant:150],
        
        [self.placeholderLabel.topAnchor constraintEqualToAnchor:self.contentTextView.topAnchor constant:8],
        [self.placeholderLabel.leadingAnchor constraintEqualToAnchor:self.contentTextView.leadingAnchor constant:5],
        
        [self.selectedImageView.topAnchor constraintEqualToAnchor:self.contentTextView.bottomAnchor constant:16],
        [self.selectedImageView.leadingAnchor constraintEqualToAnchor:self.view.leadingAnchor constant:16],
        [self.selectedImageView.trailingAnchor constraintEqualToAnchor:self.view.trailingAnchor constant:-16],
        [self.selectedImageView.heightAnchor constraintEqualToConstant:200],
        
        [self.addImageButton.topAnchor constraintEqualToAnchor:self.selectedImageView.bottomAnchor constant:16],
        [self.addImageButton.leadingAnchor constraintEqualToAnchor:self.view.leadingAnchor constant:16],
        [self.addImageButton.bottomAnchor constraintEqualToAnchor:self.scrollView.bottomAnchor constant:-16]
    ]];
    
    [self.contentTextView becomeFirstResponder];
}

- (void)addImageTapped {
    UIImagePickerController *picker = [[UIImagePickerController alloc] init];
    picker.sourceType = UIImagePickerControllerSourceTypePhotoLibrary;
    picker.delegate = self;
    picker.allowsEditing = YES;
    [self presentViewController:picker animated:YES completion:nil];
}

- (void)imagePickerController:(UIImagePickerController *)picker 
didFinishPickingMediaWithInfo:(NSDictionary<UIImagePickerControllerInfoKey, id> *)info {
    UIImage *image = info[UIImagePickerControllerEditedImage] ?: info[UIImagePickerControllerOriginalImage];
    
    if (image) {
        self.selectedImage = image;
        self.selectedImageView.image = image;
        self.selectedImageView.hidden = NO;
    }
    
    [picker dismissViewControllerAnimated:YES completion:nil];
}

- (void)imagePickerControllerDidCancel:(UIImagePickerController *)picker {
    [picker dismissViewControllerAnimated:YES completion:nil];
}

- (void)postTapped {
    NSString *content = [self.contentTextView.text stringByTrimmingCharactersInSet:[NSCharacterSet whitespaceAndNewlineCharacterSet]];
    
    if (content.length == 0 && !self.selectedImage) {
        return;
    }
    
    self.isPosting = YES;
    self.postButton.enabled = NO;
    
    if (self.selectedImage) {
        [[SAImageManager sharedManager] uploadImage:self.selectedImage 
                                         completion:^(NSString *imageURL, NSError *error) {
            if (error) {
                [self showError:error];
                self.isPosting = NO;
                self.postButton.enabled = YES;
                return;
            }
            
            [self createPostWithContent:content imageURL:imageURL];
        }];
    } else {
        [self createPostWithContent:content imageURL:nil];
    }
}

- (void)createPostWithContent:(NSString *)content imageURL:(nullable NSString *)imageURL {
    NSMutableDictionary *params = [NSMutableDictionary dictionaryWithObject:content forKey:@"content"];
    if (imageURL) params[@"image_url"] = imageURL;
    
    [[SAPostService sharedService] createPostWithParameters:params 
                                                completion:^(SAPost *post, NSError *error) {
        self.isPosting = NO;
        
        if (error) {
            [self showError:error];
            self.postButton.enabled = YES;
            return;
        }
        
        if (self.onPostCreated) self.onPostCreated(post);
        [self dismissViewControllerAnimated:YES completion:nil];
    }];
}

- (void)cancelTapped {
    if (self.contentTextView.text.length > 0 || self.selectedImage) {
        UIAlertController *alert = [UIAlertController 
                                    alertControllerWithTitle:@"ยกเลิกโพสต์?"
                                    message:@"ร่างโพสต์ของคุณจะหายไป"
                                    preferredStyle:UIAlertControllerStyleAlert];
        [alert addAction:[UIAlertAction actionWithTitle:@"ยกเลิก" style:UIAlertActionStyleCancel handler:nil]];
        [alert addAction:[UIAlertAction actionWithTitle:@"ทิ้งร่าง" 
                                                  style:UIAlertActionStyleDestructive 
                                                handler:^(UIAlertAction *action) {
            [self dismissViewControllerAnimated:YES completion:nil];
        }]];
        [self presentViewController:alert animated:YES completion:nil];
    } else {
        [self dismissViewControllerAnimated:YES completion:nil];
    }
}

// UITextViewDelegate
- (void)textViewDidChange:(UITextView *)textView {
    self.placeholderLabel.hidden = textView.text.length > 0;
    self.postButton.enabled = textView.text.length > 0 || self.selectedImage != nil;
}

- (void)showError:(NSError *)error {
    UIAlertController *alert = [UIAlertController alertControllerWithTitle:@"เกิดข้อผิดพลาด"
                                                                   message:error.localizedDescription
                                                            preferredStyle:UIAlertControllerStyleAlert];
    [alert addAction:[UIAlertAction actionWithTitle:@"ตกลง" style:UIAlertActionStyleDefault handler:nil]];
    [self presentViewController:alert animated:YES completion:nil];
}

@end
```

---

## 91.11 Notifications

```objc
// SANotificationsViewController.m
#import "SANotificationsViewController.h"
#import "SANotificationCell.h"
#import "SANotificationService.h"

@interface SANotificationsViewController () <UITableViewDataSource, UITableViewDelegate>

@property (nonatomic, strong) UITableView *tableView;
@property (nonatomic, strong) NSMutableArray<SANotification *> *notifications;

@end

@implementation SANotificationsViewController

- (void)viewDidLoad {
    [super viewDidLoad];
    self.title = @"การแจ้งเตือน";
    self.notifications = [NSMutableArray array];
    
    [self setupTableView];
    [self loadNotifications];
    [self markAllAsRead];
}

- (void)setupTableView {
    self.tableView = [[UITableView alloc] initWithFrame:self.view.bounds style:UITableViewStylePlain];
    self.tableView.translatesAutoresizingMaskIntoConstraints = NO;
    self.tableView.dataSource = self;
    self.tableView.delegate = self;
    self.tableView.rowHeight = UITableViewAutomaticDimension;
    self.tableView.estimatedRowHeight = 80;
    [self.tableView registerClass:[SANotificationCell class] forCellReuseIdentifier:@"NotificationCell"];
    [self.view addSubview:self.tableView];
    
    [NSLayoutConstraint activateConstraints:@[
        [self.tableView.topAnchor constraintEqualToAnchor:self.view.safeAreaLayoutGuide.topAnchor],
        [self.tableView.leadingAnchor constraintEqualToAnchor:self.view.leadingAnchor],
        [self.tableView.trailingAnchor constraintEqualToAnchor:self.view.trailingAnchor],
        [self.tableView.bottomAnchor constraintEqualToAnchor:self.view.bottomAnchor]
    ]];
}

- (void)loadNotifications {
    [[SANotificationService sharedService] getNotificationsWithCompletion:^(NSArray<SANotification *> *notifications, NSError *error) {
        if (error) return;
        
        [self.notifications removeAllObjects];
        [self.notifications addObjectsFromArray:notifications];
        [self.tableView reloadData];
    }];
}

- (void)markAllAsRead {
    [[SANotificationService sharedService] markAllAsReadWithCompletion:^(NSError *error) {
        // Update badge
        [UIApplication sharedApplication].applicationIconBadgeNumber = 0;
        
        // Update tab bar badge
        UITabBarItem *tabItem = self.tabBarItem;
        tabItem.badgeValue = nil;
    }];
}

- (NSInteger)tableView:(UITableView *)tableView numberOfRowsInSection:(NSInteger)section {
    return self.notifications.count;
}

- (UITableViewCell *)tableView:(UITableView *)tableView cellForRowAtIndexPath:(NSIndexPath *)indexPath {
    SANotificationCell *cell = [tableView dequeueReusableCellWithIdentifier:@"NotificationCell" 
                                                               forIndexPath:indexPath];
    [cell configureWithNotification:self.notifications[indexPath.row]];
    return cell;
}

@end
```

---

## 91.12 Chat/Messaging Basics

```objc
// SAChatViewController.m
#import "SAChatViewController.h"
#import "SAChatService.h"
#import "SAMessage.h"

@interface SAChatViewController () <UITableViewDataSource, UITableViewDelegate, UITextFieldDelegate>

@property (nonatomic, strong) UITableView *tableView;
@property (nonatomic, strong) UIView *inputContainer;
@property (nonatomic, strong) UITextField *messageTextField;
@property (nonatomic, strong) UIButton *sendButton;
@property (nonatomic, strong) NSMutableArray<SAMessage *> *messages;
@property (nonatomic, strong) SAUser *recipient;
@property (nonatomic, strong) NSString *conversationID;

@end

@implementation SAChatViewController

- (instancetype)initWithRecipient:(SAUser *)recipient {
    if (self = [super init]) {
        _recipient = recipient;
    }
    return self;
}

- (void)viewDidLoad {
    [super viewDidLoad];
    self.title = self.recipient.displayName ?: self.recipient.username;
    self.messages = [NSMutableArray array];
    
    [self setupTableView];
    [self setupInputBar];
    [self loadMessages];
    
    [[NSNotificationCenter defaultCenter] addObserver:self 
                                             selector:@selector(keyboardWillShow:) 
                                                 name:UIKeyboardWillShowNotification 
                                               object:nil];
    [[NSNotificationCenter defaultCenter] addObserver:self 
                                             selector:@selector(keyboardWillHide:) 
                                                 name:UIKeyboardWillHideNotification 
                                               object:nil];
}

- (void)setupTableView {
    self.tableView = [[UITableView alloc] initWithFrame:self.view.bounds style:UITableViewStylePlain];
    self.tableView.translatesAutoresizingMaskIntoConstraints = NO;
    self.tableView.dataSource = self;
    self.tableView.delegate = self;
    self.tableView.separatorStyle = UITableViewCellSeparatorStyleNone;
    self.tableView.rowHeight = UITableViewAutomaticDimension;
    self.tableView.estimatedRowHeight = 60;
    self.tableView.transform = CGAffineTransformMakeScale(1, -1);
    [self.view addSubview:self.tableView];
    
    [NSLayoutConstraint activateConstraints:@[
        [self.tableView.topAnchor constraintEqualToAnchor:self.view.safeAreaLayoutGuide.topAnchor],
        [self.tableView.leadingAnchor constraintEqualToAnchor:self.view.leadingAnchor],
        [self.tableView.trailingAnchor constraintEqualToAnchor:self.view.trailingAnchor]
    ]];
}

- (void)setupInputBar {
    self.inputContainer = [[UIView alloc] init];
    self.inputContainer.translatesAutoresizingMaskIntoConstraints = NO;
    self.inputContainer.backgroundColor = [UIColor secondarySystemBackgroundColor];
    [self.view addSubview:self.inputContainer];
    
    self.messageTextField = [[UITextField alloc] init];
    self.messageTextField.translatesAutoresizingMaskIntoConstraints = NO;
    self.messageTextField.placeholder = @"พิมพ์ข้อความ...";
    self.messageTextField.borderStyle = UITextBorderStyleRoundedRect;
    self.messageTextField.delegate = self;
    [self.inputContainer addSubview:self.messageTextField];
    
    self.sendButton = [UIButton buttonWithType:UIButtonTypeSystem];
    self.sendButton.translatesAutoresizingMaskIntoConstraints = NO;
    [self.sendButton setImage:[UIImage systemImageNamed:@"paperplane.fill"] forState:UIControlStateNormal];
    self.sendButton.tintColor = [UIColor systemBlueColor];
    [self.sendButton addTarget:self action:@selector(sendMessage) forControlEvents:UIControlEventTouchUpInside];
    [self.inputContainer addSubview:self.sendButton];
    
    [NSLayoutConstraint activateConstraints:@[
        [self.inputContainer.topAnchor constraintEqualToAnchor:self.tableView.bottomAnchor],
        [self.inputContainer.leadingAnchor constraintEqualToAnchor:self.view.leadingAnchor],
        [self.inputContainer.trailingAnchor constraintEqualToAnchor:self.view.trailingAnchor],
        [self.inputContainer.bottomAnchor constraintEqualToAnchor:self.view.keyboardLayoutGuide.topAnchor],
        [self.inputContainer.heightAnchor constraintGreaterThanOrEqualToConstant:60],
        
        [self.messageTextField.topAnchor constraintEqualToAnchor:self.inputContainer.topAnchor constant:8],
        [self.messageTextField.leadingAnchor constraintEqualToAnchor:self.inputContainer.leadingAnchor constant:16],
        [self.messageTextField.bottomAnchor constraintEqualToAnchor:self.inputContainer.bottomAnchor constant:-8],
        [self.messageTextField.trailingAnchor constraintEqualToAnchor:self.sendButton.leadingAnchor constant:-8],
        
        [self.sendButton.centerYAnchor constraintEqualToAnchor:self.inputContainer.centerYAnchor],
        [self.sendButton.trailingAnchor constraintEqualToAnchor:self.inputContainer.trailingAnchor constant:-16],
        [self.sendButton.widthAnchor constraintEqualToConstant:44],
        [self.sendButton.heightAnchor constraintEqualToConstant:44]
    ]];
}

- (void)loadMessages {
    [[SAChatService sharedService] getMessagesForConversationID:self.conversationID
                                                    completion:^(NSArray<SAMessage *> *messages, 
                                                                NSError *error) {
        if (error) return;
        
        [self.messages removeAllObjects];
        [self.messages addObjectsFromArray:messages];
        [self.tableView reloadData];
        
        // Scroll to bottom
        if (self.messages.count > 0) {
            [self.tableView scrollToRowAtIndexPath:[NSIndexPath indexPathForRow:0 inSection:0]
                                 atScrollPosition:UITableViewScrollPositionBottom 
                                         animated:NO];
        }
    }];
}

- (void)sendMessage {
    NSString *text = [self.messageTextField.text stringByTrimmingCharactersInSet:[NSCharacterSet whitespaceAndNewlineCharacterSet]];
    if (text.length == 0) return;
    
    self.messageTextField.text = @"";
    
    [[SAChatService sharedService] sendMessage:text 
                              toConversationID:self.conversationID
                                    completion:^(SAMessage *message, NSError *error) {
        if (error || !message) return;
        
        [self.messages insertObject:message atIndex:0];
        [self.tableView insertRowsAtIndexPaths:@[[NSIndexPath indexPathForRow:0 inSection:0]]
                              withRowAnimation:UITableViewRowAnimationTop];
    }];
}

// UITableViewDataSource
- (NSInteger)tableView:(UITableView *)tableView numberOfRowsInSection:(NSInteger)section {
    return self.messages.count;
}

- (UITableViewCell *)tableView:(UITableView *)tableView cellForRowAtIndexPath:(NSIndexPath *)indexPath {
    UITableViewCell *cell = [tableView dequeueReusableCellWithIdentifier:@"MessageCell"];
    if (!cell) {
        cell = [[UITableViewCell alloc] initWithStyle:UITableViewCellStyleDefault reuseIdentifier:@"MessageCell"];
    }
    cell.transform = CGAffineTransformMakeScale(1, -1);
    
    SAMessage *message = self.messages[indexPath.row];
    cell.textLabel.text = message.content;
    
    return cell;
}

- (void)keyboardWillShow:(NSNotification *)notification {
    // iOS 15+ keyboardLayoutGuide handles this automatically
}

- (void)keyboardWillHide:(NSNotification *)notification {
    // iOS 15+ keyboardLayoutGuide handles this automatically
}

- (void)dealloc {
    [[NSNotificationCenter defaultCenter] removeObserver:self];
}

@end
```

---

## สรุป Part 91

ในบทนี้เราได้สร้าง Social Media App แบบสมบูรณ์ด้วย Objective-C ครอบคลุม:

1. **Architecture** - MVC + Service Layer pattern
2. **Data Models** - SAUser, SAPost พร้อม NSCoding/NSCopying
3. **Networking Layer** - SANetworkManager แบบ reusable
4. **Authentication** - Login/Register พร้อม session persistence
5. **Feed** - UITableView พร้อม pagination และ pull-to-refresh
6. **Post Cell** - Custom cell พร้อม delegate pattern
7. **User Profile** - Profile header, follow/unfollow
8. **Search** - UISearchController พร้อม debouncing
9. **Caching** - NSCache-based caching strategy
10. **Create Post** - Image picker integration
11. **Notifications** - Notification center pattern
12. **Chat** - Basic messaging implementation

---

*บทต่อไป: ตอนที่ 92 - โปรเจกต์สมบูรณ์: E-Commerce App*
