# Part 87: ARKit - Augmented Reality

## บทนำ

ARKit คือ framework ของ Apple สำหรับการสร้างประสบการณ์ Augmented Reality (AR) บน iOS และ iPadOS ARKit ใช้กล้องของอุปกรณ์ร่วมกับเซนเซอร์ต่างๆ เพื่อสร้างแผนที่โลกจริงและวาง virtual objects ลงในโลกนั้นได้อย่างแม่นยำ

ARKit เปิดตัวครั้งแรกใน iOS 11 และได้รับการพัฒนาอย่างต่อเนื่องในทุกๆ ปี

---

## 87.1 ARKit Overview

### หลักการทำงานของ ARKit

```
กล้อง (Camera) → ARKit → Virtual World
     ↓                         ↓
  รูปภาพ          การประมวลผล    Object 3D
  Video frames    VIO tracking  Planes
                  Scene understanding
                  
VIO = Visual Inertial Odometry
```

### ARKit Features หลัก

```
Feature                  | iOS version
-------------------------|------------
World Tracking           | iOS 11
Face Tracking            | iOS 11 (TrueDepth camera)
Image Tracking           | iOS 12
Object Detection         | iOS 12
People Occlusion         | iOS 13
Motion Capture           | iOS 13
Scene Geometry           | iOS 13.4 (LiDAR)
Location Anchors         | iOS 14
Body Tracking            | iOS 13
```

### Project Setup

```objc
// Info.plist - ต้องขอ permission กล้อง
/*
<key>NSCameraUsageDescription</key>
<string>ต้องการใช้กล้องเพื่อแสดง AR</string>
*/

// ตรวจสอบว่า ARKit รองรับหรือไม่
- (BOOL)checkARKitSupport {
    if (@available(iOS 11.0, *)) {
        return ARWorldTrackingConfiguration.isSupported;
    }
    return NO;
}
```

---

## 87.2 ARSession และ ARConfiguration

### ARSession - หัวใจของ ARKit

```objc
#import <ARKit/ARKit.h>

@interface ARViewController : UIViewController <ARSessionDelegate>

@property (strong, nonatomic) ARSession *session;
@property (strong, nonatomic) ARSCNView *sceneView;

@end

@implementation ARViewController

- (void)viewDidLoad {
    [super viewDidLoad];
    
    // สร้าง AR scene view
    _sceneView = [[ARSCNView alloc] initWithFrame:self.view.bounds];
    _sceneView.autoresizingMask = UIViewAutoresizingFlexibleWidth | UIViewAutoresizingFlexibleHeight;
    [self.view addSubview:_sceneView];
    
    // ดึง session จาก ARSCNView
    _session = _sceneView.session;
    _session.delegate = self;
    
    // ตั้งค่า debug options
    _sceneView.debugOptions = ARSCNDebugOptionShowFeaturePoints | 
                               ARSCNDebugOptionShowWorldOrigin;
    _sceneView.showsStatistics = YES;
    _sceneView.automaticallyUpdatesLighting = YES;
}

- (void)viewWillAppear:(BOOL)animated {
    [super viewWillAppear:animated];
    
    // สร้างและเริ่ม configuration
    ARWorldTrackingConfiguration *config = [[ARWorldTrackingConfiguration alloc] init];
    config.planeDetection = ARPlaneDetectionHorizontal | ARPlaneDetectionVertical;
    config.environmentTexturing = AREnvironmentTexturingAutomatic;
    
    // เริ่ม AR session
    [_session runWithConfiguration:config 
                           options:ARSessionRunOptionResetTracking | 
                                   ARSessionRunOptionRemoveExistingAnchors];
}

- (void)viewWillDisappear:(BOOL)animated {
    [super viewWillDisappear:animated];
    [_session pause];
}

// ARSessionDelegate methods
- (void)session:(ARSession *)session didUpdateFrame:(ARFrame *)frame {
    // เรียกทุก frame (~60 fps)
    // frame.camera.transform = current device position/rotation
    // frame.capturedImage = camera image
}

- (void)session:(ARSession *)session didFailWithError:(NSError *)error {
    NSLog(@"AR Session failed: %@", error.localizedDescription);
    
    if ([error.domain isEqualToString:ARErrorDomain]) {
        switch (error.code) {
            case ARErrorCodeCameraUnauthorized:
                // ไม่ได้รับสิทธิ์กล้อง
                break;
            case ARErrorCodeSensorUnavailable:
                // เซนเซอร์ไม่พร้อม
                break;
            case ARErrorCodeWorldTrackingFailed:
                // Tracking ล้มเหลว
                [self restartSession];
                break;
        }
    }
}

- (void)session:(ARSession *)session cameraDidChangeTrackingState:(ARCamera *)camera {
    switch (camera.trackingState) {
        case ARTrackingStateNotAvailable:
            NSLog(@"Tracking: ไม่พร้อม");
            break;
        case ARTrackingStateLimited:
            NSLog(@"Tracking: จำกัด - %@", [self trackingLimitedReason:camera]);
            break;
        case ARTrackingStateNormal:
            NSLog(@"Tracking: ปกติ");
            break;
    }
}

- (NSString *)trackingLimitedReason:(ARCamera *)camera {
    switch (camera.trackingStateReason) {
        case ARTrackingStateReasonInitializing:
            return @"กำลังเริ่มต้น";
        case ARTrackingStateReasonExcessiveMotion:
            return @"เคลื่อนไหวเร็วเกินไป";
        case ARTrackingStateReasonInsufficientFeatures:
            return @"สภาพแสงไม่เพียงพอ";
        case ARTrackingStateReasonRelocalizing:
            return @"กำลัง relocalize";
        default:
            return @"ไม่ทราบสาเหตุ";
    }
}

- (void)restartSession {
    ARWorldTrackingConfiguration *config = [[ARWorldTrackingConfiguration alloc] init];
    [_session runWithConfiguration:config 
                           options:ARSessionRunOptionResetTracking | 
                                   ARSessionRunOptionRemoveExistingAnchors];
}

@end
```

---

## 87.3 ARWorldTrackingConfiguration

### การตั้งค่า World Tracking

