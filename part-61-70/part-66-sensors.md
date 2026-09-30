# ตอนที่ 66: Sensors และ Hardware ใน Objective-C

## บทนำ

iOS มี sensors และ hardware หลากหลายประเภทที่สามารถใช้งานผ่าน API ที่ Apple จัดเตรียมไว้ ในตอนนี้เราจะเรียนรู้การใช้งาน CoreMotion framework สำหรับ motion sensors รวมถึง hardware อื่นๆ เช่น กล้อง ไฟฉาย และ proximity sensor

---

## 66.1 CoreMotion Framework

CoreMotion เป็น framework สำหรับเข้าถึง motion sensors ทั้งหมดของอุปกรณ์

### การเพิ่ม CoreMotion

```objc
#import <CoreMotion/CoreMotion.h>
```

### CMMotionManager

`CMMotionManager` คือคลาสหลักสำหรับ motion data ทั้งหมด ควรสร้างเพียง instance เดียว

```objc
// MotionViewController.h
#import <UIKit/UIKit.h>
#import <CoreMotion/CoreMotion.h>

@interface MotionViewController : UIViewController

@property (nonatomic, strong) CMMotionManager *motionManager;

@end
```

```objc
// MotionViewController.m
#import "MotionViewController.h"

@implementation MotionViewController

- (void)viewDidLoad {
    [super viewDidLoad];
    
    // สร้าง CMMotionManager (ควรสร้างครั้งเดียว)
    self.motionManager = [[CMMotionManager alloc] init];
    
    NSLog(@"Accelerometer รองรับ: %@", 
          self.motionManager.isAccelerometerAvailable ? @"ใช่" : @"ไม่");
    NSLog(@"Gyroscope รองรับ: %@", 
          self.motionManager.isGyroAvailable ? @"ใช่" : @"ไม่");
    NSLog(@"Magnetometer รองรับ: %@", 
          self.motionManager.isMagnetometerAvailable ? @"ใช่" : @"ไม่");
    NSLog(@"Device Motion รองรับ: %@", 
          self.motionManager.isDeviceMotionAvailable ? @"ใช่" : @"ไม่");
}

- (void)viewWillDisappear:(BOOL)animated {
    [super viewWillDisappear:animated];
    
    // หยุด sensors เมื่อ view หายไป
    [self stopAllMotionUpdates];
}

- (void)stopAllMotionUpdates {
    [self.motionManager stopAccelerometerUpdates];
    [self.motionManager stopGyroUpdates];
    [self.motionManager stopMagnetometerUpdates];
    [self.motionManager stopDeviceMotionUpdates];
}

@end
```

---

## 66.2 Accelerometer

Accelerometer วัดแรงเร่ง (acceleration) ในแกน X, Y, Z หน่วยเป็น g (9.8 m/s²)

```objc
- (void)startAccelerometer {
    if (!self.motionManager.isAccelerometerAvailable) {
        NSLog(@"Accelerometer ไม่รองรับ");
        return;
    }
    
    // ตั้งค่า update interval (วินาที)
    // 1/60 = 60 Hz (สำหรับเกม)
    // 1/10 = 10 Hz (สำหรับ UI)
    self.motionManager.accelerometerUpdateInterval = 1.0 / 60.0;
    
    NSOperationQueue *queue = [NSOperationQueue mainQueue];
    
    // เริ่ม updates แบบ block (push model)
    [self.motionManager startAccelerometerUpdatesToQueue:queue 
                                            withHandler:^(CMAccelerometerData *data, 
                                                          NSError *error) {
        if (error) {
            NSLog(@"Accelerometer error: %@", error);
            return;
        }
        
        CMAcceleration acceleration = data.acceleration;
        
        // X: ซ้าย/ขวา (portrait: ซ้าย=-1, ขวา=+1)
        // Y: บน/ล่าง (portrait: ล่าง=-1, บน=+1)
        // Z: หน้า/หลัง (หน้าจอหน้าเราข้างบน=+1)
        NSLog(@"X: %.3f, Y: %.3f, Z: %.3f", 
              acceleration.x, acceleration.y, acceleration.z);
        
        // ตรวจจับการเขย่า
        double magnitude = sqrt(
            acceleration.x * acceleration.x +
            acceleration.y * acceleration.y +
            acceleration.z * acceleration.z
        );
        
        if (magnitude > 2.5) {
            NSLog(@"ตรวจพบการเขย่า! ขนาด: %.2f g", magnitude);
        }
    }];
    
    NSLog(@"เริ่ม accelerometer updates");
}

- (void)stopAccelerometer {
    [self.motionManager stopAccelerometerUpdates];
    NSLog(@"หยุด accelerometer");
}

// Pull model - อ่านค่าปัจจุบันเลย
- (void)readAccelerometerOnce {
    if (self.motionManager.isAccelerometerActive) {
        CMAcceleration acc = self.motionManager.accelerometerData.acceleration;
        NSLog(@"Acceleration: X=%.3f, Y=%.3f, Z=%.3f", acc.x, acc.y, acc.z);
    }
}
```

### ตัวอย่าง: Level Meter (เครื่องมือวัดระดับ)

