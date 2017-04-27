## gradle 学习笔记

### 参考文献

1. [一个完整的 Android 项目所用到的gradle配置技巧](http://blog.csdn.net/lj402159806/article/details/54958056)

### 1. project 中的 build.gradle

~~~groovy
buildscript{  //配置用于驱动构建的代码
  repositories{
    jcenter() //声明使用 jCenter 仓库，并且生命了一个 jCenter 文件的 classpath
  }
  dependencies{
    classpath 'com.android.tools.build:gradle:1.3.1'
  }
}

allprojects{
  repositories{
    jcenter()
    maven{
      url="http://maven.mbd.qiyi.domain/nexus/content/repositories/mbd-vertical/"
    }
  }
}
~~~

> 这里的配置只会影响构建过程所需要的类库，而不会影响项目的源代码（只是用于构建）

- local.properties

  在此文件中使用 sdk.dir 属性来设置 SDK 路径。 比如：

  ~~~groovy
  ndk.dir=/Users/zhenzhen/Library/Android/sdk/ndk-bundle
  sdk.dir=/Users/zhenzhen/Library/Android/sdk
  或者
  ndk.dir=D\:\\AndroidSdk\\ndk-bundle
  sdk.dir=D\:\\AndroidSdk
  ~~~



### 2. 项目结构

Gradle 遵循约定优先于配置的概念，尽可能提供合理的默认的配置参数。最基本的项目有两个 “sources sets” 组件，分别存放应用代码和测试代码。

~~~
src/main/
src/androidTest/
~~~

对于 Android plugin 来说，它还拥有以下特有的文件和文件夹结构：

~~~
AndroidManifest.xml
res/
assets/
aidl/
rs/
jni/
jniLibs/
~~~

配置和对照如下：

![](https://ww2.sinaimg.cn/large/006tKfTcgy1fect5r5vp0j30xe0h4wj9.jpg)

![](https://ww1.sinaimg.cn/large/006tKfTcgy1fectcvmpeaj30hx05udft.jpg)

另外一种对照如下：

![](https://ww3.sinaimg.cn/large/006tKfTcgy1fectdn26eaj30zy07imxh.jpg)

![](https://ww4.sinaimg.cn/large/006tKfTcgy1fectdtngqjj30iu05rmx6.jpg)

> 文件的目录结构发生了改变。
>
> setRoot 是指将所有 sourceSets (包含它的子目录) 到新的目录



### 2. application 和 libarary 设置

#### 2.1 application

~~~groovy
com.android.application
~~~

#### 2.2 library

~~~groovy
com.android.library
~~~

library 不能设置 applicationId (因为不是一个 application )