---
share: true
---

> https://juejin.cn/column/7123935861976072199


### gradle 配置

Gradle 是一个通用的自动化构建工具，`通用` 也就是说，不仅可以构建 Android 项目，还可以构建 java、kotlin、swift 等项目。再结合 `android{}` 里面的配置属性，我们可以确定，那这就是 Android 项目专属的配置 `DSL` 了。
```
plugins {
    id 'com.android.application'
    id 'org.jetbrains.kotlin.android'
}

```


项目结构
```
.

├── app
│   ├── build.gradle
│   ├── ...
├── build.gradle
├── gradle
│   └── wrapper
│       ├── gradle-wrapper.jar
│       └── gradle-wrapper.properties
├── gradle.properties
├── local.properties
└── settings.gradle

```





### gradle-wrapper

```
.
├── gradle
│   └── wrapper
│       ├── gradle-wrapper.jar
│       └── gradle-wrapper.properties
├── gradlew
└── gradlew.bat

```


- `gradle-wrapper.jar`：主要是 Gradle 的运行逻辑，包含下载 Gradle；
- `gradle-wrapper.properties`：gradle-wrapper的配置文件，核心是定义了Gradle版本；
- `gradlew`：gradle wrapper的简称，linux下的执行脚本
- `gradlew.bat`：windows下的执行脚本

  

#### gradle-wrapper.properties

```
distributionBase=GRADLE_USER_HOME
distributionUrl= https://services.gradle.org/distributions/gradle-7.4-bin.zip
distributionPath=wrapper/dists
zipStorePath=wrapper/dists
zipStoreBase=GRADLE_USER_HOME
```


- distributionBase：下载的 Gradle 的压缩包解压后的主目录；
- zipStoreBase：同distributionBase，不过是存放zip压缩包的主目录；
- distributionPath：相对于distributionBase的解压后的Gradle的路径，为wrapper/dists；
- zipStorePath：同distributionPath，不过是存放zip压缩包的；
- distributionUrl：Gradle版本的下载地址；

  

### build.gradle（Project）

位于项目的根目录下，用于定义适用于项目中所有模块的依赖项。

Gradle7.0之后，project下的`build.gradle`文件变动很大，默认只有plugin的引用了，其他原有的配置挪到`settings.gradle`文件中了


#### 引用 Plugin
`apply false`表示不将该plugin应用于当前项目，比如在多项目构建中，我只想在某个子项目依赖该plugin就好了，那可以这么写：

``` bash
plugins {
  id "yechaoa" version "1.0.0" apply false
}

subprojects { subproject ->
    if (subproject.name == "subProject") {
        apply plugin: 'yechaoa'
    }
}

```

引用自定义脚本，比如自定义了一个 common. gradle
``` bash
apply from: 'common.gradle'
```


这两种方式的区别：

- apply plugin：'yechaoa'：叫做二进制插件，二进制插件一般都是被打包在一个jar里独立发布的，比如我们自定义的插件，再发布的时候我们也可以为其指定plugin id，这个plugin id最好是一个全限定名称，就像你的包名一样；
- apply from：'yechaoa.gradle'：叫做应用脚本插件，应用脚本插件，其实就是把这个脚本加载进来，和二进制插件不同的是它使用的是`from`关键字，后面紧跟一个脚本文件，可以是本地的，也可以是网络存在的，如果是网络上的话要使用`HTTP URL`。

- - 虽然它不是一个真正的插件，但是不能忽视它的作用，它是脚本文件模块化的基础，我们可以把庞大的脚本文件进行分块、分段整理拆分成一个个共用、职责分明的文件，然后使用apply from来引用它们，比如我们可以把常用的函数放在一个utils.gradle脚本里，供其他脚本文件引用。

  
#### buildscript

buildscript中的声明是gradle脚本自身需要使用的资源。可以声明的资源包括依赖项、第三方插件、maven仓库地址等。7.0之后改在settings.gradle中配置了（pluginManagement）。

``` bash
// Top-level build file where you can add configuration options common to all sub-projects/modules.
buildscript {
    ext.kotlin_version = "1.5.0"
    repositories {
        google()
        mavenCentral()
    }
    dependencies {
        classpath "com.android.tools.build:gradle:4.2.1"
        classpath "org.jetbrains.kotlin:kotlin-gradle-plugin:$kotlin_version"

        // NOTE: Do not place your application dependencies here; they belong
        // in the individual module build.gradle files
    }
}

allprojects {
    repositories {
        google()
        mavenCentral()
        jcenter() // Warning: this repository is going to shut down soon
    }
}

task clean(type: Delete) {
    delete rootProject.buildDir
}

```

##### ext

项目全局属性，多用于自定义，比如把关于版本的信息都利用ext放在另一个新建的gradle文件中集中管理，比如version.gradle，然后apply引用即可，这是较为早期的版本管理方式。

