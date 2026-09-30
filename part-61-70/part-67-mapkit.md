# ตอนที่ 67: MapKit ใน Objective-C

## บทนำ

MapKit เป็น framework ของ Apple ที่ให้บริการแผนที่แบบครบวงจร สามารถแสดงแผนที่, เพิ่ม annotations, วาด overlays, หาเส้นทาง และค้นหาสถานที่ได้ ในตอนนี้เราจะเรียนรู้การใช้งาน MapKit อย่างละเอียด

---

## 67.1 MKMapView Setup

### การเพิ่ม MapKit

```objc
#import <MapKit/MapKit.h>
```

### การสร้าง MKMapView

```objc
// MapViewController.h
#import <UIKit/UIKit.h>
#import <MapKit/MapKit.h>
#import <CoreLocation/CoreLocation.h>

@interface MapViewController : UIViewController <MKMapViewDelegate, CLLocationManagerDelegate>

@property (nonatomic, strong) MKMapView *mapView;
@property (nonatomic, strong) CLLocationManager *locationManager;

@end
```

```objc
// MapViewController.m
#import "MapViewController.h"

@implementation MapViewController

- (void)viewDidLoad {
    [super viewDidLoad];
    
    [self setupMapView];
    [self setupLocationManager];
}

- (void)setupMapView {
    // สร้าง MKMapView
    self.mapView = [[MKMapView alloc] initWithFrame:self.view.bounds];
    self.mapView.autoresizingMask = UIViewAutoresizingFlexibleWidth | 
                                    UIViewAutoresizingFlexibleHeight;
    
    // ตั้ง delegate
    self.mapView.delegate = self;
    
    // เพิ่มใน view
    [self.view addSubview:self.mapView];
    
    NSLog(@"MapView สร้างเสร็จแล้ว");
}

- (void)setupLocationManager {
    self.locationManager = [[CLLocationManager alloc] init];
    self.locationManager.delegate = self;
    self.locationManager.desiredAccuracy = kCLLocationAccuracyBest;
    [self.locationManager requestWhenInUseAuthorization];
}

// ถ้าใช้ Interface Builder สามารถ connect IBOutlet แทน
@end
```

### การตั้งค่าเบื้องต้น

```objc
- (void)configureMapView {
    // ประเภทแผนที่
    self.mapView.mapType = MKMapTypeStandard;
    
    // ตำแหน่งผู้ใช้
    self.mapView.showsUserLocation = YES;
    
    // Zoom controls
    self.mapView.zoomEnabled = YES;
    self.mapView.scrollEnabled = YES;
    self.mapView.rotateEnabled = YES;
    self.mapView.pitchEnabled = YES; // 3D view
    
    // Compass
    self.mapView.showsCompass = YES;
    
    // Scale indicator
    self.mapView.showsScale = YES;
    
    // Traffic
    self.mapView.showsTraffic = YES;
    
    // Buildings (3D)
    self.mapView.showsBuildings = YES;
    
    // Point of interest
    // iOS 13+
    if (@available(iOS 13.0, *)) {
        self.mapView.pointOfInterestFilter = [MKPointOfInterestFilter filterIncludingAllCategories];
    }
}
```

---

## 67.2 Map Types

MKMapView รองรับหลายประเภทของแผนที่

```objc
- (void)setupMapTypeSelector {
    UISegmentedControl *segmentedControl = [[UISegmentedControl alloc] 
        initWithItems:@[@"Standard", @"Satellite", @"Hybrid", @"Satellite Flyover", 
                        @"Hybrid Flyover", @"Muted Standard"]];
    
    segmentedControl.selectedSegmentIndex = 0;
    [segmentedControl addTarget:self 
                         action:@selector(mapTypeChanged:) 
               forControlEvents:UIControlEventValueChanged];
    
    // เพิ่มใน navigation bar
    self.navigationItem.titleView = segmentedControl;
}

- (void)mapTypeChanged:(UISegmentedControl *)sender {
    switch (sender.selectedSegmentIndex) {
        case 0:
            self.mapView.mapType = MKMapTypeStandard;
            break;
        case 1:
            self.mapView.mapType = MKMapTypeSatellite;
            break;
        case 2:
            self.mapView.mapType = MKMapTypeHybrid;
            break;
        case 3:
            self.mapView.mapType = MKMapTypeSatelliteFlyover;
            break;
        case 4:
            self.mapView.mapType = MKMapTypeHybridFlyover;
            break;
        case 5:
            self.mapView.mapType = MKMapTypeMutedStandard;
            break;
    }
}
```

---

## 67.3 User Location on Map

### แสดงตำแหน่งผู้ใช้บนแผนที่

