## Android知识点梳理



## Bitmap简介（摘抄于网络）
位图文件（Bitmap），扩展名可以是.bmp或者.dib。位图是Windows标准格式图形文件，它将图像定义为由点（像素）组成，每个点可以由多种色彩表示，包括2、4、8、16、24和32位色彩。

例如，一幅1024×768分辨率的32位真彩图片，其所占存储字节数为：1024×768×32/(8*1024)=3072KB
位图文件图像效果好，但是非压缩格式的，需要占用较大存储空间，不利于在网络上传送。jpg/png格式则恰好弥补了位图文件的缺点。

在Android中计算bitmap的大小：bitmap.getByteCount()（返回byte）

扫盲：1M=1024KB=1024\*1024byte

一般1920X1080尺寸的图片在内存中的大小，1920x1080x32/(8*1024)=8100kb=7.91M

在安卓系统中bitmap图片一般是以ARGB_8888（ARGB分别代表的是透明度,红色,绿色,蓝色,每个值分别用8bit来记录,也就是一个像素会占用4byte,共32bit。）来进行存储的。

#### Android中图片有四种颜色格式

| 颜色格式 | 每个像素占用内存（单位byte） | 每个像素占用内存（单位bit） |
| :--------: | :--------:| :--: |
| ALPHA_8 | 1 | 8 |
| ARGB_8888(默认) | 4 | 32 |
| ARGB_4444 | 2 | 16 |
| RGB_565 | 2 | 16|

ARGB_8888占位算法： 8+8+8+8 =32
 
1 bit   0/1 ；最小单位

1 byte = 8bit      10101010  （0-255）00000000  ---  11111111

说明：

ARGB_8888：ARGB分别代表的是透明度,红色,绿色,蓝色,每个值分别用8bit来记录,也就是一个像素会占用4byte,共32bit.

ARGB_4444：ARGB的是每个值分别用4bit来记录,一个像素会占用2byte,共16bit.

RGB_565：R=5bit,G=6bit,B=5bit，不存在透明度,每个像素会占用2byte,共16bit.

ALPHA_8:该像素只保存透明度,会占用1byte,共8bit.

在实际应用中而言,建议使用ARGB_8888以及RGB_565。
如果你不需要透明度,那么就选择RGB_565,可以减少一半的内存占用.

## Bitmap的回收
在Android3.0以前Bitmap是存放在内存中的，我们需要回收native层和Java层的内存

在Android3.0以后Bitmap是存放在堆中的，我们只要回收堆内存即可

官方建议我们3.0以后使用recycle方法进行回收，该方法可以不主动调用，因为垃圾回收器会自动收集不可用的Bitmap对象进行回收

recycle方法的官方说明：

释放与此位图关联的本地对象，并清除对像素数据的引用。这不会同步释放像素数据;它只是允许它被垃圾收集，如果没有其他的参考。位图被标记为“dead”，意味着如果getPixels（）或setPixels（）被调用，它将抛出异常，并且不会画任何东西。这个操作是不能逆转的，所以只有在你确定没有进一步的位图使用时才能调用它。这是一个高级调用，通常不需要调用，因为当没有更多的引用到这个位图时，正常的GC进程将释放这个内存。

（PS：在此感谢‘凶残的程序员’的指错。）

## LruCache原理
LruCache是个泛型类，内部采用LinkedHashMap来实现缓存机制，它提供get方法和put方法来获取缓存和添加缓存，其最重要的方法trimToSize是用来移除最少使用的缓存和使用最久的缓存，并添加最新的缓存到队列中
## 计算inSampleSize
```
    public static int calculateInSampleSize(BitmapFactory.Options options, int reqWidth, int reqHeight) {
        final int width = options.outWidth;
        final int height = options.outHeight;
        int inSampleSize = 1;
        
        if (width > reqWidth || height > reqHeight) {
            if (width > height) {
                inSampleSize = Math.round((float) height / (float) reqHeight);
            } else {
                inSampleSize = Math.round((float) width / (float) reqWidth);
            }
        }
        return inSampleSize;
    }
```
## 缩略图