```objc
- (void)configureWorldTracking {
    ARWorldTrackingConfiguration *config = [[ARWorldTrackingConfiguration alloc] init];
    
    // ตรวจจับระนาบ
    config.planeDetection = ARPlaneDetectionHorizontal | ARPlaneDetectionVertical;
    
    // จัดการแสง environment
    config.environmentTexturing = AREnvironmentTexturingAutomatic;
    config.wantsHDREnvironmentTextures = YES;
    
    // Light estimation
    config.lightEstimationEnabled = YES;
    
    // Collaboration (Multi-user AR)
    config.isCollaborationEnabled = YES;  // iOS 13+
    
    // User face tracking (ต้องการ TrueDepth camera)
    if (ARWorldTrackingConfiguration.supportsUserFaceTracking) {
        config.userFaceTrackingEnabled = YES;
    }
    
    // Auto focus
    config.autoFocusEnabled = YES;
    
    // Video format
    // เลือก video format ที่ต้องการ
    NSArray<ARVideoFormat *> *formats = ARWorldTrackingConfiguration.supportedVideoFormats;
    if (formats.count > 0) {
        // เลือก format ที่มี resolution สูงสุด
        config.videoFormat = formats.firstObject;
    }
    
    [_session runWithConfiguration:config];
}

// รับ frame เพื่อประมวลผล
- (void)session:(ARSession *)session didUpdateFrame:(ARFrame *)frame {
    ARCamera *camera = frame.camera;
    
    // ข้อมูล camera
    simd_float4x4 transform = camera.transform;  // Camera transform matrix
    CGSize imageSize = frame.camera.imageResolution;
    float focalLength = camera.intrinsics.columns[0][0];  // fx
    
    // Tracking state
    ARTrackingState state = camera.trackingState;
    
    // รูปภาพจากกล้อง
    CVPixelBufferRef pixelBuffer = frame.capturedImage;
    
    // Light estimation
    ARLightEstimate *lightEstimate = frame.lightEstimate;
    if (lightEstimate) {
        CGFloat ambientIntensity = lightEstimate.ambientIntensity;
        CGFloat ambientColorTemperature = lightEstimate.ambientColorTemperature;
        
        NSLog(@"แสง: intensity=%.0f, temperature=%.0f", 
              ambientIntensity, ambientColorTemperature);
    }
    
    // Depth data (LiDAR devices)
    if (frame.sceneDepth) {
        ARDepthData *depthData = frame.sceneDepth;
        CVPixelBufferRef depthMap = depthData.depthMap;
        CVPixelBufferRef confidenceMap = depthData.confidenceMap;
    }
}
```

---

## 87.4 ARSCNView - SceneKit Integration

### การวาง 3D Objects บนพื้นผิว

```objc
@interface SceneARViewController : UIViewController 
    <ARSCNViewDelegate, ARSessionDelegate>

@end

@implementation SceneARViewController {
    ARSCNView *_sceneView;
    NSMutableArray<ARAnchor *> *_placedAnchors;
}

- (void)viewDidLoad {
    [super viewDidLoad];
    
    _sceneView = [[ARSCNView alloc] initWithFrame:self.view.bounds];
    _sceneView.delegate = self;
    _sceneView.session.delegate = self;
    [self.view addSubview:_sceneView];
    
    // เพิ่ม tap gesture สำหรับวาง objects
    UITapGestureRecognizer *tap = [[UITapGestureRecognizer alloc] 
        initWithTarget:self action:@selector(handleTap:)];
    [_sceneView addGestureRecognizer:tap];
    
    // Lighting
    _sceneView.autoenablesDefaultLighting = YES;
    _sceneView.automaticallyUpdatesLighting = YES;
}

// Tap to place object
- (void)handleTap:(UITapGestureRecognizer *)recognizer {
    CGPoint touchLocation = [recognizer locationInView:_sceneView];
    
    // Raycast จาก touch location
    if (@available(iOS 14.0, *)) {
        ARRaycastQuery *query = [_sceneView raycastQueryFromPoint:touchLocation
                                                    allowingTarget:ARRaycastTargetExistingPlaneInfinite
                                                         alignment:ARRaycastTargetAlignmentAny];
        
        NSArray<ARRaycastResult *> *results = [_sceneView.session raycast:query];
        
        if (results.count > 0) {
            ARRaycastResult *result = results.firstObject;
            [self placeObjectAtTransform:result.worldTransform];
        }
    } else {
        // iOS 13 และเก่ากว่า - ใช้ hitTest
        NSArray<ARHitTestResult *> *results = [_sceneView hitTest:touchLocation
                                                             types:ARHitTestResultTypeExistingPlaneUsingGeometry];
        if (results.count > 0) {
            ARHitTestResult *result = results.firstObject;
            [self placeObjectAtTransform:result.worldTransform];
        }
    }
}

- (void)placeObjectAtTransform:(simd_float4x4)transform {
    // สร้าง anchor ที่ตำแหน่งที่แตะ
    ARAnchor *anchor = [[ARAnchor alloc] initWithTransform:transform];
    [_sceneView.session addAnchor:anchor];
    [_placedAnchors addObject:anchor];
}

// ARSCNViewDelegate - สร้าง SceneKit node สำหรับ anchor ใหม่
- (SCNNode *)renderer:(id<SCNSceneRenderer>)renderer 
       nodeForAnchor:(ARAnchor *)anchor {
    
    // ไม่สร้าง node สำหรับ plane anchors (จะจัดการใน didAdd)
    if ([anchor isKindOfClass:[ARPlaneAnchor class]]) {
        return [SCNNode node];
    }
    
    // สร้าง 3D object
    SCNBox *box = [SCNBox boxWithWidth:0.1 height:0.1 length:0.1 chamferRadius:0.01];
    
    // Material
    SCNMaterial *material = [SCNMaterial material];
    material.diffuse.contents = [UIColor systemBlueColor];
    material.metalness.contents = @0.5;
    material.roughness.contents = @0.3;
    box.materials = @[material];
    
    // Node
    SCNNode *node = [SCNNode nodeWithGeometry:box];
    
    return node;
}

// สร้าง node สำหรับ plane anchor
- (void)renderer:(id<SCNSceneRenderer>)renderer 
      didAddNode:(SCNNode *)node 
       forAnchor:(ARAnchor *)anchor {
    
    if (![anchor isKindOfClass:[ARPlaneAnchor class]]) return;
    
    ARPlaneAnchor *planeAnchor = (ARPlaneAnchor *)anchor;
    
    SCNNode *planeNode = [self createPlaneNodeForAnchor:planeAnchor];
    [node addChildNode:planeNode];
    
    NSLog(@"พบระนาบ: %@ - ขนาด: %.2f x %.2f",
          planeAnchor.alignment == ARPlaneAnchorAlignmentHorizontal ? @"แนวนอน" : @"แนวตั้ง",
          planeAnchor.planeExtent.width,
          planeAnchor.planeExtent.height);
}

// อัปเดต node เมื่อ plane ขยายออก
- (void)renderer:(id<SCNSceneRenderer>)renderer 
   didUpdateNode:(SCNNode *)node 
       forAnchor:(ARAnchor *)anchor {
    
    if (![anchor isKindOfClass:[ARPlaneAnchor class]]) return;
    
    ARPlaneAnchor *planeAnchor = (ARPlaneAnchor *)anchor;
    
    // อัปเดต plane geometry
    SCNNode *planeNode = node.childNodes.firstObject;
    SCNPlane *plane = (SCNPlane *)planeNode.geometry;
    plane.width = planeAnchor.planeExtent.width;
    plane.height = planeAnchor.planeExtent.height;
    
    // อัปเดต position
    planeNode.position = SCNVector3Make(
        planeAnchor.center.x,
        0,
        planeAnchor.center.z
    );
}

- (SCNNode *)createPlaneNodeForAnchor:(ARPlaneAnchor *)anchor {
    SCNPlane *plane = [SCNPlane planeWithWidth:anchor.planeExtent.width 
                                        height:anchor.planeExtent.height];
    
    SCNMaterial *material = [SCNMaterial material];
    material.diffuse.contents = [UIColor colorWithRed:0.0 green:0.8 blue:0.0 alpha:0.3];
    material.doubleSided = YES;
    plane.materials = @[material];
    
    SCNNode *planeNode = [SCNNode nodeWithGeometry:plane];
    planeNode.eulerAngles = SCNVector3Make(-M_PI_2, 0, 0);  // หมุน 90 องศา
    planeNode.position = SCNVector3Make(anchor.center.x, 0, anchor.center.z);
    
    return planeNode;
}

// โหลด 3D model จาก .scn file
- (SCNNode *)loadModelNamed:(NSString *)name {
    NSString *path = [[NSBundle mainBundle] pathForResource:name ofType:@"scn"];
    NSURL *url = [NSURL fileURLWithPath:path];
    
    NSError *error;
    SCNScene *scene = [SCNScene sceneWithURL:url options:nil error:&error];
    
    if (error) {
        NSLog(@"ไม่สามารถโหลด model: %@", error);
        return nil;
    }
    
    return scene.rootNode;
}

@end
```

