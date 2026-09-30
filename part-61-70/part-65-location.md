# ตอนที่ 65: Location Services ใน Objective-C

## บทนำ

Location Services เป็นหนึ่งในฟีเจอร์ที่สำคัญที่สุดของ iOS ที่ช่วยให้แอปพลิเคชันสามารถรู้ตำแหน่งของผู้ใช้งานได้ในแบบ real-time ในตอนนี้เราจะเรียนรู้การใช้งาน CoreLocation framework ตั้งแต่พื้นฐานจนถึงขั้นสูง

---

## 65.1 CoreLocation Framework

CoreLocation เป็น framework ที่ Apple จัดเตรียมไว้สำหรับการทำงานกับตำแหน่งทางภูมิศาสตร์ รองรับการทำงานทั้ง GPS, Wi-Fi, Bluetooth และ cellular network เพื่อระบุตำแหน่งที่แม่นยำ

### การเพิ่ม CoreLocation เข้าโปรเจค

ก่อนใช้งาน CoreLocation ต้องเพิ่ม framework เข้าโปรเจคก่อน:

1. เปิด Project Settings
2. เลือก Target ของแอป
3. ไปที่แท็บ "General" → "Frameworks, Libraries, and Embedded Content"
4. กด "+" และค้นหา "CoreLocation.framework"

### การ Import Framework

```objc
#import <CoreLocation/CoreLocation.h>
```

### คลาสหลักใน CoreLocation

| คลาส/Protocol | หน้าที่ |
|---------------|---------|
| `CLLocationManager` | จัดการและควบคุมการรับข้อมูลตำแหน่ง |
| `CLLocation` | เก็บข้อมูลตำแหน่ง (lat, long, altitude) |
| `CLLocationManagerDelegate` | รับ callback จาก LocationManager |
| `CLRegion` | กำหนดพื้นที่สำหรับ Geofencing |
| `CLCircularRegion` | พื้นที่วงกลมสำหรับ Geofencing |
| `CLBeaconRegion` | พื้นที่สำหรับ iBeacon |
| `CLHeading` | ข้อมูลทิศทาง (Compass) |
| `CLGeocoder` | แปลงพิกัดเป็นที่อยู่และกลับกัน |
| `CLPlacemark` | เก็บข้อมูลสถานที่ |

---

## 65.2 CLLocationManager

`CLLocationManager` คือคลาสหลักที่ใช้ในการจัดการ Location Services ทั้งหมด

### การสร้าง CLLocationManager

```objc
// ViewController.h
#import <UIKit/UIKit.h>
#import <CoreLocation/CoreLocation.h>

@interface ViewController : UIViewController <CLLocationManagerDelegate>

@property (nonatomic, strong) CLLocationManager *locationManager;
@property (nonatomic, strong) CLLocation *currentLocation;

@end
```

```objc
// ViewController.m
#import "ViewController.h"

@implementation ViewController

- (void)viewDidLoad {
    [super viewDidLoad];
    
    // สร้าง locationManager
    self.locationManager = [[CLLocationManager alloc] init];
    
    // ตั้ง delegate
    self.locationManager.delegate = self;
    
    // ตั้งค่าความแม่นยำ
    self.locationManager.desiredAccuracy = kCLLocationAccuracyBest;
    
    // ตั้งระยะห่างขั้นต่ำก่อน update (เมตร)
    self.locationManager.distanceFilter = 10.0;
    
    NSLog(@"LocationManager สร้างเสร็จแล้ว");
}

@end
```

### คุณสมบัติสำคัญของ CLLocationManager

```objc
// desiredAccuracy - ความแม่นยำที่ต้องการ
self.locationManager.desiredAccuracy = kCLLocationAccuracyBest;
self.locationManager.desiredAccuracy = kCLLocationAccuracyNearestTenMeters;
self.locationManager.desiredAccuracy = kCLLocationAccuracyHundredMeters;
self.locationManager.desiredAccuracy = kCLLocationAccuracyKilometer;
self.locationManager.desiredAccuracy = kCLLocationAccuracyThreeKilometers;

// distanceFilter - ระยะทางขั้นต่ำก่อน update
self.locationManager.distanceFilter = kCLDistanceFilterNone; // update ทุกครั้ง
self.locationManager.distanceFilter = 50.0; // update เมื่อเคลื่อนที่ 50 เมตร

// allowsBackgroundLocationUpdates - อนุญาต background update
self.locationManager.allowsBackgroundLocationUpdates = YES;

// pausesLocationUpdatesAutomatically - หยุดอัตโนมัติเมื่อไม่เคลื่อนที่
self.locationManager.pausesLocationUpdatesAutomatically = NO;

// activityType - ประเภทกิจกรรม
self.locationManager.activityType = CLActivityTypeAutomotiveNavigation;
self.locationManager.activityType = CLActivityTypeFitness;
self.locationManager.activityType = CLActivityTypeOther;
```

---

## 65.3 การขอ Permissions

การขอสิทธิ์การเข้าถึงตำแหน่งเป็นขั้นตอนที่สำคัญมาก

### ประเภทของ Permission

