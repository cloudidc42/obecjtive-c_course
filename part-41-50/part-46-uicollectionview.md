# Part 46: UICollectionView

## บทนำ

UICollectionView เป็น UI component ที่ยืดหยุ่นกว่า UITableView มาก สามารถแสดงข้อมูลในรูปแบบ grid, horizontal scroll, custom layouts และอื่นๆ ได้อย่างหลากหลาย ใช้กันอย่างแพร่หลายใน apps เช่น Photos, App Store, Spotify, Instagram

ความแตกต่างหลักจาก UITableView:
- **Layout ยืดหยุ่น**: ไม่ผูกกับแถวเดียว
- **Multiple columns**: แสดงหลายคอลัมน์ได้
- **Custom layouts**: สร้าง layout เองได้
- **Supplementary views**: header/footer แบบกำหนดเอง
- **Decoration views**: ตกแต่ง background

---

## 1. UICollectionView Setup

### 1.1 การสร้างแบบ Programmatic

```objc
// PhotoGalleryViewController.m

@interface PhotoGalleryViewController () <UICollectionViewDelegate, UICollectionViewDataSource>
@property (nonatomic, strong) UICollectionView *collectionView;
@property (nonatomic, strong) NSArray<UIImage *> *photos;
@end

@implementation PhotoGalleryViewController

- (void)viewDidLoad {
    [super viewDidLoad];
    
    [self setupCollectionView];
    [self loadPhotos];
}

- (void)setupCollectionView {
    // สร้าง layout ก่อน
    UICollectionViewFlowLayout *layout = [[UICollectionViewFlowLayout alloc] init];
    layout.scrollDirection = UICollectionViewScrollDirectionVertical;
    layout.minimumInteritemSpacing = 2;
    layout.minimumLineSpacing = 2;
    
    // คำนวณขนาด item (3 columns)
    CGFloat width = (UIScreen.mainScreen.bounds.size.width - 4) / 3;
    layout.itemSize = CGSizeMake(width, width); // square
    
    // สร้าง collection view
    self.collectionView = [[UICollectionView alloc] initWithFrame:self.view.bounds 
                                             collectionViewLayout:layout];
    self.collectionView.backgroundColor = [UIColor systemBackgroundColor];
    self.collectionView.delegate = self;
    self.collectionView.dataSource = self;
    self.collectionView.autoresizingMask = UIViewAutoresizingFlexibleWidth | UIViewAutoresizingFlexibleHeight;
    
    // Register cells
    [self.collectionView registerClass:[PhotoCell class] forCellWithReuseIdentifier:@"PhotoCell"];
    
    [self.view addSubview:self.collectionView];
}

@end
```

### 1.2 การ Register Cell Types

```objc
// Register โดยใช้ Class
[self.collectionView registerClass:[MyCell class] forCellWithReuseIdentifier:@"MyCell"];

// Register โดยใช้ NIB
UINib *nib = [UINib nibWithNibName:@"MyCell" bundle:nil];
[self.collectionView registerNib:nib forCellWithReuseIdentifier:@"MyCell"];

// Register Supplementary Views (headers/footers)
[self.collectionView registerClass:[HeaderView class] 
        forSupplementaryViewOfKind:UICollectionElementKindSectionHeader 
               withReuseIdentifier:@"Header"];

[self.collectionView registerClass:[FooterView class] 
        forSupplementaryViewOfKind:UICollectionElementKindSectionFooter 
               withReuseIdentifier:@"Footer"];
```

---

## 2. UICollectionViewFlowLayout

FlowLayout เป็น layout พื้นฐานที่ใช้บ่อยที่สุด จัดวาง items เป็น grid

### 2.1 คุณสมบัติหลัก

```objc
UICollectionViewFlowLayout *layout = [[UICollectionViewFlowLayout alloc] init];

// ทิศทาง scroll
layout.scrollDirection = UICollectionViewScrollDirectionVertical;   // ลง
// layout.scrollDirection = UICollectionViewScrollDirectionHorizontal; // ขวา

// ขนาด item (fixed)
layout.itemSize = CGSizeMake(100, 100);

// ระยะห่างระหว่าง items ใน row เดียวกัน (แนวนอน)
layout.minimumInteritemSpacing = 8;

// ระยะห่างระหว่าง rows (แนวตั้ง)
layout.minimumLineSpacing = 8;

// Padding รอบๆ ทุก section
layout.sectionInset = UIEdgeInsetsMake(16, 16, 16, 16);

// ขนาด header/footer
layout.headerReferenceSize = CGSizeMake(0, 50);  // width ถูก ignore สำหรับ vertical
layout.footerReferenceSize = CGSizeMake(0, 30);

// Pin header (sticky)
layout.sectionHeadersPinToVisibleBounds = YES;  // iOS 9+
```

### 2.2 UICollectionViewDelegateFlowLayout

ใช้เมื่อต้องการ item sizes ที่ต่างกันในแต่ละ index path

```objc
@interface FlexibleGridVC () <UICollectionViewDelegateFlowLayout>
@end

@implementation FlexibleGridVC

// กำหนดขนาดแต่ละ item
- (CGSize)collectionView:(UICollectionView *)collectionView 
                  layout:(UICollectionViewLayout *)collectionViewLayout 
  sizeForItemAtIndexPath:(NSIndexPath *)indexPath {
    
    // สลับขนาดใหญ่/เล็ก
    if (indexPath.item % 3 == 0) {
        return CGSizeMake(collectionView.bounds.size.width - 32, 200);
    }
    
    CGFloat width = (collectionView.bounds.size.width - 40) / 2;
    return CGSizeMake(width, width);
}

// Insets สำหรับแต่ละ section
- (UIEdgeInsets)collectionView:(UICollectionView *)collectionView 
                        layout:(UICollectionViewLayout *)collectionViewLayout 
        insetForSectionAtIndex:(NSInteger)section {
    return UIEdgeInsetsMake(16, 16, 16, 16);
}

// ระยะห่างแนวตั้งสำหรับแต่ละ section
- (CGFloat)collectionView:(UICollectionView *)collectionView 
                   layout:(UICollectionViewLayout *)collectionViewLayout 
minimumLineSpacingForSectionAtIndex:(NSInteger)section {
    return 8;
}

// ระยะห่างแนวนอน
- (CGFloat)collectionView:(UICollectionView *)collectionView 
                   layout:(UICollectionViewLayout *)collectionViewLayout 
minimumInteritemSpacingForSectionAtIndex:(NSInteger)section {
    return 8;
}

// ขนาด header สำหรับแต่ละ section
- (CGSize)collectionView:(UICollectionView *)collectionView 
                  layout:(UICollectionViewLayout *)collectionViewLayout 
referenceSizeForHeaderInSection:(NSInteger)section {
    return CGSizeMake(0, 50);
}

@end
```

