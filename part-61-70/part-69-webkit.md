# Part 69: WebKit Framework ใน Objective-C

## บทนำ

WebKit เป็น framework ที่ Apple พัฒนาขึ้นเพื่อให้นักพัฒนาสามารถแสดงเนื้อหาเว็บในแอปพลิเคชัน iOS และ macOS ได้อย่างมีประสิทธิภาพ ใน iOS 8 Apple ได้แนะนำ `WKWebView` เพื่อแทนที่ `UIWebView` เดิม โดย `WKWebView` มีประสิทธิภาพสูงกว่า ใช้ memory น้อยกว่า และรองรับ JavaScript engine ที่ทันสมัยกว่า

ในบทนี้เราจะเรียนรู้:
- การตั้งค่าและใช้งาน WKWebView
- การโหลด URL และ HTML String
- การใช้งาน WKNavigationDelegate
- การใช้งาน WKUIDelegate
- การ inject JavaScript
- การสื่อสารระหว่าง JavaScript และ Native code
- การจัดการ Cookies
- การกำหนดค่า WKWebViewConfiguration
- Content Blockers

---

## 69.1 การติดตั้งและ Setup WKWebView

### การ Import Framework

```objc
#import <WebKit/WebKit.h>
```

### การสร้าง WKWebView แบบพื้นฐาน

```objc
// ViewController.h
#import <UIKit/UIKit.h>
#import <WebKit/WebKit.h>

@interface ViewController : UIViewController <WKNavigationDelegate, WKUIDelegate>

@property (nonatomic, strong) WKWebView *webView;

@end
```

```objc
// ViewController.m
#import "ViewController.h"

@implementation ViewController

- (void)viewDidLoad {
    [super viewDidLoad];
    
    // สร้าง WKWebView ด้วย frame เท่ากับ view ของ ViewController
    self.webView = [[WKWebView alloc] initWithFrame:self.view.bounds];
    
    // กำหนด delegate
    self.webView.navigationDelegate = self;
    self.webView.UIDelegate = self;
    
    // ทำให้ webView ขยายได้เมื่อหมุนหน้าจอ
    self.webView.autoresizingMask = UIViewAutoresizingFlexibleWidth | UIViewAutoresizingFlexibleHeight;
    
    // เพิ่ม webView เป็น subview
    [self.view addSubview:self.webView];
}

@end
```

### การใช้ Auto Layout กับ WKWebView

```objc
- (void)setupWebViewWithAutoLayout {
    self.webView = [[WKWebView alloc] init];
    self.webView.translatesAutoresizingMaskIntoConstraints = NO;
    self.webView.navigationDelegate = self;
    
    [self.view addSubview:self.webView];
    
    // กำหนด constraints ให้ webView เต็ม safe area
    [NSLayoutConstraint activateConstraints:@[
        [self.webView.topAnchor constraintEqualToAnchor:self.view.safeAreaLayoutGuide.topAnchor],
        [self.webView.leadingAnchor constraintEqualToAnchor:self.view.leadingAnchor],
        [self.webView.trailingAnchor constraintEqualToAnchor:self.view.trailingAnchor],
        [self.webView.bottomAnchor constraintEqualToAnchor:self.view.safeAreaLayoutGuide.bottomAnchor]
    ]];
}
```

---

## 69.2 การโหลด URL และ HTML

### การโหลด URL จากอินเทอร์เน็ต

```objc
- (void)loadURL:(NSString *)urlString {
    NSURL *url = [NSURL URLWithString:urlString];
    
    if (url) {
        NSURLRequest *request = [NSURLRequest requestWithURL:url];
        [self.webView loadRequest:request];
    } else {
        NSLog(@"URL ไม่ถูกต้อง: %@", urlString);
    }
}

// เรียกใช้งาน
[self loadURL:@"https://www.apple.com"];
```

### การโหลด URL พร้อม Custom Headers

```objc
- (void)loadURLWithCustomHeaders:(NSString *)urlString {
    NSURL *url = [NSURL URLWithString:urlString];
    NSMutableURLRequest *request = [NSMutableURLRequest requestWithURL:url];
    
    // เพิ่ม headers ที่ต้องการ
    [request setValue:@"MyApp/1.0" forHTTPHeaderField:@"User-Agent"];
    [request setValue:@"application/json" forHTTPHeaderField:@"Accept"];
    [request setValue:@"Bearer mytoken123" forHTTPHeaderField:@"Authorization"];
    
    [self.webView loadRequest:request];
}
```

### การโหลด HTML String โดยตรง

```objc
- (void)loadHTMLString {
    NSString *htmlString = @"<!DOCTYPE html>"
                           "<html>"
                           "<head>"
                           "    <meta name='viewport' content='width=device-width, initial-scale=1.0'>"
                           "    <style>"
                           "        body { font-family: -apple-system; padding: 20px; }"
                           "        h1 { color: #007AFF; }"
                           "    </style>"
                           "</head>"
                           "<body>"
                           "    <h1>สวัสดีจาก WKWebView!</h1>"
                           "    <p>นี่คือ HTML ที่โหลดจาก Objective-C</p>"
                           "</body>"
                           "</html>";
    
    // baseURL ใช้สำหรับ resolve relative URLs
    NSURL *baseURL = [NSURL fileURLWithPath:[[NSBundle mainBundle] bundlePath]];
    [self.webView loadHTMLString:htmlString baseURL:baseURL];
}
```

### การโหลดไฟล์ HTML จาก Bundle

```objc
- (void)loadLocalHTMLFile {
    NSString *filePath = [[NSBundle mainBundle] pathForResource:@"index" ofType:@"html"];
    
    if (filePath) {
        NSURL *fileURL = [NSURL fileURLWithPath:filePath];
        [self.webView loadFileURL:fileURL allowingReadAccessToURL:[fileURL URLByDeletingLastPathComponent]];
    } else {
        NSLog(@"ไม่พบไฟล์ HTML");
    }
}
```

### การโหลด Data โดยตรง

```objc
- (void)loadDataDirectly {
    NSString *htmlContent = @"<html><body><h1>Hello from Data!</h1></body></html>";
    NSData *data = [htmlContent dataUsingEncoding:NSUTF8StringEncoding];
    
    [self.webView loadData:data
                 MIMEType:@"text/html"
    characterEncodingName:@"UTF-8"
                  baseURL:[NSURL URLWithString:@"https://example.com"]];
}
```

---

## 69.3 WKNavigationDelegate

`WKNavigationDelegate` ใช้สำหรับติดตามสถานะการโหลดหน้าเว็บ และควบคุมการ navigation

### Protocol Methods หลัก

