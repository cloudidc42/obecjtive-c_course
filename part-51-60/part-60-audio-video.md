# ส่วนที่ 60: Audio และ Video ด้วย AVFoundation ใน Objective-C

## บทนำ

AVFoundation เป็น framework หลักของ Apple สำหรับการทำงานกับสื่อมัลติมีเดีย ครอบคลุมทั้งการเล่น การบันทึก และการประมวลผลทั้ง audio และ video บทนี้จะครอบคลุมการใช้งานจากพื้นฐานจนถึงขั้นสูง

---

## 1. AVFoundation Overview

### 1.1 Import Frameworks

```objc
#import <AVFoundation/AVFoundation.h>
#import <AVKit/AVKit.h>
```

### 1.2 โครงสร้างหลักของ AVFoundation

```
AVFoundation
├── Audio
│   ├── AVAudioPlayer      - เล่นไฟล์เสียง
│   ├── AVAudioRecorder    - บันทึกเสียง
│   ├── AVAudioEngine      - ประมวลผลเสียงขั้นสูง
│   └── AVAudioSession     - จัดการ audio session
├── Video
│   ├── AVPlayer           - เล่น video/audio
│   ├── AVPlayerViewController - UI สำหรับเล่น video
│   ├── AVCaptureSession   - จับภาพ/วิดีโอจากกล้อง
│   └── AVAsset            - จัดการ media assets
└── Synthesis
    ├── AVSpeechSynthesizer - Text-to-Speech
    └── AVSpeechRecognizer  - Speech-to-Text
```

### 1.3 AVAudioSession Configuration

```objc
- (void)configureAudioSession {
    AVAudioSession *session = [AVAudioSession sharedInstance];
    NSError *error = nil;
    
    // กำหนด category
    [session setCategory:AVAudioSessionCategoryPlayback 
             withOptions:AVAudioSessionCategoryOptionMixWithOthers
                   error:&error];
    
    if (error) {
        NSLog(@"Audio session error: %@", error.localizedDescription);
        return;
    }
    
    // เปิดใช้งาน session
    [session setActive:YES error:&error];
    
    NSLog(@"Sample rate: %f", session.sampleRate);
    NSLog(@"Output channels: %ld", (long)session.outputNumberOfChannels);
    
    // ฟัง interruptions (เช่น incoming call)
    [[NSNotificationCenter defaultCenter] 
        addObserver:self 
           selector:@selector(handleAudioInterruption:)
               name:AVAudioSessionInterruptionNotification 
             object:session];
}

- (void)handleAudioInterruption:(NSNotification *)notification {
    NSDictionary *info = notification.userInfo;
    AVAudioSessionInterruptionType type = 
        [info[AVAudioSessionInterruptionTypeKey] unsignedIntegerValue];
    
    if (type == AVAudioSessionInterruptionTypeBegan) {
        [self.audioPlayer pause];
        NSLog(@"Audio interrupted - paused");
    } else if (type == AVAudioSessionInterruptionTypeEnded) {
        AVAudioSessionInterruptionOptions options = 
            [info[AVAudioSessionInterruptionOptionKey] unsignedIntegerValue];
        
        if (options & AVAudioSessionInterruptionOptionShouldResume) {
            [self.audioPlayer play];
            NSLog(@"Audio resumed");
        }
    }
}
```

---

## 2. AVAudioPlayer

### 2.1 เล่นไฟล์เสียงพื้นฐาน

```objc
@interface AudioPlayerViewController : UIViewController <AVAudioPlayerDelegate>
@property (nonatomic, strong) AVAudioPlayer *audioPlayer;
@end

@implementation AudioPlayerViewController

- (void)setupPlayer {
    // โหลดไฟล์จาก bundle
    NSURL *audioURL = [[NSBundle mainBundle] URLForResource:@"music" withExtension:@"mp3"];
    
    if (!audioURL) {
        NSLog(@"ไม่พบไฟล์เสียง");
        return;
    }
    
    NSError *error = nil;
    self.audioPlayer = [[AVAudioPlayer alloc] initWithContentsOfURL:audioURL error:&error];
    
    if (error) {
        NSLog(@"สร้าง player ล้มเหลว: %@", error.localizedDescription);
        return;
    }
    
    self.audioPlayer.delegate = self;
    self.audioPlayer.volume = 1.0;      // 0.0 - 1.0
    self.audioPlayer.numberOfLoops = 0; // 0 = เล่นครั้งเดียว, -1 = loop ตลอด
    self.audioPlayer.rate = 1.0;        // ความเร็ว 0.5 - 2.0
    
    // เตรียม buffer ล่วงหน้า
    [self.audioPlayer prepareToPlay];
    
    NSLog(@"ระยะเวลา: %.1f วินาที", self.audioPlayer.duration);
}

- (void)playAudio {
    [self.audioPlayer play];
}

- (void)pauseAudio {
    [self.audioPlayer pause];
}

- (void)stopAudio {
    [self.audioPlayer stop];
    self.audioPlayer.currentTime = 0; // กลับต้น
}

// Seek ไปยังตำแหน่งที่ต้องการ
- (void)seekToTime:(NSTimeInterval)time {
    self.audioPlayer.currentTime = time;
}

#pragma mark - AVAudioPlayerDelegate

- (void)audioPlayerDidFinishPlaying:(AVAudioPlayer *)player successfully:(BOOL)flag {
    NSLog(@"เล่นเสร็จแล้ว success: %@", flag ? @"YES" : @"NO");
    
    // เล่นเพลงถัดไป
    [self playNextSong];
}

- (void)audioPlayerDecodeErrorDidOccur:(AVAudioPlayer *)player error:(NSError *)error {
    NSLog(@"Decode error: %@", error.localizedDescription);
}

@end
```

### 2.2 Player ที่ครบฟีเจอร์

```objc
@interface MusicPlayer : NSObject <AVAudioPlayerDelegate>

@property (nonatomic, strong) AVAudioPlayer *player;
@property (nonatomic, strong) NSArray<NSURL *> *playlist;
@property (nonatomic, assign) NSInteger currentIndex;
@property (nonatomic, assign) BOOL shuffle;
@property (nonatomic, assign) BOOL repeatAll;

@property (nonatomic, copy) void(^onProgressUpdate)(NSTimeInterval current, NSTimeInterval total);
@property (nonatomic, copy) void(^onSongChanged)(NSInteger index);

@end

@implementation MusicPlayer {
    NSTimer *_progressTimer;
}

- (instancetype)initWithPlaylist:(NSArray<NSURL *> *)playlist {
    self = [super init];
    if (self) {
        _playlist = [playlist copy];
        _currentIndex = 0;
        _shuffle = NO;
        _repeatAll = YES;
    }
    return self;
}

- (void)playAtIndex:(NSInteger)index {
    if (index < 0 || index >= self.playlist.count) return;
    
    self.currentIndex = index;
    NSURL *url = self.playlist[index];
    
    NSError *error = nil;
    self.player = [[AVAudioPlayer alloc] initWithContentsOfURL:url error:&error];
    
    if (error) {
        NSLog(@"เกิดข้อผิดพลาด: %@", error.localizedDescription);
        return;
    }
    
    self.player.delegate = self;
    [self.player prepareToPlay];
    [self.player play];
    
    [self startProgressTimer];
    
    if (self.onSongChanged) {
        self.onSongChanged(index);
    }
}

- (void)next {
    NSInteger nextIndex;
    
    if (self.shuffle) {
        nextIndex = arc4random_uniform((uint32_t)self.playlist.count);
    } else {
        nextIndex = self.currentIndex + 1;
        if (nextIndex >= self.playlist.count) {
            nextIndex = self.repeatAll ? 0 : self.currentIndex;
        }
    }
    
    [self playAtIndex:nextIndex];
}

- (void)previous {
    // ถ้าเล่นมาแล้วเกิน 3 วิ กลับต้นเพลง
    if (self.player.currentTime > 3.0) {
        self.player.currentTime = 0;
    } else {
        NSInteger prevIndex = self.currentIndex - 1;
        if (prevIndex < 0) {
            prevIndex = self.repeatAll ? self.playlist.count - 1 : 0;
        }
        [self playAtIndex:prevIndex];
    }
}

- (void)setVolume:(CGFloat)volume {
    self.player.volume = volume;
}

- (void)startProgressTimer {
    [_progressTimer invalidate];
    _progressTimer = [NSTimer scheduledTimerWithTimeInterval:0.1
                                                      target:self
                                                    selector:@selector(updateProgress)
                                                    userInfo:nil
                                                     repeats:YES];
}

- (void)updateProgress {
    if (self.onProgressUpdate && self.player) {
        self.onProgressUpdate(self.player.currentTime, self.player.duration);
    }
}

- (void)audioPlayerDidFinishPlaying:(AVAudioPlayer *)player successfully:(BOOL)flag {
    if (flag) [self next];
}

- (void)dealloc {
    [_progressTimer invalidate];
}

@end
```

### 2.3 Audio Metering (VU Meter)

```objc
- (void)enableMetering {
    self.audioPlayer.meteringEnabled = YES;
}

- (void)updateMeter {
    [self.audioPlayer updateMeters];
    
    for (NSInteger channel = 0; channel < self.audioPlayer.numberOfChannels; channel++) {
        float averagePower = [self.audioPlayer averagePowerForChannel:channel];
        float peakPower = [self.audioPlayer peakPowerForChannel:channel];
        
        // แปลง dB ให้เป็น linear scale (0.0 - 1.0)
        // averagePower อยู่ในช่วง -160 ถึง 0 dB
        float level = pow(10.0, averagePower / 20.0);
        
        NSLog(@"Channel %ld: avg=%.1f dB, peak=%.1f dB, level=%.2f",
              (long)channel, averagePower, peakPower, level);
    }
}
```