---

## 3. UICollectionViewDataSource

### 3.1 Methods พื้นฐาน

```objc
// จำนวน sections (optional, default = 1)
- (NSInteger)numberOfSectionsInCollectionView:(UICollectionView *)collectionView {
    return self.sections.count;
}

// จำนวน items ใน section (บังคับ)
- (NSInteger)collectionView:(UICollectionView *)collectionView 
     numberOfItemsInSection:(NSInteger)section {
    return self.photos.count;
}

// สร้าง cell (บังคับ)
- (UICollectionViewCell *)collectionView:(UICollectionView *)collectionView 
              cellForItemAtIndexPath:(NSIndexPath *)indexPath {
    
    PhotoCell *cell = [collectionView dequeueReusableCellWithReuseIdentifier:@"PhotoCell" 
                                                                forIndexPath:indexPath];
    UIImage *photo = self.photos[indexPath.item];
    [cell configureWithImage:photo];
    return cell;
}

// Supplementary views (headers/footers)
- (UICollectionReusableView *)collectionView:(UICollectionView *)collectionView 
           viewForSupplementaryElementOfKind:(NSString *)kind 
                                 atIndexPath:(NSIndexPath *)indexPath {
    
    if ([kind isEqualToString:UICollectionElementKindSectionHeader]) {
        SectionHeaderView *header = [collectionView dequeueReusableSupplementaryViewOfKind:kind 
                                                                       withReuseIdentifier:@"Header" 
                                                                              forIndexPath:indexPath];
        header.titleLabel.text = self.sectionTitles[indexPath.section];
        return header;
    }
    
    SectionFooterView *footer = [collectionView dequeueReusableSupplementaryViewOfKind:kind 
                                                                   withReuseIdentifier:@"Footer" 
                                                                          forIndexPath:indexPath];
    return footer;
}

// Reorder support (iOS 9+)
- (BOOL)collectionView:(UICollectionView *)collectionView 
    canMoveItemAtIndexPath:(NSIndexPath *)indexPath {
    return YES;
}

- (void)collectionView:(UICollectionView *)collectionView 
    moveItemAtIndexPath:(NSIndexPath *)sourceIndexPath 
           toIndexPath:(NSIndexPath *)destinationIndexPath {
    UIImage *photo = self.photos[sourceIndexPath.item];
    NSMutableArray *mutable = [self.photos mutableCopy];
    [mutable removeObjectAtIndex:sourceIndexPath.item];
    [mutable insertObject:photo atIndex:destinationIndexPath.item];
    self.photos = [mutable copy];
}
```

---

## 4. UICollectionViewDelegate

```objc
// เมื่อ tap cell
- (void)collectionView:(UICollectionView *)collectionView 
didSelectItemAtIndexPath:(NSIndexPath *)indexPath {
    UIImage *photo = self.photos[indexPath.item];
    // แสดง full screen
    [self showPhotoViewer:photo];
}

// เมื่อ deselect
- (void)collectionView:(UICollectionView *)collectionView 
didDeselectItemAtIndexPath:(NSIndexPath *)indexPath {
    // ทำอะไรบางอย่าง
}

// ก่อนที่จะ select (return NO เพื่อยกเลิก)
- (BOOL)collectionView:(UICollectionView *)collectionView 
shouldSelectItemAtIndexPath:(NSIndexPath *)indexPath {
    return YES;
}

// Context menu (iOS 13+)
- (UIContextMenuConfiguration *)collectionView:(UICollectionView *)collectionView 
contextMenuConfigurationForItemAtIndexPath:(NSIndexPath *)indexPath 
                                         point:(CGPoint)point {
    return [UIContextMenuConfiguration configurationWithIdentifier:nil
                                                   previewProvider:nil
                                                    actionProvider:^UIMenu *(NSArray *suggestions) {
        UIAction *share = [UIAction actionWithTitle:@"แชร์" 
                                              image:[UIImage systemImageNamed:@"square.and.arrow.up"]
                                         identifier:nil
                                            handler:^(__kindof UIAction *action) {
            [self sharePhotoAtIndexPath:indexPath];
        }];
        
        UIAction *delete = [UIAction actionWithTitle:@"ลบ"
                                               image:[UIImage systemImageNamed:@"trash"]
                                          identifier:nil
                                             handler:^(__kindof UIAction *action) {
            [self deletePhotoAtIndexPath:indexPath];
        }];
        delete.attributes = UIMenuElementAttributesDestructive;
        
        return [UIMenu menuWithTitle:@"" children:@[share, delete]];
    }];
}
```

---

## 5. Custom UICollectionViewCell

### 5.1 สร้าง Photo Cell

```objc
// PhotoCell.h
#import <UIKit/UIKit.h>

@interface PhotoCell : UICollectionViewCell

@property (nonatomic, strong) UIImageView *imageView;
@property (nonatomic, strong) UILabel *captionLabel;
@property (nonatomic, strong) UIView *overlayView;

- (void)configureWithImage:(UIImage *)image caption:(NSString *)caption;

@end
```