```objc
- (void)showUserLocation {
    // แสดง blue dot
    self.mapView.showsUserLocation = YES;
    
    // Zoom ไปที่ตำแหน่งผู้ใช้
    self.mapView.userTrackingMode = MKUserTrackingModeNone;
    // MKUserTrackingModeFollow - ติดตาม
    // MKUserTrackingModeFollowWithHeading - ติดตามพร้อมทิศทาง
}

- (void)centerMapOnUserLocation {
    CLLocation *userLocation = self.mapView.userLocation.location;
    
    if (userLocation) {
        MKCoordinateRegion region = MKCoordinateRegionMakeWithDistance(
            userLocation.coordinate,
            1000, // 1 กม. กว้าง
            1000  // 1 กม. สูง
        );
        [self.mapView setRegion:region animated:YES];
    }
}

// Delegate
- (void)mapView:(MKMapView *)mapView 
    didUpdateUserLocation:(MKUserLocation *)userLocation {
    
    NSLog(@"ตำแหน่งผู้ใช้: %f, %f", 
          userLocation.coordinate.latitude,
          userLocation.coordinate.longitude);
    
    // Center map ครั้งแรก
    static BOOL firstUpdate = YES;
    if (firstUpdate) {
        firstUpdate = NO;
        MKCoordinateRegion region = MKCoordinateRegionMakeWithDistance(
            userLocation.coordinate, 2000, 2000);
        [self.mapView setRegion:region animated:YES];
    }
}

- (void)mapView:(MKMapView *)mapView 
    didFailToLocateUserWithError:(NSError *)error {
    NSLog(@"ไม่สามารถหาตำแหน่งผู้ใช้: %@", error.localizedDescription);
}
```

---

## 67.4 Annotations (MKPointAnnotation)

Annotations คือ pins หรือ markers บนแผนที่

### การเพิ่ม Basic Annotation

```objc
// เพิ่ม annotation เดียว
- (void)addAnnotationAtCoordinate:(CLLocationCoordinate2D)coordinate 
                            title:(NSString *)title 
                         subtitle:(NSString *)subtitle {
    
    MKPointAnnotation *annotation = [[MKPointAnnotation alloc] init];
    annotation.coordinate = coordinate;
    annotation.title = title;
    annotation.subtitle = subtitle;
    
    [self.mapView addAnnotation:annotation];
}

// ตัวอย่างการใช้
- (void)addSampleAnnotations {
    // กรุงเทพฯ
    [self addAnnotationAtCoordinate:CLLocationCoordinate2DMake(13.7563, 100.5018)
                              title:@"กรุงเทพมหานคร"
                           subtitle:@"เมืองหลวงของประเทศไทย"];
    
    // เชียงใหม่
    [self addAnnotationAtCoordinate:CLLocationCoordinate2DMake(18.7883, 98.9853)
                              title:@"เชียงใหม่"
                           subtitle:@"เมืองเหนือ"];
    
    // ภูเก็ต
    [self addAnnotationAtCoordinate:CLLocationCoordinate2DMake(7.8804, 98.3923)
                              title:@"ภูเก็ต"
                           subtitle:@"จังหวัดชายทะเล"];
    
    // Zoom to fit all annotations
    [self.mapView showAnnotations:self.mapView.annotations animated:YES];
}

// เพิ่มหลาย annotations พร้อมกัน
- (void)addMultipleAnnotations:(NSArray<NSDictionary *> *)places {
    NSMutableArray *annotations = [NSMutableArray array];
    
    for (NSDictionary *place in places) {
        MKPointAnnotation *annotation = [[MKPointAnnotation alloc] init];
        annotation.coordinate = CLLocationCoordinate2DMake(
            [place[@"lat"] doubleValue],
            [place[@"lng"] doubleValue]
        );
        annotation.title = place[@"name"];
        annotation.subtitle = place[@"description"];
        [annotations addObject:annotation];
    }
    
    [self.mapView addAnnotations:annotations];
}

// ลบ annotations
- (void)removeAllAnnotations {
    // เก็บ user location ไว้
    NSArray *annotationsToRemove = [self.mapView.annotations filteredArrayUsingPredicate:
        [NSPredicate predicateWithBlock:^BOOL(id annotation, NSDictionary *bindings) {
            return ![annotation isKindOfClass:[MKUserLocation class]];
        }]];
    
    [self.mapView removeAnnotations:annotationsToRemove];
}
```

### Custom Annotation Model

```objc
// PlaceAnnotation.h
#import <MapKit/MapKit.h>

typedef NS_ENUM(NSInteger, PlaceCategory) {
    PlaceCategoryRestaurant,
    PlaceCategoryHotel,
    PlaceCategoryAttraction,
    PlaceCategoryShop
};

@interface PlaceAnnotation : NSObject <MKAnnotation>

@property (nonatomic, assign) CLLocationCoordinate2D coordinate;
@property (nonatomic, copy) NSString *title;
@property (nonatomic, copy) NSString *subtitle;
@property (nonatomic, assign) PlaceCategory category;
@property (nonatomic, strong) UIImage *thumbnailImage;
@property (nonatomic, strong) NSDictionary *extraInfo;

- (instancetype)initWithCoordinate:(CLLocationCoordinate2D)coordinate
                              title:(NSString *)title
                           category:(PlaceCategory)category;

@end
```

```objc
// PlaceAnnotation.m
@implementation PlaceAnnotation

- (instancetype)initWithCoordinate:(CLLocationCoordinate2D)coordinate
                              title:(NSString *)title
                           category:(PlaceCategory)category {
    self = [super init];
    if (self) {
        _coordinate = coordinate;
        _title = [title copy];
        _category = category;
    }
    return self;
}

@end
```

---

## 67.5 Custom Annotation Views

Annotation views คือ visual representation ของ annotations

