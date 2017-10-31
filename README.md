# ChoseHead
调用系统相机、相册、剪裁图片并上传（常用于上传头像，兼容Android7.0）

[Hansion的博客](http://blog.csdn.net/hansion3333)


-------------------
由于在Android 7.0 采用了StrictMode API政策禁，其中有一条限制就是对目录访问的限制。

这项变更意味着我们无法通过File API访问手机存储上的数据，也就是说，给其他应用传递 file:// URI 类型的Uri，可能会导致接受者无法访问该路径，并且会会触发 FileUriExposedException异常。

StrictMode API政策禁中的应用间共享文件就是对上述限制的应对方法，它指明了我们在在应用间共享文件可以发送 content:// URI类型的Uri，并授予 URI 临时访问权限，即使用FileProvider


接下来，我们使用FileProvider实现调用系统相机、相册、剪裁图片的功能兼容Android 7.0


## 第一步：FileProvider相关准备工作

 - 在AndroidManifest.xml中增加provider节点，如下：
 
 

```
        <provider
            android:name="android.support.v4.content.FileProvider"
            android:authorities="com.hansion.chosehead"
            android:grantUriPermissions="true"
            android:exported="false">
            <meta-data
                android:name="android.support.FILE_PROVIDER_PATHS"
                android:resource="@xml/filepaths" />
        </provider>
```

> 其中：
>  **android:authorities** 表示授权列表，填写你的应用包名，当有多个授权时，用分号隔开
>  **android:exported** 表示该内容提供器(ContentProvider)是否能被第三方程序组件使用，必须为false，否则会报异常：ava.lang.RuntimeException: Unable to get provider android.support.v4.content.FileProvider: java.lang.SecurityException: Provider must not be exported
>  **android:grantUriPermissions="true"** 表示授予 URI 临时访问权限
>  **android:resource** 属性指向我们自及创建的xml文件的路径，文件名随便起

 - 接下来，我们需要在资源(res)目录下创建一个xml目录，并建立一个以上面名字为文件名的xml文件，内容如下：

```
<?xml version="1.0" encoding="utf-8"?>
<paths>
    <external-path path="" name="image" />
</paths>
```

> 其中：
> **external-path** 代表根目录为: Environment.getExternalStorageDirectory() ，也可以写其他的，如：
> files-path 代表根目录为:Context.getFilesDir() 
>  cache-path 代表根目录为:getCacheDir() 
> 其**path**属性的值代表路径后层级名称，为空则代表就是根目录，假如为“pictures”,就代表对应根目录下的pictures目录



## 第二步：使用FileProvider

 - 在这之前，我们需要在AndroidManifest.xml中增加必要的读写权限：

```
    <uses-permission android:name="android.permission.READ_EXTERNAL_STORAGE" />
    <uses-permission android:name="android.permission.WRITE_EXTERNAL_STORAGE" />
```

### 1. 通过相机获取图片
在通过Intent跳转系统相机前，我们需要对版本进行判断，如果在Android7.0以上,使用FileProvider获取Uri，代码如下：

```
    /**
     * 从相机获取图片
     */
    private void getPicFromCamera() {
        //用于保存调用相机拍照后所生成的文件
        tempFile = new File(Environment.getExternalStorageDirectory().getPath(), System.currentTimeMillis() + ".jpg");
        //跳转到调用系统相机
        Intent intent = new Intent(MediaStore.ACTION_IMAGE_CAPTURE);
        //判断版本
        if (Build.VERSION.SDK_INT >= Build.VERSION_CODES.N) {   //如果在Android7.0以上,使用FileProvider获取Uri
            intent.setFlags(Intent.FLAG_GRANT_WRITE_URI_PERMISSION);
            Uri contentUri = FileProvider.getUriForFile(MainActivity.this, "com.hansion.chosehead", tempFile);
            intent.putExtra(MediaStore.EXTRA_OUTPUT, contentUri);
        } else {    //否则使用Uri.fromFile(file)方法获取Uri
            intent.putExtra(MediaStore.EXTRA_OUTPUT, Uri.fromFile(tempFile));
        }
        startActivityForResult(intent, CAMERA_REQUEST_CODE);
    }
```

如果你好奇通过FileProvider获取的Uri是什么样的，可以打印出来看一看，例如：

```
content://com.hansion.chosehead/image/1509356493642.jpg
```
> 其中：
>      **com.hansion.chosehead** 是我的包名
        **image** 是上文中xml文件中的name属性的值
        **1509356493642.jpg** 是我创建的图片的名字
        也就是说，**content://com.hansion.chosehead/image/ 代表的就是根目录**


### 2.通过相册获取图片

 

```
    /**
     * 从相册获取图片
     */
    private void getPicFromAlbm() {
        Intent photoPickerIntent = new Intent(Intent.ACTION_PICK);
        photoPickerIntent.setType("image/*");
        startActivityForResult(photoPickerIntent, ALBUM_REQUEST_CODE);
    }
```

### 3.剪裁图片

```
    /**
     * 裁剪图片
     */
    private void cropPhoto(Uri uri) {
        Intent intent = new Intent("com.android.camera.action.CROP");
        intent.addFlags(Intent.FLAG_GRANT_READ_URI_PERMISSION);
        intent.addFlags(Intent.FLAG_GRANT_WRITE_URI_PERMISSION);
        intent.setDataAndType(uri, "image/*");
        intent.putExtra("crop", "true");
        intent.putExtra("aspectX", 1);
        intent.putExtra("aspectY", 1);

        intent.putExtra("outputX", 300);
        intent.putExtra("outputY", 300);
        intent.putExtra("return-data", true);

        startActivityForResult(intent, CROP_REQUEST_CODE);
    }
```

> 上文中，你会发现，我们只对Uri进行了特殊处理。没错，这就是核心变化

## 第三步：接收图片信息

- 我们在onActivityResult方法中获得返回的图片信息,在这里我们会先调用剪裁去剪裁图片,然后对剪裁返回的图片进行设置、保存、上传等操作


```
    @Override
    protected void onActivityResult(int requestCode, int resultCode, Intent intent) {
        switch (requestCode) {
            case CAMERA_REQUEST_CODE:   //调用相机后返回
                if (resultCode == RESULT_OK) {
                    //用相机返回的照片去调用剪裁也需要对Uri进行处理
                    if (Build.VERSION.SDK_INT >= Build.VERSION_CODES.N) {
                        Uri contentUri = FileProvider.getUriForFile(MainActivity.this, "com.hansion.chosehead", tempFile);
                        cropPhoto(contentUri);
                    } else {
                        cropPhoto(Uri.fromFile(tempFile));
                    }
                }
                break;
            case ALBUM_REQUEST_CODE:    //调用相册后返回
                if (resultCode == RESULT_OK) {
                    Uri uri = intent.getData();
                    cropPhoto(uri);
                }
                break;
            case CROP_REQUEST_CODE:     //调用剪裁后返回
                Bundle bundle = intent.getExtras();
                if (bundle != null) {
                    //在这里获得了剪裁后的Bitmap对象，可以用于上传
                    Bitmap image = bundle.getParcelable("data");
                    //设置到ImageView上
                    mHeader_iv.setImageBitmap(image);
                    //也可以进行一些保存、压缩等操作后上传
//                    String path = saveImage("crop", image);
                }
                break;
        }
    }
```

- 保存Bitmap到本地的方法：

```
    public String saveImage(String name, Bitmap bmp) {
        File appDir = new File(Environment.getExternalStorageDirectory().getPath());
        if (!appDir.exists()) {
            appDir.mkdir();
        }
        String fileName = name + ".jpg";
        File file = new File(appDir, fileName);
        try {
            FileOutputStream fos = new FileOutputStream(file);
            bmp.compress(Bitmap.CompressFormat.PNG, 100, fos);
            fos.flush();
            fos.close();
            return file.getAbsolutePath();
        } catch (IOException e) {
            e.printStackTrace();
        }
        return null;
    }
```

> 至此，对Android7.0的兼容就结束了
总结一下，在调用相机和剪裁时，传入的Uri需要使用FileProvider来获取

----------
## 完整代码：
- MainActivity.java

```
public class MainActivity extends AppCompatActivity implements View.OnClickListener {

    private ImageView mHeader_iv;

    //相册请求码
    private static final int ALBUM_REQUEST_CODE = 1;
    //相机请求码
    private static final int CAMERA_REQUEST_CODE = 2;
    //剪裁请求码
    private static final int CROP_REQUEST_CODE = 3;

    //调用照相机返回图片文件
    private File tempFile;

    @Override
    protected void onCreate(Bundle savedInstanceState) {
        super.onCreate(savedInstanceState);
        setContentView(R.layout.activity_main);

        initView();
    }

    private void initView() {
        mHeader_iv = (ImageView) findViewById(R.id.mHeader_iv);
        Button mGoCamera_btn = (Button) findViewById(R.id.mGoCamera_btn);
        Button mGoAlbm_btn = (Button) findViewById(R.id.mGoAlbm_btn);
        mGoCamera_btn.setOnClickListener(this);
        mGoAlbm_btn.setOnClickListener(this);
    }

    @Override
    public void onClick(View view) {
        switch (view.getId()) {
            case R.id.mGoCamera_btn:
                getPicFromCamera();
                break;
            case R.id.mGoAlbm_btn:
                getPicFromAlbm();
                break;
            default:
                break;
        }
    }


    /**
     * 从相机获取图片
     */
    private void getPicFromCamera() {
        //用于保存调用相机拍照后所生成的文件
        tempFile = new File(Environment.getExternalStorageDirectory().getPath(), System.currentTimeMillis() + ".jpg");
        //跳转到调用系统相机
        Intent intent = new Intent(MediaStore.ACTION_IMAGE_CAPTURE);
        //判断版本
        if (Build.VERSION.SDK_INT >= Build.VERSION_CODES.N) {   //如果在Android7.0以上,使用FileProvider获取Uri
            intent.setFlags(Intent.FLAG_GRANT_WRITE_URI_PERMISSION);
            Uri contentUri = FileProvider.getUriForFile(MainActivity.this, "com.hansion.chosehead", tempFile);
            intent.putExtra(MediaStore.EXTRA_OUTPUT, contentUri);
            Log.e("dasd", contentUri.toString());
        } else {    //否则使用Uri.fromFile(file)方法获取Uri
            intent.putExtra(MediaStore.EXTRA_OUTPUT, Uri.fromFile(tempFile));
        }
        startActivityForResult(intent, CAMERA_REQUEST_CODE);
    }

    /**
     * 从相册获取图片
     */
    private void getPicFromAlbm() {
        Intent photoPickerIntent = new Intent(Intent.ACTION_PICK);
        photoPickerIntent.setType("image/*");
        startActivityForResult(photoPickerIntent, ALBUM_REQUEST_CODE);
    }


    /**
     * 裁剪图片
     */
    private void cropPhoto(Uri uri) {
        Intent intent = new Intent("com.android.camera.action.CROP");
        intent.addFlags(Intent.FLAG_GRANT_READ_URI_PERMISSION);
        intent.addFlags(Intent.FLAG_GRANT_WRITE_URI_PERMISSION);
        intent.setDataAndType(uri, "image/*");
        intent.putExtra("crop", "true");
        intent.putExtra("aspectX", 1);
        intent.putExtra("aspectY", 1);

        intent.putExtra("outputX", 300);
        intent.putExtra("outputY", 300);
        intent.putExtra("return-data", true);

        startActivityForResult(intent, CROP_REQUEST_CODE);
    }

    @Override
    protected void onActivityResult(int requestCode, int resultCode, Intent intent) {
        switch (requestCode) {
            case CAMERA_REQUEST_CODE:   //调用相机后返回
                if (resultCode == RESULT_OK) {
                    //用相机返回的照片去调用剪裁也需要对Uri进行处理
                    if (Build.VERSION.SDK_INT >= Build.VERSION_CODES.N) {
                        Uri contentUri = FileProvider.getUriForFile(MainActivity.this, "com.hansion.chosehead", tempFile);
                        cropPhoto(contentUri);
                    } else {
                        cropPhoto(Uri.fromFile(tempFile));
                    }
                }
                break;
            case ALBUM_REQUEST_CODE:    //调用相册后返回
                if (resultCode == RESULT_OK) {
                    Uri uri = intent.getData();
                    cropPhoto(uri);
                }
                break;
            case CROP_REQUEST_CODE:     //调用剪裁后返回
                Bundle bundle = intent.getExtras();
                if (bundle != null) {
                    //在这里获得了剪裁后的Bitmap对象，可以用于上传
                    Bitmap image = bundle.getParcelable("data");
                    //设置到ImageView上
                    mHeader_iv.setImageBitmap(image);
                    //也可以进行一些保存、压缩等操作后上传
//                    String path = saveImage("crop", image);
                }
                break;
        }
    }

    public String saveImage(String name, Bitmap bmp) {
        File appDir = new File(Environment.getExternalStorageDirectory().getPath());
        if (!appDir.exists()) {
            appDir.mkdir();
        }
        String fileName = name + ".jpg";
        File file = new File(appDir, fileName);
        try {
            FileOutputStream fos = new FileOutputStream(file);
            bmp.compress(Bitmap.CompressFormat.PNG, 100, fos);
            fos.flush();
            fos.close();
            return file.getAbsolutePath();
        } catch (IOException e) {
            e.printStackTrace();
        }
        return null;
    }
}
```

- activity_main.xml

```
<?xml version="1.0" encoding="utf-8"?>
<LinearLayout xmlns:android="http://schemas.android.com/apk/res/android"
    android:padding="5dp"
    android:layout_width="match_parent"
    android:layout_height="match_parent"
    android:orientation="vertical">

    <TextView
        android:layout_width="wrap_content"
        android:layout_height="wrap_content"
        android:layout_gravity="center_horizontal"
        android:layout_marginTop="30dp"
        android:text="选择头像"
        android:textSize="18sp" />

    <ImageView
        android:id="@+id/mHeader_iv"
        android:layout_width="100dp"
        android:layout_height="100dp"
        android:layout_gravity="center_horizontal"
        android:layout_marginTop="50dp"
        android:src="@mipmap/ic_launcher" />

    <android.support.v4.widget.Space
        android:layout_width="match_parent"
        android:layout_height="0dp"
        android:layout_weight="1"/>

    <Button
        android:id="@+id/mGoCamera_btn"
        android:text="拍照选择"
        android:layout_marginBottom="5dp"
        android:layout_width="match_parent"
        android:layout_height="wrap_content" />
    <Button
        android:id="@+id/mGoAlbm_btn"
        android:text="本地相册选择"
        android:layout_marginBottom="10dp"
        android:layout_width="match_parent"
        android:layout_height="wrap_content" />


</LinearLayout>
```


----------
> 本文还有多处可优化的地方，比如Android 6.0以上动态权限的获取、保存图片前对SD卡是否可用的判断、容错措施等等。这些都是不属于本文的主要内容，本文就不再多说了。
> 其实还有一些偏门的方法，不过本人是不提倡的，比如下文自己处理StrictMode严苛模式、或者将targetSdkVersion改为24以下等方法，这些都是违背开发思想的方法，最好不要使用。


```
        //建议在application 的onCreate()的方法中调用
        if (Build.VERSION.SDK_INT >= Build.VERSION_CODES.N) {
            StrictMode.VmPolicy.Builder builder = new StrictMode.VmPolicy.Builder();
            StrictMode.setVmPolicy(builder.build());
        }
```


