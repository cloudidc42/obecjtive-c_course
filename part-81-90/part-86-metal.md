# Part 86: Metal Framework - GPU Programming

## บทนำ

Metal เป็น low-level graphics และ compute API ของ Apple ที่ให้ประสิทธิภาพสูงในการเข้าถึง GPU บน iOS, macOS และ tvOS Metal ถูกออกแบบมาเพื่อลด CPU overhead และให้ developer ควบคุม GPU ได้โดยตรง เปิดตัวครั้งแรกในปี 2014 พร้อม iOS 8

---

## 86.1 Metal Overview และ GPU Programming Concepts

### ทำไมต้อง Metal?

```
CPU (Central Processing Unit)     GPU (Graphics Processing Unit)
- หน่วยประมวลผลทั่วไป             - หน่วยประมวลผลกราฟิก
- 4-16 cores (general purpose)   - หลายพันหน่วยประมวลผล
- เหมาะกับงาน sequential         - เหมาะกับงาน parallel
- latency สั้น                   - throughput สูง
- ดีสำหรับ: logic, I/O           - ดีสำหรับ: rendering, compute
```

### Metal vs เทคโนโลยีเก่า

```
OpenGL ES (เก่า)          Metal (ใหม่)
- High-level API          - Low-level API
- CPU overhead สูง        - CPU overhead ต่ำ
- Driver validation       - App-controlled validation
- Global state            - Explicit state objects
- Cross-platform          - Apple-only แต่มี performance ดีกว่า
- Deprecated บน iOS 12+   - ใช้งานได้บนทุก Apple platform
```

### Metal Pipeline Overview

```
Application Code (Objective-C/Swift)
    ↓
Metal API (MTLDevice, MTLCommandQueue, etc.)
    ↓
Metal Compiler (MSL → GPU machine code)
    ↓
GPU Hardware
    ↓
Display / Compute Results
```

---

## 86.2 MTLDevice - การเข้าถึง GPU

### การสร้าง MTLDevice

```objc
#import <Metal/Metal.h>
#import <MetalKit/MetalKit.h>

@implementation MetalRenderer {
    id<MTLDevice> _device;
}

- (instancetype)init {
    self = [super init];
    if (self) {
        // ขอ default GPU device
        _device = MTLCreateSystemDefaultDevice();
        
        if (!_device) {
            NSLog(@"Metal ไม่รองรับบนอุปกรณ์นี้");
            return nil;
        }
        
        NSLog(@"GPU: %@", _device.name);
        NSLog(@"ขนาด RAM ที่แบ่งกัน: %lu MB", _device.recommendedMaxWorkingSetSize / 1024 / 1024);
        NSLog(@"รองรับ unified memory: %d", _device.hasUnifiedMemory);
    }
    return self;
}

// บน macOS อาจมีหลาย GPU
- (void)enumerateGPUs {
    NSArray<id<MTLDevice>> *devices = MTLCopyAllDevices();
    
    for (id<MTLDevice> device in devices) {
        NSLog(@"พบ GPU: %@", device.name);
        NSLog(@"  Low power: %d", device.isLowPower);
        NSLog(@"  Headless: %d", device.isHeadless);
        NSLog(@"  Removable: %d", device.isRemovable);
    }
}

@end
```

### MTLDevice Capabilities

```objc
- (void)checkCapabilities {
    // ตรวจสอบ features ที่รองรับ
    NSLog(@"รองรับ Raytracing: %d", [_device supportsRaytracing]);
    NSLog(@"รองรับ Mesh Shaders: %d", [_device supportsMeshShaders]);
    NSLog(@"รองรับ shader atomics: %d", 
          [_device supportsFamily:MTLGPUFamilyApple7]);
    
    // ตรวจสอบ texture limits
    NSLog(@"Max texture size: %lu", _device.maxTextureArgumentEntriesPerFunction);
    NSLog(@"Max buffer length: %lu MB", _device.maxBufferLength / 1024 / 1024);
}
```

---

## 86.3 MTLCommandQueue และ MTLCommandBuffer

### Command Queue คืออะไร?

Command Queue เป็น queue ที่เก็บ command buffers ที่จะส่งไปยัง GPU ตามลำดับ

```
Application → CommandQueue → [Buffer1] [Buffer2] [Buffer3] → GPU
```

### การสร้างและใช้ Command Queue

```objc
@implementation MetalRenderer {
    id<MTLDevice> _device;
    id<MTLCommandQueue> _commandQueue;
}

- (void)setupMetal {
    _device = MTLCreateSystemDefaultDevice();
    
    // สร้าง command queue (สร้างครั้งเดียวนำมาใช้ซ้ำ)
    _commandQueue = [_device newCommandQueue];
    _commandQueue.label = @"Main Command Queue";  // สำหรับ debugging
}

- (void)renderFrame {
    // สร้าง command buffer สำหรับแต่ละ frame
    id<MTLCommandBuffer> commandBuffer = [_commandQueue commandBuffer];
    commandBuffer.label = @"Frame Command Buffer";
    
    // เพิ่ม completion handler
    [commandBuffer addCompletedHandler:^(id<MTLCommandBuffer> buffer) {
        // ทำงานเมื่อ GPU ประมวลผลเสร็จ
        if (buffer.error) {
            NSLog(@"GPU error: %@", buffer.error);
        } else {
            NSLog(@"Frame ประมวลผลเสร็จใน: %.3f ms", 
                  (buffer.GPUEndTime - buffer.GPUStartTime) * 1000);
        }
    }];
    
    // เพิ่ม render commands...
    // (จะอธิบายในหัวข้อถัดไป)
    
    // ส่ง command buffer ไปยัง GPU
    [commandBuffer commit];
    
    // รอให้ GPU เสร็จ (blocking - ใช้เฉพาะตอน debug)
    // [commandBuffer waitUntilCompleted];
}

@end
```