```objc
@implementation ViewController

#pragma mark - WKNavigationDelegate

// เรียกเมื่อเริ่มต้นการโหลดหน้าเว็บ
- (void)webView:(WKWebView *)webView didStartProvisionalNavigation:(WKNavigation *)navigation {
    NSLog(@"เริ่มโหลดหน้าเว็บ...");
    [UIApplication sharedApplication].networkActivityIndicatorVisible = YES;
    
    // แสดง Loading Indicator
    [self showLoadingIndicator];
}

// เรียกเมื่อได้รับ response จากเซิร์ฟเวอร์แล้ว
- (void)webView:(WKWebView *)webView didCommitNavigation:(WKNavigation *)navigation {
    NSLog(@"กำลังโหลดเนื้อหา...");
}

// เรียกเมื่อโหลดเสร็จสมบูรณ์
- (void)webView:(WKWebView *)webView didFinishNavigation:(WKNavigation *)navigation {
    NSLog(@"โหลดเสร็จแล้ว: %@", webView.URL.absoluteString);
    [UIApplication sharedApplication].networkActivityIndicatorVisible = NO;
    [self hideLoadingIndicator];
    
    // อัพเดต UI
    self.title = webView.title;
    [self updateNavigationButtons];
}

// เรียกเมื่อเกิด error ระหว่างโหลด
- (void)webView:(WKWebView *)webView didFailNavigation:(WKNavigation *)navigation withError:(NSError *)error {
    NSLog(@"เกิด error: %@", error.localizedDescription);
    [self hideLoadingIndicator];
    [self showErrorAlert:error.localizedDescription];
}

// เรียกเมื่อเกิด error ก่อนเริ่มโหลด (เช่น DNS ไม่พบ)
- (void)webView:(WKWebView *)webView didFailProvisionalNavigation:(WKNavigation *)navigation withError:(NSError *)error {
    NSLog(@"เกิด error ก่อนโหลด: %@", error.localizedDescription);
    [self hideLoadingIndicator];
    
    // ตรวจสอบประเภท error
    if (error.code == NSURLErrorNotConnectedToInternet) {
        [self showErrorAlert:@"ไม่มีการเชื่อมต่ออินเทอร์เน็ต"];
    } else {
        [self showErrorAlert:error.localizedDescription];
    }
}

// ใช้สำหรับตัดสินใจว่าจะอนุญาต navigation หรือไม่
- (void)webView:(WKWebView *)webView
decidePolicyForNavigationAction:(WKNavigationAction *)navigationAction
decisionHandler:(void (^)(WKNavigationActionPolicy))decisionHandler {
    
    NSURL *url = navigationAction.request.URL;
    NSLog(@"กำลังจะไปที่: %@", url.absoluteString);
    
    // ตรวจสอบ URL scheme
    if ([url.scheme isEqualToString:@"tel"]) {
        // เปิด dialer
        [[UIApplication sharedApplication] openURL:url options:@{} completionHandler:nil];
        decisionHandler(WKNavigationActionPolicyCancel);
        return;
    }
    
    if ([url.scheme isEqualToString:@"mailto"]) {
        // เปิด Mail app
        [[UIApplication sharedApplication] openURL:url options:@{} completionHandler:nil];
        decisionHandler(WKNavigationActionPolicyCancel);
        return;
    }
    
    // ตรวจสอบ domain ที่ไม่อนุญาต
    NSArray *blockedDomains = @[@"ads.example.com", @"tracker.example.com"];
    if ([blockedDomains containsObject:url.host]) {
        decisionHandler(WKNavigationActionPolicyCancel);
        return;
    }
    
    // อนุญาต navigation
    decisionHandler(WKNavigationActionPolicyAllow);
}

// เรียกเมื่อเซิร์ฟเวอร์ส่ง response กลับมา
- (void)webView:(WKWebView *)webView
decidePolicyForNavigationResponse:(WKNavigationResponse *)navigationResponse
decisionHandler:(void (^)(WKNavigationResponsePolicy))decisionHandler {
    
    NSHTTPURLResponse *httpResponse = (NSHTTPURLResponse *)navigationResponse.response;
    NSInteger statusCode = httpResponse.statusCode;
    
    NSLog(@"Response Status Code: %ld", (long)statusCode);
    
    if (statusCode == 404) {
        // จัดการ 404 error
        decisionHandler(WKNavigationResponsePolicyCancel);
        [self showErrorAlert:@"ไม่พบหน้าที่ต้องการ (404)"];
    } else {
        decisionHandler(WKNavigationResponsePolicyAllow);
    }
}

@end
```

### การจัดการ Back/Forward Navigation

```objc
- (void)setupNavigationBar {
    // ปุ่ม Back
    UIBarButtonItem *backButton = [[UIBarButtonItem alloc] 
        initWithTitle:@"◀" 
        style:UIBarButtonItemStylePlain 
        target:self 
        action:@selector(goBack)];
    
    // ปุ่ม Forward
    UIBarButtonItem *forwardButton = [[UIBarButtonItem alloc] 
        initWithTitle:@"▶" 
        style:UIBarButtonItemStylePlain 
        target:self 
        action:@selector(goForward)];
    
    // ปุ่ม Reload
    UIBarButtonItem *reloadButton = [[UIBarButtonItem alloc] 
        initWithBarButtonSystemItem:UIBarButtonSystemItemRefresh 
        target:self 
        action:@selector(reload)];
    
    self.navigationItem.leftBarButtonItems = @[backButton, forwardButton];
    self.navigationItem.rightBarButtonItem = reloadButton;
}

- (void)goBack {
    if (self.webView.canGoBack) {
        [self.webView goBack];
    }
}

- (void)goForward {
    if (self.webView.canGoForward) {
        [self.webView goForward];
    }
}

- (void)reload {
    [self.webView reload];
}

- (void)updateNavigationButtons {
    // อัพเดตสถานะปุ่ม
    self.backButton.enabled = self.webView.canGoBack;
    self.forwardButton.enabled = self.webView.canGoForward;
}
```

---

## 69.4 WKUIDelegate

`WKUIDelegate` ใช้สำหรับจัดการ UI ที่เว็บเพจต้องการแสดง เช่น Alert, Confirm dialog