1. **WhenInUse (in-use)** - เข้าถึงตำแหน่งเฉพาะเมื่อแอปทำงานอยู่เบื้องหน้า
2. **Always** - เข้าถึงตำแหน่งได้ตลอดเวลา แม้แอปอยู่ใน background

### การตั้งค่า Info.plist

ต้องเพิ่ม key ใน Info.plist:

```xml
<!-- สำหรับ WhenInUse -->
<key>NSLocationWhenInUseUsageDescription</key>
<string>แอปต้องการตำแหน่งของคุณเพื่อแสดงข้อมูลใกล้เคียง</string>

<!-- สำหรับ Always (ต้องมีทั้งสองอัน) -->
<key>NSLocationAlwaysAndWhenInUseUsageDescription</key>
<string>แอปต้องการตำแหน่งของคุณตลอดเวลาเพื่อแจ้งเตือนเมื่อใกล้สถานที่สำคัญ</string>

<key>NSLocationAlwaysUsageDescription</key>
<string>แอปต้องการตำแหน่งของคุณเพื่อให้บริการตลอดเวลา</string>
```

### การขอสิทธิ์ใน Code

```objc
- (void)requestLocationPermission {
    CLAuthorizationStatus status;
    
    if (@available(iOS 14.0, *)) {
        status = self.locationManager.authorizationStatus;
    } else {
        status = [CLLocationManager authorizationStatus];
    }
    
    switch (status) {
        case kCLAuthorizationStatusNotDetermined:
            // ยังไม่ได้ตัดสินใจ - ขอสิทธิ์
            NSLog(@"กำลังขอสิทธิ์...");
            [self.locationManager requestWhenInUseAuthorization];
            // หรือ
            // [self.locationManager requestAlwaysAuthorization];
            break;
            
        case kCLAuthorizationStatusAuthorizedWhenInUse:
            NSLog(@"ได้รับสิทธิ์ WhenInUse");
            [self startLocationUpdates];
            break;
            
        case kCLAuthorizationStatusAuthorizedAlways:
            NSLog(@"ได้รับสิทธิ์ Always");
            [self startLocationUpdates];
            break;
            
        case kCLAuthorizationStatusDenied:
            NSLog(@"ผู้ใช้ปฏิเสธสิทธิ์");
            [self showLocationPermissionAlert];
            break;
            
        case kCLAuthorizationStatusRestricted:
            NSLog(@"การเข้าถึงตำแหน่งถูกจำกัด");
            break;
    }
}

- (void)showLocationPermissionAlert {
    UIAlertController *alert = [UIAlertController 
        alertControllerWithTitle:@"ต้องการสิทธิ์ตำแหน่ง"
        message:@"กรุณาเปิดสิทธิ์ตำแหน่งในการตั้งค่า"
        preferredStyle:UIAlertControllerStyleAlert];
    
    UIAlertAction *settingsAction = [UIAlertAction 
        actionWithTitle:@"ไปที่การตั้งค่า"
        style:UIAlertActionStyleDefault
        handler:^(UIAlertAction *action) {
            NSURL *settingsURL = [NSURL URLWithString:UIApplicationOpenSettingsURLString];
            [[UIApplication sharedApplication] openURL:settingsURL 
                                               options:@{} 
                                     completionHandler:nil];
        }];
    
    UIAlertAction *cancelAction = [UIAlertAction 
        actionWithTitle:@"ยกเลิก"
        style:UIAlertActionStyleCancel
        handler:nil];
    
    [alert addAction:settingsAction];
    [alert addAction:cancelAction];
    
    [self presentViewController:alert animated:YES completion:nil];
}
```

### Delegate สำหรับรับการเปลี่ยนแปลง Authorization

```objc
// รับ callback เมื่อ authorization เปลี่ยน (iOS 14+)
- (void)locationManagerDidChangeAuthorization:(CLLocationManager *)manager {
    switch (manager.authorizationStatus) {
        case kCLAuthorizationStatusAuthorizedWhenInUse:
        case kCLAuthorizationStatusAuthorizedAlways:
            [self startLocationUpdates];
            break;
        case kCLAuthorizationStatusDenied:
            [self handlePermissionDenied];
            break;
        default:
            break;
    }
}

// iOS 13 และก่อนหน้า
- (void)locationManager:(CLLocationManager *)manager 
    didChangeAuthorizationStatus:(CLAuthorizationStatus)status {
    if (status == kCLAuthorizationStatusAuthorizedWhenInUse ||
        status == kCLAuthorizationStatusAuthorizedAlways) {
        [self startLocationUpdates];
    }
}
```

---

## 65.4 การรับตำแหน่งปัจจุบัน

### การเริ่ม/หยุด Location Updates

```objc
- (void)startLocationUpdates {
    if ([CLLocationManager locationServicesEnabled]) {
        [self.locationManager startUpdatingLocation];
        NSLog(@"เริ่ม location updates");
    } else {
        NSLog(@"Location services ไม่เปิดใช้งาน");
    }
}

- (void)stopLocationUpdates {
    [self.locationManager stopUpdatingLocation];
    NSLog(@"หยุด location updates");
}

// ขอตำแหน่งเพียงครั้งเดียว (iOS 9+)
- (void)requestSingleLocation {
    [self.locationManager requestLocation];
}
```

