# Part 85: macOS Development ด้วย Objective-C

## บทนำ

การพัฒนาแอปพลิเคชันสำหรับ macOS เป็นหัวข้อที่น่าสนใจและมีความแตกต่างจากการพัฒนาสำหรับ iOS อย่างมีนัยสำคัญ ในบทนี้เราจะเรียนรู้เกี่ยวกับ AppKit framework ซึ่งเป็น UI framework หลักของ macOS รวมถึงคอมโพเนนต์ต่างๆ ที่ใช้ในการสร้างแอปพลิเคชัน macOS ที่สมบูรณ์แบบ

---

## 85.1 AppKit vs UIKit - ความแตกต่างที่สำคัญ

### ภาพรวมของความแตกต่าง

AppKit และ UIKit เป็น framework สองตัวที่ใช้สำหรับพัฒนา UI บนแพลตฟอร์มต่างกัน:

| คุณสมบัติ | AppKit (macOS) | UIKit (iOS/iPadOS) |
|----------|---------------|-------------------|
| แพลตฟอร์ม | macOS | iOS, iPadOS, tvOS |
| View class | NSView | UIView |
| Window class | NSWindow | UIWindow |
| Color class | NSColor | UIColor |
| Image class | NSImage | UIImage |
| Font class | NSFont | UIFont |
| Event | NSEvent | UIEvent |
| Coordinate system | Lower-left origin | Upper-left origin |
| Mouse/Touch | Mouse + Trackpad | Touch |

### Coordinate System ที่แตกต่างกัน

```objc
// macOS (AppKit) - origin อยู่ที่มุมล่างซ้าย
// Y เพิ่มขึ้นจากล่างขึ้นบน
NSRect rect = NSMakeRect(0, 0, 100, 100);  // x, y, width, height

// iOS (UIKit) - origin อยู่ที่มุมบนซ้าย
// Y เพิ่มขึ้นจากบนลงล่าง
CGRect rect = CGRectMake(0, 0, 100, 100);  // x, y, width, height
```

### การ Setup Application

```objc
// macOS - main.m
#import <Cocoa/Cocoa.h>

int main(int argc, const char * argv[]) {
    return NSApplicationMain(argc, argv);
}

// AppDelegate.h
#import <Cocoa/Cocoa.h>

@interface AppDelegate : NSObject <NSApplicationDelegate>
@property (strong) NSWindow *window;
@end

// AppDelegate.m
#import "AppDelegate.h"

@implementation AppDelegate

- (void)applicationDidFinishLaunching:(NSNotification *)aNotification {
    // ตั้งค่าแอปพลิเคชันหลังจากเปิดขึ้นมา
    NSLog(@"macOS App launched!");
}

- (void)applicationWillTerminate:(NSNotification *)aNotification {
    // ทำงานก่อนแอปพลิเคชันปิด
}

- (BOOL)applicationShouldTerminateAfterLastWindowClosed:(NSApplication *)sender {
    return YES;  // ปิดแอปเมื่อปิด window สุดท้าย
}

@end
```

### NSResponder Chain

```objc
// macOS ใช้ Responder Chain ในการจัดการ events
// NSApplication -> NSWindow -> NSViewController -> NSView

@interface MyView : NSView

@end

@implementation MyView

// รับ mouse events
- (void)mouseDown:(NSEvent *)event {
    NSPoint location = [self convertPoint:event.locationInWindow fromView:nil];
    NSLog(@"Mouse down at: %.1f, %.1f", location.x, location.y);
}

- (void)mouseMoved:(NSEvent *)event {
    // ต้องเรียก setAcceptsMouseMovedEvents: บน window ก่อน
    NSPoint location = [self convertPoint:event.locationInWindow fromView:nil];
    NSLog(@"Mouse moved to: %.1f, %.1f", location.x, location.y);
}

// รับ keyboard events
- (void)keyDown:(NSEvent *)event {
    NSLog(@"Key pressed: %@", event.charactersIgnoringModifiers);
    [super keyDown:event];  // ส่งต่อไปยัง responder chain
}

// ต้องกำหนดให้ view รับ first responder ได้
- (BOOL)acceptsFirstResponder {
    return YES;
}

@end
```

---

## 85.2 NSWindow - Window Management

### การสร้างและจัดการ Window

```objc
// การสร้าง Window แบบ programmatic
NSWindow *window = [[NSWindow alloc] 
    initWithContentRect:NSMakeRect(100, 100, 800, 600)
              styleMask:NSWindowStyleMaskTitled | 
                        NSWindowStyleMaskClosable | 
                        NSWindowStyleMaskMiniaturizable |
                        NSWindowStyleMaskResizable
                backing:NSBackingStoreBuffered
                  defer:NO];

[window setTitle:@"My macOS App"];
[window makeKeyAndOrderFront:nil];  // แสดง window
```

### NSWindowStyleMask Options

```objc
// Style mask options ที่ใช้บ่อย
NSWindowStyleMaskBorderless           // ไม่มีขอบ window
NSWindowStyleMaskTitled               // มี title bar
NSWindowStyleMaskClosable             // มีปุ่มปิด
NSWindowStyleMaskMiniaturizable       // มีปุ่มย่อเล็ก
NSWindowStyleMaskResizable            // ปรับขนาดได้
NSWindowStyleMaskFullScreen           // เต็มจอ
NSWindowStyleMaskUnifiedTitleAndToolbar // รวม title กับ toolbar
NSWindowStyleMaskFullSizeContentView  // content view ใต้ title bar
```

### Window Delegate

```objc
@interface WindowController : NSWindowController <NSWindowDelegate>

@end

@implementation WindowController

- (void)windowDidLoad {
    [super windowDidLoad];
    self.window.delegate = self;
    self.window.title = @"My Window";
}

// NSWindowDelegate methods
- (void)windowDidBecomeKey:(NSNotification *)notification {
    NSLog(@"Window became key (active)");
}

- (void)windowDidResignKey:(NSNotification *)notification {
    NSLog(@"Window resigned key");
}

- (void)windowDidResize:(NSNotification *)notification {
    NSSize newSize = self.window.frame.size;
    NSLog(@"Window resized to: %.0f x %.0f", newSize.width, newSize.height);
}

- (BOOL)windowShouldClose:(NSWindow *)sender {
    // ถามผู้ใช้ก่อนปิด window
    NSAlert *alert = [[NSAlert alloc] init];
    alert.messageText = @"ต้องการปิดหรือไม่?";
    alert.informativeText = @"การเปลี่ยนแปลงที่ยังไม่ได้บันทึกจะหายไป";
    [alert addButtonWithTitle:@"ปิด"];
    [alert addButtonWithTitle:@"ยกเลิก"];
    
    NSModalResponse response = [alert runModal];
    return response == NSAlertFirstButtonReturn;
}

- (NSSize)windowWillResize:(NSWindow *)sender toSize:(NSSize)frameSize {
    // จำกัดขนาด window ขั้นต่ำ
    frameSize.width = MAX(frameSize.width, 400);
    frameSize.height = MAX(frameSize.height, 300);
    return frameSize;
}

@end
```

### การใช้ NSWindowController

```objc
// WindowController.h
@interface MyWindowController : NSWindowController

- (instancetype)init;

@end

// WindowController.m
@implementation MyWindowController

- (instancetype)init {
    // โหลดจาก NIB/Storyboard
    self = [super initWithWindowNibName:@"MyWindow"];
    return self;
}

// หรือสร้างแบบ programmatic
- (instancetype)initProgrammatic {
    NSWindow *window = [[NSWindow alloc] 
        initWithContentRect:NSMakeRect(0, 0, 800, 600)
                  styleMask:NSWindowStyleMaskTitled | NSWindowStyleMaskClosable
                    backing:NSBackingStoreBuffered
                      defer:NO];
    
    self = [super initWithWindow:window];
    if (self) {
        [window center];
        // ตั้งค่า content view
        window.contentViewController = [[MainViewController alloc] init];
    }
    return self;
}

@end
```