```objc
// PhotoCell.m
#import "PhotoCell.h"

@implementation PhotoCell

- (instancetype)initWithFrame:(CGRect)frame {
    self = [super initWithFrame:frame];
    if (self) {
        [self setupUI];
    }
    return self;
}

- (void)setupUI {
    // Image view เต็ม cell
    self.imageView = [[UIImageView alloc] init];
    self.imageView.translatesAutoresizingMaskIntoConstraints = NO;
    self.imageView.contentMode = UIViewContentModeScaleAspectFill;
    self.imageView.clipsToBounds = YES;
    self.imageView.backgroundColor = [UIColor systemGray5Color];
    [self.contentView addSubview:self.imageView];
    
    // Overlay gradient
    self.overlayView = [[UIView alloc] init];
    self.overlayView.translatesAutoresizingMaskIntoConstraints = NO;
    [self.contentView addSubview:self.overlayView];
    
    // Caption label ด้านล่าง
    self.captionLabel = [[UILabel alloc] init];
    self.captionLabel.translatesAutoresizingMaskIntoConstraints = NO;
    self.captionLabel.font = [UIFont systemFontOfSize:11 weight:UIFontWeightMedium];
    self.captionLabel.textColor = [UIColor whiteColor];
    self.captionLabel.numberOfLines = 2;
    [self.contentView addSubview:self.captionLabel];
    
    [NSLayoutConstraint activateConstraints:@[
        // Image fills entire cell
        [self.imageView.topAnchor constraintEqualToAnchor:self.contentView.topAnchor],
        [self.imageView.bottomAnchor constraintEqualToAnchor:self.contentView.bottomAnchor],
        [self.imageView.leadingAnchor constraintEqualToAnchor:self.contentView.leadingAnchor],
        [self.imageView.trailingAnchor constraintEqualToAnchor:self.contentView.trailingAnchor],
        
        // Overlay fills entire cell
        [self.overlayView.topAnchor constraintEqualToAnchor:self.contentView.topAnchor],
        [self.overlayView.bottomAnchor constraintEqualToAnchor:self.contentView.bottomAnchor],
        [self.overlayView.leadingAnchor constraintEqualToAnchor:self.contentView.leadingAnchor],
        [self.overlayView.trailingAnchor constraintEqualToAnchor:self.contentView.trailingAnchor],
        
        // Caption at bottom
        [self.captionLabel.bottomAnchor constraintEqualToAnchor:self.contentView.bottomAnchor constant:-6],
        [self.captionLabel.leadingAnchor constraintEqualToAnchor:self.contentView.leadingAnchor constant:6],
        [self.captionLabel.trailingAnchor constraintEqualToAnchor:self.contentView.trailingAnchor constant:-6],
    ]];
    
    // Add gradient
    [self setupGradient];
    
    // Selection effect
    UIView *selectedBG = [[UIView alloc] init];
    selectedBG.backgroundColor = [[UIColor systemBlueColor] colorWithAlphaComponent:0.3];
    self.selectedBackgroundView = selectedBG;
    
    // Corner radius
    self.contentView.layer.cornerRadius = 4;
    self.contentView.clipsToBounds = YES;
}

- (void)setupGradient {
    CAGradientLayer *gradient = [CAGradientLayer layer];
    gradient.colors = @[
        (__bridge id)[UIColor clearColor].CGColor,
        (__bridge id)[[UIColor blackColor] colorWithAlphaComponent:0.7].CGColor
    ];
    gradient.locations = @[@0.5, @1.0];
    self.overlayView.layer.insertSublayer:gradient atIndex:0];
    
    // อัปเดต frame ใน layoutSubviews
}

- (void)layoutSubviews {
    [super layoutSubviews];
    // อัปเดต gradient frame
    for (CALayer *layer in self.overlayView.layer.sublayers) {
        if ([layer isKindOfClass:[CAGradientLayer class]]) {
            layer.frame = self.overlayView.bounds;
        }
    }
}

- (void)configureWithImage:(UIImage *)image caption:(NSString *)caption {
    self.imageView.image = image;
    self.captionLabel.text = caption;
    self.captionLabel.hidden = (caption.length == 0);
    self.overlayView.hidden = (caption.length == 0);
}

- (void)prepareForReuse {
    [super prepareForReuse];
    self.imageView.image = nil;
    self.captionLabel.text = nil;
}

@end
```

### 5.2 App Icon-Style Cell

```objc
// AppIconCell.h
@interface AppIconCell : UICollectionViewCell
@property (nonatomic, strong) UIImageView *iconView;
@property (nonatomic, strong) UILabel *nameLabel;
- (void)configureWithName:(NSString *)name icon:(UIImage *)icon;
@end

// AppIconCell.m
@implementation AppIconCell

- (instancetype)initWithFrame:(CGRect)frame {
    self = [super initWithFrame:frame];
    if (self) {
        self.iconView = [[UIImageView alloc] init];
        self.iconView.translatesAutoresizingMaskIntoConstraints = NO;
        self.iconView.layer.cornerRadius = 14;
        self.iconView.clipsToBounds = YES;
        self.iconView.contentMode = UIViewContentModeScaleAspectFit;
        self.iconView.backgroundColor = [UIColor systemGray5Color];
        [self.contentView addSubview:self.iconView];
        
        self.nameLabel = [[UILabel alloc] init];
        self.nameLabel.translatesAutoresizingMaskIntoConstraints = NO;
        self.nameLabel.textAlignment = NSTextAlignmentCenter;
        self.nameLabel.font = [UIFont systemFontOfSize:11];
        self.nameLabel.numberOfLines = 2;
        [self.contentView addSubview:self.nameLabel];
        
        // Icon size = 80% of cell width, square
        CGFloat iconSize = frame.size.width * 0.8;
        
        [NSLayoutConstraint activateConstraints:@[
            [self.iconView.topAnchor constraintEqualToAnchor:self.contentView.topAnchor constant:4],
            [self.iconView.centerXAnchor constraintEqualToAnchor:self.contentView.centerXAnchor],
            [self.iconView.widthAnchor constraintEqualToConstant:iconSize],
            [self.iconView.heightAnchor constraintEqualToConstant:iconSize],
            
            [self.nameLabel.topAnchor constraintEqualToAnchor:self.iconView.bottomAnchor constant:4],
            [self.nameLabel.leadingAnchor constraintEqualToAnchor:self.contentView.leadingAnchor],
            [self.nameLabel.trailingAnchor constraintEqualToAnchor:self.contentView.trailingAnchor],
        ]];
    }
    return self;
}

- (void)configureWithName:(NSString *)name icon:(UIImage *)icon {
    self.nameLabel.text = name;
    self.iconView.image = icon;
}

@end
```

---

## 6. Section Headers และ Footers

### 6.1 สร้าง Custom Header View

```objc
// SectionHeaderView.h
@interface SectionHeaderView : UICollectionReusableView
@property (nonatomic, strong) UILabel *titleLabel;
@property (nonatomic, strong) UIButton *seeAllButton;
- (void)configureWithTitle:(NSString *)title showSeeAll:(BOOL)show;
@end

// SectionHeaderView.m
@implementation SectionHeaderView

- (instancetype)initWithFrame:(CGRect)frame {
    self = [super initWithFrame:frame];
    if (self) {
        self.backgroundColor = [UIColor systemBackgroundColor];
        
        self.titleLabel = [[UILabel alloc] init];
        self.titleLabel.translatesAutoresizingMaskIntoConstraints = NO;
        self.titleLabel.font = [UIFont boldSystemFontOfSize:20];
        [self addSubview:self.titleLabel];
        
        self.seeAllButton = [UIButton buttonWithType:UIButtonTypeSystem];
        self.seeAllButton.translatesAutoresizingMaskIntoConstraints = NO;
        [self.seeAllButton setTitle:@"ดูทั้งหมด" forState:UIControlStateNormal];
        [self addSubview:self.seeAllButton];
        
        [NSLayoutConstraint activateConstraints:@[
            [self.titleLabel.leadingAnchor constraintEqualToAnchor:self.leadingAnchor constant:16],
            [self.titleLabel.centerYAnchor constraintEqualToAnchor:self.centerYAnchor],
            
            [self.seeAllButton.trailingAnchor constraintEqualToAnchor:self.trailingAnchor constant:-16],
            [self.seeAllButton.centerYAnchor constraintEqualToAnchor:self.centerYAnchor],
        ]];
    }
    return self;
}

- (void)configureWithTitle:(NSString *)title showSeeAll:(BOOL)show {
    self.titleLabel.text = title;
    self.seeAllButton.hidden = !show;
}

@end
```