```objc
// Custom Annotation View Identifier
static NSString * const kAnnotationIdentifier = @"PlaceAnnotation";
static NSString * const kClusterAnnotationIdentifier = @"PlaceCluster";

// MKMapViewDelegate
- (MKAnnotationView *)mapView:(MKMapView *)mapView 
            viewForAnnotation:(id<MKAnnotation>)annotation {
    
    // ไม่แก้ไข user location
    if ([annotation isKindOfClass:[MKUserLocation class]]) {
        return nil; // ใช้ default
    }
    
    if ([annotation isKindOfClass:[PlaceAnnotation class]]) {
        return [self annotationViewForPlace:(PlaceAnnotation *)annotation 
                                   inMapView:mapView];
    }
    
    return nil;
}

- (MKAnnotationView *)annotationViewForPlace:(PlaceAnnotation *)place 
                                    inMapView:(MKMapView *)mapView {
    
    // ใช้ MKMarkerAnnotationView (iOS 11+) - สวยกว่า MKPinAnnotationView
    MKMarkerAnnotationView *view = (MKMarkerAnnotationView *)
        [mapView dequeueReusableAnnotationViewWithIdentifier:kAnnotationIdentifier];
    
    if (!view) {
        view = [[MKMarkerAnnotationView alloc] 
            initWithAnnotation:place 
               reuseIdentifier:kAnnotationIdentifier];
        view.canShowCallout = YES;
    }
    
    view.annotation = place;
    
    // ตั้งค่า marker ตาม category
    switch (place.category) {
        case PlaceCategoryRestaurant:
            view.markerTintColor = [UIColor orangeColor];
            view.glyphImage = [UIImage systemImageNamed:@"fork.knife"];
            break;
        case PlaceCategoryHotel:
            view.markerTintColor = [UIColor blueColor];
            view.glyphImage = [UIImage systemImageNamed:@"bed.double"];
            break;
        case PlaceCategoryAttraction:
            view.markerTintColor = [UIColor purpleColor];
            view.glyphImage = [UIImage systemImageNamed:@"camera"];
            break;
        case PlaceCategoryShop:
            view.markerTintColor = [UIColor greenColor];
            view.glyphImage = [UIImage systemImageNamed:@"bag"];
            break;
    }
    
    // Callout accessories
    // Left - ภาพ thumbnail
    if (place.thumbnailImage) {
        UIImageView *leftImageView = [[UIImageView alloc] 
            initWithFrame:CGRectMake(0, 0, 40, 40)];
        leftImageView.image = place.thumbnailImage;
        leftImageView.layer.cornerRadius = 5;
        leftImageView.clipsToBounds = YES;
        view.leftCalloutAccessoryView = leftImageView;
    }
    
    // Right - ปุ่ม detail
    UIButton *detailButton = [UIButton buttonWithType:UIButtonTypeDetailDisclosure];
    view.rightCalloutAccessoryView = detailButton;
    
    return view;
}

// Handle callout tap
- (void)mapView:(MKMapView *)mapView 
    annotationView:(MKAnnotationView *)view 
    calloutAccessoryControlTapped:(UIControl *)control {
    
    if ([view.annotation isKindOfClass:[PlaceAnnotation class]]) {
        PlaceAnnotation *place = (PlaceAnnotation *)view.annotation;
        NSLog(@"เปิด detail สำหรับ: %@", place.title);
        // เปิด detail view controller
    }
}

// Selection
- (void)mapView:(MKMapView *)mapView 
    didSelectAnnotationView:(MKAnnotationView *)view {
    NSLog(@"เลือก annotation: %@", view.annotation.title);
}

- (void)mapView:(MKMapView *)mapView 
    didDeselectAnnotationView:(MKAnnotationView *)view {
    NSLog(@"ยกเลิกการเลือก annotation");
}
```

### Annotation Clustering (iOS 11+)

```objc
// ลงทะเบียน annotation view classes
- (void)registerAnnotationViewClasses {
    [self.mapView registerClass:[MKMarkerAnnotationView class] 
        forAnnotationViewWithReuseIdentifier:kAnnotationIdentifier];
    
    [self.mapView registerClass:[MKMarkerAnnotationView class] 
        forAnnotationViewWithReuseIdentifier:MKMapViewDefaultClusterAnnotationViewReuseIdentifier];
}

// ตั้ง cluster identifier ใน annotation view
- (MKAnnotationView *)mapView:(MKMapView *)mapView 
            viewForAnnotation:(id<MKAnnotation>)annotation {
    
    if ([annotation isKindOfClass:[MKClusterAnnotation class]]) {
        MKMarkerAnnotationView *view = (MKMarkerAnnotationView *)
            [mapView dequeueReusableAnnotationViewWithIdentifier:
             MKMapViewDefaultClusterAnnotationViewReuseIdentifier];
        
        view.markerTintColor = [UIColor systemBlueColor];
        
        MKClusterAnnotation *cluster = (MKClusterAnnotation *)annotation;
        view.glyphText = [NSString stringWithFormat:@"%lu", 
                          (unsigned long)cluster.memberAnnotations.count];
        
        return view;
    }
    
    MKMarkerAnnotationView *view = (MKMarkerAnnotationView *)
        [mapView dequeueReusableAnnotationViewWithIdentifier:kAnnotationIdentifier];
    
    if (!view) {
        view = [[MKMarkerAnnotationView alloc] initWithAnnotation:annotation 
                                                  reuseIdentifier:kAnnotationIdentifier];
    }
    
    // เปิด clustering
    view.clusteringIdentifier = @"place-cluster";
    view.canShowCallout = YES;
    
    return view;
}
```

---

## 67.6 Overlays

Overlays คือ graphic elements บนแผนที่ เช่น เส้น, รูปทรง, วงกลม

### MKPolyline (เส้น)