### Delegate สำหรับรับตำแหน่ง

```objc
// รับ callback เมื่อได้ตำแหน่งใหม่
- (void)locationManager:(CLLocationManager *)manager 
     didUpdateLocations:(NSArray<CLLocation *> *)locations {
    
    // ดึงตำแหน่งล่าสุด
    CLLocation *latestLocation = [locations lastObject];
    
    self.currentLocation = latestLocation;
    
    // Coordinate
    CLLocationCoordinate2D coordinate = latestLocation.coordinate;
    NSLog(@"Latitude: %f", coordinate.latitude);
    NSLog(@"Longitude: %f", coordinate.longitude);
    
    // ความสูง
    NSLog(@"Altitude: %f เมตร", latestLocation.altitude);
    
    // ความแม่นยำ (เมตร)
    NSLog(@"Horizontal Accuracy: %f เมตร", latestLocation.horizontalAccuracy);
    NSLog(@"Vertical Accuracy: %f เมตร", latestLocation.verticalAccuracy);
    
    // เวลาที่บันทึก
    NSLog(@"Timestamp: %@", latestLocation.timestamp);
    
    // ความเร็ว (เมตร/วินาที)
    NSLog(@"Speed: %f m/s", latestLocation.speed);
    
    // ทิศทาง (องศา)
    NSLog(@"Course: %f องศา", latestLocation.course);
    
    // ตรวจสอบความแม่นยำก่อนใช้งาน
    if (latestLocation.horizontalAccuracy > 0 && 
        latestLocation.horizontalAccuracy < 100) {
        [self updateUIWithLocation:latestLocation];
    }
}

// รับ callback เมื่อเกิด error
- (void)locationManager:(CLLocationManager *)manager 
       didFailWithError:(NSError *)error {
    NSLog(@"Location error: %@", error.localizedDescription);
    
    if (error.code == kCLErrorDenied) {
        [self stopLocationUpdates];
        NSLog(@"ผู้ใช้ปฏิเสธสิทธิ์");
    } else if (error.code == kCLErrorLocationUnknown) {
        NSLog(@"ตำแหน่งไม่ทราบ - กำลังพยายามอีกครั้ง");
    }
}

- (void)updateUIWithLocation:(CLLocation *)location {
    dispatch_async(dispatch_get_main_queue(), ^{
        self.latLabel.text = [NSString stringWithFormat:@"Lat: %.6f", 
                              location.coordinate.latitude];
        self.lonLabel.text = [NSString stringWithFormat:@"Lon: %.6f", 
                              location.coordinate.longitude];
        self.accLabel.text = [NSString stringWithFormat:@"ความแม่นยำ: %.1f ม.", 
                              location.horizontalAccuracy];
    });
}
```

---

## 65.5 Location Accuracy

การเลือกความแม่นยำที่เหมาะสมมีผลต่อการใช้พลังงาน

### ระดับความแม่นยำ

```objc
// ดีที่สุด - ใช้ GPS เต็มกำลัง (ใช้พลังงานสูง)
self.locationManager.desiredAccuracy = kCLLocationAccuracyBest;

// สำหรับ Navigation - ความแม่นยำสูงสุด
self.locationManager.desiredAccuracy = kCLLocationAccuracyBestForNavigation;

// ภายใน 10 เมตร
self.locationManager.desiredAccuracy = kCLLocationAccuracyNearestTenMeters;

// ภายใน 100 เมตร
self.locationManager.desiredAccuracy = kCLLocationAccuracyHundredMeters;

// ภายใน 1 กิโลเมตร
self.locationManager.desiredAccuracy = kCLLocationAccuracyKilometer;

// ภายใน 3 กิโลเมตร (ใช้พลังงานต่ำที่สุด)
self.locationManager.desiredAccuracy = kCLLocationAccuracyThreeKilometers;

// Reduced Accuracy (iOS 14+)
self.locationManager.desiredAccuracy = kCLLocationAccuracyReduced;
```

### การตรวจสอบความแม่นยำ

```objc
- (BOOL)isLocationAccurate:(CLLocation *)location {
    // horizontalAccuracy ต้อง > 0 และน้อยกว่าค่าที่กำหนด
    if (location.horizontalAccuracy < 0) {
        return NO; // ข้อมูลไม่ถูกต้อง
    }
    
    if (location.horizontalAccuracy > 50) {
        return NO; // ความแม่นยำต่ำกว่าที่ต้องการ
    }
    
    // ตรวจสอบอายุของข้อมูล
    NSTimeInterval age = -[location.timestamp timeIntervalSinceNow];
    if (age > 30) {
        return NO; // ข้อมูลเก่าเกินไป
    }
    
    return YES;
}

- (void)locationManager:(CLLocationManager *)manager 
     didUpdateLocations:(NSArray<CLLocation *> *)locations {
    
    for (CLLocation *location in locations) {
        if ([self isLocationAccurate:location]) {
            NSLog(@"ตำแหน่งแม่นยำ: %f, %f", 
                  location.coordinate.latitude,
                  location.coordinate.longitude);
            self.currentLocation = location;
            break;
        }
    }
}
```

