## APK中包含的内容

1.resources.arsc：包含了所有资源文件的映射，可以理解为索引，通过该文件能找到对应的资源文件信息

2.AndroidManifest.xml：Project中AndroidManifest.xml编译后得到的二进制xml文件

3.META-INF：主要保存各个资源文件的SHA1 hash值，用于校验资源文件是否被篡改，防止二次打包时资源文件被替换，该目录下主要包括下面三个文件：

  - CERT.RSA：保存签名和公钥证书
  - MANIFEST.MF：保存版本号以及对每个文件（包括资源文件）整体的SHA1 hash
  - CERT.SF：保存对每个文件头3行的SHA1 hash
  - res：Project中res目录下资源文件编译后得到的二进制xml文件

3.classes.dex：Dex是DalvikVM executes的缩写，即Android Dalvik执行程序

4.lib：对应Project中的libs目录，包含.so文件。

## 打包APK用到的工具

在APK编译打包过程中，用到了以下工具，这些工具大部分位于Android SDK的build-tools目录下：

1.aapt：全称Android Asset Packaging Tool，即Android资源打包工具

2.aidl：将.aidl文件转换为.java文件的工具

3.Java Compiler：java编译器，将.java文件转换为.class文件的工具，运行命令javac

4.dex：将.class文件转换为Davik VM能识别的.dex文件的工具，运行命令dx

5.apkbuilder：生成APK的工具

6.Jarsigner：.jar文件的签名工具

7.zipalign：字节码对齐工具


## 打包流程

![](https://p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/ae7b715dd08f40078903f56eeabd5a54~tplv-k3u1fbpfcp-watermark.webp)

                         （方形:表示文件，椭圆:表示工具及操作）

上面这张图，显示了更为详细的构建流程。以虚线为界，前半部分描述了 编译流程 ，后半部分则描述了 打包流程。

下面具体分析构建流，分为七步（其中编译1-4、打包5-7）：

1.aapt过程：使用aapt/aapt2打包res目录资源文件，生成R.java、resources.arsc和res目录。

R.java保存了res目录下所有资源的id，数据类型都是整型，我们在程序中都是通过使用Android API依据R文件中的资源id来获取对应资源

2.aidl生成Java文件：AIDL是Android Interface Definition Language的缩写，是Android跨进程通讯的一种方式，该阶段会检索Project中所有的aidl文件，并转换为对应的Java文件。

3.javac编译：使用JDK里的javac编译Project src目录下的Java源文件、R.java以及aidl生成的Java文件，并生成.class文件。

4.生成DEX文件：通过dx工具将.class文件转换为classes.dex，目前的gradle multi-dex编译方式会生成classes2.dex ... classesN.dex。

5.打包生成APK：使用apkBuilder将resources.arsc、res目录、AndroidManifest.xml、assets目录、dex文件打包成初始APK，具体逻辑是在com.android.sdklib.build.ApkBuilder中实现的。

6.签名apk文件：使用apksigner为APK添加签名信息

7.zipalign优化签名包：使用zipalign工具对签名包进行内存对齐操作，即优化安装包的结构。


[Android APK编译打包过程](https://www.jianshu.com/p/a71bd35d6dd9)