---

## 3. AVAudioRecorder

### 3.1 ตั้งค่าและบันทึกเสียง

ต้องเพิ่ม `NSMicrophoneUsageDescription` ใน Info.plist

```objc
@interface AudioRecorderViewController : UIViewController <AVAudioRecorderDelegate>
@property (nonatomic, strong) AVAudioRecorder *recorder;
@property (nonatomic, strong) NSURL *recordingURL;
@end

@implementation AudioRecorderViewController

- (void)requestMicrophonePermission {
    [[AVAudioSession sharedInstance] requestRecordPermission:^(BOOL granted) {
        dispatch_async(dispatch_get_main_queue(), ^{
            if (granted) {
                NSLog(@"ได้รับสิทธิ์ใช้ไมโครโฟน");
                [self setupRecorder];
            } else {
                NSLog(@"ผู้ใช้ปฏิเสธสิทธิ์ไมโครโฟน");
            }
        });
    }];
}

- (void)setupRecorder {
    // กำหนด URL สำหรับบันทึก
    NSString *docPath = [NSSearchPathForDirectoriesInDomains(
        NSDocumentDirectory, NSUserDomainMask, YES) firstObject];
    
    NSString *filename = [NSString stringWithFormat:@"recording_%@.m4a",
                          [[NSDate date] description]];
    self.recordingURL = [NSURL fileURLWithPath:[docPath stringByAppendingPathComponent:filename]];
    
    // ตั้งค่า format
    NSDictionary *settings = @{
        AVFormatIDKey: @(kAudioFormatMPEG4AAC),
        AVSampleRateKey: @44100.0,
        AVNumberOfChannelsKey: @1,         // Mono
        AVEncoderAudioQualityKey: @(AVAudioQualityHigh),
        AVEncoderBitRateKey: @128000
    };
    
    // ตั้งค่า session สำหรับ record
    [[AVAudioSession sharedInstance] setCategory:AVAudioSessionCategoryRecord error:nil];
    [[AVAudioSession sharedInstance] setActive:YES error:nil];
    
    NSError *error = nil;
    self.recorder = [[AVAudioRecorder alloc] initWithURL:self.recordingURL
                                                settings:settings
                                                   error:&error];
    
    if (error) {
        NSLog(@"ไม่สามารถสร้าง recorder: %@", error.localizedDescription);
        return;
    }
    
    self.recorder.delegate = self;
    self.recorder.meteringEnabled = YES;
    [self.recorder prepareToRecord];
}

- (void)startRecording {
    if ([self.recorder isRecording]) return;
    [self.recorder record];
    NSLog(@"เริ่มบันทึก...");
}

- (void)stopRecording {
    if (![self.recorder isRecording]) return;
    [self.recorder stop];
}

- (void)recordForDuration:(NSTimeInterval)duration {
    [self.recorder recordForDuration:duration];
}

- (void)pauseRecording {
    [self.recorder pause];
}

// ดูความดังขณะบันทึก
- (float)currentRecordingLevel {
    [self.recorder updateMeters];
    return [self.recorder averagePowerForChannel:0];
}

#pragma mark - AVAudioRecorderDelegate

- (void)audioRecorderDidFinishRecording:(AVAudioRecorder *)recorder 
                            successfully:(BOOL)flag {
    if (flag) {
        NSLog(@"บันทึกสำเร็จที่: %@", recorder.url.path);
        // เปิดใช้ playback mode
        [[AVAudioSession sharedInstance] 
            setCategory:AVAudioSessionCategoryPlayback error:nil];
    } else {
        NSLog(@"การบันทึกล้มเหลว");
    }
}

- (void)audioRecorderEncodeErrorDidOccur:(AVAudioRecorder *)recorder 
                                    error:(NSError *)error {
    NSLog(@"Encode error: %@", error.localizedDescription);
}

@end
```

### 3.2 Voice Memo App

```objc
// VoiceMemo model
@interface VoiceMemo : NSObject
@property (nonatomic, strong) NSURL *fileURL;
@property (nonatomic, strong) NSDate *recordDate;
@property (nonatomic, assign) NSTimeInterval duration;
@property (nonatomic, strong) NSString *title;
@end

@implementation VoiceMemo
@end

// VoiceMemoManager
@interface VoiceMemoManager : NSObject

@property (nonatomic, strong) NSMutableArray<VoiceMemo *> *memos;
@property (nonatomic, strong) AVAudioRecorder *recorder;
@property (nonatomic, strong) AVAudioPlayer *player;

- (void)startRecordingWithTitle:(NSString *)title;
- (VoiceMemo *)stopRecording;
- (void)playMemo:(VoiceMemo *)memo;
- (void)deleteMemo:(VoiceMemo *)memo;
- (void)loadSavedMemos;

@end

@implementation VoiceMemoManager

- (instancetype)init {
    self = [super init];
    if (self) {
        _memos = [NSMutableArray array];
        [self loadSavedMemos];
    }
    return self;
}

- (void)startRecordingWithTitle:(NSString *)title {
    NSString *docPath = [NSSearchPathForDirectoriesInDomains(
        NSDocumentDirectory, NSUserDomainMask, YES) firstObject];
    NSString *fileName = [NSString stringWithFormat:@"%@.m4a", 
                          [[NSUUID UUID] UUIDString]];
    NSURL *url = [NSURL fileURLWithPath:[docPath stringByAppendingPathComponent:fileName]];
    
    NSDictionary *settings = @{
        AVFormatIDKey: @(kAudioFormatMPEG4AAC),
        AVSampleRateKey: @44100.0,
        AVNumberOfChannelsKey: @1,
        AVEncoderAudioQualityKey: @(AVAudioQualityMedium)
    };
    
    [[AVAudioSession sharedInstance] setCategory:AVAudioSessionCategoryRecord error:nil];
    [[AVAudioSession sharedInstance] setActive:YES error:nil];
    
    self.recorder = [[AVAudioRecorder alloc] initWithURL:url settings:settings error:nil];
    [self.recorder prepareToRecord];
    [self.recorder record];
    
    NSLog(@"บันทึก: %@", title);
}

- (VoiceMemo *)stopRecording {
    if (!self.recorder.recording) return nil;
    
    NSURL *url = self.recorder.url;
    NSTimeInterval duration = self.recorder.currentTime;
    
    [self.recorder stop];
    
    VoiceMemo *memo = [[VoiceMemo alloc] init];
    memo.fileURL = url;
    memo.recordDate = [NSDate date];
    memo.duration = duration;
    memo.title = [NSString stringWithFormat:@"บันทึก %lu", 
                  (unsigned long)(self.memos.count + 1)];
    
    [self.memos addObject:memo];
    [self saveMemos];
    
    return memo;
}

- (void)playMemo:(VoiceMemo *)memo {
    [[AVAudioSession sharedInstance] setCategory:AVAudioSessionCategoryPlayback error:nil];
    [[AVAudioSession sharedInstance] setActive:YES error:nil];
    
    self.player = [[AVAudioPlayer alloc] initWithContentsOfURL:memo.fileURL error:nil];
    [self.player play];
}

- (void)deleteMemo:(VoiceMemo *)memo {
    [[NSFileManager defaultManager] removeItemAtURL:memo.fileURL error:nil];
    [self.memos removeObject:memo];
    [self saveMemos];
}

- (void)saveMemos {
    // บันทึก metadata (ไม่รวม file data)
    NSMutableArray *savedData = [NSMutableArray array];
    for (VoiceMemo *memo in self.memos) {
        [savedData addObject:@{
            @"path": memo.fileURL.path,
            @"date": memo.recordDate,
            @"duration": @(memo.duration),
            @"title": memo.title
        }];
    }
    
    NSUserDefaults *defaults = [NSUserDefaults standardUserDefaults];
    [defaults setObject:savedData forKey:@"voiceMemos"];
    [defaults synchronize];
}

- (void)loadSavedMemos {
    NSArray *savedData = [[NSUserDefaults standardUserDefaults] objectForKey:@"voiceMemos"];
    
    for (NSDictionary *data in savedData) {
        NSString *path = data[@"path"];
        if ([[NSFileManager defaultManager] fileExistsAtPath:path]) {
            VoiceMemo *memo = [[VoiceMemo alloc] init];
            memo.fileURL = [NSURL fileURLWithPath:path];
            memo.recordDate = data[@"date"];
            memo.duration = [data[@"duration"] doubleValue];
            memo.title = data[@"title"];
            [self.memos addObject:memo];
        }
    }
}

@end
```

---

## 4. Playing Video (AVPlayer, AVPlayerViewController)

### 4.1 AVPlayerViewController - วิธีง่ายที่สุด