##### repositories

仓库，比如google()、maven()、jcenter()、jitpack等三方托管平台。

##### dependencies

当然配置了仓库还不够，我们还需要在dependencies{}里面的配置里，把需要配置的依赖用classpath配置上，因为这个dependencies在buildscript{}里面，所以代表的是Gradle需要的插件。

7.0之后把plugin的配置简化了，由apply+classpath简化为只要apply就可以了。


#### allprojects

allprojects块的repositories用于多项目构建，为所有项目提供共同所需依赖包。而子项目可以配置自己的repositories以获取自己独需的依赖包。

#### task clean(type: Delete)

运行gradle clean时，执行此处定义的task。

该任务继承自Delete，删除根目录中的build目录。相当于执行Delete.delete(rootProject.buildDir)。其实这个任务的执行就是可以删除生成的Build文件的，跟Android Studio的clean是一个道理。


### build.gradle（Module）

位于每个 **project**/**module**/ 目录下，用于为其所在的特定模块配置 build 设置。

7.0 之后
``` bash
plugins {
    id 'com.android.application'
    id 'org.jetbrains.kotlin.android'
}

android {
    namespace 'com.yechaoa.gradlex'
    compileSdk 32

    defaultConfig {
        applicationId "com.yechaoa.gradlex"
        minSdk 23
        targetSdk 32
        versionCode 1
        versionName "1.0"

        testInstrumentationRunner "androidx.test.runner.AndroidJUnitRunner"
    }

    buildTypes {
        release {
            minifyEnabled false
            proguardFiles getDefaultProguardFile('proguard-android-optimize.txt'), 'proguard-rules.pro'
        }
    }
    compileOptions {
        sourceCompatibility JavaVersion.VERSION_1_8
        targetCompatibility JavaVersion.VERSION_1_8
    }
    kotlinOptions {
        jvmTarget = '1.8'
    }

}

dependencies {

    implementation 'androidx.core:core-ktx:1.7.0'
    implementation 'androidx.appcompat:appcompat:1.4.1'
}

```


#### android{}
``` bash
android {
    namespace 'com.yechaoa.gradlex'  // 应用程序的命名空间。主要用于访问应用程序资源。
	compileSdk 32    // 编译所依赖的Android SDK的版本，即API Level，可以使用此API级别及更低级别中包含的API功能
	
	// 默认配置，它是一个ProductFlavor。ProductFlavor允许我们根据不同的情况同时生成多个不同的apk包。
    defaultConfig {
        applicationId "com.yechaoa.gradlex"
        minSdk 23
        targetSdk 32
        versionCode 1
        versionName "1.0"

        testInstrumentationRunner "androidx.test.runner.AndroidJUnitRunner"
	    ndk {  
		    // sdk提供的so只有armeabi-v7a，如果不指定cpu类型，引入的第三方有可能有其它类型的so，  
			// 这时候如果运行在非armeabi-v7a cpu的手机时，就会在对应的jniLibs找so，然后出现UnsatisfiedLinkError。  
		    abiFilters "arm64-v8a", "armeabi-v7a"  
		}  
  
		javaCompileOptions {  
		    annotationProcessorOptions {  
		        includeCompileClasspath true  
		    }  
		}
    }

    buildTypes {
        release {
            minifyEnabled false
            proguardFiles getDefaultProguardFile('proguard-android-optimize.txt'), 'proguard-rules.pro'
        }
    }
    compileOptions {
        sourceCompatibility JavaVersion.VERSION_1_8
        targetCompatibility JavaVersion.VERSION_1_8
    }
    kotlinOptions {
        jvmTarget = '1.8'
    }
	sourceSets {  
	    main {  
	        java.srcDirs = ['src/main/java']  
	        assets.srcDir new File(buildDir, "asset-skin").path  
	    }  
	    debug {  
	        java.srcDirs = ['src/debug/java', 'build/generated/source/apt2/debug']  
	    }  
	    release {  
	        java.srcDirs = ['src/release/java', 'build/generated/source/apt2/release']  
	    }  
	}
}
```


##### multiDexEnabled

用于配置该BuildType是否启用自动拆分多个Dex的功能。一般用程序中代码太多，超过了65535个方法的时候。

##### ndk{}

多平台编译，生成有so包的时候使用，之前用armeabi的比较多，要求做32/64位适配之后开始拆分（提高性能），'armeabi-v7a'表示32位cpu架构, 'arm64-v8a'表示64位, 'x86'是只模拟器或特定rom。

##### sourceSets

源代码集合，是 Java 插件用来描述和管理源代码及资源的一个抽象概念，是一个 Java 源代码文件和资源文件的集合，我们可以通过 sourceSets 更改源集的 Java 目录或者资源目录等。