### การจัดการ Frame Synchronization

```objc
// ใช้ semaphore เพื่อ limit จำนวน frames in-flight
static const NSInteger kMaxFramesInFlight = 3;

@implementation MetalRenderer {
    dispatch_semaphore_t _frameSemaphore;
    NSInteger _currentFrameIndex;
}

- (void)setupSynchronization {
    _frameSemaphore = dispatch_semaphore_create(kMaxFramesInFlight);
    _currentFrameIndex = 0;
}

- (void)renderFrame {
    // รอถ้า GPU ยังประมวลผลค้างอยู่
    dispatch_semaphore_wait(_frameSemaphore, DISPATCH_TIME_FOREVER);
    
    id<MTLCommandBuffer> commandBuffer = [_commandQueue commandBuffer];
    
    __block dispatch_semaphore_t semaphore = _frameSemaphore;
    [commandBuffer addCompletedHandler:^(id<MTLCommandBuffer> buffer) {
        dispatch_semaphore_signal(semaphore);  // ปลดล็อค
    }];
    
    // render...
    
    [commandBuffer commit];
    
    _currentFrameIndex = (_currentFrameIndex + 1) % kMaxFramesInFlight;
}

@end
```

---

## 86.4 Metal Shading Language (MSL)

### พื้นฐาน MSL

MSL คือ shading language ของ Metal ซึ่ง based on C++14

```metal
// Shaders.metal

#include <metal_stdlib>
using namespace metal;

// ========== Vertex Shader ==========

// โครงสร้างข้อมูล vertex
struct VertexIn {
    float3 position [[attribute(0)]];  // position ใน 3D space
    float4 color    [[attribute(1)]];  // color
    float2 texCoord [[attribute(2)]];  // texture coordinate
};

// ข้อมูลที่ส่งจาก vertex shader ไปยัง fragment shader
struct VertexOut {
    float4 position [[position]];  // clip space position (required)
    float4 color;
    float2 texCoord;
};

// Vertex shader function
vertex VertexOut vertexShader(
    VertexIn in [[stage_in]],
    constant float4x4 &modelMatrix [[buffer(1)]],
    constant float4x4 &viewMatrix [[buffer(2)]],
    constant float4x4 &projectionMatrix [[buffer(3)]]
) {
    VertexOut out;
    
    // แปลง position จาก model space → clip space
    float4 worldPos = modelMatrix * float4(in.position, 1.0);
    float4 viewPos = viewMatrix * worldPos;
    out.position = projectionMatrix * viewPos;
    
    out.color = in.color;
    out.texCoord = in.texCoord;
    
    return out;
}

// ========== Fragment Shader ==========

// Fragment shader function
fragment float4 fragmentShader(
    VertexOut in [[stage_in]],
    texture2d<float> colorTexture [[texture(0)]],
    sampler textureSampler [[sampler(0)]]
) {
    // อ่านสีจาก texture
    float4 texColor = colorTexture.sample(textureSampler, in.texCoord);
    
    // ผสมสีกับ vertex color
    return texColor * in.color;
}
```

### Compute Shader

```metal
// Compute shader สำหรับ parallel computation
kernel void imageProcessing(
    texture2d<float, access::read> inputTexture [[texture(0)]],
    texture2d<float, access::write> outputTexture [[texture(1)]],
    uint2 gid [[thread_position_in_grid]]
) {
    // ตรวจสอบขอบเขต
    if (gid.x >= outputTexture.get_width() || gid.y >= outputTexture.get_height()) {
        return;
    }
    
    // อ่านสีจาก input texture
    float4 color = inputTexture.read(gid);
    
    // แปลงเป็น grayscale
    float gray = dot(color.rgb, float3(0.2126, 0.7152, 0.0722));
    
    // เขียนไปยัง output texture
    outputTexture.write(float4(gray, gray, gray, color.a), gid);
}

// Kernel สำหรับ vector addition
kernel void vectorAdd(
    device const float *vectorA [[buffer(0)]],
    device const float *vectorB [[buffer(1)]],
    device float *resultVector [[buffer(2)]],
    uint index [[thread_position_in_grid]]
) {
    resultVector[index] = vectorA[index] + vectorB[index];
}
```

---

## 86.5 MTLRenderPipelineState

### การสร้าง Render Pipeline