```objc
- (void)addRoutePolyline {
    // ตัวอย่าง: วาดเส้นทางจากกรุงเทพ ถึง เชียงใหม่
    CLLocationCoordinate2D coordinates[] = {
        CLLocationCoordinate2DMake(13.7563, 100.5018), // กรุงเทพ
        CLLocationCoordinate2DMake(14.9000, 100.1000), // อยุธยา
        CLLocationCoordinate2DMake(16.4321, 100.1962), // พิษณุโลก
        CLLocationCoordinate2DMake(17.9290, 100.0000), // ลำปาง
        CLLocationCoordinate2DMake(18.7883, 98.9853),  // เชียงใหม่
    };
    
    NSUInteger count = sizeof(coordinates) / sizeof(CLLocationCoordinate2D);
    MKPolyline *polyline = [MKPolyline polylineWithCoordinates:coordinates 
                                                         count:count];
    polyline.title = @"เส้นทางกรุงเทพ-เชียงใหม่";
    
    [self.mapView addOverlay:polyline];
    
    // Zoom to show polyline
    [self.mapView setVisibleMapRect:polyline.boundingMapRect 
                         edgePadding:UIEdgeInsetsMake(40, 40, 40, 40) 
                            animated:YES];
}
```

### MKPolygon (รูปทรง)

```objc
- (void)addAreaPolygon {
    // วาดรูปทรง polygon
    CLLocationCoordinate2D coordinates[] = {
        CLLocationCoordinate2DMake(13.80, 100.45),
        CLLocationCoordinate2DMake(13.80, 100.56),
        CLLocationCoordinate2DMake(13.71, 100.56),
        CLLocationCoordinate2DMake(13.71, 100.45),
    };
    
    NSUInteger count = sizeof(coordinates) / sizeof(CLLocationCoordinate2D);
    MKPolygon *polygon = [MKPolygon polygonWithCoordinates:coordinates count:count];
    polygon.title = @"กรุงเทพฯ ชั้นใน";
    
    [self.mapView addOverlay:polygon];
    
    // Polygon with hole (interior polygon)
    CLLocationCoordinate2D holeCoords[] = {
        CLLocationCoordinate2DMake(13.76, 100.49),
        CLLocationCoordinate2DMake(13.76, 100.52),
        CLLocationCoordinate2DMake(13.74, 100.52),
        CLLocationCoordinate2DMake(13.74, 100.49),
    };
    
    MKPolygon *hole = [MKPolygon polygonWithCoordinates:holeCoords count:4];
    MKPolygon *polygonWithHole = [MKPolygon 
        polygonWithCoordinates:coordinates 
                         count:count 
              interiorPolygons:@[hole]];
    
    [self.mapView addOverlay:polygonWithHole];
}
```

### MKCircle (วงกลม)

```objc
- (void)addCircleOverlay {
    CLLocationCoordinate2D center = CLLocationCoordinate2DMake(13.7563, 100.5018);
    CLLocationDistance radius = 5000.0; // 5 กิโลเมตร
    
    MKCircle *circle = [MKCircle circleWithCenterCoordinate:center 
                                                     radius:radius];
    circle.title = @"รัศมี 5 กม. จากใจกลางกรุงเทพ";
    
    [self.mapView addOverlay:circle];
}
```

---

## 67.7 Custom Overlay Renderers

Renderer กำหนดการแสดงผลของ overlay

```objc
// Delegate
- (MKOverlayRenderer *)mapView:(MKMapView *)mapView 
            rendererForOverlay:(id<MKOverlay>)overlay {
    
    if ([overlay isKindOfClass:[MKPolyline class]]) {
        return [self rendererForPolyline:(MKPolyline *)overlay];
    }
    
    if ([overlay isKindOfClass:[MKPolygon class]]) {
        return [self rendererForPolygon:(MKPolygon *)overlay];
    }
    
    if ([overlay isKindOfClass:[MKCircle class]]) {
        return [self rendererForCircle:(MKCircle *)overlay];
    }
    
    return nil;
}

- (MKPolylineRenderer *)rendererForPolyline:(MKPolyline *)polyline {
    MKPolylineRenderer *renderer = [[MKPolylineRenderer alloc] 
        initWithPolyline:polyline];
    
    renderer.strokeColor = [[UIColor blueColor] colorWithAlphaComponent:0.8];
    renderer.lineWidth = 4.0;
    renderer.lineDashPattern = nil; // เส้นต่อเนื่อง
    // renderer.lineDashPattern = @[@4, @4]; // เส้นประ
    renderer.lineJoin = kCGLineJoinRound;
    renderer.lineCap = kCGLineCapRound;
    
    return renderer;
}

- (MKPolygonRenderer *)rendererForPolygon:(MKPolygon *)polygon {
    MKPolygonRenderer *renderer = [[MKPolygonRenderer alloc] 
        initWithPolygon:polygon];
    
    renderer.fillColor = [[UIColor greenColor] colorWithAlphaComponent:0.3];
    renderer.strokeColor = [[UIColor greenColor] colorWithAlphaComponent:0.8];
    renderer.lineWidth = 2.0;
    
    return renderer;
}

- (MKCircleRenderer *)rendererForCircle:(MKCircle *)circle {
    MKCircleRenderer *renderer = [[MKCircleRenderer alloc] 
        initWithCircle:circle];
    
    renderer.fillColor = [[UIColor redColor] colorWithAlphaComponent:0.2];
    renderer.strokeColor = [[UIColor redColor] colorWithAlphaComponent:0.8];
    renderer.lineWidth = 2.0;
    
    return renderer;
}
```

### Gradient Polyline Renderer (iOS 14+)

