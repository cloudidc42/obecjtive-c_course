# Part 27: NSData

## บทนำ

**NSData** เป็นคลาสที่ใช้เก็บข้อมูล binary (ข้อมูลดิบ) ในรูปแบบ byte sequence พอจะนึกภาพได้ว่ามันคือ "ถุงเก็บข้อมูล" ที่เก็บ bytes ได้โดยไม่สนใจว่า bytes เหล่านั้นแทนอะไร

NSData ใช้กันมากใน:
- การอ่าน/เขียนไฟล์
- การสื่อสารผ่านเครือข่าย (HTTP requests/responses)
- การเข้ารหัสข้อมูล (encryption, encoding)
- การแปลงข้อมูล (serialization, deserialization)
- การเก็บรูปภาพ, เสียง, ไฟล์ binary ต่าง ๆ

---

## 27.1 การสร้าง NSData

### จาก Bytes โดยตรง

```objc
#import <Foundation/Foundation.h>

int main(int argc, const char * argv[]) {
    @autoreleasepool {
        
        // วิธีที่ 1: สร้าง NSData ว่าง
        NSData *emptyData = [NSData data];
        NSLog(@"Empty data length: %lu bytes", (unsigned long)emptyData.length);
        
        // วิธีที่ 2: จาก byte array
        uint8_t bytes[] = {0x48, 0x65, 0x6C, 0x6C, 0x6F}; // "Hello" in ASCII
        NSData *bytesData = [NSData dataWithBytes:bytes length:sizeof(bytes)];
        NSLog(@"Bytes data: %@", bytesData);
        NSLog(@"Length: %lu", (unsigned long)bytesData.length);
        
        // วิธีที่ 3: จาก NSString
        NSString *text = @"Hello, World!";
        NSData *stringData = [text dataUsingEncoding:NSUTF8StringEncoding];
        NSLog(@"\nString data: %@", stringData);
        NSLog(@"String data length: %lu bytes", (unsigned long)stringData.length);
        
        // วิธีที่ 4: สร้างข้อมูลซ้ำ
        uint8_t zeroByte = 0x00;
        NSData *zeros = [NSData dataWithBytes:&zeroByte length:1];
        NSLog(@"\nZero byte: %@", zeros);
        
        // วิธีที่ 5: จาก NSData อื่น (copy)
        NSData *copy = [NSData dataWithData:bytesData];
        NSLog(@"Copy equals original: %@", 
              [copy isEqualToData:bytesData] ? @"YES" : @"NO");
        
    }
    return 0;
}
```

### จากไฟล์

```objc
#import <Foundation/Foundation.h>

int main(int argc, const char * argv[]) {
    @autoreleasepool {
        
        // สร้างไฟล์ทดสอบก่อน
        NSString *content = @"Line 1\nLine 2\nLine 3\n";
        NSString *filePath = @"/tmp/test_file.txt";
        [content writeToFile:filePath atomically:YES encoding:NSUTF8StringEncoding error:nil];
        
        // อ่านไฟล์เป็น NSData
        NSData *fileData = [NSData dataWithContentsOfFile:filePath];
        if (fileData) {
            NSLog(@"File read successfully: %lu bytes", (unsigned long)fileData.length);
            
            // แปลงกลับเป็น String
            NSString *fileContent = [[NSString alloc] initWithData:fileData
                                                          encoding:NSUTF8StringEncoding];
            NSLog(@"Content:\n%@", fileContent);
        } else {
            NSLog(@"Failed to read file");
        }
        
        // อ่านด้วย Error handling
        NSError *error = nil;
        NSData *dataWithError = [NSData dataWithContentsOfFile:@"/nonexistent/file"
                                                       options:NSDataReadingMappedIfSafe
                                                         error:&error];
        if (error) {
            NSLog(@"\nError reading file: %@", error.localizedDescription);
        }
        
        // อ่านจาก URL (file URL)
        NSURL *fileURL = [NSURL fileURLWithPath:filePath];
        NSData *urlData = [NSData dataWithContentsOfURL:fileURL];
        NSLog(@"\nFrom URL: %lu bytes", (unsigned long)urlData.length);
        
    }
    return 0;
}
```

---

## 27.2 NSMutableData - ข้อมูลที่แก้ไขได้

```objc
#import <Foundation/Foundation.h>

int main(int argc, const char * argv[]) {
    @autoreleasepool {
        
        // สร้าง NSMutableData
        NSMutableData *mutableData = [NSMutableData data];
        NSLog(@"Initial length: %lu", (unsigned long)mutableData.length);
        
        // เพิ่มข้อมูล
        uint8_t header[] = {0xFF, 0xFE, 0x00, 0x01};
        [mutableData appendBytes:header length:sizeof(header)];
        NSLog(@"After header: %@", mutableData);
        
        // เพิ่ม NSData อื่น
        NSString *payload = @"Hello";
        NSData *payloadData = [payload dataUsingEncoding:NSUTF8StringEncoding];
        [mutableData appendData:payloadData];
        NSLog(@"After payload: %@", mutableData);
        NSLog(@"Total length: %lu", (unsigned long)mutableData.length);
        
        // แทนที่ข้อมูลในช่วง
        uint8_t newBytes[] = {0xAA, 0xBB};
        [mutableData replaceBytesInRange:NSMakeRange(0, 2) withBytes:newBytes];
        NSLog(@"After replace: %@", mutableData);
        
        // ลบข้อมูล
        [mutableData deleteBytesInRange:NSMakeRange(0, 2)];
        NSLog(@"After delete: %@", mutableData);
        
        // เพิ่มขนาด (เติม zeros)
        [mutableData increaseLengthBy:4];
        NSLog(@"After increase: %lu bytes", (unsigned long)mutableData.length);
        
        // กำหนดขนาด (truncate หรือ extend)
        mutableData.length = 5;
        NSLog(@"After setLength(5): %@", mutableData);
        
        // สร้างด้วยขนาดที่กำหนด (ล้วนเป็น zeros)
        NSMutableData *zeroData = [NSMutableData dataWithLength:10];
        NSLog(@"\n10 zero bytes: %@", zeroData);
        
    }
    return 0;
}
```

---

## 27.3 การอ่านข้อมูลจาก NSData