```objc
@implementation MetalRenderer {
    id<MTLDevice> _device;
    id<MTLRenderPipelineState> _pipelineState;
    id<MTLDepthStencilState> _depthStencilState;
}

- (BOOL)buildPipelineState {
    // โหลด shaders จาก default Metal library
    id<MTLLibrary> library = [_device newDefaultLibrary];
    if (!library) {
        NSLog(@"ไม่พบ Metal shader library");
        return NO;
    }
    
    // โหลด vertex และ fragment functions
    id<MTLFunction> vertexFunction = [library newFunctionWithName:@"vertexShader"];
    id<MTLFunction> fragmentFunction = [library newFunctionWithName:@"fragmentShader"];
    
    if (!vertexFunction || !fragmentFunction) {
        NSLog(@"ไม่พบ shader functions");
        return NO;
    }
    
    // สร้าง pipeline descriptor
    MTLRenderPipelineDescriptor *descriptor = [[MTLRenderPipelineDescriptor alloc] init];
    descriptor.label = @"Main Render Pipeline";
    descriptor.vertexFunction = vertexFunction;
    descriptor.fragmentFunction = fragmentFunction;
    
    // กำหนด pixel format สำหรับ color attachment
    descriptor.colorAttachments[0].pixelFormat = MTLPixelFormatBGRA8Unorm;
    
    // กำหนด blending สำหรับ transparency
    MTLRenderPipelineColorAttachmentDescriptor *colorDesc = descriptor.colorAttachments[0];
    colorDesc.blendingEnabled = YES;
    colorDesc.rgbBlendOperation = MTLBlendOperationAdd;
    colorDesc.alphaBlendOperation = MTLBlendOperationAdd;
    colorDesc.sourceRGBBlendFactor = MTLBlendFactorSourceAlpha;
    colorDesc.destinationRGBBlendFactor = MTLBlendFactorOneMinusSourceAlpha;
    colorDesc.sourceAlphaBlendFactor = MTLBlendFactorOne;
    colorDesc.destinationAlphaBlendFactor = MTLBlendFactorOneMinusSourceAlpha;
    
    // กำหนด depth buffer format
    descriptor.depthAttachmentPixelFormat = MTLPixelFormatDepth32Float;
    
    // กำหนด vertex descriptor
    descriptor.vertexDescriptor = [self buildVertexDescriptor];
    
    // สร้าง pipeline state
    NSError *error;
    _pipelineState = [_device newRenderPipelineStateWithDescriptor:descriptor error:&error];
    
    if (error) {
        NSLog(@"Pipeline error: %@", error);
        return NO;
    }
    
    // สร้าง depth stencil state
    MTLDepthStencilDescriptor *depthDesc = [[MTLDepthStencilDescriptor alloc] init];
    depthDesc.depthCompareFunction = MTLCompareFunctionLess;
    depthDesc.depthWriteEnabled = YES;
    _depthStencilState = [_device newDepthStencilStateWithDescriptor:depthDesc];
    
    return YES;
}

- (MTLVertexDescriptor *)buildVertexDescriptor {
    MTLVertexDescriptor *vertexDescriptor = [[MTLVertexDescriptor alloc] init];
    
    // Position attribute (float3 = 12 bytes)
    vertexDescriptor.attributes[0].format = MTLVertexFormatFloat3;
    vertexDescriptor.attributes[0].offset = 0;
    vertexDescriptor.attributes[0].bufferIndex = 0;
    
    // Color attribute (float4 = 16 bytes)
    vertexDescriptor.attributes[1].format = MTLVertexFormatFloat4;
    vertexDescriptor.attributes[1].offset = 12;
    vertexDescriptor.attributes[1].bufferIndex = 0;
    
    // Texture coord attribute (float2 = 8 bytes)
    vertexDescriptor.attributes[2].format = MTLVertexFormatFloat2;
    vertexDescriptor.attributes[2].offset = 28;
    vertexDescriptor.attributes[2].bufferIndex = 0;
    
    // Stride = 12 + 16 + 8 = 36 bytes per vertex
    vertexDescriptor.layouts[0].stride = 36;
    vertexDescriptor.layouts[0].stepFunction = MTLVertexStepFunctionPerVertex;
    
    return vertexDescriptor;
}

@end
```

---

## 86.6 Hello Triangle - การวาด Triangle แรก

### Vertex Data และ Buffer

```objc
// โครงสร้าง Vertex (ต้องตรงกับ MSL struct)
typedef struct {
    vector_float3 position;
    vector_float4 color;
} Vertex;

@implementation TriangleRenderer {
    id<MTLDevice> _device;
    id<MTLCommandQueue> _commandQueue;
    id<MTLRenderPipelineState> _pipelineState;
    id<MTLBuffer> _vertexBuffer;
}

- (void)setupBuffers {
    // กำหนด vertices ของสามเหลี่ยม
    // พิกัดใน Normalized Device Coordinates (-1 ถึง 1)
    static const Vertex vertices[] = {
        // position (x, y, z)      color (r, g, b, a)
        {{ 0.0,  0.5, 0.0}, {1.0, 0.0, 0.0, 1.0}},  // บน - แดง
        {{-0.5, -0.5, 0.0}, {0.0, 1.0, 0.0, 1.0}},  // ซ้ายล่าง - เขียว
        {{ 0.5, -0.5, 0.0}, {0.0, 0.0, 1.0, 1.0}},  // ขวาล่าง - น้ำเงิน
    };
    
    // สร้าง vertex buffer
    _vertexBuffer = [_device newBufferWithBytes:vertices
                                         length:sizeof(vertices)
                                        options:MTLResourceStorageModeShared];
    _vertexBuffer.label = @"Triangle Vertices";
}

- (void)drawInMTKView:(MTKView *)view {
    id<MTLCommandBuffer> commandBuffer = [_commandQueue commandBuffer];
    commandBuffer.label = @"Triangle Frame";
    
    // ดึง render pass descriptor จาก MTKView
    MTLRenderPassDescriptor *renderPassDescriptor = view.currentRenderPassDescriptor;
    if (!renderPassDescriptor) return;
    
    // ตั้งค่า background color
    renderPassDescriptor.colorAttachments[0].clearColor = MTLClearColorMake(0.1, 0.1, 0.2, 1.0);
    renderPassDescriptor.colorAttachments[0].loadAction = MTLLoadActionClear;
    renderPassDescriptor.colorAttachments[0].storeAction = MTLStoreActionStore;
    
    // สร้าง render command encoder
    id<MTLRenderCommandEncoder> encoder = [commandBuffer 
        renderCommandEncoderWithDescriptor:renderPassDescriptor];
    encoder.label = @"Triangle Encoder";
    
    // ตั้งค่า pipeline
    [encoder setRenderPipelineState:_pipelineState];
    
    // ผูก vertex buffer
    [encoder setVertexBuffer:_vertexBuffer offset:0 atIndex:0];
    
    // วาด triangle
    [encoder drawPrimitives:MTLPrimitiveTypeTriangle 
                vertexStart:0 
                vertexCount:3];
    
    // สิ้นสุดการ encode
    [encoder endEncoding];
    
    // แสดงผลบนหน้าจอ
    [commandBuffer presentDrawable:view.currentDrawable];
    [commandBuffer commit];
}

@end
```