---

## 85.3 NSViewController

### การสร้าง ViewController

```objc
// ViewController.h
#import <Cocoa/Cocoa.h>

@interface MainViewController : NSViewController

@end

// ViewController.m
#import "MainViewController.h"

@implementation MainViewController

- (void)viewDidLoad {
    [super viewDidLoad];
    
    // ตั้งค่า view หลังจาก load
    self.view.wantsLayer = YES;
    self.view.layer.backgroundColor = [NSColor whiteColor].CGColor;
    
    [self setupUI];
}

- (void)setupUI {
    // สร้าง label
    NSTextField *label = [NSTextField labelWithString:@"Hello, macOS!"];
    label.font = [NSFont systemFontOfSize:24 weight:NSFontWeightBold];
    label.translatesAutoresizingMaskIntoConstraints = NO;
    [self.view addSubview:label];
    
    // สร้าง button
    NSButton *button = [NSButton buttonWithTitle:@"คลิกที่นี่" 
                                          target:self 
                                          action:@selector(buttonClicked:)];
    button.translatesAutoresizingMaskIntoConstraints = NO;
    [self.view addSubview:button];
    
    // Auto Layout Constraints
    [NSLayoutConstraint activateConstraints:@[
        [label.centerXAnchor constraintEqualToAnchor:self.view.centerXAnchor],
        [label.centerYAnchor constraintEqualToAnchor:self.view.centerYAnchor constant:-30],
        [button.centerXAnchor constraintEqualToAnchor:self.view.centerXAnchor],
        [button.topAnchor constraintEqualToAnchor:label.bottomAnchor constant:20],
    ]];
}

- (void)buttonClicked:(NSButton *)sender {
    NSAlert *alert = [[NSAlert alloc] init];
    alert.messageText = @"สวัสดี!";
    alert.informativeText = @"คุณได้คลิกปุ่มแล้ว";
    [alert runModal];
}

// Lifecycle methods
- (void)viewWillAppear {
    [super viewWillAppear];
    NSLog(@"View will appear");
}

- (void)viewDidAppear {
    [super viewDidAppear];
    NSLog(@"View did appear");
}

- (void)viewWillDisappear {
    [super viewWillDisappear];
    NSLog(@"View will disappear");
}

@end
```

### Container View Controllers

```objc
// การเพิ่ม child view controller
- (void)addChildViewController {
    ChildViewController *child = [[ChildViewController alloc] init];
    
    [self addChildViewController:child];
    child.view.frame = CGRectMake(0, 0, 200, 200);
    [self.view addSubview:child.view];
    
    // แจ้ง child ว่า parent เพิ่มแล้ว
    [child viewDidMoveToParentViewController:self];
}

// การลบ child view controller
- (void)removeChildViewController:(NSViewController *)child {
    [child viewWillMoveToParentViewController:nil];
    [child.view removeFromSuperview];
    [child removeFromParentViewController];
}
```

---

## 85.4 NSView และการ Drawing

### การสร้าง Custom View

```objc
// CustomView.h
@interface CustomView : NSView

@property (nonatomic, strong) NSColor *fillColor;
@property (nonatomic, assign) CGFloat borderWidth;

@end

// CustomView.m
@implementation CustomView

- (instancetype)initWithFrame:(NSRect)frameRect {
    self = [super initWithFrame:frameRect];
    if (self) {
        _fillColor = [NSColor blueColor];
        _borderWidth = 2.0;
    }
    return self;
}

// drawRect: เป็น method หลักในการ draw
- (void)drawRect:(NSRect)dirtyRect {
    [super drawRect:dirtyRect];
    
    // วาดพื้นหลัง
    [self.fillColor setFill];
    NSRectFill(dirtyRect);
    
    // วาดขอบ
    [[NSColor blackColor] setStroke];
    NSBezierPath *border = [NSBezierPath bezierPathWithRect:self.bounds];
    border.lineWidth = self.borderWidth;
    [border stroke];
    
    // วาดข้อความ
    NSString *text = @"Custom View";
    NSDictionary *attrs = @{
        NSFontAttributeName: [NSFont systemFontOfSize:16],
        NSForegroundColorAttributeName: [NSColor whiteColor]
    };
    NSSize textSize = [text sizeWithAttributes:attrs];
    NSPoint textPoint = NSMakePoint(
        (self.bounds.size.width - textSize.width) / 2,
        (self.bounds.size.height - textSize.height) / 2
    );
    [text drawAtPoint:textPoint withAttributes:attrs];
}

// ทริกเกอร์ให้ view redraw เมื่อ property เปลี่ยน
- (void)setFillColor:(NSColor *)fillColor {
    _fillColor = fillColor;
    [self setNeedsDisplay:YES];  // เหมือน setNeedsDisplay ใน UIKit
}

@end
```

### การวาดรูปทรงต่างๆ ด้วย NSBezierPath

```objc
- (void)drawRect:(NSRect)dirtyRect {
    [super drawRect:dirtyRect];
    
    // วาดวงกลม
    NSBezierPath *circle = [NSBezierPath bezierPathWithOvalInRect:
        NSMakeRect(50, 50, 100, 100)];
    [[NSColor redColor] setFill];
    [circle fill];
    
    // วาดสี่เหลี่ยมมุมโค้ง
    NSBezierPath *roundedRect = [NSBezierPath bezierPathWithRoundedRect:
        NSMakeRect(200, 50, 150, 100) 
        xRadius:10 
        yRadius:10];
    [[NSColor greenColor] setFill];
    [roundedRect fill];
    
    // วาดเส้น
    NSBezierPath *line = [NSBezierPath bezierPath];
    [line moveToPoint:NSMakePoint(50, 200)];
    [line lineToPoint:NSMakePoint(350, 200)];
    [line setLineWidth:3.0];
    [[NSColor blueColor] setStroke];
    [line stroke];
    
    // วาด path ซับซ้อน
    NSBezierPath *path = [NSBezierPath bezierPath];
    [path moveToPoint:NSMakePoint(100, 300)];
    [path lineToPoint:NSMakePoint(200, 400)];
    [path lineToPoint:NSMakePoint(300, 300)];
    [path curveToPoint:NSMakePoint(200, 250)
         controlPoint1:NSMakePoint(280, 280)
         controlPoint2:NSMakePoint(180, 260)];
    [path closePath];
    
    [[NSColor purpleColor] setFill];
    [path fill];
    [[NSColor blackColor] setStroke];
    [path stroke];
    
    // วาด gradient
    NSGradient *gradient = [[NSGradient alloc] initWithColors:@[
        [NSColor blueColor],
        [NSColor cyanColor]
    ]];
    [gradient drawInRect:NSMakeRect(50, 350, 200, 100) angle:45.0];
}
```

### Layer-backed Views