```objc
@interface LevelMeterViewController : UIViewController

@property (nonatomic, strong) CMMotionManager *motionManager;
@property (weak, nonatomic) IBOutlet UIView *bubbleView;
@property (weak, nonatomic) IBOutlet UILabel *angleLabel;

@end

@implementation LevelMeterViewController

- (void)viewDidLoad {
    [super viewDidLoad];
    self.motionManager = [[CMMotionManager alloc] init];
    self.motionManager.accelerometerUpdateInterval = 1.0 / 30.0;
}

- (void)viewDidAppear:(BOOL)animated {
    [super viewDidAppear:animated];
    
    [self.motionManager startAccelerometerUpdatesToQueue:[NSOperationQueue mainQueue] 
                                            withHandler:^(CMAccelerometerData *data, 
                                                          NSError *error) {
        CMAcceleration acc = data.acceleration;
        
        // คำนวณมุมเอียง
        double pitch = atan2(acc.y, acc.z) * 180.0 / M_PI;
        double roll = atan2(acc.x, acc.z) * 180.0 / M_PI;
        
        // แสดงมุม
        self.angleLabel.text = [NSString stringWithFormat:@"Pitch: %.1f° Roll: %.1f°", 
                                 pitch, roll];
        
        // ขยับ bubble
        CGFloat maxOffset = 100.0;
        CGFloat xOffset = (CGFloat)(acc.x) * maxOffset;
        CGFloat yOffset = -(CGFloat)(acc.y) * maxOffset;
        
        xOffset = MAX(-maxOffset, MIN(maxOffset, xOffset));
        yOffset = MAX(-maxOffset, MIN(maxOffset, yOffset));
        
        self.bubbleView.center = CGPointMake(
            self.view.center.x + xOffset,
            self.view.center.y + yOffset
        );
    }];
}

- (void)viewWillDisappear:(BOOL)animated {
    [super viewWillDisappear:animated];
    [self.motionManager stopAccelerometerUpdates];
}

@end
```

---

## 66.3 Gyroscope

Gyroscope วัดอัตราการหมุน (rotation rate) หน่วยเป็น radians/second

```objc
- (void)startGyroscope {
    if (!self.motionManager.isGyroAvailable) {
        NSLog(@"Gyroscope ไม่รองรับ");
        return;
    }
    
    self.motionManager.gyroUpdateInterval = 1.0 / 60.0;
    
    [self.motionManager startGyroUpdatesToQueue:[NSOperationQueue mainQueue] 
                                   withHandler:^(CMGyroData *data, NSError *error) {
        if (error) {
            NSLog(@"Gyro error: %@", error);
            return;
        }
        
        CMRotationRate rotationRate = data.rotationRate;
        
        // x: pitch (เอียงหน้า-หลัง)
        // y: roll (เอียงซ้าย-ขวา)
        // z: yaw (หมุนซ้าย-ขวาบนแกนตั้ง)
        NSLog(@"Gyro - X: %.3f, Y: %.3f, Z: %.3f rad/s", 
              rotationRate.x, rotationRate.y, rotationRate.z);
        
        // แปลงเป็น degrees/second
        double xDeg = rotationRate.x * 180.0 / M_PI;
        double yDeg = rotationRate.y * 180.0 / M_PI;
        double zDeg = rotationRate.z * 180.0 / M_PI;
        
        NSLog(@"Gyro degrees - X: %.1f°/s, Y: %.1f°/s, Z: %.1f°/s", 
              xDeg, yDeg, zDeg);
    }];
}

// ตัวอย่าง: วัดการหมุนสะสม
@interface RotationTracker : NSObject

@property (nonatomic, strong) CMMotionManager *motionManager;
@property (nonatomic, assign) double totalRotationZ;
@property (nonatomic, assign) NSTimeInterval lastTimestamp;

@end

@implementation RotationTracker

- (void)startTracking {
    self.totalRotationZ = 0.0;
    self.lastTimestamp = 0.0;
    self.motionManager.gyroUpdateInterval = 1.0 / 100.0;
    
    [self.motionManager startGyroUpdatesToQueue:[NSOperationQueue mainQueue] 
                                   withHandler:^(CMGyroData *data, NSError *error) {
        if (self.lastTimestamp > 0) {
            NSTimeInterval dt = data.timestamp - self.lastTimestamp;
            self.totalRotationZ += data.rotationRate.z * dt;
        }
        self.lastTimestamp = data.timestamp;
        
        double degrees = self.totalRotationZ * 180.0 / M_PI;
        NSLog(@"การหมุนสะสม: %.1f องศา", degrees);
    }];
}

@end
```

---

## 66.4 Magnetometer

Magnetometer (แมกนีโตมิเตอร์) วัดสนามแม่เหล็กในแกน X, Y, Z

```objc
- (void)startMagnetometer {
    if (!self.motionManager.isMagnetometerAvailable) {
        NSLog(@"Magnetometer ไม่รองรับ");
        return;
    }
    
    self.motionManager.magnetometerUpdateInterval = 1.0 / 10.0;
    
    [self.motionManager startMagnetometerUpdatesToQueue:[NSOperationQueue mainQueue] 
                                           withHandler:^(CMMagnetometerData *data, 
                                                         NSError *error) {
        if (error) {
            NSLog(@"Magnetometer error: %@", error);
            return;
        }
        
        CMMagneticField field = data.magneticField;
        
        // หน่วย microteslas (μT)
        NSLog(@"Magnetic field - X: %.2f, Y: %.2f, Z: %.2f μT", 
              field.x, field.y, field.z);
        
        // คำนวณทิศทาง (compass heading)
        double heading = atan2(field.y, field.x) * 180.0 / M_PI;
        if (heading < 0) heading += 360.0;
        
        NSLog(@"ทิศทางแม่เหล็ก: %.1f องศา", heading);
    }];
}
```

---

## 66.5 Device Motion

Device Motion รวม accelerometer + gyroscope + magnetometer เข้าด้วยกัน ให้ข้อมูลที่ถูกต้องและ stable กว่า