### Triangle Shader (MSL)

```metal
// TriangleShaders.metal

#include <metal_stdlib>
using namespace metal;

struct VertexIn {
    float3 position [[attribute(0)]];
    float4 color    [[attribute(1)]];
};

struct VertexOut {
    float4 position [[position]];
    float4 color;
};

vertex VertexOut triangleVertexShader(
    VertexIn in [[stage_in]]
) {
    VertexOut out;
    // แปลง 3D position เป็น 4D clip space
    out.position = float4(in.position, 1.0);
    out.color = in.color;
    return out;
}

fragment float4 triangleFragmentShader(
    VertexOut in [[stage_in]]
) {
    return in.color;
}
```

---

## 86.7 Textures

### การโหลดและใช้ Texture

```objc
#import <MetalKit/MetalKit.h>

@implementation TextureRenderer {
    id<MTLTexture> _texture;
    id<MTLSamplerState> _sampler;
}

// โหลด texture จากไฟล์ภาพ
- (id<MTLTexture>)loadTextureFromFile:(NSString *)filename {
    MTKTextureLoader *loader = [[MTKTextureLoader alloc] initWithDevice:_device];
    
    NSURL *url = [[NSBundle mainBundle] URLForResource:filename 
                                        withExtension:@"png"];
    
    NSDictionary *options = @{
        MTKTextureLoaderOptionTextureUsage: @(MTLTextureUsageShaderRead),
        MTKTextureLoaderOptionTextureStorageMode: @(MTLStorageModePrivate),
        MTKTextureLoaderOptionSRGB: @YES,
        MTKTextureLoaderOptionGenerateMipmaps: @YES,
    };
    
    NSError *error;
    id<MTLTexture> texture = [loader newTextureWithContentsOfURL:url 
                                                         options:options 
                                                           error:&error];
    if (error) {
        NSLog(@"Texture load error: %@", error);
        return nil;
    }
    
    return texture;
}

// สร้าง texture แบบ programmatic
- (id<MTLTexture>)createTextureWithWidth:(NSUInteger)width 
                                  height:(NSUInteger)height {
    MTLTextureDescriptor *descriptor = [MTLTextureDescriptor 
        texture2DDescriptorWithPixelFormat:MTLPixelFormatRGBA8Unorm
                                     width:width
                                    height:height
                                 mipmapped:YES];
    
    descriptor.usage = MTLTextureUsageRenderTarget | MTLTextureUsageShaderRead;
    descriptor.storageMode = MTLStorageModePrivate;
    
    return [_device newTextureWithDescriptor:descriptor];
}

// สร้าง Sampler State
- (id<MTLSamplerState>)createSampler {
    MTLSamplerDescriptor *desc = [[MTLSamplerDescriptor alloc] init];
    
    // Filtering
    desc.minFilter = MTLSamplerMinMagFilterLinear;     // zoom out
    desc.magFilter = MTLSamplerMinMagFilterLinear;     // zoom in
    desc.mipFilter = MTLSamplerMipFilterLinear;        // mipmap
    
    // Addressing mode (สำหรับ UV นอกช่วง 0-1)
    desc.sAddressMode = MTLSamplerAddressModeRepeat;   // ซ้ำในแนวนอน
    desc.tAddressMode = MTLSamplerAddressModeRepeat;   // ซ้ำในแนวตั้ง
    
    // Anisotropic filtering
    desc.maxAnisotropy = 16;
    
    return [_device newSamplerStateWithDescriptor:desc];
}

// การ render ด้วย texture
- (void)renderWithTexture:(id<MTLRenderCommandEncoder>)encoder {
    [encoder setFragmentTexture:_texture atIndex:0];
    [encoder setFragmentSamplerState:_sampler atIndex:0];
    
    // Draw...
}

@end
```

### Texture สำหรับ Render Target

```objc
// การใช้ texture เป็น render target (render to texture)
- (void)renderToTexture {
    id<MTLTexture> renderTarget = [self createTextureWithWidth:1024 height:1024];
    
    MTLRenderPassDescriptor *renderPass = [[MTLRenderPassDescriptor alloc] init];
    renderPass.colorAttachments[0].texture = renderTarget;
    renderPass.colorAttachments[0].loadAction = MTLLoadActionClear;
    renderPass.colorAttachments[0].storeAction = MTLStoreActionStore;
    renderPass.colorAttachments[0].clearColor = MTLClearColorMake(0, 0, 0, 1);
    
    id<MTLCommandBuffer> commandBuffer = [_commandQueue commandBuffer];
    id<MTLRenderCommandEncoder> encoder = [commandBuffer 
        renderCommandEncoderWithDescriptor:renderPass];
    
    // render scene ลงใน texture...
    
    [encoder endEncoding];
    [commandBuffer commit];
    [commandBuffer waitUntilCompleted];
    
    // นำ renderTarget ไปใช้เป็น texture ต่อไป
}
```

---

## 86.8 Compute Shaders

### Parallel Computation ด้วย Compute