```objc
#import <AVKit/AVKit.h>

- (void)playVideoWithURL:(NSURL *)url {
    AVPlayer *player = [AVPlayer playerWithURL:url];
    AVPlayerViewController *playerVC = [[AVPlayerViewController alloc] init];
    playerVC.player = player;
    
    // ตัวเลือก UI
    playerVC.showsPlaybackControls = YES;  // แสดงปุ่ม control
    playerVC.videoGravity = AVLayerVideoGravityResizeAspect;
    
    [self presentViewController:playerVC animated:YES completion:^{
        [player play];
    }];
}

// เล่นวิดีโอจาก bundle
- (void)playLocalVideo {
    NSURL *url = [[NSBundle mainBundle] URLForResource:@"intro" withExtension:@"mp4"];
    if (!url) {
        NSLog(@"ไม่พบไฟล์วิดีโอ");
        return;
    }
    [self playVideoWithURL:url];
}

// เล่นวิดีโอจาก Internet
- (void)playRemoteVideo {
    NSURL *url = [NSURL URLWithString:@"https://example.com/video.mp4"];
    [self playVideoWithURL:url];
}
```

### 4.2 AVPlayer พร้อม Custom UI

```objc
@interface VideoPlayerViewController : UIViewController

@property (nonatomic, strong) AVPlayer *player;
@property (nonatomic, strong) AVPlayerLayer *playerLayer;

// Custom UI
@property (nonatomic, strong) UISlider *progressSlider;
@property (nonatomic, strong) UIButton *playPauseButton;
@property (nonatomic, strong) UILabel *timeLabel;
@property (nonatomic, strong) id timeObserver;

@end

@implementation VideoPlayerViewController

- (void)viewDidLoad {
    [super viewDidLoad];
    [self setupPlayer];
    [self setupUI];
}

- (void)setupPlayer {
    NSURL *url = [NSURL URLWithString:@"https://example.com/video.mp4"];
    
    AVAsset *asset = [AVAsset assetWithURL:url];
    AVPlayerItem *item = [AVPlayerItem playerItemWithAsset:asset];
    
    self.player = [AVPlayer playerWithPlayerItem:item];
    
    // เพิ่ม layer สำหรับแสดงวิดีโอ
    self.playerLayer = [AVPlayerLayer playerLayerWithPlayer:self.player];
    self.playerLayer.frame = CGRectMake(0, 0, self.view.bounds.size.width, 250);
    self.playerLayer.videoGravity = AVLayerVideoGravityResizeAspect;
    self.playerLayer.backgroundColor = [UIColor blackColor].CGColor;
    
    [self.view.layer addSublayer:self.playerLayer];
    
    // สังเกต status
    [item addObserver:self forKeyPath:@"status" options:NSKeyValueObservingOptionNew context:nil];
    [item addObserver:self forKeyPath:@"loadedTimeRanges" options:NSKeyValueObservingOptionNew context:nil];
    
    // ฟัง notification เมื่อเล่นจบ
    [[NSNotificationCenter defaultCenter] addObserver:self
                                             selector:@selector(playerItemDidFinish:)
                                                 name:AVPlayerItemDidPlayToEndTimeNotification
                                               object:item];
    
    // Update progress ทุก 0.1 วินาที
    __weak typeof(self) weakSelf = self;
    self.timeObserver = [self.player addPeriodicTimeObserverForInterval:CMTimeMake(1, 10)
                                                                 queue:dispatch_get_main_queue()
                                                            usingBlock:^(CMTime time) {
        [weakSelf updateProgress];
    }];
}

- (void)setupUI {
    // Play/Pause button
    self.playPauseButton = [UIButton buttonWithType:UIButtonTypeSystem];
    self.playPauseButton.frame = CGRectMake(20, 260, 44, 44);
    [self.playPauseButton setImage:[UIImage systemImageNamed:@"play.fill"] 
                          forState:UIControlStateNormal];
    [self.playPauseButton addTarget:self action:@selector(togglePlayPause) 
                   forControlEvents:UIControlEventTouchUpInside];
    [self.view addSubview:self.playPauseButton];
    
    // Progress slider
    self.progressSlider = [[UISlider alloc] initWithFrame:CGRectMake(80, 268, 200, 30)];
    [self.progressSlider addTarget:self action:@selector(sliderChanged:) 
                  forControlEvents:UIControlEventValueChanged];
    [self.view addSubview:self.progressSlider];
    
    // Time label
    self.timeLabel = [[UILabel alloc] initWithFrame:CGRectMake(290, 260, 80, 44)];
    self.timeLabel.text = @"0:00 / 0:00";
    self.timeLabel.font = [UIFont monospacedSystemFontOfSize:12 weight:UIFontWeightRegular];
    [self.view addSubview:self.timeLabel];
}

- (void)togglePlayPause {
    if (self.player.rate > 0) {
        [self.player pause];
        [self.playPauseButton setImage:[UIImage systemImageNamed:@"play.fill"] 
                              forState:UIControlStateNormal];
    } else {
        [self.player play];
        [self.playPauseButton setImage:[UIImage systemImageNamed:@"pause.fill"] 
                              forState:UIControlStateNormal];
    }
}

- (void)sliderChanged:(UISlider *)slider {
    CMTime duration = self.player.currentItem.duration;
    if (CMTIME_IS_INVALID(duration)) return;
    
    Float64 durationSeconds = CMTimeGetSeconds(duration);
    CMTime seekTime = CMTimeMakeWithSeconds(slider.value * durationSeconds, 600);
    
    [self.player seekToTime:seekTime toleranceBefore:kCMTimeZero toleranceAfter:kCMTimeZero];
}

- (void)updateProgress {
    CMTime current = self.player.currentTime;
    CMTime total = self.player.currentItem.duration;
    
    if (CMTIME_IS_INVALID(current) || CMTIME_IS_INVALID(total)) return;
    
    Float64 currentSec = CMTimeGetSeconds(current);
    Float64 totalSec = CMTimeGetSeconds(total);
    
    if (totalSec > 0) {
        self.progressSlider.value = currentSec / totalSec;
    }
    
    self.timeLabel.text = [NSString stringWithFormat:@"%@ / %@",
                           [self formatTime:currentSec],
                           [self formatTime:totalSec]];
}

- (NSString *)formatTime:(Float64)seconds {
    NSInteger mins = (NSInteger)seconds / 60;
    NSInteger secs = (NSInteger)seconds % 60;
    return [NSString stringWithFormat:@"%ld:%02ld", (long)mins, (long)secs];
}

- (void)playerItemDidFinish:(NSNotification *)notification {
    NSLog(@"วิดีโอเล่นจบแล้ว");
    [self.player seekToTime:kCMTimeZero];
    [self.playPauseButton setImage:[UIImage systemImageNamed:@"play.fill"] 
                          forState:UIControlStateNormal];
}

- (void)observeValueForKeyPath:(NSString *)keyPath 
                       ofObject:(id)object 
                         change:(NSDictionary *)change 
                        context:(void *)context {
    
    if ([keyPath isEqualToString:@"status"]) {
        AVPlayerItem *item = (AVPlayerItem *)object;
        
        switch (item.status) {
            case AVPlayerItemStatusReadyToPlay:
                NSLog(@"วิดีโอพร้อมเล่น");
                break;
            case AVPlayerItemStatusFailed:
                NSLog(@"เกิดข้อผิดพลาด: %@", item.error.localizedDescription);
                break;
            default:
                break;
        }
    } else if ([keyPath isEqualToString:@"loadedTimeRanges"]) {
        AVPlayerItem *item = (AVPlayerItem *)object;
        NSArray *ranges = item.loadedTimeRanges;
        
        if (ranges.count > 0) {
            CMTimeRange range = [ranges.firstObject CMTimeRangeValue];
            Float64 loaded = CMTimeGetSeconds(CMTimeRangeGetEnd(range));
            Float64 total = CMTimeGetSeconds(item.duration);
            
            NSLog(@"โหลดแล้ว %.0f%%", (loaded / total) * 100);
        }
    }
}

- (void)dealloc {
    [self.player removeTimeObserver:self.timeObserver];
    [self.player.currentItem removeObserver:self forKeyPath:@"status"];
    [self.player.currentItem removeObserver:self forKeyPath:@"loadedTimeRanges"];
    [[NSNotificationCenter defaultCenter] removeObserver:self];
}

@end
```

### 4.3 Picture-in-Picture (PiP)

```objc
#import <AVKit/AVKit.h>

@interface PiPViewController : UIViewController <AVPictureInPictureControllerDelegate>
@property (nonatomic, strong) AVPictureInPictureController *pipController;
@property (nonatomic, strong) AVPlayerLayer *playerLayer;
@end

@implementation PiPViewController

- (void)setupPiP {
    if (![AVPictureInPictureController isPictureInPictureSupported]) {
        NSLog(@"PiP ไม่รองรับบนอุปกรณ์นี้");
        return;
    }
    
    self.pipController = [[AVPictureInPictureController alloc] 
                           initWithPlayerLayer:self.playerLayer];
    self.pipController.delegate = self;
}

- (void)startPiP {
    [self.pipController startPictureInPicture];
}

- (void)stopPiP {
    [self.pipController stopPictureInPicture];
}

#pragma mark - AVPictureInPictureControllerDelegate

- (void)pictureInPictureControllerDidStartPictureInPicture:
        (AVPictureInPictureController *)controller {
    NSLog(@"เริ่ม PiP แล้ว");
}

- (void)pictureInPictureControllerDidStopPictureInPicture:
        (AVPictureInPictureController *)controller {
    NSLog(@"หยุด PiP แล้ว");
}

- (void)pictureInPictureController:(AVPictureInPictureController *)controller
    restoreUserInterfaceForPictureInPictureStopWithCompletionHandler:
        (void (^)(BOOL))completionHandler {
    
    // แสดง UI กลับมา
    [self.navigationController popToViewController:self animated:YES];
    completionHandler(YES);
}

@end
```