```objc
- (void)startDeviceMotion {
    if (!self.motionManager.isDeviceMotionAvailable) {
        NSLog(@"Device Motion ไม่รองรับ");
        return;
    }
    
    self.motionManager.deviceMotionUpdateInterval = 1.0 / 60.0;
    
    // เลือก reference frame
    // CMAttitudeReferenceFrameXArbitraryZVertical - default
    // CMAttitudeReferenceFrameXArbitraryCorrectedZVertical - แก้ไข yaw drift
    // CMAttitudeReferenceFrameXMagneticNorthZVertical - ใช้ compass
    // CMAttitudeReferenceFrameXTrueNorthZVertical - ใช้ GPS+compass
    
    [self.motionManager startDeviceMotionUpdatesUsingReferenceFrame:
         CMAttitudeReferenceFrameXArbitraryZVertical
         toQueue:[NSOperationQueue mainQueue]
         withHandler:^(CMDeviceMotion *motion, NSError *error) {
        
        if (error) {
            NSLog(@"Device Motion error: %@", error);
            return;
        }
        
        // 1. Attitude - ท่าทางของอุปกรณ์
        CMAttitude *attitude = motion.attitude;
        NSLog(@"Pitch: %.2f rad (%.1f°)", 
              attitude.pitch, attitude.pitch * 180.0 / M_PI);
        NSLog(@"Roll: %.2f rad (%.1f°)", 
              attitude.roll, attitude.roll * 180.0 / M_PI);
        NSLog(@"Yaw: %.2f rad (%.1f°)", 
              attitude.yaw, attitude.yaw * 180.0 / M_PI);
        
        // Quaternion
        CMQuaternion q = attitude.quaternion;
        NSLog(@"Quaternion: w=%.3f, x=%.3f, y=%.3f, z=%.3f", 
              q.w, q.x, q.y, q.z);
        
        // 2. Rotation Rate
        CMRotationRate rotation = motion.rotationRate;
        NSLog(@"Rotation Rate: x=%.3f, y=%.3f, z=%.3f", 
              rotation.x, rotation.y, rotation.z);
        
        // 3. Gravity - แรงโน้มถ่วง (g-force)
        CMAcceleration gravity = motion.gravity;
        NSLog(@"Gravity: x=%.3f, y=%.3f, z=%.3f g", 
              gravity.x, gravity.y, gravity.z);
        
        // 4. User Acceleration - แรงเร่งจากผู้ใช้ (ไม่รวม gravity)
        CMAcceleration userAcc = motion.userAcceleration;
        NSLog(@"User Acceleration: x=%.3f, y=%.3f, z=%.3f g", 
              userAcc.x, userAcc.y, userAcc.z);
        
        // 5. Magnetic Field (calibrated)
        CMCalibratedMagneticField calibField = motion.magneticField;
        NSLog(@"Calibrated Field - accuracy: %ld", (long)calibField.accuracy);
        
        // 6. Heading
        double heading = motion.heading;
        NSLog(@"Heading: %.1f องศา", heading);
    }];
}
```

### ตัวอย่าง: Shake Detection

```objc
#define SHAKE_THRESHOLD 2.5

@interface ShakeDetectionViewController : UIViewController

@property (nonatomic, strong) CMMotionManager *motionManager;
@property (nonatomic, strong) NSDate *lastShakeTime;

@end

@implementation ShakeDetectionViewController

- (void)viewDidLoad {
    [super viewDidLoad];
    self.motionManager = [[CMMotionManager alloc] init];
    self.motionManager.accelerometerUpdateInterval = 1.0 / 60.0;
}

- (void)viewDidAppear:(BOOL)animated {
    [super viewDidAppear:animated];
    
    [self.motionManager startAccelerometerUpdatesToQueue:[NSOperationQueue mainQueue] 
                                            withHandler:^(CMAccelerometerData *data, 
                                                          NSError *error) {
        CMAcceleration acc = data.acceleration;
        double magnitude = sqrt(acc.x*acc.x + acc.y*acc.y + acc.z*acc.z);
        
        if (magnitude > SHAKE_THRESHOLD) {
            // ป้องกัน double-trigger
            NSDate *now = [NSDate date];
            if (!self.lastShakeTime || 
                [now timeIntervalSinceDate:self.lastShakeTime] > 0.5) {
                self.lastShakeTime = now;
                [self handleShake];
            }
        }
    }];
}

- (void)handleShake {
    NSLog(@"ตรวจพบการเขย่า!");
    
    // สั่น feedback
    UIImpactFeedbackGenerator *generator = [[UIImpactFeedbackGenerator alloc] 
        initWithStyle:UIImpactFeedbackStyleHeavy];
    [generator prepare];
    [generator impactOccurred];
    
    // ทำอะไรบางอย่าง
    dispatch_async(dispatch_get_main_queue(), ^{
        self.view.backgroundColor = [UIColor randomColor];
    });
}

@end
```

---

## 66.6 Step Counter (CMPedometer)

CMPedometer ใช้ตรวจนับก้าวเดิน ต้องการ M7/M8 co-processor