---

## 7. Custom UICollectionViewLayout

สร้าง layout เองสำหรับ Pinterest-style หรือ Waterfall layout

### 7.1 WaterfallLayout

```objc
// WaterfallLayout.h
#import <UIKit/UIKit.h>

@protocol WaterfallLayoutDelegate <NSObject>
- (CGFloat)collectionView:(UICollectionView *)collectionView 
                   layout:(UICollectionViewLayout *)layout 
    heightForItemAtIndexPath:(NSIndexPath *)indexPath 
                withWidth:(CGFloat)width;
@end

@interface WaterfallLayout : UICollectionViewLayout

@property (nonatomic, weak) id<WaterfallLayoutDelegate> delegate;
@property (nonatomic, assign) NSInteger numberOfColumns;
@property (nonatomic, assign) CGFloat cellPadding;

@end
```

```objc
// WaterfallLayout.m
#import "WaterfallLayout.h"

@interface WaterfallLayout ()
@property (nonatomic, strong) NSMutableArray<UICollectionViewLayoutAttributes *> *cache;
@property (nonatomic, assign) CGFloat contentHeight;
@property (nonatomic, readonly) CGFloat contentWidth;
@end

@implementation WaterfallLayout

- (instancetype)init {
    self = [super init];
    if (self) {
        _numberOfColumns = 2;
        _cellPadding = 6;
        _cache = [NSMutableArray array];
    }
    return self;
}

- (CGFloat)contentWidth {
    UIEdgeInsets insets = self.collectionView.contentInset;
    return CGRectGetWidth(self.collectionView.bounds) - insets.left - insets.right;
}

- (CGSize)collectionViewContentSize {
    return CGSizeMake(self.contentWidth, self.contentHeight);
}

- (void)prepareLayout {
    if (self.cache.count > 0) return;
    
    CGFloat columnWidth = self.contentWidth / self.numberOfColumns;
    NSMutableArray *xOffsets = [NSMutableArray array];
    for (NSInteger i = 0; i < self.numberOfColumns; i++) {
        [xOffsets addObject:@(i * columnWidth)];
    }
    
    NSMutableArray *yOffsets = [NSMutableArray arrayWithCapacity:self.numberOfColumns];
    for (NSInteger i = 0; i < self.numberOfColumns; i++) {
        [yOffsets addObject:@(0)];
    }
    
    NSInteger column = 0;
    NSInteger itemCount = [self.collectionView numberOfItemsInSection:0];
    
    for (NSInteger item = 0; item < itemCount; item++) {
        NSIndexPath *indexPath = [NSIndexPath indexPathForItem:item inSection:0];
        
        CGFloat photoWidth = columnWidth - self.cellPadding * 2;
        CGFloat photoHeight = [self.delegate collectionView:self.collectionView 
                                                     layout:self 
                                      heightForItemAtIndexPath:indexPath 
                                                  withWidth:photoWidth];
        CGFloat height = self.cellPadding * 2 + photoHeight;
        
        CGFloat xOffset = [xOffsets[column] floatValue];
        CGFloat yOffset = [yOffsets[column] floatValue];
        
        CGRect frame = CGRectMake(xOffset, yOffset, columnWidth, height);
        CGRect insetFrame = CGRectInset(frame, self.cellPadding, self.cellPadding);
        
        UICollectionViewLayoutAttributes *attrs = 
            [UICollectionViewLayoutAttributes layoutAttributesForCellWithIndexPath:indexPath];
        attrs.frame = insetFrame;
        [self.cache addObject:attrs];
        
        self.contentHeight = MAX(self.contentHeight, CGRectGetMaxY(frame));
        yOffsets[column] = @([yOffsets[column] floatValue] + height);
        
        // เลือก column ที่สั้นที่สุด
        column = 0;
        CGFloat minY = [yOffsets[0] floatValue];
        for (NSInteger c = 1; c < self.numberOfColumns; c++) {
            if ([yOffsets[c] floatValue] < minY) {
                minY = [yOffsets[c] floatValue];
                column = c;
            }
        }
    }
}

- (NSArray<UICollectionViewLayoutAttributes *> *)layoutAttributesForElementsInRect:(CGRect)rect {
    NSMutableArray *visibleAttributes = [NSMutableArray array];
    for (UICollectionViewLayoutAttributes *attrs in self.cache) {
        if (CGRectIntersectsRect(attrs.frame, rect)) {
            [visibleAttributes addObject:attrs];
        }
    }
    return visibleAttributes;
}

- (UICollectionViewLayoutAttributes *)layoutAttributesForItemAtIndexPath:(NSIndexPath *)indexPath {
    return self.cache[indexPath.item];
}

- (void)invalidateLayout {
    [super invalidateLayout];
    [self.cache removeAllObjects];
    self.contentHeight = 0;
}

@end
```

### 7.2 Horizontal Carousel Layout

```objc
// CarouselLayout.h
@interface CarouselLayout : UICollectionViewFlowLayout
@end

// CarouselLayout.m
@implementation CarouselLayout

- (instancetype)init {
    self = [super init];
    if (self) {
        self.scrollDirection = UICollectionViewScrollDirectionHorizontal;
        self.minimumLineSpacing = 16;
    }
    return self;
}

- (BOOL)shouldInvalidateLayoutForBoundsChange:(CGRect)newBounds {
    return YES; // อัปเดตทุกครั้งที่ scroll
}

// ทำให้ item ตรงกลางใหญ่กว่า
- (NSArray<UICollectionViewLayoutAttributes *> *)layoutAttributesForElementsInRect:(CGRect)rect {
    NSArray *attrs = [super layoutAttributesForElementsInRect:rect];
    CGRect visibleRect = CGRectMake(self.collectionView.contentOffset.x, 
                                    self.collectionView.contentOffset.y,
                                    self.collectionView.bounds.size.width,
                                    self.collectionView.bounds.size.height);
    CGFloat centerX = CGRectGetMidX(visibleRect);
    
    for (UICollectionViewLayoutAttributes *attr in attrs) {
        CGFloat distance = fabs(attr.center.x - centerX);
        CGFloat maxDistance = self.collectionView.bounds.size.width / 2 + self.itemSize.width / 2;
        CGFloat normalizedDistance = MIN(distance / maxDistance, 1.0);
        
        CGFloat scale = 1.0 - normalizedDistance * 0.2;
        attr.transform = CGAffineTransformMakeScale(scale, scale);
        attr.alpha = 1.0 - normalizedDistance * 0.5;
    }
    
    return attrs;
}

// Snap to center
- (CGPoint)targetContentOffsetForProposedContentOffset:(CGPoint)proposedOffset 
                                        withScrollingVelocity:(CGPoint)velocity {
    CGRect targetRect = CGRectMake(proposedOffset.x, 0, 
                                   self.collectionView.bounds.size.width,
                                   self.collectionView.bounds.size.height);
    
    NSArray *attrs = [super layoutAttributesForElementsInRect:targetRect];
    CGFloat centerX = proposedOffset.x + self.collectionView.bounds.size.width / 2;
    
    CGFloat minDistance = CGFLOAT_MAX;
    CGFloat targetX = proposedOffset.x;
    
    for (UICollectionViewLayoutAttributes *attr in attrs) {
        CGFloat distance = fabs(attr.center.x - centerX);
        if (distance < minDistance) {
            minDistance = distance;
            targetX = attr.center.x - self.collectionView.bounds.size.width / 2;
        }
    }
    
    return CGPointMake(targetX, proposedOffset.y);
}

@end
```

