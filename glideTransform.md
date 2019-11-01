### Glide

Glide 内置了几个常用变化:
- CenterCrop
- FitCenter
- CircleCrop

可以使用`dontTransform`禁用图片变换,使用`.override(Target.SIZE_ORIGINAL,Target.SIZE_ORIGINAL)`可以保留原始图片尺寸

这里需要提一下的是，Glide的scaleType和imageView的scaleType互相影响的问题
从`.into`的源码中可以找到一些答案：

```
if (!requestOptions.isTransformationSet()
        && requestOptions.isTransformationAllowed()
        && view.getScaleType() != null) {
      // Clone in this method so that if we use this RequestBuilder to load into a View and then
      // into a different target, we don't retain the transformation applied based on the previous
      // View's scale type.
      switch (view.getScaleType()) {
        case CENTER_CROP:
          requestOptions = requestOptions.clone().optionalCenterCrop();
          break;
        case CENTER_INSIDE:
          requestOptions = requestOptions.clone().optionalCenterInside();
          break;
        case FIT_CENTER:
        case FIT_START:
        case FIT_END:
          requestOptions = requestOptions.clone().optionalFitCenter();
          break;
        case FIT_XY:
          requestOptions = requestOptions.clone().optionalCenterInside();
          break;
        case CENTER:
        case MATRIX:
        default:
          // Do nothing.
      }
```

根据源码可以发现，如果设置了transform
Glide从网络下载到的图片，会按照两种格式来下载，一种是centerCrop，一种是fitCenter，一旦下载下来后，设置到ImageView的时候，还会根据ImageView的scaleType来确立图片位置。 
如果Glide没有手动调用过centerCrop和fitCenter，那么Glide从网络下载的图片格式，由ImageView的scaleType决定，如果scaleType是center_crop，那么Glide以centerCrop下载图片，如果scaleType是 FIT_CENTER，FIT_START，FIT_END，那么Glide以fitCenter格式下载图片

在这里也提下郭林大神的那篇文章，文章给的例子是设置的wrapcontent，没有设置Glide的transform的时候，Glide使用的是imageView的fitCenter的缩放，百度的那张图片充满了，文中说由于ImageView默认的scaleType是FIT_CENTER，因此会自动添加一个FitCenter的图片变换，而在这个图片变换过程中做了某些操作，导致图片充满了全屏。文中说使用`dontTransform`就可以使刚才调用的applyCenterCrop()、applyFitCenter()就统统无效，
但是我测试的是没有生效，查看`dontTransform`的源码

```
 public T dontTransform() {
    if (isAutoCloneEnabled) {
      return clone().dontTransform();
    }

    transformations.clear();
    fields &= ~TRANSFORMATION;
    isTransformationRequired = false;
    fields &= ~TRANSFORMATION_REQUIRED;
    isTransformationAllowed = false;
    fields |= TRANSFORMATION_ALLOWED;
    isScaleOnlyOrNoTransform = true;
    return selfOrThrowIfLocked();
  }
```

发现`isScaleOnlyOrNoTransform`是true，根据意思仅缩放不变换，设置`dontTransform`仅仅是禁用了变换。

新版的Glide也支持圆角了，`RoundedCorners`
使用方法：
`Glide.with(this).load(url).centerCrop().into(imageView);`
或
```
RequestOptions options = new RequestOptions();
options.centerCrop();

Glide.with(fragment)
    .load(url)
    .apply(options)
    .into(imageView);

```
或
```
Glide.with(activity)
	.load(url)
	.transform(transform1,transform2,...)
	.into(imageView)
```
或
```
Glide.with(activity)
	.load(url)
	.transform(new MultiTransformation<>(transform1,transform2,...))
	.into(imageView)
```
### 自定义transform
过去使用Glide加载圆形，加边框的圆形，圆角我都是找的网上别人自定义的transform，现在最新的Glide已经支持了圆形和圆角，根据官方文档
和RoundedCorners的源码我们可以试着写出自己的transform

1. `equals()`
2. `hashCode()`
3. `updateDiskCacheKey`
**这三个方法官方要求必须实现他们，以使磁盘和内存缓存正常工作**，虽然目前即使没有重写，编译也不会报错，如果你的 Transformation 
需要参数而且它会影响到 Bitmap 被变换的方式，它们也必须被包含到这三个方法中，比如`RoundedCorners`的圆角角度

