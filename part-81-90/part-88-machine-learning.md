# Part 88: Core ML และ Vision Framework - Machine Learning บน Apple

## บทนำ

Apple มี framework สำหรับ Machine Learning หลายตัวที่ทำงานบนอุปกรณ์ (on-device) โดยไม่ต้องส่งข้อมูลไปยัง server ซึ่งมีข้อดีด้านความเป็นส่วนตัว ความเร็ว และการทำงาน offline

- **Core ML**: Framework หลักสำหรับ inference ด้วย ML models
- **Vision**: Framework สำหรับ computer vision tasks
- **Natural Language**: Framework สำหรับ text processing
- **Create ML**: Tool สำหรับ train models บน Mac

---

## 88.1 Core ML Framework Overview

### สถาปัตยกรรมของ Core ML

```
ML Model (.mlmodel / .mlpackage)
         ↓
    Core ML API
         ↓
   Hardware Acceleration
    ├── Neural Engine (ANE)
    ├── GPU
    └── CPU
         ↓
    Predictions / Results
```

### ประเภทของ Models

```
Model Type          | ตัวอย่างการใช้งาน
--------------------|----------------------------
Image Classification | จำแนกประเภทรูปภาพ
Object Detection    | ค้นหาและระบุตำแหน่ง objects
Image Segmentation  | แบ่งส่วนของรูปภาพ
Text Classification | จำแนกประเภทข้อความ
Sound Analysis      | วิเคราะห์เสียง
Activity Recognition| จำแนกกิจกรรมจาก sensor
Recommendation      | แนะนำ content
Regression          | ทำนายค่าต่อเนื่อง
```

### Project Setup

```objc
// เพิ่ม .mlmodel ไฟล์ลงใน project
// Xcode จะ generate class โดยอัตโนมัติ

// ตัวอย่าง: เพิ่ม MobileNetV2.mlmodel
// Xcode จะสร้าง class MobileNetV2 ให้อัตโนมัติ

// Import Core ML
#import <CoreML/CoreML.h>
#import <Vision/Vision.h>
```

---

## 88.2 MLModel - การโหลดและใช้ Models

### การโหลด Model

```objc
@interface CoreMLManager : NSObject

@property (strong, nonatomic) MLModel *model;

@end

@implementation CoreMLManager

// วิธีที่ 1: โหลดจาก bundle โดยตรง (Xcode-generated class)
- (void)loadGeneratedModel {
    NSError *error;
    
    // Xcode generate class ให้อัตโนมัติ
    MobileNetV2 *model = [[MobileNetV2 alloc] initWithConfiguration:nil error:&error];
    
    if (error) {
        NSLog(@"Model load error: %@", error);
        return;
    }
    
    self.model = model.model;
}

// วิธีที่ 2: โหลดแบบ generic จาก URL
- (void)loadModelFromURL:(NSURL *)modelURL {
    NSError *error;
    
    MLModelConfiguration *config = [[MLModelConfiguration alloc] init];
    config.computeUnits = MLComputeUnitsAll;  // ใช้ทุก hardware ที่ดีที่สุด
    
    MLModel *model = [MLModel modelWithContentsOfURL:modelURL 
                                        configuration:config 
                                               error:&error];
    if (error) {
        NSLog(@"Model load error: %@", error);
        return;
    }
    
    self.model = model;
    NSLog(@"โหลด model สำเร็จ: %@", model.modelDescription.metadata);
}

// วิธีที่ 3: โหลด async (iOS 16+)
- (void)loadModelAsync {
    NSURL *modelURL = [[NSBundle mainBundle] URLForResource:@"MyModel" 
                                             withExtension:@"mlmodelc"];
    
    MLModelConfiguration *config = [[MLModelConfiguration alloc] init];
    config.computeUnits = MLComputeUnitsAll;
    
    [MLModel loadContentsOfURL:modelURL 
                 configuration:config 
             completionHandler:^(MLModel *model, NSError *error) {
        if (error) {
            NSLog(@"Async load error: %@", error);
            return;
        }
        
        dispatch_async(dispatch_get_main_queue(), ^{
            self.model = model;
            NSLog(@"Model โหลดสำเร็จ async");
        });
    }];
}

// MLComputeUnits options
/*
MLComputeUnitsCPUOnly     - CPU เท่านั้น (ช้าแต่ predictable)
MLComputeUnitsCPUAndGPU   - CPU + GPU
MLComputeUnitsAll         - CPU + GPU + Neural Engine (เร็วสุด)
MLComputeUnitsCPUAndNeuralEngine - CPU + ANE
*/

@end
```

### การ Inspect Model

```objc
- (void)inspectModel:(MLModel *)model {
    MLModelDescription *desc = model.modelDescription;
    
    // Input features
    NSLog(@"=== Model Inputs ===");
    for (NSString *name in desc.inputDescriptionsByName) {
        MLFeatureDescription *feature = desc.inputDescriptionsByName[name];
        NSLog(@"Input: %@ (type: %ld)", name, (long)feature.type);
        
        if (feature.type == MLFeatureTypeImage) {
            MLImageConstraint *imageConstraint = feature.imageConstraint;
            NSLog(@"  Image size: %lu x %lu", 
                  imageConstraint.pixelsWide, 
                  imageConstraint.pixelsHigh);
            NSLog(@"  Pixel format: %u", imageConstraint.pixelFormatType);
        }
    }
    
    // Output features
    NSLog(@"=== Model Outputs ===");
    for (NSString *name in desc.outputDescriptionsByName) {
        MLFeatureDescription *feature = desc.outputDescriptionsByName[name];
        NSLog(@"Output: %@ (type: %ld)", name, (long)feature.type);
        
        if (feature.type == MLFeatureTypeMultiArray) {
            MLMultiArrayConstraint *arrayConstraint = feature.multiArrayConstraint;
            NSLog(@"  Shape: %@", arrayConstraint.shape);
            NSLog(@"  Data type: %ld", (long)arrayConstraint.dataType);
        }
    }
    
    // Metadata
    NSLog(@"=== Metadata ===");
    NSDictionary *metadata = desc.metadata;
    NSLog(@"Author: %@", metadata[MLModelMetadataKeyAuthor]);
    NSLog(@"Description: %@", metadata[MLModelMetadataKeyDescription]);
    NSLog(@"Version: %@", metadata[MLModelMetadataKeyVersionString]);
}
```

---

## 88.3 Image Classification

### การจำแนกประเภทรูปภาพ