```
   public static Bitmap thumbnail(String  path, int maxWidth, int maxHeight,boolean autoRotate) {
        BitmapFactory.Options options = new BitmapFactory.Options();
        options.inJustDecodeBounds = true;
        Bitmap bitmap = BitmapFactory.decodeFile(path);
        options.inJustDecodeBounds = false;
        int sampleSize = calculateInSampleSize(options,maxWidth,maxHeight);
        options.inSampleSize =sampleSize;
        options.inPreferredConfig = Bitmap.Config.RGB_565;
        options.inPurgeable =true;
        options.inInputShareable = true;
        if(bitmap !=null&&!bitmap.isRecycled()){
            bitmap.recycle();
        }
        bitmap = BitmapFactory.decodeFile(path,options);
        return bitmap;
    }
```
## 保存Bitmap
```java
public static String save(Bitmap  bitmap, Bitmap.CompressFormat format, int quality,File desFile) {
        try{
            FileOutputStream out = new FileOutputStream(desFile);
            if(bitmap.compress(format,quality,out)){
                out.flush();
                out.close();
            }
            if(bitmap!=null&&!bitmap.isRecycled()){
                bitmap.recycle();
            }
            return  desFile.getAbsolutePath();
        }catch (FileNotFoundException e){
            e.printStackTrace();
        } catch (IOException e) {
            e.printStackTrace();
        }
        return null;
    }
```
## 保存到SD卡
```
public static String save(Bitmap  bitmap, Bitmap.CompressFormat format, int quality,Context context) {
        if(!Environment.getExternalStorageState()
                .equals(Environment.MEDIA_MOUNTED)){
            return null;
        }
        File dir = new File(Environment.getExternalStorageDirectory()+
        "/"+context.getPackageName());
        if(!dir.exists()){
            dir.mkdir();
        }
        File desFile = new File(dir, UUID.randomUUID().toString());
        return save(bitmap,format,quality,desFile);
    }
```
## 压缩
### 一、质量压缩
质量压缩方法：在保持像素的前提下改变图片的位深及透明度等，来达到压缩图片的目的:

1、bitmap图片的大小不会改变

2、bytes.length是随着quality变小而变小的。

这样适合去传递二进制的图片数据，比如分享图片，要传入二进制数据过去，限制500kb之内。
```
public static Bitmap compressImage(Bitmap image) {
    ByteArrayOutputStream baos = new ByteArrayOutputStream();
    // 把ByteArrayInputStream数据生成图片
    Bitmap bitmap = null;
    image.compress(Bitmap.CompressFormat.JPEG, 5, baos);
    byte[] bytes = baos.toByteArray();
    // 把压缩后的数据baos存放到bytes中
    bitmap = BitmapFactory.decodeByteArray(bytes, 0, bytes.length);
    if (bitmap != null) {
        loga(bitmap, baos.toByteArray());
    }
    return bitmap;
}

// 质量压缩方法，options的值是0-100，这里100表示原来图片的质量，不压缩，把压缩后的数据存放到baos中
image.compress(Bitmap.CompressFormat.JPEG, 100, baos);
int options = 90;
// 循环判断如果压缩后图片是否大于500kb,大于继续压缩
while (baos.toByteArray().length / 1024 > 100) {
    // 重置baos即清空baos
    baos.reset();
    // 这里压缩options%，把压缩后的数据存放到baos中
    image.compress(Bitmap.CompressFormat.JPEG, options, baos);
    // 每次都减少10
    if(options == 1){
        break;
    }else if (options <= 10) {
        options -= 1;
    } else {
        options -= 10;
    }
}
```
### 二、采样率压缩
设置inSampleSize的值(int类型)后，假如设为n，则宽和高都为原来的1/n，宽高都减少，内存降低。上面的代码没用过options.inJustDecodeBounds = true; 因为我是固定来取样的数据，为什么这个压缩方法叫采样率压缩？

是因为配合inJustDecodeBounds，先获取图片的宽、高(这个过程就是取样)。

然后通过获取的宽高，动态的设置inSampleSize的值。 当inJustDecodeBounds设置为true的时候， BitmapFactory通过decodeResource或者decodeFile解码图片时， 将会返回空(null)的Bitmap对象，这样可以避免Bitmap的内存分配， 但是它可以返回Bitmap的宽度、高度以及MimeType。
#### 用法
int inSampleSize = getScaling(bitmap); bitmap = samplingRateCompression(path,inSampleSize);
```
private Bitmap samplingRateCompression(String path, int scaling) {

BitmapFactory.Options options = new BitmapFactory.Options();
options.inSampleSize = scaling;

Bitmap bitmap = BitmapFactory.decodeFile(path, options);

int size = (bitmap.getByteCount() / 1024 / 1024);

Log.i("wechat", "压缩后图片的大小" + (bitmap.getByteCount() / 1024 / 1024)
        + "M宽度为" + bitmap.getWidth() + "高度为" + bitmap.getHeight());
return bitmap;
}

/**
* 获取缩放比例
* @param bitmap
* @return
*/
private int getScaling(Bitmap bitmap) {
    //设置目标尺寸（以像素的宽度为标准）
    int Targetsize = 1500;
    int width = bitmap.getWidth();
    int height = bitmap.getHeight();
    //选择最大值作为比较值（保证图片的压缩大小）
    int handleValue = width > height ? width : height;
    //循环计算压缩比
    int i = 1;
    while (handleValue / i > Targetsize) {
        i++;
    }
}
```
### 三、缩放法压缩, 效果和方法2一样
Android中使用Matrix对图像进行缩放、旋转、平移、斜切等变换的。 Matrix是一个3*3的矩阵，其值对应如下：