```objc
- (MKGradientPolylineRenderer *)gradientRendererForPolyline:(MKPolyline *)polyline {
    if (@available(iOS 14.0, *)) {
        MKGradientPolylineRenderer *renderer = [[MKGradientPolylineRenderer alloc] 
            initWithPolyline:polyline];
        
        renderer.lineWidth = 5.0;
        
        // Gradient colors
        [renderer setColors:@[[UIColor greenColor], 
                              [UIColor yellowColor], 
                              [UIColor redColor]]
              atLocations:@[@0.0, @0.5, @1.0]];
        
        return renderer;
    }
    return nil;
}
```

### Custom Overlay Class

```objc
// HeatmapOverlay.h
@interface HeatmapOverlay : NSObject <MKOverlay>

@property (nonatomic, assign) CLLocationCoordinate2D coordinate;
@property (nonatomic, assign) MKMapRect boundingMapRect;
@property (nonatomic, strong) NSArray<CLLocation *> *heatPoints;

- (instancetype)initWithPoints:(NSArray<CLLocation *> *)points;

@end

// HeatmapOverlay.m
@implementation HeatmapOverlay

- (instancetype)initWithPoints:(NSArray<CLLocation *> *)points {
    self = [super init];
    if (self) {
        _heatPoints = points;
        
        // คำนวณ bounding rect
        MKMapPoint minPoint = MKMapPointMake(DBL_MAX, DBL_MAX);
        MKMapPoint maxPoint = MKMapPointMake(-DBL_MAX, -DBL_MAX);
        
        for (CLLocation *loc in points) {
            MKMapPoint mapPoint = MKMapPointForCoordinate(loc.coordinate);
            minPoint.x = MIN(minPoint.x, mapPoint.x);
            minPoint.y = MIN(minPoint.y, mapPoint.y);
            maxPoint.x = MAX(maxPoint.x, mapPoint.x);
            maxPoint.y = MAX(maxPoint.y, mapPoint.y);
        }
        
        _boundingMapRect = MKMapRectMake(
            minPoint.x, minPoint.y,
            maxPoint.x - minPoint.x,
            maxPoint.y - minPoint.y
        );
        
        // Center coordinate
        CLLocationCoordinate2D minCoord = MKCoordinateForMapPoint(minPoint);
        CLLocationCoordinate2D maxCoord = MKCoordinateForMapPoint(maxPoint);
        _coordinate = CLLocationCoordinate2DMake(
            (minCoord.latitude + maxCoord.latitude) / 2.0,
            (minCoord.longitude + maxCoord.longitude) / 2.0
        );
    }
    return self;
}

@end
```

---

## 67.8 Map Routing (MKDirections)

การหาเส้นทางระหว่างสองจุด

```objc
- (void)requestDirectionsFrom:(CLLocationCoordinate2D)source 
                           to:(CLLocationCoordinate2D)destination {
    
    // สร้าง placemark
    MKPlacemark *sourcePlacemark = [[MKPlacemark alloc] 
        initWithCoordinate:source];
    MKPlacemark *destinationPlacemark = [[MKPlacemark alloc] 
        initWithCoordinate:destination];
    
    // Map items
    MKMapItem *sourceItem = [[MKMapItem alloc] 
        initWithPlacemark:sourcePlacemark];
    MKMapItem *destinationItem = [[MKMapItem alloc] 
        initWithPlacemark:destinationPlacemark];
    
    // Request
    MKDirectionsRequest *request = [[MKDirectionsRequest alloc] init];
    request.source = sourceItem;
    request.destination = destinationItem;
    request.transportType = MKDirectionsTransportTypeAutomobile;
    // MKDirectionsTransportTypeWalking
    // MKDirectionsTransportTypeTransit
    // MKDirectionsTransportTypeAny
    
    request.requestsAlternateRoutes = YES; // ขอเส้นทางเพิ่มเติม
    
    // คำนวณเส้นทาง
    MKDirections *directions = [[MKDirections alloc] initWithRequest:request];
    
    [directions calculateDirectionsWithCompletionHandler:^(
        MKDirectionsResponse *response, NSError *error) {
        
        if (error) {
            NSLog(@"Directions error: %@", error.localizedDescription);
            return;
        }
        
        NSLog(@"พบ %lu เส้นทาง", (unsigned long)response.routes.count);
        
        // แสดงเส้นทางแรก (ดีที่สุด)
        MKRoute *bestRoute = response.routes.firstObject;
        
        if (bestRoute) {
            [self displayRoute:bestRoute];
        }
        
        // แสดงข้อมูลทุกเส้นทาง
        for (MKRoute *route in response.routes) {
            [self logRouteInfo:route];
        }
    }];
}

- (void)displayRoute:(MKRoute *)route {
    // ลบ overlays เดิม
    [self.mapView removeOverlays:self.mapView.overlays];
    
    // เพิ่ม polyline
    [self.mapView addOverlay:route.polyline level:MKOverlayLevelAboveRoads];
    
    // Zoom to show route
    MKMapRect rect = route.polyline.boundingMapRect;
    [self.mapView setVisibleMapRect:rect 
                         edgePadding:UIEdgeInsetsMake(60, 40, 60, 40) 
                            animated:YES];
}

- (void)logRouteInfo:(MKRoute *)route {
    NSLog(@"=== เส้นทาง: %@ ===", route.name);
    NSLog(@"ระยะทาง: %.2f กม.", route.distance / 1000.0);
    
    NSUInteger minutes = (NSUInteger)(route.expectedTravelTime / 60.0);
    NSUInteger hours = minutes / 60;
    minutes = minutes % 60;
    
    if (hours > 0) {
        NSLog(@"เวลา: %lu ชั่วโมง %lu นาที", (unsigned long)hours, (unsigned long)minutes);
    } else {
        NSLog(@"เวลา: %lu นาที", (unsigned long)minutes);
    }
    
    NSLog(@"ประเภท: %@", [self transportTypeString:route.transportType]);
    NSLog(@"Advisory notices: %@", route.advisoryNotices);
    
    // ขั้นตอนการเดินทาง
    for (MKRouteStep *step in route.steps) {
        NSLog(@"  - %@ (%.0f ม.)", step.instructions, step.distance);
    }
}

- (NSString *)transportTypeString:(MKDirectionsTransportType)type {
    switch (type) {
        case MKDirectionsTransportTypeAutomobile: return @"รถยนต์";
        case MKDirectionsTransportTypeWalking:    return @"เดินเท้า";
        case MKDirectionsTransportTypeTransit:    return @"ขนส่งสาธารณะ";
        default: return @"ไม่ทราบ";
    }
}
```

