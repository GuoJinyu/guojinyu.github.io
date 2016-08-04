title: Android使用Builder模式自定义Dialog
categories: 技术
tags: [Android,设计模式]
---

首先说说为啥要自定义Dialog，在任何软件操作系统中，Dialog即对话框都是一种重要的交互模式与信息载体，而Android系统本身的Dialog拥有固定的样式，并且在5.0后采用Material Design设计风格的Dialog美观大气。但是我们开发人员在实际项目过程中遇到的需求是多种多样的，有时我们要匹配APP自己的设计风格，有时我们会觉得系统的对话框使用起来不够自由，因此自己定义一个适合自己的Dialog是很有必要的。

然后为什么要用Builder模式呢，Builder设计模式是一步步创建一个复杂对象的创建型模式，它允许用户在不知道内部构建细节的情况下，可以更精细地控制对象的构造流程。它的优点就在于将对象的构建和表示分离从而解耦。我们都知道Android系统自身的对话框如AlertDialog就采用了Builder模式，因此可见Builder模式很适合用来构建Dialog对象。

好，废话少说，上代码。

- **BaseDialog.java**

```java
package com.acker.android.dialog;

import android.app.Dialog;
import android.content.Context;
import android.os.Bundle;
import android.text.TextUtils;
import android.view.View;
import android.widget.Button;
import android.widget.FrameLayout;
import android.widget.LinearLayout;
import android.widget.ProgressBar;
import android.widget.TextView;

/**
 * 自定义Dialog基类
 *
 * @author guojinyu
 */
public class BaseDialog extends Dialog {

    private TextView tvTitle;
    private TextView tvMsg;
    private ProgressBar pbLoading;
    private Button btnPositive;
    private Button btnNegative;
    private FrameLayout flCustom;
    private View.OnClickListener onDefaultClickListener = new View.OnClickListener() {

        @Override
        public void onClick(View view) {
            cancel();
        }

    };
    private View.OnClickListener onPositiveListener = onDefaultClickListener;
    private View.OnClickListener onNegativeListener = onDefaultClickListener;
    private String mTitle;
    private String mMessage;
    private String positiveText;
    private String negativeText;
    private boolean isProgressBarShow = false;
    private boolean isNegativeBtnShow = true;
    private View mView;

    private BaseDialog(Context context) {
        super(context, R.style.MyDialog);
    }

    @Override
    public void onCreate(Bundle savedInstanceState) {
        super.onCreate(savedInstanceState);
        setContentView(R.layout.dialog_base);
        flCustom = (FrameLayout) findViewById(R.id.fl_dialog_content);
        tvTitle = (TextView) findViewById(R.id.tv_title);
        pbLoading = (ProgressBar) findViewById(R.id.pb_loading);
        tvMsg = (TextView) findViewById(R.id.tv_msg);
        btnPositive = (Button) findViewById(R.id.btn_positive);
        btnNegative = (Button) findViewById(R.id.btn_negative);
    }

    /**
     * 调用完Builder类的create()方法后显示该对话框的方法
     */
    @Override
    public void show() {
        super.show();
        show(this);
    }

    private void show(BaseDialog mDialog) {
        if (!TextUtils.isEmpty(mDialog.mTitle)) {
            mDialog.tvTitle.setText(mDialog.mTitle);
        }
        if (mDialog.mView != null) {
            mDialog.flCustom.addView(mDialog.mView);
            mDialog.pbLoading.setVisibility(View.GONE);
            mDialog.tvMsg.setVisibility(View.GONE);
        } else {
            if (!TextUtils.isEmpty(mDialog.mMessage)) {
                mDialog.tvMsg.setText(mDialog.mMessage);
                mDialog.tvMsg.setVisibility(View.VISIBLE);
            }
            if (isProgressBarShow) {
                mDialog.pbLoading.setVisibility(View.VISIBLE);
                mDialog.btnPositive.setVisibility(View.GONE);
                mDialog.btnNegative.setVisibility(View.GONE);
            }
        }
        if (!mDialog.isNegativeBtnShow) {
            mDialog.btnNegative.setVisibility(View.GONE);
            LinearLayout.LayoutParams layoutParams = (LinearLayout.LayoutParams) mDialog.btnPositive
                    .getLayoutParams();
            layoutParams.setMargins(150, layoutParams.topMargin, 150, layoutParams.bottomMargin);
            mDialog.btnPositive.setLayoutParams(layoutParams);
        } else {
            mDialog.btnNegative.setOnClickListener(mDialog.onNegativeListener);
            if (!TextUtils.isEmpty(mDialog.negativeText)) {
                mDialog.btnNegative.setText(mDialog.negativeText);
            }
        }
        mDialog.btnPositive.setOnClickListener(mDialog.onPositiveListener);
        if (!TextUtils.isEmpty(mDialog.positiveText)) {
            mDialog.btnPositive.setText(mDialog.positiveText);
        }
    }

    public static class Builder {

        private BaseDialog mDialog;

        public Builder(Context context) {
            mDialog = new BaseDialog(context);
        }

        /**
         * 设置对话框标题
         *
         * @param title
         */
        public Builder setTitle(String title) {
            mDialog.mTitle = title;
            return this;
        }

        /**
         * 设置对话框文本内容,如果调用了setView()方法，该项失效
         *
         * @param msg
         */
        public Builder setMessage(String msg) {
            mDialog.mMessage = msg;
            return this;
        }

        /**
         * 设置确认按钮的回调
         *
         * @param onClickListener
         */
        public Builder setPositiveButton(View.OnClickListener onClickListener) {
            mDialog.onPositiveListener = onClickListener;
            return this;
        }

        /**
         * 设置确认按钮的回调
         *
         * @param btnText,onClickListener
         */
        public Builder setPositiveButton(String btnText, View.OnClickListener onClickListener) {
            mDialog.positiveText = btnText;
            mDialog.onPositiveListener = onClickListener;
            return this;
        }

        /**
         * 设置取消按钮的回掉
         *
         * @param onClickListener
         */
        public Builder setNegativeButton(View.OnClickListener onClickListener) {
            mDialog.onNegativeListener = onClickListener;
            return this;
        }

        /**
         * 设置取消按钮的回调
         *
         * @param btnText,onClickListener
         */
        public Builder setNegativeButton(String btnText, View.OnClickListener onClickListener) {
            mDialog.negativeText = btnText;
            mDialog.onNegativeListener = onClickListener;
            return this;
        }

        /**
         * 设置手否显示ProgressBar，默认不显示
         *
         * @param isProgressBarShow
         */
        public Builder setProgressBarShow(boolean isProgressBarShow) {
            mDialog.isProgressBarShow = isProgressBarShow;
            return this;
        }

        /**
         * 设置是否显示取消按钮，默认显示
         *
         * @param isNegativeBtnShow
         */
        public Builder setNegativeBtnShow(boolean isNegativeBtnShow) {
            mDialog.isNegativeBtnShow = isNegativeBtnShow;
            return this;
        }

        /**
         * 设置自定义内容View
         *
         * @param view
         */
        public Builder setView(View view) {
            mDialog.mView = view;
            return this;
        }

        /**
         * 设置该对话框能否被Cancel掉，默认可以
         *
         * @param cancelable
         */
        public Builder setCancelable(boolean cancelable) {
            mDialog.setCancelable(cancelable);
            return this;
        }

        /**
         * 设置对话框被cancel对应的回调接口，cancel()方法被调用时才会回调该接口
         *
         * @param onCancelListener
         */
        public Builder setOnCancelListener(OnCancelListener onCancelListener) {
            mDialog.setOnCancelListener(onCancelListener);
            return this;
        }

        /**
         * 设置对话框消失对应的回调接口，一切对话框消失都会回调该接口
         *
         * @param onDismissListener
         */
        public Builder setOnDismissListener(OnDismissListener onDismissListener) {
            mDialog.setOnDismissListener(onDismissListener);
            return this;
        }

        /**
         * 通过Builder类设置完属性后构造对话框的方法
         */
        public BaseDialog create() {
            return mDialog;
        }
    }
}
```
代码很简单，BaseDialog类内定义一些对话框要显示的控件和这些控件对应的一些属性，以及最终将所有属性填入到控件的方法show()。Builder类是BaseDialog的一个内部类，其中定义了BaseDialog类的所有属性的set方法以及装配完毕后的create()方法。