```objc
#import <CoreML/CoreML.h>
#import <Vision/Vision.h>

@interface ImageClassifier : NSObject

- (void)classifyImage:(UIImage *)image 
           completion:(void(^)(NSArray<VNClassificationObservation *> *results, NSError *error))completion;

@end

@implementation ImageClassifier {
    VNCoreMLModel *_model;
    VNCoreMLRequest *_request;
}

- (instancetype)init {
    self = [super init];
    if (self) {
        [self loadModel];
    }
    return self;
}

- (void)loadModel {
    NSError *error;
    
    // โหลด MobileNetV2 model
    NSURL *modelURL = [[NSBundle mainBundle] URLForResource:@"MobileNetV2" 
                                             withExtension:@"mlmodelc"];
    
    MLModel *mlModel = [MLModel modelWithContentsOfURL:modelURL 
                                         configuration:nil 
                                                 error:&error];
    if (error) {
        NSLog(@"Load error: %@", error);
        return;
    }
    
    // Wrap ด้วย VNCoreMLModel สำหรับ Vision
    _model = [VNCoreMLModel modelForMLModel:mlModel error:&error];
    if (error) {
        NSLog(@"VNCoreMLModel error: %@", error);
        return;
    }
    
    // สร้าง request
    _request = [[VNCoreMLRequest alloc] initWithModel:_model 
                                    completionHandler:^(VNRequest *request, NSError *error) {
        // จัดการผลลัพธ์ใน completion handler ของ classifyImage:
    }];
    
    // ตั้งค่า crop and scale
    _request.imageCropAndScaleOption = VNImageCropAndScaleOptionCenterCrop;
}

- (void)classifyImage:(UIImage *)image 
           completion:(void(^)(NSArray<VNClassificationObservation *> *results, NSError *error))completion {
    
    // แปลง UIImage เป็น CGImage
    CGImageRef cgImage = image.CGImage;
    if (!cgImage) {
        completion(nil, [NSError errorWithDomain:@"Classifier" code:-1 userInfo:nil]);
        return;
    }
    
    // สร้าง image request handler
    NSDictionary *options = @{
        VNImageOptionCIContext: [[CIContext alloc] init]
    };
    
    VNImageRequestHandler *handler = [[VNImageRequestHandler alloc] 
        initWithCGImage:cgImage options:options];
    
    // สร้าง request ใหม่พร้อม completion handler
    __block VNCoreMLRequest *classifyRequest = [[VNCoreMLRequest alloc] 
        initWithModel:_model 
    completionHandler:^(VNRequest *request, NSError *error) {
        
        if (error) {
            completion(nil, error);
            return;
        }
        
        NSArray<VNClassificationObservation *> *observations = request.results;
        
        // กรองผลลัพธ์ที่มี confidence สูง
        NSArray *filteredResults = [observations filteredArrayUsingPredicate:
            [NSPredicate predicateWithFormat:@"confidence > 0.1"]];
        
        // เรียงตาม confidence
        NSArray *sortedResults = [filteredResults sortedArrayUsingDescriptors:@[
            [NSSortDescriptor sortDescriptorWithKey:@"confidence" ascending:NO]
        ]];
        
        dispatch_async(dispatch_get_main_queue(), ^{
            completion(sortedResults, nil);
        });
    }];
    
    classifyRequest.imageCropAndScaleOption = VNImageCropAndScaleOptionCenterCrop;
    
    // Execute request
    dispatch_async(dispatch_get_global_queue(DISPATCH_QUEUE_PRIORITY_DEFAULT, 0), ^{
        NSError *error;
        [handler performRequests:@[classifyRequest] error:&error];
        if (error) {
            dispatch_async(dispatch_get_main_queue(), ^{
                completion(nil, error);
            });
        }
    });
}

@end

// การใช้งาน
@implementation ViewController

- (void)classifySelectedImage:(UIImage *)image {
    ImageClassifier *classifier = [[ImageClassifier alloc] init];
    
    [classifier classifyImage:image completion:^(NSArray *results, NSError *error) {
        if (error) {
            NSLog(@"Classification error: %@", error);
            return;
        }
        
        NSLog(@"=== ผลการจำแนกภาพ ===");
        for (VNClassificationObservation *obs in results) {
            NSLog(@"%@ (%.1f%%)", obs.identifier, obs.confidence * 100);
        }
        
        // แสดงผลลัพธ์อันดับ 1
        if (results.count > 0) {
            VNClassificationObservation *topResult = results.firstObject;
            self.resultLabel.text = [NSString stringWithFormat:@"%@ (%.0f%%)",
                topResult.identifier, topResult.confidence * 100];
        }
    }];
}

@end
```

---

## 88.4 Object Detection

### การตรวจจับ Objects ในรูปภาพ

```objc
@implementation ObjectDetector {
    VNCoreMLModel *_detectionModel;
}

- (void)detectObjectsInImage:(UIImage *)image 
                  completion:(void(^)(NSArray<VNRecognizedObjectObservation *> *objects))completion {
    
    // โหลด YOLOv3 หรือ model อื่นๆ
    NSURL *modelURL = [[NSBundle mainBundle] URLForResource:@"YOLOv3" 
                                             withExtension:@"mlmodelc"];
    NSError *error;
    MLModel *mlModel = [MLModel modelWithContentsOfURL:modelURL 
                                         configuration:nil 
                                                 error:&error];
    
    _detectionModel = [VNCoreMLModel modelForMLModel:mlModel error:&error];
    
    // สร้าง request
    VNCoreMLRequest *request = [[VNCoreMLRequest alloc] 
        initWithModel:_detectionModel 
    completionHandler:^(VNRequest *request, NSError *error) {
        
        NSArray<VNRecognizedObjectObservation *> *observations = request.results;
        
        // กรอง objects ที่มี confidence สูงกว่า threshold
        NSArray *detectedObjects = [observations filteredArrayUsingPredicate:
            [NSPredicate predicateWithFormat:@"confidence > 0.5"]];
        
        dispatch_async(dispatch_get_main_queue(), ^{
            completion(detectedObjects);
        });
    }];
    
    request.imageCropAndScaleOption = VNImageCropAndScaleOptionScaleFill;
    
    // Execute
    VNImageRequestHandler *handler = [[VNImageRequestHandler alloc] 
        initWithCGImage:image.CGImage options:@{}];
    
    [handler performRequests:@[request] error:nil];
}

// วาด bounding boxes บนภาพ
- (void)drawDetections:(NSArray<VNRecognizedObjectObservation *> *)detections
               onImage:(UIImageView *)imageView {
    
    // ลบ layers เก่า
    for (CALayer *layer in [imageView.layer.sublayers copy]) {
        [layer removeFromSuperlayer];
    }
    
    CGSize imageSize = imageView.image.size;
    CGRect imageRect = AVMakeRectWithAspectRatioInsideRect(imageSize, imageView.bounds);
    
    for (VNRecognizedObjectObservation *detection in detections) {
        // แปลง normalized coordinates เป็น view coordinates
        // VNObservation ใช้ coordinate system ที่กลับ Y axis
        CGRect normalizedRect = detection.boundingBox;
        CGRect viewRect = CGRectMake(
            normalizedRect.origin.x * imageRect.size.width + imageRect.origin.x,
            (1 - normalizedRect.origin.y - normalizedRect.size.height) * imageRect.size.height + imageRect.origin.y,
            normalizedRect.size.width * imageRect.size.width,
            normalizedRect.size.height * imageRect.size.height
        );
        
        // สร้าง bounding box layer
        CALayer *boxLayer = [CALayer layer];
        boxLayer.frame = viewRect;
        boxLayer.borderColor = [UIColor systemGreenColor].CGColor;
        boxLayer.borderWidth = 2.0;
        boxLayer.cornerRadius = 4.0;
        
        // Label
        CATextLayer *labelLayer = [CATextLayer layer];
        NSString *labelName = detection.labels.firstObject.identifier ?: @"unknown";
        float confidence = detection.labels.firstObject.confidence ?: 0;
        labelLayer.string = [NSString stringWithFormat:@"%@ (%.0f%%)", 
                            labelName, confidence * 100];
        labelLayer.fontSize = 14;
        labelLayer.foregroundColor = [UIColor whiteColor].CGColor;
        labelLayer.backgroundColor = [UIColor colorWithRed:0 green:0.5 blue:0 alpha:0.7].CGColor;
        labelLayer.frame = CGRectMake(0, 0, viewRect.size.width, 20);
        
        [boxLayer addSublayer:labelLayer];
        [imageView.layer addSublayer:boxLayer];
        
        NSLog(@"ตรวจพบ: %@ (%.1f%%)", labelName, confidence * 100);
    }
}

@end
```