### เปิดแผนที่ใน Apple Maps App

```objc
- (void)openInMapsApp {
    CLLocationCoordinate2D coordinate = CLLocationCoordinate2DMake(13.7563, 100.5018);
    MKPlacemark *placemark = [[MKPlacemark alloc] initWithCoordinate:coordinate];
    MKMapItem *mapItem = [[MKMapItem alloc] initWithPlacemark:placemark];
    mapItem.name = @"กรุงเทพมหานคร";
    
    // เปิดใน Maps app
    [mapItem openInMapsWithLaunchOptions:@{
        MKLaunchOptionsMapTypeKey: @(MKMapTypeStandard),
        MKLaunchOptionsDirectionsModeKey: MKLaunchOptionsDirectionsModeDefault
    }];
}

// เปิดเส้นทาง
- (void)openDirectionsToLocation:(CLLocationCoordinate2D)coordinate 
                        withName:(NSString *)name {
    MKPlacemark *placemark = [[MKPlacemark alloc] initWithCoordinate:coordinate];
    MKMapItem *destination = [[MKMapItem alloc] initWithPlacemark:placemark];
    destination.name = name;
    
    [MKMapItem openMapsWithItems:@[[MKMapItem mapItemForCurrentLocation], destination] 
                   launchOptions:@{
        MKLaunchOptionsDirectionsModeKey: MKLaunchOptionsDirectionsModeDefault
    }];
}
```

---

## 67.9 Geocoding/Reverse Geocoding

### CLGeocoder

```objc
@interface GeocodingManager : NSObject

@property (nonatomic, strong) CLGeocoder *geocoder;

@end

@implementation GeocodingManager

- (instancetype)init {
    self = [super init];
    if (self) {
        _geocoder = [[CLGeocoder alloc] init];
    }
    return self;
}

// Reverse Geocoding - พิกัด → ที่อยู่
- (void)reverseGeocodeCoordinate:(CLLocationCoordinate2D)coordinate 
                      completion:(void(^)(NSString *address, NSError *error))completion {
    
    CLLocation *location = [[CLLocation alloc] 
        initWithLatitude:coordinate.latitude 
               longitude:coordinate.longitude];
    
    [self.geocoder reverseGeocodeLocation:location 
                        completionHandler:^(NSArray<CLPlacemark *> *placemarks, 
                                            NSError *error) {
        if (error) {
            if (completion) completion(nil, error);
            return;
        }
        
        CLPlacemark *placemark = placemarks.firstObject;
        if (!placemark) {
            if (completion) completion(nil, nil);
            return;
        }
        
        // สร้างที่อยู่
        NSMutableArray *components = [NSMutableArray array];
        
        if (placemark.subThoroughfare) [components addObject:placemark.subThoroughfare];
        if (placemark.thoroughfare)    [components addObject:placemark.thoroughfare];
        if (placemark.subLocality)     [components addObject:placemark.subLocality];
        if (placemark.locality)        [components addObject:placemark.locality];
        if (placemark.administrativeArea) [components addObject:placemark.administrativeArea];
        if (placemark.country)         [components addObject:placemark.country];
        
        NSString *fullAddress = [components componentsJoinedByString:@", "];
        
        if (completion) completion(fullAddress, nil);
    }];
}

// Forward Geocoding - ที่อยู่ → พิกัด
- (void)geocodeAddress:(NSString *)address 
            completion:(void(^)(CLLocationCoordinate2D coordinate, NSError *error))completion {
    
    [self.geocoder geocodeAddressString:address 
                     completionHandler:^(NSArray<CLPlacemark *> *placemarks, 
                                         NSError *error) {
        if (error) {
            if (completion) {
                completion(kCLLocationCoordinate2DInvalid, error);
            }
            return;
        }
        
        CLPlacemark *placemark = placemarks.firstObject;
        if (!placemark) {
            if (completion) {
                completion(kCLLocationCoordinate2DInvalid, nil);
            }
            return;
        }
        
        CLLocationCoordinate2D coordinate = placemark.location.coordinate;
        NSLog(@"พิกัดของ '%@': %.6f, %.6f", 
              address, coordinate.latitude, coordinate.longitude);
        
        if (completion) completion(coordinate, nil);
    }];
}

// ยกเลิก geocoding ที่กำลังทำงาน
- (void)cancelGeocoding {
    [self.geocoder cancelGeocode];
}

@end
```