```objc
#import <Foundation/Foundation.h>

int main(int argc, const char * argv[]) {
    @autoreleasepool {
        
        uint8_t rawBytes[] = {0x01, 0x02, 0x03, 0x04, 0x05, 
                              0x06, 0x07, 0x08, 0x09, 0x0A};
        NSData *data = [NSData dataWithBytes:rawBytes length:sizeof(rawBytes)];
        
        NSLog(@"Data: %@", data);
        NSLog(@"Length: %lu", (unsigned long)data.length);
        
        // เข้าถึง bytes โดยตรง
        const uint8_t *bytes = (const uint8_t *)data.bytes;
        NSLog(@"\nFirst byte: 0x%02X", bytes[0]);
        NSLog(@"Last byte: 0x%02X", bytes[data.length - 1]);
        
        // อ่านทีละ byte
        NSLog(@"\nAll bytes:");
        for (NSUInteger i = 0; i < data.length; i++) {
            printf("0x%02X ", bytes[i]);
        }
        printf("\n");
        
        // ดึง sub-data
        NSData *subData = [data subdataWithRange:NSMakeRange(2, 4)];
        NSLog(@"\nBytes 2-5: %@", subData);
        
        // getBytes:range:
        uint8_t buffer[4];
        [data getBytes:buffer range:NSMakeRange(3, 4)];
        NSLog(@"Bytes 3-6: %02X %02X %02X %02X", 
              buffer[0], buffer[1], buffer[2], buffer[3]);
        
        // ค้นหาข้อมูล
        uint8_t searchBytes[] = {0x05, 0x06};
        NSData *searchData = [NSData dataWithBytes:searchBytes length:2];
        NSRange found = [data rangeOfData:searchData 
                                  options:0 
                                    range:NSMakeRange(0, data.length)];
        if (found.location != NSNotFound) {
            NSLog(@"\nFound {0x05,0x06} at location: %lu", 
                  (unsigned long)found.location);
        }
        
    }
    return 0;
}
```

---

## 27.4 การอ่านและเขียนไฟล์

### เขียนไฟล์

```objc
#import <Foundation/Foundation.h>

int main(int argc, const char * argv[]) {
    @autoreleasepool {
        
        // เขียน text เป็น data
        NSString *text = @"สวัสดีโลก!\nHello, World!\n";
        NSData *textData = [text dataUsingEncoding:NSUTF8StringEncoding];
        
        NSString *path = @"/tmp/hello.txt";
        
        // วิธีที่ 1: writeToFile:atomically:
        BOOL success = [textData writeToFile:path atomically:YES];
        NSLog(@"Write success: %@", success ? @"YES" : @"NO");
        
        // วิธีที่ 2: writeToFile:options:error:
        NSError *error = nil;
        success = [textData writeToFile:@"/tmp/hello2.txt"
                                options:NSDataWritingAtomic
                                  error:&error];
        if (error) {
            NSLog(@"Error: %@", error.localizedDescription);
        }
        
        // วิธีที่ 3: writeToURL:atomically:
        NSURL *fileURL = [NSURL fileURLWithPath:@"/tmp/hello3.txt"];
        success = [textData writeToURL:fileURL atomically:YES];
        NSLog(@"Write to URL: %@", success ? @"OK" : @"FAIL");
        
        // ตรวจสอบว่าไฟล์ถูกสร้างแล้ว
        NSFileManager *fm = [NSFileManager defaultManager];
        if ([fm fileExistsAtPath:path]) {
            NSDictionary *attrs = [fm attributesOfItemAtPath:path error:nil];
            NSLog(@"File size: %@ bytes", attrs[NSFileSize]);
        }
        
    }
    return 0;
}
```

### อ่านไฟล์ Binary

```objc
#import <Foundation/Foundation.h>

int main(int argc, const char * argv[]) {
    @autoreleasepool {
        
        // สร้างไฟล์ binary ตัวอย่าง
        uint8_t binaryContent[] = {
            0x89, 0x50, 0x4E, 0x47, // PNG header magic bytes
            0x0D, 0x0A, 0x1A, 0x0A,
            0x00, 0x00, 0x00, 0x0D
        };
        NSData *pngHeader = [NSData dataWithBytes:binaryContent 
                                          length:sizeof(binaryContent)];
        NSString *binaryPath = @"/tmp/sample.bin";
        [pngHeader writeToFile:binaryPath atomically:YES];
        
        // อ่านกลับ
        NSData *readData = [NSData dataWithContentsOfFile:binaryPath];
        NSLog(@"Binary file: %lu bytes", (unsigned long)readData.length);
        
        // ตรวจสอบ magic bytes
        const uint8_t *b = (const uint8_t *)readData.bytes;
        if (readData.length >= 4 && 
            b[0] == 0x89 && b[1] == 0x50 && b[2] == 0x4E && b[3] == 0x47) {
            NSLog(@"File starts with PNG signature");
        }
        
        // อ่านแบบ Memory-mapped (ประหยัดหน่วยความจำ)
        NSError *error = nil;
        NSData *mappedData = [NSData dataWithContentsOfFile:binaryPath
                                                    options:NSDataReadingMappedIfSafe
                                                      error:&error];
        if (!error) {
            NSLog(@"Memory-mapped read: %lu bytes", (unsigned long)mappedData.length);
        }
        
        // อ่านแบบ streamed (ไฟล์ใหญ่มาก)
        NSInputStream *stream = [NSInputStream inputStreamWithFileAtPath:binaryPath];
        [stream open];
        
        uint8_t buffer[1024];
        NSInteger bytesRead;
        NSMutableData *streamData = [NSMutableData data];
        
        while ((bytesRead = [stream read:buffer maxLength:sizeof(buffer)]) > 0) {
            [streamData appendBytes:buffer length:bytesRead];
        }
        [stream close];
        
        NSLog(@"Stream read: %lu bytes", (unsigned long)streamData.length);
        NSLog(@"Data matches: %@", 
              [streamData isEqualToData:readData] ? @"YES" : @"NO");
        
    }
    return 0;
}
```

---

## 27.5 NSData Encoding - Base64

Base64 คือวิธีการ encode binary data ให้เป็น text ที่สามารถส่งผ่านระบบที่รับได้เฉพาะ text (เช่น JSON, email)

```objc
#import <Foundation/Foundation.h>

int main(int argc, const char * argv[]) {
    @autoreleasepool {
        
        // Encoding ข้อมูลเป็น Base64
        NSString *original = @"Hello, Objective-C! This is binary data.";
        NSData *data = [original dataUsingEncoding:NSUTF8StringEncoding];
        
        // Encode เป็น Base64 string
        NSString *base64 = [data base64EncodedStringWithOptions:0];
        NSLog(@"Original: %@", original);
        NSLog(@"Base64: %@", base64);
        
        // Decode จาก Base64
        NSData *decoded = [[NSData alloc] initWithBase64EncodedString:base64 options:0];
        NSString *decodedString = [[NSString alloc] initWithData:decoded 
                                                        encoding:NSUTF8StringEncoding];
        NSLog(@"Decoded: %@", decodedString);
        NSLog(@"Match: %@", [original isEqualToString:decodedString] ? @"YES" : @"NO");
        
        // Base64 พร้อม line breaks (มาตรฐาน MIME = 76 chars per line)
        NSString *base64WithLines = [data base64EncodedStringWithOptions:
                                     NSDataBase64Encoding76CharacterLineLength];
        NSLog(@"\nBase64 with line breaks:\n%@", base64WithLines);
        
        // Decode โดยไม่สนใจ whitespace
        NSData *decoded2 = [[NSData alloc] initWithBase64EncodedString:base64WithLines
                                                               options:NSDataBase64DecodingIgnoreUnknownCharacters];
        NSLog(@"Decoded2 matches: %@", [decoded2 isEqualToData:data] ? @"YES" : @"NO");
        
        // Encode เป็น Base64 Data (ไม่ใช่ string)
        NSData *base64Data = [data base64EncodedDataWithOptions:0];
        NSLog(@"\nBase64 as data: %lu bytes", (unsigned long)base64Data.length);
        
        // ใช้กับรูปภาพ (ตัวอย่าง)
        NSData *imageData = [NSData dataWithContentsOfFile:@"/tmp/image.png"];
        if (imageData) {
            NSString *imageBase64 = [imageData base64EncodedStringWithOptions:0];
            NSString *dataURL = [NSString stringWithFormat:@"data:image/png;base64,%@", 
                                 imageBase64];
            NSLog(@"Image as data URL length: %lu", (unsigned long)dataURL.length);
        }
        
    }
    return 0;
}
```