---

## 88.5 Natural Language Processing

### NaturalLanguage Framework

```objc
#import <NaturalLanguage/NaturalLanguage.h>

@interface NLPAnalyzer : NSObject

@end

@implementation NLPAnalyzer

// ตรวจสอบภาษา
- (void)detectLanguage:(NSString *)text {
    NLLanguageRecognizer *recognizer = [[NLLanguageRecognizer alloc] init];
    [recognizer processString:text];
    
    // ภาษาที่น่าจะเป็น
    NLLanguage dominantLanguage = recognizer.dominantLanguage;
    NSLog(@"ภาษาหลัก: %@", dominantLanguage);
    
    // ความน่าจะเป็นของแต่ละภาษา
    NSDictionary *hypotheses = [recognizer languageHypothesesWithMaximum:5];
    for (NLLanguage lang in hypotheses) {
        NSLog(@"%@: %.1f%%", lang, [hypotheses[lang] floatValue] * 100);
    }
}

// Tokenization
- (void)tokenizeText:(NSString *)text {
    NLTokenizer *tokenizer = [[NLTokenizer alloc] initWithUnit:NLTokenUnitWord];
    tokenizer.string = text;
    
    [tokenizer enumerateTokensInRange:NSMakeRange(0, text.length) 
                           usingBlock:^(NSRange tokenRange, 
                                       NLTokenizerAttributes flags, 
                                       BOOL *stop) {
        NSString *token = [text substringWithRange:tokenRange];
        NSLog(@"Token: '%@'", token);
    }];
    
    // ประเภทของ tokenizer
    // NLTokenUnitWord    - แบ่งด้วยคำ
    // NLTokenUnitSentence - แบ่งด้วยประโยค
    // NLTokenUnitParagraph - แบ่งด้วยย่อหน้า
    // NLTokenUnitDocument - ทั้ง document
}

// Part of Speech Tagging
- (void)tagPartsOfSpeech:(NSString *)text {
    NLTagger *tagger = [[NLTagger alloc] initWithTagSchemes:@[NLTagSchemeNameType, 
                                                               NLTagSchemeLexicalClass]];
    tagger.string = text;
    
    [tagger enumerateTagsInRange:NSMakeRange(0, text.length) 
                            unit:NLTokenUnitWord 
                          scheme:NLTagSchemeLexicalClass 
                         options:NLTaggerOmitPunctuation | NLTaggerOmitWhitespace
                      usingBlock:^(NLTag tag, NSRange tokenRange, BOOL *stop) {
        
        NSString *word = [text substringWithRange:tokenRange];
        NSLog(@"%@: %@", word, tag);
        
        // Tags ที่เป็นไปได้
        // NLTagNoun, NLTagVerb, NLTagAdjective, NLTagAdverb
        // NLTagPronoun, NLTagDeterminer, NLTagParticle
        // NLTagPreposition, NLTagNumber, NLTagConjunction
        // NLTagInterjection, NLTagClassifier, NLTagIdiom
        // NLTagOtherWord
    }];
}

// Named Entity Recognition
- (void)recognizeNamedEntities:(NSString *)text {
    NLTagger *tagger = [[NLTagger alloc] initWithTagSchemes:@[NLTagSchemeNameType]];
    tagger.string = text;
    
    NLTaggerOptions options = NLTaggerOmitPunctuation | NLTaggerOmitWhitespace | NLTaggerJoinNames;
    
    [tagger enumerateTagsInRange:NSMakeRange(0, text.length) 
                            unit:NLTokenUnitWord 
                          scheme:NLTagSchemeNameType 
                         options:options 
                      usingBlock:^(NLTag tag, NSRange tokenRange, BOOL *stop) {
        
        if (tag) {  // มีเฉพาะ named entities
            NSString *entity = [text substringWithRange:tokenRange];
            NSLog(@"Entity: '%@' (%@)", entity, tag);
            
            // entity types
            // NLTagPersonalName - ชื่อบุคคล
            // NLTagPlaceName - ชื่อสถานที่
            // NLTagOrganizationName - ชื่อองค์กร
        }
    }];
}

// Word Embedding (semantic similarity)
- (void)findSimilarWords:(NSString *)word {
    if (@available(iOS 14.0, *)) {
        NLEmbedding *embedding = [NLEmbedding wordEmbeddingForLanguage:NLLanguageEnglish];
        
        if (!embedding) {
            NSLog(@"Word embedding ไม่รองรับ");
            return;
        }
        
        // หาคำที่ใกล้เคียง
        [embedding enumerateNeighborsForString:word
                                   maximumCount:10
                                distanceFunction:NLDistanceCosine
                                    usingBlock:^(NSString *neighbor, double distance, BOOL *stop) {
            NSLog(@"'%@' (distance: %.3f)", neighbor, distance);
        }];
        
        // คำนวณ distance ระหว่างสองคำ
        double distance = [embedding distanceBetweenString:word 
                                                  andString:@"similar_word"
                                         distanceFunction:NLDistanceCosine];
        NSLog(@"Distance: %.3f", distance);
    }
}

// Text Classification ด้วย Create ML model
- (void)classifyText:(NSString *)text {
    NSError *error;
    NSURL *modelURL = [[NSBundle mainBundle] URLForResource:@"SentimentClassifier" 
                                             withExtension:@"mlmodelc"];
    
    NLModel *nlModel = [NLModel modelWithContentsOfURL:modelURL error:&error];
    if (error) {
        NSLog(@"Model error: %@", error);
        return;
    }
    
    // Classify
    NSString *label = [nlModel predictedLabelForString:text];
    NSLog(@"Classification: %@", label);
    
    // ความน่าจะเป็นของแต่ละ label
    NSDictionary *predictions = [nlModel predictedLabelHypothesesForString:text 
                                                             maximumCount:5];
    for (NSString *predictedLabel in predictions) {
        NSLog(@"%@: %.1f%%", predictedLabel, [predictions[predictedLabel] floatValue] * 100);
    }
}

@end
```

---

## 88.6 Create ML Overview

### การ Train Model บน Mac

```swift
// Create ML ใช้ Swift (ไม่ใช่ Objective-C)
// แต่ผลลัพธ์ .mlmodel ใช้งานกับ Objective-C ได้

// ตัวอย่างการ train Image Classifier
import CreateML

let trainingData = MLImageClassifier.DataSource.labeledDirectories(at: URL(fileURLWithPath: "/path/to/training/data"))
let validationData = MLImageClassifier.DataSource.labeledDirectories(at: URL(fileURLWithPath: "/path/to/validation/data"))

// Configuration
var parameters = MLImageClassifier.ModelParameters()
parameters.maxIterations = 10
parameters.augmentationOptions = [.flip, .rotate, .crop]
parameters.validationData = validationData

// Train
let classifier = try MLImageClassifier(trainingData: trainingData, parameters: parameters)

// Evaluate
let evaluation = classifier.evaluation(on: validationData)
print("Accuracy: \(evaluation.classificationError)")

// Save
try classifier.write(to: URL(fileURLWithPath: "/path/to/MyClassifier.mlmodel"))
```

### การใช้ Model ที่ Train แล้วใน Objective-C

