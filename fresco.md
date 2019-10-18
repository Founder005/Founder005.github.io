# Fresco
#### 特性
- 支持渐进式图片展示
- 支持自定义焦点 foucsCrop
- 支持gif和webP
- 图片不显示时及时释放内存，减少OOM，在低端机上同样表现出色
- 支持设置叠加图overlayImage（比如加个水印图）和按压图
- 更方便的裁剪圆形和圆角
- 支持宽高比显示

#### 缺点
- 体积过大，虽然可以通过gradle过滤so库，相比glide依然庞大；
- 获取bitmap不容易

Fresco加载图片不是将图片放进ImageView，虽然继承自ImageView，但是不支持原有的ImageView的`setImageXxx，setScaleType`和类似方法，相当于一个全新的ImageView，使用自定义的属性和方法，xml文件中使用`SimpleDraweeView`代替`ImageView`

```
<com.facebook.drawee.view.SimpleDraweeView
  android:id="@+id/my_image_view"
  android:layout_width="20dp"
  android:layout_height="20dp"
  fresco:fadeDuration="300"
  fresco:actualImageScaleType="focusCrop" 
  fresco:placeholderImage="@color/wait_color"
  fresco:placeholderImageScaleType="fitCenter"
  fresco:failureImage="@drawable/error"
  fresco:failureImageScaleType="centerInside"
  fresco:retryImage="@drawable/retrying"
  fresco:retryImageScaleType="centerCrop"
  fresco:progressBarImage="@drawable/progress_bar"
  fresco:progressBarImageScaleType="centerInside"
  fresco:progressBarAutoRotateInterval="1000"
  fresco:backgroundImage="@color/blue"
  fresco:overlayImage="@drawable/watermark"
  fresco:pressedStateOverlayImage="@color/red"
  fresco:roundAsCircle="false"
  fresco:roundedCornerRadius="1dp"
  fresco:roundTopLeft="true"
  fresco:roundTopRight="false"
  fresco:roundBottomLeft="false"
  fresco:roundBottomRight="true"
  fresco:roundWithOverlayColor="@color/corner_color"
  fresco:roundingBorderWidth="2dp"
  fresco:roundingBorderColor="@color/border_color"
/>
```

|属性|含义|
| ------------ | ------------ |
| fadeDuration  | 淡入淡出动画持续时间(单位：毫秒ms)  |
| actualImageScaleType  | 实际图像的缩放类型  |
| placeholderImage  | 占位图  |
| placeholderImageScaleType  | 占位图缩放类型  |
| failureImage | 失败图 |
| failureImageScaleType | 失败图的缩放类型 |
| retryImage | 重试图 |
| retryImageScaleType | 重试图的缩放类型 |
| progressBarImage | 进度图 |
| progressBarImageScaleType | 进度图的缩放类型 |
| progressBarAutoRotateInterval  | 进度图自动旋转间隔时间(单位：毫秒ms)  |
| backgroundImage  | 背景图或颜色  |
| overlayImage  | 叠加图  |
| pressedStateOverlayImage  | 按压时显示图或者颜色  |
| roundAsCircle  | 是否设置圆形（boolean）  |
| roundedCornerRadius  | 圆角的角度（当我们同时设置图像显示为圆形图像和圆角图像时，只会显示为圆形图像）  |
| roundTopLeft  | 左上圆角（boolean）  |
| roundTopRight  | 右上圆角（boolean）  |
| roundBottomLeft  | 左下角是否为圆角 (boolean） |
| roundBottomRight |  右下角是否为圆角（boolean） |
| roundingBorderWidth  | 圆形或者圆角图边框的宽度  |
| roundingBorderColor  | 圆形或者圆角图边框的颜色  |
| roundWithOverlayColor  | 圆形或者圆角图底下的叠加颜色(只能设置颜色)  |
| viewAspectRatio  | 控件纵横比  |