---

## 5. Video in Background

### 5.1 Background Audio/Video

ต้องเพิ่มใน Info.plist:
- `UIBackgroundModes` > `audio`

```objc
- (void)enableBackgroundPlayback {
    // ตั้ง audio session category ที่รองรับ background
    [[AVAudioSession sharedInstance] 
        setCategory:AVAudioSessionCategoryPlayback 
              error:nil];
    [[AVAudioSession sharedInstance] setActive:YES error:nil];
    
    // ตั้ง Now Playing info (แสดงบน Lock Screen)
    MPNowPlayingInfoCenter *center = [MPNowPlayingInfoCenter defaultCenter];
    center.nowPlayingInfo = @{
        MPMediaItemPropertyTitle: @"ชื่อเพลง",
        MPMediaItemPropertyArtist: @"ชื่อศิลปิน",
        MPMediaItemPropertyAlbumTitle: @"ชื่ออัลบั้ม",
        MPNowPlayingInfoPropertyElapsedPlaybackTime: @(self.player.currentTime),
        MPMediaItemPropertyPlaybackDuration: @(CMTimeGetSeconds(self.player.currentItem.duration)),
        MPNowPlayingInfoPropertyPlaybackRate: @(self.player.rate)
    };
    
    // ตั้ง artwork
    UIImage *artwork = [UIImage imageNamed:@"album_art"];
    if (artwork) {
        MPMediaItemArtwork *mpArtwork = [[MPMediaItemArtwork alloc] 
                                          initWithBoundsSize:artwork.size 
                                          requestHandler:^UIImage * _Nonnull(CGSize size) {
            return artwork;
        }];
        NSMutableDictionary *info = [center.nowPlayingInfo mutableCopy];
        info[MPMediaItemPropertyArtwork] = mpArtwork;
        center.nowPlayingInfo = info;
    }
}

// Remote Control Events (lock screen controls)
- (void)setupRemoteControls {
    MPRemoteCommandCenter *commandCenter = [MPRemoteCommandCenter sharedCommandCenter];
    
    // Play
    [commandCenter.playCommand addTargetWithHandler:^MPRemoteCommandHandlerStatus(MPRemoteCommandEvent *event) {
        [self.player play];
        return MPRemoteCommandHandlerStatusSuccess;
    }];
    
    // Pause
    [commandCenter.pauseCommand addTargetWithHandler:^MPRemoteCommandHandlerStatus(MPRemoteCommandEvent *event) {
        [self.player pause];
        return MPRemoteCommandHandlerStatusSuccess;
    }];
    
    // Next
    [commandCenter.nextTrackCommand addTargetWithHandler:^MPRemoteCommandHandlerStatus(MPRemoteCommandEvent *event) {
        // เล่นเพลงถัดไป
        return MPRemoteCommandHandlerStatusSuccess;
    }];
    
    // Previous
    [commandCenter.previousTrackCommand addTargetWithHandler:^MPRemoteCommandHandlerStatus(MPRemoteCommandEvent *event) {
        // เล่นเพลงก่อนหน้า
        return MPRemoteCommandHandlerStatusSuccess;
    }];
    
    // Seek (ลากแถบ progress บน lock screen)
    [commandCenter.changePlaybackPositionCommand addTargetWithHandler:
     ^MPRemoteCommandHandlerStatus(MPRemoteCommandEvent *event) {
        MPChangePlaybackPositionCommandEvent *posEvent = (MPChangePlaybackPositionCommandEvent *)event;
        CMTime seekTime = CMTimeMakeWithSeconds(posEvent.positionTime, 600);
        [self.player seekToTime:seekTime];
        return MPRemoteCommandHandlerStatusSuccess;
    }];
}
```

---

## 6. System Sounds (AudioServicesPlaySystemSound)

### 6.1 เล่น System Sounds

```objc
#import <AudioToolbox/AudioToolbox.h>

- (void)playSystemSounds {
    // เล่น standard system sound (ID 1000-1100+)
    AudioServicesPlaySystemSound(1007);   // เสียง lock
    AudioServicesPlaySystemSound(1013);   // เสียง SMS received
    AudioServicesPlaySystemSound(1052);   // เสียง camera shutter
    AudioServicesPlaySystemSound(1057);   // เสียง photo
    AudioServicesPlaySystemSound(kSystemSoundID_Vibrate); // สั่น
}

// เล่นไฟล์เสียงของตัวเอง
- (void)playCustomSoundEffect {
    NSURL *url = [[NSBundle mainBundle] URLForResource:@"click" withExtension:@"wav"];
    SystemSoundID soundID;
    
    AudioServicesCreateSystemSoundID((__bridge CFURLRef)url, &soundID);
    AudioServicesPlaySystemSound(soundID);
    
    // Cleanup หลังเล่นเสร็จ (ใช้ callback)
    AudioServicesAddSystemSoundCompletion(soundID, NULL, NULL, 
                                          systemSoundFinishedCallback, NULL);
}

void systemSoundFinishedCallback(SystemSoundID soundID, void *clientData) {
    AudioServicesRemoveSystemSoundCompletion(soundID);
    AudioServicesDisposeSystemSoundID(soundID);
}
```

---

## 7. Speech Synthesis (AVSpeechSynthesizer)

### 7.1 Text-to-Speech พื้นฐาน

```objc
#import <AVFoundation/AVFoundation.h>

@interface TTSViewController : UIViewController <AVSpeechSynthesizerDelegate>
@property (nonatomic, strong) AVSpeechSynthesizer *synthesizer;
@end

@implementation TTSViewController

- (void)setupSynthesizer {
    self.synthesizer = [[AVSpeechSynthesizer alloc] init];
    self.synthesizer.delegate = self;
}

- (void)speak:(NSString *)text {
    // หยุด utterance ที่กำลังพูดอยู่
    if (self.synthesizer.isSpeaking) {
        [self.synthesizer stopSpeakingAtBoundary:AVSpeechBoundaryImmediate];
    }
    
    AVSpeechUtterance *utterance = [[AVSpeechUtterance alloc] initWithString:text];
    
    // เลือกภาษา
    utterance.voice = [AVSpeechSynthesisVoice voiceWithLanguage:@"th-TH"]; // ภาษาไทย
    // utterance.voice = [AVSpeechSynthesisVoice voiceWithLanguage:@"en-US"]; // ภาษาอังกฤษ
    
    // ตั้งค่าต่างๆ
    utterance.rate = AVSpeechUtteranceDefaultSpeechRate;      // ความเร็ว (0.0 - 1.0)
    utterance.pitchMultiplier = 1.0;  // ระดับเสียง (0.5 - 2.0)
    utterance.volume = 1.0;           // ความดัง (0.0 - 1.0)
    utterance.preUtteranceDelay = 0;  // หน่วงก่อนพูด
    utterance.postUtteranceDelay = 0.1; // หน่วงหลังพูด
    
    [self.synthesizer speakUtterance:utterance];
}

- (void)pause {
    [self.synthesizer pauseSpeakingAtBoundary:AVSpeechBoundaryWord];
}

- (void)resume {
    [self.synthesizer continueSpeaking];
}

- (void)stop {
    [self.synthesizer stopSpeakingAtBoundary:AVSpeechBoundaryImmediate];
}

// แสดงรายชื่อ voice ทั้งหมด
- (void)listAvailableVoices {
    NSArray<AVSpeechSynthesisVoice *> *voices = [AVSpeechSynthesisVoice speechVoices];
    for (AVSpeechSynthesisVoice *voice in voices) {
        NSLog(@"Voice: %@ (%@) quality: %ld", 
              voice.name, voice.language, (long)voice.quality);
    }
}

#pragma mark - AVSpeechSynthesizerDelegate

- (void)speechSynthesizer:(AVSpeechSynthesizer *)synthesizer 
         didStartSpeechUtterance:(AVSpeechUtterance *)utterance {
    NSLog(@"เริ่มพูด: %@", utterance.speechString);
}

- (void)speechSynthesizer:(AVSpeechSynthesizer *)synthesizer 
        didFinishSpeechUtterance:(AVSpeechUtterance *)utterance {
    NSLog(@"พูดเสร็จแล้ว");
}

- (void)speechSynthesizer:(AVSpeechSynthesizer *)synthesizer 
            willSpeakRangeOfSpeechString:(NSRange)characterRange 
                             utterance:(AVSpeechUtterance *)utterance {
    // ไฮไลต์คำที่กำลังพูด (ใช้สำหรับ karaoke-style reading)
    NSString *word = [utterance.speechString substringWithRange:characterRange];
    NSLog(@"กำลังพูด: %@", word);
}

@end
```

### 7.2 TTS Queue

```objc
- (void)speakMultipleSentences:(NSArray<NSString *> *)sentences {
    AVSpeechSynthesizer *syn = [[AVSpeechSynthesizer alloc] init];
    self.synthesizer = syn;
    
    for (NSString *sentence in sentences) {
        AVSpeechUtterance *utterance = [[AVSpeechUtterance alloc] initWithString:sentence];
        utterance.voice = [AVSpeechSynthesisVoice voiceWithLanguage:@"th-TH"];
        utterance.rate = 0.5;
        utterance.postUtteranceDelay = 0.3;
        
        [syn speakUtterance:utterance];
    }
    // AVSpeechSynthesizer จะ queue utterances อัตโนมัติ
}
```

---

## 8. Speech Recognition (SFSpeechRecognizer)