```objc
@implementation ViewController

#pragma mark - WKUIDelegate

// จัดการ JavaScript alert()
- (void)webView:(WKWebView *)webView
runJavaScriptAlertPanelWithMessage:(NSString *)message
initiatedByFrame:(WKFrameInfo *)frame
completionHandler:(void (^)(void))completionHandler {
    
    UIAlertController *alertController = [UIAlertController 
        alertControllerWithTitle:@"แจ้งเตือนจากเว็บ"
        message:message
        preferredStyle:UIAlertControllerStyleAlert];
    
    UIAlertAction *okAction = [UIAlertAction 
        actionWithTitle:@"ตกลง"
        style:UIAlertActionStyleDefault
        handler:^(UIAlertAction *action) {
            completionHandler(); // ต้องเรียก completionHandler เสมอ
        }];
    
    [alertController addAction:okAction];
    [self presentViewController:alertController animated:YES completion:nil];
}

// จัดการ JavaScript confirm()
- (void)webView:(WKWebView *)webView
runJavaScriptConfirmPanelWithMessage:(NSString *)message
initiatedByFrame:(WKFrameInfo *)frame
completionHandler:(void (^)(BOOL result))completionHandler {
    
    UIAlertController *alertController = [UIAlertController 
        alertControllerWithTitle:@"ยืนยัน"
        message:message
        preferredStyle:UIAlertControllerStyleAlert];
    
    UIAlertAction *confirmAction = [UIAlertAction 
        actionWithTitle:@"ตกลง"
        style:UIAlertActionStyleDefault
        handler:^(UIAlertAction *action) {
            completionHandler(YES);
        }];
    
    UIAlertAction *cancelAction = [UIAlertAction 
        actionWithTitle:@"ยกเลิก"
        style:UIAlertActionStyleCancel
        handler:^(UIAlertAction *action) {
            completionHandler(NO);
        }];
    
    [alertController addAction:confirmAction];
    [alertController addAction:cancelAction];
    [self presentViewController:alertController animated:YES completion:nil];
}

// จัดการ JavaScript prompt()
- (void)webView:(WKWebView *)webView
runJavaScriptTextInputPanelWithPrompt:(NSString *)prompt
defaultText:(NSString *)defaultText
initiatedByFrame:(WKFrameInfo *)frame
completionHandler:(void (^)(NSString *result))completionHandler {
    
    UIAlertController *alertController = [UIAlertController 
        alertControllerWithTitle:@"ป้อนข้อมูล"
        message:prompt
        preferredStyle:UIAlertControllerStyleAlert];
    
    [alertController addTextFieldWithConfigurationHandler:^(UITextField *textField) {
        textField.text = defaultText;
        textField.placeholder = @"ป้อนข้อความ...";
    }];
    
    UIAlertAction *okAction = [UIAlertAction 
        actionWithTitle:@"ตกลง"
        style:UIAlertActionStyleDefault
        handler:^(UIAlertAction *action) {
            NSString *inputText = alertController.textFields.firstObject.text;
            completionHandler(inputText);
        }];
    
    UIAlertAction *cancelAction = [UIAlertAction 
        actionWithTitle:@"ยกเลิก"
        style:UIAlertActionStyleCancel
        handler:^(UIAlertAction *action) {
            completionHandler(nil);
        }];
    
    [alertController addAction:okAction];
    [alertController addAction:cancelAction];
    [self presentViewController:alertController animated:YES completion:nil];
}

// จัดการเมื่อเว็บเปิด window ใหม่
- (WKWebView *)webView:(WKWebView *)webView
createWebViewWithConfiguration:(WKWebViewConfiguration *)configuration
forNavigationAction:(WKNavigationAction *)navigationAction
windowFeatures:(WKWindowFeatures *)windowFeatures {
    
    // ตรวจสอบว่าเป็น target="_blank"
    if (!navigationAction.targetFrame.isMainFrame) {
        // โหลดใน webView เดิม แทนที่จะเปิดหน้าต่างใหม่
        [self.webView loadRequest:navigationAction.request];
    }
    
    return nil;
}

@end
```

---

## 69.5 JavaScript Injection ด้วย evaluateJavaScript

### การรัน JavaScript จาก Native Code

```objc
// การรัน JavaScript แบบง่าย
- (void)runSimpleJavaScript {
    [self.webView evaluateJavaScript:@"document.title" 
                   completionHandler:^(id result, NSError *error) {
        if (error) {
            NSLog(@"JavaScript error: %@", error.localizedDescription);
        } else {
            NSLog(@"ชื่อหน้าเว็บ: %@", result);
        }
    }];
}

// การเปลี่ยน UI ผ่าน JavaScript
- (void)changePageBackground:(UIColor *)color {
    CGFloat r, g, b, a;
    [color getRed:&r green:&g blue:&b alpha:&a];
    
    NSString *jsCode = [NSString stringWithFormat:
        @"document.body.style.backgroundColor = 'rgb(%d, %d, %d)';",
        (int)(r * 255), (int)(g * 255), (int)(b * 255)];
    
    [self.webView evaluateJavaScript:jsCode completionHandler:nil];
}

// การ scroll ไปยังตำแหน่งที่กำหนด
- (void)scrollToPosition:(CGFloat)yPosition {
    NSString *jsCode = [NSString stringWithFormat:
        @"window.scrollTo({top: %f, behavior: 'smooth'});", yPosition];
    
    [self.webView evaluateJavaScript:jsCode completionHandler:nil];
}

// การดึงข้อมูลจากหน้าเว็บ
- (void)extractPageData {
    NSString *jsCode = @"JSON.stringify({"
                       "    title: document.title,"
                       "    url: window.location.href,"
                       "    links: Array.from(document.querySelectorAll('a')).map(a => a.href).slice(0, 10)"
                       "})";
    
    [self.webView evaluateJavaScript:jsCode 
                   completionHandler:^(id result, NSError *error) {
        if (!error && result) {
            NSData *jsonData = [result dataUsingEncoding:NSUTF8StringEncoding];
            NSDictionary *pageData = [NSJSONSerialization JSONObjectWithData:jsonData 
                                                                     options:0 
                                                                       error:nil];
            NSLog(@"Page Data: %@", pageData);
        }
    }];
}

// การ inject CSS
- (void)injectCSS:(NSString *)cssString {
    NSString *escapedCSS = [cssString stringByReplacingOccurrencesOfString:@"'" withString:@"\\'"];
    NSString *jsCode = [NSString stringWithFormat:
        @"var style = document.createElement('style');"
        @"style.textContent = '%@';"
        @"document.head.appendChild(style);",
        escapedCSS];
    
    [self.webView evaluateJavaScript:jsCode completionHandler:^(id result, NSError *error) {
        if (error) {
            NSLog(@"Error injecting CSS: %@", error.localizedDescription);
        } else {
            NSLog(@"CSS injected successfully");
        }
    }];
}

// ตัวอย่างการใช้งาน inject CSS
- (void)hideAds {
    NSString *adBlockCSS = @".ad, .advertisement, [id*='ad'], [class*='ad'] { display: none !important; }";
    [self injectCSS:adBlockCSS];
}
```

### การใช้ WKUserScript สำหรับ Inject ก่อน/หลังโหลด

```objc
- (WKWebViewConfiguration *)createConfigurationWithUserScript {
    WKWebViewConfiguration *config = [[WKWebViewConfiguration alloc] init];
    WKUserContentController *userContentController = [[WKUserContentController alloc] init];
    
    // Script ที่รันก่อนโหลดเอกสาร
    NSString *earlyScript = @"window.APP_STARTED = true;"
                            @"console.log('App script injected before document loaded');";
    
    WKUserScript *earlyUserScript = [[WKUserScript alloc]
        initWithSource:earlyScript
        injectionTime:WKUserScriptInjectionTimeAtDocumentStart
        forMainFrameOnly:YES];
    
    // Script ที่รันหลังโหลดเอกสารแล้ว
    NSString *lateScript = @"document.querySelectorAll('a[target=\"_blank\"]').forEach(function(link) {"
                           @"    link.setAttribute('target', '_self');"
                           @"});"
                           @"console.log('Links updated');";
    
    WKUserScript *lateUserScript = [[WKUserScript alloc]
        initWithSource:lateScript
        injectionTime:WKUserScriptInjectionTimeAtDocumentEnd
        forMainFrameOnly:YES];
    
    [userContentController addUserScript:earlyUserScript];
    [userContentController addUserScript:lateUserScript];
    
    config.userContentController = userContentController;
    return config;
}
```

---

## 69.6 WKScriptMessageHandler - การสื่อสารจาก JavaScript ไปยัง Native

### การตั้งค่า Message Handler

```objc
// ViewController.h
@interface ViewController : UIViewController <WKNavigationDelegate, WKScriptMessageHandler>

@property (nonatomic, strong) WKWebView *webView;

@end
```

