---
layout: post
title: "Flutter just_audio & audio_service 심화: 백그라운드 오디오 재생·미디어 알림·AudioSession 완전 정복"
date: 2026-10-25
categories: [android, flutter]
tags: [flutter, dart, audio, just_audio, audio_service, audio_session, background_audio, media_notification]
---

모바일 앱에서 오디오 재생은 단순히 소리를 내는 것 이상의 도전을 요구한다. 화면이 꺼진 상태에서도 음악이 계속 재생되어야 하고, 잠금 화면과 알림창에는 제어 버튼이 표시되어야 하며, 전화가 오면 자동으로 일시 정지해야 한다. Flutter 생태계에서 이 모든 요구를 충족하는 사실상의 표준 조합은 **just_audio** + **audio_service** + **audio_session** 트리오다.

이 글에서는 세 패키지의 역할을 명확히 구분하고, 백그라운드 재생부터 미디어 알림, 오디오 포커스 관리까지 프로덕션 수준의 오디오 플레이어를 단계별로 구현한다.

---

## 패키지 역할 분담

| 패키지 | 버전 | 역할 |
|--------|------|------|
| just_audio | 0.10.6 | 실제 오디오 재생 엔진 (URL, 파일, 에셋, HLS, DASH) |
| audio_service | 0.18.19 | 백그라운드 격리 실행 + 시스템 미디어 알림 |
| audio_session | 0.2.4 | iOS 오디오 세션 & Android 오디오 포커스 |
| just_audio_background | 0.0.1-beta.17 | just_audio + audio_service 단순 통합 레이어 |

### 왜 just_audio만으로는 부족한가

`just_audio`는 강력한 재생 엔진이지만, 앱이 백그라운드로 전환되는 순간 Android에서는 시스템이 프로세스를 종료시킬 수 있다. iOS에서는 **Background Modes > Audio** 설정 없이는 재생이 즉시 중단된다. `audio_service`는 오디오 코드를 별도의 백그라운드 격리 환경(iOS: UIBackgroundModes, Android: Foreground Service)에서 실행시켜 이 문제를 근본적으로 해결한다.

### just_audio_background vs audio_service 직접 사용

단일 `AudioPlayer` 인스턴스만 쓰는 단순 앱이라면 `just_audio_background`가 편하다. `JustAudioBackground.init()` 한 줄로 백그라운드 재생과 미디어 알림을 얻는다. 반면 팟캐스트 앱처럼 복잡한 큐 관리, 커스텀 액션, 멀티-플레이어가 필요하면 `audio_service`를 직접 사용해야 한다.

---

## 환경 설정

### pubspec.yaml

```yaml
dependencies:
  just_audio: ^0.10.6
  audio_service: ^0.18.19
  audio_session: ^0.2.4
  rxdart: ^0.28.0
```

### Android 설정 (AndroidManifest.xml)

```xml
<manifest>
  <uses-permission android:name="android.permission.WAKE_LOCK"/>
  <uses-permission android:name="android.permission.FOREGROUND_SERVICE"/>
  <uses-permission android:name="android.permission.FOREGROUND_SERVICE_MEDIA_PLAYBACK"/>

  <application>
    <service
        android:name="com.ryanheise.audioservice.AudioServiceFragmentedActivity"
        android:foregroundServiceType="mediaPlayback"
        android:exported="true">
      <intent-filter>
        <action android:name="android.media.browse.MediaBrowserService"/>
      </intent-filter>
    </service>

    <receiver android:name="com.ryanheise.audioservice.MediaButtonReceiver"
        android:exported="true">
      <intent-filter>
        <action android:name="android.intent.action.MEDIA_BUTTON"/>
      </intent-filter>
    </receiver>
  </application>
</manifest>
```

`android:foregroundServiceType="mediaPlayback"` 속성은 **Android 14(API 34)** 이상에서 필수다. 누락 시 앱이 포그라운드 서비스 시작 시점에 크래시된다.

### iOS 설정 (Info.plist)

```xml
<key>UIBackgroundModes</key>
<array>
  <string>audio</string>
</array>
```

---

## 코드 예제 1: just_audio 플레이리스트 재생기

just_audio의 설계 철학은 **AudioSource 트리**다. 단일 URL부터 여러 소스를 이어 붙인 Concatenating Sequence까지 동일한 API로 다룬다.