---

## 87.5 ARSKView - SpriteKit Integration

### AR กับ 2D Sprites

```objc
@interface ARSpriteViewController : UIViewController 
    <ARSKViewDelegate, ARSessionDelegate>

@end

@implementation ARSpriteViewController {
    ARSKView *_arView;
}

- (void)viewDidLoad {
    [super viewDidLoad];
    
    _arView = [[ARSKView alloc] initWithFrame:self.view.bounds];
    _arView.delegate = self;
    _arView.session.delegate = self;
    [self.view addSubview:_arView];
    
    // โหลด SpriteKit scene
    SKScene *scene = [SKScene sceneWithSize:self.view.bounds.size];
    scene.scaleMode = SKSceneScaleModeResizeFill;
    [_arView presentScene:scene];
    
    // Tap to place sprite
    UITapGestureRecognizer *tap = [[UITapGestureRecognizer alloc] 
        initWithTarget:self action:@selector(handleTap:)];
    [_arView addGestureRecognizer:tap];
}

- (void)handleTap:(UITapGestureRecognizer *)recognizer {
    CGPoint touchLocation = [recognizer locationInView:_arView];
    
    // Raycast
    ARRaycastQuery *query = [_arView raycastQueryFromPoint:touchLocation
                                             allowingTarget:ARRaycastTargetExistingPlaneInfinite
                                                  alignment:ARRaycastTargetAlignmentHorizontal] API_AVAILABLE(ios(14.0));
    
    if (query) {
        NSArray<ARRaycastResult *> *results = [_arView.session raycast:query];
        if (results.count > 0) {
            // วาง anchor ที่ตำแหน่ง
            ARAnchor *anchor = [[ARAnchor alloc] initWithTransform:results.firstObject.worldTransform];
            [_arView.session addAnchor:anchor];
        }
    }
}

// สร้าง SpriteKit node สำหรับ anchor
- (SKNode *)view:(ARSKView *)view nodeForAnchor:(ARAnchor *)anchor {
    // สร้าง emoji label
    SKLabelNode *label = [[SKLabelNode alloc] initWithFontNamed:@"System"];
    label.text = @"🌟";
    label.fontSize = 50;
    label.horizontalAlignmentMode = SKLabelHorizontalAlignmentModeCenter;
    label.verticalAlignmentMode = SKLabelVerticalAlignmentModeCenter;
    
    // Animation
    SKAction *rotate = [SKAction rotateByAngle:M_PI * 2 duration:2.0];
    SKAction *pulse = [SKAction sequence:@[
        [SKAction scaleBy:1.5 duration:0.5],
        [SKAction scaleBy:1.0/1.5 duration:0.5]
    ]];
    SKAction *combined = [SKAction group:@[
        [SKAction repeatActionForever:rotate],
        [SKAction repeatActionForever:pulse]
    ]];
    [label runAction:combined];
    
    return label;
}

@end
```

---

## 87.6 Plane Detection

### การจัดการ Horizontal และ Vertical Planes

```objc
@implementation PlaneDetectionManager {
    NSMutableDictionary<NSUUID *, SCNNode *> *_planeNodes;
}

- (void)renderer:(id<SCNSceneRenderer>)renderer 
      didAddNode:(SCNNode *)node 
       forAnchor:(ARAnchor *)anchor {
    
    if (![anchor isKindOfClass:[ARPlaneAnchor class]]) return;
    
    ARPlaneAnchor *planeAnchor = (ARPlaneAnchor *)anchor;
    
    // สร้าง visualization สำหรับ plane ใหม่
    SCNNode *planeVisualization = [self createPlaneVisualization:planeAnchor];
    [node addChildNode:planeVisualization];
    
    _planeNodes[anchor.identifier] = planeVisualization;
    
    // แจ้งเตือนผู้ใช้
    dispatch_async(dispatch_get_main_queue(), ^{
        [self showMessage:[NSString stringWithFormat:@"พบระนาบ%@",
            planeAnchor.alignment == ARPlaneAnchorAlignmentHorizontal 
                ? @"แนวนอน" : @"แนวตั้ง"]];
    });
}

- (void)renderer:(id<SCNSceneRenderer>)renderer 
   didUpdateNode:(SCNNode *)node 
       forAnchor:(ARAnchor *)anchor {
    
    if (![anchor isKindOfClass:[ARPlaneAnchor class]]) return;
    
    ARPlaneAnchor *planeAnchor = (ARPlaneAnchor *)anchor;
    SCNNode *planeVisualization = _planeNodes[anchor.identifier];
    
    if (!planeVisualization) return;
    
    // อัปเดตขนาดและ geometry
    [self updatePlaneVisualization:planeVisualization forAnchor:planeAnchor];
}

- (void)renderer:(id<SCNSceneRenderer>)renderer 
   didRemoveNode:(SCNNode *)node 
       forAnchor:(ARAnchor *)anchor {
    
    // ลบ plane visualization เมื่อ planes merge กัน
    [_planeNodes removeObjectForKey:anchor.identifier];
}

// สร้าง plane visualization แบบ grid
- (SCNNode *)createPlaneVisualization:(ARPlaneAnchor *)anchor {
    // สร้าง geometry จาก ARPlaneGeometry
    ARSCNPlaneGeometry *planeGeometry = [ARSCNPlaneGeometry planeGeometryWithDevice:
        [MTLCreateSystemDefaultDevice() autorelease]];
    [planeGeometry updateFromPlaneGeometry:anchor.geometry];
    
    SCNMaterial *material = [SCNMaterial material];
    material.diffuse.contents = [UIColor colorWithRed:0.0 green:0.5 blue:1.0 alpha:0.3];
    material.doubleSided = YES;
    material.blendMode = SCNBlendModeAlpha;
    planeGeometry.materials = @[material];
    
    return [SCNNode nodeWithGeometry:planeGeometry];
}

- (void)updatePlaneVisualization:(SCNNode *)node forAnchor:(ARPlaneAnchor *)anchor {
    ARSCNPlaneGeometry *planeGeometry = (ARSCNPlaneGeometry *)node.geometry;
    [planeGeometry updateFromPlaneGeometry:anchor.geometry];
}

@end
```