```objc
@implementation AnimatedView

- (instancetype)initWithFrame:(NSRect)frameRect {
    self = [super initWithFrame:frameRect];
    if (self) {
        // เปิดใช้งาน Core Animation layer
        self.wantsLayer = YES;
        
        // ตั้งค่า layer properties
        self.layer.cornerRadius = 10;
        self.layer.backgroundColor = [NSColor blueColor].CGColor;
        self.layer.shadowOpacity = 0.5;
        self.layer.shadowRadius = 5.0;
        self.layer.shadowOffset = CGSizeMake(2, -2);
    }
    return self;
}

// Animation ด้วย Core Animation
- (void)startAnimation {
    CABasicAnimation *animation = [CABasicAnimation animationWithKeyPath:@"opacity"];
    animation.fromValue = @1.0;
    animation.toValue = @0.3;
    animation.duration = 1.0;
    animation.autoreverses = YES;
    animation.repeatCount = HUGE_VALF;
    
    [self.layer addAnimation:animation forKey:@"pulseAnimation"];
}

// NSView animation (เหมือน UIView.animate)
- (void)animateWithNSAnimator {
    [NSAnimationContext runAnimationGroup:^(NSAnimationContext *context) {
        context.duration = 0.5;
        context.timingFunction = [CAMediaTimingFunction 
            functionWithName:kCAMediaTimingFunctionEaseInEaseOut];
        
        // ใช้ animator() proxy
        self.animator.frame = NSMakeRect(100, 100, 200, 200);
        self.animator.alphaValue = 0.5;
    } completionHandler:^{
        NSLog(@"Animation complete");
    }];
}

@end
```

---

## 85.5 Menu Bars - NSMenu และ NSMenuItem

### การสร้าง Application Menu

```objc
// MenuManager.m
@implementation MenuManager

+ (void)setupMainMenu {
    NSMenu *mainMenu = [[NSMenu alloc] initWithTitle:@"MainMenu"];
    [NSApp setMainMenu:mainMenu];
    
    // App Menu (ชื่อแอป)
    NSMenuItem *appMenuItem = [[NSMenuItem alloc] init];
    [mainMenu addItem:appMenuItem];
    
    NSMenu *appMenu = [[NSMenu alloc] initWithTitle:@"MyApp"];
    appMenuItem.submenu = appMenu;
    
    // About
    [appMenu addItemWithTitle:@"About MyApp"
                       action:@selector(orderFrontStandardAboutPanel:)
                keyEquivalent:@""];
    
    [appMenu addItem:[NSMenuItem separatorItem]];
    
    // Preferences
    NSMenuItem *prefsItem = [appMenu addItemWithTitle:@"Preferences..."
                                               action:@selector(openPreferences:)
                                        keyEquivalent:@","];
    prefsItem.target = [NSApp delegate];
    
    [appMenu addItem:[NSMenuItem separatorItem]];
    
    // Quit
    [appMenu addItemWithTitle:@"Quit MyApp"
                       action:@selector(terminate:)
                keyEquivalent:@"q"];
    
    // File Menu
    NSMenuItem *fileMenuItem = [[NSMenuItem alloc] init];
    [mainMenu addItem:fileMenuItem];
    
    NSMenu *fileMenu = [[NSMenu alloc] initWithTitle:@"File"];
    fileMenuItem.submenu = fileMenu;
    
    [fileMenu addItemWithTitle:@"New" 
                        action:@selector(newDocument:) 
                 keyEquivalent:@"n"];
    [fileMenu addItemWithTitle:@"Open..." 
                        action:@selector(openDocument:) 
                 keyEquivalent:@"o"];
    [fileMenu addItem:[NSMenuItem separatorItem]];
    [fileMenu addItemWithTitle:@"Save" 
                        action:@selector(saveDocument:) 
                 keyEquivalent:@"s"];
    
    // Edit Menu
    NSMenuItem *editMenuItem = [[NSMenuItem alloc] init];
    [mainMenu addItem:editMenuItem];
    
    NSMenu *editMenu = [[NSMenu alloc] initWithTitle:@"Edit"];
    editMenuItem.submenu = editMenu;
    
    [editMenu addItemWithTitle:@"Undo" 
                        action:@selector(undo:) 
                 keyEquivalent:@"z"];
    [editMenu addItemWithTitle:@"Redo" 
                        action:@selector(redo:) 
                 keyEquivalent:@"Z"];  // Shift+Cmd+Z
    [editMenu addItem:[NSMenuItem separatorItem]];
    [editMenu addItemWithTitle:@"Cut" 
                        action:@selector(cut:) 
                 keyEquivalent:@"x"];
    [editMenu addItemWithTitle:@"Copy" 
                        action:@selector(copy:) 
                 keyEquivalent:@"c"];
    [editMenu addItemWithTitle:@"Paste" 
                        action:@selector(paste:) 
                 keyEquivalent:@"v"];
    [editMenu addItemWithTitle:@"Select All" 
                        action:@selector(selectAll:) 
                 keyEquivalent:@"a"];
}

@end
```

### Context Menu (Right-click Menu)

```objc
@implementation MyView

- (void)rightMouseDown:(NSEvent *)event {
    NSMenu *contextMenu = [self buildContextMenu];
    [NSMenu popUpContextMenu:contextMenu withEvent:event forView:self];
}

- (NSMenu *)buildContextMenu {
    NSMenu *menu = [[NSMenu alloc] initWithTitle:@"Context Menu"];
    
    NSMenuItem *item1 = [[NSMenuItem alloc] initWithTitle:@"ตัดเลือก" 
                                                    action:@selector(cut:) 
                                             keyEquivalent:@""];
    [menu addItem:item1];
    
    NSMenuItem *item2 = [[NSMenuItem alloc] initWithTitle:@"คัดลอก" 
                                                    action:@selector(copy:) 
                                             keyEquivalent:@""];
    [menu addItem:item2];
    
    NSMenuItem *item3 = [[NSMenuItem alloc] initWithTitle:@"วาง" 
                                                    action:@selector(paste:) 
                                             keyEquivalent:@""];
    [menu addItem:item3];
    
    [menu addItem:[NSMenuItem separatorItem]];
    
    NSMenuItem *customItem = [[NSMenuItem alloc] initWithTitle:@"ดำเนินการพิเศษ" 
                                                         action:@selector(specialAction:) 
                                                  keyEquivalent:@""];
    customItem.target = self;
    [menu addItem:customItem];
    
    return menu;
}

// NSMenuDelegate สำหรับ dynamic menus
- (void)menuNeedsUpdate:(NSMenu *)menu {
    // อัปเดต menu items ก่อนแสดง
    [menu removeAllItems];
    
    // เพิ่ม items ตาม state ปัจจุบัน
    if ([self hasSelection]) {
        [menu addItemWithTitle:@"คัดลอกที่เลือก" 
                        action:@selector(copySelection:) 
                 keyEquivalent:@""];
    }
}

@end
```

### MenuItem Validation

```objc
// การ validate menu items (เปิด/ปิดตาม state)
- (BOOL)validateMenuItem:(NSMenuItem *)menuItem {
    if (menuItem.action == @selector(save:)) {
        return self.hasUnsavedChanges;
    }
    if (menuItem.action == @selector(copy:)) {
        return self.hasSelection;
    }
    if (menuItem.action == @selector(paste:)) {
        NSPasteboard *pasteboard = [NSPasteboard generalPasteboard];
        return [pasteboard canReadItemWithDataConformingToTypes:
                @[NSPasteboardTypeString]];
    }
    return YES;
}
```

---

## 85.6 NSToolbar

### การสร้าง Toolbar