- 对应的自定义Dialog的布局文件 **dialog_base.xml**如下：

```xml
<LinearLayout xmlns:android="http://schemas.android.com/apk/res/android"
    android:layout_width="match_parent"
    android:layout_height="wrap_content"
    android:layout_marginLeft="20dp"
    android:layout_marginRight="20dp"
    android:background="@drawable/bg_base_dialog"
    android:orientation="vertical">

    <TextView
        android:id="@+id/tv_title"
        android:layout_width="match_parent"
        android:layout_height="wrap_content"
        android:layout_marginBottom="10dp"
        android:layout_marginTop="10dp"
        android:gravity="center"
        android:textColor="@android:color/black"
        android:textSize="18sp" />

    <View
        android:layout_width="match_parent"
        android:layout_height="1dp"
        android:background="@android:color/black" />

    <LinearLayout
        android:layout_width="match_parent"
        android:layout_height="wrap_content"
        android:layout_marginLeft="20dp"
        android:layout_marginRight="20dp"
        android:orientation="horizontal">

        <ProgressBar
            android:id="@+id/pb_loading"
            android:layout_width="wrap_content"
            android:layout_height="match_parent"
            android:layout_gravity="center"
            android:layout_marginBottom="20dp"
            android:layout_marginRight="20dp"
            android:layout_marginTop="20dp"
            android:visibility="gone" />

        <TextView
            android:id="@+id/tv_msg"
            android:layout_width="match_parent"
            android:layout_height="wrap_content"
            android:layout_gravity="center"
            android:layout_marginBottom="20dp"
            android:layout_marginTop="20dp"
            android:textColor="@android:color/black"
            android:textSize="18sp"
            android:visibility="gone" />
    </LinearLayout>

    <FrameLayout
        android:id="@+id/fl_dialog_content"
        android:layout_width="match_parent"
        android:layout_height="wrap_content"></FrameLayout>

    <LinearLayout
        android:layout_width="match_parent"
        android:layout_height="wrap_content"
        android:orientation="horizontal">

        <Button
            android:id="@+id/btn_negative"
            android:layout_width="0dp"
            android:layout_height="40dp"
            android:layout_marginBottom="20dp"
            android:layout_marginLeft="20dp"
            android:layout_marginRight="20dp"
            android:layout_weight="1"
            android:background="@drawable/bg_dialog_btn_negative"
            android:gravity="center"
            android:text="取消"
            android:textColor="@android:color/white"
            android:textSize="18sp" />

        <Button
            android:id="@+id/btn_positive"
            android:layout_width="0dp"
            android:layout_height="40dp"
            android:layout_marginBottom="20dp"
            android:layout_marginLeft="20dp"
            android:layout_marginRight="20dp"
            android:layout_weight="1"
            android:background="@drawable/bg_dialog_btn_positive"
            android:gravity="center"
            android:text="确定"
            android:textColor="@android:color/white"
            android:textSize="18sp" />
    </LinearLayout>

</LinearLayout>
```
涉及到的其他资源文件如下：