---

## 8. Supplementary Views

### 8.1 Registration และ Usage

```objc
// Register
[self.collectionView registerClass:[CategoryHeader class] 
        forSupplementaryViewOfKind:UICollectionElementKindSectionHeader 
               withReuseIdentifier:@"CategoryHeader"];

// DataSource method
- (UICollectionReusableView *)collectionView:(UICollectionView *)collectionView 
           viewForSupplementaryElementOfKind:(NSString *)kind 
                                 atIndexPath:(NSIndexPath *)indexPath {
    
    if ([kind isEqualToString:UICollectionElementKindSectionHeader]) {
        CategoryHeader *header = [collectionView 
            dequeueReusableSupplementaryViewOfKind:kind 
                               withReuseIdentifier:@"CategoryHeader" 
                                      forIndexPath:indexPath];
        header.titleLabel.text = self.categories[indexPath.section];
        return header;
    }
    
    // Footer
    SectionFooter *footer = [collectionView 
        dequeueReusableSupplementaryViewOfKind:kind 
                           withReuseIdentifier:@"Footer" 
                                  forIndexPath:indexPath];
    return footer;
}
```

### 8.2 Custom Supplementary Kind

```objc
NSString * const ElementKindBadge = @"ElementKindBadge";

// Register
[self.collectionView registerClass:[BadgeView class] 
        forSupplementaryViewOfKind:ElementKindBadge 
               withReuseIdentifier:@"Badge"];

// ใน custom layout - สร้าง attributes สำหรับ badge
- (UICollectionViewLayoutAttributes *)layoutAttributesForSupplementaryViewOfKind:(NSString *)elementKind 
                                                                     atIndexPath:(NSIndexPath *)indexPath {
    if ([elementKind isEqualToString:ElementKindBadge]) {
        UICollectionViewLayoutAttributes *badgeAttrs = 
            [UICollectionViewLayoutAttributes layoutAttributesForSupplementaryViewOfKind:elementKind 
                                                                            withIndexPath:indexPath];
        // วาง badge ที่มุมบนขวาของ cell
        UICollectionViewLayoutAttributes *cellAttrs = [self layoutAttributesForItemAtIndexPath:indexPath];
        badgeAttrs.frame = CGRectMake(CGRectGetMaxX(cellAttrs.frame) - 12, 
                                      cellAttrs.frame.origin.y - 6,
                                      24, 24);
        return badgeAttrs;
    }
    return nil;
}
```

---

## 9. Animations

### 9.1 Insert/Delete Items

```objc
- (void)insertItem:(id)item {
    [self.items addObject:item];
    NSIndexPath *indexPath = [NSIndexPath indexPathForItem:self.items.count - 1 inSection:0];
    [self.collectionView insertItemsAtIndexPaths:@[indexPath]];
}

- (void)deleteItemAtIndex:(NSInteger)index {
    [self.items removeObjectAtIndex:index];
    NSIndexPath *indexPath = [NSIndexPath indexPathForItem:index inSection:0];
    [self.collectionView deleteItemsAtIndexPaths:@[indexPath]];
}

// Batch updates
[self.collectionView performBatchUpdates:^{
    // Insert
    NSIndexPath *newPath = [NSIndexPath indexPathForItem:0 inSection:0];
    [self.collectionView insertItemsAtIndexPaths:@[newPath]];
    
    // Delete
    NSIndexPath *oldPath = [NSIndexPath indexPathForItem:5 inSection:0];
    [self.collectionView deleteItemsAtIndexPaths:@[oldPath]];
    
    // Move
    [self.collectionView moveItemAtIndexPath:[NSIndexPath indexPathForItem:1 inSection:0]
                                toIndexPath:[NSIndexPath indexPathForItem:3 inSection:0]];
    
} completion:^(BOOL finished) {
    NSLog(@"Animation complete");
}];
```

### 9.2 Animated Layout Changes

```objc
- (void)switchToGridLayout {
    UICollectionViewFlowLayout *gridLayout = [[UICollectionViewFlowLayout alloc] init];
    CGFloat width = (self.collectionView.bounds.size.width - 4) / 3;
    gridLayout.itemSize = CGSizeMake(width, width);
    gridLayout.minimumInteritemSpacing = 2;
    gridLayout.minimumLineSpacing = 2;
    
    [self.collectionView setCollectionViewLayout:gridLayout animated:YES];
}

- (void)switchToListLayout {
    UICollectionViewFlowLayout *listLayout = [[UICollectionViewFlowLayout alloc] init];
    listLayout.itemSize = CGSizeMake(self.collectionView.bounds.size.width, 80);
    listLayout.minimumLineSpacing = 0;
    
    [self.collectionView setCollectionViewLayout:listLayout animated:YES];
}
```

### 9.3 Custom Cell Animation

```objc
// Animation เมื่อ cell ปรากฏ
- (void)collectionView:(UICollectionView *)collectionView 
       willDisplayCell:(UICollectionViewCell *)cell 
    forItemAtIndexPath:(NSIndexPath *)indexPath {
    
    // Scale animation
    cell.transform = CGAffineTransformMakeScale(0.8, 0.8);
    cell.alpha = 0;
    
    [UIView animateWithDuration:0.3 
                          delay:0.05 * indexPath.item 
                        options:UIViewAnimationOptionCurveEaseOut 
                     animations:^{
        cell.transform = CGAffineTransformIdentity;
        cell.alpha = 1;
    } completion:nil];
}
```

