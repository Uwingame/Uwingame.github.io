---
title: Android Flavors使用
date: 2017-04-10 11:19:59
tags: 
	- Android
---
## 通过buildConfigField控制版本
添加一个生产版本和内部测试版本的flavors
```groovy
productFlavors {
        internal {
            buildConfigField "boolean", "PRODUCTION", "false"
            multiDexEnabled true
        }
        production {
            buildConfigField "boolean", "PRODUCTION", "true"
            resConfigs "ar"
            multiDexEnabled true
        }
    }
```
在代码中就可以通过BuildConfig.PRODUCTION判断是否是正式版
```java
if (BuildConfig.PRODUCTION) {
	// 正式版
} else {
	// 测试版
}
```

##正式版和测试版代码隔离
只在正式版上传Firebase Crash信息

app的build.gradle中
```
productionCompile 'com.google.firebase:firebase-crash:10.2.1'
```
internal版本中
```java
public class CrashReport {

    public static void report(Throwable e) {

    }
}
```
production版本中
```java
public class CrashReport {

    public static void report(Throwable e) {
        FirebaseCrash.report(e);
    }
}
```

## 为测试版添加一个测试工具Activity
在app/src/internal文件夹下添加AndroidManifest.xml
```xml
<manifest xmlns:android="http://schemas.android.com/apk/res/android">

    <application>
        <activity
            android:name="com.example.TestToolBoxActivity"
            android:icon="@mipmap/ic_launcher_round"
            android:label="Hayaa Test">
            <intent-filter>
                <action android:name="android.intent.action.MAIN"/>
                <category android:name="android.intent.category.LAUNCHER"/>
            </intent-filter>
        </activity>
    </application>
</manifest>

```
新建对应的Activity，测试版启动的时候就会多出一条测试工具的入口