```objc
// การใช้ compute shader สำหรับ image processing
@implementation ComputeProcessor {
    id<MTLDevice> _device;
    id<MTLComputePipelineState> _computePipeline;
}

- (void)setupComputePipeline {
    id<MTLLibrary> library = [_device newDefaultLibrary];
    id<MTLFunction> kernelFunction = [library newFunctionWithName:@"imageProcessing"];
    
    NSError *error;
    _computePipeline = [_device newComputePipelineStateWithFunction:kernelFunction 
                                                              error:&error];
    
    if (error) {
        NSLog(@"Compute pipeline error: %@", error);
    }
    
    NSLog(@"Max threads per threadgroup: %lu", 
          _computePipeline.maxTotalThreadsPerThreadgroup);
}

- (void)processImage:(id<MTLTexture>)inputTexture 
            output:(id<MTLTexture>)outputTexture {
    
    id<MTLCommandBuffer> commandBuffer = [_commandQueue commandBuffer];
    
    // สร้าง compute encoder
    id<MTLComputeCommandEncoder> encoder = [commandBuffer computeCommandEncoder];
    encoder.label = @"Image Processing";
    
    // ตั้งค่า pipeline
    [encoder setComputePipelineState:_computePipeline];
    
    // ผูก textures
    [encoder setTexture:inputTexture atIndex:0];
    [encoder setTexture:outputTexture atIndex:1];
    
    // กำหนด thread configuration
    MTLSize threadsPerThreadgroup = MTLSizeMake(16, 16, 1);
    
    NSUInteger threadgroupsX = (outputTexture.width + 15) / 16;
    NSUInteger threadgroupsY = (outputTexture.height + 15) / 16;
    MTLSize threadgroupCount = MTLSizeMake(threadgroupsX, threadgroupsY, 1);
    
    // Dispatch compute
    [encoder dispatchThreadgroups:threadgroupCount 
            threadsPerThreadgroup:threadsPerThreadgroup];
    
    [encoder endEncoding];
    [commandBuffer commit];
    [commandBuffer waitUntilCompleted];
}

// การทำ vector computation
- (void)computeVectorAddition {
    const NSUInteger vectorSize = 1000000;
    
    // สร้าง buffers
    id<MTLBuffer> bufferA = [_device newBufferWithLength:vectorSize * sizeof(float) 
                                                 options:MTLResourceStorageModeShared];
    id<MTLBuffer> bufferB = [_device newBufferWithLength:vectorSize * sizeof(float) 
                                                 options:MTLResourceStorageModeShared];
    id<MTLBuffer> result = [_device newBufferWithLength:vectorSize * sizeof(float) 
                                                options:MTLResourceStorageModeShared];
    
    // เติมข้อมูล
    float *a = bufferA.contents;
    float *b = bufferB.contents;
    for (NSUInteger i = 0; i < vectorSize; i++) {
        a[i] = (float)i;
        b[i] = (float)(vectorSize - i);
    }
    
    // Dispatch compute
    id<MTLCommandBuffer> commandBuffer = [_commandQueue commandBuffer];
    id<MTLComputeCommandEncoder> encoder = [commandBuffer computeCommandEncoder];
    
    [encoder setComputePipelineState:_vectorAddPipeline];
    [encoder setBuffer:bufferA offset:0 atIndex:0];
    [encoder setBuffer:bufferB offset:0 atIndex:1];
    [encoder setBuffer:result offset:0 atIndex:2];
    
    MTLSize gridSize = MTLSizeMake(vectorSize, 1, 1);
    MTLSize threadgroupSize = MTLSizeMake(
        MIN(vectorSize, _vectorAddPipeline.maxTotalThreadsPerThreadgroup), 1, 1
    );
    
    [encoder dispatchThreads:gridSize threadsPerThreadgroup:threadgroupSize];
    [encoder endEncoding];
    [commandBuffer commit];
    [commandBuffer waitUntilCompleted];
    
    // อ่านผลลัพธ์
    float *resultData = result.contents;
    NSLog(@"Result[0] = %.0f", resultData[0]);  // ควรได้ vectorSize (= 1000000)
}

@end
```

---

## 86.9 Metal Performance Shaders (MPS)

### การใช้งาน MPS

