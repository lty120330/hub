name: apk

# ============ 一台手机就能用的云编译流水线（v2：修卡死） ============
# v2 改动：
#   1) 模型加载从主线程挪到工作线程 —— v1 卡死的主因
#   2) 推理输入 640 -> 416，负载降到 1/2.4
#   3) 通知栏加「停止」动作，卡住时下拉点一下就停
#   4) 慢帧熔断：连续 8 帧超过 800ms 自动停服务
#   5) 自适应降帧：按实测耗时在 15/8/5 fps 之间自动切
#
# 用法：仓库 -> Add file -> Create new file -> 文件名 .github/workflows/build.yml
#       粘贴全文 -> Commit changes -> Actions 等 6~10 分钟 -> Artifacts 下 apk

on:
  workflow_dispatch:
  push:
    branches: [main, master]

jobs:
  apk:
    runs-on: ubuntu-22.04
    steps:
      - uses: actions/checkout@v4

      - uses: actions/setup-java@v4
        with:
          distribution: temurin
          java-version: '17'

      - name: Android SDK
        run: |
          SDK="$ANDROID_HOME/cmdline-tools/latest/bin/sdkmanager"
          yes | "$SDK" --licenses > /dev/null 2>&1 || true
          "$SDK" "platforms;android-34" "build-tools;34.0.0" > /dev/null

      - name: 生成工程
        run: |
          mkdir -p app/src/main/java/com/lc/hud app/src/main/assets

          cat > settings.gradle <<'EOF_SET'
          pluginManagement {
              repositories { google(); mavenCentral(); gradlePluginPortal() }
          }
          dependencyResolutionManagement {
              repositories { google(); mavenCentral() }
          }
          rootProject.name = 'hud'
          include ':app'
          EOF_SET

          cat > build.gradle <<'EOF_ROOT'
          plugins {
              id 'com.android.application' version '8.5.2' apply false
          }
          EOF_ROOT

          cat > gradle.properties <<'EOF_PROP'
          org.gradle.jvmargs=-Xmx2048m -Dfile.encoding=UTF-8
          android.useAndroidX=true
          android.nonFinalResIds=false
          EOF_PROP

          cat > app/build.gradle <<'EOF_APP'
          plugins { id 'com.android.application' }

          android {
              namespace 'com.lc.hud'
              compileSdk 34

              defaultConfig {
                  applicationId 'com.lc.hud'
                  minSdk 26
                  targetSdk 34
                  versionCode 3
                  versionName '1.2'
                  ndk { abiFilters 'arm64-v8a' }
              }

              compileOptions {
                  sourceCompatibility JavaVersion.VERSION_17
                  targetCompatibility JavaVersion.VERSION_17
              }

              buildTypes {
                  debug { }
                  release { minifyEnabled false }
              }

              lint { abortOnError false }
              packaging { jniLibs { useLegacyPackaging = false } }
          }

          dependencies {
              implementation 'com.microsoft.onnxruntime:onnxruntime-android:1.19.2'
          }
          EOF_APP

          cat > app/src/main/AndroidManifest.xml <<'EOF_MAN'
          <?xml version="1.0" encoding="utf-8"?>
          <manifest xmlns:android="http://schemas.android.com/apk/res/android">
              <uses-permission android:name="android.permission.SYSTEM_ALERT_WINDOW" />
              <uses-permission android:name="android.permission.FOREGROUND_SERVICE" />
              <uses-permission android:name="android.permission.FOREGROUND_SERVICE_MEDIA_PROJECTION" />
              <uses-permission android:name="android.permission.POST_NOTIFICATIONS" />

              <application
                  android:allowBackup="false"
                  android:label="hud"
                  android:hardwareAccelerated="true"
                  android:largeHeap="true"
                  android:theme="@android:style/Theme.DeviceDefault.NoActionBar">

                  <activity
                      android:name=".MainActivity"
                      android:exported="true"
                      android:configChanges="orientation|screenSize|screenLayout|keyboardHidden"
                      android:launchMode="singleTask">
                      <intent-filter>
                          <action android:name="android.intent.action.MAIN" />
                          <category android:name="android.intent.category.LAUNCHER" />
                      </intent-filter>
                  </activity>

                  <service
                      android:name=".CapService"
                      android:exported="false"
                      android:foregroundServiceType="mediaProjection" />
              </application>
          </manifest>
          EOF_MAN

          cat > app/src/main/java/com/lc/hud/MainActivity.java <<'EOF_MAIN'
          package com.lc.hud;

          import android.app.Activity;
          import android.content.Intent;
          import android.media.projection.MediaProjectionConfig;
          import android.media.projection.MediaProjectionManager;
          import android.net.Uri;
          import android.os.Build;
          import android.os.Bundle;
          import android.provider.Settings;
          import android.view.View;
          import android.widget.Button;
          import android.widget.LinearLayout;
          import android.widget.ScrollView;
          import android.widget.SeekBar;
          import android.widget.Switch;
          import android.widget.TextView;
          import android.widget.Toast;

          public class MainActivity extends Activity {

              private static final int REQ_OV = 11;
              private static final int REQ_CP = 12;
              private TextView status;

              @Override
              protected void onCreate(Bundle b) {
                  super.onCreate(b);
                  LinearLayout root = new LinearLayout(this);
                  root.setOrientation(LinearLayout.VERTICAL);
                  root.setPadding(dp(18), dp(26), dp(18), dp(18));
                  root.setBackgroundColor(0xFF12161C);

                  status = new TextView(this);
                  status.setTextColor(0xFF9BE7A8);
                  status.setTextSize(13f);
                  root.addView(status);

                  TextView hint = new TextView(this);
                  hint.setText("1) 先给悬浮窗权限，确认 overlay=OK\n"
                          + "2) 点开始，等 3~10 秒加载模型\n"
                          + "3) 卡住/屏幕点不动：下拉通知栏点「停止」\n"
                          + "4) 关掉「防自反馈」可以测是不是覆盖层导致的触摸问题");
                  hint.setTextColor(0xFF6E7A87);
                  hint.setTextSize(12f);
                  root.addView(hint);

                  root.addView(button("1  悬浮窗权限", new Runnable() {
                      public void run() { overlayPerm(); }
                  }));
                  root.addView(button("2  开始透视", new Runnable() {
                      public void run() { request(); }
                  }));
                  root.addView(button("3  停止", new Runnable() {
                      public void run() { stop(); }
                  }));

                  root.addView(slider("置信度 %", 5, 95, Math.round(Cfg.conf * 100), true));
                  root.addView(slider("捕获缩放 %", 30, 100, Math.round(Cfg.capScale * 100), false));

                  Switch sw = new Switch(this);
                  sw.setText("显示 FPS");
                  sw.setTextColor(0xFFFFFFFF);
                  sw.setChecked(Cfg.showFps);
                  sw.setOnCheckedChangeListener(new android.widget.CompoundButton.OnCheckedChangeListener() {
                      public void onCheckedChanged(android.widget.CompoundButton v, boolean on) {
                          Cfg.showFps = on;
                      }
                  });
                  root.addView(sw);

                  Switch sec = new Switch(this);
                  sec.setText("防自反馈 FLAG_SECURE（有触摸问题就关掉它）");
                  sec.setTextColor(0xFFFFFFFF);
                  sec.setChecked(Cfg.secure);
                  sec.setOnCheckedChangeListener(new android.widget.CompoundButton.OnCheckedChangeListener() {
                      public void onCheckedChanged(android.widget.CompoundButton v, boolean on) {
                          Cfg.secure = on;
                      }
                  });
                  root.addView(sec);

                  ScrollView sv = new ScrollView(this);
                  sv.addView(root);
                  setContentView(sv);
                  refresh();
              }

              @Override
              protected void onResume() {
                  super.onResume();
                  refresh();
              }

              private void refresh() {
                  boolean ok = Settings.canDrawOverlays(this);
                  status.setText("overlay=" + (ok ? "OK" : "缺失")
                          + "   conf=" + Cfg.conf
                          + "   in=" + Cfg.INPUT
                          + "   scale=" + Math.round(Cfg.capScale * 100) + "%"
                          + "   fps<=" + Cfg.maxFps);
              }

              private void overlayPerm() {
                  startActivityForResult(new Intent(
                          Settings.ACTION_MANAGE_OVERLAY_PERMISSION,
                          Uri.parse("package:" + getPackageName())), REQ_OV);
              }

              private void request() {
                  if (!Settings.canDrawOverlays(this)) {
                      Toast.makeText(this, "先给悬浮窗权限", Toast.LENGTH_SHORT).show();
                      return;
                  }
                  MediaProjectionManager m =
                          (MediaProjectionManager) getSystemService(MEDIA_PROJECTION_SERVICE);
                  Intent i;
                  if (Build.VERSION.SDK_INT >= 34) {
                      i = m.createScreenCaptureIntent(
                              MediaProjectionConfig.createConfigForDefaultDisplay());
                  } else {
                      i = m.createScreenCaptureIntent();
                  }
                  startActivityForResult(i, REQ_CP);
              }

              private void stop() {
                  Intent i = new Intent(this, CapService.class);
                  i.setAction(CapService.ACT_STOP);
                  startService(i);
              }

              @Override
              protected void onActivityResult(int req, int res, Intent data) {
                  super.onActivityResult(req, res, data);
                  if (req == REQ_CP && res == RESULT_OK && data != null) {
                      Toast.makeText(this, "模型加载中，请等 5 秒左右", Toast.LENGTH_LONG).show();
                      Intent i = new Intent(this, CapService.class);
                      i.setAction(CapService.ACT_START);
                      i.putExtra(CapService.EX_CODE, res);
                      i.putExtra(CapService.EX_DATA, data);
                      if (Build.VERSION.SDK_INT >= 26) startForegroundService(i); else startService(i);
                  }
                  refresh();
              }

              private Button button(String t, final Runnable r) {
                  Button b = new Button(this);
                  b.setText(t);
                  b.setAllCaps(false);
                  b.setOnClickListener(new View.OnClickListener() {
                      public void onClick(View v) { r.run(); }
                  });
                  return b;
              }

              private View slider(String label, final int min, final int max,
                                  int cur, final boolean isConf) {
                  LinearLayout box = new LinearLayout(this);
                  box.setOrientation(LinearLayout.VERTICAL);
                  final TextView tv = new TextView(this);
                  tv.setTextColor(0xFFFFFFFF);
                  tv.setTextSize(14f);
                  tv.setText(label + ": " + cur);
                  box.addView(tv);

                  SeekBar sb = new SeekBar(this);
                  sb.setMax(max - min);
                  sb.setProgress(cur - min);
                  sb.setOnSeekBarChangeListener(new SeekBar.OnSeekBarChangeListener() {
                      public void onProgressChanged(SeekBar s, int p, boolean u) {
                          int v = p + min;
                          tv.setText(label + ": " + v);
                          if (isConf) Cfg.conf = v / 100f; else Cfg.capScale = v / 100f;
                          refresh();
                      }
                      public void onStartTrackingTouch(SeekBar s) { }
                      public void onStopTrackingTouch(SeekBar s) { }
                  });
                  box.addView(sb, new LinearLayout.LayoutParams(-1, -2));
                  return box;
              }

              private int dp(int v) {
                  return (int) (v * getResources().getDisplayMetrics().density);
              }
          }
          EOF_MAIN

          cat > app/src/main/java/com/lc/hud/CapService.java <<'EOF_CAP'
          package com.lc.hud;

          import android.app.Notification;
          import android.app.NotificationChannel;
          import android.app.NotificationManager;
          import android.app.PendingIntent;
          import android.app.Service;
          import android.content.Context;
          import android.content.Intent;
          import android.content.pm.ServiceInfo;
          import android.content.res.Configuration;
          import android.graphics.Bitmap;
          import android.graphics.Canvas;
          import android.graphics.PixelFormat;
          import android.graphics.Rect;
          import android.hardware.display.DisplayManager;
          import android.hardware.display.VirtualDisplay;
          import android.media.Image;
          import android.media.ImageReader;
          import android.media.projection.MediaProjection;
          import android.media.projection.MediaProjectionManager;
          import android.os.Build;
          import android.os.Handler;
          import android.os.HandlerThread;
          import android.os.IBinder;
          import android.os.Looper;
          import android.os.SystemClock;
          import android.util.DisplayMetrics;
          import android.view.Gravity;
          import android.view.WindowManager;
          import android.widget.Toast;

          import java.nio.ByteBuffer;
          import java.util.concurrent.atomic.AtomicBoolean;

          /**
           * 投屏捕获 -> 推理 -> 悬浮窗。全程不碰游戏进程。
           *
           * v1 卡死原因：new Ort(...) 在主线程加载 12MB 模型 + ORT 图优化，
           * 手机要 5~15 秒，把 UI 线程占死。v2 全部挪到工作线程。
           */
          public class CapService extends Service {

              public static final String ACT_START = "hud.start";
              public static final String ACT_STOP = "hud.stop";
              public static final String EX_CODE = "code";
              public static final String EX_DATA = "data";

              private static final int NID = 1517;
              private static final String CH = "hud";

              private MediaProjection proj;
              private VirtualDisplay vd;
              private ImageReader reader;
              private HandlerThread ht;
              private Handler h;
              private Handler main;
              private Ort ort;
              private HudView view;
              private WindowManager wm;

              private final AtomicBoolean busy = new AtomicBoolean(false);
              private volatile boolean alive = false;
              private volatile boolean stopping = false;

              private int capW = 1, capH = 1;
              private Bitmap pad, frame;
              private long lastTs = 0L;
              private float msEma = 0f;
              private int slowFrames = 0;

              private final MediaProjection.Callback cb = new MediaProjection.Callback() {
                  @Override public void onStop() {
                      alive = false;
                      stopSelf();
                  }
              };

              @Override public IBinder onBind(Intent i) { return null; }

              @Override public void onCreate() {
                  super.onCreate();
                  fg();
                  main = new Handler(Looper.getMainLooper());
                  ht = new HandlerThread("pipe");
                  ht.start();
                  h = new Handler(ht.getLooper());
                  wm = (WindowManager) getSystemService(Context.WINDOW_SERVICE);
              }

              @Override public int onStartCommand(Intent it, int f, int s) {
                  if (it != null && ACT_STOP.equals(it.getAction())) {
                      stopSelf();
                      return START_NOT_STICKY;
                  }
                  if (it != null && ACT_START.equals(it.getAction())) {
                      int code = it.getIntExtra(EX_CODE, 0);
                      Intent data = it.getParcelableExtra(EX_DATA);
                      if (data != null) {
                          try {
                              start(code, data);
                          } catch (Throwable t) {
                              toast("启动失败: " + t.getClass().getSimpleName());
                              stopSelf();
                          }
                      }
                  }
                  return START_STICKY;
              }

              @Override public void onConfigurationChanged(Configuration c) {
                  super.onConfigurationChanged(c);
                  if (h != null) h.post(new Runnable() { public void run() { build(); } });
              }

              @Override public void onDestroy() {
                  alive = false;
                  stopping = true;
                  try {
                      if (reader != null) {
                          reader.setOnImageAvailableListener(null, null);
                          reader.close();
                          reader = null;
                      }
                      if (vd != null) { vd.release(); vd = null; }
                      if (proj != null) {
                          proj.unregisterCallback(cb);
                          proj.stop();
                          proj = null;
                      }
                      if (view != null) { wm.removeViewImmediate(view); view = null; }
                      if (ort != null) { ort.close(); ort = null; }
                  } catch (Throwable t) { }
                  if (ht != null) { ht.quitSafely(); ht = null; }
                  super.onDestroy();
              }

              /** 主线程只做轻活：拿 projection、挂悬浮窗。重活丢工作线程。 */
              private void start(int code, Intent data) {
                  if (proj != null) return;
                  MediaProjectionManager m =
                          (MediaProjectionManager) getSystemService(Context.MEDIA_PROJECTION_SERVICE);
                  proj = m.getMediaProjection(code, data);
                  proj.registerCallback(cb, main);
                  overlay();
                  alive = true;

                  h.post(new Runnable() {
                      public void run() {
                          try {
                              ort = new Ort(getAssets());
                          } catch (Throwable t) {
                              ort = null;
                          }
                          if (stopping) return;
                          if (ort == null) {
                              toast("模型加载失败");
                              stopSelf();
                              return;
                          }
                          toast("就绪");
                          build();
                      }
                  });
              }

              private void overlay() {
                  view = new HudView(this);
                  view.setFocusable(false);
                  view.setClickable(false);
                  int type = Build.VERSION.SDK_INT >= 26
                          ? WindowManager.LayoutParams.TYPE_APPLICATION_OVERLAY
                          : WindowManager.LayoutParams.TYPE_PHONE;
                  // 关键：不要 FLAG_LAYOUT_NO_LIMITS。它让窗口几何超出屏幕，
                  // 部分 ROM 上会把这个叠层当成整屏可触摸区，表现就是
                  // 「退出 App 后整块屏幕点不动」。宁可在挖孔区少画几像素。
                  int flags = WindowManager.LayoutParams.FLAG_NOT_FOCUSABLE
                          | WindowManager.LayoutParams.FLAG_NOT_TOUCHABLE
                          | WindowManager.LayoutParams.FLAG_LAYOUT_IN_SCREEN
                          | WindowManager.LayoutParams.FLAG_HARDWARE_ACCELERATED;
                  if (Cfg.secure) flags |= WindowManager.LayoutParams.FLAG_SECURE;
                  WindowManager.LayoutParams lp = new WindowManager.LayoutParams(
                          WindowManager.LayoutParams.MATCH_PARENT,
                          WindowManager.LayoutParams.MATCH_PARENT,
                          type, flags, PixelFormat.TRANSLUCENT);
                  lp.gravity = Gravity.TOP | Gravity.START;
                  if (Build.VERSION.SDK_INT >= 28) {
                      lp.layoutInDisplayCutoutMode =
                              WindowManager.LayoutParams.LAYOUT_IN_DISPLAY_CUTOUT_MODE_SHORT_EDGES;
                  }
                  wm.addView(view, lp);
              }

              private void build() {
                  if (proj == null) return;
                  DisplayMetrics dm = getResources().getDisplayMetrics();
                  int w = ((int) (dm.widthPixels * Cfg.capScale)) & ~1;
                  int hh = ((int) (dm.heightPixels * Cfg.capScale)) & ~1;
                  if (w < 240) w = 240;
                  if (hh < 240) hh = 240;
                  capW = w;
                  capH = hh;

                  if (reader != null) {
                      reader.setOnImageAvailableListener(null, null);
                      reader.close();
                  }
                  reader = ImageReader.newInstance(capW, capH, PixelFormat.RGBA_8888, 2);
                  reader.setOnImageAvailableListener(new ImageReader.OnImageAvailableListener() {
                      public void onImageAvailable(ImageReader r) { drain(r); }
                  }, h);

                  int dpi = dm.densityDpi;
                  int flag = DisplayManager.VIRTUAL_DISPLAY_FLAG_AUTO_MIRROR;
                  if (vd == null) {
                      vd = proj.createVirtualDisplay("mirror", capW, capH, dpi, flag,
                              reader.getSurface(), null, h);
                  } else {
                      vd.setSurface(null);
                      vd.resize(capW, capH, dpi);
                      vd.setSurface(reader.getSurface());
                  }
                  if (view != null) view.frameSize(capW, capH);
              }

              private void drain(ImageReader r) {
                  if (!alive) return;
                  if (!busy.compareAndSet(false, true)) {
                      try {
                          Image i = r.acquireLatestImage();
                          if (i != null) i.close();
                      } catch (Throwable t) { }
                      return;
                  }
                  Image img = null;
                  try {
                      img = r.acquireLatestImage();
                      if (img != null) handle(img);
                  } catch (Throwable t) {
                  } finally {
                      if (img != null) {
                          try { img.close(); } catch (Throwable t) { }
                      }
                      busy.set(false);
                  }
              }

              private void handle(Image img) {
                  long now = SystemClock.elapsedRealtime();
                  int interval = 1000 / Math.max(3, Cfg.maxFps);
                  if (now - lastTs < interval) return;
                  lastTs = now;

                  Image.Plane[] planes = img.getPlanes();
                  if (planes.length == 0) return;
                  Image.Plane p = planes[0];
                  int pixelStride = p.getPixelStride();
                  int rowPadding = p.getRowStride() - pixelStride * capW;
                  int paddedW = capW + Math.max(0, rowPadding) / pixelStride;

                  if (pad == null || pad.getWidth() != paddedW || pad.getHeight() != capH) {
                      pad = Bitmap.createBitmap(paddedW, capH, Bitmap.Config.ARGB_8888);
                  }
                  ByteBuffer buf = p.getBuffer();
                  buf.rewind();
                  pad.copyPixelsFromBuffer(buf);

                  if (frame == null || frame.getWidth() != capW || frame.getHeight() != capH) {
                      frame = Bitmap.createBitmap(capW, capH, Bitmap.Config.ARGB_8888);
                  }
                  Canvas c = new Canvas(frame);
                  c.drawBitmap(pad, new Rect(0, 0, capW, capH),
                          new Rect(0, 0, capW, capH), null);

                  Ort d = ort;
                  if (d == null) return;
                  long t0 = SystemClock.elapsedRealtimeNanos();
                  float[] flat = d.detect(frame);
                  float ms = (SystemClock.elapsedRealtimeNanos() - t0) / 1000000f;
                  msEma = msEma == 0f ? ms : msEma * 0.8f + ms * 0.2f;

                  // 自适应降帧：这台机器算力够多少就跑多少
                  if (msEma > 300f) Cfg.maxFps = 5;
                  else if (msEma > 150f) Cfg.maxFps = 8;
                  else if (Cfg.maxFps < 15) Cfg.maxFps = 15;

                  // 熔断：再慢就停，别把整机拖死
                  if (msEma > 800f) {
                      if (++slowFrames >= 8) {
                          toast("推理过慢，已自动停止");
                          stopSelf();
                          return;
                      }
                  } else {
                      slowFrames = 0;
                  }

                  if (view != null) view.submit(flat, msEma);
              }

              private void toast(final String s) {
                  Handler m = main != null ? main : new Handler(Looper.getMainLooper());
                  m.post(new Runnable() {
                      public void run() {
                          Toast.makeText(CapService.this, s, Toast.LENGTH_SHORT).show();
                      }
                  });
              }

              private void fg() {
                  NotificationManager nm =
                          (NotificationManager) getSystemService(Context.NOTIFICATION_SERVICE);
                  Notification.Builder b;
                  if (Build.VERSION.SDK_INT >= 26) {
                      nm.createNotificationChannel(new NotificationChannel(
                              CH, "run", NotificationManager.IMPORTANCE_LOW));
                      b = new Notification.Builder(this, CH);
                  } else {
                      b = new Notification.Builder(this);
                  }

                  // 卡住时唯一的救命通道：下拉通知栏点通知本体或「停止」
                  Intent si = new Intent(this, CapService.class);
                  si.setAction(ACT_STOP);
                  int piFlag = PendingIntent.FLAG_UPDATE_CURRENT;
                  if (Build.VERSION.SDK_INT >= 23) piFlag |= PendingIntent.FLAG_IMMUTABLE;
                  PendingIntent pi = PendingIntent.getService(this, 1, si, piFlag);

                  b.setContentTitle("hud 运行中");
                  b.setContentText("点这里停止");
                  b.setSmallIcon(android.R.drawable.ic_menu_view);
                  b.setOngoing(true);
                  b.setContentIntent(pi);
                  b.setDeleteIntent(pi);
                  b.addAction(0, "停止", pi);

                  Notification n = b.build();
                  if (Build.VERSION.SDK_INT >= 29) {
                      startForeground(NID, n, ServiceInfo.FOREGROUND_SERVICE_TYPE_MEDIA_PROJECTION);
                  } else {
                      startForeground(NID, n);
                  }
              }
          }
          EOF_CAP

          cat > app/src/main/java/com/lc/hud/Aux.java <<'EOF_AUX'
          package com.lc.hud;

          import android.content.res.AssetManager;
          import android.graphics.Bitmap;
          import android.graphics.Canvas;
          import android.graphics.Color;
          import android.graphics.Paint;
          import android.graphics.RectF;
          import android.os.SystemClock;
          import android.view.View;

          import java.io.ByteArrayOutputStream;
          import java.io.Closeable;
          import java.io.InputStream;
          import java.nio.FloatBuffer;
          import java.util.Arrays;
          import java.util.Collections;

          import ai.onnxruntime.OnnxTensor;
          import ai.onnxruntime.OrtEnvironment;
          import ai.onnxruntime.OrtSession;

          /** 运行参数。INPUT 必须和导出 onnx 时的 imgsz 一致。 */
          class Cfg {
              static final int INPUT = 416;
              static volatile float conf = 0.35f;
              static volatile float iou = 0.45f;
              static volatile float lw = 3f;
              static volatile int maxFps = 8;
              static volatile float capScale = 0.4f;
              static volatile boolean showFps = true;
              static volatile boolean secure = true;
              static volatile int color = 0xFF00E676;
          }

          /** ONNX Runtime YOLOv8 推理（输出 [1, 4+nc, anchors]） */
          class Ort implements Closeable {

              private final OrtEnvironment env;
              private final OrtSession sess;
              private final String inName;
              private final int sz = Cfg.INPUT;
              private final float[] in = new float[3 * Cfg.INPUT * Cfg.INPUT];
              private int[] px = new int[0];

              Ort(AssetManager am) throws Exception {
                  env = OrtEnvironment.getEnvironment();
                  byte[] model;
                  InputStream is = am.open("model.onnx");
                  try {
                      ByteArrayOutputStream bos = new ByteArrayOutputStream(1 << 25);
                      byte[] tmp = new byte[1 << 16];
                      int n;
                      while ((n = is.read(tmp)) > 0) bos.write(tmp, 0, n);
                      model = bos.toByteArray();
                  } finally {
                      is.close();
                  }
                  OrtSession.SessionOptions o = new OrtSession.SessionOptions();
                  int th = Math.max(1, Math.min(4, Runtime.getRuntime().availableProcessors() / 2));
                  o.setIntraOpNumThreads(th);
                  o.setInterOpNumThreads(1);
                  o.setOptimizationLevel(OrtSession.SessionOptions.OptLevel.ALL_OPT);
                  sess = env.createSession(model, o);
                  inName = sess.getInputNames().iterator().next();
              }

              /** 返回 [n, x,y,w,h,score, ...]，坐标是入参 bitmap 的像素空间 */
              float[] detect(Bitmap src) {
                  int w = src.getWidth();
                  int h = src.getHeight();
                  if (w <= 0 || h <= 0) return new float[0];

                  float s = Math.min(sz / (float) w, sz / (float) h);
                  int nw = Math.max(1, Math.round(w * s));
                  int nh = Math.max(1, Math.round(h * s));
                  Bitmap rs = Bitmap.createScaledBitmap(src, nw, nh, false);
                  int need = nw * nh;
                  if (px.length < need) px = new int[need];
                  rs.getPixels(px, 0, nw, 0, 0, nw, nh);
                  if (rs != src) rs.recycle();

                  Arrays.fill(in, 114f / 255f);
                  int padX = (sz - nw) / 2;
                  int padY = (sz - nh) / 2;
                  int cs = sz * sz;
                  for (int y = 0; y < nh; y++) {
                      int row = y * nw;
                      int dst = (y + padY) * sz + padX;
                      for (int x = 0; x < nw; x++) {
                          int c = px[row + x];
                          int idx = dst + x;
                          in[idx] = ((c >> 16) & 0xff) / 255f;
                          in[cs + idx] = ((c >> 8) & 0xff) / 255f;
                          in[2 * cs + idx] = (c & 0xff) / 255f;
                      }
                  }

                  try (OnnxTensor t = OnnxTensor.createTensor(
                          env, FloatBuffer.wrap(in), new long[]{1, 3, sz, sz});
                       OrtSession.Result r = sess.run(Collections.singletonMap(inName, t))) {
                      float[][][] out = (float[][][]) r.get(0).getValue();
                      return decode(out[0], s, padX, padY, w, h);
                  } catch (Throwable t) {
                      return new float[0];
                  }
              }

              private float[] decode(float[][] t, float s, int padX, int padY, int w, int h) {
                  int rows = t.length;
                  if (rows < 5) return new float[0];
                  int nc = rows - 4;
                  int na = t[0].length;

                  int cap = 128, m = 0;
                  float[] bx = new float[cap], by = new float[cap];
                  float[] bw = new float[cap], bh = new float[cap], bs = new float[cap];

                  for (int i = 0; i < na; i++) {
                      float best = 0f;
                      for (int c = 0; c < nc; c++) {
                          float v = t[4 + c][i];
                          if (v > best) best = v;
                      }
                      if (best < Cfg.conf) continue;

                      float x0 = (t[0][i] - t[2][i] / 2f - padX) / s;
                      float y0 = (t[1][i] - t[3][i] / 2f - padY) / s;
                      float w0 = t[2][i] / s;
                      float h0 = t[3][i] / s;
                      if (x0 < 0) { w0 += x0; x0 = 0; }
                      if (y0 < 0) { h0 += y0; y0 = 0; }
                      if (w0 < 4 || h0 < 4) continue;
                      if (x0 > w - 1 || y0 > h - 1) continue;

                      if (m == cap) {
                          cap <<= 1;
                          bx = Arrays.copyOf(bx, cap); by = Arrays.copyOf(by, cap);
                          bw = Arrays.copyOf(bw, cap); bh = Arrays.copyOf(bh, cap);
                          bs = Arrays.copyOf(bs, cap);
                      }
                      bx[m] = x0; by[m] = y0; bw[m] = w0; bh[m] = h0; bs[m] = best;
                      m++;
                  }
                  if (m == 0) return new float[0];

                  int[] order = new int[m];
                  for (int i = 0; i < m; i++) order[i] = i;
                  for (int i = 1; i < m; i++) {
                      int v = order[i], j = i - 1;
                      while (j >= 0 && bs[order[j]] < bs[v]) { order[j + 1] = order[j]; j--; }
                      order[j + 1] = v;
                  }

                  boolean[] dead = new boolean[m];
                  int[] picked = new int[m];
                  int keep = 0;
                  for (int i = 0; i < m; i++) {
                      int a = order[i];
                      if (dead[a]) continue;
                      picked[keep++] = a;
                      for (int j = i + 1; j < m; j++) {
                          int b = order[j];
                          if (dead[b]) continue;
                          float ix1 = Math.max(bx[a], bx[b]);
                          float iy1 = Math.max(by[a], by[b]);
                          float ix2 = Math.min(bx[a] + bw[a], bx[b] + bw[b]);
                          float iy2 = Math.min(by[a] + bh[a], by[b] + bh[b]);
                          float iw = ix2 - ix1, ih = iy2 - iy1;
                          if (iw <= 0 || ih <= 0) continue;
                          float inter = iw * ih;
                          float uni = bw[a] * bh[a] + bw[b] * bh[b] - inter + 1e-6f;
                          if (inter / uni > Cfg.iou) dead[b] = true;
                      }
                  }

                  float[] out = new float[1 + keep * 5];
                  out[0] = keep;
                  for (int i = 0; i < keep; i++) {
                      int a = picked[i];
                      int b = 1 + i * 5;
                      out[b] = bx[a];
                      out[b + 1] = by[a];
                      out[b + 2] = bw[a];
                      out[b + 3] = bh[a];
                      out[b + 4] = bs[a];
                  }
                  return out;
              }

              @Override public void close() {
                  try { sess.close(); } catch (Throwable t) { }
              }
          }

          /** 悬浮窗绘制层 */
          class HudView extends View {

              private final Paint box = new Paint(Paint.ANTI_ALIAS_FLAG);
              private final Paint txt = new Paint(Paint.ANTI_ALIAS_FLAG);
              private final RectF r = new RectF();

              private float[] data = new float[0];
              private int fw = 1, fh = 1;
              private float ms = 0f;
              private long lastData = 0L;
              private int frames = 0;
              private long fpsT0 = 0L;
              private float fps = 0f;

              HudView(android.content.Context c) {
                  super(c);
                  box.setStyle(Paint.Style.STROKE);
                  box.setStrokeCap(Paint.Cap.ROUND);
                  txt.setColor(Color.WHITE);
                  txt.setTextSize(30f);
                  txt.setFakeBoldText(true);
                  txt.setShadowLayer(3f, 0f, 1f, Color.BLACK);
                  setBackgroundColor(Color.TRANSPARENT);
              }

              void frameSize(int w, int h) {
                  fw = Math.max(1, w);
                  fh = Math.max(1, h);
              }

              void submit(float[] d, float inferMs) {
                  data = d;
                  ms = inferMs;
                  lastData = SystemClock.elapsedRealtime();
                  // 只在有新结果时重绘。v2 在这里自激成 60fps 全屏重绘，
                  // 而屏幕又被自己镜像，等于每帧再触发一次合成，把合成器打满，
                  // 表现就是「整机触摸发飘/点不动」。
                  postInvalidateOnAnimation();
              }

              @Override protected void onDraw(Canvas c) {
                  long now = SystemClock.elapsedRealtime();
                  frames++;
                  if (fpsT0 == 0) fpsT0 = now;
                  if (now - fpsT0 >= 1000) {
                      fps = frames * 1000f / (now - fpsT0);
                      frames = 0;
                      fpsT0 = now;
                  }

                  float sx = getWidth() / (float) fw;
                  float sy = getHeight() / (float) fh;
                  float d = getResources().getDisplayMetrics().density;
                  box.setStrokeWidth(Math.max(2f, Cfg.lw * d));
                  box.setColor(Cfg.color);

                  int n = data.length > 0 ? (int) data[0] : 0;
                  for (int i = 0; i < n; i++) {
                      int b = 1 + i * 5;
                      if (b + 4 >= data.length) break;
                      float sc = data[b + 4];
                      if (sc < Cfg.conf) continue;
                      r.set(data[b] * sx, data[b + 1] * sy,
                              (data[b] + data[b + 2]) * sx,
                              (data[b + 1] + data[b + 3]) * sy);
                      if (r.width() < 6f || r.height() < 6f) continue;
                      c.drawRoundRect(r, 3f, 3f, box);
                      txt.setColor(Cfg.color);
                      c.drawText(String.format("%.2f", sc),
                              r.left, Math.max(30f, r.top - 8f), txt);
                  }

                  if (Cfg.showFps) {
                      txt.setColor(Color.WHITE);
                      c.drawText(String.format("%.0f fps  %.1f ms  n=%d", fps, ms, n),
                              24f, 60f, txt);
                  }

                  // 不再自我续帧：没有新检测结果就停止重绘，
                  // 把 GPU 和 SurfaceFlinger 让给游戏本身。
              }
          }
          EOF_AUX

      - name: 占位模型（yolov8n 通用 80 类，仅用于验证管线）
        run: |
          python3 -m pip install -q --upgrade pip
          python3 -m pip install -q ultralytics onnx onnxsim
          python3 -c "from ultralytics import YOLO; YOLO('yolov8n.pt').export(format='onnx', imgsz=416, opset=12, simplify=True)"
          ls -la *.onnx
          cp yolov8n.onnx app/src/main/assets/model.onnx
          ls -la app/src/main/assets

      - uses: gradle/actions/setup-gradle@v3
        with:
          gradle-version: '8.7'

      - name: Build
        run: gradle :app:assembleDebug --no-daemon --stacktrace

      - uses: actions/upload-artifact@v4
        with:
          name: apk
          path: app/build/outputs/apk/debug/*.apk
          retention-days: 30