---

## 27.6 การแปลงระหว่าง NSData และ NSString

```objc
#import <Foundation/Foundation.h>

int main(int argc, const char * argv[]) {
    @autoreleasepool {
        
        // NSString -> NSData
        NSString *str = @"Hello, สวัสดี, 你好";
        
        // UTF-8 Encoding (แนะนำสำหรับ Unicode)
        NSData *utf8Data = [str dataUsingEncoding:NSUTF8StringEncoding];
        NSLog(@"UTF-8 bytes: %lu", (unsigned long)utf8Data.length);
        
        // UTF-16 Encoding
        NSData *utf16Data = [str dataUsingEncoding:NSUTF16StringEncoding];
        NSLog(@"UTF-16 bytes: %lu", (unsigned long)utf16Data.length);
        
        // ASCII Encoding (ไม่รองรับ Unicode)
        NSData *asciiData = [str dataUsingEncoding:NSASCIIStringEncoding
                              allowLossyConversion:YES];
        NSLog(@"ASCII bytes (lossy): %lu", (unsigned long)asciiData.length);
        
        // NSData -> NSString
        NSString *backFromUTF8 = [[NSString alloc] initWithData:utf8Data 
                                                       encoding:NSUTF8StringEncoding];
        NSLog(@"\nBack from UTF-8: %@", backFromUTF8);
        
        NSString *backFromUTF16 = [[NSString alloc] initWithData:utf16Data 
                                                        encoding:NSUTF16StringEncoding];
        NSLog(@"Back from UTF-16: %@", backFromUTF16);
        
        // ตรวจสอบ encoding ที่ใช้ได้
        NSStringEncoding enc;
        NSData *detectedData = [NSData dataWithContentsOfFile:@"/tmp/hello.txt"
                                                      options:0
                                                        error:nil];
        if (detectedData) {
            NSString *detected = [NSString stringWithContentsOfFile:@"/tmp/hello.txt"
                                                       usedEncoding:&enc
                                                              error:nil];
            NSLog(@"\nFile encoding: %lu", (unsigned long)enc);
            NSLog(@"Content: %@", detected);
        }
        
        // Handle ไม่สามารถแปลงได้
        NSData *invalidData = [NSData dataWithBytes:"\xFF\xFE\x00" length:3];
        NSString *invalid = [[NSString alloc] initWithData:invalidData 
                                                  encoding:NSUTF8StringEncoding];
        if (invalid == nil) {
            NSLog(@"\nInvalid UTF-8 data - cannot convert to string");
        }
        
    }
    return 0;
}
```

---

## 27.7 Data Comparison

```objc
#import <Foundation/Foundation.h>

int main(int argc, const char * argv[]) {
    @autoreleasepool {
        
        NSData *data1 = [@"Hello" dataUsingEncoding:NSUTF8StringEncoding];
        NSData *data2 = [@"Hello" dataUsingEncoding:NSUTF8StringEncoding];
        NSData *data3 = [@"World" dataUsingEncoding:NSUTF8StringEncoding];
        
        // ตรวจสอบความเท่ากัน
        BOOL equal12 = [data1 isEqualToData:data2];
        BOOL equal13 = [data1 isEqualToData:data3];
        
        NSLog(@"data1 == data2: %@", equal12 ? @"YES" : @"NO");
        NSLog(@"data1 == data3: %@", equal13 ? @"YES" : @"NO");
        
        // ตรวจสอบ prefix/suffix
        uint8_t pngMagic[] = {0x89, 0x50, 0x4E, 0x47};
        NSData *testData = [NSData dataWithBytes:pngMagic length:sizeof(pngMagic)];
        
        // สร้าง data ที่ขึ้นต้นด้วย PNG magic
        NSMutableData *pngLike = [NSMutableData dataWithBytes:pngMagic 
                                                       length:sizeof(pngMagic)];
        [pngLike appendData:[@"rest of data" dataUsingEncoding:NSUTF8StringEncoding]];
        
        // ตรวจสอบว่าขึ้นต้นด้วย PNG magic หรือไม่
        NSData *prefix = [pngLike subdataWithRange:NSMakeRange(0, testData.length)];
        BOOL isPNG = [prefix isEqualToData:testData];
        NSLog(@"\nIs PNG: %@", isPNG ? @"YES" : @"NO");
        
        // เปรียบเทียบแบบ byte-by-byte
        const uint8_t *b1 = (const uint8_t *)data1.bytes;
        const uint8_t *b3 = (const uint8_t *)data3.bytes;
        NSInteger result = memcmp(b1, b3, MIN(data1.length, data3.length));
        NSLog(@"\nLexicographic comparison: %ld", (long)result);
        
        // Hash
        NSLog(@"\ndata1 hash: %lu", (unsigned long)data1.hash);
        NSLog(@"data2 hash: %lu", (unsigned long)data2.hash);
        NSLog(@"Same hash: %@", data1.hash == data2.hash ? @"YES" : @"NO");
        
    }
    return 0;
}
```

---

## 27.8 Hex String จาก NSData

การแปลง NSData เป็น hex string เป็นเรื่องที่ทำบ่อยในการ debug และ cryptography