```objc
// ToolbarController.h
@interface ToolbarController : NSObject <NSToolbarDelegate>

- (NSToolbar *)createToolbar;

@end

// ToolbarController.m
@implementation ToolbarController {
    NSMutableDictionary<NSToolbarItemIdentifier, NSToolbarItem *> *_toolbarItems;
}

// กำหนด identifier สำหรับ toolbar items
static NSToolbarItemIdentifier const NewDocumentToolbarItemID = @"NewDocument";
static NSToolbarItemIdentifier const OpenDocumentToolbarItemID = @"OpenDocument";
static NSToolbarItemIdentifier const SaveDocumentToolbarItemID = @"SaveDocument";
static NSToolbarItemIdentifier const SearchToolbarItemID = @"SearchField";

- (instancetype)init {
    self = [super init];
    if (self) {
        [self setupToolbarItems];
    }
    return self;
}

- (void)setupToolbarItems {
    _toolbarItems = [NSMutableDictionary dictionary];
    
    // New Document button
    NSToolbarItem *newItem = [[NSToolbarItem alloc] 
        initWithItemIdentifier:NewDocumentToolbarItemID];
    newItem.label = @"New";
    newItem.paletteLabel = @"New Document";
    newItem.toolTip = @"สร้างเอกสารใหม่";
    newItem.image = [NSImage imageWithSystemSymbolName:@"doc.badge.plus" 
                                  accessibilityDescription:@"New"];
    newItem.target = nil;
    newItem.action = @selector(newDocument:);
    _toolbarItems[NewDocumentToolbarItemID] = newItem;
    
    // Open button
    NSToolbarItem *openItem = [[NSToolbarItem alloc] 
        initWithItemIdentifier:OpenDocumentToolbarItemID];
    openItem.label = @"Open";
    openItem.paletteLabel = @"Open Document";
    openItem.toolTip = @"เปิดเอกสาร";
    openItem.image = [NSImage imageWithSystemSymbolName:@"folder.badge.plus" 
                                   accessibilityDescription:@"Open"];
    openItem.target = nil;
    openItem.action = @selector(openDocument:);
    _toolbarItems[OpenDocumentToolbarItemID] = openItem;
    
    // Search field
    NSSearchField *searchField = [[NSSearchField alloc] initWithFrame:NSMakeRect(0, 0, 200, 32)];
    searchField.placeholderString = @"ค้นหา...";
    
    NSToolbarItem *searchItem = [[NSToolbarItem alloc] 
        initWithItemIdentifier:SearchToolbarItemID];
    searchItem.label = @"Search";
    searchItem.view = searchField;
    searchItem.minSize = NSMakeSize(100, 32);
    searchItem.maxSize = NSMakeSize(300, 32);
    _toolbarItems[SearchToolbarItemID] = searchItem;
}

- (NSToolbar *)createToolbar {
    NSToolbar *toolbar = [[NSToolbar alloc] initWithIdentifier:@"MainToolbar"];
    toolbar.delegate = self;
    toolbar.allowsUserCustomization = YES;
    toolbar.autosavesConfiguration = YES;
    toolbar.displayMode = NSToolbarDisplayModeIconAndLabel;
    return toolbar;
}

// NSToolbarDelegate methods
- (NSToolbarItem *)toolbar:(NSToolbar *)toolbar 
     itemForItemIdentifier:(NSToolbarItemIdentifier)itemIdentifier 
 willBeInsertedIntoToolbar:(BOOL)flag {
    return _toolbarItems[itemIdentifier];
}

- (NSArray<NSToolbarItemIdentifier> *)toolbarDefaultItemIdentifiers:(NSToolbar *)toolbar {
    return @[
        NewDocumentToolbarItemID,
        OpenDocumentToolbarItemID,
        SaveDocumentToolbarItemID,
        NSToolbarFlexibleSpaceItem,
        SearchToolbarItemID
    ];
}

- (NSArray<NSToolbarItemIdentifier> *)toolbarAllowedItemIdentifiers:(NSToolbar *)toolbar {
    return @[
        NewDocumentToolbarItemID,
        OpenDocumentToolbarItemID,
        SaveDocumentToolbarItemID,
        SearchToolbarItemID,
        NSToolbarFlexibleSpaceItem,
        NSToolbarSpaceItem,
        NSToolbarSeparatorItem
    ];
}

@end
```

---

## 85.7 NSSplitViewController

### การสร้าง Split View Interface

```objc
// MainSplitViewController.m
@implementation MainSplitViewController

- (void)viewDidLoad {
    [super viewDidLoad];
    
    [self setupSplitView];
}

- (void)setupSplitView {
    // สร้าง sidebar
    SidebarViewController *sidebar = [[SidebarViewController alloc] init];
    NSSplitViewItem *sidebarItem = [NSSplitViewItem sidebarWithViewController:sidebar];
    sidebarItem.minimumThickness = 150;
    sidebarItem.maximumThickness = 300;
    sidebarItem.preferredThicknessFraction = 0.25;
    
    // สร้าง content
    ContentViewController *content = [[ContentViewController alloc] init];
    NSSplitViewItem *contentItem = [NSSplitViewItem splitViewItemWithViewController:content];
    
    // สร้าง inspector/detail panel (optional)
    InspectorViewController *inspector = [[InspectorViewController alloc] init];
    NSSplitViewItem *inspectorItem = [NSSplitViewItem inspectorWithViewController:inspector];
    inspectorItem.minimumThickness = 200;
    inspectorItem.isCollapsed = YES;  // ซ่อนไว้ตอนเริ่มต้น
    
    // เพิ่ม items
    [self addSplitViewItem:sidebarItem];
    [self addSplitViewItem:contentItem];
    [self addSplitViewItem:inspectorItem];
    
    // ตั้งค่า split view
    self.splitView.isVertical = YES;  // แนวตั้ง (ซ้าย-ขวา)
}

// Toggle sidebar
- (IBAction)toggleSidebar:(id)sender {
    NSSplitViewItem *sidebarItem = self.splitViewItems.firstObject;
    
    [NSAnimationContext runAnimationGroup:^(NSAnimationContext *context) {
        context.duration = 0.2;
        context.allowsImplicitAnimation = YES;
        sidebarItem.animator.isCollapsed = !sidebarItem.isCollapsed;
    } completionHandler:nil];
}

@end
```

---

## 85.8 NSTableView

### การสร้างและใช้งาน Table View