```dart
import 'package:just_audio/just_audio.dart';
import 'package:audio_service/audio_service.dart';

class MusicPlayer {
  final _player = AudioPlayer();
  late final ConcatenatingAudioSource _playlist;

  Future<void> init(List<MediaItem> mediaItems) async {
    _playlist = ConcatenatingAudioSource(
      useLazyPreparation: true,   // 다음 트랙을 미리 로드해 갭 최소화
      shuffleOrder: DefaultShuffleOrder(),
      children: mediaItems.map((item) {
        return AudioSource.uri(
          Uri.parse(item.extras!['url'] as String),
          tag: item,
        );
      }).toList(),
    );

    await _player.setAudioSource(
      _playlist,
      initialIndex: 0,
      initialPosition: Duration.zero,
    );
    await _player.setLoopMode(LoopMode.all);
    await _player.play();
  }

  // --- 컨트롤 ---
  Future<void> play() => _player.play();
  Future<void> pause() => _player.pause();
  Future<void> seek(Duration pos) => _player.seek(pos);
  Future<void> skipToNext() => _player.seekToNext();
  Future<void> skipToPrevious() => _player.seekToPrevious();

  /// 재생 속도: 0.5x ~ 2.0x, 피치 변환 없이
  Future<void> setSpeed(double speed) => _player.setSpeed(speed);

  /// 현재 재생 중인 트랙 바로 다음에 삽입 (동적 큐)
  Future<void> insertNext(MediaItem item) async {
    final idx = (_player.currentIndex ?? 0) + 1;
    await _playlist.insert(idx, AudioSource.uri(
      Uri.parse(item.extras!['url'] as String),
      tag: item,
    ));
  }

  // --- 스트림 (UI 구독용) ---
  Stream<PlayerState>  get playerStateStream      => _player.playerStateStream;
  Stream<Duration>     get positionStream          => _player.positionStream;
  Stream<Duration?>    get durationStream          => _player.durationStream;
  Stream<Duration>     get bufferedPositionStream  => _player.bufferedPositionStream;
  Stream<SequenceState?> get sequenceStateStream  => _player.sequenceStateStream;

  void dispose() => _player.dispose();
}
```

### 재생 바 위젯

```dart
class PositionSlider extends StatelessWidget {
  final MusicPlayer player;
  const PositionSlider({required this.player});

  @override
  Widget build(BuildContext context) {
    return StreamBuilder<Duration?>(
      stream: player.durationStream,
      builder: (context, durSnap) {
        final duration = durSnap.data ?? Duration.zero;
        return StreamBuilder<Duration>(
          stream: player.positionStream,
          builder: (context, posSnap) {
            var position = posSnap.data ?? Duration.zero;
            if (position > duration) position = duration;

            return Slider(
              min: 0,
              max: duration.inMilliseconds.toDouble(),
              value: position.inMilliseconds.clamp(
                  0, duration.inMilliseconds).toDouble(),
              onChanged: (v) =>
                  player.seek(Duration(milliseconds: v.round())),
            );
          },
        );
      },
    );
  }
}
```

---

## 코드 예제 2: audio_service로 백그라운드 재생 + 미디어 알림

`audio_service`의 핵심 추상화는 **AudioHandler**다. 이 클래스 안에 오디오 로직을 담으면, 라이브러리가 백그라운드 실행과 시스템 미디어 알림을 자동으로 연결해 준다.