```objc
#import <Foundation/Foundation.h>

// Extension สำหรับแปลง NSData <-> Hex String
@interface NSData (HexString)
- (NSString *)hexString;
+ (NSData *)dataFromHexString:(NSString *)hexString;
@end

@implementation NSData (HexString)

- (NSString *)hexString {
    const uint8_t *bytes = (const uint8_t *)self.bytes;
    NSMutableString *hex = [NSMutableString stringWithCapacity:self.length * 2];
    for (NSUInteger i = 0; i < self.length; i++) {
        [hex appendFormat:@"%02x", bytes[i]];
    }
    return [hex copy];
}

+ (NSData *)dataFromHexString:(NSString *)hexString {
    // ลบ spaces และ prefix "0x"
    hexString = [hexString stringByReplacingOccurrencesOfString:@" " withString:@""];
    hexString = [hexString stringByReplacingOccurrencesOfString:@"0x" withString:@""];
    
    if (hexString.length % 2 != 0) return nil;
    
    NSMutableData *data = [NSMutableData dataWithCapacity:hexString.length / 2];
    
    for (NSUInteger i = 0; i < hexString.length; i += 2) {
        NSString *byteStr = [hexString substringWithRange:NSMakeRange(i, 2)];
        uint8_t byte = (uint8_t)strtol(byteStr.UTF8String, NULL, 16);
        [data appendBytes:&byte length:1];
    }
    
    return [data copy];
}

@end

int main(int argc, const char * argv[]) {
    @autoreleasepool {
        
        // NSData -> Hex String
        NSData *data = [@"Hello, World!" dataUsingEncoding:NSUTF8StringEncoding];
        NSString *hex = [data hexString];
        NSLog(@"Original: %@", 
              [[NSString alloc] initWithData:data encoding:NSUTF8StringEncoding]);
        NSLog(@"Hex: %@", hex);
        
        // Hex String -> NSData
        NSData *back = [NSData dataFromHexString:hex];
        NSString *restored = [[NSString alloc] initWithData:back 
                                                   encoding:NSUTF8StringEncoding];
        NSLog(@"Restored: %@", restored);
        NSLog(@"Match: %@", [data isEqualToData:back] ? @"YES" : @"NO");
        
        // ตัวอย่างกับ Binary data
        uint8_t binaryBytes[] = {0xDE, 0xAD, 0xBE, 0xEF, 0xCA, 0xFE};
        NSData *binaryData = [NSData dataWithBytes:binaryBytes length:sizeof(binaryBytes)];
        NSLog(@"\nBinary hex: %@", [binaryData hexString]);
        
        // สร้างจาก hex string
        NSData *fromHex = [NSData dataFromHexString:@"DEADBEEF"];
        NSLog(@"From 'DEADBEEF': %@", [fromHex hexString]);
        
        // Hex string พร้อม spaces
        NSData *fromSpaced = [NSData dataFromHexString:@"DE AD BE EF"];
        NSLog(@"From 'DE AD BE EF': %@", [fromSpaced hexString]);
        
    }
    return 0;
}
```

---

## 27.9 Network Data Handling

```objc
#import <Foundation/Foundation.h>

// Synchronous HTTP request (ใช้เพื่อ demo เท่านั้น, ใน production ใช้ async)
void fetchAndProcessData(NSString *urlString) {
    NSURL *url = [NSURL URLWithString:urlString];
    if (!url) {
        NSLog(@"Invalid URL: %@", urlString);
        return;
    }
    
    NSError *error = nil;
    NSData *responseData = [NSData dataWithContentsOfURL:url];
    
    if (!responseData) {
        NSLog(@"Failed to fetch data from: %@", urlString);
        return;
    }
    
    NSLog(@"Received %lu bytes", (unsigned long)responseData.length);
    
    // ลองแปลงเป็น JSON
    id json = [NSJSONSerialization JSONObjectWithData:responseData
                                             options:NSJSONReadingAllowFragments
                                               error:&error];
    if (json && !error) {
        NSLog(@"JSON parsed successfully");
        if ([json isKindOfClass:[NSDictionary class]]) {
            NSDictionary *dict = (NSDictionary *)json;
            NSLog(@"Keys: %@", dict.allKeys);
        }
    }
    
    // ลองแปลงเป็น String
    NSString *text = [[NSString alloc] initWithData:responseData 
                                           encoding:NSUTF8StringEncoding];
    if (text) {
        NSLog(@"As text (%lu chars): %.100@...", 
              (unsigned long)text.length, text);
    }
}

int main(int argc, const char * argv[]) {
    @autoreleasepool {
        
        // ตัวอย่างการสร้าง HTTP Request แบบง่าย
        NSMutableURLRequest *request = [NSMutableURLRequest 
                                        requestWithURL:[NSURL URLWithString:@"https://httpbin.org/post"]];
        request.HTTPMethod = @"POST";
        [request setValue:@"application/json" forHTTPHeaderField:@"Content-Type"];
        
        // สร้าง body data
        NSDictionary *body = @{
            @"name"  : @"Objective-C Developer",
            @"skill" : @"NSData",
            @"level" : @5
        };
        
        NSError *error = nil;
        NSData *bodyData = [NSJSONSerialization dataWithJSONObject:body
                                                           options:0
                                                             error:&error];
        request.HTTPBody = bodyData;
        
        NSLog(@"Request body: %lu bytes", (unsigned long)bodyData.length);
        NSString *bodyString = [[NSString alloc] initWithData:bodyData 
                                                     encoding:NSUTF8StringEncoding];
        NSLog(@"Body: %@", bodyString);
        
        // Response data processing
        NSData *sampleResponse = [@"{\"status\":\"ok\",\"received\":true}" 
                                  dataUsingEncoding:NSUTF8StringEncoding];
        
        id parsed = [NSJSONSerialization JSONObjectWithData:sampleResponse
                                                   options:0
                                                     error:&error];
        NSLog(@"\nParsed response: %@", parsed);
        
    }
    return 0;
}
```

### Async Network Request

```objc
#import <Foundation/Foundation.h>

void performAsyncRequest(NSString *urlString, void(^completion)(NSData *data, NSError *error)) {
    NSURL *url = [NSURL URLWithString:urlString];
    NSURLSession *session = [NSURLSession sharedSession];
    
    NSURLSessionDataTask *task = [session dataTaskWithURL:url
                                       completionHandler:^(NSData *data, 
                                                          NSURLResponse *response, 
                                                          NSError *error) {
        if (error) {
            dispatch_async(dispatch_get_main_queue(), ^{
                completion(nil, error);
            });
            return;
        }
        
        // ตรวจสอบ HTTP status code
        NSHTTPURLResponse *httpResponse = (NSHTTPURLResponse *)response;
        if (httpResponse.statusCode != 200) {
            NSError *statusError = [NSError errorWithDomain:@"HTTPError"
                                                       code:httpResponse.statusCode
                                                   userInfo:@{NSLocalizedDescriptionKey: 
                                                              [NSString stringWithFormat:@"HTTP %ld", 
                                                               (long)httpResponse.statusCode]}];
            dispatch_async(dispatch_get_main_queue(), ^{
                completion(nil, statusError);
            });
            return;
        }
        
        dispatch_async(dispatch_get_main_queue(), ^{
            completion(data, nil);
        });
    }];
    
    [task resume];
}

int main(int argc, const char * argv[]) {
    @autoreleasepool {
        
        dispatch_semaphore_t semaphore = dispatch_semaphore_create(0);
        
        performAsyncRequest(@"https://httpbin.org/json", ^(NSData *data, NSError *error) {
            if (error) {
                NSLog(@"Error: %@", error.localizedDescription);
            } else {
                NSLog(@"Received: %lu bytes", (unsigned long)data.length);
                
                NSDictionary *json = [NSJSONSerialization JSONObjectWithData:data
                                                                     options:0
                                                                       error:nil];
                NSLog(@"JSON: %@", json);
            }
            dispatch_semaphore_signal(semaphore);
        });
        
        dispatch_semaphore_wait(semaphore, dispatch_time(DISPATCH_TIME_NOW, 10 * NSEC_PER_SEC));
        
    }
    return 0;
}
```

---

## 27.10 Compression และ Decompression

iOS 13+ / macOS 10.15+ มี NSData compression built-in