---

## 87.7 Image Tracking

### การ Track รูปภาพในโลกจริง

```objc
- (void)setupImageTracking {
    ARImageTrackingConfiguration *config = [[ARImageTrackingConfiguration alloc] init];
    
    // โหลด reference images จาก AR Resources group ใน Assets.xcassets
    NSSet<ARReferenceImage *> *referenceImages = [ARReferenceImage 
        referenceImagesInGroupNamed:@"AR Resources" 
                              bundle:NSBundle.mainBundle];
    
    config.trackingImages = referenceImages;
    config.maximumNumberOfTrackedImages = 5;  // จำนวน images ที่ track พร้อมกัน
    
    [_session runWithConfiguration:config];
}

// หรือสร้าง reference image แบบ programmatic
- (void)addCustomReferenceImage {
    UIImage *markerImage = [UIImage imageNamed:@"my_marker"];
    CGImageRef cgImage = markerImage.CGImage;
    
    // กำหนดขนาดจริงของรูปภาพ (หน่วยเมตร)
    ARReferenceImage *referenceImage = [[ARReferenceImage alloc] 
        initWithCGImage:cgImage 
             orientation:kCGImagePropertyOrientationUp 
     physicalWidth:0.1];  // 10 เซนติเมตร
    referenceImage.name = @"custom_marker";
    
    // เพิ่มเข้า configuration
    ARImageTrackingConfiguration *config = (ARImageTrackingConfiguration *)_session.configuration;
    NSMutableSet *images = [NSMutableSet setWithSet:config.trackingImages];
    [images addObject:referenceImage];
    config.trackingImages = images;
}

// ARSCNViewDelegate - จัดการ image tracking
- (void)renderer:(id<SCNSceneRenderer>)renderer 
      didAddNode:(SCNNode *)node 
       forAnchor:(ARAnchor *)anchor {
    
    if (![anchor isKindOfClass:[ARImageAnchor class]]) return;
    
    ARImageAnchor *imageAnchor = (ARImageAnchor *)anchor;
    ARReferenceImage *refImage = imageAnchor.referenceImage;
    
    NSLog(@"พบรูปภาพ: %@ (%.0f x %.0f cm)", 
          refImage.name,
          refImage.physicalSize.width * 100,
          refImage.physicalSize.height * 100);
    
    // วาง 3D content บนรูปภาพที่พบ
    dispatch_async(dispatch_get_main_queue(), ^{
        SCNNode *contentNode = [self createContentForImage:refImage.name];
        [node addChildNode:contentNode];
    });
}

- (SCNNode *)createContentForImage:(NSString *)imageName {
    if ([imageName isEqualToString:@"product_poster"]) {
        // แสดงข้อมูลสินค้าเหนือโปสเตอร์
        return [self createProductInfoNode];
    } else if ([imageName isEqualToString:@"business_card"]) {
        // แสดง 3D model เหนือนามบัตร
        return [self create3DModelNode];
    }
    return [SCNNode node];
}

@end
```

---

## 87.8 Face Tracking

### ARFaceTrackingConfiguration