|scaleX,	 skewX,	 translateX|

|skewY, 	scaleY,	 translateY|

|0 	，	0 ， 		scale |

Matrix提供了一些方法来控制图片变换：

*	setTranslate(float dx,float dy)：控制Matrix进行位移。
*	setSkew(float kx,float ky)：控制Matrix进行倾斜，kx、ky为X、Y方向上的比例。
*	setSkew(float kx,float ky,float px,float py)：控制Matrix以px、py为轴心进行倾斜，kx、ky为X、Y方向上的倾斜比例。
*	setRotate(float degrees)：控制Matrix进行depress角度的旋转，轴心为（0,0）。
*	setRotate(float degrees,float px,float py)：控制Matrix进行depress角度的旋转，轴心为(px,py)。
*	setScale(float sx,float sy)：设置Matrix进行缩放，sx、sy为X、Y方向上的缩放比例。
*   setScale(float sx,float sy,float px,float py)：设置Matrix以(px,py)为轴心进行缩放，sx、sy为X、Y方向上的缩放比例。

注意：以上的set方法，均有对应的post和pre方法，
Matrix调用一系列set,pre,post方法时,可视为将这些方法插入到一个队列. 当然,按照队列中从头至尾的顺序调用执行. 其中pre表示在队头插入一个方法,post表示在队尾插入一个方法. 而set表示把当前队列清空,并且总是位于队列的最中间位置. 当执行了一次set后: pre方法总是插入到set前部的队列的最前面,post方法总是插入到set后部的队列的最后面

```
private Bitmap ScalingCompression(Bitmap bitmap) {
    Matrix matrix = new Matrix();
    matrix.setScale(0.25f, 0.25f);//缩放效果类似于方法2
    Bitmap bm = Bitmap.createBitmap(bitmap, 0, 0, bitmap.getWidth(),
        bitmap.getHeight(), matrix, true);
    Log.i("wechat", "压缩后图片的大小" + (bm.getByteCount() / 1024 / 1024)
        + "M宽度为" + bm.getWidth() + "高度为" + bm.getHeight());
    return bm;
}
```
### 四、Bitmap.Config

原图尺寸：4M----转化为File---Bitmap大小

*	ALPHA_8----------6.77M -------45M
*	ARGB_4444-------9.37M -------22M
*	ARGB_8888-------6.77M -------45M
*	RGB_565-----------8.13M -------22M

一般情况下默认使用的是ARGB8888，由此可知它是最占内存的，因为一个像素占32位，8位=1字节，所以一个像素占4字节的内存。假设有一张480x800的图片，如果格式为ARGB8888，那么将会占用1500KB的内存。
```
private Bitmap bitmapConfig(String path) {
    BitmapFactory.Options options = new BitmapFactory.Options();
    options.inPreferredConfig = Bitmap.Config.RGB_565;

    Bitmap bm = BitmapFactory.decodeFile(path, options);
    Log.i("wechat", "压缩后图片的大小" + (bm.getByteCount() / 1024f / 1024f)
    + "M宽度为" + bm.getWidth() + "高度为" + bm.getHeight());
    return bm;
} 
```
### 五、Bitmap提供的：createScaledBitmap
#### 方法
Bitmap.createScaledBitmap(src, dstWidth, dstHeight, filter);
#### 参数说明：
* src        用来构建子集的源位图
* dstWidth   新位图期望的宽度
* dstHeight  新位图期望的高度
* filter     为true则选择抗锯齿

#### 补充抗锯齿的知识点
在Android中，目前，我知道有两种出现锯齿的情况。
*	1、当我们用Canvas绘制位图的时候，如果对位图进行了选择，则位图会出现锯齿。
*	2、在用View的RotateAnimation做动画时候， 如果View当中包含有大量的图形，也会出现锯齿。

我们分别以这两种情况加以考虑。 用Canvas绘制位图的的情况。 在用Canvas绘制位图时，一般地，我们使用drawBitmap函数家族， 在这些函数中，都有一个Paint参数， 要做到防止锯齿，我们就要使用到这个参数。如下：

首先在你的构造函数中，需要创建一个Paint。 Paint mPaint = new Paint（）；
然后，您需要设置两个参数:

*	1)mPaint.setAntiAlias(Boolean aa);
*	2)mPaint.setBitmapFilter(true)。

第一个函数是用来防止边缘的锯齿， (true时图像边缘相对清晰一点，锯齿痕迹不那么明显， false时，写上去的字不饱满，不美观，看地不太清楚)。