### การใช้งาน GeocodingManager

```objc
GeocodingManager *geocodingManager = [[GeocodingManager alloc] init];

// Reverse geocode
CLLocationCoordinate2D bangkokCoord = CLLocationCoordinate2DMake(13.7563, 100.5018);
[geocodingManager reverseGeocodeCoordinate:bangkokCoord 
                                 completion:^(NSString *address, NSError *error) {
    if (error) {
        NSLog(@"Error: %@", error);
    } else {
        NSLog(@"ที่อยู่: %@", address);
    }
}];

// Forward geocode
[geocodingManager geocodeAddress:@"เซ็นทรัลเวิลด์, กรุงเทพมหานคร" 
                       completion:^(CLLocationCoordinate2D coordinate, NSError *error) {
    if (!error && CLLocationCoordinate2DIsValid(coordinate)) {
        NSLog(@"พิกัด: %.6f, %.6f", coordinate.latitude, coordinate.longitude);
    }
}];
```

---

## 67.10 Search (MKLocalSearch)

การค้นหาสถานที่บนแผนที่

```objc
- (void)searchNearbyPlaces:(NSString *)query 
                  inRegion:(MKCoordinateRegion)region {
    
    MKLocalSearchRequest *request = [[MKLocalSearchRequest alloc] init];
    request.naturalLanguageQuery = query;
    request.region = region;
    
    // ระบุประเภท POI (iOS 13+)
    if (@available(iOS 13.0, *)) {
        request.pointOfInterestFilter = [MKPointOfInterestFilter 
            filterIncludingCategories:@[
                MKPointOfInterestCategoryRestaurant,
                MKPointOfInterestCategoryCafe
            ]];
        request.resultTypes = MKLocalSearchResultTypePointOfInterest;
    }
    
    MKLocalSearch *search = [[MKLocalSearch alloc] initWithRequest:request];
    
    [search startWithCompletionHandler:^(MKLocalSearchResponse *response, 
                                         NSError *error) {
        if (error) {
            NSLog(@"Search error: %@", error.localizedDescription);
            return;
        }
        
        NSLog(@"พบ %lu สถานที่", (unsigned long)response.mapItems.count);
        
        for (MKMapItem *item in response.mapItems) {
            NSLog(@"=== %@ ===", item.name);
            NSLog(@"พิกัด: %.6f, %.6f", 
                  item.placemark.coordinate.latitude,
                  item.placemark.coordinate.longitude);
            NSLog(@"โทร: %@", item.phoneNumber ?: @"ไม่มีข้อมูล");
            NSLog(@"URL: %@", item.url ?: @"ไม่มีข้อมูล");
            NSLog(@"ที่อยู่: %@", item.placemark.thoroughfare ?: @"ไม่มีข้อมูล");
            
            if (item.isCurrentLocation) {
                NSLog(@"(ตำแหน่งปัจจุบัน)");
            }
        }
        
        // เพิ่ม annotations บนแผนที่
        dispatch_async(dispatch_get_main_queue(), ^{
            [self displaySearchResults:response.mapItems];
        });
    }];
}

- (void)displaySearchResults:(NSArray<MKMapItem *> *)items {
    // ลบ annotations เดิม
    [self removeAllAnnotations];
    
    NSMutableArray *annotations = [NSMutableArray array];
    
    for (MKMapItem *item in items) {
        MKPointAnnotation *annotation = [[MKPointAnnotation alloc] init];
        annotation.coordinate = item.placemark.coordinate;
        annotation.title = item.name;
        annotation.subtitle = item.placemark.thoroughfare;
        [annotations addObject:annotation];
    }
    
    [self.mapView addAnnotations:annotations];
    
    if (annotations.count > 0) {
        [self.mapView showAnnotations:annotations animated:YES];
    }
}

// Autocomplete Search (iOS 9.3+)
- (void)autocompleteSearch:(NSString *)query {
    MKLocalSearchCompleter *completer = [[MKLocalSearchCompleter alloc] init];
    completer.delegate = self;
    completer.queryFragment = query;
    
    // จำกัดพื้นที่ค้นหา
    completer.region = self.mapView.region;
    
    // ประเภทผลลัพธ์ (iOS 13+)
    if (@available(iOS 13.0, *)) {
        completer.resultTypes = MKLocalSearchCompleterResultTypePointOfInterest | 
                               MKLocalSearchCompleterResultTypeAddress;
    }
}

// MKLocalSearchCompleterDelegate
- (void)completerDidUpdateResults:(MKLocalSearchCompleter *)completer {
    for (MKLocalSearchCompletion *result in completer.results) {
        NSLog(@"Suggestion: %@ - %@", result.title, result.subtitle);
    }
}

- (void)completer:(MKLocalSearchCompleter *)completer 
    didFailWithError:(NSError *)error {
    NSLog(@"Completer error: %@", error);
}
```

---

## 67.11 Map Region และ Zoom

### การควบคุม Map Region