```objc
@implementation FaceTrackingViewController {
    ARSCNView *_sceneView;
    SCNNode *_faceNode;
}

- (void)setupFaceTracking {
    // ตรวจสอบว่ารองรับ face tracking
    if (!ARFaceTrackingConfiguration.isSupported) {
        NSLog(@"อุปกรณ์นี้ไม่รองรับ face tracking");
        return;
    }
    
    ARFaceTrackingConfiguration *config = [[ARFaceTrackingConfiguration alloc] init];
    config.isLightEstimationEnabled = YES;
    config.maximumNumberOfTrackedFaces = 1;  // จำนวนใบหน้าที่ track พร้อมกัน (iOS 13+)
    
    [_session runWithConfiguration:config];
}

// จัดการ face anchor
- (void)renderer:(id<SCNSceneRenderer>)renderer 
      didAddNode:(SCNNode *)node 
       forAnchor:(ARAnchor *)anchor {
    
    if (![anchor isKindOfClass:[ARFaceAnchor class]]) return;
    
    ARFaceAnchor *faceAnchor = (ARFaceAnchor *)anchor;
    
    // สร้าง face mesh
    ARSCNFaceGeometry *faceGeometry = [ARSCNFaceGeometry faceGeometryWithDevice:
        _sceneView.device fillMesh:YES];
    
    SCNMaterial *material = [SCNMaterial material];
    material.diffuse.contents = [UIColor colorWithRed:1.0 green:0.8 blue:0.7 alpha:0.7];
    material.doubleSided = YES;
    faceGeometry.materials = @[material];
    
    _faceNode = [SCNNode nodeWithGeometry:faceGeometry];
    [node addChildNode:_faceNode];
    
    // เพิ่ม AR accessories
    [self addGlassesTo:node];
    [self addHatTo:node];
}

// อัปเดต face mesh
- (void)renderer:(id<SCNSceneRenderer>)renderer 
   didUpdateNode:(SCNNode *)node 
       forAnchor:(ARAnchor *)anchor {
    
    if (![anchor isKindOfClass:[ARFaceAnchor class]]) return;
    
    ARFaceAnchor *faceAnchor = (ARFaceAnchor *)anchor;
    
    // อัปเดต face geometry
    ARSCNFaceGeometry *faceGeometry = (ARSCNFaceGeometry *)_faceNode.geometry;
    [faceGeometry updateFromFaceGeometry:faceAnchor.geometry];
    
    // อ่าน blend shapes (facial expressions)
    NSDictionary<ARBlendShapeLocation, NSNumber *> *blendShapes = faceAnchor.blendShapes;
    
    float mouthOpen = [blendShapes[ARBlendShapeLocationMouthClose] floatValue];
    float leftEyeBlinkLeft = [blendShapes[ARBlendShapeLocationEyeBlinkLeft] floatValue];
    float rightEyeBlinkRight = [blendShapes[ARBlendShapeLocationEyeBlinkRight] floatValue];
    float browRaise = [blendShapes[ARBlendShapeLocationBrowInnerUp] floatValue];
    
    // ตอบสนองต่อ facial expressions
    if (mouthOpen > 0.5) {
        [self triggerMouthOpenEffect];
    }
    
    if (leftEyeBlinkLeft > 0.8 && rightEyeBlinkRight < 0.2) {
        [self triggerWinkEffect];
    }
    
    // Log expression values
    dispatch_async(dispatch_get_main_queue(), ^{
        self->_expressionLabel.text = [NSString stringWithFormat:
            @"ปาก: %.2f | ตาซ้าย: %.2f | ตาขวา: %.2f",
            mouthOpen, leftEyeBlinkLeft, rightEyeBlinkRight];
    });
}

- (void)addGlassesTo:(SCNNode *)faceNode {
    // โหลด glasses model
    SCNScene *glassesScene = [SCNScene sceneNamed:@"glasses.scn"];
    SCNNode *glassesNode = glassesScene.rootNode.childNodes.firstObject;
    
    // ปรับตำแหน่งให้อยู่บนจมูก
    glassesNode.position = SCNVector3Make(0, -0.01, 0.08);
    glassesNode.scale = SCNVector3Make(1.0, 1.0, 1.0);
    
    [faceNode addChildNode:glassesNode];
}

@end
```

### Blend Shapes ที่ใช้บ่อย

```objc
// Blend Shape locations ที่มีใน ARKit
NSString *blendShapeExamples = @"
ARBlendShapeLocationBrowDownLeft         - คิ้วซ้ายลง
ARBlendShapeLocationBrowDownRight        - คิ้วขวาลง
ARBlendShapeLocationBrowInnerUp          - คิ้วกลางขึ้น
ARBlendShapeLocationBrowOuterUpLeft      - คิ้วซ้ายนอกขึ้น
ARBlendShapeLocationBrowOuterUpRight     - คิ้วขวานอกขึ้น
ARBlendShapeLocationCheekPuff            - แก้มพอง
ARBlendShapeLocationCheekSquintLeft      - แก้มซ้ายหรี่
ARBlendShapeLocationCheekSquintRight     - แก้มขวาหรี่
ARBlendShapeLocationEyeBlinkLeft         - ตาซ้ายกระพริบ
ARBlendShapeLocationEyeBlinkRight        - ตาขวากระพริบ
ARBlendShapeLocationEyeLookDownLeft      - มองล่างซ้าย
ARBlendShapeLocationEyeLookInLeft        - มองซ้ายในซ้าย
ARBlendShapeLocationEyeLookOutLeft       - มองนอกซ้าย
ARBlendShapeLocationEyeLookUpLeft        - มองบนซ้าย
ARBlendShapeLocationJawOpen              - อ้าปาก
ARBlendShapeLocationMouthClose           - ปิดปาก
ARBlendShapeLocationMouthSmileLeft       - ยิ้มซ้าย
ARBlendShapeLocationMouthSmileRight      - ยิ้มขวา
ARBlendShapeLocationTongueOut            - แลบลิ้น
";
```

---

## 87.9 Object Detection และ Occlusion

### ARObjectAnchor - การตรวจจับ 3D Objects

```objc
- (void)setupObjectDetection {
    ARWorldTrackingConfiguration *config = [[ARWorldTrackingConfiguration alloc] init];
    
    // โหลด reference objects จาก Assets
    NSSet<ARReferenceObject *> *referenceObjects = [ARReferenceObject 
        referenceObjectsInGroupNamed:@"AR Objects" 
                               bundle:NSBundle.mainBundle];
    
    config.detectionObjects = referenceObjects;
    config.planeDetection = ARPlaneDetectionHorizontal;
    
    [_session runWithConfiguration:config];
}

// ตรวจจับ object ได้แล้ว
- (void)renderer:(id<SCNSceneRenderer>)renderer 
      didAddNode:(SCNNode *)node 
       forAnchor:(ARAnchor *)anchor {
    
    if (![anchor isKindOfClass:[ARObjectAnchor class]]) return;
    
    ARObjectAnchor *objectAnchor = (ARObjectAnchor *)anchor;
    ARReferenceObject *refObject = objectAnchor.referenceObject;
    
    NSLog(@"พบ object: %@", refObject.name);
    
    // วาง virtual object รอบๆ ของจริง
    SCNNode *infoPanel = [self createInfoPanelFor:refObject.name];
    [node addChildNode:infoPanel];
    
    // เพิ่ม highlight effect
    [self addHighlightToObject:node];
}

// People Occlusion - iOS 13+
- (void)enablePeopleOcclusion {
    if (@available(iOS 13.0, *)) {
        ARWorldTrackingConfiguration *config = 
            (ARWorldTrackingConfiguration *)_session.configuration;
        
        if (ARWorldTrackingConfiguration.supportsFrameSemantics & 
            ARFrameSemanticPersonSegmentationWithDepth) {
            config.frameSemantics |= ARFrameSemanticPersonSegmentationWithDepth;
            [_session runWithConfiguration:config];
            NSLog(@"People occlusion เปิดใช้งาน");
        }
    }
}

@end
```

---

## 87.10 ARReferenceObject - การสร้าง Reference Object

### การสร้าง Reference Object จากโค้ด