```objc
// หลังจาก train ด้วย Create ML
// นำ .mlmodel มาใส่ใน Xcode project
// แล้วใช้งานปกติด้วย Vision หรือ Core ML

@implementation MyImageClassifierUse

- (void)useTrainedModel {
    NSError *error;
    
    // โหลด model ที่ train เอง
    NSURL *modelURL = [[NSBundle mainBundle] URLForResource:@"MyCustomClassifier" 
                                             withExtension:@"mlmodelc"];
    
    VNCoreMLModel *vnModel = [VNCoreMLModel modelForMLModel:
        [MLModel modelWithContentsOfURL:modelURL configuration:nil error:&error]
        error:&error];
    
    VNCoreMLRequest *request = [[VNCoreMLRequest alloc] 
        initWithModel:vnModel 
    completionHandler:^(VNRequest *request, NSError *error) {
        
        NSArray<VNClassificationObservation *> *results = request.results;
        VNClassificationObservation *top = results.firstObject;
        
        NSLog(@"Prediction: %@ (%.1f%%)", top.identifier, top.confidence * 100);
    }];
    
    // Process image
    VNImageRequestHandler *handler = [[VNImageRequestHandler alloc] 
        initWithCGImage:self.testImage.CGImage options:@{}];
    [handler performRequests:@[request] error:nil];
}

@end
```

---

## 88.7 Vision Framework

### VNRequest และ VNImageRequestHandler

```objc
@implementation VisionProcessor

- (void)processImage:(UIImage *)image {
    CGImageRef cgImage = image.CGImage;
    
    // สร้าง request handler
    VNImageRequestHandler *handler = [[VNImageRequestHandler alloc] 
        initWithCGImage:cgImage 
            orientation:kCGImagePropertyOrientationUp 
                options:@{}];
    
    // สร้าง requests ที่ต้องการ
    NSMutableArray<VNRequest *> *requests = [NSMutableArray array];
    
    // เพิ่ม face detection request
    VNDetectFaceRectanglesRequest *faceRequest = [self buildFaceDetectionRequest];
    [requests addObject:faceRequest];
    
    // เพิ่ม text detection request
    VNDetectTextRectanglesRequest *textRequest = [self buildTextDetectionRequest];
    [requests addObject:textRequest];
    
    // Execute หลาย requests พร้อมกัน
    dispatch_async(dispatch_get_global_queue(DISPATCH_QUEUE_PRIORITY_HIGH, 0), ^{
        NSError *error;
        [handler performRequests:requests error:&error];
        
        if (error) {
            NSLog(@"Vision error: %@", error);
        }
    });
}

// สร้าง request ด้วย handler สำหรับ Face Detection
- (VNDetectFaceRectanglesRequest *)buildFaceDetectionRequest {
    return [[VNDetectFaceRectanglesRequest alloc] 
        initWithCompletionHandler:^(VNRequest *request, NSError *error) {
        
        NSArray<VNFaceObservation *> *faces = request.results;
        NSLog(@"พบใบหน้า: %lu หน้า", faces.count);
        
        for (VNFaceObservation *face in faces) {
            CGRect boundingBox = face.boundingBox;
            NSLog(@"  ใบหน้าที่พิกัด: (%.2f, %.2f) ขนาด: %.2f x %.2f",
                  boundingBox.origin.x, boundingBox.origin.y,
                  boundingBox.size.width, boundingBox.size.height);
            
            NSLog(@"  Confidence: %.2f", face.confidence);
            
            // Facial landmarks
            VNFaceLandmarks2D *landmarks = face.landmarks;
            if (landmarks) {
                NSLog(@"  มี facial landmarks");
            }
        }
    }];
}

@end
```

---

## 88.8 Text Recognition (OCR)

### VNRecognizeTextRequest

```objc
@implementation TextRecognizer

- (void)recognizeTextInImage:(UIImage *)image 
                  completion:(void(^)(NSString *recognizedText, NSError *error))completion {
    
    VNRecognizeTextRequest *request = [[VNRecognizeTextRequest alloc] 
        initWithCompletionHandler:^(VNRequest *request, NSError *error) {
        
        if (error) {
            dispatch_async(dispatch_get_main_queue(), ^{
                completion(nil, error);
            });
            return;
        }
        
        NSMutableString *fullText = [NSMutableString string];
        
        NSArray<VNRecognizedTextObservation *> *observations = request.results;
        
        for (VNRecognizedTextObservation *observation in observations) {
            // รับข้อความที่น่าจะเป็นมากที่สุด
            VNRecognizedText *topCandidate = [observation topCandidates:1].firstObject;
            
            if (topCandidate) {
                [fullText appendString:topCandidate.string];
                [fullText appendString:@"\n"];
                
                NSLog(@"Text: '%@' (confidence: %.2f)", 
                      topCandidate.string, topCandidate.confidence);
                
                // ตำแหน่งของข้อความใน image
                NSError *boundingBoxError;
                VNRectangleObservation *boundingBox = [topCandidate boundingBoxForRange:
                    NSMakeRange(0, topCandidate.string.length) 
                    error:&boundingBoxError];
                
                if (boundingBox) {
                    NSLog(@"  ตำแหน่ง: %@", NSStringFromCGRect(boundingBox.boundingBox));
                }
            }
        }
        
        dispatch_async(dispatch_get_main_queue(), ^{
            completion([fullText copy], nil);
        });
    }];
    
    // ตั้งค่า recognition level
    request.recognitionLevel = VNRequestTextRecognitionLevelAccurate;
    // VNRequestTextRecognitionLevelFast - เร็วกว่า แต่แม่นน้อยกว่า
    // VNRequestTextRecognitionLevelAccurate - แม่นกว่า แต่ช้ากว่า
    
    // ตั้งค่าภาษา
    request.recognitionLanguages = @[@"en-US", @"th-TH"];
    
    // ใช้ language correction
    request.usesLanguageCorrection = YES;
    
    // Minimum text height (0.0 - 1.0 relative to image height)
    request.minimumTextHeight = 0.02;
    
    // Execute
    dispatch_async(dispatch_get_global_queue(DISPATCH_QUEUE_PRIORITY_DEFAULT, 0), ^{
        NSError *error;
        VNImageRequestHandler *handler = [[VNImageRequestHandler alloc] 
            initWithCGImage:image.CGImage 
                orientation:kCGImagePropertyOrientationUp 
                    options:@{}];
        
        [handler performRequests:@[request] error:&error];
        if (error) {
            dispatch_async(dispatch_get_main_queue(), ^{
                completion(nil, error);
            });
        }
    });
}

// Document Scanning App
- (void)scanDocument:(UIImage *)image {
    [self recognizeTextInImage:image completion:^(NSString *text, NSError *error) {
        if (text) {
            NSLog(@"=== ข้อความที่พบในเอกสาร ===\n%@", text);
            
            // บันทึกเป็นไฟล์
            NSString *documentsPath = [NSSearchPathForDirectoriesInDomains(
                NSDocumentDirectory, NSUserDomainMask, YES) firstObject];
            NSString *filePath = [documentsPath stringByAppendingPathComponent:@"scanned.txt"];
            [text writeToFile:filePath atomically:YES encoding:NSUTF8StringEncoding error:nil];
        }
    }];
}

@end
```

---

## 88.9 Face Detection

### การตรวจจับและวิเคราะห์ใบหน้า