### 8.1 ตั้งค่า Speech Recognition

ต้องเพิ่มใน Info.plist:
- `NSSpeechRecognitionUsageDescription`
- `NSMicrophoneUsageDescription`

```objc
#import <Speech/Speech.h>

@interface SpeechViewController : UIViewController <SFSpeechRecognizerDelegate>

@property (nonatomic, strong) SFSpeechRecognizer *recognizer;
@property (nonatomic, strong) SFSpeechAudioBufferRecognitionRequest *recognitionRequest;
@property (nonatomic, strong) SFSpeechRecognitionTask *recognitionTask;
@property (nonatomic, strong) AVAudioEngine *audioEngine;

@end

@implementation SpeechViewController

- (void)setupSpeechRecognition {
    // ตั้งค่าสำหรับภาษาไทย
    self.recognizer = [[SFSpeechRecognizer alloc] 
                        initWithLocale:[NSLocale localeWithLocaleIdentifier:@"th-TH"]];
    self.recognizer.delegate = self;
    
    if (!self.recognizer.isAvailable) {
        NSLog(@"Speech recognition ไม่พร้อมใช้งาน");
    }
}

- (void)requestSpeechPermission:(void(^)(BOOL granted))completion {
    [SFSpeechRecognizer requestAuthorization:^(SFSpeechRecognizerAuthorizationStatus status) {
        BOOL granted = (status == SFSpeechRecognizerAuthorizationStatusAuthorized);
        dispatch_async(dispatch_get_main_queue(), ^{
            completion(granted);
        });
    }];
}

- (void)startListening {
    // ยกเลิก task เดิม
    if (self.recognitionTask) {
        [self.recognitionTask cancel];
        self.recognitionTask = nil;
    }
    
    // ตั้งค่า audio session
    AVAudioSession *session = [AVAudioSession sharedInstance];
    [session setCategory:AVAudioSessionCategoryRecord 
             withOptions:AVAudioSessionCategoryOptionDefaultToSpeaker 
                   error:nil];
    [session setMode:AVAudioSessionModeMeasurement error:nil];
    [session setActive:YES withOptions:AVAudioSessionSetActiveOptionNotifyOthersOnDeactivation 
                 error:nil];
    
    // สร้าง recognition request
    self.recognitionRequest = [[SFSpeechAudioBufferRecognitionRequest alloc] init];
    self.recognitionRequest.shouldReportPartialResults = YES;
    
    // เริ่ม recognition task
    __weak typeof(self) weakSelf = self;
    self.recognitionTask = [self.recognizer recognitionTaskWithRequest:self.recognitionRequest 
                                                            resultHandler:^(SFSpeechRecognitionResult *result, NSError *error) {
        
        if (result) {
            NSString *text = result.bestTranscription.formattedString;
            NSLog(@"รับรู้: %@", text);
            
            // แสดง partial result
            weakSelf.transcriptLabel.text = text;
            
            if (result.isFinal) {
                NSLog(@"ผลลัพธ์สุดท้าย: %@", text);
                [weakSelf stopListening];
            }
        }
        
        if (error) {
            NSLog(@"เกิดข้อผิดพลาด: %@", error.localizedDescription);
            [weakSelf stopListening];
        }
    }];
    
    // เพิ่ม audio input
    self.audioEngine = [[AVAudioEngine alloc] init];
    AVAudioInputNode *inputNode = self.audioEngine.inputNode;
    AVAudioFormat *format = [inputNode outputFormatForBus:0];
    
    [inputNode installTapOnBus:0 bufferSize:1024 format:format block:^(AVAudioPCMBuffer *buffer, AVAudioTime *when) {
        [self.recognitionRequest appendAudioPCMBuffer:buffer];
    }];
    
    [self.audioEngine prepare];
    NSError *error = nil;
    [self.audioEngine startAndReturnError:&error];
    
    if (error) {
        NSLog(@"ไม่สามารถเริ่ม audio engine: %@", error.localizedDescription);
    }
    
    NSLog(@"กรุณาพูด...");
}

- (void)stopListening {
    [self.audioEngine stop];
    [self.recognitionRequest endAudio];
    [self.audioEngine.inputNode removeTapOnBus:0];
    
    self.recognitionRequest = nil;
    self.recognitionTask = nil;
    
    [[AVAudioSession sharedInstance] setActive:NO error:nil];
    
    NSLog(@"หยุดฟังแล้ว");
}

#pragma mark - SFSpeechRecognizerDelegate

- (void)speechRecognizer:(SFSpeechRecognizer *)speechRecognizer 
   availabilityDidChange:(BOOL)available {
    NSLog(@"Speech recognition %@", available ? @"พร้อมใช้งาน" : @"ไม่พร้อม");
}

@end
```

### 8.2 One-shot Recognition จากไฟล์เสียง

```objc
- (void)recognizeAudioFile:(NSURL *)audioURL {
    SFSpeechRecognizer *recognizer = [[SFSpeechRecognizer alloc] 
                                       initWithLocale:[NSLocale localeWithLocaleIdentifier:@"th-TH"]];
    
    SFSpeechURLRecognitionRequest *request = [[SFSpeechURLRecognitionRequest alloc] 
                                               initWithURL:audioURL];
    request.shouldReportPartialResults = NO;
    
    [recognizer recognitionTaskWithRequest:request resultHandler:^(SFSpeechRecognitionResult *result, NSError *error) {
        if (result.isFinal) {
            NSLog(@"ผลลัพธ์: %@", result.bestTranscription.formattedString);
            
            // ดูความมั่นใจของแต่ละคำ
            for (SFTranscriptionSegment *segment in result.bestTranscription.segments) {
                NSLog(@"  คำ: %@ (confidence: %.2f)", segment.substring, segment.confidence);
            }
        }
        
        if (error) {
            NSLog(@"เกิดข้อผิดพลาด: %@", error.localizedDescription);
        }
    }];
}
```

---

## 9. Media Library Access

### 9.1 MPMediaPickerController

```objc
#import <MediaPlayer/MediaPlayer.h>

@interface MediaPickerViewController : UIViewController <MPMediaPickerControllerDelegate>
@end

@implementation MediaPickerViewController

- (void)openMediaPicker {
    MPMediaPickerController *picker = [[MPMediaPickerController alloc] 
                                        initWithMediaTypes:MPMediaTypeMusic];
    picker.allowsPickingMultipleItems = NO;
    picker.showsItemsWithProtectedAssets = NO;
    picker.showsCloudItems = NO;
    picker.delegate = self;
    
    [self presentViewController:picker animated:YES completion:nil];
}

#pragma mark - MPMediaPickerControllerDelegate

- (void)mediaPicker:(MPMediaPickerController *)mediaPicker 
  didPickMediaItems:(MPMediaItemCollection *)mediaItemCollection {
    
    [mediaPicker dismissViewControllerAnimated:YES completion:nil];
    
    MPMediaItem *item = mediaItemCollection.items.firstObject;
    if (!item) return;
    
    NSLog(@"เลือก: %@", [item valueForProperty:MPMediaItemPropertyTitle]);
    NSLog(@"ศิลปิน: %@", [item valueForProperty:MPMediaItemPropertyArtist]);
    NSLog(@"อัลบั้ม: %@", [item valueForProperty:MPMediaItemPropertyAlbumTitle]);
    
    // ดู URL ของไฟล์
    NSURL *assetURL = [item valueForProperty:MPMediaItemPropertyAssetURL];
    if (assetURL) {
        // เล่น
        AVPlayer *player = [AVPlayer playerWithURL:assetURL];
        [player play];
    }
}

- (void)mediaPickerDidCancel:(MPMediaPickerController *)mediaPicker {
    [mediaPicker dismissViewControllerAnimated:YES completion:nil];
}

@end
```

### 9.2 MPMusicPlayerController

```objc
- (void)useMusicPlayer {
    // ใช้ player ของระบบ
    MPMusicPlayerController *player = [MPMusicPlayerController systemMusicPlayer];
    
    // ค้นหาเพลง
    MPMediaQuery *query = [MPMediaQuery songsQuery];
    [query addFilterPredicate:[MPMediaPropertyPredicate 
                                predicateWithValue:@"The Beatles" 
                                forProperty:MPMediaItemPropertyArtist]];
    
    NSArray<MPMediaItem *> *items = query.items;
    NSLog(@"พบ %lu เพลง", (unsigned long)items.count);
    
    if (items.count > 0) {
        MPMediaItemCollection *collection = [[MPMediaItemCollection alloc] initWithItems:items];
        
        [player setQueueWithItemCollection:collection];
        [player play];
    }
    
    // ควบคุม player
    NSLog(@"สถานะ: %@", player.playbackState == MPMusicPlaybackStatePlaying ? @"กำลังเล่น" : @"หยุด");
    
    player.volume = 0.8;
    player.shuffleMode = MPMusicShuffleModeOff;
    player.repeatMode = MPMusicRepeatModeAll;
    
    // ฟัง notification
    [player beginGeneratingPlaybackNotifications];
    [[NSNotificationCenter defaultCenter] addObserver:self 
                                             selector:@selector(nowPlayingItemChanged:)
                                                 name:MPMusicPlayerControllerNowPlayingItemDidChangeNotification 
                                               object:player];
}

- (void)nowPlayingItemChanged:(NSNotification *)note {
    MPMusicPlayerController *player = note.object;
    MPMediaItem *item = player.nowPlayingItem;
    NSLog(@"เล่น: %@", [item valueForProperty:MPMediaItemPropertyTitle]);
}
```