第二个函数是用来对位图进行滤波处理。

最后，在画图的时候，调用drawBitmap函数，只需要将整个Paint传入即可。

有时候，当你做RotateAnimation时， 你会发现，讨厌的锯齿又出现了。 这个时候，由于你不能控制位图的绘制， 只能用其他方法来实现防止锯齿。 另外，如果你画的位图很多。 不想每个位图的绘制都传入一个Paint。 还有的时候，你不可能控制每个窗口的绘制的时候， 您就需要用下面的方法来处理——对整个Canvas进行处理。

*	1）在您的构造函数中，创建一个Paint滤波器。 PaintFlagsDrawFilter mSetfil = new PaintFlagsDrawFilter(0, Paint.FILTERBITMAPFLAG); 第一个参数是你要清除的标志位， 第二个参数是你要设置的标志位。此处设置为对位图进行滤波。
*	2）当你在画图的时候， 如果是View则在onDraw当中，如果是ViewGroup则在dispatchDraw中调用如下函数。 canvas.setDrawFilter( mSetfil );

另外，在Drawable类及其子类中， 也有函数setFilterBitmap可以用来对Bitmap进行滤波处理， 这样，当你选择Drawable时，会有抗锯齿的效果。
```
private Bitmap createScaledBitmap(Bitmap bitmap) {
    Bitmap bm = Bitmap.createScaledBitmap(bitmap, 200, 200, true);
    Log.i("wechat", "压缩后图片的大小" + (bm.getByteCount() / 1024) + "KB宽度为"
        + bm.getWidth() + "高度为" + bm.getHeight());
    return bm;
}
```
### 六、辅助方法（上述方法的）：
#### 通过路径获取bitmap的方法
*	1、利用BitmapFactory解析文件，转换为Bitmap
bitmap = BitmapFactory.decodeFile(path);
*	2、自己写解码，转换为Bitmap过程， 同样需使用BitmapFactory.decodeByteArray(buf, 0, len)；代码如下：
```
private Bitmap getBitmapByPath(String path) {
if (!new File(path).exists()) {
    System.err.println("getBitmapByPath: 文件不存在");
    return null;
}
byte[] buf = new byte[1024 * 1024];// 1M
Bitmap bitmap = null;
try {
    FileInputStream fis = new FileInputStream(path);
    int len = fis.read(buf, 0, buf.length);
    bitmap = BitmapFactory.decodeByteArray(buf, 0, len);
    //当bitmap为空的时候，说明解析失败
    if (bitmap == null) {
        System.out.println("文件长度:" + len);
        System.err.println("path: " + path + "  无法解析!!!");
    }
} catch (Exception e) {
    e.printStackTrace();
}
return bitmap;
}
```
#### 保存图片
```
private void savaPictrue(Bitmap bitmap) {
    File file = new File("storage/emulated/0/DCIM/Camera/test.jpg");
    FileOutputStream stream = null;
    try {
        stream = new FileOutputStream(file);
        bitmap.compress(Bitmap.CompressFormat.JPEG, 100, stream);
        Log.e("图片大小：", file.length() / 1024 / 1024 + "M");
    } catch (Exception e) {
        e.printStackTrace();
    }
}
```
#### 剪切图片（这里只是裁剪图片，但是对图片的大小并不影响）
```
private void crop(Uri uri) {
    // 裁剪图片意图
    Intent intent = new Intent("com.android.camera.action.CROP");
    intent.setDataAndType(uri, "image/*");
    intent.putExtra("crop", "true");
    // 裁剪框的比例，1：1
    intent.putExtra("aspectX", 1);
    intent.putExtra("aspectY", 1);
    // 裁剪后输出图片的尺寸大小
    intent.putExtra("outputX", 300);
    intent.putExtra("outputY", 300);

    intent.putExtra("outputFormat", "JPEG");// 图片格式
    intent.putExtra("noFaceDetection", true);// 取消人脸识别
    intent.putExtra("return-data", true);
    // 开启一个带有返回值的Activity，请求码为PHOTO_REQUEST_CUT
    startActivityForResult(intent, 200);
}

private void logp(Bitmap bitmap) {
Log.i("wechat", "压缩前图片的大小" + (bitmap.getByteCount() / 1024 / 1024)
        + "M宽度为" + bitmap.getWidth() + "高度为" + bitmap.getHeight());
}

private static void loga(Bitmap bitmap, byte[] bytes) {
    Log.i("wechat", "压缩后图片的大小" + (bitmap.getByteCount() / 1024 / 1024)
        + "M宽度为" + bitmap.getWidth() + "高度为" + bitmap.getHeight()
        + "bytes.length=  " + (bytes.length / 1024) + "KB"
    );
}
```