```objc
// ViewController.m

- (void)setupWebViewWithMessageHandler {
    WKWebViewConfiguration *config = [[WKWebViewConfiguration alloc] init];
    WKUserContentController *userContentController = [[WKUserContentController alloc] init];
    
    // ลงทะเบียน message handlers
    [userContentController addScriptMessageHandler:self name:@"showAlert"];
    [userContentController addScriptMessageHandler:self name:@"logMessage"];
    [userContentController addScriptMessageHandler:self name:@"userAction"];
    [userContentController addScriptMessageHandler:self name:@"shareContent"];
    
    config.userContentController = userContentController;
    
    self.webView = [[WKWebView alloc] initWithFrame:self.view.bounds configuration:config];
    self.webView.navigationDelegate = self;
    [self.view addSubview:self.webView];
}

// Implement WKScriptMessageHandler
- (void)userContentController:(WKUserContentController *)userContentController
      didReceiveScriptMessage:(WKScriptMessage *)message {
    
    NSLog(@"Received message: %@ with body: %@", message.name, message.body);
    
    if ([message.name isEqualToString:@"showAlert"]) {
        // message.body คือ String จาก JavaScript
        NSString *alertMessage = (NSString *)message.body;
        
        UIAlertController *alert = [UIAlertController 
            alertControllerWithTitle:@"ข้อความจากเว็บ"
            message:alertMessage
            preferredStyle:UIAlertControllerStyleAlert];
        
        [alert addAction:[UIAlertAction actionWithTitle:@"OK" 
                                                  style:UIAlertActionStyleDefault 
                                                handler:nil]];
        [self presentViewController:alert animated:YES completion:nil];
    }
    else if ([message.name isEqualToString:@"logMessage"]) {
        NSString *logMessage = (NSString *)message.body;
        NSLog(@"JavaScript Log: %@", logMessage);
    }
    else if ([message.name isEqualToString:@"userAction"]) {
        // message.body คือ Dictionary จาก JavaScript
        if ([message.body isKindOfClass:[NSDictionary class]]) {
            NSDictionary *actionData = (NSDictionary *)message.body;
            NSString *action = actionData[@"action"];
            id data = actionData[@"data"];
            
            [self handleUserAction:action withData:data];
        }
    }
    else if ([message.name isEqualToString:@"shareContent"]) {
        NSDictionary *shareData = (NSDictionary *)message.body;
        NSString *text = shareData[@"text"];
        NSString *urlString = shareData[@"url"];
        
        NSMutableArray *items = [NSMutableArray array];
        if (text) [items addObject:text];
        if (urlString) [items addObject:[NSURL URLWithString:urlString]];
        
        UIActivityViewController *activityVC = [[UIActivityViewController alloc] 
            initWithActivityItems:items 
            applicationActivities:nil];
        [self presentViewController:activityVC animated:YES completion:nil];
    }
}

- (void)handleUserAction:(NSString *)action withData:(id)data {
    if ([action isEqualToString:@"login"]) {
        NSLog(@"User wants to login with data: %@", data);
        // ทำการ login
    } else if ([action isEqualToString:@"purchase"]) {
        NSLog(@"User wants to purchase: %@", data);
        // ดำเนินการซื้อ
    } else if ([action isEqualToString:@"navigate"]) {
        NSString *screen = (NSString *)data;
        NSLog(@"Navigate to screen: %@", screen);
        // นำทางไปหน้าอื่น
    }
}
```

### HTML และ JavaScript ที่ใช้ส่งข้อความมา Native

```html
<!-- index.html -->
<!DOCTYPE html>
<html>
<head>
    <meta name="viewport" content="width=device-width, initial-scale=1.0">
    <title>WebKit Demo</title>
    <style>
        body { font-family: -apple-system; padding: 20px; }
        button { 
            background: #007AFF; 
            color: white; 
            border: none; 
            padding: 12px 24px; 
            border-radius: 8px; 
            font-size: 16px;
            margin: 8px 0;
            display: block;
            width: 100%;
        }
    </style>
</head>
<body>
    <h1>WebKit Communication Demo</h1>
    
    <button onclick="showNativeAlert()">แสดง Native Alert</button>
    <button onclick="sendUserAction()">ส่ง User Action</button>
    <button onclick="shareContent()">แชร์เนื้อหา</button>
    
    <script>
        // ฟังก์ชันสำหรับส่งข้อความไปยัง Native
        function sendToNative(handlerName, data) {
            if (window.webkit && window.webkit.messageHandlers[handlerName]) {
                window.webkit.messageHandlers[handlerName].postMessage(data);
            } else {
                console.log('Message handler not available:', handlerName);
            }
        }
        
        function showNativeAlert() {
            sendToNative('showAlert', 'สวัสดีจาก JavaScript!');
        }
        
        function sendUserAction() {
            sendToNative('userAction', {
                action: 'login',
                data: {
                    username: 'user@example.com',
                    timestamp: Date.now()
                }
            });
        }
        
        function shareContent() {
            sendToNative('shareContent', {
                text: 'เนื้อหาที่น่าสนใจจากแอป',
                url: 'https://www.example.com'
            });
        }
        
        // Log ทุก click event
        document.addEventListener('click', function(event) {
            sendToNative('logMessage', 'Clicked on: ' + event.target.tagName);
        });
    </script>
</body>
</html>
```

### การป้องกัน Memory Leak ใน WKScriptMessageHandler

```objc
// ปัญหา: WKUserContentController retain ตัว handler อยู่ ทำให้เกิด retain cycle
// วิธีแก้: ใช้ Proxy Object

// WeakScriptMessageDelegate.h
@interface WeakScriptMessageDelegate : NSObject <WKScriptMessageHandler>

@property (nonatomic, weak) id<WKScriptMessageHandler> delegate;

- (instancetype)initWithDelegate:(id<WKScriptMessageHandler>)delegate;

@end

// WeakScriptMessageDelegate.m
@implementation WeakScriptMessageDelegate

- (instancetype)initWithDelegate:(id<WKScriptMessageHandler>)delegate {
    self = [super init];
    if (self) {
        _delegate = delegate;
    }
    return self;
}

- (void)userContentController:(WKUserContentController *)userContentController
      didReceiveScriptMessage:(WKScriptMessage *)message {
    [self.delegate userContentController:userContentController didReceiveScriptMessage:message];
}

@end

// การใช้งาน
- (void)setupWebViewSafely {
    WKWebViewConfiguration *config = [[WKWebViewConfiguration alloc] init];
    WKUserContentController *ucc = [[WKUserContentController alloc] init];
    
    // ใช้ WeakScriptMessageDelegate แทน
    WeakScriptMessageDelegate *weakDelegate = [[WeakScriptMessageDelegate alloc] 
        initWithDelegate:self];
    
    [ucc addScriptMessageHandler:weakDelegate name:@"myHandler"];
    config.userContentController = ucc;
    
    self.webView = [[WKWebView alloc] initWithFrame:self.view.bounds configuration:config];
    [self.view addSubview:self.webView];
}

// อย่าลืม removeScriptMessageHandler เมื่อ dealloc
- (void)dealloc {
    [self.webView.configuration.userContentController removeScriptMessageHandlerForName:@"myHandler"];
}
```

---