```objc
#import <CoreMotion/CoreMotion.h>

@interface PedometerViewController : UIViewController

@property (nonatomic, strong) CMPedometer *pedometer;
@property (weak, nonatomic) IBOutlet UILabel *stepsLabel;
@property (weak, nonatomic) IBOutlet UILabel *distanceLabel;
@property (weak, nonatomic) IBOutlet UILabel *cadenceLabel;

@end

@implementation PedometerViewController

- (void)viewDidLoad {
    [super viewDidLoad];
    self.pedometer = [[CMPedometer alloc] init];
    
    // ตรวจสอบว่ารองรับหรือไม่
    NSLog(@"Step counting รองรับ: %@", 
          [CMPedometer isStepCountingAvailable] ? @"ใช่" : @"ไม่");
    NSLog(@"Distance รองรับ: %@", 
          [CMPedometer isDistanceAvailable] ? @"ใช่" : @"ไม่");
    NSLog(@"Cadence รองรับ: %@", 
          [CMPedometer isCadenceAvailable] ? @"ใช่" : @"ไม่");
    NSLog(@"Pace รองรับ: %@", 
          [CMPedometer isPaceAvailable] ? @"ใช่" : @"ไม่");
}

- (void)startRealTimePedometer {
    [self.pedometer startPedometerUpdatesFromDate:[NSDate date] 
                                     withHandler:^(CMPedometerData *data, 
                                                   NSError *error) {
        if (error) {
            NSLog(@"Pedometer error: %@", error);
            return;
        }
        
        dispatch_async(dispatch_get_main_queue(), ^{
            // จำนวนก้าว
            NSInteger steps = data.numberOfSteps.integerValue;
            self.stepsLabel.text = [NSString stringWithFormat:@"ก้าว: %ld", (long)steps];
            
            // ระยะทาง (เมตร)
            if (data.distance) {
                double distance = data.distance.doubleValue;
                self.distanceLabel.text = [NSString stringWithFormat:@"ระยะทาง: %.0f ม.", distance];
            }
            
            // Cadence (ก้าว/วินาที)
            if (data.currentCadence) {
                double cadence = data.currentCadence.doubleValue * 60.0; // ก้าว/นาที
                self.cadenceLabel.text = [NSString stringWithFormat:@"Cadence: %.0f ก้าว/นาที", cadence];
            }
            
            // Pace (วินาที/เมตร)
            if (data.currentPace) {
                double pace = data.currentPace.doubleValue;
                NSLog(@"Pace: %.2f วินาที/เมตร", pace);
            }
            
            // ชั้น (ต้องการ barometer)
            if (data.floorsAscended) {
                NSLog(@"ชั้นที่ขึ้น: %@", data.floorsAscended);
            }
            if (data.floorsDescended) {
                NSLog(@"ชั้นที่ลง: %@", data.floorsDescended);
            }
        });
    }];
}

- (void)stopPedometer {
    [self.pedometer stopPedometerUpdates];
}

// อ่านข้อมูลย้อนหลัง
- (void)queryHistoricalData {
    NSCalendar *calendar = [NSCalendar currentCalendar];
    NSDate *now = [NSDate date];
    NSDate *startOfDay = [calendar startOfDayForDate:now];
    
    [self.pedometer queryPedometerDataFromDate:startOfDay 
                                       toDate:now 
                                  withHandler:^(CMPedometerData *data, 
                                                NSError *error) {
        if (!error) {
            NSLog(@"ก้าววันนี้: %@", data.numberOfSteps);
            NSLog(@"ระยะทางวันนี้: %@ เมตร", data.distance);
        }
    }];
}

@end
```

---

## 66.7 Altimeter (CMAltimeter)

CMAltimeter ใช้วัดความสูงและความดันอากาศ ต้องการ M7/M8 co-processor

```objc
@interface AltimeterViewController : UIViewController

@property (nonatomic, strong) CMAltimeter *altimeter;
@property (weak, nonatomic) IBOutlet UILabel *altitudeLabel;
@property (weak, nonatomic) IBOutlet UILabel *pressureLabel;

@end

@implementation AltimeterViewController

- (void)viewDidLoad {
    [super viewDidLoad];
    self.altimeter = [[CMAltimeter alloc] init];
    
    NSLog(@"Altimeter รองรับ: %@", 
          [CMAltimeter isRelativeAltitudeAvailable] ? @"ใช่" : @"ไม่");
    
    // iOS 15+
    if (@available(iOS 15.0, *)) {
        NSLog(@"Absolute altitude รองรับ: %@", 
              [CMAltimeter isAbsoluteAltitudeAvailable] ? @"ใช่" : @"ไม่");
    }
}

- (void)startAltimeterUpdates {
    if (![CMAltimeter isRelativeAltitudeAvailable]) {
        NSLog(@"Altimeter ไม่รองรับ");
        return;
    }
    
    // relative altitude (เปรียบเทียบกับจุดเริ่มต้น)
    [self.altimeter startRelativeAltitudeUpdatesToQueue:[NSOperationQueue mainQueue] 
                                           withHandler:^(CMAltitudeData *data, 
                                                         NSError *error) {
        if (error) {
            NSLog(@"Altimeter error: %@", error);
            return;
        }
        
        dispatch_async(dispatch_get_main_queue(), ^{
            // ความสูงเปลี่ยนแปลง (เมตร)
            double relativeAltitude = data.relativeAltitude.doubleValue;
            self.altitudeLabel.text = [NSString stringWithFormat:@"ความสูงเปลี่ยน: %.2f ม.", 
                                       relativeAltitude];
            
            // ความดันอากาศ (kPa)
            double pressure = data.pressure.doubleValue;
            self.pressureLabel.text = [NSString stringWithFormat:@"ความดัน: %.2f kPa", 
                                       pressure];
        });
    }];
}

- (void)stopAltimeter {
    [self.altimeter stopRelativeAltitudeUpdates];
}

// iOS 15+ Absolute Altitude
- (void)startAbsoluteAltitudeUpdates {
    if (@available(iOS 15.0, *)) {
        if (![CMAltimeter isAbsoluteAltitudeAvailable]) {
            return;
        }
        
        [self.altimeter startAbsoluteAltitudeUpdatesToQueue:[NSOperationQueue mainQueue] 
                                               withHandler:^(CMAbsoluteAltitudeData *data, 
                                                             NSError *error) {
            if (!error) {
                NSLog(@"ความสูงสัมบูรณ์: %.2f ม. (accuracy: %.2f)", 
                      data.altitude, data.accuracy);
            }
        }];
    }
}

@end
```

---

## 66.8 Camera (AVCaptureSession)

### การตั้งค่ากล้องพื้นฐาน