---

## 10. AVAudioEngine - ประมวลผลเสียงขั้นสูง

### 10.1 Playback พร้อม Effects

```objc
- (void)setupAudioEngineWithEffects {
    AVAudioEngine *engine = [[AVAudioEngine alloc] init];
    self.audioEngine = engine;
    
    // สร้าง player node
    AVAudioPlayerNode *playerNode = [[AVAudioPlayerNode alloc] init];
    [engine attachNode:playerNode];
    
    // สร้าง EQ
    AVAudioUnitEQ *eq = [[AVAudioUnitEQ alloc] initWithNumberOfBands:3];
    [engine attachNode:eq];
    
    // ตั้งค่า EQ bands
    eq.bands[0].frequency = 80;    // Bass
    eq.bands[0].gain = 3.0;
    eq.bands[0].bandwidth = 1.0;
    eq.bands[0].filterType = AVAudioUnitEQFilterTypeParametric;
    eq.bands[0].bypass = NO;
    
    eq.bands[1].frequency = 1000;  // Mid
    eq.bands[1].gain = 0.0;
    eq.bands[1].bandwidth = 1.0;
    eq.bands[1].filterType = AVAudioUnitEQFilterTypeParametric;
    eq.bands[1].bypass = NO;
    
    eq.bands[2].frequency = 10000; // Treble
    eq.bands[2].gain = 2.0;
    eq.bands[2].bandwidth = 1.0;
    eq.bands[2].filterType = AVAudioUnitEQFilterTypeParametric;
    eq.bands[2].bypass = NO;
    
    // Reverb
    AVAudioUnitReverb *reverb = [[AVAudioUnitReverb alloc] init];
    [reverb loadFactoryPreset:AVAudioUnitReverbPresetLargeChamber];
    reverb.wetDryMix = 30.0;  // 0-100%
    [engine attachNode:reverb];
    
    // Distortion
    AVAudioUnitDistortion *distortion = [[AVAudioUnitDistortion alloc] init];
    [distortion loadFactoryPreset:AVAudioUnitDistortionPresetDrumsBitBrush];
    distortion.wetDryMix = 0.0;  // ปิดไว้ก่อน
    [engine attachNode:distortion];
    
    // เชื่อมต่อ nodes
    AVAudioFormat *format = [[AVAudioFormat alloc] initStandardFormatWithSampleRate:44100 channels:2];
    
    [engine connect:playerNode to:eq format:format];
    [engine connect:eq to:reverb format:format];
    [engine connect:reverb to:distortion format:format];
    [engine connect:distortion to:engine.mainMixerNode format:format];
    
    // โหลดและเล่นไฟล์
    NSURL *url = [[NSBundle mainBundle] URLForResource:@"music" withExtension:@"mp3"];
    AVAudioFile *audioFile = [[AVAudioFile alloc] initForReading:url error:nil];
    
    [engine startAndReturnError:nil];
    [playerNode scheduleFile:audioFile atTime:nil completionHandler:nil];
    [playerNode play];
}
```

### 10.2 Real-time Recording พร้อม Monitoring

```objc
- (void)recordWithMonitoring {
    AVAudioEngine *engine = [[AVAudioEngine alloc] init];
    
    AVAudioInputNode *inputNode = engine.inputNode;
    AVAudioFormat *inputFormat = [inputNode inputFormatForBus:0];
    
    // ไฟล์สำหรับบันทึก
    NSURL *outputURL = [NSURL fileURLWithPath:@"/tmp/recording.caf"];
    AVAudioFile *outputFile = [[AVAudioFile alloc] initForWriting:outputURL
                                                          settings:inputFormat.settings
                                                             error:nil];
    
    // tap on input
    [inputNode installTapOnBus:0 bufferSize:4096 format:inputFormat 
                         block:^(AVAudioPCMBuffer *buffer, AVAudioTime *when) {
        [outputFile writeFromBuffer:buffer error:nil];
        
        // วัด level
        AudioBufferList *bufferList = buffer.audioBufferList;
        if (bufferList->mNumberBuffers > 0) {
            float *samples = (float *)bufferList->mBuffers[0].mData;
            NSUInteger frameCount = buffer.frameLength;
            
            float sum = 0;
            for (NSUInteger i = 0; i < frameCount; i++) {
                sum += samples[i] * samples[i];
            }
            float rms = sqrt(sum / frameCount);
            float dB = 20 * log10(rms);
            
            dispatch_async(dispatch_get_main_queue(), ^{
                // อัปเดต UI level meter
                NSLog(@"Level: %.1f dB", dB);
            });
        }
    }];
    
    [engine startAndReturnError:nil];
}
```

---

## 11. Video Capture (บันทึกวิดีโอ)

### 11.1 บันทึกวิดีโอด้วย AVCaptureSession

```objc
@interface VideoCaptureVC : UIViewController <AVCaptureFileOutputRecordingDelegate>

@property (nonatomic, strong) AVCaptureSession *session;
@property (nonatomic, strong) AVCaptureMovieFileOutput *movieOutput;
@property (nonatomic, strong) AVCaptureVideoPreviewLayer *previewLayer;
@property (nonatomic, assign) BOOL isRecording;

@end

@implementation VideoCaptureVC

- (void)setupVideoCapture {
    self.session = [[AVCaptureSession alloc] init];
    self.session.sessionPreset = AVCaptureSessionPresetHigh;
    
    // เพิ่ม video input
    AVCaptureDevice *videoDevice = [AVCaptureDevice defaultDeviceWithMediaType:AVMediaTypeVideo];
    AVCaptureDeviceInput *videoInput = [AVCaptureDeviceInput deviceInputWithDevice:videoDevice 
                                                                             error:nil];
    if ([self.session canAddInput:videoInput]) {
        [self.session addInput:videoInput];
    }
    
    // เพิ่ม audio input
    AVCaptureDevice *audioDevice = [AVCaptureDevice defaultDeviceWithMediaType:AVMediaTypeAudio];
    AVCaptureDeviceInput *audioInput = [AVCaptureDeviceInput deviceInputWithDevice:audioDevice 
                                                                             error:nil];
    if ([self.session canAddInput:audioInput]) {
        [self.session addInput:audioInput];
    }
    
    // เพิ่ม output
    self.movieOutput = [[AVCaptureMovieFileOutput alloc] init];
    if ([self.session canAddOutput:self.movieOutput]) {
        [self.session addOutput:self.movieOutput];
    }
    
    // Setup preview
    self.previewLayer = [AVCaptureVideoPreviewLayer layerWithSession:self.session];
    self.previewLayer.frame = self.view.bounds;
    self.previewLayer.videoGravity = AVLayerVideoGravityResizeAspectFill;
    [self.view.layer insertSublayer:self.previewLayer atIndex:0];
    
    dispatch_async(dispatch_get_global_queue(DISPATCH_QUEUE_PRIORITY_HIGH, 0), ^{
        [self.session startRunning];
    });
}

- (void)toggleRecording {
    if (self.isRecording) {
        [self stopRecording];
    } else {
        [self startRecording];
    }
}

- (void)startRecording {
    NSString *docPath = [NSSearchPathForDirectoriesInDomains(
        NSDocumentDirectory, NSUserDomainMask, YES) firstObject];
    NSString *filename = [NSString stringWithFormat:@"video_%ld.mov", (long)[NSDate date].timeIntervalSince1970];
    NSURL *outputURL = [NSURL fileURLWithPath:[docPath stringByAppendingPathComponent:filename]];
    
    [self.movieOutput startRecordingToOutputFileURL:outputURL recordingDelegate:self];
    self.isRecording = YES;
    NSLog(@"เริ่มบันทึกวิดีโอ");
}

- (void)stopRecording {
    [self.movieOutput stopRecording];
    self.isRecording = NO;
}

#pragma mark - AVCaptureFileOutputRecordingDelegate

- (void)captureOutput:(AVCaptureFileOutput *)output 
didFinishRecordingToOutputFileAtURL:(NSURL *)outputFileURL 
      fromConnections:(NSArray<AVCaptureConnection *> *)connections 
                error:(NSError *)error {
    
    if (error) {
        NSLog(@"บันทึกล้มเหลว: %@", error.localizedDescription);
        return;
    }
    
    NSLog(@"บันทึกวิดีโอสำเร็จ: %@", outputFileURL.path);
    
    // บันทึกลง Photo Library
    [[PHPhotoLibrary sharedPhotoLibrary] performChanges:^{
        [PHAssetChangeRequest creationRequestForAssetFromVideoAtFileURL:outputFileURL];
    } completionHandler:^(BOOL success, NSError *error) {
        if (success) {
            NSLog(@"บันทึกลง Photo Library สำเร็จ");
        }
    }];
}

@end
```

---

## แบบฝึกหัด (Practice Exercises)

### แบบฝึกหัดที่ 1: Music Player App

