title: Android二维码扫描的简单实现及源码分析
date: 2016-12-07 19:30:00
update: 2017-06-21 11:25:00
categories: 技术
tags: [Android]
---

二维码扫描最近两年简直是风靡移动互联网时代，尤其在国内发展神速。围绕条码扫码功能，首先说说通过本文你可以知道啥。一，如何在你的APP中快速集成一维码、二维码扫描功能；二，大概了解条码扫描功能的实现原理以及理解各个模块的代码。但是，本文不包含条码的编解码原理，如有兴趣，请自行查阅。

## 条码扫描功能的快速集成

默认你使用了Android Stuidio进行开发，直接使用这个开源项目[SimpleZXing](https://github.com/GuoJinyu/SimpleZXing),它是在ZXing库的作者为Android写的条码扫描APP Barcode Scanner的基础上优化而成的。
你可以通过简单的两步就可以实现条码扫描的功能。

* 1.添加项目依赖  

```gradle
    compile 'com.acker:simplezxing:1.5'
```

* 2.在你想调起条码扫描界面的地方(比如YourActivity)，调起二维码扫描界面CaptureActivity  

```java
    startActivityForResult(new Intent(YourActivity.this, CaptureActivity.class), CaptureActivity.REQ_CODE)
```

<!--more-->

然后就会打开如下这个界面：
<center>
![条码扫描界面][1]
</center>
将条码置于框内，扫描成功后会将解码得到的字符串返回给调起者，所以你只需要在你的Activity的onActivityResult()方法里拿到它进行后续操作即可。

当然SimpleZXing目前还支持一些设置项，包括摄像头是否开启曝光，扫码成功后是否震动，是否发声，闪光灯模式自动、常开、常关，屏幕自动旋转、横屏、竖屏。

同时，虽然该项目已经在manifest里申明了所需的照相机权限，但是在Android 6.0以上系统中你仍然需要自己处理动态权限管理。所以一个标准的使用方式如以下代码所示：

```java
    package com.acker.simplezxing.demo;

    import android.Manifest;
    import android.content.Intent;
    import android.content.pm.PackageManager;
    import android.os.Bundle;
    import android.support.annotation.NonNull;
    import android.support.v4.app.ActivityCompat;
    import android.support.v4.content.ContextCompat;
    import android.support.v7.app.AppCompatActivity;
    import android.view.View;
    import android.widget.Button;
    import android.widget.TextView;
    import android.widget.Toast;

    import com.acker.simplezxing.activity.CaptureActivity;

    public class MainActivity extends AppCompatActivity {

        private static final int REQ_CODE_PERMISSION = 0x1111;
        private TextView tvResult;

        @Override
        protected void onCreate(Bundle savedInstanceState) {
            super.onCreate(savedInstanceState);
            setContentView(R.layout.activity_main);
            tvResult = (TextView) findViewById(R.id.tv_result);
            Button btn = (Button) findViewById(R.id.btn_sm);
            btn.setOnClickListener(new View.OnClickListener() {
                @Override
                public void onClick(View view) {
                    // Open Scan Activity
                    if (ContextCompat.checkSelfPermission(MainActivity.this, Manifest.permission.CAMERA) != PackageManager.PERMISSION_GRANTED) {
                        // Do not have the permission of camera, request it.
                        ActivityCompat.requestPermissions(MainActivity.this, new String[]{Manifest.permission.CAMERA}, REQ_CODE_PERMISSION);
                    } else {
                        // Have gotten the permission
                        startCaptureActivityForResult();
                    }
                }
            });
        }

        private void startCaptureActivityForResult() {
            Intent intent = new Intent(MainActivity.this, CaptureActivity.class);
            Bundle bundle = new Bundle();
            bundle.putBoolean(CaptureActivity.KEY_NEED_BEEP, CaptureActivity.VALUE_BEEP);
            bundle.putBoolean(CaptureActivity.KEY_NEED_VIBRATION, CaptureActivity.VALUE_VIBRATION);
            bundle.putBoolean(CaptureActivity.KEY_NEED_EXPOSURE, CaptureActivity.VALUE_NO_EXPOSURE);
            bundle.putByte(CaptureActivity.KEY_FLASHLIGHT_MODE, CaptureActivity.VALUE_FLASHLIGHT_OFF);
            bundle.putByte(CaptureActivity.KEY_ORIENTATION_MODE, CaptureActivity.VALUE_ORIENTATION_AUTO);
            bundle.putBoolean(CaptureActivity.KEY_SCAN_AREA_FULL_SCREEN, CaptureActivity.VALUE_SCAN_AREA_FULL_SCREEN);
            bundle.putBoolean(CaptureActivity.KEY_NEED_SCAN_HINT_TEXT, CaptureActivity.VALUE_SCAN_HINT_TEXT);
            intent.putExtra(CaptureActivity.EXTRA_SETTING_BUNDLE, bundle);
            startActivityForResult(intent, CaptureActivity.REQ_CODE);
        }

        @Override
        public void onRequestPermissionsResult(int requestCode, @NonNull String[] permissions, @NonNull int[] grantResults) {
            super.onRequestPermissionsResult(requestCode, permissions, grantResults);
            switch (requestCode) {
                case REQ_CODE_PERMISSION: {
                    if (grantResults.length > 0 && grantResults[0] == PackageManager.PERMISSION_GRANTED) {
                        // User agree the permission
                        startCaptureActivityForResult();
                    } else {
                        // User disagree the permission
                        Toast.makeText(this, "You must agree the camera permission request before you use the code scan function", Toast.LENGTH_LONG).show();
                    }
                }
                break;
            }
        }

        @Override
        protected void onActivityResult(int requestCode, int resultCode, Intent data) {
            super.onActivityResult(requestCode, resultCode, data);
            switch (requestCode) {
                case CaptureActivity.REQ_CODE:
                    switch (resultCode) {
                        case RESULT_OK:
                            tvResult.setText(data.getStringExtra(CaptureActivity.EXTRA_SCAN_RESULT));  
                            //or do sth
                            break;
                        case RESULT_CANCELED:
                            if (data != null) {
                                // for some reason camera is not working correctly
                                tvResult.setText(data.getStringExtra(CaptureActivity.EXTRA_SCAN_RESULT));
                            }
                            break;
                    }
                    break;
            }
        }
    }
```

以上说了如何通过使用SimpleZXing开源项目来快速实现条码扫描功能，当然开发者可能会因为一些特定的需求需要修改某些地方的代码，如UI等等，那么下面我会带大家大致讲解一下这个开源项目的代码，使大家更进一步了解条码扫描的实现机制，同时方便大家在它基础之上进行修改。

## SimpleZXing关键代码分析

其实条码扫描的过程很容易理解，就是将摄像头捕捉到的预览帧数组进行处理，发现其中的一维码或二维码并进行解码。但是就是在摄像头捕捉数据的过程中有几个重要的地方需要大家注意。我们倒过来分析这个过程。

### 1.DecodeHandler.class中的decode()方法

```java
    /**
     * Decode the data within the viewfinder rectangle, and time how long it took. For efficiency,
     * reuse the same reader objects from one decode to the next.
     *
     * @param data   The YUV preview frame.
     * @param width  The width of the preview frame.
     * @param height The height of the preview frame.
     */
    private void decode(byte[] data, int width, int height) {
        long start = System.currentTimeMillis();
        if (width < height) {
            // portrait
            byte[] rotatedData = new byte[data.length];
            for (int x = 0; x < width; x++) {
                for (int y = 0; y < height; y++)
                    rotatedData[y * width + width - x - 1] = data[y + x * height];
            }
            data = rotatedData;
        }
        Result rawResult = null;
        PlanarYUVLuminanceSource source = activity.getCameraManager().buildLuminanceSource(data, width, height);
        if (source != null) {
            BinaryBitmap bitmap = new BinaryBitmap(new HybridBinarizer(source));
            try {
                rawResult = multiFormatReader.decodeWithState(bitmap);
            } catch (ReaderException re) {
                // continue
            } finally {
                multiFormatReader.reset();
            }
        }
        Handler handler = activity.getHandler();
        if (rawResult != null) {
            // Don't log the barcode contents for security.
            long end = System.currentTimeMillis();
            Log.d(TAG, "Found barcode in " + (end - start) + " ms");
            if (handler != null) {
                Message message = Message.obtain(handler, R.id.decode_succeeded, rawResult);
                message.sendToTarget();
            }
        } else {
            if (handler != null) {
                Message message = Message.obtain(handler, R.id.decode_failed);
                message.sendToTarget();
            }
        }
    }

```

显然，这个时候处理解码的线程对应的Handler已经拿到了预览帧的byte数组以及预览帧的高和宽。在这个方法中，我们首先根据此时屏幕是横屏还是竖屏对预览帧数组进行了一个预处理。

因为在Android设备上，存在以下[几个概念](https://gold.xitu.io/entry/56aa36fad342d300542e7510)：

>屏幕方向：在Android系统中，屏幕的左上角是坐标系统的原点（0,0）坐标。原点向右延伸是X轴正方向，原点向下延伸是Y轴正方向。

>相机传感器方向：手机相机的图像数据都是来自于摄像头硬件的图像传感器，这个传感器在被固定到手机上后有一个默认的取景方向，坐标原点位于手机横放时的左上角，即与横屏应用的屏幕X方向一致。换句话说，与竖屏应用的屏幕X方向呈90度角。

>相机的预览方向：由于手机屏幕可以360度旋转，为了保证用户无论怎么旋转手机都能看到“正确”的预览画面（这个“正确”是指显示在UI预览界面的画面与人眼看到的眼前的画面是一致的），Android系统底层根据当前手机屏幕的方向对图像传感器采集到的数据进行了旋转处理，然后才送给显示系统，因此可以保证预览画面始终“正确”。在相机API中可以通过setDisplayOrientation()设置相机预览方向。在默认情况下，这个值为0，与图像传感器一致。因此对于横屏应用来说，由于屏幕方向和预览方向一致，预览图像不会颠倒90度。但是对于竖屏应用，屏幕方向和预览方向垂直，所以会出现颠倒90度现象。为了得到正确的预览画面，必须通过API将相机的预览方向旋转90，保持与屏幕方向一致，如图3所示。

也就是说，相机得到的图像数据始终是一个横屏的姿态，当手机处于竖屏时，即使我们通过设置在屏幕上看到的拍摄画面是准确的，没有90度翻转的，我们通过API得到的图像数据仍然是基于横屏的，因此在判断到width < height即屏幕处于竖屏状态时，我们首先对byte数组进行一个手动90度旋转，然后将结果构造为一个PlanarYUVLuminanceSource对象，进行真正的解析处理去了，这里我们就不管了。

然后再看这个预览帧数据是怎么来的。

### 2.PreviewCallback.class中的onPreviewFrame()方法

```java
    @Override
    public void onPreviewFrame(byte[] data, Camera camera) {
        Point cameraResolution = configManager.getCameraResolution();
        Handler thePreviewHandler = previewHandler;
        if (cameraResolution != null && thePreviewHandler != null) {
            Message message;
            Point screenResolution = configManager.getScreenResolution();
            if (screenResolution.x < screenResolution.y){
                // portrait
                message = thePreviewHandler.obtainMessage(previewMessage, cameraResolution.y,
                        cameraResolution.x, data);
            } else {
                // landscape
                message = thePreviewHandler.obtainMessage(previewMessage, cameraResolution.x,
                        cameraResolution.y, data);
            }
            message.sendToTarget();
            previewHandler = null;
        } else {
            Log.d(TAG, "Got preview callback, but no handler or resolution available");
        }
    }
```

这个容易理解，就是系统Camera.PreviewCallback接口，并实现了回调方法，每次获取到预览帧时将图像数据进行回调，同样区分了横竖屏的情况以方便上文decode时的预处理。这里出现了cameraResolution和screenResolution两个对象，我们接下来看看它们分别是什么。

### 3.CameraConfigurationManager.class中的initFromCameraParameters()方法

我们可以看到，上面提到的cameraResolution和screenResolution是在CameraConfigurationManager.class中的initFromCameraParameters()方法中得到的。

```java
    /**
     * Reads, one time, values from the camera that are needed by the app.
     */
    void initFromCameraParameters(OpenCamera camera) {
        Camera.Parameters parameters = camera.getCamera().getParameters();
        WindowManager manager = (WindowManager) context.getSystemService(Context.WINDOW_SERVICE);
        Display display = manager.getDefaultDisplay();
        int displayRotation = display.getRotation();
        int cwRotationFromNaturalToDisplay;
        switch (displayRotation) {
            case Surface.ROTATION_0:
                cwRotationFromNaturalToDisplay = 0;
                break;
            case Surface.ROTATION_90:
                cwRotationFromNaturalToDisplay = 90;
                break;
            case Surface.ROTATION_180:
                cwRotationFromNaturalToDisplay = 180;
                break;
            case Surface.ROTATION_270:
                cwRotationFromNaturalToDisplay = 270;
                break;
            default:
                // Have seen this return incorrect values like -90
                if (displayRotation % 90 == 0) {
                    cwRotationFromNaturalToDisplay = (360 + displayRotation) % 360;
                } else {
                    throw new IllegalArgumentException("Bad rotation: " + displayRotation);
                }
        }
        Log.i(TAG, "Display at: " + cwRotationFromNaturalToDisplay);

        int cwRotationFromNaturalToCamera = camera.getOrientation();
        Log.i(TAG, "Camera at: " + cwRotationFromNaturalToCamera);

        // Still not 100% sure about this. But acts like we need to flip this:
        if (camera.getFacing() == CameraFacing.FRONT) {
            cwRotationFromNaturalToCamera = (360 - cwRotationFromNaturalToCamera) % 360;
            Log.i(TAG, "Front camera overriden to: " + cwRotationFromNaturalToCamera);
        }

        cwRotationFromDisplayToCamera =
                (360 + cwRotationFromNaturalToCamera - cwRotationFromNaturalToDisplay) % 360;
        Log.i(TAG, "Final display orientation: " + cwRotationFromDisplayToCamera);
        int cwNeededRotation;
        if (camera.getFacing() == CameraFacing.FRONT) {
            Log.i(TAG, "Compensating rotation for front camera");
            cwNeededRotation = (360 - cwRotationFromDisplayToCamera) % 360;
        } else {
            cwNeededRotation = cwRotationFromDisplayToCamera;
        }
        Log.i(TAG, "Clockwise rotation from display to camera: " + cwNeededRotation);

        Point theScreenResolution = new Point();
        display.getSize(theScreenResolution);
        screenResolution = theScreenResolution;
        Log.i(TAG, "Screen resolution in current orientation: " + screenResolution);
        cameraResolution = CameraConfigurationUtils.findBestPreviewSizeValue(parameters, screenResolution);
        Log.i(TAG, "Camera resolution: " + cameraResolution);
        bestPreviewSize = CameraConfigurationUtils.findBestPreviewSizeValue(parameters, screenResolution);
        Log.i(TAG, "Best available preview size: " + bestPreviewSize);
    }
```

这个方法中前面的大段通过屏幕方向和摄像头成像方向来计算预览画面的旋转度数，从而保证预览得到的画面随着手机的旋转或者前后置摄像头的更换而保持正确的显示，当然，我们也可以看到screenResolution是通过Display类来获取到的Point对象。它的x，y值就分别代表当前屏幕的横向和纵向的像素值，当然这个是和屏幕方向有关系的。然后可以看到另外两个Point对象cameraResolution以及bestPreviewSize是通过CameraConfigurationUtils.class中的findBestPreviewSizeValue()方法得到的，那我们再来看这个方法。

### 4.CameraConfigurationUtils.class中的findBestPreviewSizeValue()方法

```java
    static Point findBestPreviewSizeValue(Camera.Parameters parameters, Point screenResolution) {

        List<Camera.Size> rawSupportedSizes = parameters.getSupportedPreviewSizes();
        if (rawSupportedSizes == null) {
            Log.w(TAG, "Device returned no supported preview sizes; using default");
            Camera.Size defaultSize = parameters.getPreviewSize();
            if (defaultSize == null) {
                throw new IllegalStateException("Parameters contained no preview size!");
            }
            return new Point(defaultSize.width, defaultSize.height);
        }

        // Sort by size, descending
        List<Camera.Size> supportedPreviewSizes = new ArrayList<>(rawSupportedSizes);
        Collections.sort(supportedPreviewSizes, new Comparator<Camera.Size>() {
            @Override
            public int compare(Camera.Size a, Camera.Size b) {
                int aPixels = a.height * a.width;
                int bPixels = b.height * b.width;
                if (bPixels < aPixels) {
                    return -1;
                }
                if (bPixels > aPixels) {
                    return 1;
                }
                return 0;
            }
        });

        if (Log.isLoggable(TAG, Log.INFO)) {
            StringBuilder previewSizesString = new StringBuilder();
            for (Camera.Size supportedPreviewSize : supportedPreviewSizes) {
                previewSizesString.append(supportedPreviewSize.width).append('x')
                        .append(supportedPreviewSize.height).append(' ');
            }
            Log.i(TAG, "Supported preview sizes: " + previewSizesString);
        }

        double screenAspectRatio = screenResolution.x / (double) screenResolution.y;
        // Remove sizes that are unsuitable
        Iterator<Camera.Size> it = supportedPreviewSizes.iterator();
        while (it.hasNext()) {
            Camera.Size supportedPreviewSize = it.next();
            int realWidth = supportedPreviewSize.width;
            int realHeight = supportedPreviewSize.height;
            if (realWidth * realHeight < MIN_PREVIEW_PIXELS) {
                it.remove();
                continue;
            }
            boolean isScreenPortrait = screenResolution.x < screenResolution.y;
            int maybeFlippedWidth = isScreenPortrait ? realHeight : realWidth;
            int maybeFlippedHeight = isScreenPortrait ? realWidth : realHeight;
            double aspectRatio = (double) maybeFlippedWidth / (double) maybeFlippedHeight;
            double distortion = Math.abs(aspectRatio - screenAspectRatio);
            if (distortion > MAX_ASPECT_DISTORTION) {
                it.remove();
                continue;
            }

            if (maybeFlippedWidth == screenResolution.x && maybeFlippedHeight == screenResolution.y) {
                Point exactPoint = new Point(realWidth, realHeight);
                Log.i(TAG, "Found preview size exactly matching screen size: " + exactPoint);
                return exactPoint;
            }
        }

        // If no exact match, use largest preview size. This was not a great idea on older devices because
        // of the additional computation needed. We're likely to get here on newer Android 4+ devices, where
        // the CPU is much more powerful.
        if (!supportedPreviewSizes.isEmpty()) {
            Camera.Size largestPreview = supportedPreviewSizes.get(0);
            Point largestSize = new Point(largestPreview.width, largestPreview.height);
            Log.i(TAG, "Using largest suitable preview size: " + largestSize);
            return largestSize;
        }

        // If there is nothing at all suitable, return current preview size
        Camera.Size defaultPreview = parameters.getPreviewSize();
        if (defaultPreview == null) {
            throw new IllegalStateException("Parameters contained no preview size!");
        }
        Point defaultSize = new Point(defaultPreview.width, defaultPreview.height);
        Log.i(TAG, "No suitable preview sizes, using default: " + defaultSize);
        return defaultSize;
    }
```
可以看出所谓bestPreviewSize就是将相机支持的预览分辨率都获取到然后找一个和屏幕分辨率最接近的最为最终的结果，当然，同样，要有横竖屏的处理。
那么上面步骤3获取到的这些Point对象等等还有什么用呢，其实它们都将作为相机预览及显示的参数设置给Camera对象，如以下这个方法：

### 5.CameraConfigurationManager.class中的setDesiredCameraParameters()方法

```java
    void setDesiredCameraParameters(OpenCamera camera, boolean safeMode) {
        Camera theCamera = camera.getCamera();
        Camera.Parameters parameters = theCamera.getParameters();
        if (parameters == null) {
            Log.w(TAG, "Device error: no camera parameters are available. Proceeding without configuration.");
            return;
        }
        Log.i(TAG, "Initial camera parameters: " + parameters.flatten());

        if (safeMode) {
            Log.w(TAG, "In camera config safe mode -- most settings will not be honored");
        }
        initializeTorch(parameters, safeMode);
        CameraConfigurationUtils.setFocus(
                parameters,
                true,
                true,
                safeMode);
        if (!safeMode) {
            CameraConfigurationUtils.setBarcodeSceneMode(parameters);
            CameraConfigurationUtils.setVideoStabilization(parameters);
            CameraConfigurationUtils.setFocusArea(parameters);
            CameraConfigurationUtils.setMetering(parameters);
        }
        parameters.setPreviewSize(bestPreviewSize.x, bestPreviewSize.y);
        theCamera.setParameters(parameters);
        theCamera.setDisplayOrientation(cwRotationFromDisplayToCamera);
        Camera.Parameters afterParameters = theCamera.getParameters();
        Camera.Size afterSize = afterParameters.getPreviewSize();
        if (afterSize != null && (bestPreviewSize.x != afterSize.width || bestPreviewSize.y != afterSize.height)) {
            Log.w(TAG, "Camera said it supported preview size " + bestPreviewSize.x + 'x' + bestPreviewSize.y +
                    ", but after setting it, preview size is " + afterSize.width + 'x' + afterSize.height);
            bestPreviewSize.x = afterSize.width;
            bestPreviewSize.y = afterSize.height;
        }
    }
```
其实很明了了，这个方法是将获取到的那些参数整合成Parameters对象set到Camera里。
至此，我们大概说了两个问题，一个是如何获取并给相机设置参数，另一个是如何获取摄像头的预览数据并进行处理，接下来还有一个很重要的点需要说明，那就是我们虽然获取了整个预览帧的数据准备对其解析，但实际上，对于条码扫描来说，真正被处理的其实只是扫描框内的那部分图片或者说数据，所以我们在扫描的时候也必须将条码置于框框内，那么这就涉及到了两个部分，一个是在屏幕上绘制这样一个矩形框，另一个是在预览帧里提取框内的数据。这两点分别由以下方法实现。

### 6.CameraManager.getFramingRect()方法

```java
    /**
     * Calculates the framing rect which the UI should draw to show the user where to place the
     * barcode. This target helps with alignment as well as forces the user to hold the device
     * far enough away to ensure the image will be in focus.
     *
     * @return The rectangle to draw on screen in window coordinates.
     */
    public synchronized Rect getFramingRect() {
        if (framingRect == null) {
            if (camera == null) {
                return null;
            }
            Point screenResolution = configManager.getScreenResolution();
            if (screenResolution == null) {
                // Called early, before init even finished
                return null;
            }

            int width = findDesiredDimensionInRange(screenResolution.x, MIN_FRAME_WIDTH, MAX_FRAME_WIDTH);
            int height = findDesiredDimensionInRange(screenResolution.y, MIN_FRAME_HEIGHT, MAX_FRAME_HEIGHT);

            int leftOffset = (screenResolution.x - width) / 2;
            int topOffset = (screenResolution.y - height) / 2;
            framingRect = new Rect(leftOffset, topOffset, leftOffset + width, topOffset + height);
            Log.d(TAG, "Calculated framing rect: " + framingRect);
        }
        return framingRect;
    }
```

这个是UI中的方框Rect对象的构造，很简单了，就是根据屏幕分辨率然后按照一个固定的比例来设置方框大小。这个方法在方框的自定义View绘制时调用。

### 7.CameraManager.class的getFramingRectInPreview()方法

```java
    /**
     * Like {@link #getFramingRect} but coordinates are in terms of the preview frame,
     * not UI / screen.
     *
     * @return {@link Rect} expressing barcode scan area in terms of the preview size
     */
    public synchronized Rect getFramingRectInPreview() {
        if (framingRectInPreview == null) {
            Rect framingRect = getFramingRect();
            if (framingRect == null) {
                return null;
            }
            Rect rect = new Rect(framingRect);
            Point cameraResolution = configManager.getCameraResolution();
            Point screenResolution = configManager.getScreenResolution();
            if (cameraResolution == null || screenResolution == null) {
                // Called early, before init even finished
                return null;
            }
            if (screenResolution.x < screenResolution.y) {
                // portrait
                rect.left = rect.left * cameraResolution.y / screenResolution.x;
                rect.right = rect.right * cameraResolution.y / screenResolution.x;
                rect.top = rect.top * cameraResolution.x / screenResolution.y;
                rect.bottom = rect.bottom * cameraResolution.x / screenResolution.y;
            } else {
                // landscape
                rect.left = rect.left * cameraResolution.x / screenResolution.x;
                rect.right = rect.right * cameraResolution.x / screenResolution.x;
                rect.top = rect.top * cameraResolution.y / screenResolution.y;
                rect.bottom = rect.bottom * cameraResolution.y / screenResolution.y;
            }
            framingRectInPreview = rect;

        }
        return framingRectInPreview;
    }
```

这个是预览帧方框Rect对象的构造，其实也很简单，就是因为相机卢兰帧分辨率和屏幕显示分辨率可能不一致，因此首先计算这两者的比例，然后再按比例对步骤6中的UI方框进行缩放，同样，计算比例的时候要区分横竖屏。这个方法是在buildLuminanceSource()中调用的，也就是步骤1中的构造PlanarYUVLuminanceSource对象时，其实还传入了这一Rect对象，来代表有效数据。

看完是不是有点点乱，因为本文没有系统的讲解，只是将所涉及内容的一些关键点比如Android Camera的使用，以及相应的横竖屏的区别处理做了介绍，真正核心的条码解码算法并没有深入，“浅尝辄止”了，就酱紫吧，有什么问题欢迎大家讨论。

另，转载请注明出处！文中若有什么错误希望大家探讨指正！

[1]: http://obc3atr48.bkt.clouddn.com/WechatIMG22.jpeg