```objc
// การ scan object เพื่อสร้าง reference
@implementation ObjectScanningViewController {
    ARObjectScanningConfiguration *_scanConfig;
    ARWorldMap *_scanWorldMap;
}

- (void)startScanning {
    _scanConfig = [[ARObjectScanningConfiguration alloc] init];
    _scanConfig.planeDetection = ARPlaneDetectionHorizontal;
    
    [_session runWithConfiguration:_scanConfig];
    
    NSLog(@"เริ่ม scan object - เดินรอบๆ วัตถุ");
}

- (void)captureReferenceObject {
    // กำหนดพื้นที่ scan
    simd_float3 center = simd_make_float3(0, 0, 0);
    simd_float3 extent = simd_make_float3(0.2, 0.2, 0.2);  // 20x20x20 cm
    
    [_session createReferenceObjectWithTransform:matrix_identity_float4x4
                                           center:center
                                           extent:extent
                               completionHandler:^(ARReferenceObject *object, NSError *error) {
        if (error) {
            NSLog(@"Scan error: %@", error);
            return;
        }
        
        // บันทึก reference object
        NSString *path = [NSTemporaryDirectory() 
            stringByAppendingPathComponent:@"scanned_object.arobject"];
        NSURL *url = [NSURL fileURLWithPath:path];
        
        [object exportObjectToURL:url 
                previewImage:nil 
           completionHandler:^(NSError *exportError) {
            if (!exportError) {
                NSLog(@"บันทึก reference object แล้วที่: %@", path);
            }
        }];
    }];
}

@end
```

---

## 87.11 RealityKit Basics

### การใช้ RealityKit กับ ARKit

```objc
#import <RealityKit/RealityKit.h>

@interface RealityKitViewController : UIViewController

@end

@implementation RealityKitViewController {
    ARView *_arView;
}

- (void)viewDidLoad {
    [super viewDidLoad];
    
    // ARView จาก RealityKit (แตกต่างจาก ARSCNView)
    _arView = [[ARView alloc] initWithFrame:self.view.bounds];
    _arView.autoresizingMask = UIViewAutoresizingFlexibleWidth | UIViewAutoresizingFlexibleHeight;
    [self.view addSubview:_arView];
    
    // ตั้งค่า AR session
    ARWorldTrackingConfiguration *config = [[ARWorldTrackingConfiguration alloc] init];
    config.planeDetection = ARPlaneDetectionHorizontal;
    [_arView.session runWithConfiguration:config];
    
    // ตั้งค่า debug options
    _arView.debugOptions = ARViewDebugOptionShowFeaturePoints | ARViewDebugOptionShowAnchorOrigins;
    
    // Enable coaching
    ARCoachingOverlayView *coaching = [[ARCoachingOverlayView alloc] init];
    coaching.autoresizingMask = UIViewAutoresizingFlexibleWidth | UIViewAutoresizingFlexibleHeight;
    coaching.session = _arView.session;
    coaching.goal = ARCoachingOverlayViewGoalHorizontalPlane;
    [_arView addSubview:coaching];
    [coaching setActive:YES animated:YES];
    
    // Tap to place
    UITapGestureRecognizer *tap = [[UITapGestureRecognizer alloc] 
        initWithTarget:self action:@selector(handleTap:)];
    [_arView addGestureRecognizer:tap];
}

- (void)handleTap:(UITapGestureRecognizer *)recognizer {
    CGPoint point = [recognizer locationInView:_arView];
    
    // Raycast
    NSArray<ARRaycastResult *> *results = [_arView raycast:
        [_arView makeRaycastQueryFromPoint:point 
                             allowingTarget:ARRaycastTargetExistingPlaneGeometry
                                  alignment:ARRaycastTargetAlignmentHorizontal]];
    
    if (results.count > 0) {
        [self placeEntityAt:results.firstObject.worldTransform];
    }
}

- (void)placeEntityAt:(simd_float4x4)transform {
    // โหลด USDZ model
    NSURL *url = [[NSBundle mainBundle] URLForResource:@"model" 
                                        withExtension:@"usdz"];
    ModelEntity *model = [ModelEntity loadModelFromURL:url error:nil];
    
    // ปรับขนาด
    model.scale = simd_make_float3(0.1, 0.1, 0.1);
    
    // สร้าง anchor
    AnchorEntity *anchor = [[AnchorEntity alloc] 
        initWithWorld:simd_make_float4x4(simd_matrix_identity_float4x4())];
    anchor.transform.matrix = transform;
    
    [anchor addChild:model];
    [_arView.scene addAnchor:anchor];
}

@end
```

### RealityKit Features

```objc
// Physics simulation
- (void)addPhysicsToModel:(ModelEntity *)model {
    // Collision shape
    model.collision = [[CollisionComponent alloc] 
        initWithShapes:@[[ShapeResource generateBoxWithSize:simd_make_float3(0.1, 0.1, 0.1)]]
                  mode:CollisionComponentModeDefault
                filter:CollisionFilter.default_];
    
    // Physics body
    PhysicsBodyComponent *physics = [[PhysicsBodyComponent alloc] 
        initWithMassProperties:[PhysicsMassProperties massPropertiesWithShape:[ShapeResource generateBoxWithSize:simd_make_float3(0.1, 0.1, 0.1)] mass:0.1]
                      material:[PhysicsMaterialResource generateWithFriction:0.8 restitution:0.4]
                          mode:PhysicsBodyModeDynamic];
    model.physicsBody = physics;
}

// Animation
- (void)animateModel:(ModelEntity *)model {
    // สร้าง animation clip
    FromToByAnimation *animation = [[FromToByAnimation alloc] 
        initWithName:@"spin"
                from:@(simd_quaternion(0, simd_make_float3(0, 1, 0)))
                  to:@(simd_quaternion(2 * M_PI, simd_make_float3(0, 1, 0)))
           duration:2.0
          bindTarget:[AnimationBindTarget target:model property:@"transform/rotation"]
             repeats:YES
           fillMode:AnimationFillModeForwards
         trimStart:nil
           trimEnd:nil
         trimDuration:nil
              offset:0
              delay:0
              speed:1.0];
    
    [model playAnimation:[animation resolveWithPreamble:AnimationView.makeAnimation]];
}
```

---

## 87.12 Body Pose Estimation

### ARBodyTrackingConfiguration