```objc
@implementation FaceAnalyzer

// ตรวจจับใบหน้าพร้อม landmarks
- (void)detectFacesWithLandmarks:(UIImage *)image {
    VNDetectFaceLandmarksRequest *request = [[VNDetectFaceLandmarksRequest alloc] 
        initWithCompletionHandler:^(VNRequest *request, NSError *error) {
        
        NSArray<VNFaceObservation *> *faces = request.results;
        
        for (VNFaceObservation *face in faces) {
            [self processFace:face inImage:image];
        }
    }];
    
    // ตั้งค่า constellation
    request.constellation = VNRequestFaceLandmarksConstellation76Point;
    // VNRequestFaceLandmarksConstellation65Point - 65 จุด
    // VNRequestFaceLandmarksConstellation76Point - 76 จุด (ละเอียดกว่า)
    
    [self executeRequest:request withImage:image];
}

- (void)processFace:(VNFaceObservation *)face inImage:(UIImage *)image {
    NSLog(@"=== ใบหน้า ===");
    NSLog(@"Bounding box: %@", NSStringFromCGRect(face.boundingBox));
    NSLog(@"Confidence: %.2f", face.confidence);
    
    // Roll, Pitch, Yaw angles
    if (face.roll) NSLog(@"Roll: %.1f°", [face.roll floatValue] * 180 / M_PI);
    if (face.pitch) NSLog(@"Pitch: %.1f°", [face.pitch floatValue] * 180 / M_PI);
    if (face.yaw) NSLog(@"Yaw: %.1f°", [face.yaw floatValue] * 180 / M_PI);
    
    // Facial landmarks
    VNFaceLandmarks2D *landmarks = face.landmarks;
    if (!landmarks) return;
    
    // จุดสำคัญต่างๆ
    VNFaceLandmarkRegion2D *leftEye = landmarks.leftEye;
    VNFaceLandmarkRegion2D *rightEye = landmarks.rightEye;
    VNFaceLandmarkRegion2D *nose = landmarks.nose;
    VNFaceLandmarkRegion2D *outerLips = landmarks.outerLips;
    VNFaceLandmarkRegion2D *innerLips = landmarks.innerLips;
    VNFaceLandmarkRegion2D *leftEyebrow = landmarks.leftEyebrow;
    VNFaceLandmarkRegion2D *rightEyebrow = landmarks.rightEyebrow;
    VNFaceLandmarkRegion2D *faceContour = landmarks.faceContour;
    
    // แสดง landmarks บนภาพ
    UIImage *annotated = [self drawLandmarks:landmarks onImage:image face:face];
    
    // วิเคราะห์ว่าตาเปิดหรือปิด
    BOOL leftEyeOpen = [self isEyeOpen:leftEye];
    BOOL rightEyeOpen = [self isEyeOpen:rightEye];
    NSLog(@"ตาซ้าย: %@, ตาขวา: %@", 
          leftEyeOpen ? @"เปิด" : @"ปิด",
          rightEyeOpen ? @"เปิด" : @"ปิด");
}

- (BOOL)isEyeOpen:(VNFaceLandmarkRegion2D *)eye {
    // คำนวณ eye aspect ratio (EAR)
    // EAR = (vertical distances) / (horizontal distance)
    if (eye.pointCount < 6) return YES;
    
    const CGPoint *points = eye.normalizedPoints;
    
    // Vertical
    CGFloat v1 = hypot(points[1].x - points[5].x, points[1].y - points[5].y);
    CGFloat v2 = hypot(points[2].x - points[4].x, points[2].y - points[4].y);
    
    // Horizontal
    CGFloat h = hypot(points[0].x - points[3].x, points[0].y - points[3].y);
    
    CGFloat ear = (v1 + v2) / (2.0 * h);
    
    // EAR < 0.2 = ตาปิด
    return ear > 0.2;
}

// ตรวจจับ face quality
- (void)detectFaceQuality:(UIImage *)image {
    if (@available(iOS 15.0, *)) {
        VNDetectFaceCaptureQualityRequest *request = 
            [[VNDetectFaceCaptureQualityRequest alloc] 
                initWithCompletionHandler:^(VNRequest *request, NSError *error) {
            
            for (VNFaceObservation *face in request.results) {
                float quality = face.faceCaptureQuality.floatValue;
                NSLog(@"Face quality: %.2f (0=poor, 1=excellent)", quality);
            }
        }];
        
        [self executeRequest:request withImage:image];
    }
}

@end
```

---

## 88.10 Barcode Detection

### การสแกน Barcode และ QR Code

```objc
@implementation BarcodeScanner

- (void)scanBarcodesInImage:(UIImage *)image 
                 completion:(void(^)(NSArray<VNBarcodeObservation *> *barcodes))completion {
    
    VNDetectBarcodesRequest *request = [[VNDetectBarcodesRequest alloc] 
        initWithCompletionHandler:^(VNRequest *request, NSError *error) {
        
        NSArray<VNBarcodeObservation *> *barcodes = request.results;
        
        dispatch_async(dispatch_get_main_queue(), ^{
            completion(barcodes);
        });
    }];
    
    // กำหนดประเภท barcode ที่ต้องการ
    request.symbologies = @[
        VNBarcodeSymbologyQR,           // QR Code
        VNBarcodeSymbologyCode128,      // Code 128
        VNBarcodeSymbologyCode39,       // Code 39
        VNBarcodeSymbologyEAN13,        // EAN-13 (สินค้า)
        VNBarcodeSymbologyEAN8,         // EAN-8
        VNBarcodeSymbologyPDF417,       // PDF417
        VNBarcodeSymbologyDataMatrix,   // Data Matrix
        VNBarcodeSymbologyAztec,        // Aztec
    ];
    
    [self executeRequest:request withImage:image];
}

- (void)processBarcode:(VNBarcodeObservation *)barcode {
    NSLog(@"=== Barcode พบ ===");
    NSLog(@"Symbology: %@", barcode.symbology);
    NSLog(@"Payload: %@", barcode.payloadStringValue);
    NSLog(@"Confidence: %.2f", barcode.confidence);
    
    // สำหรับ QR Code ที่มีข้อมูล URL
    NSString *payload = barcode.payloadStringValue;
    NSURL *url = [NSURL URLWithString:payload];
    if (url && url.scheme) {
        NSLog(@"URL: %@", url);
        [[UIApplication sharedApplication] openURL:url options:@{} completionHandler:nil];
    }
    
    // สำหรับ WiFi QR Code
    if ([payload hasPrefix:@"WIFI:"]) {
        [self parseWiFiQRCode:payload];
    }
    
    // สำหรับ Contact (vCard) QR Code
    if ([payload hasPrefix:@"BEGIN:VCARD"]) {
        NSLog(@"พบ vCard contact");
    }
}

- (void)parseWiFiQRCode:(NSString *)payload {
    // Format: WIFI:T:WPA;S:NetworkName;P:Password;;
    NSRegularExpression *ssidRegex = [NSRegularExpression 
        regularExpressionWithPattern:@"S:([^;]+)" options:0 error:nil];
    NSRegularExpression *passRegex = [NSRegularExpression 
        regularExpressionWithPattern:@"P:([^;]+)" options:0 error:nil];
    
    NSTextCheckingResult *ssidMatch = [ssidRegex firstMatchInString:payload 
                                                            options:0 
                                                              range:NSMakeRange(0, payload.length)];
    NSTextCheckingResult *passMatch = [passRegex firstMatchInString:payload 
                                                            options:0 
                                                              range:NSMakeRange(0, payload.length)];
    
    if (ssidMatch) {
        NSString *ssid = [payload substringWithRange:[ssidMatch rangeAtIndex:1]];
        NSString *pass = passMatch ? [payload substringWithRange:[passMatch rangeAtIndex:1]] : @"";
        
        NSLog(@"WiFi SSID: %@, Password: %@", ssid, pass);
    }
}

// Camera-based realtime barcode scanning
- (void)setupRealtimeBarcodeScanning {
    AVCaptureSession *session = [[AVCaptureSession alloc] init];
    
    // Camera input
    AVCaptureDevice *camera = [AVCaptureDevice defaultDeviceWithMediaType:AVMediaTypeVideo];
    AVCaptureDeviceInput *input = [AVCaptureDeviceInput deviceInputWithDevice:camera error:nil];
    [session addInput:input];
    
    // Video output
    AVCaptureVideoDataOutput *output = [[AVCaptureVideoDataOutput alloc] init];
    output.videoSettings = @{(NSString *)kCVPixelBufferPixelFormatTypeKey: @(kCVPixelFormatType_32BGRA)};
    [output setSampleBufferDelegate:self queue:dispatch_get_global_queue(0, 0)];
    [session addOutput:output];
    
    [session startRunning];
}

// AVCaptureVideoDataOutputSampleBufferDelegate
- (void)captureOutput:(AVCaptureOutput *)output 
    didOutputSampleBuffer:(CMSampleBufferRef)sampleBuffer 
           fromConnection:(AVCaptureConnection *)connection {
    
    CVPixelBufferRef pixelBuffer = CMSampleBufferGetImageBuffer(sampleBuffer);
    
    VNDetectBarcodesRequest *request = [[VNDetectBarcodesRequest alloc] 
        initWithCompletionHandler:^(VNRequest *request, NSError *error) {
        
        for (VNBarcodeObservation *barcode in request.results) {
            if (barcode.confidence > 0.9) {
                dispatch_async(dispatch_get_main_queue(), ^{
                    [self handleBarcodeFound:barcode];
                });
            }
        }
    }];
    
    VNImageRequestHandler *handler = [[VNImageRequestHandler alloc] 
        initWithCVPixelBuffer:pixelBuffer options:@{}];
    [handler performRequests:@[request] error:nil];
}

@end
```