```objc
// สร้าง Music Player ที่มีฟีเจอร์ครบถ้วน
@interface FullMusicPlayerVC : UIViewController

@property (nonatomic, strong) AVPlayer *player;
@property (nonatomic, strong) NSArray<NSDictionary *> *songs;
@property (nonatomic, assign) NSInteger currentIndex;

@property (nonatomic, strong) UIImageView *albumArtView;
@property (nonatomic, strong) UILabel *titleLabel;
@property (nonatomic, strong) UILabel *artistLabel;
@property (nonatomic, strong) UISlider *progressSlider;
@property (nonatomic, strong) UILabel *currentTimeLabel;
@property (nonatomic, strong) UILabel *durationLabel;
@property (nonatomic, strong) UIButton *playPauseButton;
@property (nonatomic, strong) UIButton *previousButton;
@property (nonatomic, strong) UIButton *nextButton;
@property (nonatomic, strong) UISlider *volumeSlider;

@end

@implementation FullMusicPlayerVC

- (void)viewDidLoad {
    [super viewDidLoad];
    [self setupPlaylist];
    [self setupUI];
    [self setupRemoteControls];
    [self playAtIndex:0];
}

- (void)setupPlaylist {
    // ในแอปจริงจะโหลดจาก Music Library หรือ server
    self.songs = @[
        @{@"title": @"เพลงที่ 1", @"artist": @"ศิลปิน 1", @"url": @"song1.mp3"},
        @{@"title": @"เพลงที่ 2", @"artist": @"ศิลปิน 2", @"url": @"song2.mp3"},
        @{@"title": @"เพลงที่ 3", @"artist": @"ศิลปิน 3", @"url": @"song3.mp3"},
    ];
}

- (void)setupUI {
    self.view.backgroundColor = [UIColor systemBackgroundColor];
    
    // Album Art
    self.albumArtView = [[UIImageView alloc] initWithFrame:CGRectMake(60, 100, 250, 250)];
    self.albumArtView.contentMode = UIViewContentModeScaleAspectFill;
    self.albumArtView.clipsToBounds = YES;
    self.albumArtView.layer.cornerRadius = 12;
    self.albumArtView.backgroundColor = [UIColor systemGray5Color];
    [self.view addSubview:self.albumArtView];
    
    // Title
    self.titleLabel = [[UILabel alloc] initWithFrame:CGRectMake(20, 370, 340, 30)];
    self.titleLabel.font = [UIFont boldSystemFontOfSize:22];
    self.titleLabel.textAlignment = NSTextAlignmentCenter;
    [self.view addSubview:self.titleLabel];
    
    // Artist
    self.artistLabel = [[UILabel alloc] initWithFrame:CGRectMake(20, 405, 340, 25)];
    self.artistLabel.font = [UIFont systemFontOfSize:16];
    self.artistLabel.textColor = [UIColor secondaryLabelColor];
    self.artistLabel.textAlignment = NSTextAlignmentCenter;
    [self.view addSubview:self.artistLabel];
    
    // Progress
    self.progressSlider = [[UISlider alloc] initWithFrame:CGRectMake(20, 450, 340, 30)];
    [self.progressSlider addTarget:self action:@selector(seek:) 
                  forControlEvents:UIControlEventValueChanged];
    [self.view addSubview:self.progressSlider];
    
    // Time labels
    self.currentTimeLabel = [[UILabel alloc] initWithFrame:CGRectMake(20, 480, 80, 20)];
    self.currentTimeLabel.font = [UIFont monospacedSystemFontOfSize:12 weight:UIFontWeightRegular];
    self.currentTimeLabel.text = @"0:00";
    [self.view addSubview:self.currentTimeLabel];
    
    self.durationLabel = [[UILabel alloc] initWithFrame:CGRectMake(280, 480, 80, 20)];
    self.durationLabel.font = [UIFont monospacedSystemFontOfSize:12 weight:UIFontWeightRegular];
    self.durationLabel.text = @"0:00";
    self.durationLabel.textAlignment = NSTextAlignmentRight;
    [self.view addSubview:self.durationLabel];
    
    // Controls
    self.previousButton = [UIButton buttonWithType:UIButtonTypeSystem];
    self.previousButton.frame = CGRectMake(60, 520, 60, 60);
    [self.previousButton setImage:[UIImage systemImageNamed:@"backward.fill"] forState:UIControlStateNormal];
    self.previousButton.tintColor = [UIColor labelColor];
    [self.previousButton addTarget:self action:@selector(previous) forControlEvents:UIControlEventTouchUpInside];
    [self.view addSubview:self.previousButton];
    
    self.playPauseButton = [UIButton buttonWithType:UIButtonTypeSystem];
    self.playPauseButton.frame = CGRectMake(165, 510, 70, 70);
    [self.playPauseButton setImage:[UIImage systemImageNamed:@"play.circle.fill"] forState:UIControlStateNormal];
    self.playPauseButton.tintColor = [UIColor labelColor];
    [self.playPauseButton addTarget:self action:@selector(togglePlay) forControlEvents:UIControlEventTouchUpInside];
    [self.view addSubview:self.playPauseButton];
    
    self.nextButton = [UIButton buttonWithType:UIButtonTypeSystem];
    self.nextButton.frame = CGRectMake(260, 520, 60, 60);
    [self.nextButton setImage:[UIImage systemImageNamed:@"forward.fill"] forState:UIControlStateNormal];
    self.nextButton.tintColor = [UIColor labelColor];
    [self.nextButton addTarget:self action:@selector(next) forControlEvents:UIControlEventTouchUpInside];
    [self.view addSubview:self.nextButton];
    
    // Volume
    self.volumeSlider = [[UISlider alloc] initWithFrame:CGRectMake(20, 610, 340, 30)];
    self.volumeSlider.minimumValueImage = [UIImage systemImageNamed:@"speaker.fill"];
    self.volumeSlider.maximumValueImage = [UIImage systemImageNamed:@"speaker.3.fill"];
    self.volumeSlider.value = 1.0;
    [self.volumeSlider addTarget:self action:@selector(volumeChanged:) 
                forControlEvents:UIControlEventValueChanged];
    [self.view addSubview:self.volumeSlider];
}

- (void)playAtIndex:(NSInteger)index {
    if (index < 0 || index >= self.songs.count) return;
    self.currentIndex = index;
    
    NSDictionary *song = self.songs[index];
    NSURL *url = [[NSBundle mainBundle] URLForResource:song[@"url"] withExtension:nil];
    
    if (!url) {
        NSLog(@"ไม่พบไฟล์: %@", song[@"url"]);
        return;
    }
    
    [self.player pause];
    
    AVPlayerItem *item = [AVPlayerItem playerItemWithURL:url];
    self.player = [AVPlayer playerWithPlayerItem:item];
    
    __weak typeof(self) weakSelf = self;
    [self.player addPeriodicTimeObserverForInterval:CMTimeMake(1, 10)
                                             queue:dispatch_get_main_queue()
                                        usingBlock:^(CMTime time) {
        [weakSelf updateTimeUI];
    }];
    
    [[NSNotificationCenter defaultCenter] addObserver:self
                                             selector:@selector(songFinished)
                                                 name:AVPlayerItemDidPlayToEndTimeNotification
                                               object:item];
    
    self.titleLabel.text = song[@"title"];
    self.artistLabel.text = song[@"artist"];
    self.albumArtView.image = [UIImage imageNamed:@"album_art"];
    
    [self.player play];
    [self updatePlayButton];
    [self updateNowPlayingInfo];
}

- (void)togglePlay {
    if (self.player.rate > 0) {
        [self.player pause];
    } else {
        [self.player play];
    }
    [self updatePlayButton];
}

- (void)previous {
    NSInteger idx = self.currentIndex - 1;
    if (idx < 0) idx = self.songs.count - 1;
    [self playAtIndex:idx];
}

- (void)next {
    NSInteger idx = (self.currentIndex + 1) % self.songs.count;
    [self playAtIndex:idx];
}

- (void)songFinished {
    [self next];
}

- (void)seek:(UISlider *)slider {
    CMTime duration = self.player.currentItem.duration;
    if (CMTIME_IS_INVALID(duration)) return;
    
    Float64 seconds = CMTimeGetSeconds(duration) * slider.value;
    [self.player seekToTime:CMTimeMakeWithSeconds(seconds, 600)];
}

- (void)volumeChanged:(UISlider *)slider {
    self.player.volume = slider.value;
}

- (void)updatePlayButton {
    NSString *iconName = self.player.rate > 0 ? @"pause.circle.fill" : @"play.circle.fill";
    [self.playPauseButton setImage:[UIImage systemImageNamed:iconName] 
                          forState:UIControlStateNormal];
}

- (void)updateTimeUI {
    CMTime current = self.player.currentTime;
    CMTime total = self.player.currentItem.duration;
    
    if (CMTIME_IS_INVALID(current) || CMTIME_IS_INVALID(total)) return;
    
    Float64 currentSec = CMTimeGetSeconds(current);
    Float64 totalSec = CMTimeGetSeconds(total);
    
    self.currentTimeLabel.text = [NSString stringWithFormat:@"%ld:%02ld",
                                  (long)currentSec / 60, (long)currentSec % 60];
    self.durationLabel.text = [NSString stringWithFormat:@"%ld:%02ld",
                                (long)totalSec / 60, (long)totalSec % 60];
    
    if (totalSec > 0) {
        self.progressSlider.value = currentSec / totalSec;
    }
}

- (void)updateNowPlayingInfo {
    NSDictionary *song = self.songs[self.currentIndex];
    
    MPNowPlayingInfoCenter.defaultCenter.nowPlayingInfo = @{
        MPMediaItemPropertyTitle: song[@"title"],
        MPMediaItemPropertyArtist: song[@"artist"],
        MPNowPlayingInfoPropertyElapsedPlaybackTime: @(CMTimeGetSeconds(self.player.currentTime)),
        MPMediaItemPropertyPlaybackDuration: @(CMTimeGetSeconds(self.player.currentItem.duration)),
        MPNowPlayingInfoPropertyPlaybackRate: @(self.player.rate)
    };
}

- (void)setupRemoteControls {
    [[AVAudioSession sharedInstance] setCategory:AVAudioSessionCategoryPlayback error:nil];
    [[AVAudioSession sharedInstance] setActive:YES error:nil];
    
    MPRemoteCommandCenter *cc = [MPRemoteCommandCenter sharedCommandCenter];
    
    [cc.playCommand addTargetWithHandler:^MPRemoteCommandHandlerStatus(MPRemoteCommandEvent *event) {
        [self.player play];
        return MPRemoteCommandHandlerStatusSuccess;
    }];
    
    [cc.pauseCommand addTargetWithHandler:^MPRemoteCommandHandlerStatus(MPRemoteCommandEvent *event) {
        [self.player pause];
        return MPRemoteCommandHandlerStatusSuccess;
    }];
    
    [cc.nextTrackCommand addTargetWithHandler:^MPRemoteCommandHandlerStatus(MPRemoteCommandEvent *event) {
        [self next];
        return MPRemoteCommandHandlerStatusSuccess;
    }];
    
    [cc.previousTrackCommand addTargetWithHandler:^MPRemoteCommandHandlerStatus(MPRemoteCommandEvent *event) {
        [self previous];
        return MPRemoteCommandHandlerStatusSuccess;
    }];
}

@end
```