```objc
// TableViewController.h
@interface TableViewController : NSViewController <NSTableViewDataSource, NSTableViewDelegate>

@end

// TableViewController.m
@implementation TableViewController {
    NSTableView *_tableView;
    NSMutableArray<NSDictionary *> *_data;
}

- (void)viewDidLoad {
    [super viewDidLoad];
    
    // สร้างข้อมูลตัวอย่าง
    _data = [NSMutableArray array];
    for (int i = 1; i <= 50; i++) {
        [_data addObject:@{
            @"name": [NSString stringWithFormat:@"รายการที่ %d", i],
            @"value": @(arc4random_uniform(100)),
            @"status": i % 3 == 0 ? @"ใช้งาน" : @"ไม่ใช้งาน"
        }];
    }
    
    [self setupTableView];
}

- (void)setupTableView {
    // สร้าง scroll view
    NSScrollView *scrollView = [[NSScrollView alloc] initWithFrame:self.view.bounds];
    scrollView.autoresizingMask = NSViewWidthSizable | NSViewHeightSizable;
    scrollView.hasVerticalScroller = YES;
    scrollView.hasHorizontalScroller = NO;
    
    // สร้าง table view
    _tableView = [[NSTableView alloc] init];
    _tableView.dataSource = self;
    _tableView.delegate = self;
    _tableView.allowsMultipleSelection = YES;
    _tableView.usesAlternatingRowBackgroundColors = YES;
    _tableView.rowHeight = 40;
    
    // เพิ่ม columns
    NSTableColumn *nameColumn = [[NSTableColumn alloc] initWithIdentifier:@"name"];
    nameColumn.title = @"ชื่อ";
    nameColumn.width = 200;
    nameColumn.minWidth = 100;
    [_tableView addTableColumn:nameColumn];
    
    NSTableColumn *valueColumn = [[NSTableColumn alloc] initWithIdentifier:@"value"];
    valueColumn.title = @"ค่า";
    valueColumn.width = 100;
    [_tableView addTableColumn:valueColumn];
    
    NSTableColumn *statusColumn = [[NSTableColumn alloc] initWithIdentifier:@"status"];
    statusColumn.title = @"สถานะ";
    statusColumn.width = 100;
    [_tableView addTableColumn:statusColumn];
    
    scrollView.documentView = _tableView;
    [self.view addSubview:scrollView];
    
    // ลงทะเบียน notification สำหรับ selection change
    [[NSNotificationCenter defaultCenter] addObserver:self
                                             selector:@selector(tableSelectionChanged:)
                                                 name:NSTableViewSelectionDidChangeNotification
                                               object:_tableView];
}

// NSTableViewDataSource
- (NSInteger)numberOfRowsInTableView:(NSTableView *)tableView {
    return _data.count;
}

// NSTableViewDelegate - ใช้ View-based table view (แนะนำ)
- (NSView *)tableView:(NSTableView *)tableView
   viewForTableColumn:(NSTableColumn *)tableColumn
                  row:(NSInteger)row {
    
    NSString *identifier = tableColumn.identifier;
    NSDictionary *item = _data[row];
    
    NSTableCellView *cellView = [tableView makeViewWithIdentifier:identifier owner:self];
    if (!cellView) {
        cellView = [[NSTableCellView alloc] initWithFrame:NSMakeRect(0, 0, tableColumn.width, 40)];
        cellView.identifier = identifier;
        
        NSTextField *textField = [NSTextField labelWithString:@""];
        textField.translatesAutoresizingMaskIntoConstraints = NO;
        [cellView addSubview:textField];
        cellView.textField = textField;
        
        [NSLayoutConstraint activateConstraints:@[
            [textField.centerYAnchor constraintEqualToAnchor:cellView.centerYAnchor],
            [textField.leadingAnchor constraintEqualToAnchor:cellView.leadingAnchor constant:8],
            [textField.trailingAnchor constraintEqualToAnchor:cellView.trailingAnchor constant:-8],
        ]];
    }
    
    cellView.textField.stringValue = [item[identifier] description];
    
    // จัดสีตามสถานะ
    if ([identifier isEqualToString:@"status"]) {
        NSString *status = item[@"status"];
        cellView.textField.textColor = [status isEqualToString:@"ใช้งาน"] 
            ? [NSColor systemGreenColor] 
            : [NSColor systemRedColor];
    }
    
    return cellView;
}

- (CGFloat)tableView:(NSTableView *)tableView heightOfRow:(NSInteger)row {
    return 40;
}

- (void)tableSelectionChanged:(NSNotification *)notification {
    NSIndexSet *selectedRows = _tableView.selectedRowIndexes;
    NSLog(@"Selected rows: %@", selectedRows);
    
    if (selectedRows.count == 1) {
        NSDictionary *item = _data[selectedRows.firstIndex];
        NSLog(@"Selected: %@", item[@"name"]);
    }
}

// Sort descriptors
- (void)tableView:(NSTableView *)tableView 
    sortDescriptorsDidChange:(NSArray<NSSortDescriptor *> *)oldDescriptors {
    
    [_data sortWithOptions:0 usingComparator:^NSComparisonResult(id a, id b) {
        for (NSSortDescriptor *descriptor in tableView.sortDescriptors) {
            NSComparisonResult result = [[a valueForKey:descriptor.key] 
                compare:[b valueForKey:descriptor.key]];
            if (result != NSOrderedSame) {
                return descriptor.ascending ? result : -result;
            }
        }
        return NSOrderedSame;
    }];
    
    [_tableView reloadData];
}

@end
```

---

## 85.9 macOS-specific UI Components

### NSOutlineView (Hierarchical Data)

```objc
@interface OutlineViewController : NSViewController 
    <NSOutlineViewDataSource, NSOutlineViewDelegate>

@end

@implementation OutlineViewController {
    NSOutlineView *_outlineView;
    NSDictionary *_rootData;
}

- (void)viewDidLoad {
    [super viewDidLoad];
    
    // ข้อมูล hierarchical
    _rootData = @{
        @"ชื่อ": @"โครงการ",
        @"ลูก": @[
            @{@"ชื่อ": @"โฟลเดอร์ 1", @"ลูก": @[
                @{@"ชื่อ": @"ไฟล์ 1.txt", @"ลูก": @[]},
                @{@"ชื่อ": @"ไฟล์ 2.txt", @"ลูก": @[]}
            ]},
            @{@"ชื่อ": @"โฟลเดอร์ 2", @"ลูก": @[
                @{@"ชื่อ": @"ไฟล์ 3.txt", @"ลูก": @[]}
            ]}
        ]
    };
    
    [self setupOutlineView];
}

- (NSInteger)outlineView:(NSOutlineView *)outlineView 
    numberOfChildrenOfItem:(id)item {
    
    NSDictionary *node = item ?: _rootData;
    return [(NSArray *)node[@"ลูก"] count];
}

- (id)outlineView:(NSOutlineView *)outlineView 
            child:(NSInteger)index 
           ofItem:(id)item {
    
    NSDictionary *node = item ?: _rootData;
    return node[@"ลูก"][index];
}

- (BOOL)outlineView:(NSOutlineView *)outlineView 
   isItemExpandable:(id)item {
    
    NSDictionary *node = item;
    return [(NSArray *)node[@"ลูก"] count] > 0;
}

- (NSView *)outlineView:(NSOutlineView *)outlineView 
     viewForTableColumn:(NSTableColumn *)tableColumn 
                   item:(id)item {
    
    NSTableCellView *cellView = [outlineView makeViewWithIdentifier:@"cell" owner:self];
    if (!cellView) {
        cellView = [[NSTableCellView alloc] init];
        // ตั้งค่า cell...
    }
    
    NSDictionary *node = item;
    cellView.textField.stringValue = node[@"ชื่อ"];
    
    // ไอคอนตามประเภท
    BOOL isFolder = [(NSArray *)node[@"ลูก"] count] > 0;
    NSString *iconName = isFolder ? @"folder" : @"doc.text";
    cellView.imageView.image = [NSImage imageWithSystemSymbolName:iconName 
                                            accessibilityDescription:nil];
    
    return cellView;
}

@end
```

### NSComboBox, NSPopUpButton, NSColorWell

```objc
- (void)setupMacSpecificControls {
    // NSPopUpButton - Dropdown menu
    NSPopUpButton *popUp = [[NSPopUpButton alloc] initWithFrame:NSMakeRect(0, 0, 200, 30)];
    [popUp addItemsWithTitles:@[@"ตัวเลือก 1", @"ตัวเลือก 2", @"ตัวเลือก 3"]];
    [popUp setTarget:self];
    [popUp setAction:@selector(popUpChanged:)];
    
    // NSComboBox - Editable dropdown
    NSComboBox *comboBox = [[NSComboBox alloc] initWithFrame:NSMakeRect(0, 0, 200, 30)];
    [comboBox addItemsWithObjectValues:@[@"Apple", @"Google", @"Microsoft"]];
    comboBox.placeholderString = @"พิมพ์หรือเลือก...";
    comboBox.completes = YES;  // auto-complete
    
    // NSColorWell - Color picker
    NSColorWell *colorWell = [[NSColorWell alloc] initWithFrame:NSMakeRect(0, 0, 50, 30)];
    colorWell.color = [NSColor redColor];
    [colorWell setTarget:self];
    [colorWell setAction:@selector(colorChanged:)];
    
    // NSSlider
    NSSlider *slider = [NSSlider sliderWithValue:50
                                        minValue:0
                                        maxValue:100
                                          target:self
                                          action:@selector(sliderChanged:)];
    
    // NSDatePicker
    NSDatePicker *datePicker = [[NSDatePicker alloc] initWithFrame:NSMakeRect(0, 0, 200, 30)];
    datePicker.datePickerStyle = NSDatePickerStyleTextFieldAndStepper;
    datePicker.datePickerElements = NSDatePickerElementFlagYearMonthDay;
    datePicker.dateValue = [NSDate date];
}

- (void)colorChanged:(NSColorWell *)sender {
    NSLog(@"สีที่เลือก: %@", sender.color);
}
```