##### buildTypes

构建类型，在 Gradle Android 工程中，它已经帮我们内置了 release 构建类型，一般还会加上 debug，两种模式主要区别在于，能否在设备上调试以及签名不一样，其他代码和文件资源都是一样的

- name：build type 的名字
- applicationIdSuffix：应用 id 后缀
- versionNameSuffix：版本名称后缀
- debuggable：是否生成一个 debug 的 apk
- minifyEnabled：是否混淆
- proguardFiles：混淆文件
- signingConfig：签名配置
- manifestPlaceholders：清单占位符
- shrinkResources：是否去除未利用的资源，默认 false，表示不去除。
- zipAlignEnable：是否使用 zipalign 工具压缩。
- multiDexEnabled：是否拆成多个 Dex
- multiDexKeepFile：指定文本文件编译进主 Dex 文件中
- multiDexKeepProguard：指定混淆文件编译进主 Dex 文件中


##### signingConfigs

签名配置，一个 app 只有在签名之后才能被发布、安装、使用，签名是保护 app 的方式，标记该 app 的唯一性。

##### productFlavors

多渠道打包配置，可以实现定制化版本的需求。
```
productFlavors {  
    video {  
        dimension "mode"  
    }  
  
    full {  
        dimension "mode"  
    }  
}
```

##### buildConfigField

他是BuildConfig文件的一个函数，而BuildConfig这个类是Android Gradle构建脚本在编译后生成的。比如版本号、一些标识位什么的。
```
buildConfigField "String", "SDK_VERSION", "\"${sdkVersion}\""
```
##### Build Variants

选择编译的版本。

##### buildFeatures

开启或关闭构建功能，常见的有viewBinding、dataBinding、compose。
```
    buildFeatures {
        viewBinding = true
        // dataBinding = true
    }

```


##### compileOptions

Java 编译选项，指定 java 环境版本。
```
compileOptions {  
    sourceCompatibility JavaVersion.VERSION_1_8  
    targetCompatibility JavaVersion.VERSION_1_8  
}
```




#### dependencies

#### plugins
```
plugins { 
	id 'com.android.application' 
	id 'org.jetbrains.kotlin.android' 
}
```

### settings.gradle

位于项目的根目录下，用于定义项目级代码库设置。
```
pluginManagement {
    repositories {
        gradlePluginPortal()
        google()
        mavenCentral()
    }
}
dependencyResolutionManagement {
    repositoriesMode.set(RepositoriesMode.FAIL_ON_PROJECT_REPOS)
    repositories {
        google()
        mavenCentral()
    }
}
rootProject.name = "GradleX"
include ':app'
```

#### pluginManagement

插件管理，指定插件下载的仓库，及版本。

#### dependencyResolutionManagement

依赖管理，指定依赖库的仓库地址，及版本。即7.0之前的allprojects。

顺序决定了先从哪个仓库去找依赖库并下载，一般为了编译稳定，会把阿里的镜像地址（或自建私有仓库）放在Google()仓库之前。

#### rootProject.name

项目名称。

#### include

用于指定构建应用时应将哪些模块包含在内，即参与构建的模块。

也可用于动态引用，在编译提速时，会module打成aar依赖来节省编译时间，但是为了开发方便，一般会动态选择哪些module使用源码依赖。

  


### gradle.properties

位于项目的根目录下，用于指定 Gradle 构建工具包本身的设置，也可用于项目版本管理。

#### Gradle 本身配置

比如Gradle 守护程序的最大堆大小、编译缓存、并行编译、是否使用Androidx等等。
```
#并行编译 
org.gradle.parallel=true 
#构建缓存 
org.gradle.caching=true

org.gradle.jvmargs=-Xmx2048m -XX:MaxPermSize=512m

#开启并行编译，相当于多条线程再走  
org.gradle.parallel=true  
#启用新的孵化模式  
org.gradle.configureondemand=true  
#启用新的DEX编译器：D8  
android.enableD8=true  
#打开编译缓存功能：Gradle Cache  
org.gradle.caching=true


```

#### 版本管理
```
# buildSrc 和 项目公共使用的字段  
buildToolsVersion=4.2.0  
kotlinVersion=1.4.32
yechaoaPluginVersion="1.0.0"
```

在 settings. gradle 中可以这样获取
```
pluginManagement {
  plugins {
        id 'com.yechaoa.gradlex' version "${yechaoaPluginVersion}"
    }
}

```



### local.properties
位于项目的根目录下，用于指定 Gradle 构建配置本地环境属性，也可用于项目环境管理。
```
sdk.dir=/Users/yechao/Library/Android/sdk
ndk.dir=/Users/yechao/Library/Android/ndk
```

用作项目本地调试的一些开关：
```
isRelease=true
#isDebug=false
#isH5Debug=false
```