### การคำนวณระยะทาง

```objc
- (void)calculateDistance {
    CLLocation *bangkokLocation = [[CLLocation alloc] 
        initWithLatitude:13.7563 
               longitude:100.5018];
    
    CLLocation *chiangMaiLocation = [[CLLocation alloc] 
        initWithLatitude:18.7883 
               longitude:98.9853];
    
    CLLocationDistance distance = [bangkokLocation 
        distanceFromLocation:chiangMaiLocation];
    
    NSLog(@"ระยะทางจากกรุงเทพฯ ถึงเชียงใหม่: %.2f กิโลเมตร", 
          distance / 1000.0);
    
    // ตัวอย่างกับตำแหน่งปัจจุบัน
    if (self.currentLocation) {
        CLLocationDistance distanceToBangkok = [self.currentLocation 
            distanceFromLocation:bangkokLocation];
        NSLog(@"ระยะห่างจากตำแหน่งปัจจุบันถึงกรุงเทพฯ: %.2f กม.", 
              distanceToBangkok / 1000.0);
    }
}
```

---

## 65.6 Monitoring Regions และ Geofencing

Geofencing คือการกำหนดพื้นที่เสมือนและรับ notification เมื่อผู้ใช้เข้า/ออกพื้นที่นั้น

### การสร้างและเพิ่ม Region

```objc
- (void)setupGeofencing {
    // ตรวจสอบว่า region monitoring รองรับหรือไม่
    if (![CLLocationManager isMonitoringAvailableForClass:[CLCircularRegion class]]) {
        NSLog(@"Region monitoring ไม่รองรับ");
        return;
    }
    
    // สร้าง circular region
    CLLocationCoordinate2D center = CLLocationCoordinate2DMake(13.7563, 100.5018);
    CLLocationDistance radius = 500.0; // 500 เมตร
    NSString *identifier = @"BangkokCenter";
    
    CLCircularRegion *region = [[CLCircularRegion alloc] 
        initWithCenter:center 
                radius:radius 
            identifier:identifier];
    
    // กำหนดว่าจะ notify เมื่อไหร่
    region.notifyOnEntry = YES;  // แจ้งเมื่อเข้าพื้นที่
    region.notifyOnExit = YES;   // แจ้งเมื่อออกพื้นที่
    
    // เพิ่ม region ให้ locationManager
    [self.locationManager startMonitoringForRegion:region];
    NSLog(@"เริ่ม monitoring region: %@", identifier);
}

- (void)addGeofenceAtCoordinate:(CLLocationCoordinate2D)coordinate 
                          radius:(CLLocationDistance)radius 
                      identifier:(NSString *)identifier {
    
    // ตรวจสอบ radius limit
    CLLocationDistance maxRadius = self.locationManager.maximumRegionMonitoringDistance;
    CLLocationDistance actualRadius = MIN(radius, maxRadius);
    
    CLCircularRegion *region = [[CLCircularRegion alloc] 
        initWithCenter:coordinate 
                radius:actualRadius 
            identifier:identifier];
    
    region.notifyOnEntry = YES;
    region.notifyOnExit = YES;
    
    [self.locationManager startMonitoringForRegion:region];
    NSLog(@"เพิ่ม Geofence: %@ (radius: %.0f ม.)", identifier, actualRadius);
}

- (void)removeGeofence:(NSString *)identifier {
    for (CLRegion *region in self.locationManager.monitoredRegions) {
        if ([region.identifier isEqualToString:identifier]) {
            [self.locationManager stopMonitoringForRegion:region];
            NSLog(@"ลบ Geofence: %@", identifier);
            break;
        }
    }
}

- (void)removeAllGeofences {
    for (CLRegion *region in self.locationManager.monitoredRegions) {
        [self.locationManager stopMonitoringForRegion:region];
    }
    NSLog(@"ลบ Geofence ทั้งหมด");
}
```

### Delegate สำหรับ Region Monitoring