```objc
- (void)setupBodyTracking {
    if (!ARBodyTrackingConfiguration.isSupported) {
        NSLog(@"Body tracking ไม่รองรับ");
        return;
    }
    
    ARBodyTrackingConfiguration *config = [[ARBodyTrackingConfiguration alloc] init];
    [_session runWithConfiguration:config];
}

- (void)renderer:(id<SCNSceneRenderer>)renderer 
      didAddNode:(SCNNode *)node 
       forAnchor:(ARAnchor *)anchor {
    
    if (![anchor isKindOfClass:[ARBodyAnchor class]]) return;
    
    ARBodyAnchor *bodyAnchor = (ARBodyAnchor *)anchor;
    
    // สร้าง skeleton visualization
    [self createSkeletonFor:bodyAnchor inNode:node];
}

- (void)renderer:(id<SCNSceneRenderer>)renderer 
   didUpdateNode:(SCNNode *)node 
       forAnchor:(ARAnchor *)anchor {
    
    if (![anchor isKindOfClass:[ARBodyAnchor class]]) return;
    
    ARBodyAnchor *bodyAnchor = (ARBodyAnchor *)anchor;
    ARSkeleton3D *skeleton = bodyAnchor.skeleton;
    
    // อ่าน joint positions
    NSDictionary<NSString *, NSValue *> *localTransforms = [skeleton jointLocalTransforms];
    
    // Joint names ที่ใช้บ่อย
    // "root" - จุดกึ่งกลางสะโพก
    // "spine_1_joint", "spine_2_joint", "spine_3_joint" - กระดูกสันหลัง
    // "left_shoulder_1_joint", "right_shoulder_1_joint" - ไหล่
    // "left_forearm_joint", "right_forearm_joint" - แขน
    // "left_hand_joint", "right_hand_joint" - มือ
    // "left_leg_joint", "right_leg_joint" - ขา
    // "left_foot_joint", "right_foot_joint" - เท้า
    
    simd_float4x4 leftHandTransform = [skeleton localTransformForJointName:
        ARSkeletonDefinition.defaultBody3DSkeletonDefinition.jointNames[20]];
    
    // ตรวจสอบท่าทาง
    [self analyzePose:skeleton];
}

- (void)analyzePose:(ARSkeleton3D *)skeleton {
    // ตรวจสอบว่ายกมือสูงกว่าหัวหรือไม่
    simd_float4x4 headTransform = [skeleton modelTransformForJointName:@"head_joint"];
    simd_float4x4 leftHandTransform = [skeleton modelTransformForJointName:@"left_hand_joint"];
    simd_float4x4 rightHandTransform = [skeleton modelTransformForJointName:@"right_hand_joint"];
    
    float headY = headTransform.columns[3].y;
    float leftHandY = leftHandTransform.columns[3].y;
    float rightHandY = rightHandTransform.columns[3].y;
    
    if (leftHandY > headY || rightHandY > headY) {
        NSLog(@"🙋 ยกมือสูงกว่าหัว!");
    }
}

@end
```

---

## 87.13 Multi-user AR (Collaboration)

### การแชร์ AR Session ระหว่างอุปกรณ์

```objc
// ต้องใช้ MultipeerConnectivity framework
#import <MultipeerConnectivity/MultipeerConnectivity.h>

@interface MultiUserARViewController : UIViewController 
    <ARSessionDelegate, MCSessionDelegate>

@end

@implementation MultiUserARViewController {
    MCSession *_multipeerSession;
    MCPeerID *_peerID;
    MCNearbyServiceBrowser *_browser;
    MCNearbyServiceAdvertiser *_advertiser;
}

- (void)setupMultipeerConnectivity {
    _peerID = [[MCPeerID alloc] initWithDisplayName:UIDevice.currentDevice.name];
    _multipeerSession = [[MCSession alloc] initWithPeer:_peerID
                                       securityIdentity:nil
                                   encryptionPreference:MCEncryptionRequired];
    _multipeerSession.delegate = self;
    
    // advertise
    _advertiser = [[MCNearbyServiceAdvertiser alloc] 
        initWithPeer:_peerID 
       discoveryInfo:nil 
         serviceType:@"ar-collab"];
    [_advertiser startAdvertisingPeer];
    
    // browse
    _browser = [[MCNearbyServiceBrowser alloc] 
        initWithPeer:_peerID 
         serviceType:@"ar-collab"];
    [_browser startBrowsingForPeers];
}

// ส่ง AR collaboration data
- (void)session:(ARSession *)session 
    didOutputCollaborationData:(ARCollaborationData *)data {
    
    if (!_multipeerSession.connectedPeers.count) return;
    
    // Serialize และส่งไปยัง peers ทั้งหมด
    NSData *encodedData = [NSKeyedArchiver archivedDataWithRootObject:data 
                                               requiringSecureCoding:YES 
                                                               error:nil];
    
    NSError *error;
    [_multipeerSession sendData:encodedData 
                        toPeers:_multipeerSession.connectedPeers 
                       withMode:MCSessionSendDataReliable 
                          error:&error];
}

// รับ collaboration data จาก peers
- (void)session:(MCSession *)session 
   didReceiveData:(NSData *)data 
         fromPeer:(MCPeerID *)peerID {
    
    ARCollaborationData *collaborationData = [NSKeyedUnarchiver 
        unarchivedObjectOfClass:[ARCollaborationData class] 
                       fromData:data 
                          error:nil];
    
    if (collaborationData) {
        [_arSession updateWithCollaborationData:collaborationData];
    }
}

@end
```

---

## 87.14 ARCoachingOverlayView

### การแสดง Coaching UI

```objc
- (void)setupCoachingOverlay {
    ARCoachingOverlayView *coaching = [[ARCoachingOverlayView alloc] init];
    coaching.frame = self.view.bounds;
    coaching.autoresizingMask = UIViewAutoresizingFlexibleWidth | UIViewAutoresizingFlexibleHeight;
    
    // เชื่อมกับ AR session
    coaching.session = _sceneView.session;
    coaching.delegate = self;
    
    // กำหนดเป้าหมาย
    coaching.goal = ARCoachingOverlayViewGoalHorizontalPlane;
    // Options:
    // ARCoachingOverlayViewGoalTracking - tracking ทั่วไป
    // ARCoachingOverlayViewGoalHorizontalPlane - หา horizontal plane
    // ARCoachingOverlayViewGoalVerticalPlane - หา vertical plane
    // ARCoachingOverlayViewGoalAnyPlane - หา plane ใดก็ได้
    // ARCoachingOverlayViewGoalGeoTracking - location tracking
    
    coaching.activatesAutomatically = YES;  // แสดงเองเมื่อ tracking ไม่ดี
    
    [_sceneView addSubview:coaching];
}

// ARCoachingOverlayViewDelegate
- (void)coachingOverlayViewWillActivate:(ARCoachingOverlayView *)coachingOverlayView {
    // ซ่อน UI ที่ไม่จำเป็น
}

- (void)coachingOverlayViewDidDeactivate:(ARCoachingOverlayView *)coachingOverlayView {
    // แสดง UI กลับมา - tracking พร้อมแล้ว
    [self placeInitialContent];
}

@end
```

---

## 87.15 แบบฝึกหัด

### แบบฝึกหัดที่ 1: AR Furniture Placement App