```objc
#import <MetalPerformanceShaders/MetalPerformanceShaders.h>

@implementation MPSDemo {
    id<MTLDevice> _device;
    id<MTLCommandQueue> _commandQueue;
}

// การทำ Gaussian Blur ด้วย MPS
- (id<MTLTexture>)applyGaussianBlur:(id<MTLTexture>)inputTexture 
                             sigma:(float)sigma {
    
    // สร้าง output texture
    MTLTextureDescriptor *desc = [MTLTextureDescriptor 
        texture2DDescriptorWithPixelFormat:inputTexture.pixelFormat
                                     width:inputTexture.width
                                    height:inputTexture.height
                                 mipmapped:NO];
    desc.usage = MTLTextureUsageShaderRead | MTLTextureUsageShaderWrite;
    id<MTLTexture> outputTexture = [_device newTextureWithDescriptor:desc];
    
    // สร้าง MPS Gaussian Blur filter
    MPSImageGaussianBlur *blur = [[MPSImageGaussianBlur alloc] 
        initWithDevice:_device sigma:sigma];
    
    // Apply blur
    id<MTLCommandBuffer> commandBuffer = [_commandQueue commandBuffer];
    [blur encodeToCommandBuffer:commandBuffer
                  sourceTexture:inputTexture
             destinationTexture:outputTexture];
    [commandBuffer commit];
    [commandBuffer waitUntilCompleted];
    
    return outputTexture;
}

// Matrix Multiplication ด้วย MPS
- (void)matrixMultiplication {
    const NSUInteger rows = 1000;
    const NSUInteger cols = 1000;
    
    // สร้าง MPS Matrix
    MPSMatrixDescriptor *desc = [MPSMatrixDescriptor 
        matrixDescriptorWithRows:rows 
                         columns:cols
                        rowBytes:cols * sizeof(float)
                        dataType:MPSDataTypeFloat32];
    
    id<MTLBuffer> bufferA = [_device newBufferWithLength:rows * cols * sizeof(float) 
                                                 options:MTLResourceStorageModeShared];
    id<MTLBuffer> bufferB = [_device newBufferWithLength:rows * cols * sizeof(float) 
                                                 options:MTLResourceStorageModeShared];
    id<MTLBuffer> bufferC = [_device newBufferWithLength:rows * cols * sizeof(float) 
                                                 options:MTLResourceStorageModeShared];
    
    MPSMatrix *matrixA = [[MPSMatrix alloc] initWithBuffer:bufferA descriptor:desc];
    MPSMatrix *matrixB = [[MPSMatrix alloc] initWithBuffer:bufferB descriptor:desc];
    MPSMatrix *matrixC = [[MPSMatrix alloc] initWithBuffer:bufferC descriptor:desc];
    
    // Matrix multiplication: C = A * B
    MPSMatrixMultiplication *matMul = [[MPSMatrixMultiplication alloc] 
        initWithDevice:_device
        transposeLeft:NO
       transposeRight:NO
           resultRows:rows
        resultColumns:cols
      interiorColumns:cols
                alpha:1.0
                 beta:0.0];
    
    id<MTLCommandBuffer> commandBuffer = [_commandQueue commandBuffer];
    [matMul encodeToCommandBuffer:commandBuffer 
                      leftMatrix:matrixA
                     rightMatrix:matrixB
                    resultMatrix:matrixC];
    [commandBuffer commit];
    [commandBuffer waitUntilCompleted];
}

// Neural Network inference ด้วย MPS
- (void)runNeuralNetwork {
    MPSNNGraph *graph = [self buildNeuralNetwork];
    
    // สร้าง input image
    MPSImage *inputImage = [[MPSImage alloc] initWithDevice:_device 
                                           imageDescriptor:[MPSImageDescriptor 
                                               imageDescriptorWithChannelFormat:MPSImageFeatureChannelFormatFloat32
                                                                          width:224
                                                                         height:224
                                                                featureChannels:3]];
    
    // Run inference
    id<MTLCommandBuffer> commandBuffer = [_commandQueue commandBuffer];
    MPSImage *outputImage = [graph encodeToCommandBuffer:commandBuffer 
                                            sourceImages:@[inputImage]];
    [commandBuffer commit];
    [commandBuffer waitUntilCompleted];
}

@end
```

---

## 86.10 MetalKit (MTKView)

### การสร้าง Metal App ด้วย MTKView

```objc
// MetalViewController.m
#import <MetalKit/MetalKit.h>

@interface MetalViewController : NSViewController <MTKViewDelegate>

@end

@implementation MetalViewController {
    MTKView *_metalView;
    id<MTLDevice> _device;
    id<MTLCommandQueue> _commandQueue;
    id<MTLRenderPipelineState> _pipelineState;
    
    // Animation
    float _rotationAngle;
    CFTimeInterval _lastTime;
}

- (void)viewDidLoad {
    [super viewDidLoad];
    
    // สร้าง device
    _device = MTLCreateSystemDefaultDevice();
    
    // สร้างและตั้งค่า MTKView
    _metalView = [[MTKView alloc] initWithFrame:self.view.bounds device:_device];
    _metalView.autoresizingMask = NSViewWidthSizable | NSViewHeightSizable;
    _metalView.delegate = self;
    _metalView.colorPixelFormat = MTLPixelFormatBGRA8Unorm_sRGB;
    _metalView.depthStencilPixelFormat = MTLPixelFormatDepth32Float;
    _metalView.clearColor = MTLClearColorMake(0.05, 0.05, 0.1, 1.0);
    _metalView.preferredFramesPerSecond = 60;
    _metalView.enableSetNeedsDisplay = NO;  // continuous rendering
    
    [self.view addSubview:_metalView];
    
    // Setup
    _commandQueue = [_device newCommandQueue];
    [self setupPipeline];
}

// MTKViewDelegate - เรียกทุก frame
- (void)drawInMTKView:(MTKView *)view {
    // อัปเดต animation
    CFTimeInterval currentTime = CACurrentMediaTime();
    float deltaTime = (float)(currentTime - _lastTime);
    _lastTime = currentTime;
    _rotationAngle += deltaTime * 1.0;  // 1 radian ต่อวินาที
    
    // Render
    id<MTLCommandBuffer> commandBuffer = [_commandQueue commandBuffer];
    MTLRenderPassDescriptor *rpd = view.currentRenderPassDescriptor;
    
    if (rpd) {
        id<MTLRenderCommandEncoder> encoder = [commandBuffer 
            renderCommandEncoderWithDescriptor:rpd];
        
        [encoder setRenderPipelineState:_pipelineState];
        [self drawScene:encoder];
        [encoder endEncoding];
        
        [commandBuffer presentDrawable:view.currentDrawable];
    }
    
    [commandBuffer commit];
}

// MTKViewDelegate - เรียกเมื่อ view ถูก resize
- (void)mtkView:(MTKView *)view drawableSizeWillChange:(CGSize)size {
    NSLog(@"Drawable size changed to: %.0f x %.0f", size.width, size.height);
    
    // อัปเดต projection matrix ตาม aspect ratio ใหม่
    float aspect = (float)size.width / (float)size.height;
    [self updateProjectionMatrix:aspect];
}

- (void)drawScene:(id<MTLRenderCommandEncoder>)encoder {
    // Uniforms สำหรับ transformation
    typedef struct {
        matrix_float4x4 modelMatrix;
        matrix_float4x4 viewMatrix;
        matrix_float4x4 projectionMatrix;
    } Uniforms;
    
    Uniforms uniforms;
    uniforms.modelMatrix = matrix_rotation(_rotationAngle, (vector_float3){0, 1, 0});
    uniforms.viewMatrix = matrix_look_at((vector_float3){0, 0, -3},
                                          (vector_float3){0, 0, 0},
                                          (vector_float3){0, 1, 0});
    uniforms.projectionMatrix = _projectionMatrix;
    
    [encoder setVertexBytes:&uniforms 
                     length:sizeof(uniforms) 
                    atIndex:1];
    
    // Draw
    [encoder setVertexBuffer:_vertexBuffer offset:0 atIndex:0];
    [encoder drawIndexedPrimitives:MTLPrimitiveTypeTriangle
                        indexCount:_indexCount
                         indexType:MTLIndexTypeUInt16
                       indexBuffer:_indexBuffer
                 indexBufferOffset:0];
}

@end
```