```objc
// เมื่อผู้ใช้เข้าพื้นที่
- (void)locationManager:(CLLocationManager *)manager 
         didEnterRegion:(CLRegion *)region {
    NSLog(@"เข้าพื้นที่: %@", region.identifier);
    
    // แสดง local notification
    [self sendLocalNotificationWithTitle:@"เข้าพื้นที่" 
                                    body:[NSString stringWithFormat:@"คุณเข้ามาในพื้นที่ %@", 
                                          region.identifier]];
}

// เมื่อผู้ใช้ออกจากพื้นที่
- (void)locationManager:(CLLocationManager *)manager 
          didExitRegion:(CLRegion *)region {
    NSLog(@"ออกจากพื้นที่: %@", region.identifier);
    
    [self sendLocalNotificationWithTitle:@"ออกจากพื้นที่"
                                    body:[NSString stringWithFormat:@"คุณออกจากพื้นที่ %@", 
                                          region.identifier]];
}

// เมื่อเริ่ม monitoring สำเร็จ
- (void)locationManager:(CLLocationManager *)manager 
     didStartMonitoringForRegion:(CLRegion *)region {
    NSLog(@"เริ่ม monitoring สำเร็จ: %@", region.identifier);
    
    // ตรวจสอบสถานะปัจจุบัน
    [self.locationManager requestStateForRegion:region];
}

// รับสถานะ region ปัจจุบัน
- (void)locationManager:(CLLocationManager *)manager 
       didDetermineState:(CLRegionState)state 
              forRegion:(CLRegion *)region {
    switch (state) {
        case CLRegionStateInside:
            NSLog(@"ปัจจุบันอยู่ในพื้นที่: %@", region.identifier);
            break;
        case CLRegionStateOutside:
            NSLog(@"ปัจจุบันอยู่นอกพื้นที่: %@", region.identifier);
            break;
        case CLRegionStateUnknown:
            NSLog(@"ไม่ทราบสถานะ: %@", region.identifier);
            break;
    }
}

// เมื่อ monitoring เกิด error
- (void)locationManager:(CLLocationManager *)manager 
    monitoringDidFailForRegion:(CLRegion *)region 
             withError:(NSError *)error {
    NSLog(@"Monitoring error สำหรับ %@: %@", 
          region.identifier, error.localizedDescription);
}

- (void)sendLocalNotificationWithTitle:(NSString *)title body:(NSString *)body {
    // iOS 10+
    UNMutableNotificationContent *content = [[UNMutableNotificationContent alloc] init];
    content.title = title;
    content.body = body;
    content.sound = [UNNotificationSound defaultSound];
    
    UNTimeIntervalNotificationTrigger *trigger = [UNTimeIntervalNotificationTrigger 
        triggerWithTimeInterval:1 repeats:NO];
    
    NSString *identifier = [NSUUID UUID].UUIDString;
    UNNotificationRequest *request = [UNNotificationRequest 
        requestWithIdentifier:identifier 
                      content:content 
                      trigger:trigger];
    
    [[UNUserNotificationCenter currentNotificationCenter] 
        addNotificationRequest:request 
         withCompletionHandler:^(NSError *error) {
        if (error) {
            NSLog(@"Notification error: %@", error);
        }
    }];
}
```

---

## 65.7 Heading และ Compass

CLLocationManager สามารถให้ข้อมูลทิศทาง (Compass) ได้ด้วย

### การใช้ Heading

```objc
- (void)startHeadingUpdates {
    if ([CLLocationManager headingAvailable]) {
        // ตั้งค่าก่อน update
        self.locationManager.headingFilter = 5.0; // update เมื่อเปลี่ยน 5 องศา
        
        [self.locationManager startUpdatingHeading];
        NSLog(@"เริ่ม heading updates");
    } else {
        NSLog(@"Heading ไม่รองรับในอุปกรณ์นี้");
    }
}

- (void)stopHeadingUpdates {
    [self.locationManager stopUpdatingHeading];
}

// Delegate
- (void)locationManager:(CLLocationManager *)manager 
       didUpdateHeading:(CLHeading *)newHeading {
    
    // True Heading - ทิศเหนือจริง (ต้องใช้งานร่วมกับ GPS)
    CLLocationDirection trueHeading = newHeading.trueHeading;
    
    // Magnetic Heading - ทิศเหนือแม่เหล็ก
    CLLocationDirection magneticHeading = newHeading.magneticHeading;
    
    // ความแม่นยำ (องศา)
    CLLocationDirection accuracy = newHeading.headingAccuracy;
    
    if (accuracy < 0) {
        NSLog(@"Heading ไม่แม่นยำ");
        return;
    }
    
    NSLog(@"True Heading: %.2f องศา", trueHeading);
    NSLog(@"Magnetic Heading: %.2f องศา", magneticHeading);
    
    // แปลงเป็นทิศ
    NSString *direction = [self directionStringFromDegrees:trueHeading];
    NSLog(@"ทิศ: %@", direction);
    
    // หมุน compass image
    dispatch_async(dispatch_get_main_queue(), ^{
        CGFloat radians = trueHeading * M_PI / 180.0;
        self.compassImageView.transform = CGAffineTransformMakeRotation(-radians);
    });
}

- (NSString *)directionStringFromDegrees:(CLLocationDirection)degrees {
    NSArray *directions = @[@"เหนือ", @"ตะวันออกเฉียงเหนือ", @"ตะวันออก", 
                            @"ตะวันออกเฉียงใต้", @"ใต้", @"ตะวันตกเฉียงใต้", 
                            @"ตะวันตก", @"ตะวันตกเฉียงเหนือ"];
    NSInteger index = (NSInteger)((degrees + 22.5) / 45.0) % 8;
    return directions[index];
}

// แสดงหรือซ่อน Heading Calibration
- (BOOL)locationManagerShouldDisplayHeadingCalibration:(CLLocationManager *)manager {
    return YES; // แสดง calibration UI เมื่อต้องการ
}
```

---

## 65.8 Background Location Updates

การรับตำแหน่งใน background ต้องการการตั้งค่าพิเศษ

### การตั้งค่า Background Modes

ใน Xcode:
1. เลือก Target → Signing & Capabilities
2. เพิ่ม "Background Modes"
3. เลือก "Location updates"

### Code สำหรับ Background Updates