---

## 88.11 Body Pose Detection (Vision)

### VNDetectHumanBodyPoseRequest

```objc
@implementation BodyPoseDetector

- (void)detectBodyPose:(UIImage *)image {
    if (@available(iOS 14.0, *)) {
        VNDetectHumanBodyPoseRequest *request = [[VNDetectHumanBodyPoseRequest alloc] 
            initWithCompletionHandler:^(VNRequest *request, NSError *error) {
            
            NSArray<VNHumanBodyPoseObservation *> *observations = request.results;
            
            for (VNHumanBodyPoseObservation *observation in observations) {
                [self processBodyPose:observation];
            }
        }];
        
        [self executeRequest:request withImage:image];
    }
}

- (void)processBodyPose:(VNHumanBodyPoseObservation *)observation API_AVAILABLE(ios(14.0)) {
    NSError *error;
    
    // รับ joint points ทั้งหมด
    NSDictionary<VNHumanBodyPoseObservationJointName, VNRecognizedPoint *> *joints = 
        [observation recognizedPointsForGroupKey:VNHumanBodyPoseObservationJointsGroupNameAll 
                                           error:&error];
    
    // Joint names ที่ใช้บ่อย
    VNRecognizedPoint *nose = joints[VNHumanBodyPoseObservationJointNameNose];
    VNRecognizedPoint *leftWrist = joints[VNHumanBodyPoseObservationJointNameLeftWrist];
    VNRecognizedPoint *rightWrist = joints[VNHumanBodyPoseObservationJointNameRightWrist];
    VNRecognizedPoint *leftAnkle = joints[VNHumanBodyPoseObservationJointNameLeftAnkle];
    VNRecognizedPoint *rightAnkle = joints[VNHumanBodyPoseObservationJointNameRightAnkle];
    VNRecognizedPoint *leftShoulder = joints[VNHumanBodyPoseObservationJointNameLeftShoulder];
    VNRecognizedPoint *rightShoulder = joints[VNHumanBodyPoseObservationJointNameRightShoulder];
    VNRecognizedPoint *leftHip = joints[VNHumanBodyPoseObservationJointNameLeftHip];
    VNRecognizedPoint *rightHip = joints[VNHumanBodyPoseObservationJointNameRightHip];
    
    // ตรวจสอบว่า joint มีความน่าเชื่อถือสูงพอ
    float confidenceThreshold = 0.3;
    
    if (leftWrist && leftWrist.confidence > confidenceThreshold &&
        rightWrist && rightWrist.confidence > confidenceThreshold &&
        nose && nose.confidence > confidenceThreshold) {
        
        // ตรวจสอบว่ายกมือเหนือหัว
        BOOL leftHandRaised = leftWrist.location.y > nose.location.y;
        BOOL rightHandRaised = rightWrist.location.y > nose.location.y;
        
        if (leftHandRaised && rightHandRaised) {
            NSLog(@"🙌 ยกมือทั้งสองข้างเหนือหัว!");
        }
    }
    
    // คำนวณมุมของข้อต่อ
    [self calculateJointAngles:joints];
}

- (void)calculateJointAngles:(NSDictionary *)joints API_AVAILABLE(ios(14.0)) {
    VNRecognizedPoint *leftShoulder = joints[VNHumanBodyPoseObservationJointNameLeftShoulder];
    VNRecognizedPoint *leftElbow = joints[VNHumanBodyPoseObservationJointNameLeftElbow];
    VNRecognizedPoint *leftWrist = joints[VNHumanBodyPoseObservationJointNameLeftWrist];
    
    if (leftShoulder && leftElbow && leftWrist) {
        // คำนวณมุมของข้อศอก
        CGPoint a = leftShoulder.location;
        CGPoint b = leftElbow.location;
        CGPoint c = leftWrist.location;
        
        float angle = [self angleBetweenA:a B:b C:c];
        NSLog(@"มุมข้อศอกซ้าย: %.1f°", angle);
    }
}

- (float)angleBetweenA:(CGPoint)a B:(CGPoint)b C:(CGPoint)c {
    CGFloat ab_x = a.x - b.x, ab_y = a.y - b.y;
    CGFloat cb_x = c.x - b.x, cb_y = c.y - b.y;
    
    CGFloat dot = ab_x * cb_x + ab_y * cb_y;
    CGFloat cross = ab_x * cb_y - ab_y * cb_x;
    
    return fabs(atan2(cross, dot)) * 180.0 / M_PI;
}

@end
```

---

## 88.12 Contour Detection

### การตรวจจับขอบของ Objects

```objc
@implementation ContourDetector

- (void)detectContours:(UIImage *)image {
    if (@available(iOS 14.0, *)) {
        VNDetectContoursRequest *request = [[VNDetectContoursRequest alloc] 
            initWithCompletionHandler:^(VNRequest *request, NSError *error) {
            
            VNContoursObservation *observation = request.results.firstObject;
            if (!observation) return;
            
            NSLog(@"พบ %lu contours", observation.contourCount);
            NSLog(@"Top-level contours: %lu", observation.topLevelContours.count);
            
            [self processContours:observation];
        }];
        
        // ตั้งค่า
        request.detectsDarkOnLight = YES;  // ตรวจจับเส้นมืดบนพื้นสว่าง
        request.contrastAdjustment = 1.0;  // ปรับ contrast (0.0 - 3.0)
        
        [self executeRequest:request withImage:image];
    }
}

- (void)processContours:(VNContoursObservation *)observation API_AVAILABLE(ios(14.0)) {
    for (NSInteger i = 0; i < observation.topLevelContours.count; i++) {
        VNContour *contour = observation.topLevelContours[i];
        
        NSLog(@"Contour %ld:", i);
        NSLog(@"  จำนวนจุด: %lu", contour.pointCount);
        NSLog(@"  Bounding box: %@", NSStringFromCGRect(contour.normalizedBoundingBox));
        
        // Simplify contour (ลดจำนวนจุด)
        NSError *error;
        VNContour *simplified = [contour polygonApproximationWithEpsilon:0.01 error:&error];
        NSLog(@"  หลัง simplify: %lu จุด", simplified.pointCount);
        
        // Child contours (holes)
        for (VNContour *child in contour.childContours) {
            NSLog(@"  Child contour: %lu จุด", child.pointCount);
        }
    }
    
    // แปลง contour เป็น path สำหรับ drawing
    UIBezierPath *path = [observation contourAtIndex:0 error:nil].normalizedPath ? 
        [UIBezierPath bezierPathWithCGPath:[observation contourAtIndex:0 error:nil].normalizedPath] : 
        nil;
}

@end
```