### NSStatusItem (Menu Bar Item)

```objc
// การสร้าง Menu Bar item (icon ใน menu bar)
@implementation StatusBarController {
    NSStatusItem *_statusItem;
}

- (void)setupStatusBar {
    _statusItem = [[NSStatusBar systemStatusBar] 
        statusItemWithLength:NSVariableStatusItemLength];
    
    // กำหนด button
    NSStatusBarButton *button = _statusItem.button;
    button.image = [NSImage imageWithSystemSymbolName:@"star.fill" 
                                 accessibilityDescription:@"MyApp"];
    button.toolTip = @"MyApp";
    [button setTarget:self];
    [button setAction:@selector(statusBarButtonClicked:)];
    
    // กำหนด menu
    NSMenu *menu = [[NSMenu alloc] init];
    [menu addItemWithTitle:@"เปิดแอป" 
                    action:@selector(showApp:) 
             keyEquivalent:@""];
    [menu addItem:[NSMenuItem separatorItem]];
    [menu addItemWithTitle:@"ออก" 
                    action:@selector(quit:) 
             keyEquivalent:@""];
    
    _statusItem.menu = menu;
}

- (void)statusBarButtonClicked:(NSStatusBarButton *)sender {
    // แสดง popover หรือ window
    NSPopover *popover = [[NSPopover alloc] init];
    popover.contentViewController = [[StatusPopoverViewController alloc] init];
    popover.behavior = NSPopoverBehaviorTransient;
    
    [popover showRelativeToRect:sender.bounds 
                         ofView:sender 
                  preferredEdge:NSRectEdgeMaxY];
}

@end
```

---

## 85.10 Sandboxing บน macOS

### App Sandbox Entitlements

```xml
<!-- MyApp.entitlements -->
<?xml version="1.0" encoding="UTF-8"?>
<!DOCTYPE plist PUBLIC "-//Apple//DTD PLIST 1.0//EN" 
    "http://www.apple.com/DTDs/PropertyList-1.0.dtd">
<plist version="1.0">
<dict>
    <!-- เปิดใช้ App Sandbox -->
    <key>com.apple.security.app-sandbox</key>
    <true/>
    
    <!-- เข้าถึงอินเทอร์เน็ต (outbound) -->
    <key>com.apple.security.network.client</key>
    <true/>
    
    <!-- เปิดรับการเชื่อมต่อ (server) -->
    <key>com.apple.security.network.server</key>
    <true/>
    
    <!-- อ่านไฟล์ที่ผู้ใช้เลือก -->
    <key>com.apple.security.files.user-selected.read-only</key>
    <true/>
    
    <!-- อ่านและเขียนไฟล์ที่ผู้ใช้เลือก -->
    <key>com.apple.security.files.user-selected.read-write</key>
    <true/>
    
    <!-- เข้าถึงกล้อง -->
    <key>com.apple.security.device.camera</key>
    <true/>
    
    <!-- เข้าถึงไมโครโฟน -->
    <key>com.apple.security.device.audio-input</key>
    <true/>
    
    <!-- เข้าถึงผู้ติดต่อ -->
    <key>com.apple.security.personal-information.addressbook</key>
    <true/>
</dict>
</plist>
```

### Security-Scoped Bookmarks

```objc
// บันทึกสิทธิ์การเข้าถึงไฟล์ระหว่าง sessions
- (void)saveBookmarkForURL:(NSURL *)fileURL {
    NSError *error = nil;
    NSData *bookmarkData = [fileURL bookmarkDataWithOptions:
        NSURLBookmarkCreationWithSecurityScope
        includingResourceValuesForKeys:nil
        relativeToURL:nil
        error:&error];
    
    if (error) {
        NSLog(@"Error creating bookmark: %@", error);
        return;
    }
    
    // บันทึก bookmark
    NSUserDefaults *defaults = [NSUserDefaults standardUserDefaults];
    [defaults setObject:bookmarkData forKey:@"savedBookmark"];
}

- (NSURL *)resolveBookmark {
    NSData *bookmarkData = [[NSUserDefaults standardUserDefaults] 
        objectForKey:@"savedBookmark"];
    
    if (!bookmarkData) return nil;
    
    BOOL isStale = NO;
    NSError *error = nil;
    NSURL *url = [NSURL URLByResolvingBookmarkData:bookmarkData
                                           options:NSURLBookmarkResolutionWithSecurityScope
                                     relativeToURL:nil
                               bookmarkDataIsStale:&isStale
                                             error:&error];
    
    if (error) {
        NSLog(@"Error resolving bookmark: %@", error);
        return nil;
    }
    
    if (isStale) {
        // ต้องสร้าง bookmark ใหม่
        [self saveBookmarkForURL:url];
    }
    
    // ต้องเรียก startAccessingSecurityScopedResource ก่อนใช้
    if ([url startAccessingSecurityScopedResource]) {
        // ใช้ url...
        // เมื่อเสร็จแล้ว:
        [url stopAccessingSecurityScopedResource];
    }
    
    return url;
}
```

---

## 85.11 Drag and Drop

### การ Implement Drag and Drop

```objc
// DragDropView.m
@implementation DragDropView

- (instancetype)initWithFrame:(NSRect)frameRect {
    self = [super initWithFrame:frameRect];
    if (self) {
        // ลงทะเบียน types ที่รับได้
        [self registerForDraggedTypes:@[
            NSPasteboardTypeFileURL,
            NSPasteboardTypeString,
            NSPasteboardTypePNG,
            NSPasteboardTypeTIFF
        ]];
    }
    return self;
}

// ========== Drag Destination (รับ drag) ==========

- (NSDragOperation)draggingEntered:(id<NSDraggingInfo>)sender {
    // ตรวจสอบว่ารับ type นี้ได้ไหม
    NSPasteboard *pasteboard = sender.draggingPasteboard;
    
    if ([pasteboard availableTypeFromArray:@[NSPasteboardTypeFileURL]]) {
        // แสดง highlight
        _isHighlighted = YES;
        [self setNeedsDisplay:YES];
        return NSDragOperationCopy;
    }
    
    return NSDragOperationNone;
}

- (void)draggingExited:(id<NSDraggingInfo>)sender {
    _isHighlighted = NO;
    [self setNeedsDisplay:YES];
}

- (NSDragOperation)draggingUpdated:(id<NSDraggingInfo>)sender {
    return NSDragOperationCopy;
}

- (BOOL)performDragOperation:(id<NSDraggingInfo>)sender {
    NSPasteboard *pasteboard = sender.draggingPasteboard;
    
    // รับไฟล์
    NSArray *fileURLs = [pasteboard readObjectsForClasses:@[[NSURL class]] 
                                                  options:@{
        NSPasteboardURLReadingFileURLsOnlyKey: @YES
    }];
    
    if (fileURLs.count > 0) {
        for (NSURL *fileURL in fileURLs) {
            NSLog(@"ไฟล์ที่ drop: %@", fileURL.path);
            [self processDroppedFile:fileURL];
        }
        _isHighlighted = NO;
        [self setNeedsDisplay:YES];
        return YES;
    }
    
    return NO;
}

// ========== Drag Source (ส่ง drag) ==========

- (void)mouseDragged:(NSEvent *)event {
    NSPoint mouseLocation = [self convertPoint:event.locationInWindow fromView:nil];
    
    // สร้าง drag item
    NSPasteboardItem *pasteboardItem = [[NSPasteboardItem alloc] init];
    [pasteboardItem setString:@"ข้อมูลที่ drag" forType:NSPasteboardTypeString];
    
    NSDraggingItem *draggingItem = [[NSDraggingItem alloc] initWithPasteboardWriter:pasteboardItem];
    
    // กำหนด drag image
    NSImage *dragImage = [NSImage imageWithSystemSymbolName:@"doc" 
                                       accessibilityDescription:nil];
    [draggingItem setDraggingFrame:NSMakeRect(mouseLocation.x - 25, 
                                              mouseLocation.y - 25, 
                                              50, 50)
                          contents:dragImage];
    
    // เริ่ม drag session
    [self beginDraggingSessionWithItems:@[draggingItem] 
                                  event:event 
                                 source:self];
}

- (NSDragOperation)draggingSession:(NSDraggingSession *)session 
    sourceOperationMaskForDraggingContext:(NSDraggingContext)context {
    
    return NSDragOperationCopy | NSDragOperationMove;
}

@end
```

