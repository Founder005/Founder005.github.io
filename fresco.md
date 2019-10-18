# Fresco
### 特性
- 支持渐进式图片展示
- 支持自定义焦点 foucsCrop
- 支持gif和webP(可以使老Android版本同样获得支持能力)
- 图片不显示时及时释放内存，减少OOM，在低端机上同样表现出色
- 支持设置叠加图overlayImage（比如加个水印图）和按压图
- 更方便的裁剪圆形和圆角
- 支持宽高比显示

### 缺点
- 体积过大，虽然可以通过gradle过滤so库，相比glide依然庞大；
- 获取bitmap不容易
- 布局文件不支持宽高同时设置`wrapcontet`,必须固定尺寸或者使用宽高比

Fresco加载图片不是将图片放进ImageView，虽然继承自ImageView，但是不支持原有的ImageView的`setImageXxx，setScaleType`和类似方法，相当于一个全新的ImageView，使用自定义的属性和方法，xml文件中使用`SimpleDraweeView`代替`ImageView`

### 开始使用
#### 1.build.gradle引入
`implementation 'com.facebook.fresco:fresco:2.0.0'`
###### 支持gif
`implementation 'com.facebook.fresco:animated-gif:2.0.0'`
###### 支持WebP动图
`implementation 'com.facebook.fresco:animated-webp:2.0.0'`
###### 支持WebP静图
`implementation 'com.facebook.fresco:webpsupport:2.0.0'`
#### 2.初始化
自定义的Application中onCreate方法`Fresco.initialize(this);`
#### 3.xml文件中加入命名空间
`xmlns:fresco="http://schemas.android.com/apk/res-auto"`

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

### GenericDraweeHierarchy
#### 代码中使用
一般设置图片使用`mSimpleDraweeView.setImageURI(uri);`配合xml中的各种效果就可以了，推荐使用这一种，容易理解而且一般我们也不需要另外设置某个图片的展示效果，除了最终要显示的目标图片，所有的图片层都可以在 XML 里面设置，他们的值可以是一个 @drawable/ 图片资源 或者 @color 颜色资源。
当然如果必须在代码中改变，fresco也提供的有方法，如下:
代码中设置xml中的效果使用`GenericDraweeHierarchy`
```
GenericDraweeHierarchyBuilder builder =
                new GenericDraweeHierarchyBuilder(getResources());
GenericDraweeHierarchy hierarchy = builder
                //.setPlaceholderImage()
                //...
                .setProgressBarImage(new ProgressBarDrawable())
                .build();
 simpleDraweeView.setHierarchy(hierarchy);
```

比较特殊的：
1.如果选择的缩放类型是focusCrop，需要指定一个中心点：`hierarchy.setActualImageFocusPoint(point);`
2.如果设置的圆形（原来为圆角的不能修改为圆圈，反之亦然），需要先获取圆角的参数
```
RoundingParams roundingParams = hierarchy.getRoundingParams();
roundingParams.setCornersRadius(10);
hierarchy.setRoundingParams(roundingParams);
```
 
> 对于同一个View，不要多次调用setHierarchy，即使这个View是可回收的,可以通过`simpleDraweeView.getHierarchy();`获取到该simpleDraweeView的hierarchy然后去修改效果。创建 DraweeHierarchy 的较为耗时的一个过程，应该多次利用。
> 注意：一个DraweeHierarchy 是不可以被多个 View 共用的！实验证明，共用只会在最后一个view上生效

### DraweeController
#### 设置点击重试
在某些app中，当加载图片失败时，可以点击重新加载，fresco提供了这一功能，而且很容易使用
在xml中设置好retry的图片后，在代码中需要通过DraweeController控制
```
  DraweeController controller = Fresco.newDraweeControllerBuilder()
                .setTapToRetryEnabled(true)
                .setOldController(simpleDraweeView3.getController())
                .setUri("http://hbimg.b00.upaiyun.com/12d9aab22322829a2beb01000a549156fc3e65902f415-Xw2YIl_fw658")
                .build();
```
> 点击重试有四次机会，如果还是加载失败，则显示加载失败提示图片。
> 在指定一个新的controller的时候，使用setOldController，后续需要设置其他的控制时，可以通过`simpleDraweeView.getController()`获取原来的conller,这可节省不必要的内存分配。