---

## 10. Multi-Selection

```objc
// เปิด multi-selection
self.collectionView.allowsMultipleSelection = YES;

// จัดการ select/deselect
- (void)collectionView:(UICollectionView *)collectionView 
didSelectItemAtIndexPath:(NSIndexPath *)indexPath {
    [self updateSelectionUI];
}

- (void)collectionView:(UICollectionView *)collectionView 
didDeselectItemAtIndexPath:(NSIndexPath *)indexPath {
    [self updateSelectionUI];
}

- (void)updateSelectionUI {
    NSArray *selected = self.collectionView.indexPathsForSelectedItems;
    self.navigationItem.title = [NSString stringWithFormat:@"เลือก %ld รายการ", (long)selected.count];
    self.deleteButton.enabled = selected.count > 0;
}

- (void)deleteSelectedItems {
    NSArray *selectedPaths = [self.collectionView.indexPathsForSelectedItems 
                             sortedArrayUsingComparator:^NSComparisonResult(NSIndexPath *a, NSIndexPath *b) {
        return [b compare:a]; // reverse order
    }];
    
    [self.collectionView performBatchUpdates:^{
        for (NSIndexPath *path in selectedPaths) {
            [self.items removeObjectAtIndex:path.item];
            [self.collectionView deleteItemsAtIndexPaths:@[path]];
        }
    } completion:nil];
}
```

---

## 11. ตัวอย่างสมบูรณ์: Photo Gallery App

### 11.1 Photo Model

```objc
// Photo.h
@interface Photo : NSObject
@property (nonatomic, copy) NSString *identifier;
@property (nonatomic, copy) NSString *title;
@property (nonatomic, copy) NSString *photographer;
@property (nonatomic, strong) UIImage *image;
@property (nonatomic, copy) NSURL *imageURL;
@property (nonatomic, assign) CGFloat aspectRatio; // height/width
@property (nonatomic, assign) BOOL isFavorite;
@end
```

### 11.2 Gallery View Controller