```objc
#import <Foundation/Foundation.h>
#import <Compression/Compression.h>

int main(int argc, const char * argv[]) {
    @autoreleasepool {
        
        // สร้าง data ที่มีข้อมูลซ้ำกัน (ซึ่ง compress ได้ดี)
        NSMutableString *largeText = [NSMutableString string];
        for (int i = 0; i < 1000; i++) {
            [largeText appendString:@"Hello World! This is repetitive text. "];
        }
        NSData *originalData = [largeText dataUsingEncoding:NSUTF8StringEncoding];
        
        NSLog(@"Original size: %lu bytes", (unsigned long)originalData.length);
        
        // Compress ด้วย LZFSE (Apple's compression algorithm)
        NSError *error = nil;
        
        if (@available(macOS 10.15, iOS 13.0, *)) {
            NSData *compressed = [originalData compressedDataUsingAlgorithm:NSDataCompressionAlgorithmLZFSE
                                                                      error:&error];
            if (compressed && !error) {
                double ratio = (double)compressed.length / originalData.length * 100;
                NSLog(@"Compressed size: %lu bytes (%.1f%%)", 
                      (unsigned long)compressed.length, ratio);
                
                // Decompress
                NSData *decompressed = [compressed decompressedDataUsingAlgorithm:NSDataCompressionAlgorithmLZFSE
                                                                            error:&error];
                
                BOOL matches = [decompressed isEqualToData:originalData];
                NSLog(@"Decompressed matches original: %@", matches ? @"YES" : @"NO");
            }
            
            // ลอง algorithms อื่น ๆ
            NSArray *algorithms = @[
                @(NSDataCompressionAlgorithmZlib),
                @(NSDataCompressionAlgorithmLZMA),
                @(NSDataCompressionAlgorithmLZ4),
                @(NSDataCompressionAlgorithmLZFSE)
            ];
            NSArray *names = @[@"zlib", @"LZMA", @"LZ4", @"LZFSE"];
            
            NSLog(@"\nCompression comparison:");
            for (NSUInteger i = 0; i < algorithms.count; i++) {
                NSDataCompressionAlgorithm algo = [algorithms[i] intValue];
                NSData *comp = [originalData compressedDataUsingAlgorithm:algo error:&error];
                if (comp) {
                    double r = (double)comp.length / originalData.length * 100;
                    NSLog(@"  %@: %lu bytes (%.1f%%)", names[i], 
                          (unsigned long)comp.length, r);
                }
            }
        } else {
            NSLog(@"Compression API not available on this OS version");
            
            // ทางเลือก: ใช้ zlib โดยตรง
            NSLog(@"Use zlib compression library as alternative");
        }
        
    }
    return 0;
}
```

---

## 27.11 NSData กับรูปภาพ

```objc
#import <Foundation/Foundation.h>

#if TARGET_OS_IPHONE
#import <UIKit/UIKit.h>

int main(int argc, const char * argv[]) {
    @autoreleasepool {
        
        // UIImage -> NSData
        UIImage *image = [UIImage imageNamed:@"photo.jpg"];
        
        // JPEG
        NSData *jpegData = UIImageJPEGRepresentation(image, 0.8); // 80% quality
        NSLog(@"JPEG size: %lu bytes", (unsigned long)jpegData.length);
        
        // PNG (lossless)
        NSData *pngData = UIImagePNGRepresentation(image);
        NSLog(@"PNG size: %lu bytes", (unsigned long)pngData.length);
        
        // บันทึก
        [jpegData writeToFile:@"/tmp/photo.jpg" atomically:YES];
        [pngData writeToFile:@"/tmp/photo.png" atomically:YES];
        
        // NSData -> UIImage
        NSData *loadedData = [NSData dataWithContentsOfFile:@"/tmp/photo.jpg"];
        UIImage *loadedImage = [UIImage imageWithData:loadedData];
        NSLog(@"Loaded image size: %.0fx%.0f", 
              loadedImage.size.width, loadedImage.size.height);
        
        // ส่งรูปภาพผ่าน API (แปลงเป็น Base64)
        NSString *base64Image = [jpegData base64EncodedStringWithOptions:0];
        NSDictionary *payload = @{@"image": base64Image};
        NSData *jsonPayload = [NSJSONSerialization dataWithJSONObject:payload
                                                             options:0
                                                               error:nil];
        NSLog(@"JSON payload size: %lu bytes", (unsigned long)jsonPayload.length);
        
    }
    return 0;
}
#else

int main(int argc, const char * argv[]) {
    @autoreleasepool {
        NSLog(@"Image operations are primarily for iOS/macOS with AppKit/UIKit");
        
        // บน macOS ใช้ NSImage
        // NSImage *image = [[NSImage alloc] initWithContentsOfFile:@"photo.jpg"];
        // NSData *tiffData = [image TIFFRepresentation];
        // NSBitmapImageRep *rep = [NSBitmapImageRep imageRepWithData:tiffData];
        // NSData *jpegData = [rep representationUsingType:NSBitmapImageFileTypeJPEG
        //                                     properties:@{NSImageCompressionFactor: @0.8}];
        
        // ตัวอย่าง: ตรวจสอบ image format จาก magic bytes
        uint8_t jpegMagic[] = {0xFF, 0xD8, 0xFF};
        uint8_t pngMagic[]  = {0x89, 0x50, 0x4E, 0x47};
        uint8_t gifMagic[]  = {0x47, 0x49, 0x46, 0x38};
        uint8_t webpMagic[] = {0x52, 0x49, 0x46, 0x46}; // "RIFF"
        
        NSData *pngData = [NSData dataWithBytes:pngMagic length:sizeof(pngMagic)];
        
        NSString *detectImageFormat(NSData *data) {
            if (data.length < 4) return @"unknown";
            const uint8_t *b = (const uint8_t *)data.bytes;
            
            if (b[0] == 0xFF && b[1] == 0xD8 && b[2] == 0xFF) return @"JPEG";
            if (b[0] == 0x89 && b[1] == 0x50 && b[2] == 0x4E && b[3] == 0x47) return @"PNG";
            if (b[0] == 0x47 && b[1] == 0x49 && b[2] == 0x46) return @"GIF";
            if (b[0] == 0x52 && b[1] == 0x49 && b[2] == 0x46 && b[3] == 0x46) return @"WebP/RIFF";
            return @"unknown";
        }
        
        NSLog(@"Format: %@", detectImageFormat(pngData));
        
        NSData *jpegSample = [NSData dataWithBytes:jpegMagic length:sizeof(jpegMagic)];
        NSLog(@"Format: %@", detectImageFormat(jpegSample));
        
    }
    return 0;
}
#endif
```

---

## 27.12 NSData กับ NSCoding / NSKeyedArchiver