```dart
import 'package:audio_service/audio_service.dart';
import 'package:just_audio/just_audio.dart';

class MyAudioHandler extends BaseAudioHandler
    with QueueHandler, SeekHandler {
  final _player = AudioPlayer();
  final _playlist = ConcatenatingAudioSource(children: []);

  MyAudioHandler() {
    _player.setAudioSource(_playlist);
    _listenToPlaybackEvents();
    _listenToSequenceState();
  }

  void _listenToPlaybackEvents() {
    _player.playbackEventStream.listen((event) {
      final playing = _player.playing;
      playbackState.add(playbackState.value.copyWith(
        controls: [
          MediaControl.skipToPrevious,
          if (playing) MediaControl.pause else MediaControl.play,
          MediaControl.stop,
          MediaControl.skipToNext,
        ],
        systemActions: const {
          MediaAction.seek,
          MediaAction.seekForward,
          MediaAction.seekBackward,
        },
        androidCompactActionIndices: const [0, 1, 3],
        processingState: const {
          ProcessingState.idle:      AudioProcessingState.idle,
          ProcessingState.loading:   AudioProcessingState.loading,
          ProcessingState.buffering: AudioProcessingState.buffering,
          ProcessingState.ready:     AudioProcessingState.ready,
          ProcessingState.completed: AudioProcessingState.completed,
        }[_player.processingState]!,
        playing: playing,
        updatePosition: _player.position,
        bufferedPosition: _player.bufferedPosition,
        speed: _player.speed,
        queueIndex: event.currentIndex,
      ));
    });
  }

  void _listenToSequenceState() {
    _player.sequenceStateStream.listen((state) {
      if (state == null) return;
      // 현재 트랙 메타데이터를 시스템에 전달 (알림·잠금 화면 표시)
      mediaItem.add(state.currentSource?.tag as MediaItem?);
      // 셔플 순서 반영하여 큐 업데이트
      queue.add(state.effectiveSequence
          .map((s) => s.tag as MediaItem)
          .toList());
    });
  }

  // --- BaseAudioHandler 오버라이드 ---

  @override
  Future<void> addQueueItems(List<MediaItem> mediaItems) async {
    final sources = mediaItems
        .map((item) => AudioSource.uri(
              Uri.parse(item.extras!['url'] as String),
              tag: item,
            ))
        .toList();
    await _playlist.addAll(sources);
    queue.add(queue.value..addAll(mediaItems));
  }

  @override
  Future<void> play() => _player.play();

  @override
  Future<void> pause() => _player.pause();

  @override
  Future<void> stop() async {
    await _player.stop();
    return super.stop();
  }

  @override
  Future<void> seek(Duration position) => _player.seek(position);

  @override
  Future<void> skipToQueueItem(int index) =>
      _player.seek(Duration.zero, index: index);

  @override
  Future<void> skipToNext() => _player.seekToNext();

  @override
  Future<void> skipToPrevious() => _player.seekToPrevious();

  @override
  Future<void> setRepeatMode(AudioServiceRepeatMode mode) async {
    _player.setLoopMode({
      AudioServiceRepeatMode.none:  LoopMode.off,
      AudioServiceRepeatMode.one:   LoopMode.one,
      AudioServiceRepeatMode.group: LoopMode.all,
      AudioServiceRepeatMode.all:   LoopMode.all,
    }[mode]!);
  }

  @override
  Future<void> setShuffleMode(AudioServiceShuffleMode mode) async {
    if (mode == AudioServiceShuffleMode.none) {
      _player.setShuffleModeEnabled(false);
    } else {
      await _player.shuffle();
      _player.setShuffleModeEnabled(true);
    }
  }

  @override
  Future<void> customAction(
      String name, [Map<String, dynamic>? extras]) async {
    if (name == 'setSpeed') {
      _player.setSpeed((extras!['speed'] as num).toDouble());
    }
  }
}
```

### main.dart — 전역 초기화

```dart
late AudioHandler audioHandler;

Future<void> main() async {
  WidgetsFlutterBinding.ensureInitialized();

  audioHandler = await AudioService.init(
    builder: () => MyAudioHandler(),
    config: const AudioServiceConfig(
      androidNotificationChannelId: 'com.example.myapp.channel',
      androidNotificationChannelName: 'Music Playback',
      androidNotificationOngoing: true,
      androidStopForegroundOnPause: true,
      notificationColor: Color(0xFF2196F3),
    ),
  );

  runApp(const MyApp());
}
```

### UI에서 audioHandler 사용

```dart
// 플레이리스트 추가 후 재생 시작
await audioHandler.addQueueItems([
  MediaItem(
    id: 'track_01',
    title: 'Rainy Day',
    artist: 'Chill Studio',
    duration: const Duration(minutes: 3, seconds: 42),
    artUri: Uri.parse('https://cdn.example.com/art/rainy_day.jpg'),
    extras: {'url': 'https://cdn.example.com/audio/rainy_day.mp3'},
  ),
]);
await audioHandler.play();

// 재생 속도 변경 (커스텀 액션)
await audioHandler.customAction('setSpeed', {'speed': 1.5});
```

---

## 코드 예제 3: audio_session으로 오디오 포커스 관리