```objc
// GalleryViewController.m
#import "GalleryViewController.h"
#import "PhotoCell.h"
#import "SectionHeaderView.h"
#import "WaterfallLayout.h"
#import "Photo.h"
#import "PhotoDetailViewController.h"

typedef NS_ENUM(NSInteger, LayoutMode) {
    LayoutModeGrid,
    LayoutModeWaterfall,
    LayoutModeHorizontal
};

@interface GalleryViewController () <UICollectionViewDelegate, UICollectionViewDataSource,
                                     UICollectionViewDelegateFlowLayout,
                                     WaterfallLayoutDelegate,
                                     UISearchResultsUpdating>

@property (nonatomic, strong) UICollectionView *collectionView;
@property (nonatomic, strong) NSArray<NSArray<Photo *> *> *photoSections;
@property (nonatomic, strong) NSArray<NSString *> *sectionTitles;
@property (nonatomic, assign) LayoutMode layoutMode;
@property (nonatomic, strong) UISearchController *searchController;
@property (nonatomic, strong) NSArray<Photo *> *filteredPhotos;

@end

@implementation GalleryViewController

- (void)viewDidLoad {
    [super viewDidLoad];
    
    self.title = @"Photos";
    self.layoutMode = LayoutModeGrid;
    
    [self setupNavigationBar];
    [self setupCollectionView];
    [self setupSearchController];
    [self loadPhotos];
}

- (void)setupNavigationBar {
    // Layout toggle button
    UIBarButtonItem *layoutBtn = [[UIBarButtonItem alloc] 
                                 initWithImage:[UIImage systemImageNamed:@"square.grid.2x2"]
                                         style:UIBarButtonItemStylePlain
                                        target:self
                                        action:@selector(toggleLayout)];
    
    // Select button
    UIBarButtonItem *selectBtn = [[UIBarButtonItem alloc] 
                                 initWithTitle:@"เลือก"
                                         style:UIBarButtonItemStylePlain
                                        target:self
                                        action:@selector(toggleSelection)];
    
    self.navigationItem.rightBarButtonItems = @[layoutBtn, selectBtn];
}

- (void)setupCollectionView {
    UICollectionViewFlowLayout *layout = [self createGridLayout];
    
    self.collectionView = [[UICollectionView alloc] initWithFrame:self.view.bounds 
                                             collectionViewLayout:layout];
    self.collectionView.backgroundColor = [UIColor systemBackgroundColor];
    self.collectionView.delegate = self;
    self.collectionView.dataSource = self;
    self.collectionView.autoresizingMask = UIViewAutoresizingFlexibleWidth | UIViewAutoresizingFlexibleHeight;
    self.collectionView.allowsMultipleSelection = NO;
    
    [self.collectionView registerClass:[PhotoCell class] forCellWithReuseIdentifier:@"PhotoCell"];
    [self.collectionView registerClass:[SectionHeaderView class]
            forSupplementaryViewOfKind:UICollectionElementKindSectionHeader 
                   withReuseIdentifier:@"Header"];
    
    [self.view addSubview:self.collectionView];
}

- (void)setupSearchController {
    self.searchController = [[UISearchController alloc] initWithSearchResultsController:nil];
    self.searchController.searchResultsUpdater = self;
    self.searchController.obscuresBackgroundDuringPresentation = NO;
    self.searchController.searchBar.placeholder = @"ค้นหาภาพ...";
    self.navigationItem.searchController = self.searchController;
}

- (UICollectionViewFlowLayout *)createGridLayout {
    UICollectionViewFlowLayout *layout = [[UICollectionViewFlowLayout alloc] init];
    layout.scrollDirection = UICollectionViewScrollDirectionVertical;
    CGFloat width = (UIScreen.mainScreen.bounds.size.width - 4) / 3;
    layout.itemSize = CGSizeMake(width, width);
    layout.minimumInteritemSpacing = 2;
    layout.minimumLineSpacing = 2;
    return layout;
}

- (WaterfallLayout *)createWaterfallLayout {
    WaterfallLayout *layout = [[WaterfallLayout alloc] init];
    layout.delegate = self;
    layout.numberOfColumns = 2;
    layout.cellPadding = 4;
    return layout;
}

- (UICollectionViewFlowLayout *)createHorizontalLayout {
    UICollectionViewFlowLayout *layout = [[UICollectionViewFlowLayout alloc] init];
    layout.scrollDirection = UICollectionViewScrollDirectionHorizontal;
    layout.itemSize = CGSizeMake(200, 200);
    layout.minimumLineSpacing = 12;
    layout.sectionInset = UIEdgeInsetsMake(0, 16, 0, 16);
    return layout;
}

- (void)loadPhotos {
    // ข้อมูลตัวอย่าง
    NSMutableArray *section1 = [NSMutableArray array];
    NSMutableArray *section2 = [NSMutableArray array];
    
    for (NSInteger i = 0; i < 15; i++) {
        Photo *p = [[Photo alloc] init];
        p.identifier = [NSString stringWithFormat:@"photo_%ld", (long)i];
        p.title = [NSString stringWithFormat:@"Photo %ld", (long)(i + 1)];
        p.photographer = @"Photographer Name";
        p.aspectRatio = 0.5 + (i % 5) * 0.2; // varies 0.5 - 1.3
        [section1 addObject:p];
    }
    
    for (NSInteger i = 0; i < 10; i++) {
        Photo *p = [[Photo alloc] init];
        p.identifier = [NSString stringWithFormat:@"fav_%ld", (long)i];
        p.title = [NSString stringWithFormat:@"Favorite %ld", (long)(i + 1)];
        p.isFavorite = YES;
        p.aspectRatio = 0.5 + (i % 3) * 0.3;
        [section2 addObject:p];
    }
    
    self.photoSections = @[section1, section2];
    self.sectionTitles = @[@"ภาพทั้งหมด", @"รายการโปรด"];
    [self.collectionView reloadData];
}

// DataSource
- (NSInteger)numberOfSectionsInCollectionView:(UICollectionView *)collectionView {
    return self.photoSections.count;
}

- (NSInteger)collectionView:(UICollectionView *)collectionView 
     numberOfItemsInSection:(NSInteger)section {
    return self.photoSections[section].count;
}

- (UICollectionViewCell *)collectionView:(UICollectionView *)collectionView 
              cellForItemAtIndexPath:(NSIndexPath *)indexPath {
    PhotoCell *cell = [collectionView dequeueReusableCellWithReuseIdentifier:@"PhotoCell" 
                                                                forIndexPath:indexPath];
    Photo *photo = self.photoSections[indexPath.section][indexPath.item];
    [cell configureWithImage:photo.image caption:photo.title];
    return cell;
}

- (UICollectionReusableView *)collectionView:(UICollectionView *)collectionView 
           viewForSupplementaryElementOfKind:(NSString *)kind 
                                 atIndexPath:(NSIndexPath *)indexPath {
    if ([kind isEqualToString:UICollectionElementKindSectionHeader]) {
        SectionHeaderView *header = [collectionView 
            dequeueReusableSupplementaryViewOfKind:kind 
                               withReuseIdentifier:@"Header" 
                                      forIndexPath:indexPath];
        [header configureWithTitle:self.sectionTitles[indexPath.section] showSeeAll:YES];
        return header;
    }
    return nil;
}

// Delegate
- (void)collectionView:(UICollectionView *)collectionView 
didSelectItemAtIndexPath:(NSIndexPath *)indexPath {
    Photo *photo = self.photoSections[indexPath.section][indexPath.item];
    
    PhotoDetailViewController *detailVC = [[PhotoDetailViewController alloc] initWithPhoto:photo];
    [self.navigationController pushViewController:detailVC animated:YES];
}

// DelegateFlowLayout
- (CGSize)collectionView:(UICollectionView *)collectionView 
                  layout:(UICollectionViewLayout *)layout 
  sizeForItemAtIndexPath:(NSIndexPath *)indexPath {
    
    if (self.layoutMode == LayoutModeGrid) {
        CGFloat width = (collectionView.bounds.size.width - 4) / 3;
        return CGSizeMake(width, width);
    }
    return CGSizeMake(collectionView.bounds.size.width, 80);
}

- (CGSize)collectionView:(UICollectionView *)collectionView 
                  layout:(UICollectionViewLayout *)layout 
referenceSizeForHeaderInSection:(NSInteger)section {
    return CGSizeMake(0, 50);
}

// WaterfallLayoutDelegate
- (CGFloat)collectionView:(UICollectionView *)collectionView 
                   layout:(UICollectionViewLayout *)layout 
    heightForItemAtIndexPath:(NSIndexPath *)indexPath 
                withWidth:(CGFloat)width {
    Photo *photo = self.photoSections[indexPath.section][indexPath.item];
    return width * photo.aspectRatio;
}

// Layout toggle
- (void)toggleLayout {
    LayoutMode nextMode = (self.layoutMode + 1) % 3;
    self.layoutMode = nextMode;
    
    UICollectionViewLayout *newLayout;
    switch (nextMode) {
        case LayoutModeGrid:
            newLayout = [self createGridLayout];
            break;
        case LayoutModeWaterfall:
            newLayout = [self createWaterfallLayout];
            break;
        case LayoutModeHorizontal:
            newLayout = [self createHorizontalLayout];
            break;
    }
    
    [self.collectionView setCollectionViewLayout:newLayout animated:YES];
}

- (void)toggleSelection {
    BOOL isSelecting = self.collectionView.allowsMultipleSelection;
    self.collectionView.allowsMultipleSelection = !isSelecting;
    
    if (!isSelecting) {
        // เข้าสู่ selection mode
        [self.navigationItem.rightBarButtonItems[1] setTitle:@"เสร็จ"];
        UIBarButtonItem *deleteBtn = [[UIBarButtonItem alloc] 
                                     initWithTitle:@"ลบ" 
                                             style:UIBarButtonItemStylePlain 
                                            target:self 
                                            action:@selector(deleteSelected)];
        self.navigationItem.leftBarButtonItem = deleteBtn;
    } else {
        // ออกจาก selection mode
        [self.collectionView.indexPathsForSelectedItems 
            enumerateObjectsUsingBlock:^(NSIndexPath *path, NSUInteger idx, BOOL *stop) {
            [self.collectionView deselectItemAtIndexPath:path animated:NO];
        }];
        [self.navigationItem.rightBarButtonItems[1] setTitle:@"เลือก"];
        self.navigationItem.leftBarButtonItem = nil;
    }
}

- (void)deleteSelected {
    NSArray *selectedPaths = [self.collectionView.indexPathsForSelectedItems 
                             sortedArrayUsingComparator:^NSComparisonResult(NSIndexPath *a, NSIndexPath *b) {
        return [b compare:a];
    }];
    
    NSMutableArray *mutableSections = [self.photoSections mutableCopy];
    NSMutableDictionary *sectionChanges = [NSMutableDictionary dictionary];
    
    for (NSIndexPath *path in selectedPaths) {
        NSMutableArray *sectionArray = [mutableSections[path.section] mutableCopy];
        [sectionArray removeObjectAtIndex:path.item];
        mutableSections[path.section] = sectionArray;
        
        NSMutableArray *changes = sectionChanges[@(path.section)];
        if (!changes) {
            changes = [NSMutableArray array];
            sectionChanges[@(path.section)] = changes;
        }
        [changes addObject:path];
    }
    
    self.photoSections = [mutableSections copy];
    
    [self.collectionView performBatchUpdates:^{
        [self.collectionView deleteItemsAtIndexPaths:selectedPaths];
    } completion:nil];
    
    [self toggleSelection];
}

// Search
- (void)updateSearchResultsForSearchController:(UISearchController *)searchController {
    // TODO: implement search filtering
}

@end
```

