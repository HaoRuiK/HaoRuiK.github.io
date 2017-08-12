---
title: 拿来即用：App签名校验和Dex文件校验
date: 2016-11-06 08:24:37
tags: 应用安全
---





##  应用签名校验

<!-- more -->

``` java

/**
 * Create By HaoRui
 */
public class SignCheck {
    private Context context;
    private String cer = null;
    private String realCer = null;
    private static final String TAG = "SignCheck";

    public SignCheck(Context context) {
        this.context = context;
        this.cer = getCertificateSHA1Fingerprint();
    }

    public SignCheck(Context context, String realCer) {
        this.context = context;
        this.realCer = realCer;
        this.cer = getCertificateSHA1Fingerprint();
    }
<!-- more -->
    public String getRealCer() {
        return realCer;
    }

    /**
     * 设置正确的签名
     *
     * @param realCer
     */
    public void setRealCer(String realCer) {
        this.realCer = realCer;
    }

    /**
     * 获取应用的签名
     *
     * @return
     */
    public String getCertificateSHA1Fingerprint() {
        //获取包管理器
        PackageManager pm = context.getPackageManager();

        //获取当前要获取 SHA1 值的包名，也可以用其他的包名，但需要注意，
        //在用其他包名的前提是，此方法传递的参数 Context 应该是对应包的上下文。
        String packageName = context.getPackageName();

        //返回包括在包中的签名信息
        int flags = PackageManager.GET_SIGNATURES;

        PackageInfo packageInfo = null;

        try {
            //获得包的所有内容信息类
            packageInfo = pm.getPackageInfo(packageName, flags);
        } catch (PackageManager.NameNotFoundException e) {
            e.printStackTrace();
        }

        //签名信息
        Signature[] signatures = packageInfo.signatures;
        byte[] cert = signatures[0].toByteArray();

        //将签名转换为字节数组流
        InputStream input = new ByteArrayInputStream(cert);

        //证书工厂类，这个类实现了出厂合格证算法的功能
        CertificateFactory cf = null;

        try {
            cf = CertificateFactory.getInstance("X509");
        } catch (Exception e) {
            e.printStackTrace();
        }

        //X509 证书，X.509 是一种非常通用的证书格式
        X509Certificate c = null;

        try {
            c = (X509Certificate) cf.generateCertificate(input);
        } catch (Exception e) {
            e.printStackTrace();
        }

        String hexString = null;

        try {
            //加密算法的类，这里的参数可以使 MD4,MD5 等加密算法
            MessageDigest md = MessageDigest.getInstance("SHA1");

            //获得公钥
            byte[] publicKey = md.digest(c.getEncoded());

            //字节到十六进制的格式转换
            hexString = byte2HexFormatted(publicKey);

        } catch (NoSuchAlgorithmException e1) {
            e1.printStackTrace();
        } catch (CertificateEncodingException e) {
            e.printStackTrace();
        }
        return hexString;
    }

    //这里是将获取到得编码进行16 进制转换
    private String byte2HexFormatted(byte[] arr) {

        StringBuilder str = new StringBuilder(arr.length * 2);

        for (int i = 0; i <arr.length; i++) {
            String h = Integer.toHexString(arr[i]);
            int l =h.length();
            if (l == 1)
                h = "0" + h;
            if (l > 2)
                h = h.substring(l - 2, l);
            str.append(h.toUpperCase());
            if (i < (arr.length - 1))
                str.append(':');
        }
        return str.toString();
    }

    /**
     * 检测签名是否正确
     * @return true 签名正常 false 签名不正常
     */
    public boolean check() {

        if (this.realCer != null) {
            cer = cer.trim();
            realCer = realCer.trim();
            if (this.cer.equals(this.realCer)) {
                return true;
            }
        }else {
            Log.e(TAG, "未给定真实的签名 SHA-1 值");
        }
        return false;
    }
}
```


**使用方法：**
    首先使用 keytool 获取签名的 sha-1 值，命令为 keytool -list -v -keystore xxx.jks （将 xx.jks 换成你的应用签名所用文件， 这里会提示输入 keystore 的密码  ps：签名文件在eclipse 是 .keystore 文件， 在 Android Studio中就是 .jks 文件））

然后使用如下代码校验签名：

```java
SignCheck signCheck = new SignCheck(this,"27:19:6E:38:6B:87:5E:76:AD:F7:00:E7:EA:84:E4:C6:EE:E3:3D:FA");
  if(signCheck.check()) {
  //TODO 签名正常
  }else                {
  //TODO 签名不正确
  newAlertDialog.Builder(this).setMessage("请前往官方渠道下载正版 app， http://.....").setPositiveButton("确定",null).show();
  }
```
## Dex文件校验

```java
private boolean checkcrc(){
    boolean checkResult = false;
    long crc = Long.parseLong(getString(R.string.crc));//获取字符资源中预埋的crc值
    ZipFile zf;
    try{
        String path = getApplicationContext().getPackageCodePath();//获取apk安装路径
        zf = new ZipFile(path);//将apk封装成zip对象
        ZipEntry ze = zf.getEntry("classes.dex");//获取apk中的classes.dex
        long CurrentCRC = ze.getCrc();//计算当前应用classes.dex的crc值
        if(CurrentCRC != crc){//crc值对比
            checkResult = true;
        }
    }catch(IOException e){
        handleError();
        checkResult = false;
    }
    return checkResult;
}
```