```objc
// ตั้ง region
- (void)setMapRegionWithCenter:(CLLocationCoordinate2D)center 
                    latitudeDelta:(CGFloat)latDelta 
                   longitudeDelta:(CGFloat)lonDelta {
    
    MKCoordinateSpan span = MKCoordinateSpanMake(latDelta, lonDelta);
    MKCoordinateRegion region = MKCoordinateRegionMake(center, span);
    
    // ปรับให้พอดีกับ map view
    MKCoordinateRegion adjustedRegion = [self.mapView regionThatFits:region];
    
    [self.mapView setRegion:adjustedRegion animated:YES];
}

// ตั้ง region ด้วยระยะทาง
- (void)zoomToCoordinate:(CLLocationCoordinate2D)coordinate 
          withRadiusMeters:(CLLocationDistance)radius {
    
    MKCoordinateRegion region = MKCoordinateRegionMakeWithDistance(
        coordinate,
        radius * 2.0,
        radius * 2.0
    );
    
    [self.mapView setRegion:region animated:YES];
}

// Zoom in/out
- (void)zoomIn {
    MKCoordinateRegion region = self.mapView.region;
    region.span.latitudeDelta /= 2.0;
    region.span.longitudeDelta /= 2.0;
    [self.mapView setRegion:region animated:YES];
}

- (void)zoomOut {
    MKCoordinateRegion region = self.mapView.region;
    region.span.latitudeDelta = MIN(region.span.latitudeDelta * 2.0, 180.0);
    region.span.longitudeDelta = MIN(region.span.longitudeDelta * 2.0, 360.0);
    [self.mapView setRegion:region animated:YES];
}

// แปลง MKMapRect
- (void)showAllAnnotations {
    if (self.mapView.annotations.count == 0) return;
    
    MKMapRect rect = MKMapRectNull;
    for (id<MKAnnotation> annotation in self.mapView.annotations) {
        MKMapPoint point = MKMapPointForCoordinate(annotation.coordinate);
        MKMapRect pointRect = MKMapRectMake(point.x, point.y, 0, 0);
        rect = MKMapRectUnion(rect, pointRect);
    }
    
    [self.mapView setVisibleMapRect:rect 
                         edgePadding:UIEdgeInsetsMake(80, 40, 80, 40) 
                            animated:YES];
}

// Delegate - map region changed
- (void)mapView:(MKMapView *)mapView 
    regionDidChangeAnimated:(BOOL)animated {
    
    MKCoordinateRegion region = mapView.region;
    NSLog(@"Region เปลี่ยน - Center: %.4f, %.4f, Span: %.4f x %.4f", 
          region.center.latitude, 
          region.center.longitude,
          region.span.latitudeDelta, 
          region.span.longitudeDelta);
}
```

---

## 67.12 MKLocalPointsOfInterestRequest (iOS 14+)

```objc
- (void)fetchPointsOfInterest {
    if (@available(iOS 14.0, *)) {
        MKLocalPointsOfInterestRequest *request = 
            [[MKLocalPointsOfInterestRequest alloc] 
                initWithCoordinateRegion:self.mapView.region];
        
        request.pointOfInterestFilter = [MKPointOfInterestFilter 
            filterIncludingCategories:@[
                MKPointOfInterestCategoryHospital,
                MKPointOfInterestCategoryPharmacy,
                MKPointOfInterestCategoryPolice
            ]];
        
        MKLocalSearch *search = [[MKLocalSearch alloc] 
            initWithRequest:request];
        
        [search startWithCompletionHandler:^(MKLocalSearchResponse *response, 
                                              NSError *error) {
            if (!error) {
                for (MKMapItem *item in response.mapItems) {
                    NSLog(@"POI: %@ - %@", 
                          item.name, 
                          item.pointOfInterestCategory);
                }
            }
        }];
    }
}
```

---

## 67.13 แบบฝึกหัด

### แบบฝึกหัดที่ 1: Map Explorer App
สร้าง map app ที่:
- แสดงแผนที่พร้อมตำแหน่งผู้ใช้
- ให้เพิ่ม pin ด้วยการ long press
- แสดง callout พร้อมข้อมูล
- บันทึก pins ใน UserDefaults

### แบบฝึกหัดที่ 2: Route Planner
สร้าง route planner ที่:
- รับต้นทางและปลายทาง
- แสดงเส้นทาง 3 เส้นทางบนแผนที่
- แสดงระยะทางและเวลา
- Turn-by-turn directions

### แบบฝึกหัดที่ 3: Nearby Search App
สร้าง nearby search ที่:
- ค้นหาสถานที่ใกล้เคียง
- แสดง search bar พร้อม autocomplete
- แสดง results เป็น list และ map
- Filter ตามประเภท POI

### แบบฝึกหัดที่ 4: Geofence Map
สร้าง geofence map ที่:
- วาด circle overlays
- เพิ่ม/ลบ geofence ด้วย long press
- แสดง list ของ geofences
- แจ้งเตือนเมื่อเข้า/ออก

---

## สรุป

ในตอนนี้เราได้เรียนรู้:
- **MKMapView** การตั้งค่าและประเภทแผนที่
- **User Location** การแสดงตำแหน่งผู้ใช้
- **Annotations** การเพิ่มและ custom annotation views
- **Annotation Clustering** รวม pins ที่ซ้อนกัน
- **Overlays** Polyline, Polygon, Circle
- **Custom Renderers** ปรับแต่ง overlay appearance
- **MKDirections** การหาเส้นทาง
- **CLGeocoder** แปลงพิกัด/ที่อยู่
- **MKLocalSearch** ค้นหาสถานที่
- **Map Region** การควบคุม zoom และ pan

ตอนต่อไปจะเรียนรู้ Contacts และ Calendar framework

---

*หมายเหตุ: MapKit ต้องการ entitlement สำหรับ Maps ใน Xcode และ App Store Connect หากต้องการใช้ฟีเจอร์ขั้นสูง*