## 69.7 การสื่อสารจาก Native ไปยัง JavaScript

```objc
// ส่งข้อมูลไปยัง JavaScript
- (void)sendDataToJavaScript:(NSDictionary *)data {
    NSError *error;
    NSData *jsonData = [NSJSONSerialization dataWithJSONObject:data 
                                                       options:0 
                                                         error:&error];
    if (error) {
        NSLog(@"Error serializing data: %@", error.localizedDescription);
        return;
    }
    
    NSString *jsonString = [[NSString alloc] initWithData:jsonData encoding:NSUTF8StringEncoding];
    NSString *jsCode = [NSString stringWithFormat:@"receiveDataFromNative(%@);", jsonString];
    
    [self.webView evaluateJavaScript:jsCode 
                   completionHandler:^(id result, NSError *error) {
        if (error) {
            NSLog(@"Error calling JavaScript function: %@", error.localizedDescription);
        }
    }];
}

// อัพเดตสถานะ login ในเว็บ
- (void)updateLoginState:(BOOL)isLoggedIn userInfo:(NSDictionary *)userInfo {
    NSDictionary *data = @{
        @"isLoggedIn": @(isLoggedIn),
        @"userInfo": userInfo ?: [NSNull null]
    };
    
    [self sendDataToJavaScript:data];
}

// ตัวอย่าง: บอกเว็บว่าซื้อสำเร็จแล้ว
- (void)notifyPurchaseComplete:(NSString *)productId {
    NSString *jsCode = [NSString stringWithFormat:
        @"if (typeof onPurchaseComplete === 'function') { onPurchaseComplete('%@'); }", 
        productId];
    
    [self.webView evaluateJavaScript:jsCode completionHandler:nil];
}
```

### JavaScript ที่รับข้อมูลจาก Native

```javascript
// รับข้อมูลจาก Native
function receiveDataFromNative(data) {
    console.log('Received from native:', data);
    
    if (data.isLoggedIn) {
        showUserInterface(data.userInfo);
    } else {
        showLoginForm();
    }
}

function onPurchaseComplete(productId) {
    // อัพเดต UI หลังจากซื้อสำเร็จ
    alert('ซื้อสำเร็จ! Product: ' + productId);
    unlockPremiumContent(productId);
}
```

---

## 69.8 การจัดการ Cookies

### การอ่าน Cookies

```objc
- (void)getAllCookies {
    WKHTTPCookieStore *cookieStore = self.webView.configuration.websiteDataStore.httpCookieStore;
    
    [cookieStore getAllCookies:^(NSArray<NSHTTPCookie *> *cookies) {
        NSLog(@"จำนวน cookies: %lu", (unsigned long)cookies.count);
        
        for (NSHTTPCookie *cookie in cookies) {
            NSLog(@"Cookie: name=%@, value=%@, domain=%@", 
                  cookie.name, cookie.value, cookie.domain);
        }
    }];
}
```

### การเพิ่ม Cookie

```objc
- (void)addCookieForDomain:(NSString *)domain {
    NSHTTPCookie *cookie = [NSHTTPCookie cookieWithProperties:@{
        NSHTTPCookieName: @"session_token",
        NSHTTPCookieValue: @"abc123xyz",
        NSHTTPCookieDomain: domain,
        NSHTTPCookiePath: @"/",
        NSHTTPCookieSecure: @YES,
        NSHTTPCookieExpires: [NSDate dateWithTimeIntervalSinceNow:3600] // 1 ชั่วโมง
    }];
    
    WKHTTPCookieStore *cookieStore = self.webView.configuration.websiteDataStore.httpCookieStore;
    
    [cookieStore setCookie:cookie completionHandler:^{
        NSLog(@"Cookie added successfully");
    }];
}
```

### การลบ Cookie

```objc
- (void)deleteCookiesForDomain:(NSString *)domain {
    WKHTTPCookieStore *cookieStore = self.webView.configuration.websiteDataStore.httpCookieStore;
    
    [cookieStore getAllCookies:^(NSArray<NSHTTPCookie *> *cookies) {
        for (NSHTTPCookie *cookie in cookies) {
            if ([cookie.domain containsString:domain]) {
                [cookieStore deleteCookie:cookie completionHandler:^{
                    NSLog(@"ลบ cookie แล้ว: %@", cookie.name);
                }];
            }
        }
    }];
}
```

### การล้างข้อมูลทั้งหมด

```objc
- (void)clearAllWebData {
    NSSet *dataTypes = [WKWebsiteDataStore allWebsiteDataTypes];
    NSDate *dateFrom = [NSDate dateWithTimeIntervalSince1970:0];
    
    [[WKWebsiteDataStore defaultDataStore] removeDataOfTypes:dataTypes
                                              modifiedSince:dateFrom
                                          completionHandler:^{
        NSLog(@"ล้างข้อมูลเว็บทั้งหมดแล้ว");
    }];
}

- (void)clearCookiesOnly {
    NSSet *cookieTypes = [NSSet setWithObject:WKWebsiteDataTypeCookies];
    NSDate *dateFrom = [NSDate dateWithTimeIntervalSince1970:0];
    
    [[WKWebsiteDataStore defaultDataStore] removeDataOfTypes:cookieTypes
                                              modifiedSince:dateFrom
                                          completionHandler:^{
        NSLog(@"ล้าง cookies ทั้งหมดแล้ว");
    }];
}
```

---

## 69.9 WKWebViewConfiguration

### การตั้งค่าขั้นสูง

```objc
- (WKWebViewConfiguration *)createAdvancedConfiguration {
    WKWebViewConfiguration *config = [[WKWebViewConfiguration alloc] init];
    
    // อนุญาตให้เล่น media แบบ inline (ไม่ต้อง fullscreen)
    config.allowsInlineMediaPlayback = YES;
    
    // ต้องการ user interaction ก่อนเล่น media
    config.mediaTypesRequiringUserActionForPlayback = WKAudiovisualMediaTypeNone;
    
    // ตั้งค่า WebView Preferences
    WKPreferences *preferences = [[WKPreferences alloc] init];
    preferences.javaScriptEnabled = YES;
    preferences.minimumFontSize = 12.0;
    
    // ใน iOS 14+ ใช้ WKWebpagePreferences แทน
    if (@available(iOS 14.0, *)) {
        WKWebpagePreferences *pagePreferences = [[WKWebpagePreferences alloc] init];
        pagePreferences.allowsContentJavaScript = YES;
        config.defaultWebpagePreferences = pagePreferences;
    }
    
    config.preferences = preferences;
    
    // ใช้ non-persistent data store (ไม่บันทึก cookies/cache)
    config.websiteDataStore = [WKWebsiteDataStore nonPersistentDataStore];
    
    // หรือใช้ persistent (default)
    // config.websiteDataStore = [WKWebsiteDataStore defaultDataStore];
    
    return config;
}
```

### การกำหนด Custom User Agent

```objc
- (void)setCustomUserAgent {
    // กำหนด User Agent ของทั้ง WKWebView
    self.webView.customUserAgent = @"MyApp/1.0 iOS WebKit";
    
    // หรือกำหนดใน configuration
    WKWebViewConfiguration *config = [[WKWebViewConfiguration alloc] init];
    // config ไม่มี customUserAgent โดยตรง ต้องกำหนดหลัง init
}

// ตรวจสอบ User Agent ปัจจุบัน
- (void)checkCurrentUserAgent {
    [self.webView evaluateJavaScript:@"navigator.userAgent" 
                   completionHandler:^(id result, NSError *error) {
        NSLog(@"User Agent: %@", result);
    }];
}
```