---

## 86.11 เปรียบเทียบ Metal กับ OpenGL ES

### ความแตกต่างหลัก

```
Feature              | OpenGL ES              | Metal
---------------------|------------------------|---------------------------
State Management     | Global state machine   | Explicit state objects
Shader Compilation   | Runtime compilation    | Offline compilation
Command Submission   | Immediate              | Explicit command buffers
Resource Management  | Driver manages         | App manages explicitly
Memory               | Driver allocates       | App controls allocation
Multi-threading      | One context per thread | Full multi-thread support
Validation           | Always validates       | Debug layer optional
Performance          | CPU bottleneck         | Lower CPU overhead
```

### การ Migrate จาก OpenGL ES

```objc
// OpenGL ES approach (เก่า)
// glGenTextures(1, &textureID);
// glBindTexture(GL_TEXTURE_2D, textureID);
// glTexImage2D(GL_TEXTURE_2D, 0, GL_RGBA, width, height, 0, GL_RGBA, GL_UNSIGNED_BYTE, data);

// Metal approach (ใหม่)
- (id<MTLTexture>)createTextureFromData:(const void *)data 
                                  width:(NSUInteger)width 
                                 height:(NSUInteger)height {
    
    MTLTextureDescriptor *desc = [MTLTextureDescriptor 
        texture2DDescriptorWithPixelFormat:MTLPixelFormatRGBA8Unorm
                                     width:width
                                    height:height
                                 mipmapped:NO];
    
    id<MTLTexture> texture = [_device newTextureWithDescriptor:desc];
    
    MTLRegion region = MTLRegionMake2D(0, 0, width, height);
    [texture replaceRegion:region
               mipmapLevel:0
                 withBytes:data
               bytesPerRow:width * 4];  // 4 bytes per pixel (RGBA)
    
    return texture;
}
```

---

## 86.12 Advanced Metal Techniques

### Indirect Command Buffers

```objc
// ICB ช่วยให้ GPU สร้าง commands ได้เอง โดยไม่ต้องผ่าน CPU
- (void)setupIndirectCommandBuffer {
    MTLIndirectCommandBufferDescriptor *icbDesc = 
        [[MTLIndirectCommandBufferDescriptor alloc] init];
    icbDesc.commandTypes = MTLIndirectCommandTypeDraw;
    icbDesc.inheritBuffers = NO;
    icbDesc.maxVertexBufferBindCount = 4;
    icbDesc.maxFragmentBufferBindCount = 4;
    
    _icb = [_device newIndirectCommandBufferWithDescriptor:icbDesc 
                                           maxCommandCount:1000 
                                                   options:0];
}
```

### Resource Heaps

```objc
// ใช้ Heap สำหรับ sub-allocate resources
- (void)setupHeap {
    // ประเมินขนาดที่ต้องการ
    MTLTextureDescriptor *texDesc = [MTLTextureDescriptor 
        texture2DDescriptorWithPixelFormat:MTLPixelFormatRGBA8Unorm
                                     width:1024
                                    height:1024
                                 mipmapped:NO];
    MTLSizeAndAlign texSizeAlign = [_device heapTextureSizeAndAlignWithDescriptor:texDesc];
    
    // สร้าง heap
    MTLHeapDescriptor *heapDesc = [[MTLHeapDescriptor alloc] init];
    heapDesc.size = texSizeAlign.size * 10;  // สำหรับ 10 textures
    heapDesc.storageMode = MTLStorageModePrivate;
    heapDesc.type = MTLHeapTypeAutomatic;
    
    id<MTLHeap> heap = [_device newHeapWithDescriptor:heapDesc];
    
    // สร้าง texture จาก heap
    id<MTLTexture> texture = [heap newTextureWithDescriptor:texDesc];
}
```

### Event Synchronization

```objc
// Synchronize ระหว่าง compute และ render passes
- (void)synchronizeWithEvents {
    id<MTLEvent> event = [_device newEvent];
    
    id<MTLCommandBuffer> computeCommandBuffer = [_commandQueue commandBuffer];
    
    id<MTLComputeCommandEncoder> computeEncoder = 
        [computeCommandBuffer computeCommandEncoder];
    // compute work...
    [computeEncoder endEncoding];
    
    // Signal event หลังจาก compute เสร็จ
    [computeCommandBuffer encodeSignalEvent:event value:1];
    [computeCommandBuffer commit];
    
    id<MTLCommandBuffer> renderCommandBuffer = [_commandQueue commandBuffer];
    
    // รอ event ก่อน render
    [renderCommandBuffer encodeWaitForEvent:event value:1];
    
    id<MTLRenderCommandEncoder> renderEncoder = [renderCommandBuffer 
        renderCommandEncoderWithDescriptor:_renderPassDescriptor];
    // render work...
    [renderEncoder endEncoding];
    
    [renderCommandBuffer commit];
}
```