### แบบฝึกหัดที่ 2: Voice Translator
```objc
// App ที่ฟังเสียง แล้วแปลงเป็นข้อความ
// ให้ implement:
// 1. กดปุ่มค้างเพื่อบันทึก
// 2. ใช้ SFSpeechRecognizer แปลงเสียงเป็นข้อความ
// 3. แสดงผลบนหน้าจอ
// 4. ใช้ AVSpeechSynthesizer พูดข้อความที่รับรู้

@interface VoiceTranslatorVC : UIViewController <SFSpeechRecognizerDelegate>
@property (nonatomic, strong) SFSpeechRecognizer *speechRecognizer;
@property (nonatomic, strong) SFSpeechAudioBufferRecognitionRequest *request;
@property (nonatomic, strong) SFSpeechRecognitionTask *task;
@property (nonatomic, strong) AVAudioEngine *audioEngine;
@property (nonatomic, strong) AVSpeechSynthesizer *synthesizer;
@property (nonatomic, strong) UITextView *resultTextView;
@property (nonatomic, strong) UIButton *listenButton;
@end

@implementation VoiceTranslatorVC

- (void)viewDidLoad {
    [super viewDidLoad];
    
    self.speechRecognizer = [[SFSpeechRecognizer alloc] 
                               initWithLocale:[NSLocale localeWithLocaleIdentifier:@"th-TH"]];
    self.synthesizer = [[AVSpeechSynthesizer alloc] init];
    
    [self setupUI];
    [self requestPermissions];
}

- (void)setupUI {
    self.view.backgroundColor = [UIColor systemBackgroundColor];
    
    self.resultTextView = [[UITextView alloc] initWithFrame:CGRectMake(20, 100, 340, 300)];
    self.resultTextView.font = [UIFont systemFontOfSize:18];
    self.resultTextView.editable = NO;
    self.resultTextView.layer.borderWidth = 1;
    self.resultTextView.layer.borderColor = [UIColor systemGrayColor].CGColor;
    self.resultTextView.layer.cornerRadius = 8;
    [self.view addSubview:self.resultTextView];
    
    self.listenButton = [UIButton buttonWithType:UIButtonTypeSystem];
    self.listenButton.frame = CGRectMake(130, 440, 120, 120);
    [self.listenButton setTitle:@"พูด" forState:UIControlStateNormal];
    self.listenButton.titleLabel.font = [UIFont boldSystemFontOfSize:20];
    self.listenButton.backgroundColor = [UIColor systemBlueColor];
    self.listenButton.tintColor = [UIColor whiteColor];
    self.listenButton.layer.cornerRadius = 60;
    
    [self.listenButton addTarget:self action:@selector(startListening) 
                forControlEvents:UIControlEventTouchDown];
    [self.listenButton addTarget:self action:@selector(stopListening) 
                forControlEvents:UIControlEventTouchUpInside];
    [self.listenButton addTarget:self action:@selector(stopListening) 
                forControlEvents:UIControlEventTouchUpOutside];
    
    [self.view addSubview:self.listenButton];
}

- (void)requestPermissions {
    [SFSpeechRecognizer requestAuthorization:^(SFSpeechRecognizerAuthorizationStatus status) {
        dispatch_async(dispatch_get_main_queue(), ^{
            self.listenButton.enabled = (status == SFSpeechRecognizerAuthorizationStatusAuthorized);
        });
    }];
}

- (void)startListening {
    if (self.task) {
        [self.task cancel];
        self.task = nil;
    }
    
    [[AVAudioSession sharedInstance] setCategory:AVAudioSessionCategoryRecord error:nil];
    [[AVAudioSession sharedInstance] setActive:YES error:nil];
    
    self.request = [[SFSpeechAudioBufferRecognitionRequest alloc] init];
    self.request.shouldReportPartialResults = YES;
    
    self.audioEngine = [[AVAudioEngine alloc] init];
    AVAudioInputNode *inputNode = self.audioEngine.inputNode;
    AVAudioFormat *format = [inputNode outputFormatForBus:0];
    
    [inputNode installTapOnBus:0 bufferSize:1024 format:format 
                         block:^(AVAudioPCMBuffer *buffer, AVAudioTime *when) {
        [self.request appendAudioPCMBuffer:buffer];
    }];
    
    [self.audioEngine prepare];
    [self.audioEngine startAndReturnError:nil];
    
    __weak typeof(self) weakSelf = self;
    self.task = [self.speechRecognizer recognitionTaskWithRequest:self.request 
                                                  resultHandler:^(SFSpeechRecognitionResult *result, NSError *error) {
        if (result) {
            dispatch_async(dispatch_get_main_queue(), ^{
                weakSelf.resultTextView.text = result.bestTranscription.formattedString;
            });
        }
    }];
    
    [UIView animateWithDuration:0.2 animations:^{
        self.listenButton.transform = CGAffineTransformMakeScale(1.1, 1.1);
        self.listenButton.backgroundColor = [UIColor systemRedColor];
    }];
}

- (void)stopListening {
    [self.audioEngine stop];
    [self.request endAudio];
    [self.audioEngine.inputNode removeTapOnBus:0];
    
    [UIView animateWithDuration:0.2 animations:^{
        self.listenButton.transform = CGAffineTransformIdentity;
        self.listenButton.backgroundColor = [UIColor systemBlueColor];
    }];
    
    // พูดซ้ำสิ่งที่รับรู้
    NSString *recognizedText = self.resultTextView.text;
    if (recognizedText.length > 0) {
        AVSpeechUtterance *utterance = [[AVSpeechUtterance alloc] initWithString:recognizedText];
        utterance.voice = [AVSpeechSynthesisVoice voiceWithLanguage:@"th-TH"];
        utterance.rate = 0.5;
        
        [[AVAudioSession sharedInstance] setCategory:AVAudioSessionCategoryPlayback error:nil];
        [self.synthesizer speakUtterance:utterance];
    }
}

@end
```

---

## สรุป

ในบทนี้เราได้เรียนรู้ AVFoundation อย่างครอบคลุม:

1. **AVFoundation Overview** - โครงสร้างและ AVAudioSession
2. **AVAudioPlayer** - เล่นไฟล์เสียงพร้อม metering และ playlist
3. **AVAudioRecorder** - บันทึกเสียงพร้อม monitoring
4. **AVPlayer + AVPlayerViewController** - เล่น video ทั้งแบบ simple และ custom UI
5. **Background Playback** - เล่นในพื้นหลัง + Lock Screen controls
6. **System Sounds** - AudioServicesPlaySystemSound
7. **AVSpeechSynthesizer** - Text-to-Speech ทั้งไทยและอังกฤษ
8. **SFSpeechRecognizer** - Speech-to-Text แบบ real-time และจากไฟล์
9. **MPMediaPickerController** - เข้าถึง Music Library
10. **AVAudioEngine** - ประมวลผลเสียงขั้นสูงพร้อม EQ และ Reverb
11. **AVCaptureSession** - บันทึกวิดีโอจากกล้อง

---

## จบบทที่ 60 และส่วนที่ 51-60

ยินดีด้วย! คุณได้เรียนจบส่วนที่ 51-60 ซึ่งครอบคลุมหัวข้อขั้นสูงสำหรับ iOS Development:

- Part 51-56: หัวข้อขั้นสูงก่อนหน้า
- **Part 57**: การจัดการรูปภาพ (UIImage, UIImageView, Async Loading, PHPhotoLibrary, Camera)
- **Part 58**: Core Graphics (CGContext, Shapes, Gradients, Text, Custom Drawing)
- **Part 59**: Core Animation (CALayer, CABasicAnimation, CAKeyframeAnimation, CAEmitterLayer)
- **Part 60**: Audio & Video (AVAudioPlayer, AVAudioRecorder, AVPlayer, SFSpeechRecognizer, AVSpeechSynthesizer)

ทักษะเหล่านี้จะช่วยให้คุณสร้างแอปพลิเคชัน iOS ที่มีความสามารถด้านมัลติมีเดียระดับมืออาชีพได้