```objc
// Info.plist ต้องมี:
// NSCameraUsageDescription - เหตุผลการใช้กล้อง
// NSMicrophoneUsageDescription - เหตุผลการใช้ไมค์

#import <AVFoundation/AVFoundation.h>

@interface CameraViewController : UIViewController

@property (nonatomic, strong) AVCaptureSession *captureSession;
@property (nonatomic, strong) AVCaptureVideoPreviewLayer *previewLayer;
@property (nonatomic, strong) AVCaptureDevice *currentCamera;
@property (nonatomic, strong) AVCapturePhotoOutput *photoOutput;
@property (nonatomic, strong) AVCaptureMovieFileOutput *movieOutput;

@end

@implementation CameraViewController

- (void)viewDidLoad {
    [super viewDidLoad];
    [self setupCamera];
}

- (void)setupCamera {
    // ตรวจสอบสิทธิ์
    AVAuthorizationStatus status = [AVCaptureDevice 
        authorizationStatusForMediaType:AVMediaTypeVideo];
    
    if (status == AVAuthorizationStatusAuthorized) {
        [self configureCaptureSession];
    } else if (status == AVAuthorizationStatusNotDetermined) {
        [AVCaptureDevice requestAccessForMediaType:AVMediaTypeVideo 
                               completionHandler:^(BOOL granted) {
            if (granted) {
                dispatch_async(dispatch_get_main_queue(), ^{
                    [self configureCaptureSession];
                });
            }
        }];
    } else {
        NSLog(@"ไม่ได้รับสิทธิ์ใช้กล้อง");
    }
}

- (void)configureCaptureSession {
    self.captureSession = [[AVCaptureSession alloc] init];
    
    // ตั้งค่าความละเอียด
    self.captureSession.sessionPreset = AVCaptureSessionPresetPhoto;
    // options: AVCaptureSessionPresetHigh, AVCaptureSessionPreset1920x1080,
    //          AVCaptureSessionPreset1280x720, etc.
    
    // เลือกกล้องหน้า/หลัง
    self.currentCamera = [self cameraWithPosition:AVCaptureDevicePositionBack];
    
    if (!self.currentCamera) {
        NSLog(@"ไม่พบกล้อง");
        return;
    }
    
    NSError *error;
    AVCaptureDeviceInput *input = [AVCaptureDeviceInput 
        deviceInputWithDevice:self.currentCamera 
                        error:&error];
    
    if (error) {
        NSLog(@"Camera input error: %@", error);
        return;
    }
    
    if ([self.captureSession canAddInput:input]) {
        [self.captureSession addInput:input];
    }
    
    // Photo Output
    self.photoOutput = [[AVCapturePhotoOutput alloc] init];
    if ([self.captureSession canAddOutput:self.photoOutput]) {
        [self.captureSession addOutput:self.photoOutput];
    }
    
    // Preview Layer
    self.previewLayer = [AVCaptureVideoPreviewLayer 
        layerWithSession:self.captureSession];
    self.previewLayer.videoGravity = AVLayerVideoGravityResizeAspectFill;
    self.previewLayer.frame = self.view.bounds;
    [self.view.layer insertSublayer:self.previewLayer atIndex:0];
    
    // เริ่ม session
    dispatch_async(dispatch_get_global_queue(DISPATCH_QUEUE_PRIORITY_DEFAULT, 0), ^{
        [self.captureSession startRunning];
    });
}

- (AVCaptureDevice *)cameraWithPosition:(AVCaptureDevicePosition)position {
    AVCaptureDeviceDiscoverySession *session = [AVCaptureDeviceDiscoverySession 
        discoverySessionWithDeviceTypes:@[AVCaptureDeviceTypeBuiltInWideAngleCamera]
        mediaType:AVMediaTypeVideo
        position:position];
    
    return session.devices.firstObject;
}

// สลับกล้องหน้า/หลัง
- (void)toggleCamera {
    AVCaptureDevicePosition currentPosition = self.currentCamera.position;
    AVCaptureDevicePosition newPosition = (currentPosition == AVCaptureDevicePositionBack) 
        ? AVCaptureDevicePositionFront 
        : AVCaptureDevicePositionBack;
    
    AVCaptureDevice *newCamera = [self cameraWithPosition:newPosition];
    
    [self.captureSession beginConfiguration];
    
    // ลบ input เดิม
    for (AVCaptureInput *input in self.captureSession.inputs) {
        [self.captureSession removeInput:input];
    }
    
    // เพิ่ม input ใหม่
    NSError *error;
    AVCaptureDeviceInput *newInput = [AVCaptureDeviceInput 
        deviceInputWithDevice:newCamera 
                        error:&error];
    
    if (!error && [self.captureSession canAddInput:newInput]) {
        [self.captureSession addInput:newInput];
        self.currentCamera = newCamera;
    }
    
    [self.captureSession commitConfiguration];
}

// ถ่ายภาพ
- (void)capturePhoto {
    AVCapturePhotoSettings *settings = [AVCapturePhotoSettings photoSettings];
    settings.flashMode = AVCaptureFlashModeAuto;
    
    [self.photoOutput capturePhotoWithSettings:settings delegate:self];
}

@end

// MARK: - AVCapturePhotoCaptureDelegate
@implementation CameraViewController (PhotoCapture)

- (void)captureOutput:(AVCapturePhotoOutput *)output 
    didFinishProcessingPhoto:(AVCapturePhoto *)photo 
                       error:(NSError *)error {
    
    if (error) {
        NSLog(@"Photo capture error: %@", error);
        return;
    }
    
    NSData *imageData = [photo fileDataRepresentation];
    UIImage *image = [UIImage imageWithData:imageData];
    
    // บันทึกลง Photo Library
    UIImageWriteToSavedPhotosAlbum(image, self, 
        @selector(image:didFinishSavingWithError:contextInfo:), nil);
    
    NSLog(@"ถ่ายภาพสำเร็จ");
}

- (void)image:(UIImage *)image 
    didFinishSavingWithError:(NSError *)error 
                 contextInfo:(void *)contextInfo {
    if (error) {
        NSLog(@"บันทึกภาพ error: %@", error);
    } else {
        NSLog(@"บันทึกภาพสำเร็จ");
    }
}

@end
```

### การตั้งค่า Camera Focus และ Exposure

```objc
- (void)configureFocusAtPoint:(CGPoint)touchPoint {
    CGPoint devicePoint = [self.previewLayer captureDevicePointOfInterestForPoint:touchPoint];
    
    NSError *error;
    if ([self.currentCamera lockForConfiguration:&error]) {
        
        // Focus
        if ([self.currentCamera isFocusPointOfInterestSupported] &&
            [self.currentCamera isFocusModeSupported:AVCaptureFocusModeAutoFocus]) {
            self.currentCamera.focusPointOfInterest = devicePoint;
            self.currentCamera.focusMode = AVCaptureFocusModeAutoFocus;
        }
        
        // Exposure
        if ([self.currentCamera isExposurePointOfInterestSupported] &&
            [self.currentCamera isExposureModeSupported:AVCaptureExposureModeAutoExpose]) {
            self.currentCamera.exposurePointOfInterest = devicePoint;
            self.currentCamera.exposureMode = AVCaptureExposureModeAutoExpose;
        }
        
        [self.currentCamera unlockForConfiguration];
    } else {
        NSLog(@"Camera config error: %@", error);
    }
}
```