---

## 85.12 Services Menu

### การ Register Service

```objc
// Info.plist - กำหนด Service
/*
<key>NSServices</key>
<array>
    <dict>
        <key>NSMenuItem</key>
        <dict>
            <key>default</key>
            <string>แปลงข้อความ</string>
        </dict>
        <key>NSMessage</key>
        <string>processText</string>
        <key>NSPortName</key>
        <string>MyApp</string>
        <key>NSSendTypes</key>
        <array>
            <string>NSStringPboardType</string>
        </array>
        <key>NSReturnTypes</key>
        <array>
            <string>NSStringPboardType</string>
        </array>
    </dict>
</array>
*/

// AppDelegate.m - ลงทะเบียน service provider
- (void)applicationDidFinishLaunching:(NSNotification *)aNotification {
    [NSApp setServicesProvider:self];
}

// ฟังก์ชัน service handler
- (void)processText:(NSPasteboard *)pasteboard 
           userData:(NSString *)userData
              error:(NSString **)error {
    
    NSString *text = [pasteboard stringForType:NSPasteboardTypeString];
    if (!text) {
        *error = @"ไม่มีข้อความที่เลือก";
        return;
    }
    
    // ประมวลผลข้อความ
    NSString *processed = [text uppercaseString];
    
    // คืนผลลัพธ์
    [pasteboard clearContents];
    [pasteboard setString:processed forType:NSPasteboardTypeString];
}
```

---

## 85.13 Sparkle Framework สำหรับ Auto-Update

### การตั้งค่า Sparkle

```objc
// AppDelegate.m
#import <Sparkle/Sparkle.h>

@interface AppDelegate () <SPUUpdaterDelegate>

@property (strong) SPUStandardUpdaterController *updaterController;

@end

@implementation AppDelegate

- (void)applicationDidFinishLaunching:(NSNotification *)aNotification {
    // ตั้งค่า Sparkle
    self.updaterController = [[SPUStandardUpdaterController alloc] 
        initWithStartingUpdater:YES
                updaterDelegate:self
              userDriverDelegate:nil];
}

// สร้าง menu item สำหรับ Check for Updates
- (IBAction)checkForUpdates:(id)sender {
    [self.updaterController checkForUpdates:sender];
}

// SPUUpdaterDelegate
- (BOOL)updater:(SPUUpdater *)updater 
    shouldPostponeRelaunchForUpdate:(SUAppcastItem *)item 
                   untilInvokingBlock:(void (^)(void))installHandler {
    
    // บันทึกงานก่อน restart
    [self saveWork];
    installHandler();
    return NO;
}

// Info.plist settings
/*
<key>SUFeedURL</key>
<string>https://myapp.example.com/appcast.xml</string>
<key>SUPublicEDKey</key>
<string>YOUR_PUBLIC_KEY_HERE</string>
*/

@end
```

### Appcast XML Format

```xml
<?xml version="1.0" encoding="utf-8"?>
<rss version="2.0" xmlns:sparkle="http://www.andymatuschak.org/xml-namespaces/sparkle"
     xmlns:dc="http://purl.org/dc/elements/1.1/">
    <channel>
        <title>MyApp Changelog</title>
        <link>https://myapp.example.com</link>
        <description>Most recent changes with links to updates.</description>
        <language>th</language>
        
        <item>
            <title>Version 2.0</title>
            <description>
                <![CDATA[
                    <h2>สิ่งใหม่ในเวอร์ชัน 2.0</h2>
                    <ul>
                        <li>ปรับปรุงประสิทธิภาพ</li>
                        <li>แก้ไขข้อบกพร่อง</li>
                    </ul>
                ]]>
            </description>
            <pubDate>Wed, 01 Jan 2025 12:00:00 +0000</pubDate>
            <sparkle:version>200</sparkle:version>
            <sparkle:shortVersionString>2.0</sparkle:shortVersionString>
            <sparkle:minimumSystemVersion>12.0</sparkle:minimumSystemVersion>
            <enclosure url="https://myapp.example.com/MyApp-2.0.zip"
                       length="5000000"
                       type="application/octet-stream"
                       sparkle:edSignature="YOUR_SIGNATURE"/>
        </item>
    </channel>
</rss>
```

---

## 85.14 Mac Catalyst Overview

### การ migrate iOS App มา macOS ด้วย Mac Catalyst

Mac Catalyst ช่วยให้นำแอป iOS มาทำงานบน macOS ได้โดยใช้โค้ดเดียวกัน

```objc
// ตรวจสอบว่ากำลังทำงานบน Mac Catalyst
#if TARGET_OS_MACCATALYST
    NSLog(@"กำลังทำงานบน Mac Catalyst");
    // โค้ดสำหรับ Mac
#else
    NSLog(@"กำลังทำงานบน iOS");
    // โค้ดสำหรับ iOS
#endif

// การปรับ UI สำหรับ Mac Catalyst
- (void)setupForMac {
#if TARGET_OS_MACCATALYST
    // ปรับ navigation bar style
    self.navigationController.navigationBar.prefersLargeTitles = NO;
    
    // เพิ่ม toolbar items
    NSToolbar *toolbar = [[NSToolbar alloc] initWithIdentifier:@"MainToolbar"];
    self.navigationController.navigationBar.hidden = YES;
    
    // ใช้ UIKit ปกติ แต่ Apple แปลงเป็น AppKit controls ให้อัตโนมัติ
    UIBarButtonItem *addButton = [[UIBarButtonItem alloc] 
        initWithBarButtonSystemItem:UIBarButtonSystemItemAdd
                             target:self
                             action:@selector(addItem:)];
    self.navigationItem.rightBarButtonItem = addButton;
#endif
}

// การเพิ่ม menu commands สำหรับ Mac
- (void)buildMenuWithBuilder:(id<UIMenuBuilder>)builder API_AVAILABLE(ios(13.0)) {
    [super buildMenuWithBuilder:builder];
    
    // เพิ่ม custom menu
    UICommand *newCommand = [UICommand commandWithTitle:@"New Item"
                                                 image:nil
                                                action:@selector(createNewItem:)
                                          propertyList:nil];
    
    UIMenu *fileMenu = [UIMenu menuWithTitle:@""
                                      image:nil
                                 identifier:UIMenuFile
                                    options:UIMenuOptionsDisplayInline
                                   children:@[newCommand]];
    
    [builder insertChildMenu:fileMenu atStartOfMenuForIdentifier:UIMenuFile];
}
```

---

## 85.15 แบบฝึกหัด

### แบบฝึกหัดที่ 1: สร้าง Text Editor อย่างง่าย