- 对话框样式 **styles.xml**

```xml
<resources>
    <!-- 全局Dialog样式 -->
    <style name="MyDialog" parent="@android:style/Theme.Dialog">
        <item name="android:windowBackground">@android:color/transparent</item>
        <item name="android:windowFrame">@null</item>
        <item name="android:windowNoTitle">true</item>
        <item name="android:windowIsFloating">true</item>
        <item name="android:windowIsTranslucent">true</item>
        <item name="android:background">@android:color/transparent</item>
        <item name="android:backgroundDimEnabled">true</item>
    </style>
</resources>
```
通过设置这些属性可以保证对话框背景透明无黑边。

- 确定取消按钮的颜色值 **colors.xml**

```xml
<?xml version="1.0" encoding="utf-8"?>
<resources>
    <color name="btn_dialog_negative_normal">#ff0000</color>
    <color name="btn_dialog_negative_pressed">#bf0000</color>
    <color name="btn_dialog_positive_normal">#368bff</color>
    <color name="btn_dialog_positive_pressed">#0067f3</color>
</resources>
```
- 对话框背景 **bg_base_dialog.xml**
```xml
<?xml version="1.0" encoding="utf-8"?>
<shape xmlns:android="http://schemas.android.com/apk/res/android"
    android:shape="rectangle">
    <corners android:radius="4dp" />
    <solid android:color="@android:color/white" />
    <stroke
        android:width="1dp"
        android:color="#e5e7ea" />
</shape>
```
包含背景、圆角、阴影效果。

- 确定按钮背景 **bg_dialog_btn_positive.xml**

```xml
<?xml version="1.0" encoding="UTF-8"?>
<selector xmlns:android="http://schemas.android.com/apk/res/android">
    <item android:drawable="@color/btn_dialog_positive_normal" android:state_pressed="false"></item>
    <item android:drawable="@color/btn_dialog_positive_pressed" android:state_pressed="true"></item>
</selector>
```
包含正常和按下的效果。