---

## 88.13 Saliency Detection

### การหาจุดที่น่าสนใจในภาพ

```objc
@implementation SaliencyDetector

- (void)detectSaliency:(UIImage *)image {
    // Attention-based saliency
    VNGenerateAttentionBasedSaliencyImageRequest *attentionRequest = 
        [[VNGenerateAttentionBasedSaliencyImageRequest alloc] 
            initWithCompletionHandler:^(VNRequest *request, NSError *error) {
        
        VNSaliencyImageObservation *observation = request.results.firstObject;
        CVPixelBufferRef saliencyMap = observation.pixelBuffer;
        
        // แสดง saliency map
        UIImage *saliencyImage = [CIImage imageWithCVPixelBuffer:saliencyMap].UIImage ?: nil;
        NSLog(@"Attention saliency map สร้างแล้ว");
        
        // Salient objects
        NSArray<VNRectangleObservation *> *objects = observation.salientObjects;
        for (VNRectangleObservation *obj in objects) {
            NSLog(@"Salient object: %@", NSStringFromCGRect(obj.boundingBox));
        }
    }];
    
    // Objectness-based saliency
    VNGenerateObjectnessBasedSaliencyImageRequest *objectnessRequest = 
        [[VNGenerateObjectnessBasedSaliencyImageRequest alloc] 
            initWithCompletionHandler:^(VNRequest *request, NSError *error) {
        
        VNSaliencyImageObservation *observation = request.results.firstObject;
        NSLog(@"Objectness saliency: %lu objects", observation.salientObjects.count);
    }];
    
    [self executeRequests:@[attentionRequest, objectnessRequest] withImage:image];
}

@end
```

---

## 88.14 On-Device Model Download

### การ Download Model ด้วย Core ML Model Deployment

```objc
@implementation ModelDownloader

// Download model จาก CloudKit หรือ server
- (void)downloadLatestModel {
    NSURL *modelURL = [NSURL URLWithString:@"https://example.com/model.mlmodel"];
    
    NSURLSessionDownloadTask *task = [[NSURLSession sharedSession] 
        downloadTaskWithURL:modelURL 
        completionHandler:^(NSURL *location, NSURLResponse *response, NSError *error) {
        
        if (error) {
            NSLog(@"Download error: %@", error);
            return;
        }
        
        // บันทึกไว้ใน documents
        NSString *documents = [NSSearchPathForDirectoriesInDomains(
            NSDocumentDirectory, NSUserDomainMask, YES) firstObject];
        NSURL *destURL = [[NSURL fileURLWithPath:documents] 
            URLByAppendingPathComponent:@"model.mlmodel"];
        
        [[NSFileManager defaultManager] moveItemAtURL:location toURL:destURL error:nil];
        
        // Compile model
        [self compileAndLoadModel:destURL];
    }];
    
    [task resume];
}

- (void)compileAndLoadModel:(NSURL *)modelURL {
    // Compile .mlmodel เป็น .mlmodelc
    NSError *error;
    NSURL *compiledURL = [MLModel compileModelAtURL:modelURL error:&error];
    
    if (error) {
        NSLog(@"Compile error: %@", error);
        return;
    }
    
    // บันทึก compiled model
    NSString *documents = [NSSearchPathForDirectoriesInDomains(
        NSDocumentDirectory, NSUserDomainMask, YES) firstObject];
    NSURL *permanentURL = [[NSURL fileURLWithPath:documents] 
        URLByAppendingPathComponent:@"model.mlmodelc"];
    
    [[NSFileManager defaultManager] copyItemAtURL:compiledURL 
                                            toURL:permanentURL 
                                            error:nil];
    
    // โหลด model
    MLModel *model = [MLModel modelWithContentsOfURL:permanentURL 
                                       configuration:nil 
                                               error:&error];
    if (!error) {
        NSLog(@"Model ดาวน์โหลดและโหลดสำเร็จ");
        self.currentModel = model;
    }
}

@end
```

---

## 88.15 แบบฝึกหัด

### แบบฝึกหัดที่ 1: Smart Camera App

```objc
// สร้างแอปกล้องอัจฉริยะที่:
// 1. Real-time object detection
// 2. Face detection พร้อม landmark
// 3. Text OCR
// 4. Barcode scanning
// 5. Body pose estimation
// ผู้ใช้เลือกโหมดได้จาก tab bar

@interface SmartCameraViewController : UIViewController 
    <AVCaptureVideoDataOutputSampleBufferDelegate>

@property (nonatomic, assign) NSInteger currentMode;  // 0-4

@end

@implementation SmartCameraViewController {
    AVCaptureSession *_captureSession;
    AVCaptureVideoPreviewLayer *_previewLayer;
    dispatch_queue_t _processingQueue;
    
    // Vision requests
    NSArray<VNRequest *> *_objectDetectionRequests;
    NSArray<VNRequest *> *_faceRequests;
    NSArray<VNRequest *> *_textRequests;
    NSArray<VNRequest *> *_barcodeRequests;
    NSArray<VNRequest *> *_poseRequests;
    
    // Overlay view สำหรับวาด bounding boxes
    DrawingView *_overlayView;
}

- (void)setupCamera {
    _captureSession = [[AVCaptureSession alloc] init];
    _captureSession.sessionPreset = AVCaptureSessionPreset1920x1080;
    
    AVCaptureDevice *camera = [AVCaptureDevice defaultDeviceWithMediaType:AVMediaTypeVideo];
    AVCaptureDeviceInput *input = [AVCaptureDeviceInput deviceInputWithDevice:camera error:nil];
    [_captureSession addInput:input];
    
    AVCaptureVideoDataOutput *output = [[AVCaptureVideoDataOutput alloc] init];
    output.videoSettings = @{
        (NSString *)kCVPixelBufferPixelFormatTypeKey: @(kCVPixelFormatType_32BGRA)
    };
    output.alwaysDiscardsLateVideoFrames = YES;
    
    _processingQueue = dispatch_queue_create("vision.processing", DISPATCH_QUEUE_SERIAL);
    [output setSampleBufferDelegate:self queue:_processingQueue];
    
    [_captureSession addOutput:output];
    
    // Preview layer
    _previewLayer = [AVCaptureVideoPreviewLayer layerWithSession:_captureSession];
    _previewLayer.videoGravity = AVLayerVideoGravityResizeAspectFill;
    _previewLayer.frame = self.view.bounds;
    [self.view.layer addSublayer:_previewLayer];
    
    [_captureSession startRunning];
}

- (void)captureOutput:(AVCaptureOutput *)output 
    didOutputSampleBuffer:(CMSampleBufferRef)sampleBuffer 
           fromConnection:(AVCaptureConnection *)connection {
    
    CVPixelBufferRef pixelBuffer = CMSampleBufferGetImageBuffer(sampleBuffer);
    
    NSArray<VNRequest *> *requests;
    switch (self.currentMode) {
        case 0: requests = _objectDetectionRequests; break;
        case 1: requests = _faceRequests; break;
        case 2: requests = _textRequests; break;
        case 3: requests = _barcodeRequests; break;
        case 4: requests = _poseRequests; break;
        default: return;
    }
    
    VNImageRequestHandler *handler = [[VNImageRequestHandler alloc] 
        initWithCVPixelBuffer:pixelBuffer 
                  orientation:kCGImagePropertyOrientationRight 
                      options:@{}];
    
    [handler performRequests:requests error:nil];
}

@end
```