```objc
- (void)enableBackgroundLocationUpdates {
    // ต้องมีสิทธิ์ Always
    if (self.locationManager.authorizationStatus != kCLAuthorizationStatusAuthorizedAlways) {
        NSLog(@"ต้องการสิทธิ์ Always สำหรับ background updates");
        [self.locationManager requestAlwaysAuthorization];
        return;
    }
    
    // เปิดใช้งาน background updates
    self.locationManager.allowsBackgroundLocationUpdates = YES;
    
    // ป้องกันการหยุดอัตโนมัติ
    self.locationManager.pausesLocationUpdatesAutomatically = NO;
    
    // เปิดใช้ตัวแสดงสถานะ background (แถบสีน้ำเงิน)
    self.locationManager.showsBackgroundLocationIndicator = YES;
    
    [self.locationManager startUpdatingLocation];
    NSLog(@"Background location updates เปิดใช้งาน");
}
```

---

## 65.9 Significant Location Changes

การติดตามการเปลี่ยนแปลงตำแหน่งอย่างมีนัยสำคัญ (ประหยัดพลังงานกว่า)

```objc
- (void)startSignificantLocationChanges {
    if ([CLLocationManager significantLocationChangeMonitoringAvailable]) {
        [self.locationManager startMonitoringSignificantLocationChanges];
        NSLog(@"เริ่ม significant location changes monitoring");
    } else {
        NSLog(@"Significant location changes ไม่รองรับ");
    }
}

- (void)stopSignificantLocationChanges {
    [self.locationManager stopMonitoringSignificantLocationChanges];
}

// Delegate เดียวกัน
- (void)locationManager:(CLLocationManager *)manager 
     didUpdateLocations:(NSArray<CLLocation *> *)locations {
    CLLocation *location = [locations lastObject];
    NSLog(@"Significant location change: %f, %f", 
          location.coordinate.latitude, 
          location.coordinate.longitude);
}
```

Significant Location Changes เหมาะสำหรับ:
- แอปที่ต้องการตรวจสอบตำแหน่งคร่าวๆ
- ไม่ต้องการความแม่นยำสูง
- ประหยัดแบตเตอรี่

---

## 65.10 Visit Monitoring

Visit Monitoring ติดตามสถานที่ที่ผู้ใช้มักไปบ่อย (ช่วงเวลาพักอยู่)

```objc
- (void)startVisitMonitoring {
    [self.locationManager startMonitoringVisits];
    NSLog(@"เริ่ม visit monitoring");
}

- (void)stopVisitMonitoring {
    [self.locationManager stopMonitoringVisits];
}

// Delegate
- (void)locationManager:(CLLocationManager *)manager 
         didVisit:(CLVisit *)visit {
    
    // ตำแหน่ง
    CLLocationCoordinate2D coordinate = visit.coordinate;
    
    // เวลาถึง
    NSDate *arrivalDate = visit.arrivalDate;
    
    // เวลาออก (อาจยังไม่ออก)
    NSDate *departureDate = visit.departureDate;
    
    // ความแม่นยำ
    CLLocationAccuracy accuracy = visit.horizontalAccuracy;
    
    BOOL isArriving = [departureDate isEqualToDate:[NSDate distantFuture]];
    
    if (isArriving) {
        NSLog(@"มาถึงสถานที่ (%.6f, %.6f) เวลา: %@", 
              coordinate.latitude, coordinate.longitude, arrivalDate);
    } else {
        NSTimeInterval duration = [departureDate timeIntervalSinceDate:arrivalDate];
        NSLog(@"ออกจากสถานที่หลังอยู่ %.0f นาที", duration / 60.0);
    }
}
```

---

## 65.11 Geocoding และ Reverse Geocoding

### Reverse Geocoding (พิกัด → ที่อยู่)

```objc
- (void)reverseGeocodeLocation:(CLLocation *)location {
    CLGeocoder *geocoder = [[CLGeocoder alloc] init];
    
    [geocoder reverseGeocodeLocation:location 
                   completionHandler:^(NSArray<CLPlacemark *> *placemarks, 
                                       NSError *error) {
        if (error) {
            NSLog(@"Geocoding error: %@", error.localizedDescription);
            return;
        }
        
        if (placemarks.count > 0) {
            CLPlacemark *placemark = placemarks.firstObject;
            
            NSLog(@"ชื่อ: %@", placemark.name);
            NSLog(@"ถนน: %@", placemark.thoroughfare);
            NSLog(@"ย่าน: %@", placemark.subLocality);
            NSLog(@"เขต/อำเภอ: %@", placemark.locality);
            NSLog(@"จังหวัด: %@", placemark.administrativeArea);
            NSLog(@"รหัสไปรษณีย์: %@", placemark.postalCode);
            NSLog(@"ประเทศ: %@", placemark.country);
            NSLog(@"รหัสประเทศ: %@", placemark.ISOcountryCode);
            
            // สร้างที่อยู่แบบเต็ม
            NSString *fullAddress = [NSString stringWithFormat:@"%@, %@, %@, %@", 
                                     placemark.thoroughfare ?: @"",
                                     placemark.locality ?: @"",
                                     placemark.administrativeArea ?: @"",
                                     placemark.country ?: @""];
            NSLog(@"ที่อยู่: %@", fullAddress);
        }
    }];
}
```