自定义转换之前，我们可以参考官方圆形裁剪和圆角的方法，点开RoundedCorners的源码
```
@Override
  protected Bitmap transform(
      @NonNull BitmapPool pool, @NonNull Bitmap toTransform, int outWidth, int outHeight) {
    return TransformationUtils.roundedCorners(pool, toTransform, roundingRadius);
  }
```
其中的`TransformationUtils`是Glide提供的一个工具类，里面包含了圆形和圆角的方法
```
public static Bitmap roundedCorners(
      @NonNull BitmapPool pool, @NonNull Bitmap inBitmap, final int roundingRadius) {
    Preconditions.checkArgument(roundingRadius > 0, "roundingRadius must be greater than 0.");

    return roundedCorners(
        pool,
        inBitmap,
        new DrawRoundedCornerFn() {
          @Override
          public void drawRoundedCorners(Canvas canvas, Paint paint, RectF rect) {
            canvas.drawRoundRect(rect, roundingRadius, roundingRadius, paint);
          }
        });
  }
```
继续点击`roundCorners`方法
```
  private static Bitmap roundedCorners(
      @NonNull BitmapPool pool, @NonNull Bitmap inBitmap, DrawRoundedCornerFn drawRoundedCornerFn) {

    // Alpha is required for this transformation.
    Bitmap.Config safeConfig = getAlphaSafeConfig(inBitmap);
    Bitmap toTransform = getAlphaSafeBitmap(pool, inBitmap);
    Bitmap result = pool.get(toTransform.getWidth(), toTransform.getHeight(), safeConfig);

    result.setHasAlpha(true);

    BitmapShader shader =
        new BitmapShader(toTransform, Shader.TileMode.CLAMP, Shader.TileMode.CLAMP);
    Paint paint = new Paint();
    paint.setAntiAlias(true);
    paint.setShader(shader);
    RectF rect = new RectF(0, 0, result.getWidth(), result.getHeight());
    BITMAP_DRAWABLE_LOCK.lock();
    try {
      Canvas canvas = new Canvas(result);
      canvas.drawColor(Color.TRANSPARENT, PorterDuff.Mode.CLEAR);
      drawRoundedCornerFn.drawRoundedCorners(canvas, paint, rect);
      clear(canvas);
    } finally {
      BITMAP_DRAWABLE_LOCK.unlock();
    }

    if (!toTransform.equals(inBitmap)) {
      pool.put(toTransform);
    }

    return result;
  }
```
第一个参数pool，这个是Glide中的一个Bitmap缓存池，用于对Bitmap对象进行重用，否则每次图片变换都重新创建Bitmap对象将会非常消耗内存。
第二个参数toTransform，这个是原始图片的Bitmap对象，我们就是要对它来进行图片变换。第三和第四个参数比较简单，分别代表图片变换后的宽度和高度，其实也就是override()方法中传入的宽和高的值了
从上面我们可以发现，变化最终使用的是`BitmapShader`(bitmap着色器),是shader的子类，我们可以理解为使用bitmap来填充指定的imageView，它里面只有一个构造方法`BitmapShader(Bitmap bitmap, Shader.TileMode tileX, Shader.TileMode tileY)`
参数：
bitmap：要使用的 Bitmap 对象
tileX：横向的 TileMode ，视图X轴方向的绘制方式
tileY：纵向的 TileMode ，视图Y轴方向的绘制方式

TileMode 有三种取值
- TileMode.CLAMP:用边缘色彩填充多余空间
- TileMode.REPEAT:重复原图像来填充多余空间
- TileMode.MIRROR:重复使用镜像模式的图像来填充多余空间

开始试编写
1. 首先继承`BitmapTransformation`
2. 按照官方的写法定义三个变量
```
	private static final int VERSION = 1;
    private static final String ID = "com.gd.terminalmanager.glidetransformationdemo.transform.MyTransform." + VERSION;
    private static final byte[] ID_BYTES = ID.getBytes(CHARSET);
```
3. 重写三个方法
```
 @Override
    public void updateDiskCacheKey(@NonNull MessageDigest messageDigest) {
        messageDigest.update(ID_BYTES);
    }

    @Override
    public int hashCode() {
        return ID.hashCode();
    }

    @Override
    public boolean equals(@Nullable Object obj) {
        return obj instanceof MyTransform;
    }
```
4. 重点**transform**方法修改自己想要的变换风格
通过` Bitmap result = pool.get(toTransform.getWidth(), toTransform.getHeight(), Bitmap.Config.ARGB_8888);`获取要进行变换的原位图，
在此基础上进行裁剪，虚化，滤镜等操作
比如圆形带边框
```
 /*圆形加边框*/
  	BitmapShader shader = new BitmapShader(toTransform, Shader.TileMode.CLAMP, Shader.TileMode.CLAMP);
        Bitmap result = pool.get(toTransform.getWidth(), toTransform.getHeight(), Bitmap.Config.ARGB_8888);// 获取可复用的bitmap对象
        if (result == null) {
            result = Bitmap.createBitmap(toTransform.getWidth(), toTransform.getHeight(), Bitmap.Config.ARGB_8888);
        }
        int width = toTransform.getWidth();
        int height = toTransform.getHeight();

        Paint paint = new Paint(Paint.ANTI_ALIAS_FLAG);
        paint.setStyle(Paint.Style.FILL);
        paint.setShader(shader);
   	int width = toTransform.getWidth();
        int height = toTransform.getHeight();
        Canvas canvas = new Canvas(result);
        Paint strokePaint  =new Paint(Paint.ANTI_ALIAS_FLAG);
        strokePaint.setColor(Color.BLUE);
        canvas.drawCircle(width/2,height/2,width/2,strokePaint);
        canvas.drawCircle(width/2,height/2,width/2-10,paint);
	return result;
```