### แบบฝึกหัดที่ 2: Sentiment Analyzer App

```
สร้างแอปวิเคราะห์ sentiment ของข้อความที่:
1. รับ text input จาก UITextView
2. ใช้ NL framework วิเคราะห์ภาษา
3. แสดง sentiment score (positive/negative/neutral)
4. Highlight คำสำคัญ
5. แสดง named entities ที่พบ
```

```objc
@implementation SentimentAnalyzerViewController {
    UITextView *_textView;
    UILabel *_sentimentLabel;
    UILabel *_languageLabel;
    UITextView *_entitiesTextView;
}

- (void)analyzeText {
    NSString *text = _textView.text;
    if (text.length == 0) return;
    
    // วิเคราะห์ภาษา
    NLLanguageRecognizer *recognizer = [[NLLanguageRecognizer alloc] init];
    [recognizer processString:text];
    NSString *language = recognizer.dominantLanguage;
    _languageLabel.text = [NSString stringWithFormat:@"ภาษา: %@", language];
    
    // Named entities
    NLTagger *tagger = [[NLTagger alloc] initWithTagSchemes:@[NLTagSchemeNameType]];
    tagger.string = text;
    
    NSMutableArray *entities = [NSMutableArray array];
    [tagger enumerateTagsInRange:NSMakeRange(0, text.length) 
                            unit:NLTokenUnitWord 
                          scheme:NLTagSchemeNameType 
                         options:NLTaggerOmitPunctuation | NLTaggerOmitWhitespace | NLTaggerJoinNames
                      usingBlock:^(NLTag tag, NSRange range, BOOL *stop) {
        if (tag) {
            NSString *entity = [text substringWithRange:range];
            [entities addObject:[NSString stringWithFormat:@"%@ (%@)", entity, tag]];
        }
    }];
    
    _entitiesTextView.text = [entities componentsJoinedByString:@"\n"];
    
    // Sentiment analysis (ต้องใช้ model ที่ train มาเอง)
    [self analyzeSentimentWithModel:text];
}

- (void)analyzeSentimentWithModel:(NSString *)text {
    // ใช้ NLModel ที่ train เอง
    NSError *error;
    NSURL *modelURL = [[NSBundle mainBundle] URLForResource:@"SentimentClassifier" 
                                             withExtension:@"mlmodelc"];
    
    if (!modelURL) {
        // ถ้าไม่มี model ใช้ heuristic แทน
        [self heuristicSentiment:text];
        return;
    }
    
    NLModel *model = [NLModel modelWithContentsOfURL:modelURL error:&error];
    NSString *sentiment = [model predictedLabelForString:text];
    
    UIColor *color = [UIColor systemGrayColor];
    if ([sentiment isEqualToString:@"positive"]) {
        color = [UIColor systemGreenColor];
    } else if ([sentiment isEqualToString:@"negative"]) {
        color = [UIColor systemRedColor];
    }
    
    _sentimentLabel.text = sentiment;
    _sentimentLabel.textColor = color;
}

- (void)heuristicSentiment:(NSString *)text {
    // Sentiment อย่างง่ายโดยดูจากคำ
    NSArray *positiveWords = @[@"ดี", @"ยอดเยี่ยม", @"สุดยอด", @"love", @"great", @"awesome"];
    NSArray *negativeWords = @[@"แย่", @"ไม่ดี", @"เสียใจ", @"bad", @"terrible", @"awful"];
    
    NSString *lowercased = [text lowercaseString];
    
    NSInteger positiveCount = 0, negativeCount = 0;
    for (NSString *word in positiveWords) {
        if ([lowercased containsString:word]) positiveCount++;
    }
    for (NSString *word in negativeWords) {
        if ([lowercased containsString:word]) negativeCount++;
    }
    
    if (positiveCount > negativeCount) {
        _sentimentLabel.text = @"😊 เชิงบวก";
        _sentimentLabel.textColor = [UIColor systemGreenColor];
    } else if (negativeCount > positiveCount) {
        _sentimentLabel.text = @"😞 เชิงลบ";
        _sentimentLabel.textColor = [UIColor systemRedColor];
    } else {
        _sentimentLabel.text = @"😐 กลาง";
        _sentimentLabel.textColor = [UIColor systemGrayColor];
    }
}

@end
```

### แบบฝึกหัดที่ 3: Plant Identification App

```
สร้างแอปจำแนกพืชที่:
1. ใช้ camera หรือ photo picker เพื่อเลือกรูปพืช
2. ใช้ Core ML model จำแนกประเภทพืช
3. แสดงชื่อพืช, ความน่าจะเป็น, และข้อมูลเพิ่มเติม
4. บันทึกประวัติการสแกน
5. Share ผลลัพธ์ผ่าน UIActivityViewController
```

---

## สรุป

ในบทนี้เราได้เรียนรู้:

- **Core ML**: การโหลดและใช้ ML models บน device
- **MLModel**: การ inspect model และ input/output
- **Image Classification**: การจำแนกประเภทรูปภาพ
- **Object Detection**: การตรวจจับและระบุตำแหน่ง objects
- **Natural Language**: Language detection, tokenization, NER, word embeddings
- **Create ML**: การ train model บน Mac
- **Vision Framework**: VNRequest, VNImageRequestHandler
- **Text Recognition**: OCR ด้วย VNRecognizeTextRequest
- **Face Detection**: การตรวจจับใบหน้าและ facial landmarks
- **Barcode Detection**: การสแกน QR code และ barcodes
- **Body Pose**: การตรวจจับท่าทางร่างกาย
- **Contour Detection**: การหาขอบของ objects
- **Saliency**: การหาจุดที่น่าสนใจในภาพ

### ทรัพยากรเพิ่มเติม

```
Apple Developer Documentation:
- Core ML: https://developer.apple.com/documentation/coreml
- Vision: https://developer.apple.com/documentation/vision
- Natural Language: https://developer.apple.com/documentation/naturallanguage
- Create ML: https://developer.apple.com/documentation/createml

Pre-trained models:
- Apple Models: https://developer.apple.com/machine-learning/models/
- Hugging Face (Core ML): https://huggingface.co/models?library=coreml
- TuriCreate: https://github.com/apple/turicreate

Tools:
- Core ML Tools: pip install coremltools
- Create ML App: Xcode → Open Developer Tool → Create ML
```

บทนี้เป็นบทสุดท้ายของ Part 81-90 แล้ว ยินดีด้วยที่เรียนมาถึงจุดนี้! 🎉

หัวข้อที่เรียนไปใน Part 81-90:
- Part 81: Swift Integration
- Part 82: Testing และ Debugging
- Part 83: Performance Optimization
- Part 84: Networking
- Part 85: macOS Development
- Part 86: Metal Framework
- Part 87: ARKit
- Part 88: Core ML และ Vision (บทนี้)
