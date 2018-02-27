### NFC相关

> 一个小项目用到了NFC，当时对着文档整理了一下相关的内容。

​	NFC通讯有三种模式，分别为：一、读/写模式，允许NFC设备读写NFC标签。二、点对点通信模式，进行两个同类NFC设备间的数据交换，该操作模式使用的是**Android Beam**。三、虚拟卡模式，让一个NFC设备以NFC卡的方式工作。
​	NFC的标准数据格式是**NDEF**（对于其他格式的数据操作，可以参见android.nfc.tech类），其在Android中的主要应用模式有：一、从一个NFC标签中读取NDEF数据。该模式下用到了标签分发系统，当android设备发现NFC标签时，通过分析数据的类别能够声明其filter并发送请求是否对这些数据进行操作。二、通过Android Beam同另一个NFC设备进行NDEF数据交互，它的交互过程不需要验证，只要两个设备在一定距离内就能发生。
标签分发系统：当屏幕未锁时，android设备会自动搜索NFC标签，并且选择最适合的应用去处理相应的intent，因为如果有用户自主选择应用的话，可能会在过程中离开有效通信距离导致通信中断，其过程如下：一、解析NFC标签并且获取其标识的MIME类型或者一个url。二、将其MIME类型或者url封装入intent。三、打开响应该intent的应用。
​	NDEF的数据封装在**NdefMessage**中，而一条NdefMessage又有多个**NdefRecord**，每一个Record中的数据都是严格按照一个编码规范排列的。一条规范的NdefMessage起第一个NdefRecord包含如下：3bit的类型格式（MIME or url etc. 系统识别NFC类型的依据） 可变长度的类型信息 可变长度的ID标识（不常用，主要用于标注一个标签） 可变长度的负载信息（信息的主体部分）。
如果分发系统在第一个Record中获取到了需要的类型信息，便会将负载信息放入ACTION_NDEF_DISCOVERED标记的intent中，若无法找到相应的类型信息或该Tag内的信息不是Ndef信息时，则会将负载信息放到ACTION_TECH_DISCOVERED标记的intent中。
当多个应用同时支持NFC响应时，按照优先顺序，依次挑选**ACTION_NDEF_DISCOVERED、ACTION_TECH_DISCOVERED和ACTION_TAG_DISCOVERED**。
判断NFC功能是否存在的方法：getDefaultAdapter（）是否为null。
如果要获取ACTION_NDEF_DISCOVERED标记的intent，AndroidManifest中的写法为：

```xml
<intent-filter>
  <action android:name="android.nfc.action.NDEF_DISCOVERED"/>
  <category android:name="android.intent.category.DEFAULT"/>
  <data android:mimeType="text/plain" /><!-- mimeType为需要的type 该例中type为文本类型-->
  <data android:scheme="http"  
        android:host="developer.android.com
        android:pathPrefix="/index.html" /> <!-- 该种data表达方式表示了一个url-->
</intent-filter>
```
如果要获取ACTION_TECH_DISCOVERED标记的intent 则需要写一个在res/xml文件夹中写一个xml文件来声明过滤的类型，如：
```xml
<resources xmlns:xliff="urn:oasis:names:tc:xliff:document:1.2">
  <tech-list>
      <tech>android.nfc.tech.IsoDep</tech>
      <tech>android.nfc.tech.NfcA</tech>
      <tech>android.nfc.tech.NfcB</tech>
      <tech>android.nfc.tech.NfcF</tech>
      <tech>android.nfc.tech.NfcV</tech>
      <tech>android.nfc.tech.Ndef</tech>
      <tech>android.nfc.tech.NdefFormatable</tech>
      <tech>android.nfc.tech.MifareClassic</tech>
      <tech>android.nfc.tech.MifareUltralight</tech>
  </tech-list>
  <!--也可以选择分为几个tech-list写，这样每个tech-list都是独立的，只要其中一个tech-list是搜索到的Tag的getTechList（）的子集，就可以完成匹配-->
</resources>	
```
该xml文件需要在manifest文件中得到引用 以`<meta-data>`标签的方式写在intent-filter的外面，如：
    <meta-data android:name="android.nfc.action.TECH_DISCOVERED"
    android:resource="@xml/nfc_tech_filter" />