```objc
#import <Foundation/Foundation.h>

@interface UserSettings : NSObject <NSSecureCoding>
@property (nonatomic, strong) NSString *username;
@property (nonatomic, assign) NSInteger fontSize;
@property (nonatomic, assign) BOOL darkMode;
@property (nonatomic, strong) NSArray *recentFiles;
@end

@implementation UserSettings

+ (BOOL)supportsSecureCoding { return YES; }

- (instancetype)initWithCoder:(NSCoder *)coder {
    self = [super init];
    if (self) {
        _username    = [coder decodeObjectOfClass:[NSString class] forKey:@"username"];
        _fontSize    = [coder decodeIntegerForKey:@"fontSize"];
        _darkMode    = [coder decodeBoolForKey:@"darkMode"];
        _recentFiles = [coder decodeObjectOfClasses:
                        [NSSet setWithObjects:[NSArray class], [NSString class], nil]
                                            forKey:@"recentFiles"];
    }
    return self;
}

- (void)encodeWithCoder:(NSCoder *)coder {
    [coder encodeObject:self.username    forKey:@"username"];
    [coder encodeInteger:self.fontSize   forKey:@"fontSize"];
    [coder encodeBool:self.darkMode      forKey:@"darkMode"];
    [coder encodeObject:self.recentFiles forKey:@"recentFiles"];
}

@end

int main(int argc, const char * argv[]) {
    @autoreleasepool {
        
        // สร้าง settings
        UserSettings *settings = [[UserSettings alloc] init];
        settings.username = @"Alice";
        settings.fontSize = 16;
        settings.darkMode = YES;
        settings.recentFiles = @[@"doc1.txt", @"doc2.pdf", @"notes.md"];
        
        // Serialize เป็น NSData
        NSError *archiveError = nil;
        NSData *data = [NSKeyedArchiver archivedDataWithRootObject:settings
                                             requiringSecureCoding:YES
                                                            error:&archiveError];
        if (archiveError) {
            NSLog(@"Archive error: %@", archiveError);
        } else {
            NSLog(@"Serialized: %lu bytes", (unsigned long)data.length);
        }
        
        // บันทึกลงไฟล์
        NSString *filePath = @"/tmp/settings.dat";
        [data writeToFile:filePath atomically:YES];
        NSLog(@"Saved to: %@", filePath);
        
        // อ่านและ Deserialize
        NSData *loadedData = [NSData dataWithContentsOfFile:filePath];
        NSError *unarchiveError = nil;
        UserSettings *loaded = [NSKeyedUnarchiver unarchivedObjectOfClass:[UserSettings class]
                                                                 fromData:loadedData
                                                                    error:&unarchiveError];
        
        if (!unarchiveError && loaded) {
            NSLog(@"\nLoaded settings:");
            NSLog(@"  Username: %@", loaded.username);
            NSLog(@"  Font size: %ld", (long)loaded.fontSize);
            NSLog(@"  Dark mode: %@", loaded.darkMode ? @"YES" : @"NO");
            NSLog(@"  Recent files: %@", loaded.recentFiles);
        }
        
    }
    return 0;
}
```

---

## 27.13 NSData สำหรับ Cryptography

```objc
#import <Foundation/Foundation.h>
#import <CommonCrypto/CommonDigest.h>
#import <CommonCrypto/CommonCryptor.h>

// Category สำหรับ hashing
@interface NSData (Crypto)
- (NSString *)md5String;
- (NSString *)sha1String;
- (NSString *)sha256String;
@end

@implementation NSData (Crypto)

- (NSString *)md5String {
    uint8_t digest[CC_MD5_DIGEST_LENGTH];
    CC_MD5(self.bytes, (CC_LONG)self.length, digest);
    
    NSMutableString *hex = [NSMutableString stringWithCapacity:CC_MD5_DIGEST_LENGTH * 2];
    for (int i = 0; i < CC_MD5_DIGEST_LENGTH; i++) {
        [hex appendFormat:@"%02x", digest[i]];
    }
    return [hex copy];
}

- (NSString *)sha1String {
    uint8_t digest[CC_SHA1_DIGEST_LENGTH];
    CC_SHA1(self.bytes, (CC_LONG)self.length, digest);
    
    NSMutableString *hex = [NSMutableString stringWithCapacity:CC_SHA1_DIGEST_LENGTH * 2];
    for (int i = 0; i < CC_SHA1_DIGEST_LENGTH; i++) {
        [hex appendFormat:@"%02x", digest[i]];
    }
    return [hex copy];
}

- (NSString *)sha256String {
    uint8_t digest[CC_SHA256_DIGEST_LENGTH];
    CC_SHA256(self.bytes, (CC_LONG)self.length, digest);
    
    NSMutableString *hex = [NSMutableString stringWithCapacity:CC_SHA256_DIGEST_LENGTH * 2];
    for (int i = 0; i < CC_SHA256_DIGEST_LENGTH; i++) {
        [hex appendFormat:@"%02x", digest[i]];
    }
    return [hex copy];
}

@end

int main(int argc, const char * argv[]) {
    @autoreleasepool {
        
        NSString *message = @"Hello, Objective-C!";
        NSData *data = [message dataUsingEncoding:NSUTF8StringEncoding];
        
        NSLog(@"Message: %@", message);
        NSLog(@"MD5:    %@", [data md5String]);
        NSLog(@"SHA1:   %@", [data sha1String]);
        NSLog(@"SHA256: %@", [data sha256String]);
        
        // Verify: same input, same hash
        NSData *data2 = [message dataUsingEncoding:NSUTF8StringEncoding];
        NSLog(@"\nSame input, same hash: %@", 
              [[data sha256String] isEqualToString:[data2 sha256String]] ? @"YES" : @"NO");
        
        // Different input, different hash
        NSData *different = [@"different message" dataUsingEncoding:NSUTF8StringEncoding];
        NSLog(@"Different input, different hash: %@",
              ![[data sha256String] isEqualToString:[different sha256String]] ? @"YES" : @"NO");
        
        // ใช้สำหรับ checksum ของไฟล์
        NSData *fileData = [NSData dataWithContentsOfFile:@"/tmp/hello.txt"];
        if (fileData) {
            NSLog(@"\nFile checksum (SHA256): %@", [fileData sha256String]);
        }
        
    }
    return 0;
}
```

---

## 27.14 Data Chunking และ Streaming