- 取消按钮背景 **bg_dialog_btn_negative.xml**

```xml
<?xml version="1.0" encoding="UTF-8"?>
<selector xmlns:android="http://schemas.android.com/apk/res/android">
    <item android:drawable="@color/btn_dialog_negative_normal" android:state_pressed="false"></item>
    <item android:drawable="@color/btn_dialog_negative_pressed" android:state_pressed="true"></item>
</selector>
```
包含正常和按下的效果。

以上就是整个自定义Dialog的所有内容，接下来我们通过一个简单的Demo来演示如何使用它。

- **MainActivity.java**


```java
package com.acker.android.dialog;

import android.content.DialogInterface;
import android.os.Bundle;
import android.support.v7.app.AppCompatActivity;
import android.view.View;
import android.widget.Toast;

public class MainActivity extends AppCompatActivity {

    BaseDialog dialog;

    @Override
    protected void onCreate(Bundle savedInstanceState) {
        super.onCreate(savedInstanceState);
        setContentView(R.layout.activity_main);
        dialog = new BaseDialog.Builder(this).setTitle("标题").setMessage("内容")
                .setPositiveButton("哈哈", new View.OnClickListener() {
                    @Override
                    public void onClick(View view) {
                        dialog.dismiss();
                    }
                }).setOnCancelListener(new DialogInterface.OnCancelListener() {
                    @Override
                    public void onCancel(DialogInterface dialogInterface) {
                        Toast.makeText(MainActivity.this, "cancel", Toast.LENGTH_SHORT).show();
                    }
                }).setOnDismissListener(new DialogInterface.OnDismissListener() {
                    @Override
                    public void onDismiss(DialogInterface dialogInterface) {
                        Toast.makeText(MainActivity.this, "dismiss", Toast.LENGTH_SHORT).show();
                    }
                }).create();
        dialog.show();
    }

}

```
来看下效果图：

![自定义对话框效果图1][1]

很丑有木有，不过没关系，这里我们只是展示它的用法，如何把对话框做的好看一点就看各位的发挥了。可以看出自定义的BaseDialog的使用方法与Andorid自身的AlertDialog基本一致，都是通过其Builder类进行对象的构建。

该自定义对话框还支持显示ProgressBar以及自定义内容填充的功能。

- 显示ProgressBar且触摸屏幕不可取消：

```java
.setProgressBarShow(true)
.setCancelable(false)
```
效果图如下：

![自定义对话框效果图2][2]

- 自定义内容区域且不显示取消按钮：

```java
View view = getLayoutInflater().inflate(R.layout.dialog_input_amount, null);
final EditText amountEdit = (EditText) view.findViewById(R.id.dialog_et_amount);
amountEdit.setText("123456789");
```
```java
.setView(view)
.setNegativeBtnShow(false)
```
- 其对应的布局文件为 **dialog_input_amount.xml**：

```xml
<LinearLayout xmlns:android="http://schemas.android.com/apk/res/android"
    android:layout_width="match_parent"
    android:layout_height="wrap_content"
    android:orientation="vertical" >

    <EditText
        android:id="@+id/dialog_et_amount"
        android:layout_width="match_parent"
        android:layout_height="40dp"
        android:layout_marginBottom="20dp"
        android:layout_marginLeft="20dp"
        android:layout_marginRight="20dp"
        android:layout_marginTop="20dp"
        android:gravity="center_vertical"
        android:paddingLeft="10dp"
        android:paddingRight="10dp"
        android:inputType="numberDecimal"
        android:singleLine="true"
        android:textSize="18sp" >
    </EditText>

</LinearLayout>
```
效果图如下：

![自定义对话框效果图3][3]

综上，可以看出通过Builder模式自定义Dialog既可以维持原有Android对话框的使用方法，同时使用方便，自由度更高，大家完全可以按照各自的需求来对代码作出相应的修改。需要说明的是本文并没有严格按照传统的Builder设计模式来实现对话框，而是做了一些简化以更适合于我们的场景。

文中所有代码可以在[个人github主页](https://github.com/GuoJinyu/AndroidUtils/tree/master/dialog)查看和下载。

另，转载请注明出处！文中若有什么错误希望大家探讨指正！

  [1]: http://obc3atr48.bkt.clouddn.com/245941188562351983.png
  [2]: http://obc3atr48.bkt.clouddn.com/360263159663942597.png
  [3]: http://obc3atr48.bkt.clouddn.com/704131486458469105.png