```objc
// สร้างแอปวางเฟอร์นิเจอร์ใน AR ที่:
// 1. ตรวจจับ horizontal planes
// 2. ผู้ใช้เลือกเฟอร์นิเจอร์จาก collection view
// 3. แตะพื้นเพื่อวาง 3D model
// 4. หมุนและปรับขนาดด้วย gestures
// 5. ลบ object ด้วยการแตะค้าง

@interface FurnitureARViewController : UIViewController <ARSCNViewDelegate>

@end

@implementation FurnitureARViewController {
    ARSCNView *_sceneView;
    NSString *_selectedFurniture;
    SCNNode *_selectedNode;
}

- (void)setupGestures {
    // Tap - วางของ
    UITapGestureRecognizer *tap = [[UITapGestureRecognizer alloc] 
        initWithTarget:self action:@selector(handleTap:)];
    [_sceneView addGestureRecognizer:tap];
    
    // Pan - เลื่อนของ
    UIPanGestureRecognizer *pan = [[UIPanGestureRecognizer alloc] 
        initWithTarget:self action:@selector(handlePan:)];
    [_sceneView addGestureRecognizer:pan];
    
    // Rotation - หมุนของ
    UIRotationGestureRecognizer *rotation = [[UIRotationGestureRecognizer alloc] 
        initWithTarget:self action:@selector(handleRotation:)];
    [_sceneView addGestureRecognizer:rotation];
    
    // Pinch - ปรับขนาด
    UIPinchGestureRecognizer *pinch = [[UIPinchGestureRecognizer alloc] 
        initWithTarget:self action:@selector(handlePinch:)];
    [_sceneView addGestureRecognizer:pinch];
    
    // Long press - ลบ
    UILongPressGestureRecognizer *longPress = [[UILongPressGestureRecognizer alloc] 
        initWithTarget:self action:@selector(handleLongPress:)];
    [_sceneView addGestureRecognizer:longPress];
}

- (void)handleRotation:(UIRotationGestureRecognizer *)recognizer {
    if (!_selectedNode) return;
    
    if (recognizer.state == UIGestureRecognizerStateChanged) {
        _selectedNode.eulerAngles = SCNVector3Make(
            _selectedNode.eulerAngles.x,
            _selectedNode.eulerAngles.y + recognizer.rotation,
            _selectedNode.eulerAngles.z
        );
        recognizer.rotation = 0;
    }
}

- (void)handlePinch:(UIPinchGestureRecognizer *)recognizer {
    if (!_selectedNode) return;
    
    if (recognizer.state == UIGestureRecognizerStateChanged) {
        float scale = recognizer.scale;
        _selectedNode.scale = SCNVector3Make(
            _selectedNode.scale.x * scale,
            _selectedNode.scale.y * scale,
            _selectedNode.scale.z * scale
        );
        recognizer.scale = 1.0;
    }
}

@end
```

### แบบฝึกหัดที่ 2: AR Face Filter App

```
สร้างแอป face filter ที่:
1. ใช้ ARFaceTrackingConfiguration
2. ตรวจจับใบหน้าและวาด face mesh
3. วาง 3D accessories (แว่นตา, หมวก, หนวด)
4. ตอบสนองต่อ facial expressions:
   - ยิ้ม → แสดง heart particles
   - อ้าปาก → แสดง text bubble
   - กระพริบตา → เปลี่ยน filter
5. ปุ่ม Screenshot เพื่อบันทึกภาพ
```

### แบบฝึกหัดที่ 3: AR Measurement Tool

```objc
// แอปวัดระยะทางด้วย AR
@implementation MeasurementARViewController {
    ARSCNView *_sceneView;
    NSMutableArray<SCNNode *> *_measurePoints;
    SCNNode *_lineNode;
    UILabel *_distanceLabel;
}

- (void)measureDistance {
    if (_measurePoints.count < 2) return;
    
    SCNNode *start = _measurePoints[0];
    SCNNode *end = _measurePoints[1];
    
    // คำนวณระยะทาง
    SCNVector3 p1 = start.worldPosition;
    SCNVector3 p2 = end.worldPosition;
    
    float dx = p2.x - p1.x;
    float dy = p2.y - p1.y;
    float dz = p2.z - p1.z;
    float distance = sqrt(dx*dx + dy*dy + dz*dz);
    
    // แปลงเป็น cm
    float distanceCM = distance * 100;
    
    dispatch_async(dispatch_get_main_queue(), ^{
        self->_distanceLabel.text = [NSString stringWithFormat:@"%.1f cm", distanceCM];
    });
    
    // วาดเส้นระหว่างจุด
    [self drawLineBetween:p1 and:p2];
}

- (void)drawLineBetween:(SCNVector3)start and:(SCNVector3)end {
    SCNVector3 middle = SCNVector3Make(
        (start.x + end.x) / 2,
        (start.y + end.y) / 2,
        (start.z + end.z) / 2
    );
    
    float dx = end.x - start.x;
    float dy = end.y - start.y;
    float dz = end.z - start.z;
    float length = sqrt(dx*dx + dy*dy + dz*dz);
    
    SCNCylinder *cylinder = [SCNCylinder cylinderWithRadius:0.002 height:length];
    cylinder.firstMaterial.diffuse.contents = [UIColor yellowColor];
    
    _lineNode = [SCNNode nodeWithGeometry:cylinder];
    _lineNode.position = middle;
    
    // หมุนให้ตรงกับทิศทาง
    SCNVector3 direction = SCNVector3Make(dx/length, dy/length, dz/length);
    _lineNode.rotation = [self rotationFromDirection:direction];
    
    [_sceneView.scene.rootNode addChildNode:_lineNode];
}

@end
```

---

## สรุป

ในบทนี้เราได้เรียนรู้:

- **ARKit Overview**: สถาปัตยกรรมและ tracking state
- **ARSession**: การสร้างและจัดการ AR session
- **ARWorldTracking**: การ track สภาพแวดล้อมจริง
- **ARSCNView**: การ integrate กับ SceneKit
- **ARSKView**: การ integrate กับ SpriteKit
- **Plane Detection**: การตรวจจับ horizontal/vertical planes
- **Image Tracking**: การ track รูปภาพในโลกจริง
- **Face Tracking**: การ track ใบหน้าและ blend shapes
- **Object Detection**: การตรวจจับ 3D objects
- **RealityKit**: framework AR รุ่นใหม่จาก Apple
- **Body Tracking**: การ track ท่าทางของร่างกาย
- **Multi-user AR**: การแชร์ AR experience

บทต่อไปจะเรียนรู้เกี่ยวกับ **Core ML และ Vision Framework** สำหรับ Machine Learning บน Apple devices