```objc
#import <Foundation/Foundation.h>

// ประมวลผล data ทีละ chunk (สำหรับไฟล์ใหญ่)
void processDataInChunks(NSData *data, NSUInteger chunkSize, 
                          void(^chunkHandler)(NSData *chunk, NSUInteger offset)) {
    NSUInteger offset = 0;
    NSUInteger total = data.length;
    
    while (offset < total) {
        NSUInteger length = MIN(chunkSize, total - offset);
        NSData *chunk = [data subdataWithRange:NSMakeRange(offset, length)];
        chunkHandler(chunk, offset);
        offset += chunkSize;
    }
}

// Combine chunks กลับเป็น data
NSData *combineChunks(NSArray *chunks) {
    NSMutableData *combined = [NSMutableData data];
    for (NSData *chunk in chunks) {
        [combined appendData:chunk];
    }
    return [combined copy];
}

int main(int argc, const char * argv[]) {
    @autoreleasepool {
        
        // สร้าง data ขนาด 1KB
        NSMutableData *largeData = [NSMutableData dataWithLength:1024];
        uint8_t *bytes = (uint8_t *)largeData.mutableBytes;
        for (int i = 0; i < 1024; i++) {
            bytes[i] = (uint8_t)(i % 256);
        }
        
        NSLog(@"Total data: %lu bytes", (unsigned long)largeData.length);
        
        // ประมวลผลทีละ 256 bytes
        NSMutableArray *chunks = [NSMutableArray array];
        processDataInChunks(largeData, 256, ^(NSData *chunk, NSUInteger offset) {
            NSLog(@"Processing chunk at offset %lu: %lu bytes", 
                  (unsigned long)offset, (unsigned long)chunk.length);
            [chunks addObject:chunk];
        });
        
        NSLog(@"Number of chunks: %lu", (unsigned long)chunks.count);
        
        // รวมกลับ
        NSData *reassembled = combineChunks(chunks);
        NSLog(@"Reassembled: %lu bytes", (unsigned long)reassembled.length);
        NSLog(@"Matches original: %@", 
              [reassembled isEqualToData:largeData] ? @"YES" : @"NO");
        
        // OutputStream สำหรับเขียนทีละ chunk
        NSString *outputPath = @"/tmp/chunked_output.bin";
        NSOutputStream *outputStream = [NSOutputStream outputStreamToFileAtPath:outputPath
                                                                         append:NO];
        [outputStream open];
        
        for (NSData *chunk in chunks) {
            [outputStream write:(const uint8_t *)chunk.bytes maxLength:chunk.length];
        }
        [outputStream close];
        
        // ตรวจสอบไฟล์ที่เขียน
        NSData *written = [NSData dataWithContentsOfFile:outputPath];
        NSLog(@"Written file matches: %@", 
              [written isEqualToData:largeData] ? @"YES" : @"NO");
        
    }
    return 0;
}
```

---

## 27.15 NSData Utilities - Extensions ที่มีประโยชน์

```objc
#import <Foundation/Foundation.h>

@interface NSData (Utilities)
+ (NSData *)randomDataOfLength:(NSUInteger)length;
- (BOOL)startsWith:(NSData *)prefix;
- (BOOL)endsWith:(NSData *)suffix;
- (NSString *)prettyHexString;
- (NSUInteger)occurrencesOfData:(NSData *)data;
@end

@implementation NSData (Utilities)

+ (NSData *)randomDataOfLength:(NSUInteger)length {
    NSMutableData *data = [NSMutableData dataWithLength:length];
    SecRandomCopyBytes(kSecRandomDefault, length, data.mutableBytes);
    return [data copy];
}

- (BOOL)startsWith:(NSData *)prefix {
    if (prefix.length > self.length) return NO;
    NSData *head = [self subdataWithRange:NSMakeRange(0, prefix.length)];
    return [head isEqualToData:prefix];
}

- (BOOL)endsWith:(NSData *)suffix {
    if (suffix.length > self.length) return NO;
    NSData *tail = [self subdataWithRange:NSMakeRange(self.length - suffix.length, 
                                                       suffix.length)];
    return [tail isEqualToData:suffix];
}

- (NSString *)prettyHexString {
    const uint8_t *bytes = (const uint8_t *)self.bytes;
    NSMutableString *result = [NSMutableString string];
    
    for (NSUInteger i = 0; i < self.length; i++) {
        if (i > 0 && i % 16 == 0) [result appendString:@"\n"];
        else if (i > 0 && i % 8 == 0) [result appendString:@"  "];
        else if (i > 0) [result appendString:@" "];
        [result appendFormat:@"%02X", bytes[i]];
    }
    return [result copy];
}

- (NSUInteger)occurrencesOfData:(NSData *)data {
    NSUInteger count = 0;
    NSUInteger searchFrom = 0;
    
    while (searchFrom <= self.length - data.length) {
        NSRange range = [self rangeOfData:data
                                  options:0
                                    range:NSMakeRange(searchFrom, self.length - searchFrom)];
        if (range.location == NSNotFound) break;
        count++;
        searchFrom = range.location + 1;
    }
    return count;
}

@end

int main(int argc, const char * argv[]) {
    @autoreleasepool {
        
        // Random data
        NSData *random = [NSData randomDataOfLength:16];
        NSLog(@"Random 16 bytes: %@", random);
        
        // startsWith / endsWith
        NSData *data = [@"Hello, World!" dataUsingEncoding:NSUTF8StringEncoding];
        NSData *prefix = [@"Hello" dataUsingEncoding:NSUTF8StringEncoding];
        NSData *suffix = [@"World!" dataUsingEncoding:NSUTF8StringEncoding];
        
        NSLog(@"\nStarts with 'Hello': %@", [data startsWith:prefix] ? @"YES" : @"NO");
        NSLog(@"Ends with 'World!': %@", [data endsWith:suffix] ? @"YES" : @"NO");
        
        // Pretty hex
        NSData *sample = [NSData randomDataOfLength:32];
        NSLog(@"\nPretty hex dump:\n%@", [sample prettyHexString]);
        
        // Count occurrences
        NSData *text = [@"abcabcabc" dataUsingEncoding:NSUTF8StringEncoding];
        NSData *pattern = [@"abc" dataUsingEncoding:NSUTF8StringEncoding];
        NSUInteger count = [text occurrencesOfData:pattern];
        NSLog(@"\n'abc' appears %lu times", (unsigned long)count);
        
    }
    return 0;
}
```

---

## 27.16 แบบฝึกหัด (Practice Exercises)

### แบบฝึกหัดที่ 1: File Copy ด้วย NSData

```objc
#import <Foundation/Foundation.h>

BOOL copyFile(NSString *sourcePath, NSString *destinationPath) {
    // อ่านไฟล์ต้นฉบับ
    NSError *readError = nil;
    NSData *data = [NSData dataWithContentsOfFile:sourcePath
                                          options:NSDataReadingMappedIfSafe
                                            error:&readError];
    if (readError) {
        NSLog(@"Read error: %@", readError.localizedDescription);
        return NO;
    }
    
    // เขียนไปยัง destination
    NSError *writeError = nil;
    BOOL success = [data writeToFile:destinationPath
                             options:NSDataWritingAtomic
                               error:&writeError];
    if (writeError) {
        NSLog(@"Write error: %@", writeError.localizedDescription);
        return NO;
    }
    
    return success;
}

BOOL verifyFilesMatch(NSString *path1, NSString *path2) {
    NSData *data1 = [NSData dataWithContentsOfFile:path1];
    NSData *data2 = [NSData dataWithContentsOfFile:path2];
    return [data1 isEqualToData:data2];
}

int main(int argc, const char * argv[]) {
    @autoreleasepool {
        
        // สร้างไฟล์ต้นฉบับ
        NSString *source = @"/tmp/source_file.txt";
        NSString *dest = @"/tmp/dest_file.txt";
        
        [@"This is the content of the file.\nLine 2\nLine 3" 
         writeToFile:source atomically:YES encoding:NSUTF8StringEncoding error:nil];
        
        // Copy
        BOOL success = copyFile(source, dest);
        NSLog(@"Copy success: %@", success ? @"YES" : @"NO");
        
        // Verify
        BOOL matches = verifyFilesMatch(source, dest);
        NSLog(@"Files match: %@", matches ? @"YES" : @"NO");
        
        // แสดงเนื้อหา
        NSString *content = [NSString stringWithContentsOfFile:dest
                                                      encoding:NSUTF8StringEncoding
                                                         error:nil];
        NSLog(@"Destination content:\n%@", content);
        
    }
    return 0;
}
```