#### 监听图片加载的过程
```
   ControllerListener controllerListener = new BaseControllerListener<ImageInfo>() {
            @Override
            public void onSubmit(String id, Object callerContext) {
                super.onSubmit(id, callerContext); //在提交图像请求之前调用。显示图像的第一步
            }

            @Override
            public void onFinalImageSet(String id, ImageInfo imageInfo, Animatable animatable) {
                super.onFinalImageSet(id, imageInfo, animatable);//2显示第二部 设置完最终图像后调用。
                if (imageInfo == null) {
                    return;
                }
                //此处可以获取图片的信息
                QualityInfo qualityInfo = imageInfo.getQualityInfo();
                        imageInfo.getWidth(),
                        imageInfo.getHeight(),
                        qualityInfo.getQuality(),
                        qualityInfo.isOfGoodEnoughQuality(),
                        qualityInfo.isOfFullQuality()));
            }

            @Override
            public void onIntermediateImageSet(String id, ImageInfo imageInfo) {
                super.onIntermediateImageSet(id, imageInfo); //设置任何中间图像后调用。用于加载渐进式图片时回调
            }

            @Override
            public void onIntermediateImageFailed(String id, Throwable throwable) {
                super.onIntermediateImageFailed(id, throwable); //在获取中间映像失败后调用。用于加载渐进式图片时回调
            }

            @Override
            public void onFailure(String id, Throwable throwable) {
                super.onFailure(id, throwable);
            }

            @Override
            public void onRelease(String id) {
                super.onRelease(id);//view不可见时,走这个方法，比如activity消失，挂起，该图片view隐藏
            }
        };
         DraweeController controller = Fresco.newDraweeControllerBuilder()
                .setTapToRetryEnabled(true)
                .setOldController(simpleDraweeView3.getController())
                .setUri("http://hbimg.b0.upaiyun.com/12d9aab22322829a2beb01000a549156fc3e65902f415-Xw2YIl_fw658")
                .setControllerListener(controllerListener)
                .build();
```
### 加载图片的方式

| 类型 | 方式 | 例子 |
| -- | -- | -- |
| 远程图片 |	http://, https:// |	HttpURLConnection 或者参考 使用其他网络加载方案 |
| 本地文件 |	file://	| FileInputStream |
| Content | provider |	content:// |	ContentResolver |
| asset目录下的资源	 | asset:// |	AssetManager |
| res目录下的资源 |	res:// |	Resources.openRawResource  (eg:Uri uri = Uri.parse("res://包名(实际可以是任何字符串甚至留空)/" + R.drawable.ic_launcher);这个需要3个///) |
| Uri中指定图片数据 | data:mime/type;base64, |	数据类型必须符合 rfc2397规定 (仅支持 UTF-8) |

### 各种缩放

|类型|描述|
|--|--|
|center|	居中，无缩放。|
|centerCrop |	保持宽高比缩小或放大，使得两边都大于或等于显示边界，且宽或高契合显示边界。居中显示。|
|focusCrop	同centerCrop, 但居中点不是中点，而是指定的某个点。|
|centerInside|	缩放图片使两边都在显示边界内，居中显示。和 fitCenter 不同，不会对图片进行放大。如果图尺寸大于显示边界，则保持长宽比缩小图片。|
|fitCenter|	保持宽高比，缩小或者放大，使得图片完全显示在显示边界内，且宽或高契合显示边界。居中显示。|
|fitStart|	同上。但不居中，和显示边界左上对齐。|
|fitEnd|	同fitCenter， 但不居中，和显示边界右下对齐。|
|fitXY|	不保存宽高比，填充满显示边界。|
|none|	如要使用tile mode显示, 需要设置为none|

[各种scaleType](https://blog.csdn.net/u012947056/article/details/46816153)

### Fresco加载流程

1. 检查内存缓存，如有，返回
2. 后台线程开始后续工作
3. 检查是否在未解码内存缓存中。如有，解码，变换，返回，然后缓存到内存缓存中
4. 检查是否在磁盘缓存中，如果有，变换，返回。缓存到未解码缓存和内存缓存中
5. 从网络或者本地加载。加载完成后，解码，变换，返回。存到各个缓存中
![加载流程](https://www.fresco-cn.org/static/imagepipeline.png)