---

## 69.10 Content Blockers

Content Blockers ช่วยบล็อก content ที่ไม่ต้องการ เช่น โฆษณา หรือ trackers

### การสร้าง Content Blocker Rules

```objc
- (void)setupContentBlocker {
    // กำหนด rules ในรูปแบบ JSON
    NSArray *rules = @[
        @{
            @"trigger": @{
                @"url-filter": @".*\\.ads\\..*",
                @"resource-type": @[@"image", @"script", @"style-sheet"]
            },
            @"action": @{
                @"type": @"block"
            }
        },
        @{
            @"trigger": @{
                @"url-filter": @".*google-analytics.*"
            },
            @"action": @{
                @"type": @"block"
            }
        },
        @{
            @"trigger": @{
                @"url-filter": @".*"
            },
            @"action": @{
                @"type": @"css-display-none",
                @"selector": @".ad, .advertisement, #banner-ad"
            }
        }
    ];
    
    NSError *error;
    NSData *jsonData = [NSJSONSerialization dataWithJSONObject:rules 
                                                       options:0 
                                                         error:&error];
    NSString *jsonString = [[NSString alloc] initWithData:jsonData encoding:NSUTF8StringEncoding];
    
    [WKContentRuleListStore.defaultStore 
        compileContentRuleListForIdentifier:@"AdBlocker"
        encodedContentRuleList:jsonString
        completionHandler:^(WKContentRuleList *ruleList, NSError *compileError) {
            if (compileError) {
                NSLog(@"Error compiling rules: %@", compileError.localizedDescription);
                return;
            }
            
            [self.webView.configuration.userContentController addContentRuleList:ruleList];
            NSLog(@"Content blocker activated");
        }];
}
```

---

## 69.11 Progress Tracking

```objc
// ViewController.h
@property (nonatomic, strong) UIProgressView *progressView;

// ViewController.m
- (void)setupProgressTracking {
    // สร้าง Progress View
    self.progressView = [[UIProgressView alloc] initWithProgressViewStyle:UIProgressViewStyleDefault];
    self.progressView.frame = CGRectMake(0, 0, self.view.bounds.size.width, 2);
    [self.view addSubview:self.progressView];
    
    // Observe estimatedProgress
    [self.webView addObserver:self 
                   forKeyPath:@"estimatedProgress" 
                      options:NSKeyValueObservingOptionNew 
                      context:nil];
    
    // Observe loading state
    [self.webView addObserver:self 
                   forKeyPath:@"loading" 
                      options:NSKeyValueObservingOptionNew 
                      context:nil];
    
    // Observe title
    [self.webView addObserver:self 
                   forKeyPath:@"title" 
                      options:NSKeyValueObservingOptionNew 
                      context:nil];
}

- (void)observeValueForKeyPath:(NSString *)keyPath
                      ofObject:(id)object
                        change:(NSDictionary *)change
                       context:(void *)context {
    
    if ([keyPath isEqualToString:@"estimatedProgress"]) {
        float progress = self.webView.estimatedProgress;
        
        [self.progressView setProgress:progress animated:YES];
        
        if (progress >= 1.0) {
            // ซ่อน progress view เมื่อโหลดเสร็จ
            dispatch_after(dispatch_time(DISPATCH_TIME_NOW, 0.3 * NSEC_PER_SEC), 
                          dispatch_get_main_queue(), ^{
                self.progressView.hidden = YES;
                [self.progressView setProgress:0 animated:NO];
            });
        } else {
            self.progressView.hidden = NO;
        }
    }
    else if ([keyPath isEqualToString:@"loading"]) {
        // อัพเดต network activity indicator
        [UIApplication sharedApplication].networkActivityIndicatorVisible = self.webView.loading;
    }
    else if ([keyPath isEqualToString:@"title"]) {
        self.title = self.webView.title;
    }
}

- (void)dealloc {
    [self.webView removeObserver:self forKeyPath:@"estimatedProgress"];
    [self.webView removeObserver:self forKeyPath:@"loading"];
    [self.webView removeObserver:self forKeyPath:@"title"];
}
```

---

## 69.12 การสร้าง WebViewController แบบ Reusable

```objc
// WebViewController.h
#import <UIKit/UIKit.h>
#import <WebKit/WebKit.h>

@interface WebViewController : UIViewController

@property (nonatomic, strong) NSString *urlString;
@property (nonatomic, strong) NSString *htmlContent;
@property (nonatomic, assign) BOOL showsNavigationBar;
@property (nonatomic, strong) NSString *pageTitle;

// Factory methods
+ (instancetype)controllerWithURL:(NSString *)urlString;
+ (instancetype)controllerWithHTML:(NSString *)html title:(NSString *)title;

@end
```