---

## 12. UICollectionViewCompositionalLayout (iOS 13+)

### 12.1 App Store-Style Layout

```objc
// สร้าง compositional layout แบบ App Store
+ (UICollectionViewCompositionalLayout *)createAppStoreLayout {
    return [[UICollectionViewCompositionalLayout alloc] initWithSectionProvider:
        ^NSCollectionLayoutSection *(NSInteger section, id<NSCollectionLayoutEnvironment> environment) {
        
        if (section == 0) {
            // Featured section - full width
            NSCollectionLayoutSize *itemSize = [NSCollectionLayoutSize 
                sizeWithWidthDimension:[NSCollectionLayoutDimension fractionalWidthDimension:1.0]
                       heightDimension:[NSCollectionLayoutDimension fractionalHeightDimension:1.0]];
            
            NSCollectionLayoutItem *item = [NSCollectionLayoutItem itemWithLayoutSize:itemSize];
            item.contentInsets = NSDirectionalEdgeInsetsMake(8, 16, 8, 16);
            
            NSCollectionLayoutSize *groupSize = [NSCollectionLayoutSize 
                sizeWithWidthDimension:[NSCollectionLayoutDimension fractionalWidthDimension:0.9]
                       heightDimension:[NSCollectionLayoutDimension absoluteDimension:300]];
            
            NSCollectionLayoutGroup *group = [NSCollectionLayoutGroup 
                horizontalGroupWithLayoutSize:groupSize subitems:@[item]];
            
            NSCollectionLayoutSection *sectionLayout = [NSCollectionLayoutSection sectionWithGroup:group];
            sectionLayout.orthogonalScrollingBehavior = NSCollectionLayoutSectionOrthogonalScrollingBehaviorGroupPaging;
            
            return sectionLayout;
        }
        
        // Top charts section - 3 rows
        NSCollectionLayoutSize *itemSize = [NSCollectionLayoutSize 
            sizeWithWidthDimension:[NSCollectionLayoutDimension fractionalWidthDimension:1.0]
                   heightDimension:[NSCollectionLayoutDimension fractionalHeightDimension:1.0/3.0]];
        
        NSCollectionLayoutItem *item = [NSCollectionLayoutItem itemWithLayoutSize:itemSize];
        item.contentInsets = NSDirectionalEdgeInsetsMake(4, 0, 4, 0);
        
        NSCollectionLayoutSize *groupSize = [NSCollectionLayoutSize 
            sizeWithWidthDimension:[NSCollectionLayoutDimension fractionalWidthDimension:0.85]
                   heightDimension:[NSCollectionLayoutDimension absoluteDimension:200]];
        
        NSCollectionLayoutGroup *group = [NSCollectionLayoutGroup 
            verticalGroupWithLayoutSize:groupSize subitems:@[item]];
        
        NSCollectionLayoutSection *sectionLayout = [NSCollectionLayoutSection sectionWithGroup:group];
        sectionLayout.orthogonalScrollingBehavior = NSCollectionLayoutSectionOrthogonalScrollingBehaviorContinuous;
        sectionLayout.contentInsets = NSDirectionalEdgeInsetsMake(0, 16, 0, 16);
        sectionLayout.interGroupSpacing = 12;
        
        // Header
        NSCollectionLayoutSize *headerSize = [NSCollectionLayoutSize 
            sizeWithWidthDimension:[NSCollectionLayoutDimension fractionalWidthDimension:1.0]
                   heightDimension:[NSCollectionLayoutDimension absoluteDimension:44]];
        NSCollectionLayoutBoundarySupplementaryItem *header = 
            [NSCollectionLayoutBoundarySupplementaryItem 
                boundarySupplementaryItemWithLayoutSize:headerSize
                                            elementKind:UICollectionElementKindSectionHeader
                                             alignment:NSRectAlignmentTop];
        sectionLayout.boundarySupplementaryItems = @[header];
        
        return sectionLayout;
    }];
}
```

---

## 13. แบบฝึกหัด (Practice Exercises)

### Exercise 1: Instagram-Style Feed
สร้าง collection view แบบ Instagram:
- 3-column grid
- Tap เพื่อดูภาพเต็ม
- Pull-to-refresh
- Infinite scroll

### Exercise 2: Emoji Picker
สร้าง emoji picker แบบ iOS:
- Grouped sections ตาม category
- Sticky headers
- Search
- Recent emojis section

```objc
// Template
@interface EmojiPickerViewController ()
@property (nonatomic, strong) NSDictionary<NSString *, NSArray<NSString *> *> *emojiByCategory;
@property (nonatomic, strong) NSArray<NSString *> *categories;
@end
```

### Exercise 3: Drag & Drop Gallery
เพิ่ม drag-and-drop reordering ให้ photo gallery โดยใช้ long-press gesture หรือ UICollectionViewDragDelegate

### Exercise 4: Animated Filter
สร้าง filter bar ที่เมื่อ tap จะ filter items ด้วย animation:
- Tab bar ด้านบน (All, Photos, Videos, Favorites)
- Animate insert/delete เมื่อเปลี่ยน filter

### Exercise 5: Custom Mosaic Layout
สร้าง layout แบบ mosaic (บางรูปใหญ่ บางรูปเล็ก ตามรูปแบบที่กำหนด)

---

## สรุป

ในบทนี้เราได้เรียนรู้:

1. **UICollectionView Setup** - สร้างและ configure
2. **FlowLayout** - layout พื้นฐานที่ใช้บ่อย
3. **DataSource & Delegate** - การกำหนดข้อมูลและ interaction
4. **Custom Cells** - สร้าง cells ที่ซับซ้อน
5. **Section Headers/Footers** - Supplementary views
6. **Custom Layouts** - WaterfallLayout, CarouselLayout
7. **Animations** - insert/delete/move animations
8. **Multi-selection** - เลือกหลาย items
9. **CompositionalLayout** - modern API สำหรับ complex layouts

UICollectionView เป็นเครื่องมือที่ทรงพลังมาก ยิ่งฝึกสร้าง custom layouts ยิ่งสามารถทำ UI ที่สวยงามและซับซ้อนได้