---

## 66.9 Torch/Flashlight

การควบคุมไฟฉาย (Torch) ของกล้องหลัง

```objc
// TorchManager.h
@interface TorchManager : NSObject

+ (BOOL)isTorchAvailable;
+ (BOOL)isTorchActive;
+ (void)setTorchOn:(BOOL)on;
+ (void)setTorchLevel:(float)level; // 0.0 - 1.0

@end

// TorchManager.m
@implementation TorchManager

+ (AVCaptureDevice *)torchDevice {
    return [AVCaptureDevice defaultDeviceWithMediaType:AVMediaTypeVideo];
}

+ (BOOL)isTorchAvailable {
    return [self torchDevice].hasTorch;
}

+ (BOOL)isTorchActive {
    return [self torchDevice].torchActive;
}

+ (void)setTorchOn:(BOOL)on {
    AVCaptureDevice *device = [self torchDevice];
    
    if (!device.hasTorch) {
        NSLog(@"อุปกรณ์นี้ไม่มีไฟฉาย");
        return;
    }
    
    NSError *error;
    if ([device lockForConfiguration:&error]) {
        if (on) {
            if ([device isTorchModeSupported:AVCaptureTorchModeOn]) {
                device.torchMode = AVCaptureTorchModeOn;
            }
        } else {
            device.torchMode = AVCaptureTorchModeOff;
        }
        [device unlockForConfiguration];
    } else {
        NSLog(@"Torch config error: %@", error);
    }
}

+ (void)setTorchLevel:(float)level {
    AVCaptureDevice *device = [self torchDevice];
    
    if (!device.hasTorch) return;
    
    NSError *error;
    if ([device lockForConfiguration:&error]) {
        if (level <= 0) {
            device.torchMode = AVCaptureTorchModeOff;
        } else {
            float clampedLevel = MAX(AVCaptureMinAvailableTorchLevel, 
                                    MIN(1.0f, level));
            [device setTorchModeOnWithLevel:clampedLevel error:&error];
        }
        [device unlockForConfiguration];
    }
}

@end
```

### การใช้งาน Torch

```objc
// ตรวจสอบ
if ([TorchManager isTorchAvailable]) {
    NSLog(@"ไฟฉายพร้อมใช้งาน");
}

// เปิด/ปิด
[TorchManager setTorchOn:YES];
[TorchManager setTorchOn:NO];

// ปรับระดับความสว่าง
[TorchManager setTorchLevel:0.5f]; // 50%
[TorchManager setTorchLevel:1.0f]; // เต็มสว่าง

// Toggle
- (IBAction)torchButtonTapped:(UIButton *)sender {
    BOOL isOn = [TorchManager isTorchActive];
    [TorchManager setTorchOn:!isOn];
    
    NSString *buttonTitle = isOn ? @"เปิดไฟฉาย" : @"ปิดไฟฉาย";
    [sender setTitle:buttonTitle forState:UIControlStateNormal];
}
```

---

## 66.10 Proximity Sensor

Proximity Sensor ตรวจจับว่ามีสิ่งกีดขวางอยู่ใกล้ๆ หรือไม่ (เช่น เมื่อโทรศัพท์แนบหู)

```objc
- (void)enableProximitySensor {
    // เปิดใช้งาน proximity monitoring
    [[UIDevice currentDevice] setProximityMonitoringEnabled:YES];
    
    if (![UIDevice currentDevice].proximityMonitoringEnabled) {
        NSLog(@"Proximity sensor ไม่รองรับ");
        return;
    }
    
    // สมัคร notification
    [[NSNotificationCenter defaultCenter] 
        addObserver:self
           selector:@selector(proximityStateChanged:)
               name:UIDeviceProximityStateDidChangeNotification
             object:nil];
    
    NSLog(@"Proximity sensor เปิดใช้งาน");
}

- (void)disableProximitySensor {
    [[UIDevice currentDevice] setProximityMonitoringEnabled:NO];
    [[NSNotificationCenter defaultCenter] removeObserver:self 
        name:UIDeviceProximityStateDidChangeNotification object:nil];
}

- (void)proximityStateChanged:(NSNotification *)notification {
    BOOL isClose = [UIDevice currentDevice].proximityState;
    
    if (isClose) {
        NSLog(@"มีสิ่งกีดขวางใกล้หน้าจอ");
        // ปิดหน้าจอ (ระบบทำให้อัตโนมัติเมื่อเปิด proximity monitoring)
    } else {
        NSLog(@"หน้าจอโล่ง");
    }
}

- (void)dealloc {
    [self disableProximitySensor];
}
```

---

## 66.11 Battery Level Monitoring

การติดตามระดับแบตเตอรี่และสถานะการชาร์จ