```objc
// WebViewController.m
#import "WebViewController.h"

@interface WebViewController () <WKNavigationDelegate, WKUIDelegate>

@property (nonatomic, strong) WKWebView *webView;
@property (nonatomic, strong) UIProgressView *progressView;
@property (nonatomic, strong) UIBarButtonItem *backButton;
@property (nonatomic, strong) UIBarButtonItem *forwardButton;

@end

@implementation WebViewController

+ (instancetype)controllerWithURL:(NSString *)urlString {
    WebViewController *vc = [[WebViewController alloc] init];
    vc.urlString = urlString;
    return vc;
}

+ (instancetype)controllerWithHTML:(NSString *)html title:(NSString *)title {
    WebViewController *vc = [[WebViewController alloc] init];
    vc.htmlContent = html;
    vc.pageTitle = title;
    return vc;
}

- (void)viewDidLoad {
    [super viewDidLoad];
    
    self.view.backgroundColor = [UIColor systemBackgroundColor];
    
    [self setupWebView];
    [self setupProgressView];
    [self setupNavigationBar];
    [self loadContent];
}

- (void)setupWebView {
    WKWebViewConfiguration *config = [[WKWebViewConfiguration alloc] init];
    config.allowsInlineMediaPlayback = YES;
    
    self.webView = [[WKWebView alloc] initWithFrame:CGRectZero configuration:config];
    self.webView.translatesAutoresizingMaskIntoConstraints = NO;
    self.webView.navigationDelegate = self;
    self.webView.UIDelegate = self;
    
    [self.view addSubview:self.webView];
    
    [NSLayoutConstraint activateConstraints:@[
        [self.webView.topAnchor constraintEqualToAnchor:self.view.safeAreaLayoutGuide.topAnchor],
        [self.webView.leadingAnchor constraintEqualToAnchor:self.view.leadingAnchor],
        [self.webView.trailingAnchor constraintEqualToAnchor:self.view.trailingAnchor],
        [self.webView.bottomAnchor constraintEqualToAnchor:self.view.safeAreaLayoutGuide.bottomAnchor]
    ]];
    
    [self.webView addObserver:self forKeyPath:@"estimatedProgress" options:NSKeyValueObservingOptionNew context:nil];
    [self.webView addObserver:self forKeyPath:@"title" options:NSKeyValueObservingOptionNew context:nil];
}

- (void)setupProgressView {
    self.progressView = [[UIProgressView alloc] initWithProgressViewStyle:UIProgressViewStyleBar];
    self.progressView.translatesAutoresizingMaskIntoConstraints = NO;
    self.progressView.tintColor = [UIColor systemBlueColor];
    [self.view addSubview:self.progressView];
    [self.view bringSubviewToFront:self.progressView];
    
    [NSLayoutConstraint activateConstraints:@[
        [self.progressView.topAnchor constraintEqualToAnchor:self.view.safeAreaLayoutGuide.topAnchor],
        [self.progressView.leadingAnchor constraintEqualToAnchor:self.view.leadingAnchor],
        [self.progressView.trailingAnchor constraintEqualToAnchor:self.view.trailingAnchor],
        [self.progressView.heightAnchor constraintEqualToConstant:2]
    ]];
}

- (void)setupNavigationBar {
    if (self.pageTitle) {
        self.title = self.pageTitle;
    }
    
    self.backButton = [[UIBarButtonItem alloc] 
        initWithImage:[UIImage systemImageNamed:@"chevron.left"]
        style:UIBarButtonItemStylePlain
        target:self
        action:@selector(goBack)];
    
    self.forwardButton = [[UIBarButtonItem alloc] 
        initWithImage:[UIImage systemImageNamed:@"chevron.right"]
        style:UIBarButtonItemStylePlain
        target:self
        action:@selector(goForward)];
    
    UIBarButtonItem *closeButton = [[UIBarButtonItem alloc] 
        initWithBarButtonSystemItem:UIBarButtonSystemItemStop
        target:self
        action:@selector(closePage)];
    
    UIBarButtonItem *refreshButton = [[UIBarButtonItem alloc] 
        initWithBarButtonSystemItem:UIBarButtonSystemItemRefresh
        target:self
        action:@selector(reloadPage)];
    
    self.navigationItem.leftBarButtonItems = @[closeButton, self.backButton, self.forwardButton];
    self.navigationItem.rightBarButtonItem = refreshButton;
    
    [self updateNavigationButtons];
}

- (void)loadContent {
    if (self.urlString) {
        NSURL *url = [NSURL URLWithString:self.urlString];
        [self.webView loadRequest:[NSURLRequest requestWithURL:url]];
    } else if (self.htmlContent) {
        NSURL *baseURL = [NSURL fileURLWithPath:[[NSBundle mainBundle] bundlePath]];
        [self.webView loadHTMLString:self.htmlContent baseURL:baseURL];
    }
}

- (void)goBack { [self.webView goBack]; }
- (void)goForward { [self.webView goForward]; }
- (void)reloadPage { [self.webView reload]; }
- (void)closePage { [self dismissViewControllerAnimated:YES completion:nil]; }

- (void)updateNavigationButtons {
    self.backButton.enabled = self.webView.canGoBack;
    self.forwardButton.enabled = self.webView.canGoForward;
}

- (void)observeValueForKeyPath:(NSString *)keyPath ofObject:(id)object change:(NSDictionary *)change context:(void *)context {
    if ([keyPath isEqualToString:@"estimatedProgress"]) {
        double progress = self.webView.estimatedProgress;
        self.progressView.hidden = (progress == 1.0);
        [self.progressView setProgress:progress animated:YES];
    } else if ([keyPath isEqualToString:@"title"] && !self.pageTitle) {
        self.title = self.webView.title;
    }
}

- (void)webView:(WKWebView *)webView didFinishNavigation:(WKNavigation *)navigation {
    [self updateNavigationButtons];
    self.progressView.hidden = YES;
}

- (void)dealloc {
    [self.webView removeObserver:self forKeyPath:@"estimatedProgress"];
    [self.webView removeObserver:self forKeyPath:@"title"];
}

@end
```

---

## 69.13 แบบฝึกหัด

### แบบฝึกหัดที่ 1: Simple Browser

สร้าง Simple Browser ที่มีฟีเจอร์:
- Address bar สำหรับพิมพ์ URL
- ปุ่ม Back, Forward, Refresh
- แสดง progress bar
- รองรับ pull-to-refresh
- แสดง page title

```objc
// Simple Browser แบบสมบูรณ์
@interface SimpleBrowserViewController : UIViewController <WKNavigationDelegate, UITextFieldDelegate>

@property (nonatomic, strong) WKWebView *webView;
@property (nonatomic, strong) UITextField *addressBar;
@property (nonatomic, strong) UIProgressView *progressView;

@end

@implementation SimpleBrowserViewController

- (void)viewDidLoad {
    [super viewDidLoad];
    
    [self setupUI];
    [self loadURL:@"https://www.apple.com"];
}

- (void)setupUI {
    // Navigation Bar
    self.navigationItem.title = @"Browser";
    
    // Address Bar
    self.addressBar = [[UITextField alloc] init];
    self.addressBar.borderStyle = UITextBorderStyleRoundedRect;
    self.addressBar.placeholder = @"ป้อน URL...";
    self.addressBar.returnKeyType = UIReturnKeyGo;
    self.addressBar.delegate = self;
    self.addressBar.keyboardType = UIKeyboardTypeURL;
    self.addressBar.autocapitalizationType = UITextAutocapitalizationTypeNone;
    self.addressBar.autocorrectionType = UITextAutocorrectionTypeNo;
    self.navigationItem.titleView = self.addressBar;
    
    // WebView
    self.webView = [[WKWebView alloc] initWithFrame:self.view.bounds];
    self.webView.navigationDelegate = self;
    self.webView.autoresizingMask = UIViewAutoresizingFlexibleWidth | UIViewAutoresizingFlexibleHeight;
    [self.view addSubview:self.webView];
    
    // Progress View
    self.progressView = [[UIProgressView alloc] initWithProgressViewStyle:UIProgressViewStyleDefault];
    [self.view addSubview:self.progressView];
    
    // Pull to Refresh
    UIRefreshControl *refreshControl = [[UIRefreshControl alloc] init];
    [refreshControl addTarget:self action:@selector(refresh:) forControlEvents:UIControlEventValueChanged];
    self.webView.scrollView.refreshControl = refreshControl;
    
    // KVO for progress
    [self.webView addObserver:self forKeyPath:@"estimatedProgress" options:NSKeyValueObservingOptionNew context:nil];
    [self.webView addObserver:self forKeyPath:@"title" options:NSKeyValueObservingOptionNew context:nil];
}

- (void)loadURL:(NSString *)urlString {
    if (![urlString hasPrefix:@"http://"] && ![urlString hasPrefix:@"https://"]) {
        urlString = [@"https://" stringByAppendingString:urlString];
    }
    
    NSURL *url = [NSURL URLWithString:urlString];
    if (url) {
        [self.webView loadRequest:[NSURLRequest requestWithURL:url]];
        self.addressBar.text = urlString;
    }
}

- (BOOL)textFieldShouldReturn:(UITextField *)textField {
    [textField resignFirstResponder];
    [self loadURL:textField.text];
    return YES;
}

- (void)refresh:(UIRefreshControl *)sender {
    [self.webView reload];
    [sender endRefreshing];
}

- (void)observeValueForKeyPath:(NSString *)keyPath ofObject:(id)object change:(NSDictionary *)change context:(void *)context {
    if ([keyPath isEqualToString:@"estimatedProgress"]) {
        double progress = self.webView.estimatedProgress;
        [self.progressView setProgress:progress animated:YES];
        self.progressView.hidden = (progress == 1.0);
    } else if ([keyPath isEqualToString:@"title"]) {
        self.navigationItem.title = self.webView.title;
    }
}

- (void)webView:(WKWebView *)webView didFinishNavigation:(WKNavigation *)navigation {
    self.addressBar.text = webView.URL.absoluteString;
    [self.webView.scrollView.refreshControl endRefreshing];
}

@end
```