### แบบฝึกหัดที่ 2: Simple Binary Protocol Parser

```objc
#import <Foundation/Foundation.h>

// Protocol format:
// [4 bytes: magic "MSG\0"] [4 bytes: message length] [N bytes: message content]

NSData *createMessage(NSString *content) {
    NSData *contentData = [content dataUsingEncoding:NSUTF8StringEncoding];
    NSMutableData *message = [NSMutableData data];
    
    // Magic bytes
    uint8_t magic[] = {'M', 'S', 'G', 0x00};
    [message appendBytes:magic length:4];
    
    // Length (big-endian 32-bit)
    uint32_t length = (uint32_t)contentData.length;
    uint32_t bigEndianLength = CFSwapInt32HostToBig(length);
    [message appendBytes:&bigEndianLength length:4];
    
    // Content
    [message appendData:contentData];
    
    return [message copy];
}

NSString *parseMessage(NSData *data) {
    if (data.length < 8) return nil;
    
    const uint8_t *bytes = (const uint8_t *)data.bytes;
    
    // Validate magic
    if (bytes[0] != 'M' || bytes[1] != 'S' || bytes[2] != 'G' || bytes[3] != 0x00) {
        NSLog(@"Invalid magic bytes");
        return nil;
    }
    
    // Read length
    uint32_t bigEndianLength;
    memcpy(&bigEndianLength, bytes + 4, 4);
    uint32_t length = CFSwapInt32BigToHost(bigEndianLength);
    
    if (data.length < 8 + length) {
        NSLog(@"Data too short");
        return nil;
    }
    
    // Extract content
    NSData *contentData = [data subdataWithRange:NSMakeRange(8, length)];
    return [[NSString alloc] initWithData:contentData encoding:NSUTF8StringEncoding];
}

int main(int argc, const char * argv[]) {
    @autoreleasepool {
        
        NSArray *messages = @[@"Hello", @"สวัสดี", @"Message with spaces!"];
        
        for (NSString *msg in messages) {
            NSData *packet = createMessage(msg);
            NSLog(@"Created packet: %lu bytes for '%@'", 
                  (unsigned long)packet.length, msg);
            
            NSString *parsed = parseMessage(packet);
            NSLog(@"Parsed: '%@'", parsed);
            NSLog(@"Match: %@\n", [msg isEqualToString:parsed] ? @"YES" : @"NO");
        }
        
        // Invalid data
        NSData *invalid = [@"invalid data" dataUsingEncoding:NSUTF8StringEncoding];
        NSString *result = parseMessage(invalid);
        NSLog(@"Invalid parse result: %@", result);
        
    }
    return 0;
}
```

### แบบฝึกหัดที่ 3: Data Diff

```objc
#import <Foundation/Foundation.h>

// หาความแตกต่างระหว่าง 2 NSData objects
NSDictionary *diffData(NSData *data1, NSData *data2) {
    NSMutableArray *differences = [NSMutableArray array];
    NSUInteger maxLength = MAX(data1.length, data2.length);
    
    const uint8_t *b1 = (const uint8_t *)data1.bytes;
    const uint8_t *b2 = (const uint8_t *)data2.bytes;
    
    for (NSUInteger i = 0; i < maxLength; i++) {
        if (i >= data1.length) {
            [differences addObject:@{@"offset": @(i), @"type": @"added", @"value": @(b2[i])}];
        } else if (i >= data2.length) {
            [differences addObject:@{@"offset": @(i), @"type": @"removed", @"value": @(b1[i])}];
        } else if (b1[i] != b2[i]) {
            [differences addObject:@{@"offset": @(i), @"type": @"changed",
                                     @"from": @(b1[i]), @"to": @(b2[i])}];
        }
    }
    
    return @{
        @"differences": differences,
        @"isDifferent": @(differences.count > 0),
        @"sameBytes": @(maxLength - differences.count)
    };
}

int main(int argc, const char * argv[]) {
    @autoreleasepool {
        
        NSData *v1 = [@"Hello World" dataUsingEncoding:NSUTF8StringEncoding];
        NSData *v2 = [@"Hello Swift" dataUsingEncoding:NSUTF8StringEncoding];
        
        NSDictionary *diff = diffData(v1, v2);
        
        NSLog(@"Data 1: %@", [[NSString alloc] initWithData:v1 encoding:NSUTF8StringEncoding]);
        NSLog(@"Data 2: %@", [[NSString alloc] initWithData:v2 encoding:NSUTF8StringEncoding]);
        NSLog(@"Is different: %@", diff[@"isDifferent"]);
        NSLog(@"Same bytes: %@", diff[@"sameBytes"]);
        
        NSLog(@"\nDifferences:");
        for (NSDictionary *d in diff[@"differences"]) {
            NSString *type = d[@"type"];
            NSUInteger offset = [d[@"offset"] unsignedIntegerValue];
            
            if ([type isEqualToString:@"changed"]) {
                NSLog(@"  Offset %lu: 0x%02X -> 0x%02X ('%c' -> '%c')",
                      (unsigned long)offset,
                      [d[@"from"] unsignedCharValue],
                      [d[@"to"] unsignedCharValue],
                      [d[@"from"] unsignedCharValue],
                      [d[@"to"] unsignedCharValue]);
            }
        }
        
    }
    return 0;
}
```

---

## สรุปบทที่ 27

ในบทนี้เราได้เรียนรู้:

1. **NSData** - เก็บข้อมูล binary (byte sequence) ไม่สนใจ type
2. **NSMutableData** - NSData ที่แก้ไขได้ (appendData, replaceBytes, deleteBytes)
3. **การอ่านไฟล์** - dataWithContentsOfFile:, dataWithContentsOfURL:
4. **การเขียนไฟล์** - writeToFile:atomically:, writeToURL:atomically:
5. **Base64 Encoding** - base64EncodedStringWithOptions:, initWithBase64EncodedString:
6. **NSString <-> NSData** - dataUsingEncoding:, initWithData:encoding:
7. **Hex String** - การแปลงระหว่าง hex string และ NSData
8. **Network** - รับ/ส่งข้อมูลผ่าน NSURLSession
9. **Compression** - compressedDataUsingAlgorithm: (iOS 13+/macOS 10.15+)
10. **Cryptography** - MD5, SHA1, SHA256 ด้วย CommonCrypto
11. **NSCoding** - Serialize/Deserialize object เป็น NSData

### คำถามทบทวน

1. ความแตกต่างระหว่าง NSData และ NSString คืออะไร?
2. ทำไมต้องใช้ Base64 encoding?
3. ทำไมไม่ควรใช้ MD5 สำหรับ password hashing?
4. NSDataReadingMappedIfSafe ทำงานอย่างไรและมีข้อดีอะไร?

---

**ต่อไป:** Part 28 - NSURL