```objc
// BatteryMonitor.h
@interface BatteryMonitor : NSObject

@property (nonatomic, readonly) float batteryLevel;
@property (nonatomic, readonly) UIDeviceBatteryState batteryState;
@property (nonatomic, readonly) NSString *batteryStateString;

- (void)startMonitoring;
- (void)stopMonitoring;

@end

// BatteryMonitor.m
@implementation BatteryMonitor

- (void)startMonitoring {
    [UIDevice currentDevice].batteryMonitoringEnabled = YES;
    
    // แสดงระดับปัจจุบัน
    [self logBatteryInfo];
    
    // สมัคร notifications
    [[NSNotificationCenter defaultCenter] 
        addObserver:self
           selector:@selector(batteryLevelChanged:)
               name:UIDeviceBatteryLevelDidChangeNotification
             object:nil];
    
    [[NSNotificationCenter defaultCenter] 
        addObserver:self
           selector:@selector(batteryStateChanged:)
               name:UIDeviceBatteryStateDidChangeNotification
             object:nil];
}

- (void)stopMonitoring {
    [UIDevice currentDevice].batteryMonitoringEnabled = NO;
    [[NSNotificationCenter defaultCenter] removeObserver:self];
}

- (float)batteryLevel {
    return [UIDevice currentDevice].batteryLevel;
}

- (UIDeviceBatteryState)batteryState {
    return [UIDevice currentDevice].batteryState;
}

- (NSString *)batteryStateString {
    switch (self.batteryState) {
        case UIDeviceBatteryStateUnknown:
            return @"ไม่ทราบ";
        case UIDeviceBatteryStateUnplugged:
            return @"ไม่ได้ชาร์จ";
        case UIDeviceBatteryStateCharging:
            return @"กำลังชาร์จ";
        case UIDeviceBatteryStateFull:
            return @"ชาร์จเต็มแล้ว";
        default:
            return @"ไม่ทราบ";
    }
}

- (void)logBatteryInfo {
    float level = self.batteryLevel;
    if (level == -1.0f) {
        NSLog(@"ไม่สามารถอ่านระดับแบตเตอรี่ได้");
        return;
    }
    
    NSLog(@"แบตเตอรี่: %.0f%%", level * 100.0);
    NSLog(@"สถานะ: %@", self.batteryStateString);
}

- (void)batteryLevelChanged:(NSNotification *)notification {
    float level = [UIDevice currentDevice].batteryLevel;
    NSLog(@"ระดับแบตเตอรี่เปลี่ยน: %.0f%%", level * 100.0);
    
    // แจ้งเตือนเมื่อแบตเตอรี่ต่ำ
    if (level > 0 && level < 0.2) {
        NSLog(@"⚠️ แบตเตอรี่ต่ำ!");
    }
}

- (void)batteryStateChanged:(NSNotification *)notification {
    NSLog(@"สถานะแบตเตอรี่เปลี่ยน: %@", self.batteryStateString);
}

- (void)dealloc {
    [self stopMonitoring];
}

@end
```

---

## 66.12 Thermal State

การตรวจสอบอุณหภูมิของอุปกรณ์ (iOS 11+)

```objc
- (void)monitorThermalState {
    // ตรวจสอบสถานะปัจจุบัน
    [self logThermalState];
    
    // สมัคร notification
    [[NSNotificationCenter defaultCenter] 
        addObserver:self
           selector:@selector(thermalStateChanged:)
               name:NSProcessInfoThermalStateDidChangeNotification
             object:nil];
}

- (void)logThermalState {
    NSProcessInfoThermalState state = [NSProcessInfo processInfo].thermalState;
    
    switch (state) {
        case NSProcessInfoThermalStateNominal:
            NSLog(@"อุณหภูมิ: ปกติ - ทำงานเต็มประสิทธิภาพ");
            break;
            
        case NSProcessInfoThermalStateFair:
            NSLog(@"อุณหภูมิ: เริ่มสูง - ยังทำงานได้ปกติ");
            break;
            
        case NSProcessInfoThermalStateSerious:
            NSLog(@"อุณหภูมิ: ร้อน - ลดการทำงานบางส่วน");
            // ควรลด background tasks
            break;
            
        case NSProcessInfoThermalStateCritical:
            NSLog(@"อุณหภูมิ: วิกฤต! - ระบบจำกัดการทำงานอย่างมาก");
            // ควรหยุด heavy tasks ทั้งหมด
            break;
    }
}

- (void)thermalStateChanged:(NSNotification *)notification {
    NSLog(@"สถานะอุณหภูมิเปลี่ยน:");
    [self logThermalState];
    
    NSProcessInfoThermalState state = [NSProcessInfo processInfo].thermalState;
    
    if (state >= NSProcessInfoThermalStateSerious) {
        [self reduceAppActivity];
    }
}

- (void)reduceAppActivity {
    NSLog(@"ลด activity เนื่องจากอุณหภูมิสูง");
    
    // หยุด expensive tasks
    [self.motionManager stopDeviceMotionUpdates];
    
    // ลด frame rate
    // ลด network requests
    // หยุด background processing
}
```

---

## 66.13 Motion Activity

CMMotionActivityManager ตรวจจับกิจกรรมของผู้ใช้