### แบบฝึกหัดที่ 2: Hybrid App Communication

สร้างระบบการสื่อสารระหว่าง JavaScript และ Native:

```objc
// HybridBridge.h - สร้าง Bridge Class
@interface HybridBridge : NSObject <WKScriptMessageHandler>

@property (nonatomic, weak) UIViewController *viewController;
@property (nonatomic, weak) WKWebView *webView;

+ (instancetype)bridgeWithViewController:(UIViewController *)vc webView:(WKWebView *)webView;
- (void)registerHandlers:(WKUserContentController *)userContentController;
- (void)callJavaScriptFunction:(NSString *)functionName withArgs:(NSArray *)args;

@end

@implementation HybridBridge

+ (instancetype)bridgeWithViewController:(UIViewController *)vc webView:(WKWebView *)webView {
    HybridBridge *bridge = [[HybridBridge alloc] init];
    bridge.viewController = vc;
    bridge.webView = webView;
    return bridge;
}

- (void)registerHandlers:(WKUserContentController *)ucc {
    [ucc addScriptMessageHandler:self name:@"nativeBridge"];
}

- (void)userContentController:(WKUserContentController *)ucc
      didReceiveScriptMessage:(WKScriptMessage *)message {
    
    if (![message.body isKindOfClass:[NSDictionary class]]) return;
    
    NSDictionary *payload = message.body;
    NSString *command = payload[@"command"];
    NSDictionary *params = payload[@"params"];
    NSString *callbackId = payload[@"callbackId"];
    
    [self handleCommand:command params:params callbackId:callbackId];
}

- (void)handleCommand:(NSString *)command params:(NSDictionary *)params callbackId:(NSString *)callbackId {
    if ([command isEqualToString:@"getDeviceInfo"]) {
        NSDictionary *deviceInfo = @{
            @"model": [[UIDevice currentDevice] model],
            @"systemVersion": [[UIDevice currentDevice] systemVersion],
            @"identifier": [[[UIDevice currentDevice] identifierForVendor] UUIDString]
        };
        [self sendCallback:callbackId result:deviceInfo error:nil];
    }
    else if ([command isEqualToString:@"vibrate"]) {
        AudioServicesPlaySystemSound(kSystemSoundID_Vibrate);
        [self sendCallback:callbackId result:@{@"success": @YES} error:nil];
    }
    else if ([command isEqualToString:@"showToast"]) {
        // แสดง Toast message
        NSString *msg = params[@"message"];
        NSLog(@"Toast: %@", msg);
        [self sendCallback:callbackId result:@{@"shown": @YES} error:nil];
    }
}

- (void)sendCallback:(NSString *)callbackId result:(id)result error:(NSError *)error {
    NSDictionary *response;
    if (error) {
        response = @{@"callbackId": callbackId, @"error": error.localizedDescription};
    } else {
        response = @{@"callbackId": callbackId, @"result": result};
    }
    
    NSData *jsonData = [NSJSONSerialization dataWithJSONObject:response options:0 error:nil];
    NSString *jsonString = [[NSString alloc] initWithData:jsonData encoding:NSUTF8StringEncoding];
    NSString *jsCode = [NSString stringWithFormat:@"NativeBridge._handleCallback(%@);", jsonString];
    
    [self.webView evaluateJavaScript:jsCode completionHandler:nil];
}

- (void)callJavaScriptFunction:(NSString *)functionName withArgs:(NSArray *)args {
    NSData *argsData = [NSJSONSerialization dataWithJSONObject:args options:0 error:nil];
    NSString *argsString = [[NSString alloc] initWithData:argsData encoding:NSUTF8StringEncoding];
    NSString *jsCode = [NSString stringWithFormat:@"%@.apply(null, %@);", functionName, argsString];
    [self.webView evaluateJavaScript:jsCode completionHandler:nil];
}

@end
```

### แบบฝึกหัดที่ 3: Content Filter

สร้างระบบกรองเนื้อหาที่:
- บล็อก ads
- กรอง tracking scripts
- เพิ่ม reading mode CSS

```objc
- (void)enableReadingMode {
    NSString *readingModeCSS = @"body {"
        @"    max-width: 680px;"
        @"    margin: 0 auto;"
        @"    padding: 20px;"
        @"    font-family: Georgia, serif;"
        @"    font-size: 18px;"
        @"    line-height: 1.8;"
        @"    color: #333;"
        @"    background: #fff;"
        @"}"
        @"img { max-width: 100%; height: auto; }"
        @".ad, .sidebar, nav, footer, header { display: none !important; }";
    
    NSString *escapedCSS = [readingModeCSS stringByReplacingOccurrencesOfString:@"\n" withString:@""];
    NSString *jsCode = [NSString stringWithFormat:
        @"(function() {"
        @"    var existing = document.getElementById('reading-mode-style');"
        @"    if (existing) existing.remove();"
        @"    var style = document.createElement('style');"
        @"    style.id = 'reading-mode-style';"
        @"    style.textContent = '%@';"
        @"    document.head.appendChild(style);"
        @"})();",
        escapedCSS];
    
    [self.webView evaluateJavaScript:jsCode completionHandler:^(id result, NSError *error) {
        NSLog(@"Reading mode: %@", error ? error.localizedDescription : @"enabled");
    }];
}
```

---

## สรุป

ในบทนี้เราได้เรียนรู้การใช้งาน WebKit Framework ใน Objective-C:

1. **WKWebView Setup** - การตั้งค่าและใช้งาน WKWebView ด้วย Auto Layout
2. **Loading Content** - การโหลด URL, HTML String, ไฟล์ local และ Data
3. **WKNavigationDelegate** - การติดตามสถานะการโหลดและควบคุม navigation
4. **WKUIDelegate** - การจัดการ JavaScript dialogs
5. **JavaScript Injection** - การรัน JavaScript จาก Native และ inject scripts
6. **WKScriptMessageHandler** - การรับข้อความจาก JavaScript
7. **Native to JS Communication** - การส่งข้อมูลจาก Native ไป JavaScript
8. **Cookies Management** - การอ่าน เพิ่ม และลบ cookies
9. **WKWebViewConfiguration** - การตั้งค่าขั้นสูง
10. **Content Blockers** - การบล็อก content ที่ไม่ต้องการ
11. **Progress Tracking** - การแสดง progress ด้วย KVO

WebKit เป็นเครื่องมือที่ทรงพลังมากสำหรับการสร้าง Hybrid Apps หรือแสดงเนื้อหาเว็บในแอปพลิเคชัน iOS