### Forward Geocoding (ที่อยู่ → พิกัด)

```objc
- (void)geocodeAddress:(NSString *)address {
    CLGeocoder *geocoder = [[CLGeocoder alloc] init];
    
    [geocoder geocodeAddressString:address 
                 completionHandler:^(NSArray<CLPlacemark *> *placemarks, 
                                     NSError *error) {
        if (error) {
            NSLog(@"Geocoding error: %@", error.localizedDescription);
            return;
        }
        
        if (placemarks.count > 0) {
            CLPlacemark *placemark = placemarks.firstObject;
            CLLocationCoordinate2D coordinate = placemark.location.coordinate;
            
            NSLog(@"พิกัดของ '%@': %.6f, %.6f", 
                  address, coordinate.latitude, coordinate.longitude);
        }
    }];
}
```

---

## 65.12 Location Manager ครบถ้วน

### ViewController ตัวอย่างครบถ้วน

```objc
// LocationViewController.h
#import <UIKit/UIKit.h>
#import <CoreLocation/CoreLocation.h>

@interface LocationViewController : UIViewController <CLLocationManagerDelegate>

@property (nonatomic, strong) CLLocationManager *locationManager;
@property (nonatomic, strong) CLLocation *currentLocation;

// UI
@property (weak, nonatomic) IBOutlet UILabel *statusLabel;
@property (weak, nonatomic) IBOutlet UILabel *coordinateLabel;
@property (weak, nonatomic) IBOutlet UILabel *accuracyLabel;
@property (weak, nonatomic) IBOutlet UILabel *addressLabel;
@property (weak, nonatomic) IBOutlet UIButton *startButton;
@property (weak, nonatomic) IBOutlet UIButton *stopButton;

@end
```

```objc
// LocationViewController.m
#import "LocationViewController.h"
#import <UserNotifications/UserNotifications.h>

@interface LocationViewController ()
@property (nonatomic, strong) CLGeocoder *geocoder;
@property (nonatomic, assign) BOOL isUpdating;
@end

@implementation LocationViewController

- (void)viewDidLoad {
    [super viewDidLoad];
    
    self.title = @"Location Services";
    self.geocoder = [[CLGeocoder alloc] init];
    
    [self setupLocationManager];
    [self requestNotificationPermission];
}

- (void)setupLocationManager {
    self.locationManager = [[CLLocationManager alloc] init];
    self.locationManager.delegate = self;
    self.locationManager.desiredAccuracy = kCLLocationAccuracyBest;
    self.locationManager.distanceFilter = 10.0;
    
    [self updateStatus:@"พร้อมใช้งาน"];
}

- (void)requestNotificationPermission {
    [[UNUserNotificationCenter currentNotificationCenter] 
        requestAuthorizationWithOptions:UNAuthorizationOptionAlert | UNAuthorizationOptionSound
        completionHandler:^(BOOL granted, NSError *error) {
        NSLog(@"Notification permission: %@", granted ? @"ได้รับ" : @"ปฏิเสธ");
    }];
}

#pragma mark - Actions

- (IBAction)startButtonTapped:(UIButton *)sender {
    [self requestLocationPermission];
}

- (IBAction)stopButtonTapped:(UIButton *)sender {
    [self.locationManager stopUpdatingLocation];
    self.isUpdating = NO;
    [self updateStatus:@"หยุดแล้ว"];
    self.startButton.enabled = YES;
    self.stopButton.enabled = NO;
}

- (void)requestLocationPermission {
    CLAuthorizationStatus status;
    if (@available(iOS 14.0, *)) {
        status = self.locationManager.authorizationStatus;
    } else {
        status = [CLLocationManager authorizationStatus];
    }
    
    if (status == kCLAuthorizationStatusNotDetermined) {
        [self.locationManager requestWhenInUseAuthorization];
    } else if (status == kCLAuthorizationStatusAuthorizedWhenInUse ||
               status == kCLAuthorizationStatusAuthorizedAlways) {
        [self startUpdatingLocation];
    } else {
        [self showPermissionAlert];
    }
}

- (void)startUpdatingLocation {
    if ([CLLocationManager locationServicesEnabled]) {
        [self.locationManager startUpdatingLocation];
        self.isUpdating = YES;
        [self updateStatus:@"กำลังติดตามตำแหน่ง..."];
        self.startButton.enabled = NO;
        self.stopButton.enabled = YES;
    }
}

#pragma mark - UI Updates

- (void)updateStatus:(NSString *)status {
    dispatch_async(dispatch_get_main_queue(), ^{
        self.statusLabel.text = [NSString stringWithFormat:@"สถานะ: %@", status];
    });
}

- (void)updateUIWithLocation:(CLLocation *)location {
    dispatch_async(dispatch_get_main_queue(), ^{
        self.coordinateLabel.text = [NSString stringWithFormat:
                                     @"พิกัด: %.6f, %.6f",
                                     location.coordinate.latitude,
                                     location.coordinate.longitude];
        
        self.accuracyLabel.text = [NSString stringWithFormat:
                                   @"ความแม่นยำ: ±%.0f เมตร",
                                   location.horizontalAccuracy];
    });
    
    // Reverse geocode
    [self.geocoder reverseGeocodeLocation:location 
                        completionHandler:^(NSArray<CLPlacemark *> *placemarks, 
                                            NSError *error) {
        if (!error && placemarks.count > 0) {
            CLPlacemark *p = placemarks.firstObject;
            NSString *address = [NSString stringWithFormat:@"%@, %@, %@",
                                 p.thoroughfare ?: @"-",
                                 p.locality ?: @"-",
                                 p.country ?: @"-"];
            dispatch_async(dispatch_get_main_queue(), ^{
                self.addressLabel.text = address;
            });
        }
    }];
}

#pragma mark - CLLocationManagerDelegate

- (void)locationManagerDidChangeAuthorization:(CLLocationManager *)manager {
    switch (manager.authorizationStatus) {
        case kCLAuthorizationStatusAuthorizedWhenInUse:
        case kCLAuthorizationStatusAuthorizedAlways:
            [self startUpdatingLocation];
            break;
        case kCLAuthorizationStatusDenied:
        case kCLAuthorizationStatusRestricted:
            [self updateStatus:@"ไม่ได้รับสิทธิ์"];
            break;
        default:
            break;
    }
}

- (void)locationManager:(CLLocationManager *)manager 
     didUpdateLocations:(NSArray<CLLocation *> *)locations {
    
    CLLocation *location = [locations lastObject];
    
    if (location.horizontalAccuracy > 0 && location.horizontalAccuracy < 100) {
        self.currentLocation = location;
        [self updateUIWithLocation:location];
        [self updateStatus:@"ตำแหน่งอัปเดตแล้ว"];
    }
}

- (void)locationManager:(CLLocationManager *)manager 
       didFailWithError:(NSError *)error {
    NSLog(@"Location error: %@", error);
    [self updateStatus:[NSString stringWithFormat:@"Error: %@", error.localizedDescription]];
}

#pragma mark - Alert

- (void)showPermissionAlert {
    UIAlertController *alert = [UIAlertController 
        alertControllerWithTitle:@"ต้องการสิทธิ์ตำแหน่ง"
        message:@"กรุณาเปิดสิทธิ์ใน Settings > Privacy > Location Services"
        preferredStyle:UIAlertControllerStyleAlert];
    
    [alert addAction:[UIAlertAction actionWithTitle:@"Settings" 
                                             style:UIAlertActionStyleDefault 
                                           handler:^(UIAlertAction *action) {
        [[UIApplication sharedApplication] openURL:[NSURL URLWithString:UIApplicationOpenSettingsURLString] 
                                           options:@{} completionHandler:nil];
    }]];
    
    [alert addAction:[UIAlertAction actionWithTitle:@"ยกเลิก" 
                                             style:UIAlertActionStyleCancel 
                                           handler:nil]];
    
    [self presentViewController:alert animated:YES completion:nil];
}

@end
```