```
สร้างแอป macOS text editor ที่มีคุณสมบัติ:
1. NSWindow พร้อม NSToolbar ที่มีปุ่ม New, Open, Save
2. NSTextView สำหรับพิมพ์ข้อความ
3. Menu bar พร้อม File, Edit menus
4. Status bar แสดงจำนวนคำ
5. Drag & Drop รับไฟล์ .txt
```

### Solution แบบฝึกหัดที่ 1

```objc
// TextEditorViewController.m
@implementation TextEditorViewController {
    NSTextView *_textView;
    NSTextField *_statusLabel;
}

- (void)viewDidLoad {
    [super viewDidLoad];
    [self setupUI];
}

- (void)setupUI {
    // Text View
    NSScrollView *scrollView = [[NSScrollView alloc] init];
    scrollView.hasVerticalScroller = YES;
    scrollView.translatesAutoresizingMaskIntoConstraints = NO;
    
    _textView = [[NSTextView alloc] init];
    _textView.font = [NSFont monospacedSystemFontOfSize:14 weight:NSFontWeightRegular];
    _textView.allowsUndo = YES;
    _textView.delegate = self;
    scrollView.documentView = _textView;
    
    [self.view addSubview:scrollView];
    
    // Status bar
    _statusLabel = [NSTextField labelWithString:@"0 คำ"];
    _statusLabel.translatesAutoresizingMaskIntoConstraints = NO;
    [self.view addSubview:_statusLabel];
    
    // Constraints
    [NSLayoutConstraint activateConstraints:@[
        [scrollView.topAnchor constraintEqualToAnchor:self.view.topAnchor],
        [scrollView.leadingAnchor constraintEqualToAnchor:self.view.leadingAnchor],
        [scrollView.trailingAnchor constraintEqualToAnchor:self.view.trailingAnchor],
        [scrollView.bottomAnchor constraintEqualToAnchor:_statusLabel.topAnchor constant:-4],
        [_statusLabel.leadingAnchor constraintEqualToAnchor:self.view.leadingAnchor constant:8],
        [_statusLabel.bottomAnchor constraintEqualToAnchor:self.view.bottomAnchor constant:-4],
    ]];
    
    // ลงทะเบียน Drag & Drop
    [self.view registerForDraggedTypes:@[NSPasteboardTypeFileURL]];
}

- (void)textDidChange:(NSNotification *)notification {
    NSString *text = _textView.string;
    NSArray *words = [text componentsSeparatedByCharactersInSet:
                     [NSCharacterSet whitespaceAndNewlineCharacterSet]];
    NSInteger wordCount = [[words filteredArrayUsingPredicate:
                           [NSPredicate predicateWithFormat:@"length > 0"]] count];
    _statusLabel.stringValue = [NSString stringWithFormat:@"%ld คำ", wordCount];
}

// Open file
- (IBAction)openDocument:(id)sender {
    NSOpenPanel *panel = [NSOpenPanel openPanel];
    panel.allowedContentTypes = @[[UTType typeWithFilenameExtension:@"txt"]];
    panel.allowsMultipleSelection = NO;
    
    [panel beginWithCompletionHandler:^(NSModalResponse result) {
        if (result == NSModalResponseOK) {
            NSError *error;
            NSString *content = [NSString stringWithContentsOfURL:panel.URL 
                                                         encoding:NSUTF8StringEncoding 
                                                            error:&error];
            if (content) {
                dispatch_async(dispatch_get_main_queue(), ^{
                    self->_textView.string = content;
                });
            }
        }
    }];
}

// Save file
- (IBAction)saveDocument:(id)sender {
    NSSavePanel *panel = [NSSavePanel savePanel];
    panel.allowedContentTypes = @[[UTType typeWithFilenameExtension:@"txt"]];
    
    [panel beginWithCompletionHandler:^(NSModalResponse result) {
        if (result == NSModalResponseOK) {
            NSError *error;
            [self->_textView.string writeToURL:panel.URL 
                                    atomically:YES 
                                      encoding:NSUTF8StringEncoding 
                                         error:&error];
        }
    }];
}

@end
```

### แบบฝึกหัดที่ 2: Preferences Window

```objc
// PreferencesWindowController.m
@implementation PreferencesWindowController

- (instancetype)init {
    NSWindow *window = [[NSWindow alloc] 
        initWithContentRect:NSMakeRect(0, 0, 450, 300)
                  styleMask:NSWindowStyleMaskTitled | NSWindowStyleMaskClosable
                    backing:NSBackingStoreBuffered
                      defer:NO];
    window.title = @"การตั้งค่า";
    
    self = [super initWithWindow:window];
    if (self) {
        [self setupPreferences];
    }
    return self;
}

- (void)setupPreferences {
    // Tab view for different preference categories
    NSTabView *tabView = [[NSTabView alloc] initWithFrame:
        NSMakeRect(0, 44, 450, 256)];
    
    // General tab
    NSTabViewItem *generalItem = [[NSTabViewItem alloc] initWithIdentifier:@"general"];
    generalItem.label = @"ทั่วไป";
    [tabView addTabViewItem:generalItem];
    
    // Appearance tab
    NSTabViewItem *appearanceItem = [[NSTabViewItem alloc] initWithIdentifier:@"appearance"];
    appearanceItem.label = @"รูปลักษณ์";
    [tabView addTabViewItem:appearanceItem];
    
    [self.window.contentView addSubview:tabView];
    
    // OK/Cancel buttons
    NSButton *okButton = [NSButton buttonWithTitle:@"ตกลง" 
                                            target:self 
                                            action:@selector(okClicked:)];
    okButton.keyEquivalent = @"\r";  // Enter key
    okButton.frame = NSMakeRect(350, 12, 80, 28);
    [self.window.contentView addSubview:okButton];
    
    NSButton *cancelButton = [NSButton buttonWithTitle:@"ยกเลิก" 
                                                target:self 
                                                action:@selector(cancelClicked:)];
    cancelButton.keyEquivalent = @"\033";  // Escape key
    cancelButton.frame = NSMakeRect(260, 12, 80, 28);
    [self.window.contentView addSubview:cancelButton];
}

- (void)okClicked:(NSButton *)sender {
    // บันทึกการตั้งค่า
    [NSUserDefaults.standardUserDefaults synchronize];
    [self.window close];
}

- (void)cancelClicked:(NSButton *)sender {
    [self.window close];
}

@end
```

### แบบฝึกหัดที่ 3: Master-Detail Interface

```
สร้างแอปที่มี:
1. NSSplitViewController แบบ sidebar + content
2. NSOutlineView ใน sidebar แสดงหมวดหมู่
3. NSTableView ใน content แสดงรายการ
4. Detail panel ขวาสุดแสดงข้อมูล item ที่เลือก
5. Toolbar ด้านบนพร้อมปุ่ม Add/Delete
6. ค้นหาผ่าน NSSearchField
```

---

## สรุป

ในบทนี้เราได้เรียนรู้:

- **AppKit vs UIKit**: ความแตกต่างหลักระหว่างสองแพลตฟอร์ม
- **NSWindow/NSViewController**: พื้นฐานการสร้าง window และ controllers
- **NSView**: การวาดด้วย drawRect: และ NSBezierPath  
- **NSMenu/NSToolbar**: การสร้าง menu bar และ toolbar
- **NSSplitViewController**: การแบ่ง interface แบบ sidebar
- **NSTableView/NSOutlineView**: การแสดงข้อมูลแบบตาราง
- **Sandboxing**: ความปลอดภัยและการขอ permissions
- **Drag and Drop**: การรับและส่งข้อมูล
- **Sparkle**: การอัปเดตแอปอัตโนมัติ
- **Mac Catalyst**: การนำ iOS app มาใช้บน macOS

บทต่อไปจะเรียนรู้เกี่ยวกับ **Metal Framework** สำหรับการเขียนโปรแกรม GPU บน Apple platforms