---

## 86.13 แบบฝึกหัด

### แบบฝึกหัดที่ 1: สร้าง Rotating Cube

```objc
// สร้างโปรแกรมที่แสดง cube หมุนรอบแกน Y
// ต้องการ:
// 1. Vertex buffer สำหรับ 8 จุดของ cube
// 2. Index buffer สำหรับ 12 triangles (36 indices)
// 3. Uniform buffer สำหรับ MVP matrix
// 4. Depth testing เพื่อให้หน้าที่อยู่หน้าบังหน้าที่อยู่หลัง

// Cube vertices
typedef struct {
    vector_float3 position;
    vector_float4 color;
} CubeVertex;

static const CubeVertex cubeVertices[] = {
    // Front face
    {{-0.5, -0.5,  0.5}, {1.0, 0.0, 0.0, 1.0}},  // 0
    {{ 0.5, -0.5,  0.5}, {0.0, 1.0, 0.0, 1.0}},  // 1
    {{ 0.5,  0.5,  0.5}, {0.0, 0.0, 1.0, 1.0}},  // 2
    {{-0.5,  0.5,  0.5}, {1.0, 1.0, 0.0, 1.0}},  // 3
    // Back face
    {{-0.5, -0.5, -0.5}, {0.0, 1.0, 1.0, 1.0}},  // 4
    {{ 0.5, -0.5, -0.5}, {1.0, 0.0, 1.0, 1.0}},  // 5
    {{ 0.5,  0.5, -0.5}, {0.5, 0.5, 0.5, 1.0}},  // 6
    {{-0.5,  0.5, -0.5}, {1.0, 0.5, 0.0, 1.0}},  // 7
};

static const uint16_t cubeIndices[] = {
    0, 1, 2, 0, 2, 3,  // Front
    1, 5, 6, 1, 6, 2,  // Right
    5, 4, 7, 5, 7, 6,  // Back
    4, 0, 3, 4, 3, 7,  // Left
    3, 2, 6, 3, 6, 7,  // Top
    4, 5, 1, 4, 1, 0,  // Bottom
};
```

### แบบฝึกหัดที่ 2: Image Filter

```
สร้าง compute shader ที่ทำ image processing:
1. Invert colors: ทุก pixel RGB = 1.0 - originalRGB
2. Edge detection: ใช้ Sobel operator
3. Pixelate effect: group pixels เป็น blocks ขนาด N
4. เพิ่ม UI ให้ผู้ใช้เลือก filter ได้
```

```metal
// Edge detection kernel (Sobel)
kernel void sobelEdgeDetection(
    texture2d<float, access::sample> input [[texture(0)]],
    texture2d<float, access::write> output [[texture(1)]],
    uint2 gid [[thread_position_in_grid]]
) {
    constexpr sampler s(filter::linear, address::clamp_to_edge);
    
    float2 texelSize = float2(1.0 / input.get_width(), 1.0 / input.get_height());
    float2 uv = (float2(gid) + 0.5) * texelSize;
    
    // Sample surrounding pixels
    float tl = input.sample(s, uv + texelSize * float2(-1, -1)).r;
    float tc = input.sample(s, uv + texelSize * float2( 0, -1)).r;
    float tr = input.sample(s, uv + texelSize * float2( 1, -1)).r;
    float ml = input.sample(s, uv + texelSize * float2(-1,  0)).r;
    float mr = input.sample(s, uv + texelSize * float2( 1,  0)).r;
    float bl = input.sample(s, uv + texelSize * float2(-1,  1)).r;
    float bc = input.sample(s, uv + texelSize * float2( 0,  1)).r;
    float br = input.sample(s, uv + texelSize * float2( 1,  1)).r;
    
    // Sobel operator
    float sobelX = tl + 2.0 * ml + bl - tr - 2.0 * mr - br;
    float sobelY = tl + 2.0 * tc + tr - bl - 2.0 * bc - br;
    
    float edge = sqrt(sobelX * sobelX + sobelY * sobelY);
    
    output.write(float4(edge, edge, edge, 1.0), gid);
}
```

### แบบฝึกหัดที่ 3: Particle System

```
สร้าง particle system ด้วย compute shader:
1. Particles buffer: position, velocity, lifetime
2. Compute shader: อัปเดต position ตาม velocity และ gravity
3. Vertex shader: แปลง particle position เป็น quad
4. Fragment shader: render particle เป็น circle พร้อม gradient
5. Reset particles ที่หมด lifetime
```

---

## สรุป

ในบทนี้เราได้เรียนรู้:

- **Metal Overview**: สถาปัตยกรรมและแนวคิดของ GPU programming
- **MTLDevice**: การเข้าถึงและ query GPU capabilities
- **CommandQueue/Buffer**: การส่ง commands ไปยัง GPU
- **MSL**: Metal Shading Language สำหรับเขียน shaders
- **Pipeline State**: การตั้งค่า render pipeline
- **Hello Triangle**: การวาดรูปทรงแรกด้วย Metal
- **Textures**: การโหลดและ sample textures
- **Compute Shaders**: การทำ parallel computation
- **MPS**: Metal Performance Shaders ที่ปรับแต่งสำหรับงานเฉพาะ
- **MTKView**: ตัวช่วยในการ integrate Metal กับ view system

บทต่อไปจะเรียนรู้เกี่ยวกับ **ARKit** สำหรับการพัฒนา Augmented Reality
