### Glide

Glide 内置了几个常用变化:
- CenterCrop
- FitCenter
- CircleCrop

可以使用`dontTransform`禁用图片变换,使用`.override(Target.SIZE_ORIGINAL,Target.SIZE_ORIGINAL)`可以保留原始图片尺寸

新版的Glide也支持圆角了:`RoundedCorners`

默认使用方法：
`Glide.with(this).load(url).centerCrop().into(imageView);`
或者使用options添加
```
RequestOptions options = new RequestOptions();
options.centerCrop();

Glide.with(fragment)
    .load(url)
    .apply(options)
    .into(imageView);

```
多重变换使用 MultiTransformation 类，或其快捷方法 .transforms() 。
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
注意向 MultiTransformation 的构造器传入变换参数的顺序，决定了这些变换的应用顺序。

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

根据源码可以发现，如果Glide没有手动调用过centerCrop和fitCenter，那么Glide从网络下载的图片格式，由ImageView的scaleType决定，如果scaleType是center_crop，那么Glide以centerCrop下载图片，
如果scaleType是 FIT_CENTER，FIT_START，FIT_END，那么Glide以fitCenter格式下载图片。
测试发现，如果两者设置不同的scaleType，设置到ImageView的时候，还会根据ImageView的scaleType来确立图片位置，稍后我们以代码证明。

在这里也提下郭林大神的那篇文章，文章给的例子是设置的wrapcontent，没有设置Glide的transform的时候，Glide使用的是imageView的fitCenter的缩放，百度的那张图片充满了，文中说由于ImageView默认的scaleType是FIT_CENTER，
因此会自动添加一个FitCenter的图片变换，而在这个图片变换过程中做了某些操作，导致图片充满了全屏。文中说使用`dontTransform`就可以使刚才调用的applyCenterCrop()、applyFitCenter()就统统无效，
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

发现`isScaleOnlyOrNoTransform`是true，根据意思仅缩放不变换，设置`dontTransform`仅仅是禁用了变换。当然，也可能是版本不一致的原因，本文使用的是最新的4.10.0


### 自定义transform
过去使用Glide加载圆形，加边框的圆形，圆角我都是找的网上别人自定义的transform，现在最新的Glide已经支持了圆形和圆角，我们可以试着写其他的变换，
一般我们只需要变换 Bitmap，所以最好是从继承 BitmapTransformation 开始。BitmapTransformation 为我们处理了一些基础的东西，例如我们的变换返回了一个新修改的 Bitmap ，
BitmapTransformation将负责提取和回收原始的 Bitmap，无需像网上有些自定义一样自己再去回收
和RoundedCorners的源码我们可以试着写出自己的transform

1. `equals()`
2. `hashCode()`
3. `updateDiskCacheKey`
**这三个方法官方要求必须实现他们，以使磁盘和内存缓存正常工作**，虽然目前即使没有重写，编译也不会报错，如果你的 Transformation 
需要参数而且它会影响到 Bitmap 被变换的方式，它们也必须被包含到这三个方法中，例如，Glide 的 RoundedCorners 变换接受一个 int，它决定了圆角的弧度。它的equals(), hashCode() 和 updateDiskCacheKey 实现看起来像这样：

```
//重写epquals和hashcode方法，确保对象唯一性，
  @Override
  public boolean equals(Object o) {
    if (o instanceof RoundedCorners) {
      RoundedCorners other = (RoundedCorners) o;
      return roundingRadius == other.roundingRadius;
    }
    return false;
  }

  @Override
  public int hashCode() {
    return Util.hashCode(ID.hashCode(),Util.hashCode(roundingRadius));
  }

  @Override
  public void updateDiskCacheKey(MessageDigest messageDigest) {
    messageDigest.update(ID_BYTES);

    byte[] radiusData = ByteBuffer.allocate(4).putInt(roundingRadius).array();
    messageDigest.update(radiusData);
  }
```
关于updateDiskCacheKey的写法，网上也比较多，推荐一个三方转换库的写法：messageDigest.update((ID + borderSize + borderColor).getBytes(CHARSET));
额外的参数加入到这三个方法主要是为了保证唯一性。

接下来我们试着编写自己的`transform`

首先，我们着重看RoundedCorners的`transform`方法，centerCrop同理也是这个方法
```
@Override
  protected Bitmap transform(
      @NonNull BitmapPool pool, @NonNull Bitmap toTransform, int outWidth, int outHeight) {
    return TransformationUtils.roundedCorners(pool, toTransform, roundingRadius);
  }
```
其中的`TransformationUtils`是Glide提供的一个工具类
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
从上面我们可以发现，变化最终使用的是`BitmapShader`(着色器),使用bitmap来填充给定的图形，它里面只有一个构造方法`BitmapShader(Bitmap bitmap, Shader.TileMode tileX, Shader.TileMode tileY)`
参数：
bitmap：用来做模板的 Bitmap 对象
tileX：横向的 TileMode 视图剩余X轴方向的绘制方式
tileY：纵向的 TileMode 视图剩余Y轴方向的绘制方式

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
4. 在**transform**方法中修改自己想要的变换风格
通过` Bitmap result = pool.get(toTransform.getWidth(), toTransform.getHeight(), Bitmap.Config.ARGB_8888);`获取要进行变换的原位图，
在此基础上进行裁剪，虚化，滤镜等操作，我们在代码中实验