---

## 65.13 แบบฝึกหัด

### แบบฝึกหัดที่ 1: Location Tracker App
สร้างแอปที่:
- แสดงพิกัดปัจจุบันแบบ real-time
- บันทึกเส้นทางที่เดินทาง
- คำนวณระยะทางรวม
- แสดงความเร็วและทิศทาง

### แบบฝึกหัดที่ 2: Geofencing Manager
สร้างระบบ Geofencing ที่:
- เพิ่ม/ลบ geofence ได้
- แจ้งเตือนเมื่อเข้า/ออก
- แสดงรายการ geofence ทั้งหมด
- บันทึกประวัติการเข้า/ออก

### แบบฝึกหัดที่ 3: Nearby Places Finder
สร้างแอปค้นหาสถานที่ใกล้เคียงที่:
- ใช้ตำแหน่งปัจจุบัน
- Reverse geocode แสดงที่อยู่
- คำนวณระยะทางถึงสถานที่
- เรียงลำดับตามระยะทาง

### แบบฝึกหัดที่ 4: Compass App
สร้าง Compass app ที่:
- แสดงทิศทางแบบ visual
- บอกทิศ (เหนือ ใต้ ออก ตก)
- แสดงมุม (องศา)
- calibration indicator

---

## สรุป

ในตอนนี้เราได้เรียนรู้:
- **CoreLocation Framework** และ CLLocationManager
- **การขอ Permissions** ทั้งแบบ WhenInUse และ Always
- **การรับตำแหน่ง** และตรวจสอบความแม่นยำ
- **Geofencing** สำหรับการ monitoring พื้นที่
- **Heading** สำหรับ Compass
- **Background Location Updates**
- **Significant Location Changes** ประหยัดพลังงาน
- **Visit Monitoring** ติดตามสถานที่ที่ไปบ่อย
- **Geocoding** แปลงพิกัดเป็นที่อยู่

ตอนต่อไปจะเรียนรู้เกี่ยวกับ Sensors และ Hardware ต่างๆ ที่ iOS มีให้ใช้งาน

---

*หมายเหตุ: การใช้ Location Services ต้องได้รับสิทธิ์จากผู้ใช้เสมอ และควรใช้ความแม่นยำที่เหมาะสมกับงานเพื่อประหยัดพลังงาน*