```objc
#import <CoreMotion/CoreMotion.h>

@interface ActivityDetectionViewController : UIViewController

@property (nonatomic, strong) CMMotionActivityManager *activityManager;
@property (weak, nonatomic) IBOutlet UILabel *activityLabel;

@end

@implementation ActivityDetectionViewController

- (void)viewDidLoad {
    [super viewDidLoad];
    self.activityManager = [[CMMotionActivityManager alloc] init];
    
    NSLog(@"Activity detection รองรับ: %@", 
          [CMMotionActivityManager isActivityAvailable] ? @"ใช่" : @"ไม่");
}

- (void)startActivityDetection {
    if (![CMMotionActivityManager isActivityAvailable]) {
        NSLog(@"Activity detection ไม่รองรับ");
        return;
    }
    
    [self.activityManager startActivityUpdatesToQueue:[NSOperationQueue mainQueue] 
                                         withHandler:^(CMMotionActivity *activity) {
        dispatch_async(dispatch_get_main_queue(), ^{
            NSString *activityStr = [self describeActivity:activity];
            self.activityLabel.text = activityStr;
            NSLog(@"กิจกรรม: %@ (confidence: %@)", 
                  activityStr, [self confidenceString:activity.confidence]);
        });
    }];
}

- (NSString *)describeActivity:(CMMotionActivity *)activity {
    if (activity.stationary) return @"หยุดนิ่ง";
    if (activity.walking)    return @"กำลังเดิน";
    if (activity.running)    return @"กำลังวิ่ง";
    if (activity.automotive) return @"ในยานพาหนะ";
    if (activity.cycling)    return @"กำลังปั่นจักรยาน";
    if (activity.unknown)    return @"ไม่ทราบ";
    return @"ไม่ทราบ";
}

- (NSString *)confidenceString:(CMMotionActivityConfidence)confidence {
    switch (confidence) {
        case CMMotionActivityConfidenceLow:    return @"ต่ำ";
        case CMMotionActivityConfidenceMedium: return @"กลาง";
        case CMMotionActivityConfidenceHigh:   return @"สูง";
        default: return @"ไม่ทราบ";
    }
}

- (void)stopActivityDetection {
    [self.activityManager stopActivityUpdates];
}

// อ่านประวัติกิจกรรม
- (void)queryActivityHistory {
    NSDate *startDate = [[NSDate date] dateByAddingTimeInterval:-3600]; // 1 ชั่วโมงที่แล้ว
    NSDate *endDate = [NSDate date];
    
    [self.activityManager queryActivityStartingFromDate:startDate
                                                 toDate:endDate
                                                toQueue:[NSOperationQueue mainQueue]
                                            withHandler:^(NSArray<CMMotionActivity *> *activities, 
                                                          NSError *error) {
        if (error) {
            NSLog(@"Activity history error: %@", error);
            return;
        }
        
        NSLog(@"ประวัติกิจกรรม 1 ชั่วโมงที่แล้ว:");
        for (CMMotionActivity *activity in activities) {
            NSLog(@"  %@: %@", activity.startDate, [self describeActivity:activity]);
        }
    }];
}

@end
```

---

## 66.14 Haptic Feedback

การสร้าง feedback แบบสัมผัส

```objc
// Impact Feedback
- (void)lightImpact {
    UIImpactFeedbackGenerator *generator = [[UIImpactFeedbackGenerator alloc] 
        initWithStyle:UIImpactFeedbackStyleLight];
    [generator prepare];
    [generator impactOccurred];
}

- (void)mediumImpact {
    UIImpactFeedbackGenerator *generator = [[UIImpactFeedbackGenerator alloc] 
        initWithStyle:UIImpactFeedbackStyleMedium];
    [generator impactOccurred];
}

- (void)heavyImpact {
    UIImpactFeedbackGenerator *generator = [[UIImpactFeedbackGenerator alloc] 
        initWithStyle:UIImpactFeedbackStyleHeavy];
    [generator impactOccurred];
}

// iOS 13+
- (void)softImpact {
    if (@available(iOS 13.0, *)) {
        UIImpactFeedbackGenerator *generator = [[UIImpactFeedbackGenerator alloc] 
            initWithStyle:UIImpactFeedbackStyleSoft];
        [generator impactOccurred];
    }
}

// Selection Feedback (เช่น เมื่อเลื่อน picker)
- (void)selectionFeedback {
    UISelectionFeedbackGenerator *generator = [[UISelectionFeedbackGenerator alloc] init];
    [generator prepare];
    [generator selectionChanged];
}

// Notification Feedback
- (void)successFeedback {
    UINotificationFeedbackGenerator *generator = [[UINotificationFeedbackGenerator alloc] init];
    [generator prepare];
    [generator notificationOccurred:UINotificationFeedbackTypeSuccess];
}

- (void)warningFeedback {
    UINotificationFeedbackGenerator *generator = [[UINotificationFeedbackGenerator alloc] init];
    [generator notificationOccurred:UINotificationFeedbackTypeWarning];
}

- (void)errorFeedback {
    UINotificationFeedbackGenerator *generator = [[UINotificationFeedbackGenerator alloc] init];
    [generator notificationOccurred:UINotificationFeedbackTypeError];
}
```

---

## 66.15 แบบฝึกหัด

### แบบฝึกหัดที่ 1: Motion Dashboard
สร้าง dashboard แสดงข้อมูล sensors ทั้งหมดที่:
- แสดง accelerometer, gyroscope, magnetometer แบบ real-time
- มี graph แสดงค่าแต่ละแกน
- ตรวจจับการเขย่าและแสดง alert

### แบบฝึกหัดที่ 2: Fitness Tracker
สร้าง fitness app ที่:
- นับก้าวด้วย CMPedometer
- คำนวณแคลอรี่ที่เผาผลาญ
- วัดระยะทางรวม
- แสดงกิจกรรม (เดิน/วิ่ง/รถ)

### แบบฝึกหัดที่ 3: Camera App
สร้าง custom camera app ที่:
- ถ่ายภาพได้
- สลับกล้องหน้า/หลัง
- ควบคุม zoom ด้วย pinch gesture
- มี torch button

### แบบฝึกหัดที่ 4: Level Meter App
สร้าง spirit level app ที่:
- ใช้ accelerometer วัดความเอียง
- แสดง bubble ที่เคลื่อนตามความเอียง
- มีเสียงเมื่อระดับ
- แสดงองศาแบบ real-time

---

## สรุป

ในตอนนี้เราได้เรียนรู้:
- **CoreMotion Framework** และ CMMotionManager
- **Accelerometer** วัดแรงเร่ง
- **Gyroscope** วัดการหมุน
- **Magnetometer** วัดสนามแม่เหล็ก
- **Device Motion** รวมทุก sensors
- **CMPedometer** นับก้าว
- **CMAltimeter** วัดความสูง
- **AVCaptureSession** จัดการกล้อง
- **Torch** ไฟฉาย
- **Proximity Sensor** ตรวจจับสิ่งกีดขวาง
- **Battery Monitoring** ระดับแบตเตอรี่
- **Thermal State** อุณหภูมิอุปกรณ์
- **Haptic Feedback** สัมผัสสั่น

ตอนต่อไปจะเรียนรู้ MapKit สำหรับการแสดงแผนที่

---

*หมายเหตุ: CMMotionManager ควรสร้างเพียง instance เดียวต่อแอป และควรหยุด updates เมื่อไม่ใช้เพื่อประหยัดพลังงาน*