```dart
import 'package:audio_session/audio_session.dart';

Future<void> configureAudioSession(AudioPlayer player) async {
  final session = await AudioSession.instance;
  await session.configure(const AudioSessionConfiguration.music());

  // 전화·알람 등 인터럽션 처리
  session.interruptionEventStream.listen((event) {
    if (event.begin) {
      switch (event.type) {
        case AudioInterruptionType.duck:
          player.setVolume(0.3);   // 볼륨 낮춤 (Ducking)
          break;
        case AudioInterruptionType.pause:
        case AudioInterruptionType.unknown:
          player.pause();
          break;
      }
    } else {
      switch (event.type) {
        case AudioInterruptionType.duck:
          player.setVolume(1.0);
          break;
        case AudioInterruptionType.pause:
          if (event.shouldResume) player.play();
          break;
        case AudioInterruptionType.unknown:
          break;
      }
    }
  });

  // 헤드폰 분리 시 자동 일시 정지 (Android BecomingNoisy)
  session.becomingNoisyEventStream.listen((_) => player.pause());

  // 오디오 디바이스 변경 이벤트
  session.devicesChangedEventStream.listen((event) {
    final removed = event.devicesRemoved;
    if (removed.any((d) => d.type == AudioDeviceType.wiredHeadset ||
                           d.type == AudioDeviceType.bluetoothA2dp)) {
      player.pause();
    }
  });
}
```

---

## 주의사항 및 팁

### Android 14+ foregroundServiceType 필수

Android 14(API 34)부터 포그라운드 서비스 유형 선언 없이 미디어 재생 서비스를 시작하면 `ForegroundServiceStartNotAllowedException`이 발생한다. 매니페스트의 `<service>` 태그에 반드시 `android:foregroundServiceType="mediaPlayback"`을 추가한다.

### iOS 백그라운드 전환 시 오디오 끊김

iOS에서 앱 전환 직후 짧은 끊김이 발생하는 경우, `AVAudioSession`이 활성화되기 전에 시스템이 오디오 스트림을 닫는 것이 원인이다. `audio_service`의 `AudioServiceConfig.preloadArtwork: true` 옵션으로 알림 아트워크를 미리 로드하고, `AudioSession.instance.setActive(true)`를 명시 호출하면 개선된다.

### 메모리 관리

`AudioPlayer`는 무거운 네이티브 리소스를 보유한다. `audio_service` 핸들러는 싱글톤으로 관리되므로 `AudioPlayer`를 핸들러 내부에 딱 하나만 생성하고, 앱 종료(stop) 시 `dispose()`를 호출한다. 핸들러 외부에서 별도의 `AudioPlayer`를 생성하면 자원 충돌이 일어날 수 있다.

### 버퍼링 지연 최소화

`ConcatenatingAudioSource`의 `useLazyPreparation: true`는 현재 트랙 재생 중에 다음 트랙을 백그라운드에서 준비해 갭리스(gapless) 재생을 실현한다. 오디오 이펙트(`AndroidLoudnessEnhancer` 등)를 불필요하게 활성화하면 초기 로딩 시간이 늘어나므로 필요한 경우에만 적용한다.

### 동적 큐 조작

`ConcatenatingAudioSource`는 재생 중에도 `insert()`, `removeAt()`, `move()`로 트랙을 실시간 조작할 수 있다. 현재 재생 중인 트랙을 제거하면 `just_audio`는 자동으로 다음 트랙으로 넘어간다.

```dart
// 현재 재생 트랙 바로 다음에 긴급 삽입
final nextIdx = (_player.currentIndex ?? 0) + 1;
await _playlist.insert(nextIdx, newAudioSource);
```

### Riverpod·Provider와 통합

`audioHandler`를 전역 변수 대신 Riverpod의 `Provider`로 노출하면 테스트에서 Mock으로 교체하기 쉽다.

```dart
final audioHandlerProvider = Provider<AudioHandler>((ref) {
  throw UnimplementedError('Override in ProviderScope');
});
```

---

## 참고 자료
- [just_audio — pub.dev](https://pub.dev/packages/just_audio)
- [audio_service — pub.dev](https://pub.dev/packages/audio_service)
- [just_audio_background — pub.dev](https://pub.dev/packages/just_audio_background)
- [audio_session — pub.dev](https://pub.dev/packages/audio_session)
