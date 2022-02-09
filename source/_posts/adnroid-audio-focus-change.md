---
title: 实测Android音频的焦点获取和归还
date: 2022-02-09 09:26:09
tags: [Audio]
categories: Android
description:  "最近老板想在产品中的短视频后者直播播放的时候对于手机中的音乐播放器进行暂停播放，并且退出视频播放后手机的音乐播放器还能继续播放之前的音乐。"
---

# 实测Android音频的焦点获取和归还
![在这里插入图片描述](https://img-blog.csdnimg.cn/fe17c3d52ca54b74a3a323df14087e3a.png?x-oss-process=image/watermark,type_d3F5LXplbmhlaQ,shadow_50,text_Q1NETiBA6Z2Z6buY5Yqg6L29,size_20,color_FFFFFF,t_70,g_se,x_16#pic_center)

## 前言

最近老板想在产品中的短视频后者直播播放的时候对于手机中的音乐播放器进行暂停播放，并且退出视频播放后手机的音乐播放器还能继续播放之前的音乐。

先试试微信，emmm，确实可以。

[Android官网：管理音频焦点](https://developer.android.com/guide/topics/media-apps/audio-focus?hl=zh-cn)

## 官网管理音频焦点准则：

- 在即将开始播放之前调用 `requestAudioFocus()`，并验证调用是否返回 `AUDIOFOCUS_REQUEST_GRANTED`。如果按照本指南中的说明设计应用，则应在媒体会话的 `onPlay()` 回调中调用 `requestAudioFocus()`。
- 在其他应用获得音频焦点时，停止或暂停播放，或降低音量。
- 播放停止后，放弃音频焦点。

## 不同版本音频焦点的处理方式不太相同：

- 从 Android 2.2（API 级别 8）开始，应用通过调用 `requestAudioFocus()` 和 `abandonAudioFocus()` 来管理音频焦点。应用还必须为这两个调用注册 `AudioManager.OnAudioFocusChangeListener`，以便接收回调并管理自己的音量。
- 对于以 Android 5.0（API 级别 21）及更高版本为目标平台的应用，音频应用应使用 `AudioAttributes` 来描述应用正在播放的音频类型。例如，播放语音的应用应指定 `CONTENT_TYPE_SPEECH`。

- 面向 Android 8.0（API 级别 26）或更高版本的应用应使用 `requestAudioFocus()` 方法，该方法会接受 `AudioFocusRequest` 参数。`AudioFocusRequest` 包含有关应用的音频上下文和功能的信息。系统使用这些信息来自动管理音频焦点的得到和失去。

## API介绍

处理音频焦点都是通过AudioManager这个类，如下是获得该类实例的方法：
 `AudioManager am = (AudioManager) mContext.getSystemService(Context.AUDIO_SERVICE);`

> `requestAudioFocus()`//用于申请音频焦点
> `abandonAudioFocus()` //用于释放音频焦点
> `AudioManager.OnAudioFocusChangeListener` 接口，提供了 `onAudioFocusChange()` 方法来监听音频焦点变化

- `requestAudioFocus(OnAudioFocusChangeListener l, int streamType, int durationHint)`参数：

  - `AudioManager.OnAudioFocusChangeListener l`：
     用于监听音频焦点变化，从而可以进行适当的操作，例如暂停播放等。

  - `streamType` ：
     申请音频焦点处理的音频类型，例如，当播放音乐时，可以传入 `STREAM_MUSIC` ；当播放铃声时，可以传入 `STREAM_RING` 。表中列出了一些的可选值：

     | 类型                | 含义     | 值   |
     | ------------------- | -------- | ---- |
     | STREAM_VOICE_CALL   | 通话     | 0    |
     | STREAM_SYSTEM       | 系统     | 1    |
     | STREAM_RING         | 铃声     | 2    |
     | STREAM_MUSIC        | 音乐     | 3    |
     | STREAM_ALARM        | 闹铃     | 4    |
     | STREAM_NOTIFICATION | 系统通知 | 5    |
     | ...                 | ...      | ...  |
     
  - `durationHint` (PS:重要参数)：
     可选值有以下五个：
     (1)  `AUDIOFOCUS_GAIN`： 此参数表示希望申请一个永久的音频焦点，并且希望上一个持有音频焦点的App停止播放；例如在需要播放音乐时。
     (2) `AUDIOFOCUS_GAIN_TRANSIENT`：表示申请一个短暂的音频焦点，并且马上就会被释放，此时希望上一个持有音频焦点的App暂停播放。例如播放一个提醒声音。
     (3) `AUDIOFOCUS_GAIN_TRANSIENT_MAY_DUCK`：效果同 `AUDIOFOCUS_GAIN_TRANSIENT` ，只是希望上一个持有焦点的App减小其播放声音(但仍可以播放)，此时会混音播放。例如导航播报。
     (4) `AUDIOFOCUS_GAIN_TRANSIENT_EXCLUSIVE`： 表示申请一个短暂的音频焦点，并且会希望系统不要播放任何突然的声音（例如通知，提醒等），例如用户在录音。
     
  - 返回值：
     `AUDIOFOCUS_REQUEST_GRANTED` 或者 `AUDIOFOCUS_REQUEST_FAILED` 。
  
- `abandonAudioFocus(OnAudioFocusChangeListener l)` 参数通上 `AudioManager.OnAudioFocusChangeListener` .

- `AudioManager.OnAudioFocusChangeListener` :当音频焦点发生变化时进行 `onAudioFocusChange(int focusChange)` 方法的回调；

```java
new AudioManager.OnAudioFocusChangeListener() {
  @Override
  public void onAudioFocusChange(int focusChange) {
    switch (focusChange) {
      case AudioManager.AUDIOFOCUS_GAIN:
        // TBD 继续播放
        break;
      case AudioManager.AUDIOFOCUS_LOSS:
        // TBD 停止播放
        break;
      case AudioManager.AUDIOFOCUS_LOSS_TRANSIENT:
        // TBD 暂停播放
        break;
      case AudioManager.AUDIOFOCUS_LOSS_TRANSIENT_CAN_DUCK:
        // TBD 混音播放 
        break;
      default:
        break;
    }
  }
};
```

## 实测代码：

核心是需要把 `AudioManager.AUDIOFOCUS_GAIN` 改为 `AudioManager.AUDIOFOCUS_GAIN_TRANSIENT`

```java
public class MainActivity extends Activity {

    private AudioManager mAudioManager;
    private AudioFocusRequest mFocusRequest;
    private AudioManager.OnAudioFocusChangeListener mListener;
    private AudioAttributes mAttribute;
    @SuppressLint("HandlerLeak")
    private Handler mHandler = new Handler() {
        @Override
        public void handleMessage(@NonNull Message msg) {
            super.handleMessage(msg);
        }
    };

    @Override
    protected void onCreate(Bundle savedInstanceState) {
        super.onCreate(savedInstanceState);
        setContentView(R.layout.activity_main);
        mAudioManager = (AudioManager) getSystemService(Context.AUDIO_SERVICE);
        mListener = new AudioManager.OnAudioFocusChangeListener() {
            @Override
            public void onAudioFocusChange(int focusChange) {
                switch (focusChange) {
                    case AudioManager.AUDIOFOCUS_GAIN:
                       // TBD 继续播放
                        break;
                    case AudioManager.AUDIOFOCUS_LOSS:
                       // TBD 停止播放
                        break;
                    case AudioManager.AUDIOFOCUS_LOSS_TRANSIENT:
                        // TBD 暂停播放
                        break;
                    case AudioManager.AUDIOFOCUS_LOSS_TRANSIENT_CAN_DUCK:
                        // TBD 混音播放 
                        break;
                    default:
                        break;
                }

            }
        };
        //android 版本 5.0
        if (Build.VERSION.SDK_INT >= Build.VERSION_CODES.LOLLIPOP) {
            mAttribute = new AudioAttributes.Builder()
                    .setUsage(AudioAttributes.USAGE_MEDIA)
                    .setContentType(AudioAttributes.CONTENT_TYPE_MUSIC)
                    .build();
        }
        //android 版本 8.0
        if (android.os.Build.VERSION.SDK_INT >= android.os.Build.VERSION_CODES.O) {
            mFocusRequest = new AudioFocusRequest.Builder(AudioManager.AUDIOFOCUS_GAIN_TRANSIENT)
                    .setWillPauseWhenDucked(true)
                    .setAcceptsDelayedFocusGain(true)
                    .setOnAudioFocusChangeListener(mListener, mHandler)
                    .setAudioAttributes(mAttribute)
                    .build();
        }
    }
    private void requestAudioFocus() {
        if (Build.VERSION.SDK_INT >= Build.VERSION_CODES.O) {
					mAudioManager.requestAudioFocus(mFocusRequest);
        } else {
          mAudioManager.requestAudioFocus(mListener, AudioManager.STREAM_MUSIC, AudioManager.AUDIOFOCUS_GAIN_TRANSIENT);
        }
    }
    private void abandonAudioFocus() {
        if (Build.VERSION.SDK_INT >= Build.VERSION_CODES.O) {
            mAudioManager.abandonAudioFocusRequest(mFocusRequest);
        } else {
          mAudioManager.abandonAudioFocus(mListener);
        }

    }
}
```

## 参考：

https://segmentfault.com/a/1190000022234509

https://www.jianshu.com/p/26ea60c499a7


文章到这里就全部讲述完啦，若有其他需要交流的可以留言哦~！~！