当获取到了Tag的intent后，可以通过**EXTRA_TAG、EXTRA_NDEF_MESSAGES和EXTRA_ID**三种键获得相应的Extra的值，其中第一个必选，后两个根据情况任选，分别表示了Tag对象，Tag中的NDEF信息和Tag的ID。取得时直接用intent.getParcelableExtra（NfcAdapter.EXTRA_TAG）方法取得Tag对象，用intent.getParcelableArrayExtra(NfcAdapter.EXTRA_NDEF_MESSAGE)方法取得NdefMessage对象的数组.
构造NDEF标签的方法：
主要有createUri（）、createExternal（）、createMime（）三类方法（4.1版本后支持）
写法：
Uri类型Record：（不推荐使用absolute类型，而应该用TNF_WELL_KNOW中的RTD_URI）
```java
NdefRecord uriRecord = new NdefRecord(NdefRecord.TNF_ABSOLUTE_URI ,
"http://developer.android.com/index.html".getBytes(Charset.forName("US-ASCII")),
new byte[0], new byte[0]);
//or
NdefRecord rtdUriRecord1 = NdefRecord.createUri("http://example.com");
//or
//此时对方应在intent-filter添加相应的data，写法见上面的url实例
Uri uri = new Uri("http://example.com");
NdefRecord rtdUriRecord2 = NdefRecord.createUri(uri);
```
MIME类型Record：
```java
//第一个参数为MIME类型，第二个参数为数据
NdefRecord mimeRecord = NdefRecord.createMime("application/vnd.com.example.android.beam",
"Beam me up, Android".getBytes(Charset.forName("US-ASCII")));
```
文本类型Record：
```java
//传入参数依次为 文本本体、本地信息（用来获取语言）和是否用utf-8解码
public NdefRecord createTextRecord(String payload, Locale locale, boolean encodeInUtf8) {
  byte[] langBytes = locale.getLanguage().getBytes(Charset.forName("US-ASCII"));
  Charset utfEncoding = encodeInUtf8 ? Charset.forName("UTF-8") : Charset.forName("UTF-16");
  byte[] textBytes = payload.getBytes(utfEncoding);
  int utfBit = encodeInUtf8 ? 0 : (1 << 7);
  char status = (char) (utfBit + langBytes.length);
  byte[] data = new byte[1 + langBytes.length + textBytes.length];
  data[0] = (byte) status;
  System.arraycopy(langBytes, 0, data, 1, langBytes.length);
  System.arraycopy(textBytes, 0, data, 1 + langBytes.length, textBytes.length);
  NdefRecord record = new NdefRecord(NdefRecord.TNF_WELL_KNOWN,
  NdefRecord.RTD_TEXT, new byte[0], data);
  return record;
}
```
基于URN的URI类型Record：

```java
byte[] payload; //assign to your data
String domain = "com.example"; //usually your app's package name
String type = "externalType";
NdefRecord extRecord = NdefRecord.createExternal(domain, type, payload);
```

此时接收方的intent-filter为：

```xml
<intent-filter>
  <action android:name="android.nfc.action.NDEF_DISCOVERED" />
  <category android:name="android.intent.category.DEFAULT" />
  <data android:scheme="vnd.android.nfc"
      android:host="ext"
      android:pathPrefix="/com.example:externalType"/>
</intent-filter>
```
​	Android Application Records 可以放在任意NDEF标签中，其中存放了一个应用的包名，在检测到相应的标签后，系统会检查本地是否存在该应用，若存在则打开该应用，若不存在则启动Google Play下载该应用。它可以有效进行应用标识和隔离。但它是应用层面的标识，如果想要打开某一个activity，仍然需要用intent-filter。创建方式：NdefRecord.createApplicationRecord("com.example.android.beam")；注意不能放在Message的第一个Record。
​	**前台分发系统**：可以为当前应用取得最高的intent获取优先级，根据该系统注册时的intent-filter和action来获得相应的Tag发送的intent。步骤为：
```java
//初始化NFC过滤与响应相关的内容
PendingIntent pendingIntent = PendingIntent.getActivity(this, 0,
new Intent(this, getClass()).addFlags(Intent.FLAG_ACTIVITY_SINGLE_TOP), 0);//创建一个延时Intent
IntentFilter ndef = new IntentFilter(NfcAdapter.ACTION_NDEF_DISCOVERED);
try {
    ndef.addDataType("*/*");    /* Handles all MIME based dispatches.You should specify only the ones that you need. */
}
catch (MalformedMimeTypeException e) {
    throw new RuntimeException("fail", e);
}
intentFiltersArray = new IntentFilter[] {ndef, };//设置一个过滤器
techListsArray = new String { new String[] { NfcF.class.getName() } };//设置tech-list 例中为NfcF
mAdapter.setNdefPushMessageCallback(new NfcAdapter.CreateNdefMessageCallback() {
                @Override
                public NdefMessage createNdefMessage(NfcEvent event) {
                    //... 创建一个NdefRecord对象，并返回，该对象用于AndroidBeam交互时作为发送内容
                    LogUtil.i("已创建Ndef信息");
                    return new NdefMessage(new NdefRecord[]{record,});
                }
            }, this);
            mAdapter.setOnNdefPushCompleteCallback(new NfcAdapter.OnNdefPushCompleteCallback() {
                @Override
                public void onNdefPushComplete(NfcEvent event) {
                  	//AndroidBeam信息发送时的回调
                    LogUtil.i("已发送Ndef信息");
                }
            }, this);
        }
//...
//为活动生命周期的各个事件注册前台分发系统
public void onPause() {
    super.onPause();
    mAdapter.disableForegroundDispatch(this);

}

public void onResume() {
    super.onResume();
    mAdapter.enableForegroundDispatch(this, pendingIntent, intentFiltersArray, techListsArray);
}

public void onNewIntent(Intent intent) {
    Tag tagFromIntent = intent.getParcelableExtra(NfcAdapter.EXTRA_TAG);
    //do something with tagFromIntent
}
```
各种Tech数据类型的卡有不同的读取方式，android提供了一个读取并返回值的函数transceive（byte[]）,其参数为不同类型卡的指令，需要另外查询手册，待有时间再试。
实际尝试时发现，当不在manifest中写相应的data过滤文件时，系统将认为应用不匹配任何格式而没有反应。而且非前台调度的情况下newIntent（）也不会被调用，而是会打开一个全新的activity。

项目相关的一些点：
Arduino端NFC通信写不阻塞，读阻塞。
每次Android端接收到信息都会调用一次newIntent()。
在P2P模式下，NFC模块的等待接收信号会自动调用NfcAdapter的回调函数，构造一个NdefRecord并发送，这里需要人工点击屏幕确认发送。
每次接收和发送信号都会有手机的震动。

未确认：若不在Manifest中配置过滤，而仅仅开启前台分发，是否也能收到Nfc信号。（类似广播注册？）

能否控制Nfc信号接收时震动等行为？